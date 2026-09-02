# K3s + Docker + Tailscale 多节点集群部署指南

> **背景**：用 4 台机器组建 K3s 集群——1 台云服务器做 Server（仅控制面），3 台机器做 Agent（跑业务 Pod），所有节点通过 Tailscale 虚拟局域网互联。开发机通过 Docker 容器运行 K3s Agent，云服务器原生安装。

---

## 1. 架构概览

### 1.1 集群拓扑

```
┌────────────────────────────────────────────────────────────────┐
│                      Tailscale 虚拟局域网                       │
│                 100.x.x.x 全节点互通                            │
└────────────────────────────────────────────────────────────────┘
                          │
              ┌───────────┴───────────────┐
              │  云服务器 B (4C/4G)         │
              │  7×24 在线                  │
              │                           │
              │  原生 K3s                  │
              │  ┌──────────────────────┐  │
              │  │ K3s Server (仅控制面) │  │
              │  │ etcd + API + Scheduler│  │
              │  └──────────────────────┘  │
              └───────────┬───────────────┘
                          │
        ┌─────────────────┼─────────────────┐
        │                 │                 │
┌───────┴────────┐ ┌─────┴──────────┐ ┌────┴──────────────┐
│ 公司开发机(32GB) │ │ 家里开发机(32GB) │ │ 云服务器 A (2C/2G) │
│ Docker          │ │ Docker          │ │ 原生 K3s           │
│ ┌─────────────┐ │ │ ┌─────────────┐ │ │ ┌──────────────┐  │
│ │ K3s Agent   │ │ │ │ K3s Agent   │ │ │ │ K3s Agent    │  │
│ │ (跑业务Pod)  │ │ │ │ (跑业务Pod)  │ │ │ │ (轻量工作节点) │  │
│ └─────────────┘ │ │ └─────────────┘ │ │ └──────────────┘  │
└─────────────────┘ └─────────────────┘ └───────────────────┘
```

### 1.2 角色分配

| 机器 | 配置 | 角色 | 部署方式 | 在线情况 |
|------|------|------|---------|---------|
| 云服务器 B | 4C/4GB | **Server（仅控制面）** | 原生 K3s | 7×24 |
| 公司开发机 | 6C/12T + 32GB | **Agent** | Docker 容器 | 很少关机 |
| 家里开发机 | 6C/12T + 32GB | **Agent** | Docker 容器 | 可能关机 |
| 云服务器 A | 2C/2GB | **Agent** | 原生 K3s | 7×24 |

### 1.3 为什么云服务器 B 做 Server

| | sv1（云服务器 B） | 开发机 |
|---|---|---|
| 在线时间 | 7×24 永远在线 | 可能关机 |
| 网络 | 公网 IP，稳定 | 内网，Tailscale 打洞 |
| API Server 可达性 | 永远可达 | 关机后不可达 |
| kubeconfig 指向 | 始终有效 | 关机后失效 |

**sv1 只跑控制面**（`--disable-agent`），不跑业务 Pod，4C/4G 资源专门用于集群管理。

### 1.4 资源规划

| 机器 | K3s 占用 | 剩余可用 |
|------|---------|---------|
| 云服务器 B（4GB） | ~500MB（仅 server） | ~3.5GB |
| 公司开发机（32GB） | ~200MB（agent） | ~31.8GB |
| 家里开发机（32GB） | ~200MB（agent） | ~31.8GB |
| 云服务器 A（2GB） | ~500MB（agent） | ~1.5GB |

---

## 2. 前置准备

### 2.1 Docker 安装（仅开发机）

```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
# 重新登录生效
```

### 2.2 Tailscale 安装（所有机器）

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

**验证互通**：

```bash
tailscale ping <其他节点Tailscale-IP>
# 输出 "direct connection" 表示直连成功
```

### 2.3 防火墙放行端口

#### Tailscale（所有机器）

```
UDP 41641 → WireGuard 隧道（打洞用，仅云服务器需要开放）
```

#### K3s（集群内部）

| 端口 | 协议 | 用途 |
|------|------|------|
| 6443 | TCP | Kubernetes API Server |
| 8472 | UDP | Flannel VXLAN（跨节点 Pod 通信） |
| 2379-2380 | TCP | 嵌入式 etcd |
| 10250 | TCP | Kubelet API |

```bash
sudo ufw allow 41641/udp
sudo ufw allow 6443/tcp
sudo ufw allow 8472/udp
sudo ufw allow 2379:2380/tcp
sudo ufw allow 10250/tcp
```

