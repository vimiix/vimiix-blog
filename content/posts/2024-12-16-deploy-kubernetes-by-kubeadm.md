---
title: '使用 kubeadm 搭建 kubernetes 集群'
date: 2024-12-16 15:23:48
categories: 'kubernetes'
tags:
  - 'kubernetes'
  - 'notes'
---

准备4台虚拟机，拓扑为一个 master, 3 个 node 节点，操作系统均为 CentOS 7.9，网络采用桥接模式，设置静态IP。

| 机器名 | IP            |
| ------ | ------------- |
| master | 192.168.3.200 |
| node1  | 192.168.3.201 |
| node2  | 192.168.3.202 |
| node3  | 192.168.3.203 |

<!--more-->

## 安装前环境准备

1. YUM 源修改

```shell
# 1. 修改 yum 源到国内
curl -o /etc/yum.repos.d/CentOS-Base.repo https://mirrors.aliyun.com/repo/Centos-7.repo

# 2. 安装部分依赖包
yum install -y yum-utils device-mapper-persistent-data lvm2

# 3. 增加 docker yum 源
yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

# 4. 增加 kubernetes yum 源
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
       https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```

2. 安装一些常用工具

```shell
yum install wget jq psmisc vim net-tools telnet git -y
```

3. 关闭防火墙，SELinux，DNSmasq(配置DNS和DHCP的轻量级工具)，swap

```shell
systemctl disable --now firewalld
systemctl disable --now dnsmasq

swapoff -a && sysctl -w vm.swappiness=0
sed -ri '/^[^#]*swap/s@^@#@' /etc/fstab

# 修改 /etc/selinux/config 文件中的配置为 disabled，然后重启生效
sed -i 's#SELINUX=enforcing#SELINUX=disabled#g' /etc/selinux/config

```

4. 安装 ntpdate，同步时钟 (如果已经配置了时钟同步服务器，则跳过)

```shell
rpm -ivh http://mirrors.wlnmp.com/centos/wlnmp-release-centos.noarch.rpm && \
  yum install ntpdate -y && \
  ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime && \
  echo 'Asia/Shanghai' > /etc/timezone && \
  ntpdate time2.aliyun.com

# 设置定时任务每5分钟同步一下时钟
crontab -e
# 配置如下规则
*/5 * * * * /usr/sbin/ntpdate time2.aliyun.com
```

5. 默认 limit 配置

```shell
ulimit -SHn 65535

vim /etc/security/limits.conf
# 末尾追加如下内容

* soft nofile 65536
* hard nofile 131072
* soft nproc 65535
* hard nproc 655350
* soft memlock unlimited
* hard memlock unlimited
```

 6. 在master 生成ssh秘钥，拷贝到其他节点，实现master免密登录其他节点

```shell
ssh-keygen -t rsa
for i in node1 node2 node3; do ssh-copy-id -i .ssh/id_rsa.pub $i; done
```

## 安装 k8s 集群

> 1.24版本以后默认使用containerd 作为runtime，所以安装 containerd
>

