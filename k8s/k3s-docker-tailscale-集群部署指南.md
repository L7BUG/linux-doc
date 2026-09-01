# K3s + Docker + Tailscale 多节点集群部署指南

> **背景**：用 4 台机器组建 K3s 集群——2 台个人开发机（公司+家里）通过 Docker 容器运行 K3s，2 台云服务器原生安装 K3s，所有节点通过 Tailscale 虚拟局域网互联。开发机的 K3s 完全运行在 Docker 内，不污染宿主机环境。

---

## 1. 架构概览

### 1.1 集群拓扑

```
┌───────────────────────────────────────────────────────────────┐
│                     Tailscale 虚拟局域网                       │
│                  100.x.x.x 全节点互通                          │
└───────────────────────────────────────────────────────────────┘
       │                              │
┌──────┴──────────────┐    ┌──────────┴────────────────┐
│   开发机 A（公司）     │    │   开发机 B（家里）           │
│   6C/12T + 32GB     │    │   6C/12T + 32GB           │
│                     │    │                            │
│   Docker 容器        │    │   Docker 容器               │
│   ┌───────────────┐ │    │   ┌──────────────────────┐ │
│   │ K3s Server    │ │    │   │ K3s Server           │ │
│   │ + K3s Agent   │ │    │   │ + K3s Agent          │ │
│   └───────────────┘ │    │   └──────────────────────┘ │
└─────────────────────┘    └────────────────────────────┘
       │                              │
       └──────────────┬───────────────┘
                      │
       ┌──────────────┴───────────────────────┐
       │                                      │
┌──────┴──────────────┐    ┌──────────────────┴──────┐
│   云服务器 D（8C/8G）  │    │   云服务器 C（2C/2G）     │
│                     │    │                         │
│   原生 K3s           │    │   原生 K3s               │
│   ┌───────────────┐ │    │   ┌───────────────────┐ │
│   │ K3s Agent     │ │    │   │ K3s Agent         │ │
│   │ (主力工作节点)  │ │    │   │ (轻量工作节点)      │ │
│   └───────────────┘ │    │   └───────────────────┘ │
└─────────────────────┘    └─────────────────────────┘
```

### 1.2 角色分配

| 机器 | 配置 | 角色 | 部署方式 | 说明 |
|------|------|------|---------|------|
| 开发机 A（公司） | 6C/12T + 32GB | **Server + Agent** | Docker 容器 | 控制面 + 业务 Pod，不污染宿主机 |
| 开发机 B（家里） | 6C/12T + 32GB | **Server + Agent** | Docker 容器 | 控制面 + 业务 Pod，不污染宿主机 |
| 云服务器 D | 8C/8GB | **Agent** | 原生 K3s | 主力工作节点，7×24 在线 |
| 云服务器 C | 2C/2GB | **Agent** | 原生 K3s | 轻量工作节点 |

### 1.3 为什么是 2 server 而不是 3

K3s 高可用需要奇数个 server 节点（嵌入式 etcd quorum）。理论上 3 server 最优，但你的两台开发机是**个人机器，非 7×24 在线**：

| 方案 | 开发机全开时 | 开发机全关时（夜间/周末） |
|------|-----------|---------------------|
| 3 server（2开发机+1云） | ✅ 3/3 quorum | ❌ 1/3 quorum 丢失，集群挂 |
| **2 server（2开发机）** | ✅ 2/2 quorum | ❌ 0/2 集群不可用 |
| 3 server（3台云） | ✅ 正常 | ✅ 正常，但需要额外买云服务器 |

**当前选择 2 server**：够用。开发机在线时集群完整可用，关机时集群暂停（数据不丢，开机自动恢复）。将来需要 7×24 高可用时，加一台便宜云服务器（2C/4G 足够）专门跑第 3 个 server 即可。

### 1.4 资源规划

| 机器 | K3s Server 占用 | K3s Agent 占用 | 剩余可用 |
|------|----------------|---------------|---------|
| 开发机 A（32GB） | ~500MB | ~200MB | ~31GB（给业务 Pod + 宿主机） |
| 开发机 B（32GB） | ~500MB | ~200MB | ~31GB |
| 云服务器 D（8GB） | — | ~500MB | ~7.5GB |
| 云服务器 C（2GB） | — | ~500MB | ~1.5GB |