> 云厂商安全组也要放行这些端口。

### 2.4 配置镜像加速

K3s 使用内置的 containerd，配置文件是 `/etc/rancher/k3s/registries.yaml`。

> **注意**：写到 `/etc/containerd/config.toml` 对 K3s 无效，K3s 不认。

**云服务器（原生安装）**：

```bash
sudo mkdir -p /etc/rancher/k3s
sudo tee /etc/rancher/k3s/registries.yaml << 'EOF'
mirrors:
  "docker.io":
    endpoint:
      - "https://docker.1ms.run"
      - "https://docker.xuanyuan.me"
  "ghcr.io":
    endpoint:
      - "https://ghcr.xuanyuan.me"
  "registry.k8s.io":
    endpoint:
      - "https://registry.k8s.xuanyuan.me"
EOF
sudo systemctl restart k3s
```

**开发机（Docker 环境）**：

docker-compose 目录已包含 `registries.yaml`，挂载到容器内，重启即可生效。

---

## 3. 安装 Server（云服务器 B）

sv1 仅运行控制面，不跑业务 Pod（`--disable-agent`）。

```bash
# 在 sv1 上执行（Debian）
curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | \
  INSTALL_K3S_MIRROR=cn INSTALL_K3S_EXEC="server" sh -s - \
  --cluster-init \
  --tls-san=sv1 --tls-san=pc1 --tls-san=pc2 \
  --node-ip=<sv1的Tailscale-IP> \
  --disable=traefik \
  --disable-agent
```

### 3.1 等待 Server 就绪

```bash
sudo k3s kubectl get nodes
# 看到 sv1 状态为 Ready 说明启动成功
```

### 3.2 获取节点令牌

```bash
sudo cat /var/lib/rancher/k3s/server/node-token
# 记下这个令牌
```

### 3.3 配置 kubectl

```bash
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $USER:$USER ~/.kube/config
sed -i 's/127.0.0.1:6443/sv1:6443/g' ~/.kube/config

# 验证
kubectl get nodes
```

---

## 4. 安装 Agent（开发机）

开发机通过 Docker 运行 K3s Agent。

### 4.1 准备 docker-compose 文件

```bash
mkdir -p ~/k3s-cluster && cd ~/k3s-cluster
cp /path/to/docker-compose.yml .
cp /path/to/.env.company .env  # 或 .env.home
```

### 4.2 修改 .env 配置

```bash
vim .env
```

**必须确认的值**：

```bash
K3S_TOKEN=k3s-cluster-token-2026    # 和 sv1 上的令牌一致！
K3S_AGENT_FLAGS=--server https://<sv1的Tailscale-IP>:6443 --node-ip=<本机Tailscale-IP>
```

### 4.3 启动

```bash
docker compose up -d
docker compose logs -f   # 等待 Ready 后 Ctrl+C
```

### 4.4 验证

```bash
kubectl get nodes
# 应该看到 sv1 和当前开发机都 Ready
```

---

## 5. 安装 Agent（云服务器 A）

```bash
# 在 sv2 上执行（Debian）
curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | \
  INSTALL_K3S_MIRROR=cn \
  K3S_URL=https://<sv1的Tailscale-IP>:6443 \
  K3S_TOKEN=<node-token> \
  sh -s - agent \
  --node-ip=<sv2的Tailscale-IP>
```

---

## 6. 最终验证

```bash
kubectl get nodes -o wide
```

**预期输出**：

```
NAME                    STATUS   ROLES    AGE   VERSION        INTERNAL-IP
sv1                     Ready    <none>   1h    v1.36.x+k3s1   100.x.x.x
agent-company           Ready    <none>   30m   v1.34.x+k3s1   100.x.x.x
agent-home              Ready    <none>   20m   v1.34.x+k3s1   100.x.x.x
agent-cloud-a           Ready    <none>   10m   v1.36.x+k3s1   100.x.x.x
```

---

## 7. 集群验证

### 7.1 部署测试应用

```bash
kubectl create deployment test-nginx --image=nginx:alpine --replicas=5
kubectl rollout status deployment test-nginx
kubectl get pods -o wide -l app=test-nginx

kubectl expose deployment test-nginx --port=80 --type=NodePort
NODE_PORT=$(kubectl get svc test-nginx -o jsonpath='{.spec.ports[0].nodePort}')
curl http://<任意节点Tailscale-IP>:$NODE_PORT
```

