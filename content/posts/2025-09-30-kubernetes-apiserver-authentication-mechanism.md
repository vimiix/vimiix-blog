---
title: 'Kubernetes APIServer 认证机制'
date: 2025-09-30 08:55:48
categories: 'kubernetes'
mermaid: true
tags:
  - 'kubernetes'
  - 'notes'
  - 'k8s'
---

Kubernetes 作为一个分布式系统，其 API Server 是所有组件交互的核心枢纽。为了保证集群的安全，所有访问 API Server 的请求都必须经过认证（Authentication）、鉴权（Authorization）和准入控制（Admission Control）等步骤。其中，认证是安全的第一道大门，它负责确认请求者的身份。

Kubernetes 支持多种认证机制（称为 **Authenticator**），以适应不同的部署环境和安全需求。这些机制可以同时启用，API Server 会逐一尝试，直到有一个机制成功认证该请求。

<!--more-->

## 一、认证流程总览

任何访问 API Server 的请求（来自 `kubectl`、 Dashboard、 节点上的 kubelet 或其他控制器）都需要携带认证信息。API Server 接收到请求后，会依次尝试配置的认证策略。如果认证成功，请求者的用户名（`username`）和用户组（`groups`）信息会被嵌入到请求的上下文中，传递给后续的鉴权阶段。如果所有认证策略都失败，则请求以 `401 Unauthorized` 错误被拒绝。

下图描绘了这个通用的认证决策流程：

{{<mermaid>}}
flowchart TD
    A[客户端请求<br>携带认证信息] --> B[API Server]
    
    subgraph B [API Server 认证环节]
        direction LR
        C[尝试认证策略 1<br>（如: X509证书）] -->|失败| D[尝试认证策略 2<br>（如: Bearer Token）]
        D -->|失败| E[尝试认证策略 N...<br>（如: ServiceAccount Token）]
        E -->|失败| F[认证失败]
        C -->|成功| G[认证成功]
        D -->|成功| G
        E -->|成功| G
    end

    F --> H[返回 401 Unauthorized]
    G --> I[将身份信息嵌入请求上下文<br>（username, groups, uid等）]
    I --> J[进入鉴权 Authorization 阶段]
{{</mermaid>}}

## 二、各种认证机制

### 1. X509 客户端证书认证 (TLS)

这是集群间组件通信最常用和最安全的机制之一，尤其适用于集群内部组件（如 kubelet、scheduler、controller-manager）与 API Server 的通信。

#### 原理：

API Server 启动时需要配置一个 Certificate Authority (CA) 根证书（通常由 `--client-ca-file=SOMEFILE` 参数指定）。客户端需要持有一张由该 CA 签名的客户端证书。在建立 TLS 连接时，客户端会出示此证书。API Server 会验证证书的有效性（是否由信任的 CA 签发、是否在有效期内、CN 和 O 字段等）。

适用于集群组件身份认证、为外部用户或系统签发长期有效的证书。

#### 身份信息提取：

- **用户名 (username)**： 取自证书的 `CN` (Common Name) 字段。
- **用户组 (group)**： 取自证书的 `O` (Organization) 字段。

#### 时序图：

{{<mermaid>}}
sequenceDiagram
    participant C as Client (持有客户端证书)
    participant A as API Server<br>(持有CA根证书)

    C->>A: 1. HTTPS Request (Client Hello + 客户端证书)
    Note right of C: 证书由集群CA签发
    A->>A: 2. 使用 --client-ca-file 验证证书
    Note right of A: a. 验证CA签名<br>b. 验证有效期/吊销状态等<br>c. 提取 CN 和 O 字段
    alt 验证成功
        A-->>C: 3. TLS连接建立成功
        A->>A: 4. 认证身份: username=CN, groups=O
        A->>A: 5. 进入鉴权阶段
    else 验证失败
        A-->>C: 3. TLS握手失败 (证书无效)
    end
{{</mermaid>}}

### 2. 静态令牌文件 (Static Token File)

一种简单的认证方式，通过一个静态文件来管理令牌（Token）。

#### 原理：

API Server 通过 `--token-auth-file=SOMEFILE` 参数指定一个包含预定义令牌的 CSV 文件。文件格式为：`token,username,uid,"group1,group2,group3"`。客户端在请求头中携带 `Authorization: Bearer <TOKEN>`。API Server 会在文件中查找匹配的令牌。

这种认证方式的缺点是，令牌是静态的，修改需要重启 API Server，非常不灵活且不安全。一般用于临时测试或学习，**不推荐在生产环境中使用**。

