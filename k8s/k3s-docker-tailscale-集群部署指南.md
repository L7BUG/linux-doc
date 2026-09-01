# K3s + Docker + Tailscale 多节点集群部署指南

> **背景**：在多台物理机或云服务器上用 Docker 容器运行 K3s，组建 3 台控制面（server）+ 工作节点（agent）的高可用 Kubernetes 集群，节点间通过 Tailscale 虚拟局域网互联。适用于学习 K8s 多节点架构、异地测试环境搭建、边缘计算场景。

---

## 1. 架构概览

### 1.1 为什么选 K3s + Docker + Tailscale

| 对比维度 | 原生 K8s (kubeadm) | K3s + Docker | K3s + k3d |
|----------|-------------------|--------------|-----------|
| 安装复杂度 | 高（每节点手动配） | 中（Docker 统一环境） | 低（一行命令） |
| 二进制大小 | ~500MB | ~60MB | ~60MB |
| 多机部署 | 需要公网/VPN | Docker + Tailscale | 不支持（单机） |
| 高可用 etcd | 需外部 etcd 集群 | 内置嵌入式 etcd | 内置嵌入式 etcd |
| 适合场景 | 生产环境 | 异地测试/边缘部署 | 本地学习/CI |

**核心优势**：
- **K3s**：Rancher 出品的轻量 K8s，单二进制 ~60MB，内置 Flannel/etcd/CoreDNS
- **Docker 容器化**：每台机器用 Docker 运行 K3s，环境隔离、易管理、易销毁重建
- **Tailscale**：零配置 WireGuard VPN，节点间自动打洞直连，100.x.x.x 地址段天然适合集群组网

### 1.2 集群拓扑

```
                    ┌─────────────────────────────────┐
                    │       Tailscale 虚拟局域网        │
                    │     100.x.x.x / 100.y.y.y / ... │
                    └─────────────────────────────────┘
                          │           │           │
                 ┌────────┴──┐  ┌─────┴─────┐  ┌─┴────────┐
                 │  Node A    │  │  Node B    │  │  Node C   │
                 │  (Server)  │  │  (Server)  │  │  (Server) │
                 │  Docker    │  │  Docker    │  │  Docker   │
                 │  + Tailscale│ │  + Tailscale│ │ + Tailscale│
                 └─────┬──────┘  └──────┬─────┘  └─────┬─────┘
                       │                │               │
                  ┌────┴────┐     ┌─────┴────┐    ┌────┴────┐
                  │ Agent 1 │     │ Agent 2  │    │ Agent 3 │
                  │ Docker  │     │ Docker   │    │ Docker  │
                  │+Tailscale│    │+Tailscale│    │+Tailscale│
                  └─────────┘     └──────────┘    └─────────┘
```

**说明**：
- 3 台 Server 节点运行嵌入式 etcd，任意 1 台故障集群仍可用（quorum = 2）
- Agent 节点负责运行业务 Pod
- 所有节点通过 Tailscale 互联，K3s Flannel 使用 Tailscale 网络作为底层传输

### 1.3 节点规划示例

| 节点 | 角色 | Docker 主机 IP | Tailscale IP | 用途 |
|------|------|---------------|-------------|------|
| server-1 | Server (leader) | 192.168.1.10 | 100.64.0.1 | 控制面 + etcd |
| server-2 | Server | 192.168.1.11 | 100.64.0.2 | 控制面 + etcd |
| server-3 | Server | 192.168.1.12 | 100.64.0.3 | 控制面 + etcd |
| agent-1 | Agent | 192.168.1.20 | 100.64.0.10 | 工作节点 |
| agent-2 | Agent | 192.168.1.21 | 100.64.0.11 | 工作节点 |

> 如果只有 2 台机器，可以在每台机器上同时跑 1 个 server 容器 + 1 个 agent 容器，第 3 个 server 容器也放在其中一台上。

---

## 2. 前置准备

### 2.1 Docker 安装与配置