> K3s Server 组件（API Server + etcd + Controller Manager + Scheduler）在嵌入式 etcd 模式下约占 500MB~1GB 内存。K3s Agent（kubelet + Container Runtime）约 200~500MB。

---

## 2. 前置准备

### 2.1 Docker 安装（仅开发机）

开发机需要 Docker 来运行 K3s 容器：

```bash
# Debian/Ubuntu
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
# 重新登录生效

# 验证
docker version
```

> 云服务器不需要 Docker，K3s 原生安装。

### 2.2 Tailscale 安装（所有机器）

```bash
# 所有 4 台机器都执行
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
# 浏览器弹出授权页面，用 GitHub/Google/Microsoft 账号登录
```

**验证所有节点互通**：

```bash
# 在任意节点上 ping 其他节点
tailscale ping <其他节点Tailscale-IP>
# 输出 "direct connection to 100.x.x.x" 表示直连成功
```

> 完整 Tailscale 使用指南见 [Tailscale 完全指南](../tools/network/tailscale-完全指南.md)。

### 2.3 防火墙放行端口

K3s 需要以下端口（Tailscale 网络内）：

| 端口 | 协议 | 用途 | 哪些节点需要 |
|------|------|------|------------|
| 6443 | TCP | Kubernetes API Server | Server 节点 |
| 8472 | UDP | Flannel VXLAN（如果用 VXLAN 后端） | 所有节点 |
| 2379-2380 | TCP | 嵌入式 etcd | Server 节点间 |
| 10250 | TCP | Kubelet API | 所有节点 |

```bash
# Debian/Ubuntu（ufw）
sudo ufw allow 6443/tcp
sudo ufw allow 8472/udp
sudo ufw allow 2379:2380/tcp
sudo ufw allow 10250/tcp
```

> 如果所有节点都在 Tailscale 网络内，且 Flannel 使用 `host-gw` 后端，可以不开放 8472/UDP（流量走 Tailscale WireGuard 隧道）。

### 2.4 确认系统环境

| 机器 | 操作系统 | Docker | Tailscale |
|------|---------|--------|-----------|
| 开发机 A | （你的系统） | ✅ 已装 | ✅ 已装 |
| 开发机 B | （你的系统） | ✅ 已装 | ✅ 已装 |
| 云服务器 D | Debian | ❌ 不需要 | ✅ 已装 |
| 云服务器 C | Debian | ❌ 不需要 | ✅ 已装 |

---

## 3. Phase 1：初始化第一个 Server（开发机 A）

### 3.1 拉取 K3s Docker 镜像

```bash
docker pull rancher/k3s:latest
```

### 3.2 创建 Docker 网络

```bash
docker network create k3s-net
```

### 3.3 启动第一个 Server（集群初始化）

```bash
docker run -d \
  --name k3s-server \
  --privileged \
  --network k3s-net \
  --hostname k3s-server \
  --restart unless-stopped \
  -v k3s-server-data:/var/lib/rancher/k3s \
  -v /dev/net/tun:/dev/net/tun \
  -p 6443:6443 \
  -e K3S_TOKEN=k3s-cluster-token-2026 \
  rancher/k3s:latest \
  server \
  --cluster-init \
  --tls-san=<开发机A-Tailscale-IP> \
  --tls-san=<开发机B-Tailscale-IP> \
  --node-ip=<开发机A-Tailscale-IP> \
  --flannel-backend=host-gw \
  --disable=traefik
```

> **替换占位符**：`<开发机A-Tailscale-IP>` 换成实际的 100.x.x.x 地址。

**参数说明**：

| 参数 | 作用 |
|------|------|
| `--privileged` | K3s 需要特权模式操作 cgroups/网络 |
| `--cluster-init` | 初始化嵌入式 etcd 集群 |
| `--tls-san` | API Server 证书主体（允许通过 Tailscale IP 访问） |
| `--node-ip` | 告诉 K3s 使用 Tailscale IP 作为节点地址 |
| `--flannel-backend=host-gw` | Flannel 直接路由（Tailscale 已提供 L3 连通性） |
| `--disable=traefik` | 禁用内置 Traefik（按需开启） |
| `-e K3S_TOKEN` | 集群共享密钥，所有节点必须一致 |
| `--restart unless-stopped` | Docker 重启后自动恢复 |
| `-v k3s-server-data` | 持久化 etcd 数据、证书、config |