### 时序图：

{{<mermaid>}}
sequenceDiagram
    participant C as Client
    participant A as API Server<br>(--token-auth-file=tokens.csv)

    C->>A: 1. HTTP Request<br>Authorization: Bearer abc123
    A->>A: 2. 在 tokens.csv 中查找令牌 'abc123'
    alt 找到匹配项
        A->>A: 3. 认证身份: 提取文件中对应用户名和组
        A->>A: 4. 进入鉴权阶段
    else 未找到
        A-->>C: 5. 返回 401 Unauthorized
    end
{{</mermaid>}}

### 3. ServiceAccount 令牌 (Bearer Tokens)

这是 **Kubernetes Pod 内部应用访问 API Server 最主要和推荐的方式**。

#### 原理：

1.  **ServiceAccount (SA)**： 一个命名空间内的资源，代表一个应用身份。
2.  **Secret**： 当创建 SA（或由控制器自动创建）时，Kubernetes 会自动生成一个对应的 Secret。该 Secret 包含三个关键信息：`ca.crt` (API Server 的 CA)、`namespace` (当前命名空间)、`token` (一个签名的 JWT 令牌)。
3.  **令牌挂载**： Pod 通过 `spec.serviceAccountName` 指定 SA，对应的 Secret 会自动挂载到 Pod 内的 `/var/run/secrets/kubernetes.io/serviceaccount/`。
4.  **JWT 验证**： 客户端（Pod 内的进程）使用此令牌在请求头中设置 `Authorization: Bearer <JWT_TOKEN>`。API Server 使用其持有的密钥（通过 `--service-account-key-file` 指定，默认为自身 TLS 私钥）来验证 JWT 的签名。

#### 时序图：

{{<mermaid>}}
sequenceDiagram
    participant P as Pod中的应用程序
    participant A as API Server
    participant S as Secret

    Note over P, S: Pod 启动前
    A->>S: 1. 为 default SA 创建 Secret<br>(包含 token JWT, ca.crt)

    Note over P, A: Pod 运行时
    P->>P: 2. 从挂载卷读取 token 和 ca.crt
    P->>A: 3. HTTPS Request<br>Authorization: Bearer {JWT}
    A->>A: 4. 使用自身密钥验证JWT签名<br>并解析载荷（payload）
    alt JWT有效
        A->>A: 5. 认证身份: system:serviceaccount:{ns}:{sa-name}<br>用户组: system:serviceaccounts
        A->>A: 6. 进入鉴权阶段
    else JWT无效
        A-->>P: 7. 返回 401 Unauthorized
    end
{{</mermaid>}}

### 4. Bootstrap 令牌 (Bootstrap Tokens)

这是一种为了简化节点加入集群流程而设计的**短期**令牌机制，是静态令牌的“动态”方案，推荐 worker 节点首次加入集群时使用。

#### 原理：

1.  **令牌格式**： 格式为 `[a-z0-9]{6}.[a-z0-9]{16}` (例如 `abcdef.0123456789abcdef`)。
2.  **Secret 资源**： Bootstrap 令牌实际上对应着 `kube-system` 命名空间中的一个特定 Secret，其类型为 `bootstrap.kubernetes.io/token`。这使得令牌可以通过 API 动态管理，无需重启 API Server。
3.  **用途**： 新节点使用此令牌发起 TLS 引导请求，API Server 的 Bootstrap Token Authenticator 会验证 Secret 是否存在且未过期。认证成功后，节点随后会申请正式的证书用于长期通信。

#### 时序图：

{{<mermaid>}}
sequenceDiagram
    participant N as New Node (新节点)
    participant A as API Server
    participant C as Controller Manager<br>(TokenCleaner)

    N->>A: 1. 证书签名请求(CSR)<br>Authorization: Bearer bootstrap-token-abcdef
    A->>A: 2. 检查 kube-system 中是否存在<br>对应的 bootstrap-token Secret
    alt Secret 存在且有效
        A->>A: 3. 认证成功
        A->>A: 4. 批准CSR（可能需要手动）
        A-->>N: 5. 颁发正式客户端证书
        N->>A: 6. 使用新证书进行后续通信
        C->>C: 7. (稍后) 自动清理过期的bootstrap Secret
    else Secret 无效或不存在
        A-->>N: 8. 返回 401，加入集群失败
    end
{{</mermaid>}}

### 5. OpenID Connect (OIDC) 令牌