**每台机器**都要安装 Docker：

```bash
# Debian/Ubuntu
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
# 重新登录生效

# Arch Linux
sudo pacman -S docker
sudo systemctl enable --now docker
sudo usermod -aG docker $USER
```

**验证**：

```bash
docker version
docker run --rm hello-world
```

### 2.2 Tailscale 安装与组网

```bash
# 安装 Tailscale
curl -fsSL https://tailscale.com/install.sh | sh

# 启动并登录（浏览器授权）
sudo tailscale up

# 查看分配的 Tailscale IP
tailscale status
```

> 完整 Tailscale 使用指南见 [Tailscale 完全指南](../tools/network/tailscale-完全指南.md)。

**验证所有节点互通**：

```bash
# 在 Node A 上 ping Node B 的 Tailscale IP
ping 100.64.0.2

# 确认 Tailscale 直连（无 DERP 中转）
tailscale ping 100.64.0.2
# 输出 "direct connection to 100.64.0.2" 表示直连成功
```

> **重要**：确保所有节点的 Tailscale 都开启了 IP 转发。如果节点间 ping 不通，检查：
> ```bash
> sudo sysctl net.ipv4.ip_forward
> # 应该输出 1，如果不是：
> sudo sysctl -w net.ipv4.ip_forward=1
> ```

### 2.3 防火墙放行端口

K3s 需要以下端口（Tailscale 网络内）：

| 端口 | 协议 | 用途 |
|------|------|------|
| 6443 | TCP | Kubernetes API Server |
| 8472 | UDP | Flannel VXLAN（如果使用 VXLAN 后端） |
| 2379-2380 | TCP | 嵌入式 etcd（仅 server 节点间） |
| 10250 | TCP | Kubelet API |
| 51820/51821 | UDP | WireGuard（如果使用 wireguard-native 后端） |

```bash
# UFW（Ubuntu/Debian）
sudo ufw allow 6443/tcp
sudo ufw allow 8472/udp
sudo ufw allow 2379:2380/tcp
sudo ufw allow 10250/tcp
sudo ufw allow 51820:51821/udp

# firewalld（CentOS/RHEL）
sudo firewall-cmd --permanent --add-port=6443/tcp
sudo firewall-cmd --permanent --add-port=8472/udp
sudo firewall-cmd --permanent --add-port=2379-2380/tcp
sudo firewall-cmd --permanent --add-port=10250/tcp
sudo firewall-cmd --reload
```

> **注意**：如果使用 Tailscale 作为唯一网络通道（所有节点间只走 Tailscale），且 K3s Flannel 使用 `host-gw` 后端，则可以不开放 8472/UDP 和 51820/51821/UDP，因为流量已经由 Tailscale 的 WireGuard 隧道处理。

---

## 3. 方案 A：k3d 单机模拟多节点集群

> **适合场景**：学习多节点架构、本地测试、CI 环境。单台机器上用 Docker 模拟完整的 3 server + N agent 集群。

### 3.1 安装 k3d

```bash
curl -s https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | bash
```

### 3.2 创建集群

```bash
# 创建 3 个 server + 2 个 agent 的集群
k3d cluster create my-cluster \
  --servers 3 \
  --agents 2 \
  --wait
```

### 3.3 验证集群

```bash
# 查看节点
kubectl get nodes -o wide

# 输出示例：
# NAME                       STATUS   ROLES                       AGE   VERSION        INTERNAL-IP
# k3d-my-cluster-server-0    Ready    control-plane,master,etcd   30s   v1.31.x+k3s1   172.18.0.3
# k3d-my-cluster-server-1    Ready    control-plane,master,etcd   30s   v1.31.x+k3s1   172.18.0.4
# k3d-my-cluster-server-2    Ready    control-plane,master,etcd   30s   v1.31.x+k3s1   172.18.0.5
# k3d-my-cluster-agent-0     Ready    <none>                      30s   v1.31.x+k3s1   172.18.0.6
# k3d-my-cluster-agent-1     Ready    <none>                      30s   v1.31.x+k3s1   172.18.0.7

# 查看系统 Pod
kubectl get pods -A
```