### 3.4 等待 Server 就绪

```bash
# 查看容器日志，等待 "Wrote kubeconfig" 出现
docker logs -f k3s-server
# 看到类似下面的输出就说明 server 启动成功：
# Wrote control plane kubeconfig to /etc/rancher/k3s/k3s.yaml
```

### 3.5 获取节点令牌

```bash
docker exec k3s-server cat /var/lib/rancher/k3s/server/node-token
# 输出类似：K10c0f43838d824...::server:6f8c7e...
```

> **记下这个令牌**，后续所有节点加入集群都需要它。

### 3.6 在开发机 A 上配置 kubectl

```bash
# 拷贝 kubeconfig
docker cp k3s-server:/etc/rancher/k3s/k3s.yaml ~/.kube/config

# 修改 server 地址（从 localhost 改成 Tailscale IP）
sed -i 's/127.0.0.1:6443/<开发机A-Tailscale-IP>:6443/g' ~/.kube/config

# 设置权限
chmod 600 ~/.kube/config

# 验证
kubectl get nodes
# 应该看到 k3s-server 状态为 Ready
```

---

## 4. Phase 2：第二个 Server 加入（开发机 B）

### 4.1 拉取镜像 + 创建网络

```bash
# 在开发机 B 上执行
docker pull rancher/k3s:latest
docker network create k3s-net
```

### 4.2 启动第二个 Server

```bash
docker run -d \
  --name k3s-server \
  --privileged \
  --network k3s-net \
  --hostname k3s-server \
  --restart unless-stopped \
  -v k3s-server-data:/var/lib/rancher/k3s \
  -v /dev/net/tun:/dev/net/tun \
  -e K3S_TOKEN=k3s-cluster-token-2026 \
  rancher/k3s:latest \
  server \
  --server https://<开发机A-Tailscale-IP>:6443 \
  --tls-san=<开发机A-Tailscale-IP> \
  --tls-san=<开发机B-Tailscale-IP> \
  --node-ip=<开发机B-Tailscale-IP> \
  --flannel-backend=host-gw \
  --disable=traefik
```

> **关键区别**：没有 `--cluster-init`，用 `--server` 指向开发机 A 加入已有集群。

### 4.3 等待就绪

```bash
docker logs -f k3s-server
# 等待 "Node synced" 或日志不再报连接错误
```

---

## 5. Phase 3：开发机 Agent 加入

### 5.1 开发机 A 的 Agent

```bash
docker run -d \
  --name k3s-agent \
  --privileged \
  --network k3s-net \
  --hostname k3s-agent \
  --restart unless-stopped \
  -v k3s-agent-data:/var/lib/rancher/k3s \
  -v /dev/net/tun:/dev/net/tun \
  -e K3S_TOKEN=k3s-cluster-token-2026 \
  rancher/k3s:latest \
  agent \
  --server https://<开发机A-Tailscale-IP>:6443 \
  --node-ip=<开发机A-Tailscale-IP> \
  --flannel-backend=host-gw \
  --disable=traefik
```

### 5.2 开发机 B 的 Agent

```bash
docker run -d \
  --name k3s-agent \
  --privileged \
  --network k3s-net \
  --hostname k3s-agent \
  --restart unless-stopped \
  -v k3s-agent-data:/var/lib/rancher/k3s \
  -v /dev/net/tun:/dev/net/tun \
  -e K3S_TOKEN=k3s-cluster-token-2026 \
  rancher/k3s:latest \
  agent \
  --server https://<开发机A-Tailscale-IP>:6443 \
  --node-ip=<开发机B-Tailscale-IP> \
  --flannel-backend=host-gw \
  --disable=traefik
```

### 5.3 验证 4 个节点

```bash
# 在开发机 A 上执行
kubectl get nodes -o wide
```