### 7.2 清理

```bash
kubectl delete deployment test-nginx
kubectl delete svc test-nginx
```

---

## 8. Tailscale 网络集成

### 8.1 流量路径

```
Pod A (公司) → Pod B (云服务器A)
     │                    │
     └── K3s Flannel ────┘
          (vxlan 封装转发)
               │
        Tailscale WireGuard 隧道
```

Flannel vxlan 模式将 Pod 流量封装在 UDP 包中（端口 8472），通过 Tailscale 隧道传输。

### 8.2 远程管理

```bash
scp sv1:~/.kube/config ~/.kube/config
sed -i 's/127.0.0.1:6443/sv1:6443/g' ~/.kube/config
kubectl get nodes
```

---

## 9. 日常运维

### 9.1 Docker Compose 管理（开发机）

```bash
cd ~/k3s-cluster

docker compose ps                    # 查看状态
docker compose logs -f               # 实时日志
docker compose restart               # 重启
docker compose stop                  # 停止（保留数据）
docker compose start                 # 启动
docker compose down -v               # 销毁（数据丢失！）
```

### 9.2 云服务器管理

```bash
sudo systemctl status k3s            # Server 状态
sudo systemctl status k3s-agent      # Agent 状态
sudo journalctl -u k3s -f            # Server 日志
sudo systemctl restart k3s           # 重启 Server
```

### 9.3 添加新 Agent 节点

```bash
curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | \
  INSTALL_K3S_MIRROR=cn \
  K3S_URL=https://<sv1的Tailscale-IP>:6443 \
  K3S_TOKEN=<node-token> \
  sh -s - agent \
  --node-ip=<新节点Tailscale-IP>
```

### 9.4 移除节点

```bash
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data
kubectl delete node <node-name>
# 对应机器上停止服务并卸载
```

### 9.5 卸载 K3s（云服务器）

```bash
sudo /usr/local/bin/k3s-uninstall.sh     # Server
sudo /usr/local/bin/k3s-agent-uninstall.sh  # Agent
```

---

## 10. 常见问题

### Q: Pod 间 DNS 解析不通？

**Docker 环境必须加 `--disable-network-policy`**。kube-router 在 Docker 容器内无法正确管理 iptables FORWARD 链，默认 DROP 策略会阻断 Pod→Service 流量。开发/测试环境直接禁用即可。

### Q: 跨节点 Pod 通信不通？

**确认所有节点使用默认的 vxlan 后端**。K3s 默认就是 vxlan，不需要显式指定 `--flannel-backend`。如果手动指定了 `host-gw`，跨 Tailscale 网络会不通。

### Q: kubeconfig 地址是 127.0.0.1？

```bash
sed -i 's/127.0.0.1:6443/sv1:6443/g' ~/.kube/config
```

### Q: Agent 节点 containerd 启动失败？

**端口冲突**。如果 server 和 agent 在同一台机器上（Docker 环境），用 `--lb-server-port=6445` 给 agent 换端口。单 server 架构下不会有这个问题。

### Q: 镜像拉取慢或失败？

**配置镜像加速**。写到 `/etc/rancher/k3s/registries.yaml`，不要写到 `/etc/containerd/config.toml`（K3s 不认）。

### Q: K3s server 启动卡住？

**清数据重试**。`docker compose down -v` 删除 volume 后重新启动。云服务器用 `k3s-uninstall.sh` 卸载后重装。

### Q: `--flannel-backend` 和 `--disable` 能用在 agent 上吗？

**不能。** 这两个是 server 参数，agent 不认。K3s 默认使用 vxlan，不需要显式指定。

### Q: 开发机关机后集群能用吗？

**能。** sv1（7×24 在线）仍然运行 API Server，kubectl 仍然可用。只是无法调度新 Pod 到关机的开发机上，已有的 Pod 会一直运行直到手动删除。

---

## 11. 参考

- [K3s 官方文档](https://docs.k3s.io/)
- [K3s HA 安装指南](https://docs.k3s.io/installation/ha)
- [K3s 嵌入式 etcd](https://docs.k3s.io/datastore)
- [K3s Docker 镜像](https://hub.docker.com/r/rancher/k3s)
- [Tailscale 完全指南](../tools/network/tailscale-完全指南.md)（本仓库）
- [Kubernetes 开发者学习手册](README.md)（本仓库）
