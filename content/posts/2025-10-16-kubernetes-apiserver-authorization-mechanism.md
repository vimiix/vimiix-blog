---
title: 'Kubernetes APIServer 鉴权机制'
date: 2025-10-16 07:54:28
categories: 'kubernetes'
mermaid: true
tags:
  - 'kubernetes'
  - 'notes'
  - 'k8s'
---

## 1. 引言

在 Kubernetes 集群中，[认证（Authentication）](./posts/2025-09-30-kubernetes-apiserver-authentication-mechanism)解决了“你是谁”的问题，而鉴权（Authorization）则解决了“你能做什么”的问题。鉴权是 Kubernetes 安全框架的核心支柱之一，它位于认证之后，用于确定一个已经通过身份认证的用户或服务账户（ServiceAccount）是否拥有对特定资源执行特定操作的权限。Kubernetes 采用了一种声明式的、基于属性的访问控制（ABAC）和基于角色的访问控制（RBAC）等模型来实现精细化的权限管理。本文将简单介绍 Kubernetes 的鉴权机制，包括其架构、工作流程、核心模型及详细实现原理。

<!--more-->

## 2. 鉴权架构

Kubernetes API Server 是集群所有操作的唯一入口，它也是鉴权决策的中心。当一个请求通过认证后，API Server 会将该请求传递给鉴权模块进行处理。

Kubernetes 支持多种鉴权模块，例如：

- **Node**： 用于授权 kubelet 对节点和 Pod 等资源的访问。
- **ABAC（Attribute-Based Access Control）**： 基于属性的访问控制，策略存储在静态文件中。
- **RBAC（Role-Based Access Control）**： 基于角色的访问控制，是当前最主流和推荐的方式。
- **Webhook**： 通过调用外部 RESTful 服务来做出鉴权决策。

这些模块可以同时配置多个。如果配置了多个模块，请求必须被至少一个模块允许，且不能被任何模块拒绝，整个请求才会被授权。API Server 默认允许一个匿名请求，除非被认证层或鉴权层拒绝。

下图是请求经过认证与鉴权的核心流程：

{{<mermaid>}}
flowchart TD
    A[客户端请求<br>（User/ServiceAccount）] --> B[API Server]
    B --> C{认证}
    C -- 携带合法令牌/证书 --> D[认证成功<br>获取请求者身份信息]
    C -- 令牌/证书无效 --> E[认证失败<br>返回401 Unauthorized]
    D --> F[鉴权模块链]
    
    subgraph F [鉴权模块链]
        direction LR
        F1[Node Authorizer] --> F2[ABAC Authorizer] --> F3[RBAC Authorizer] --> F4[Webhook Authorizer]
    end

    F -- 任一模块显式拒绝 --> G[鉴权拒绝<br>返回403 Forbidden]
    F -- 所有模块未通过<br>（包括默认匿名允许） --> H[鉴权拒绝<br>返回403 Forbidden]
    F -- 任一模块允许<br>且无模块拒绝 --> I[鉴权通过]
    
    I --> J[准入控制<br>（Mutating/Validating）]
    J --> K[持久化存储<br>（etcd）]
{{</mermaid>}}

## 3. 鉴权核心流程

一个请求的鉴权过程涉及以下几个关键步骤和对象：

### 3.1. 请求上下文的构建

在认证成功后，API Server 会构建一个包含请求者身份和请求详细信息的上下文对象，该对象将作为输入传递给鉴权模块。这个上下文主要包括：

