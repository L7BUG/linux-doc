# K3s + Docker + Tailscale 多节点集群部署指南

> **背景**：用 4 台机器组建 K3s 集群——1 台云服务器做 Server，3 台机器做 Agent，所有节点通过 Tailscale 虚拟局域网互联。开发机通过 Docker 容器运行 K3s Agent，云服务器原生安装。

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
              │  原生 K3s Server           │
              └───────────┬───────────────┘
                          │
        ┌─────────────────┼─────────────────┐
        │                 │                 │
┌───────┴────────┐ ┌─────┴──────────┐ ┌────┴──────────────┐
│ 公司开发机(32GB) │ │ 家里开发机(32GB) │ │ 云服务器 A (2C/2G) │
│ Docker          │ │ Docker          │ │ 原生 K3s           │
│ ┌─────────────┐ │ │ ┌─────────────┐ │ │ ┌──────────────┐  │
│ │ K3s Agent   │ │ │ │ K3s Agent   │ │ │ │ K3s Agent    │  │
│ └─────────────┘ │ │ └─────────────┘ │ │ └──────────────┘  │
└─────────────────┘ └─────────────────┘ └───────────────────┘
```

### 1.2 角色分配

| 机器 | 配置 | 角色 | 部署方式 |
|------|------|------|---------|
| 云服务器 B | 4C/4GB | **Server** | 原生 K3s |
| 公司开发机 | 6C/12T + 32GB | **Agent** | Docker 容器 |
| 家里开发机 | 6C/12T + 32GB | **Agent** | Docker 容器 |
| 云服务器 A | 2C/2GB | **Agent** | 原生 K3s |

---

## 2. 前置准备

### 2.1 Docker 安装（仅开发机）

```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
```

### 2.2 Tailscale 安装（所有机器）

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

### 2.3 防火墙放行端口

```bash
sudo ufw allow 41641/udp   # Tailscale
sudo ufw allow 6443/tcp    # K3s API
sudo ufw allow 8472/udp    # Flannel VXLAN
sudo ufw allow 10250/tcp   # Kubelet
```

> 云厂商安全组也要放行。

### 2.4 配置镜像加速（所有机器）

```bash
sudo mkdir -p /etc/rancher/k3s
sudo tee /etc/rancher/k3s/registries.yaml << 'EOF'
mirrors:
  "docker.io":
    endpoint:
      - "https://docker.1ms.run"
      - "https://docker.xuanyuan.me"
EOF
```

> K3s 用的是 `/etc/rancher/k3s/registries.yaml`，不是 `/etc/containerd/config.toml`。

---

## 3. 安装 Server（云服务器 B）

```bash
curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | \
  INSTALL_K3S_MIRROR=cn sh -s - \
  --tls-san=<sv1的Tailscale-IP> \
  --node-ip=<sv1的Tailscale-IP> \
  --disable-network-policy
```

### 3.1 等待就绪 + 获取令牌

```bash
# 等待启动
sudo k3s kubectl get nodes

# 获取令牌
sudo cat /var/lib/rancher/k3s/server/node-token
```

### 3.2 配置 kubectl

```bash
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $USER:$USER ~/.kube/config
sed -i 's/127.0.0.1:6443/sv1:6443/g' ~/.kube/config
```

---

## 4. 安装 Agent（开发机 Docker）

### 4.1 复制文件

```bash
mkdir -p ~/k3s-cluster && cd ~/k3s-cluster
cp /path/to/docker-compose.yml .
cp /path/to/.env.company .env  # 或 .env.home
```

### 4.2 修改 .env

```bash
K3S_TOKEN=<sv1的token>
K3S_URL=https://<sv1的Tailscale-IP>:6443
NODE_IP=<本机Tailscale-IP>
```

### 4.3 启动

```bash
docker compose up -d
```

---

## 5. 安装 Agent（云服务器 A）

```bash
curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | \
  INSTALL_K3S_MIRROR=cn \
  K3S_URL=https://<sv1的Tailscale-IP>:6443 \
  K3S_TOKEN=<node-token> \
  sh -s - agent \
  --node-ip=<sv2的Tailscale-IP>
```

---

## 6. 验证

```bash
kubectl get nodes -o wide
```

---

## 7. 日常运维

### Docker Compose（开发机）

```bash
cd ~/k3s-cluster
docker compose ps / logs -f / restart / stop / start
docker compose down -v  # 销毁（数据丢失）
```

### 云服务器

```bash
sudo systemctl status k3s        # Server
sudo systemctl status k3s-agent  # Agent
```

### 卸载

```bash
sudo /usr/local/bin/k3s-uninstall.sh      # Server
sudo /usr/local/bin/k3s-agent-uninstall.sh  # Agent
```

---

## 8. 常见问题

### Q: `--tls-san` 有什么用？

API Server 证书只默认包含 localhost 和节点 IP。加 `--tls-san=<Tailscale-IP>` 让证书接受通过 Tailscale IP 访问。**不加就连不上 API Server**。

### Q: Pod 间 DNS 解析不通？

Server 安装时必须加 `--disable-network-policy`，否则 kube-router 会误杀 Pod 流量（CoreDNS 不通、Service 访问不了）。

### Q: 镜像拉取慢？

配置 `/etc/rancher/k3s/registries.yaml`（不是 `/etc/containerd/config.toml`）。

### Q: kubeconfig 地址是 127.0.0.1？

```bash
sed -i 's/127.0.0.1:6443/sv1:6443/g' ~/.kube/config
```

---

## 9. 参考

- [K3s 官方文档](https://docs.k3s.io/)
- [Tailscale 完全指南](../tools/network/tailscale-完全指南.md)（本仓库）
- [Kubernetes 开发者学习手册](README.md)（本仓库）