**预期输出**（此时云服务器还没加入，应该看到 4 个节点）：

```
NAME                  STATUS   ROLES                       AGE   VERSION        INTERNAL-IP
k3s-server            Ready    control-plane,master,etcd   10m   v1.31.x+k3s1   100.64.0.1
k3s-server            Ready    control-plane,master,etcd   5m    v1.31.x+k3s1   100.64.0.2
k3s-agent             Ready    <none>                      2m    v1.31.x+k3s1   100.64.0.1
k3s-agent             Ready    <none>                      1m    v1.31.x+k3s1   100.64.0.2
```

> 注意：两个 server 和两个 agent 的 hostname 都叫 `k3s-server` / `k3s-agent`，K3s 用 node name 区分。如果需要区分，可以在 Docker run 时用 `--hostname` 设置不同名称。

---

## 6. Phase 4：云服务器加入集群

### 6.1 云服务器 D（8C/8G）— 主力 Agent

```bash
# 在云服务器 D 上执行（Debian 原生安装）
curl -sfL https://get.k3s.io | K3S_URL=https://<开发机A-Tailscale-IP>:6443 \
  K3S_TOKEN=<node-token> \
  sh -s - agent \
  --node-ip=<云服务器D-Tailscale-IP> \
  --flannel-backend=host-gw \
  --disable=traefik
```

> `<node-token>` 是 Phase 1 中获取的节点令牌。

### 6.2 云服务器 C（2C/2G）— 轻量 Agent

```bash
# 在云服务器 C 上执行
curl -sfL https://get.k3s.io | K3S_URL=https://<开发机A-Tailscale-IP>:6443 \
  K3S_TOKEN=<node-token> \
  sh -s - agent \
  --node-ip=<云服务器C-Tailscale-IP> \
  --flannel-backend=host-gw \
  --disable=traefik
```

### 6.3 云服务器卸载 K3s（如果需要）

```bash
# 原生安装的 K3s 卸载
/usr/local/bin/k3s-uninstall.sh
```

---

## 7. 集群验证

### 7.1 确认所有节点 Ready

```bash
kubectl get nodes -o wide
```

**预期输出**：

```
NAME                  STATUS   ROLES                       AGE     VERSION        INTERNAL-IP
k3s-server            Ready    control-plane,master,etcd   30m     v1.31.x+k3s1   100.64.0.1
k3s-server            Ready    control-plane,master,etcd   25m     v1.31.x+k3s1   100.64.0.2
k3s-agent             Ready    <none>                      20m     v1.31.x+k3s1   100.64.0.1
k3s-agent             Ready    <none>                      15m     v1.31.x+k3s1   100.64.0.2
k3s-agent-d           Ready    <none>                      10m     v1.31.x+k3s1   100.64.0.10
k3s-agent-c           Ready    <none>                      5m      v1.31.x+k3s1   100.64.0.11
```

> **INTERNAL-IP 必须全部是 Tailscale IP**（100.x.x.x）。如果显示 Docker 内部 IP（172.x.x.x），说明 `--node-ip` 没生效。

### 7.2 系统 Pod 检查

```bash
kubectl get pods -A
# 所有 Pod 应该都是 Running 或 Completed
```

### 7.3 部署测试应用

```bash
# 创建 Deployment（5 副本，分布在不同节点）
kubectl create deployment test-nginx --image=nginx:alpine --replicas=5

# 等待就绪
kubectl rollout status deployment test-nginx

# 查看 Pod 分布
kubectl get pods -o wide -l app=test-nginx
# 应该看到 Pod 分布在不同的 agent 节点上

# 创建 Service
kubectl expose deployment test-nginx --port=80 --type=NodePort

# 获取 NodePort
NODE_PORT=$(kubectl get svc test-nginx -o jsonpath='{.spec.ports[0].nodePort}')

# 从 Tailscale 网络访问
curl http://<任意节点Tailscale-IP>:$NODE_PORT
# 应返回 nginx 欢迎页
```

### 7.4 高可用测试