### 3.4 销毁集群

```bash
k3d cluster delete my-cluster
```

> k3d 方案到此为止。下面的方案 B 是本文重点：跨机器的真实多节点集群。

---

## 4. 方案 B：多台 Docker 主机 + Tailscale（生产级）

> **适合场景**：异地测试环境、边缘计算、学习多机部署运维。

### 4.1 整体流程

```
每台 Docker 主机执行：
1. 安装 Docker + Tailscale（第 2 章已完成）
2. 拉取 K3s Docker 镜像
3. 创建 Docker 网络
4. 启动 K3s 容器（server 或 agent）
5. 验证集群
```

### 4.2 在每台机器上拉取 K3s 镜像

```bash
docker pull rancher/k3s:latest
```

> K3s 镜像约 60MB，拉取很快。

### 4.3 创建 Docker 网络

在每台机器上创建相同名称的 Docker 网络（用于 K3s 容器间通信）：

```bash
docker network create k3s-net
```

### 4.4 启动第一个 Server 节点（集群初始化）

在 **Node A**（100.64.0.1）上执行：

```bash
docker run -d \
  --name k3s-server-1 \
  --privileged \
  --network k3s-net \
  --hostname k3s-server-1 \
  -v k3s-server-1-data:/var/lib/rancher/k3s \
  -v /dev/net/tun:/dev/net/tun \
  -p 6443:6443 \
  -e K3S_TOKEN=my-secret-token \
  rancher/k3s:latest \
  server \
  --cluster-init \
  --tls-san=100.64.0.1 \
  --tls-san=100.64.0.2 \
  --tls-san=100.64.0.3 \
  --node-ip=100.64.0.1 \
  --flannel-backend=host-gw \
  --disable=traefik
```

**参数说明**：

| 参数 | 作用 |
|------|------|
| `--privileged` | K3s 需要特权模式操作 cgroups/网络 |
| `--cluster-init` | 初始化嵌入式 etcd 集群（3 server HA 必需） |
| `--tls-san` | 额外的 API Server 证书主体（Tailscale IP） |
| `--node-ip` | 告诉 K3s 使用 Tailscale IP 作为节点地址 |
| `--flannel-backend=host-gw` | Flannel 使用 host-gw 模式（Tailscale 已提供 L3 连通性） |
| `--disable=traefik` | 禁用内置 Traefik（可选，按需开启） |
| `-e K3S_TOKEN` | 集群共享密钥，其他节点加入时需要 |
| `-v k3s-server-1-data` | 持久化 K3s 数据（etcd、证书等） |

> **为什么用 `host-gw` 而不是 `vxlan`？**
> Tailscale 已经建立了节点间的三层网络（100.x.x.x 互通），Flannel 不需要再封装一层 VXLAN。`host-gw` 直接路由，性能更好、延迟更低。

### 4.5 获取节点令牌

```bash
# 查看第一个 server 的节点令牌（其他节点加入需要）
docker exec k3s-server-1 cat /var/lib/rancher/k3s/server/node-token
# 输出类似：K10c0f43838d824...::server:6f8c7e...
```

> **记下这个令牌**，后续所有节点加入集群都需要它。

### 4.6 启动第二个 Server 节点

在 **Node B**（100.64.0.2）上执行：

```bash
docker run -d \
  --name k3s-server-2 \
  --privileged \
  --network k3s-net \
  --hostname k3s-server-2 \
  -v k3s-server-2-data:/var/lib/rancher/k3s \
  -v /dev/net/tun:/dev/net/tun \
  -e K3S_TOKEN=my-secret-token \
  rancher/k3s:latest \
  server \
  --server https://100.64.0.1:6443 \
  --tls-san=100.64.0.1 \
  --tls-san=100.64.0.2 \
  --tls-san=100.64.0.3 \
  --node-ip=100.64.0.2 \
  --flannel-backend=host-gw \
  --disable=traefik
```