这是一种连接企业现有身份系统（如 Google、Azure AD、Keycloak、Dex 等）的**标准协议**，用于集成外部用户认证系统。适用于**开发者、管理员等人类用户通过企业单点登录（SSO）系统访问集群**。

#### 原理：

1.  **三方协作**： 涉及客户端（`kubectl`）、API Server 和外部身份提供商（IdP）。
2.  **用户登录**： 用户首先通过 `kubectl` 在 **外部** IdP 完成认证，获取一个 IdP 签名的 JWT 令牌（ID Token）。
3.  **传递令牌**： `kubectl` 在请求 API Server 时，在 `Authorization` 头中携带此 ID Token。
4.  **API Server 验证**： API Server 不直接与 IdP 通信，而是通过配置（`--oidc-issuer-url`, `--oidc-client-id`）获取 IdP 的公钥，来验证 JWT 令牌的签名。它还会验证令牌的受众（`aud` Claim）是否与 `--oidc-client-id` 匹配。

#### 时序图：

{{<mermaid>}}
sequenceDiagram
    participant U as User (kubectl)
    participant A as APIServer
    participant I as OIDC Identity Provider (IdP)

    U->>I: 1. 用户登录 (获取ID Token)
    I-->>U: 2. 返回签名的 JWT ID Token
    U->>A: 3. API Request<br>Authorization: Bearer {IDToken}
    A->>A: 4. 从配置的issuer-url获取JWKS公钥
    A->>A: 5. 使用公钥验证JWT签名<br>并检查audience/issuer/有效期
    alt 验证成功
        A->>A: 6. 认证身份: 提取JWT中的preferred_username/groups等字段
        A->>A: 7. 进入鉴权阶段
    else 验证失败
        A-->>U: 8. 返回 401 Unauthorized
    end
{{</mermaid>}}

### 6. Webhook 令牌认证

这是一种**扩展机制**，当 Kubernetes 内置的认证方式都无法满足需求时，可以将认证决策委托给一个外部的 RESTful 服务，适用于集成自定义的认证系统、私有身份存储等。

#### 原理：

1.  **配置**： API Server 启动时配置 `--authentication-token-webhook-config-file`，该文件指向一个外部服务的配置（URL、CA 等）。
2.  **委托验证**： 当客户端携带 Bearer Token 请求时，API Server 会将一个 `TokenReview` 对象发送给外部 Webhook 服务。
3.  **外部决策**： Webhook 服务验证令牌的有效性，并将结果（认证成功与否，以及用户信息）封装在 `TokenReview` 对象的 `status` 字段中返回。
4.  **API Server 决策**： API Server 根据 Webhook 的响应决定是否认证通过。

#### 时序图：

{{<mermaid>}}
sequenceDiagram
    participant C as Client
    participant A as APIServer
    participant W as External Webhook Service

    C->>A: 1. API Request<br>Authorization: Bearer {TOKEN}
    A->>W: 2. 构造并发送 TokenReview Request<br>{apiVersion: authentication.k8s.io/v1, kind: TokenReview, spec: {token: "TOKEN"}}
    W->>W: 3. 根据自定义逻辑验证Token<br>（查数据库、调用其他API等）
    W-->>A: 4. 返回 TokenReview Response<br>{status: {authenticated: true, user: {username: "foo", groups: ["bar"]}}}
    alt authenticated: true
        A->>A: 5. 认证成功，使用返回的用户信息
        A->>A: 6. 进入鉴权阶段
    else authenticated: false
        A-->>C: 7. 返回 401 Unauthorized
    end
{{</mermaid>}}

## 三、总结对比

| 认证机制                | 主要用途            | 管理方式         | 安全性   | 生产环境推荐          |
| :------------------: | :--------------: | :-----------: | :----: | :--------------: |
| **X509 证书**         | 组件间通信，系统级访问     | 动态（CA签发）     | 非常高   | **推荐**（用于组件）  |
| **ServiceAccount**  | **Pod内应用**访问API | 动态（K8s自动管理）  | 高     | **必须使用**（用于Pod） |
| **Bootstrap Token** | **新节点加入**集群     | 动态（Secret资源） | 中（短期） | **推荐**（用于节点引导）  |
| **OIDC**            | **人类用户**单点登录    | 外部IdP管理      | 高     | **推荐**（用于用户）  |
| **Webhook**         | 集成自定义认证系统       | 外部服务管理       | 取决于实现 | 按需使用            |
| **静态Token文件**       | 测试              | 静态文件，需重启     | 低     | **不推荐**        |

理解这些认证机制的原理和适用场景，是构建和维护一个安全、可靠的 Kubernetes 集群的基石。