```bash
# 停掉开发机 B 的 K3s（模拟故障）
docker stop k3s-server k3s-agent

# 验证集群仍可用
kubectl get nodes -o wide
# 开发机 B 的节点应显示 NotReady，但集群仍正常工作

# 部署新应用（验证写操作仍可用）
kubectl create deployment ha-test --image=nginx:alpine
kubectl rollout status deployment ha-test

# 恢复
docker start k3s-server k3s-agent
```

### 7.5 清理测试资源

```bash
kubectl delete deployment test-nginx ha-test
kubectl delete svc test-nginx
```

---

## 8. Tailscale 网络集成

### 8.1 流量路径

```
Pod A (开发机A) → Pod B (云服务器D)
     │                    │
     └── K3s Flannel ────┘
          (host-gw 直接路由)
               │
        Tailscale WireGuard 隧道
               │
     100.64.0.1 ←→ 100.64.0.10
```

Flannel 使用 `host-gw` 模式，直接通过 Tailscale 的 WireGuard 隧道路由 Pod 流量，无需额外封装。

### 8.2 查看路由表

```bash
# 在任意节点上
ip route | grep 10.42
# 应该看到类似：
# 10.42.1.0/24 via 100.64.0.2 dev tailscale0
# 10.42.2.0/24 via 100.64.0.10 dev tailscale0
```

### 8.3 远程管理集群

从任意安装了 kubectl 的机器（包括你自己的笔记本），配置 kubeconfig 即可远程管理：

```bash
# 从开发机 A 拷贝 kubeconfig
scp <开发机A-Tailscale-IP>:~/.kube/config ~/.kube/config

# 确保 server 地址是 Tailscale IP（不是 127.0.0.1）
sed -i 's/127.0.0.1:6443/<开发机A-Tailscale-IP>:6443/g' ~/.kube/config

# 验证
kubectl get nodes
```

---

## 9. 日常运维

### 9.1 Docker 容器管理（开发机）

```bash
# 查看状态
docker ps | grep k3s

# 重启
docker restart k3s-server k3s-agent

# 查看日志
docker logs -f k3s-server
docker logs -f k3s-agent

# 停止集群（开发机）
docker stop k3s-server k3s-agent

# 启动集群（开发机）
docker start k3s-server k3s-agent

# 彻底销毁（数据丢失）
docker stop k3s-server k3s-agent
docker rm k3s-server k3s-agent
docker volume rm k3s-server-data k3s-agent-data
```

> `--restart unless-stopped` 参数确保 Docker 重启后 K3s 自动恢复。

### 9.2 云服务器管理

```bash
# 查看 K3s 状态
sudo systemctl status k3s-agent

# 重启
sudo systemctl restart k3s-agent

# 查看日志
sudo journalctl -u k3s-agent -f

# 停止
sudo systemctl stop k3s-agent
```

### 9.3 添加新的 Agent 节点

在新机器上（已安装 Tailscale）：

```bash
# 原生安装
curl -sfL https://get.k3s.io | K3S_URL=https://<开发机A-Tailscale-IP>:6443 \
  K3S_TOKEN=<node-token> \
  sh -s - agent \
  --node-ip=<新节点Tailscale-IP> \
  --flannel-backend=host-gw \
  --disable=traefik
```

### 9.4 移除节点

```bash
# 标记不可调度
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data

# 从集群删除
kubectl delete node <node-name>

# 对应机器上停止服务
# 开发机：docker stop k3s-agent
# 云服务器：sudo systemctl stop k3s-agent && sudo /usr/local/bin/k3s-uninstall.sh
```

### 9.5 etcd 备份

```bash
# 在任意 server 节点上创建快照
# 开发机：
docker exec k3s-server k3s etcd-snapshot save --name backup-$(date +%Y%m%d)

# 查看快照列表
docker exec k3s-server k3s etcd-snapshot list

# 导出备份文件
docker cp k3s-server:/var/lib/rancher/k3s/server/db/snapshots/ ./k3s-backup/
```

### 9.6 K3s 升级