**关键区别**：没有 `--cluster-init`，而是用 `--server https://100.64.0.1:6443` 加入已有集群。

### 4.7 启动第三个 Server 节点

在 **Node C**（100.64.0.3）上执行：

```bash
docker run -d \
  --name k3s-server-3 \
  --privileged \
  --network k3s-net \
  --hostname k3s-server-3 \
  -v k3s-server-3-data:/var/lib/rancher/k3s \
  -v /dev/net/tun:/dev/net/tun \
  -e K3S_TOKEN=my-secret-token \
  rancher/k3s:latest \
  server \
  --server https://100.64.0.1:6443 \
  --tls-san=100.64.0.1 \
  --tls-san=100.64.0.2 \
  --tls-san=100.64.0.3 \
  --node-ip=100.64.0.3 \
  --flannel-backend=host-gw \
  --disable=traefik
```

> **注意**：`--server` 参数指向任意一个已有的 server 节点即可，不一定非要是 leader。

### 4.8 启动 Agent 节点

在 **Node D**（100.64.0.10）上执行：

```bash
docker run -d \
  --name k3s-agent-1 \
  --privileged \
  --network k3s-net \
  --hostname k3s-agent-1 \
  -v k3s-agent-1-data:/var/lib/rancher/k3s \
  -v /dev/net/tun:/dev/net/tun \
  -e K3S_TOKEN=my-secret-token \
  rancher/k3s:latest \
  agent \
  --server https://100.64.0.1:6443 \
  --node-ip=100.64.0.10 \
  --flannel-backend=host-gw \
  --disable=traefik
```

在 **Node E**（100.64.0.11）上执行：

```bash
docker run -d \
  --name k3s-agent-2 \
  --privileged \
  --network k3s-net \
  --hostname k3s-agent-2 \
  -v k3s-agent-2-data:/var/lib/rancher/k3s \
  -v /dev/net/tun:/dev/net/tun \
  -e K3S_TOKEN=my-secret-token \
  rancher/k3s:latest \
  agent \
  --server https://100.64.0.1:6443 \
  --node-ip=100.64.0.11 \
  --flannel-backend=host-gw \
  --disable=traefik
```

### 4.9 使用 KUBECONFIG 远程管理集群

在任意一台机器上（推荐你的本地开发机）配置 kubectl 远程访问：

```bash
# 从第一个 server 节点拷贝 kubeconfig
docker cp k3s-server-1:/etc/rancher/k3s/k3s.yaml ./k3s.yaml

# 编辑 kubeconfig，把 server 地址改成 Tailscale IP
# 把 127.0.0.1:6443 替换成 100.64.0.1:6443
sed -i 's/127.0.0.1:6443/100.64.0.1:6443/g' ./k3s.yaml

# 放到 kubectl 默认位置
mkdir -p ~/.kube
cp k3s.yaml ~/.kube/config
chmod 600 ~/.kube/config

# 验证
kubectl get nodes -o wide
```

**预期输出**：

```
NAME           STATUS   ROLES                       AGE     VERSION        INTERNAL-IP
k3s-server-1   Ready    control-plane,master,etcd   5m      v1.31.x+k3s1   100.64.0.1
k3s-server-2   Ready    control-plane,master,etcd   4m      v1.31.x+k3s1   100.64.0.2
k3s-server-3   Ready    control-plane,master,etcd   3m      v1.31.x+k3s1   100.64.0.3
k3s-agent-1    Ready    <none>                      2m      v1.31.x+k3s1   100.64.0.10
k3s-agent-2    Ready    <none>                      1m      v1.31.x+k3s1   100.64.0.11
```

> **INTERNAL-IP 应该全部显示 Tailscale IP**（100.x.x.x），如果显示 Docker 内部 IP（172.x.x.x），说明 `--node-ip` 参数没生效，检查 K3s 容器日志。