1. **用户（User）**： 认证后得到的用户名。
2. **用户组（Groups）**： 用户所属的组。例如，`system:authenticated` 组包含所有已认证用户。
3. **额外信息（Extra）**： 包含一些额外的键值对信息，通常由认证器提供。
4. **API 请求信息**：
    - **API Verb**： 请求的操作类型，如 `get`, `list`, `create`, `update`, `delete`, `patch`, `watch`。
    - **Resource**： 请求的 Kubernetes 资源，如 `pods`, `services`, `deployments`。
    - **Namespace**： 如果资源是命名空间范围的，则包含其命名空间。
    - **API Group**： 资源所属的 API 组，如 `apps`（对应 `deployments`）、`”`（核心组，对应 `pods`, `services`）。
    - **子资源（Subresource）**： 如 `pods/exec`, `pods/log`, `pods/status`。
    - **请求路径（Request Path）**： 对于非资源请求（如 `/api`, `/healthz`），使用完整路径进行鉴权。
    - **资源名称（Resource Name）**： 某些鉴权模式（如 ABAC）可以支持基于特定资源实例名称的授权。

### 3.2. 多模块决策机制

API Server 会按照配置顺序，依次将请求上下文传递给每个已启用的鉴权模块。每个模块独立做出决策，决策结果有三种：

```
const (
	// DecisionDeny means that an authorizer decided to deny the action.
	DecisionDeny Decision = iota
	// DecisionAllow means that an authorizer decided to allow the action.
	DecisionAllow
	// DecisionNoOpinion means that an authorizer has no opinion on whether
	// to allow or deny an action.
	DecisionNoOpinion
)
```

- **拒绝（Deny）**： 该模块明确拒绝此请求。
- **允许（Allow）**： 该模块允许此请求。
- **无意见（No Opinion）**： 该模块对此请求没有匹配的策略，不作决定。

最终的鉴权结果遵循以下规则：

1. 如果**任何一个**模块的决策是 **“拒绝”**，则整个请求被**拒绝**，API Server 返回 `403 Forbidden`。
2. 如果**至少一个**模块的决策是 **“允许”**，并且**没有任何**模块的决策是 **“拒绝”**，则整个请求被**允许**。
3. 如果**所有**模块的决策都是 **“无意见”**，则请求被**拒绝**。这是一种“默认拒绝”的安全策略。

## 4. 鉴权模型

### 4.1. RBAC（基于角色的访问控制）

RBAC 是 Kubernetes 中事实上的标准鉴权模型，它使用 `Role` / `ClusterRole` 和 `RoleBinding` / `ClusterRoleBinding` 这四种资源来定义和分配权限。

#### 4.1.1. 核心资源对象

1. **Role 与 ClusterRole**
    
- **Role**： 定义在**特定命名空间**内的一组权限规则。它指定了可以对哪些资源（如 `pods`）执行哪些操作（如 `get`, `list`, `create`）。
- **ClusterRole**： 定义在**集群范围**内的一组权限规则。用于定义：
    - 集群级别的资源权限（如 `nodes`, `persistentvolumes`）。
    - 非资源端点权限（如 `/healthz`）。
    - 跨所有命名空间的命名空间资源权限（需要与 `ClusterRoleBinding` 配合）。

**Role 示例：**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""] # 核心 API 组
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
```

**ClusterRole 示例：**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  # 不需要 "namespace"，因为 ClusterRole 是集群范围的
  name: cluster-node-viewer
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "list", "watch"]
```

**RoleBinding 与 ClusterRoleBinding**

- **RoleBinding**： 将 `Role` 或 `ClusterRole` 中定义的权限授予一个或多个用户或服务账户。`RoleBinding` 在特定命名空间中生效。如果它引用的是一个 `ClusterRole`，那么被授权者只能在 `RoleBinding` 所在的命名空间内行使该 `ClusterRole` 的权限。
- **ClusterRoleBinding**： 将 `ClusterRole` 中定义的权限授予一个或多个用户或服务账户，授权范围是**整个集群**。

**RoleBinding 示例（将 `pod-reader` Role 授予用户 `vimiix`）：**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: User
  name: vimiix
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

**ClusterRoleBinding 示例（将 `cluster-node-viewer` ClusterRole 授予组 `system:authenticated`）：**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: read-nodes-global
subjects:
- kind: Group
  name: system:authenticated
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-node-viewer
  apiGroup: rbac.authorization.k8s.io