```shell
# 修改必要系统配置
modprobe br_netfilter
echo 1 > /proc/sys/net/bridge/bridge-nf-call-iptables
echo 1 > /proc/sys/net/ipv4/ip_forward

yum install -y containerd kubelet kubeadm kubectl --disableexcludes=kubernetes
# 启动 containerd
systemctl daemon-reload
systemctl enable containerd
systemctl enable kubelet

# 把 containerd 默认配置写到配置文件中
containerd config default > /etc/containerd/config.toml
# 编辑 /etc/containerd/config.toml 文件，
# 如果存在 disabled_plugins = ["cri"]，把列表中的cri 去掉，或者注释掉这一行
# 否则 kubeadm init 的时候会报错 "CRI v1 image API is not implemented for endpoint"

# 另外把 containerd 的 cgroup 驱动改为 systemd，找到如下行，把SystemdCgroup 改为true
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
    SystemdCgroup = true

# 国内配置一下国内源，修改配置文件中 config_path 的值
  config_path = "/etc/containerd/certs.d"

# 保存配置文件退出，然后创建/etc/containerd/certs.d/目录，在这个目录下创建不同站点的代理
# hosts.toml中可以配置多个镜像仓库，containerd下载镜像时会根据配置的顺序使用镜像仓库，
# 只有当上一个仓库下载失败才会使用下一个镜像仓库。
# 因此，镜像仓库的配置原则就是镜像仓库下载速度越快，那么这个仓库就应该放在最前面。

# docker hub镜像加速
mkdir -p /etc/containerd/certs.d/docker.io
cat > /etc/containerd/certs.d/docker.io/hosts.toml << EOF
server = "https://docker.io"
[host."https://dockerproxy.com"]
  capabilities = ["pull", "resolve"]

[host."https://docker.m.daocloud.io"]
  capabilities = ["pull", "resolve"]

[host."https://reg-mirror.qiniu.com"]
  capabilities = ["pull", "resolve"]

[host."https://registry.docker-cn.com"]
  capabilities = ["pull", "resolve"]

[host."http://hub-mirror.c.163.com"]
  capabilities = ["pull", "resolve"]

EOF

# registry.k8s.io镜像加速
mkdir -p /etc/containerd/certs.d/registry.k8s.io
tee /etc/containerd/certs.d/registry.k8s.io/hosts.toml << 'EOF'
server = "https://registry.k8s.io"

[host."https://k8s.m.daocloud.io"]
  capabilities = ["pull", "resolve", "push"]
EOF

# docker.elastic.co镜像加速
mkdir -p /etc/containerd/certs.d/docker.elastic.co
tee /etc/containerd/certs.d/docker.elastic.co/hosts.toml << 'EOF'
server = "https://docker.elastic.co"

[host."https://elastic.m.daocloud.io"]
  capabilities = ["pull", "resolve", "push"]
EOF

# gcr.io镜像加速
mkdir -p /etc/containerd/certs.d/gcr.io
tee /etc/containerd/certs.d/gcr.io/hosts.toml << 'EOF'
server = "https://gcr.io"

[host."https://gcr.m.daocloud.io"]
  capabilities = ["pull", "resolve", "push"]
EOF

# ghcr.io镜像加速
mkdir -p /etc/containerd/certs.d/ghcr.io
tee /etc/containerd/certs.d/ghcr.io/hosts.toml << 'EOF'
server = "https://ghcr.io"

[host."https://ghcr.m.daocloud.io"]
  capabilities = ["pull", "resolve", "push"]
EOF

# k8s.gcr.io镜像加速
mkdir -p /etc/containerd/certs.d/k8s.gcr.io
tee /etc/containerd/certs.d/k8s.gcr.io/hosts.toml << 'EOF'
server = "https://k8s.gcr.io"

[host."https://k8s-gcr.m.daocloud.io"]
  capabilities = ["pull", "resolve", "push"]
EOF

# mcr.m.daocloud.io镜像加速
mkdir -p /etc/containerd/certs.d/mcr.microsoft.com
tee /etc/containerd/certs.d/mcr.microsoft.com/hosts.toml << 'EOF'
server = "https://mcr.microsoft.com"

[host."https://mcr.m.daocloud.io"]
  capabilities = ["pull", "resolve", "push"]
EOF

# nvcr.io镜像加速
mkdir -p /etc/containerd/certs.d/nvcr.io
tee /etc/containerd/certs.d/nvcr.io/hosts.toml << 'EOF'
server = "https://nvcr.io"

[host."https://nvcr.m.daocloud.io"]
  capabilities = ["pull", "resolve", "push"]
EOF

# quay.io镜像加速
mkdir -p /etc/containerd/certs.d/quay.io
tee /etc/containerd/certs.d/quay.io/hosts.toml << 'EOF'
server = "https://quay.io"

[host."https://quay.m.daocloud.io"]
  capabilities = ["pull", "resolve", "push"]
EOF

# registry.jujucharms.com镜像加速
mkdir -p /etc/containerd/certs.d/registry.jujucharms.com
tee /etc/containerd/certs.d/registry.jujucharms.com/hosts.toml << 'EOF'
server = "https://registry.jujucharms.com"

[host."https://jujucharms.m.daocloud.io"]
  capabilities = ["pull", "resolve", "push"]
EOF

# rocks.canonical.com镜像加速
mkdir -p /etc/containerd/certs.d/rocks.canonical.com
tee /etc/containerd/certs.d/rocks.canonical.com/hosts.toml << 'EOF'
server = "https://rocks.canonical.com"

[host."https://rocks-canonical.m.daocloud.io"]
  capabilities = ["pull", "resolve", "push"]
EOF


# 重启 containerd 服务
systemctl restart containerd

```

为了加快集群部署速度，预先下载kubeadm 所需的全部镜像

```shell
# （optional） 查看 kubeadm 所需的全部镜像
kubeadm config  images list

# 拉去所需镜像
kubeadm config  images pull
```

在master 上通过kubeadm一键安装 master节点

```shell
kubeadm init
```

在所有 node 节点上执行 `kubeadm init` 日志最后的 join 命令

```shell
# 这个命令根据实际的输出内容执行
kubeadm join 192.168.3.200:6443 --token 9unwvu.jogfdub8buf3y2os \
        --discovery-token-ca-cert-hash sha256:cad35b01dc5094aa82456e2d166393f5d790e4a452ebbd3df39801cd8997a241
```

设置kubectl 要加载的配置，可以通过把配置文件放到 ~/.kube/config 或者设置 KUBECONFIG 环境变量

这里我使用环境变量的方式，将设置环境变量的命令放到 ~/.bashrc 里

```shell
# master 节点
echo 'export KUBECONFIG=/etc/kubernetes/admin.conf' >> ~/.bashrc
# node 节点
echo 'export KUBECONFIG=/etc/kubernetes/kubelet.conf' >> ~/.bashrc
source ~/.bashrc

# 测试
[root@master ~]# kubectl get nodes
NAME     STATUS     ROLES           AGE   VERSION
master   NotReady   control-plane   19m   v1.28.2
node1    NotReady   <none>          12m   v1.28.2
node2    NotReady   <none>          12m   v1.28.2
node3    NotReady   <none>          12m   v1.28.2
```

## 安装  calico 网络插件

此时所以得节点还是处于 `NotReady` 状态，因为还没有安装 CNI 网络插件

```shell
kubectl apply -f "https://docs.projectcalico.org/manifests/calico.yaml"
```

(Optional) 安装 nerdctl 用于手动下载镜像

```shell
wget https://github.com/containerd/nerdctl/releases/download/v2.0.2/nerdctl-2.0.2-linux-amd64.tar.gz
mkdir -p /usr/local/containerd/bin/ && \
  tar -zxvf nerdctl-2.0.2-linux-amd64.tar.gz -C /usr/local/containerd/bin/ && \
  ln -s /usr/local/containerd/bin/nerdctl /usr/local/bin/nerdctl
```