---

## 5. Tailscale 网络深入

### 5.1 流量路径分析

```
Pod A (Node A) → Pod B (Node B)
     │                    │
     └── K3s Flannel ────┘
          (host-gw 直接路由)
               │
        Tailscale WireGuard 隧道
               │
     100.64.0.1 ←→ 100.64.0.2
```

**数据流向**：
1. Pod A 发送数据包到 Pod B 的 IP
2. K3s Flannel（host-gw 模式）查路由表，发现目标在 100.64.0.2
3. 直接通过 Tailscale 的 WireGuard 隧道发给 Node B
4. Node B 的 Flannel 收到后转给 Pod B

### 5.2 查看 K3s Flannel 路由

```bash
# 在任意 server 节点上
docker exec k3s-server-1 ip route
# 应该看到类似：
# 10.42.1.0/24 via 100.64.0.2 dev tailscale0    # agent-1 的 Pod 网段
# 10.42.2.0/24 via 100.64.0.3 dev tailscale0    # agent-2 的 Pod 网段
# 100.64.0.0/24 dev tailscale0 proto kernel ...
```

> 如果路由表中出现 `via 100.64.0.x dev eth0`（Docker 网桥）而不是 `dev tailscale0`，说明 K3s 没有正确使用 Tailscale 网络，需要检查 `--node-ip` 配置。

### 5.3 跨主机 Service 访问

```bash
# 在 Node A 上部署一个 nginx
kubectl create deployment nginx --image=nginx --replicas=3
kubectl expose deployment nginx --port=80 --type=NodePort

# 查看分配的 NodePort
kubectl get svc nginx
# NAME    TYPE       CLUSTER-IP    EXTERNAL-IP   PORT(S)        AGE
# nginx   NodePort   10.43.x.x    <none>        80:3xxxx/TCP   10s

# 从任意 Tailscale 节点访问
curl http://100.64.0.1:3xxxx
curl http://100.64.0.10:3xxxx
# 两个都能返回 nginx 欢迎页，说明跨节点 Service 路由正常
```

### 5.4 Tailscale ACL 建议

如果需要精细化控制节点间访问，可以在 Tailscale 管理后台（https://login.tailscale.com/admin/acls）配置 ACL：

```json
{
  "acls": [
    {
      // 允许所有 tailnet 内设备互通（适合测试环境）
      "action": "accept",
      "src": ["autogroup:member"],
      "dst": ["autogroup:member:*"]
    }
  ]
}
```

> 生产环境建议限制只有 K3s 节点才能访问 6443 端口，避免未授权访问 API Server。

---

## 6. 集群验证

### 6.1 基础验证

```bash
# 节点状态
kubectl get nodes -o wide

# 系统 Pod 是否全部 Running
kubectl get pods -A

# 组件健康状态
kubectl get componentstatuses
# 或
kubectl get --raw='/readyz?verbose'
```

### 6.2 部署测试应用

```bash
# 创建一个测试 Deployment（5 副本，分布在不同节点）
kubectl create deployment test-app \
  --image=nginx:alpine \
  --replicas=5

# 等待 Pod 就绪
kubectl rollout status deployment test-app

# 查看 Pod 分布
kubectl get pods -o wide -l app=test-app
# 应该看到 Pod 分布在不同的 agent 节点上

# 创建 Service
kubectl expose deployment test-app --port=80 --type=NodePort

# 从 Tailscale 网络访问
NODE_PORT=$(kubectl get svc test-app -o jsonpath='{.spec.ports[0].nodePort}')
curl http://100.64.0.10:$NODE_PORT  # 应返回 nginx 欢迎页
```

### 6.3 高可用测试