```

#### 4.1.2. RBAC 鉴权器工作流程

RBAC 鉴权器接收到请求上下文后，会执行以下逻辑：

1. **识别主体（Subject）**： 从上下文中提取用户和用户组。
2. **查找绑定关系**：
    - 列出所有与当前用户及所属用户组匹配的 `RoleBinding` 和 `ClusterRoleBinding`。
3. **获取角色规则**：
    - 对于找到的每个 `RoleBinding`，获取其引用的 `Role` 或 `ClusterRole` 中定义的规则。
    - 对于找到的每个 `ClusterRoleBinding`，获取其引用的 `ClusterRole` 中定义的规则。
4. **规则匹配**：
    - 遍历所有收集到的规则，检查是否有任何一条规则与当前请求的 Verb、API Group、Resource、Namespace 和 Resource Name（如果指定）相匹配。
5. **做出决策**：
    - 如果找到匹配的规则，则返回**“允许”**。
    - 如果未找到任何匹配的规则，则返回**“无意见”**。

### 4.2. ABAC（基于属性的访问控制）

ABAC 允许基于资源、用户或请求环境的任意属性组合来定义复杂的策略。策略以每行一个 JSON 对象的形式存储在策略文件（如 `--authorization-policy-file=/etc/kubernetes/abac.json`）中。API Server 需要重启才能加载策略文件的变更，这在动态性上不如 RBAC。

**ABAC 策略文件示例：**

```yaml
{
  "apiVersion": "abac.authorization.kubernetes.io/v1beta1",
  "kind": "Policy",
  "spec": {
    "user": "alice",
    "namespace": "project1",
    "resource": "pods",
    "readonly": true
  }
}
{
  "apiVersion": "abac.authorization.kubernetes.io/v1beta1",
  "kind": "Policy",
  "spec": {
    "user": "kubelet",
    "resource": "pods",
    "readonly": false
  }
}
{
  "apiVersion": "abac.authorization.kubernetes.io/v1beta1",
  "kind": "Policy",
  "spec": {
    "group": "system:authenticated",
    "readonly": true,
    "nonResourcePath": "*"
  }
}
```

ABAC 鉴权器将请求的属性与策略文件中的每条策略进行匹配，如果请求属性是某条策略属性的超集，则该策略匹配，请求被允许。

### 4.3. Node 鉴权

Node 鉴权是专门授权给 kubelet 访问 Node、Pod、Service、Endpoints 等资源的模块。它自动将 `system:nodes` 组中的用户（由 kubelet 的客户端证书中的 `CN=system:node:<nodeName>` 标识）与名为 `system:node` 的 `ClusterRole` 的权限关联起来，该 `ClusterRole` 预定义了 kubelet 运行所需的最小权限集。

### 4.4. Webhook 鉴权

Webhook 鉴权将鉴权决策外包给外部 HTTP(S) 服务。当配置了 Webhook 模式（`--authorization-webhook-config-file`）后，API Server 会向指定的外部服务发送一个 `SubjectAccessReview` 对象，该对象包含了完整的请求上下文。外部服务返回的响应指示是否允许该请求。

这种方式允许集成企业现有的权限管理系统，例如 Open Policy Agent (OPA)。

## 5. 最佳实践

1. **遵循最小权限原则**： 只授予执行任务所必需的最小权限。
2. **优先使用 RBAC**： RBAC 资源是声明式的，可以通过 `kubectl` 和 YAML 文件轻松管理，无需重启 API Server，是当前的最佳实践。
3. **定期审计权限**： 使用 `kubectl auth can-i` 命令或进行定期 RBAC 评审，确保权限分配符合预期。
4. **谨慎使用 `cluster-admin`**： `cluster-admin` ClusterRole 拥有集群最高权限，应严格控制其分配。
5. **利用命名空间进行隔离**： 为不同的团队或项目创建独立的命名空间，并结合 RBAC 实现逻辑隔离。
6. **为服务账户分配明确的权限**： 运行在 Pod 中的应用程序通过服务账户进行身份认证，务必为其绑定精确的 Role 或 ClusterRole，避免使用默认的 `default` 服务账户或过宽的权限。