```bash
# 开发机：拉取新版本镜像
docker pull rancher/k3s:latest

# 停止旧容器（保留 volume 数据）
docker stop k3s-server k3s-agent
docker rm k3s-server k3s-agent

# 重新启动（使用相同参数，新镜像会自动生效）
# 复用 Phase 1/3 的 docker run 命令

# 云服务器：重新安装
curl -sfL https://get.k3s.io | K3S_URL=https://<开发机A-Tailscale-IP>:6443 \
  K3S_TOKEN=<node-token> \
  sh -s - agent \
  --node-ip=<云服务器Tailscale-IP> \
  --flannel-backend=host-gw \
  --disable=traefik
```

> **升级顺序**：先升级 server 节点（开发机 A → 开发机 B），等 Ready 后再升级 agent 节点。

---

## 10. 常见问题

### Q: 节点 Tailscale IP 可以 ping 通，但 K3s 节点一直 NotReady？

**最常见原因是 `--node-ip` 没有正确设置**。K3s 默认使用 Docker 网桥 IP（172.x.x.x）而非 Tailscale IP。

```bash
# 检查
kubectl get nodes -o wide
# INTERNAL-IP 列应该显示 100.x.x.x
```

如果不是，删除容器后重新启动，确保 `--node-ip` 参数指向正确的 Tailscale IP。

### Q: 开发机关机后再开机，集群能自动恢复吗？

**能。** Docker 设置了 `--restart unless-stopped`，开机后 Docker 自动启动 K3s 容器，etcd 数据在 volume 中持久化，集群自动恢复。云服务器的 K3s agent 是 systemd 服务，同样开机自启。

### Q: 开发机都关了，云服务器上的 Pod 还能运行吗？

**能运行，但不能调度新 Pod。** 已有的 Pod 继续运行，因为 kubelet 是本地的。但 `kubectl apply` 等需要 API Server 的操作会失败（API Server 在开发机上）。等开发机开机后自动恢复。

### Q: 嵌入式 etcd 初始化失败？

**确保第一个 server 节点使用了 `--cluster-init` 参数**。只有第一个 server 需要这个参数。同时确保 K3s 版本 >= v1.19.1。

### Q: kubeconfig 中的 server 地址是 127.0.0.1？

```bash
sed -i 's/127.0.0.1:6443/<开发机A-Tailscale-IP>:6443/g' ~/.kube/config
```

### Q: Flannel host-gw 下跨节点 Pod 通信不通？

**检查 Tailscale 的 IP 转发是否开启**：

```bash
sysctl net.ipv4.ip_forward
# 如果是 0：
sudo sysctl -w net.ipv4.ip_forward=1
echo "net.ipv4.ip_forward = 1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

### Q: 两个 server 的 hostname 都叫 k3s-server，怎么区分？

K3s 使用 node name（不是 hostname）区分节点。Docker 容器的 hostname 会成为 K3s 的 node name。如果两个 server 都叫 `k3s-server`，会冲突。

**解决方案**：给每个节点设置不同的 `--hostname`：

```bash
# 开发机 A
docker run ... --hostname k3s-server-company ...

# 开发机 B
docker run ... --hostname k3s-server-home ...
```

### Q: 2C/2G 的云服务器跑 K3s agent 够吗？

**勉强够用。** K3s agent 约占 300~500MB 内存。2GB 机器上剩余约 1.5GB，可以运行轻量 Pod（如 nginx、小型 API）。不适合运行 Elasticsearch、Redis 等内存密集型应用。

### Q: 怎么从外网访问集群里的 Service？

1. **Tailscale 直接访问**：你的设备加入同一个 tailnet，直接用 NodePort 访问
2. **kubectl port-forward**：`kubectl port-forward svc/my-app 8080:80`
3. **将来加 Ingress**：安装 Traefik 或 Nginx Ingress Controller，配置域名路由

---

## 11. 参考

- [K3s 官方文档](https://docs.k3s.io/)
- [K3s HA 安装指南](https://docs.k3s.io/installation/ha)
- [K3s 嵌入式 etcd](https://docs.k3s.io/datastore)
- [K3s Docker 镜像](https://hub.docker.com/r/rancher/k3s)
- [k3d 官方文档](https://k3d.io/)（单机多节点模拟工具）
- [Tailscale 完全指南](../tools/network/tailscale-完全指南.md)（本仓库）
- [Kubernetes 开发者学习手册](README.md)（本仓库）