```bash
# 模拟 server-1 故障（停止容器）
docker stop k3s-server-1

# 验证集群仍可用
kubectl get nodes -o wide
# server-1 应显示 NotReady，但集群仍正常工作

# 验证 etcd quorum（2/3 存活即可）
# 在 server-2 或 server-3 上执行
docker exec k3s-server-2 kubectl get nodes

# 部署新应用（验证写操作仍可用）
kubectl create deployment ha-test --image=nginx:alpine
kubectl rollout status deployment ha-test

# 恢复 server-1
docker start k3s-server-1

# 等待 server-1 重新加入
kubectl get nodes -o wide
# server-1 应恢复为 Ready 状态
```

### 6.4 清理测试资源

```bash
kubectl delete deployment test-app ha-test
kubectl delete svc test-app
```

---

## 7. 日常运维

### 7.1 添加新 Agent 节点

在新机器上安装 Docker + Tailscale 后：

```bash
# 获取 token（从任意 server 节点）
docker exec k3s-server-1 cat /var/lib/rancher/k3s/server/node-token

# 启动 agent
docker run -d \
  --name k3s-agent-3 \
  --privileged \
  --network k3s-net \
  --hostname k3s-agent-3 \
  -v k3s-agent-3-data:/var/lib/rancher/k3s \
  -v /dev/net/tun:/dev/net/tun \
  -e K3S_TOKEN=<token> \
  rancher/k3s:latest \
  agent \
  --server https://100.64.0.1:6443 \
  --node-ip=<新节点Tailscale-IP> \
  --flannel-backend=host-gw \
  --disable=traefik
```

### 7.2 移除节点

```bash
# 标记节点为不可调度
kubectl drain k3s-agent-2 --ignore-daemonsets --delete-emptydir-data

# 从集群中删除
kubectl delete node k3s-agent-2

# 在对应机器上停止并删除容器
docker stop k3s-agent-2
docker rm k3s-agent-2
docker volume rm k3s-agent-2-data
```

### 7.3 K3s 升级

```bash
# 1. 先升级第一个 server
docker stop k3s-server-1
docker rm k3s-server-1
# 保留 volume 数据不删除

# 2. 重新启动（使用新版本镜像）
docker pull rancher/k3s:latest
docker run -d \
  --name k3s-server-1 \
  --privileged \
  --network k3s-net \
  --hostname k3s-server-1 \
  -v k3s-server-1-data:/var/lib/rancher/k3s \
  -v /dev/net/tun:/dev/net/tun \
  -p 6443:6443 \
  -e K3S_TOKEN=my-secret-token \
  rancher/k3s:latest \
  server \
  --cluster-init \
  --tls-san=100.64.0.1 \
  --tls-san=100.64.0.2 \
  --tls-san=100.64.0.3 \
  --node-ip=100.64.0.1 \
  --flannel-backend=host-gw \
  --disable=traefik

# 3. 等待 server-1 Ready 后，依次升级其他 server
# 4. 最后升级 agent 节点
```

> **升级顺序**：server 节点逐个升级 → agent 节点逐个升级。每次只升级一个节点，等它 Ready 后再升级下一个。

### 7.4 etcd 备份与恢复

```bash
# 创建备份快照
docker exec k3s-server-1 k3s etcd-snapshot save --name backup-$(date +%Y%m%d)

# 查看快照列表
docker exec k3s-server-1 k3s etcd-snapshot list

# 导出备份文件
docker cp k3s-server-1:/var/lib/rancher/k3s/server/db/snapshots/backup-*.db ./k3s-backup.db

# 恢复（需要停止所有 K3s 容器后执行）
docker stop k3s-server-1 k3s-server-2 k3s-server-3 k3s-agent-1 k3s-agent-2
docker exec k3s-server-1 k3s server \
  --cluster-init \
  --etcd-restore /var/lib/rancher/k3s/server/db/snapshots/backup-*.db
```

### 7.5 查看日志

```bash
# server 节点日志
docker logs k3s-server-1
docker logs -f k3s-server-1   # 实时跟踪

# agent 节点日志
docker logs k3s-agent-1
```

---

## 8. 常见问题

### Q: 节点间 Tailscale IP 可以 ping 通，但 K3s 节点状态一直是 NotReady？

**先检查 K3s 日志**。最常见原因是 `--node-ip` 没有正确设置，K3s 默认使用 Docker 网桥 IP（172.x.x.x）而非 Tailscale IP。在 K3s 容器内确认：

```bash
docker exec k3s-server-1 kubectl get nodes -o wide
# INTERNAL-IP 列应显示 100.64.0.x
```

如果不是，删除容器后重新启动并确保 `--node-ip` 参数正确。

### Q: 嵌入式 etcd 初始化失败，报 "no etcd candidates"？

**确保第一个 server 节点使用了 `--cluster-init` 参数**。只有第一个 server 需要这个参数，后续 server 用 `--server` 加入即可。同时确保 K3s 版本 >= v1.19.1（嵌入式 etcd 从该版本开始稳定）。

### Q: Agent 节点加入时报 "certificateAuthorities data is complete, but content doesn't match"？

**token 不一致**。确保所有节点使用完全相同的 `K3S_TOKEN`。token 包含 CA 证书哈希，哪怕差一个字符都会失败。

### Q: Flannel host-gw 模式下跨节点 Pod 通信不通？

**检查 Tailscale 的 IP 转发是否开启**。在每台机器上：

```bash
# 检查
sysctl net.ipv4.ip_forward
# 如果是 0：
sudo sysctl -w net.ipv4.ip_forward=1

# 永久生效
echo "net.ipv4.ip_forward = 1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

同时检查 Tailscale 是否启用了子网路由：

```bash
tailscale status
# 确认没有 "subnet routes not enabled" 的警告
```

### Q: kubeconfig 中的 server 地址是 127.0.0.1，远程无法访问？

**需要修改 kubeconfig 中的 server 地址**。将 `127.0.0.1:6443` 替换为任意一个 server 节点的 Tailscale IP：

```bash
sed -i 's/127.0.0.1:6443/100.64.0.1:6443/g' ~/.kube/config
```

### Q: 集群只有 2 台机器，怎么凑 3 个 server？

在其中一台性能较好的机器上运行 2 个 server 容器（不同端口）：

```bash
# 机器 A：运行 server-1 + server-2
docker run -d --name k3s-server-1 ... -p 6443:6443 ...
docker run -d --name k3s-server-2 ... --server https://100.64.0.1:6443 ...

# 机器 B：运行 server-3 + agent-1
docker run -d --name k3s-server-3 ... --server https://100.64.0.1:6443 ...
docker run -d --name k3s-agent-1 ... --server https://100.64.0.1:6443 ...
```

> 2 台机器的 3 server 方案在生产中不推荐（无法实现真正的物理隔离容灾），但用于学习和测试完全够用。

### Q: 如何从外部访问集群的 Service/Ingress？

有两种方式：

1. **Tailscale 直接访问**：在你自己的设备上安装 Tailscale，加入同一个 tailnet，然后直接用 NodePort 或 ClusterIP + 端口转发访问
2. **Tailscale Funnel**：将 Service 暴露到公网（需要 Tailscale Funnel 功能）

```bash
# 方式 1：本地 kubectl port-forward
kubectl port-forward svc/test-app 8080:80

# 方式 2：Tailscale Exit Node（让本地设备借用集群的网络出口）
# 在集群节点上：
sudo tailscale up --advertise-exit-node

# 在本地设备上：
sudo tailscale up --exit-node=100.64.0.1
```

---

## 9. 参考

- [K3s 官方文档](https://docs.k3s.io/)
- [K3s HA 安装指南](https://docs.k3s.io/installation/ha)
- [K3s 嵌入式 etcd](https://docs.k3s.io/datastore)
- [k3d 官方文档](https://k3d.io/)
- [K3s Docker 镜像](https://hub.docker.com/r/rancher/k3s)
- [Tailscale 完全指南](../tools/network/tailscale-完全指南.md)（本仓库）
- [Kubernetes 开发者学习手册](README.md)（本仓库）
