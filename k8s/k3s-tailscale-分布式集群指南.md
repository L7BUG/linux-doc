# K3s + Tailscale 分布式集群部署指南

> **背景**：基于 K3s 官方文档的分布式/多云部署方案，使用 Tailscale VPN 集成，让跨网络的节点组建 Kubernetes 集群。

---

## 1. 架构概览

### 1.1 方案说明

K3s 官方支持两种跨网络部署方案：

| 方案 | 说明 | 限制 |
|------|------|------|
| 内置 WireGuard | K3s 用 WireGuard 建立 VPN mesh | Server 必须能通过内网 IP 互通 |
| **Tailscale 集成（本文）** | K3s 用 Tailscale VPN 服务构建 mesh | 实验性功能，需配置 ACL |

**本文使用 Tailscale 集成方案**，参考官方文档：[Distributed hybrid or multicloud cluster](https://docs.k3s.io/networking/distributed-multicloud)

### 1.2 集群拓扑

```
┌────────────────────────────────────────────────────────────────┐
│                      Tailscale 虚拟局域网                       │
│                   100.x.x.x 全节点互通                          │
└────────────────────────────────────────────────────────────────┘
                          │
              ┌───────────┴───────────────┐
              │  云服务器 B (4C/4G)         │
              │  7×24 在线                  │
              │                           │
              │  K3s Server               │
              │  --vpn-auth=...           │
              └───────────┬───────────────┘
                          │
        ┌─────────────────┼─────────────────┐
        │                 │                 │
┌───────┴────────┐ ┌─────┴──────────┐ ┌────┴──────────────┐
│ 公司开发机(32GB) │ │ 家里开发机(32GB) │ │ 云服务器 A (2C/2G) │
│ K3s Agent      │ │ K3s Agent      │ │ K3s Agent         │
│ --vpn-auth=... │ │ --vpn-auth=... │ │ --vpn-auth=...    │
└─────────────────┘ └─────────────────┘ └───────────────────┘
```

### 1.3 角色分配

| 机器 | 配置 | 角色 | 部署方式 |
|------|------|------|---------|
| 云服务器 B | 4C/4GB | **Server** | 原生 K3s |
| 公司开发机 | 6C/12T + 32GB | **Agent** | 原生 K3s |
| 家里开发机 | 6C/12T + 32GB | **Agent** | 原生 K3s |
| 云服务器 A | 2C/2GB | **Agent** | 原生 K3s |

---

## 2. 前置准备

### 2.1 Tailscale 安装（所有机器）

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

**验证互通**：

```bash
tailscale ping <其他节点Tailscale-IP>
```

### 2.2 Tailscale 后台配置

#### 生成 Auth Key

1. 登录 https://login.tailscale.com/admin/settings/keys
2. 点击 **Generate auth key**
3. 勾选 **Reusable**（可复用）
4. 复制生成的 key（格式：`tskey-auth-...`）

#### 配置 ACL 和路由

在 https://login.tailscale.com/admin/acls 编辑 ACL，添加：

```json
{
  "autoApprovers": {
    "routes": {
      "10.42.0.0/16": ["your-account@email.com"]
    }
  },
  "acls": [
    {
      "action": "accept",
      "src": ["autogroup:member", "10.42.0.0/16"],
      "dst": ["autogroup:member:*", "10.42.0.0/16:*"]
    }
  ]
}
```

> `10.42.0.0/16` 是 K3s 默认的 Pod CIDR。如果修改了 `--cluster-cidr`，需要同步修改这里的路由。

### 2.3 防火墙放行端口

| 端口 | 协议 | 用途 |
|------|------|------|
| 41641 | UDP | Tailscale WireGuard |
| 6443 | TCP | K3s API Server |
| 10250 | TCP | Kubelet |
| 51820 | UDP | Flannel WireGuard（IPv4） |

```bash
sudo ufw allow 41641/udp
sudo ufw allow 6443/tcp
sudo ufw allow 10250/tcp
sudo ufw allow 51820/udp
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

---

## 3. 安装 Server（云服务器 B）

### 3.1 安装 K3s

```bash
curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | \
  INSTALL_K3S_MIRROR=cn sh -s - \
  --vpn-auth="name=tailscale,joinKey=<AUTH_KEY>" \
  --node-external-ip=<sv1的Tailscale-IP>
```

> `<AUTH_KEY>` 替换为 Tailscale 后台生成的 auth key。
> `<sv1的Tailscale-IP>` 替换为实际 IP（`tailscale status` 查看）。

### 3.2 等待 Server 就绪

```bash
sudo k3s kubectl get nodes
# 看到 sv1 状态为 Ready 说明启动成功
```

### 3.3 获取节点令牌

```bash
sudo cat /var/lib/rancher/k3s/server/node-token
# 记下这个令牌
```

### 3.4 配置 kubectl

```bash
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $USER:$USER ~/.kube/config
sed -i 's/127.0.0.1:6443/sv1:6443/g' ~/.kube/config

# 验证
kubectl get nodes
```

### 3.5 在 Tailscale 后台批准节点

执行安装命令后，访问 Tailscale 管理后台：
1. 批准新注册的 Tailscale 节点
2. 批准路由（如果 ACL 没有配置 autoApprovers）

---

## 4. 安装 Agent（其他节点）

### 4.1 公司开发机

```bash
curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | \
  INSTALL_K3S_MIRROR=cn \
  K3S_URL=https://<sv1的Tailscale-IP>:6443 \
  K3S_TOKEN=<node-token> \
  sh -s - agent \
  --vpn-auth="name=tailscale,joinKey=<AUTH_KEY>" \
  --node-external-ip=<pc1的Tailscale-IP>
```

### 4.2 家里开发机

```bash
curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | \
  INSTALL_K3S_MIRROR=cn \
  K3S_URL=https://<sv1的Tailscale-IP>:6443 \
  K3S_TOKEN=<node-token> \
  sh -s - agent \
  --vpn-auth="name=tailscale,joinKey=<AUTH_KEY>" \
  --node-external-ip=<pc2的Tailscale-IP>
```

### 4.3 云服务器 A

```bash
curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | \
  INSTALL_K3S_MIRROR=cn \
  K3S_URL=https://<sv1的Tailscale-IP>:6443 \
  K3S_TOKEN=<node-token> \
  sh -s - agent \
  --vpn-auth="name=tailscale,joinKey=<AUTH_KEY>" \
  --node-external-ip=<sv2的Tailscale-IP>
```

### 4.4 批准 Tailscale 节点

每个 agent 安装后，访问 Tailscale 管理后台批准新节点和路由。

---

## 5. 验证

```bash
kubectl get nodes -o wide
```

**预期输出**：

```
NAME           STATUS   ROLES    AGE   VERSION        INTERNAL-IP
sv1            Ready    <none>   1h    v1.36.x+k3s1   100.x.x.x
agent-company  Ready    <none>   30m   v1.36.x+k3s1   100.x.x.x
agent-home     Ready    <none>   20m   v1.36.x+k3s1   100.x.x.x
agent-cloud    Ready    <none>   10m   v1.36.x+k3s1   100.x.x.x
```

### 5.1 部署测试应用

```bash
kubectl create deployment test-nginx --image=nginx:alpine --replicas=3
kubectl rollout status deployment test-nginx
kubectl expose deployment test-nginx --port=80 --type=NodePort
```

### 5.2 测试 DNS

```bash
kubectl run test --image=busybox --rm -it --restart=Never -- sh -c "nslookup kubernetes.default.svc.cluster.local"
```

---

## 6. 日常运维

### 6.1 K3s 管理

```bash
sudo systemctl status k3s        # Server 状态
sudo systemctl status k3s-agent  # Agent 状态
sudo journalctl -u k3s -f        # Server 日志
sudo journalctl -u k3s-agent -f  # Agent 日志
```

### 6.2 添加新 Agent

```bash
curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | \
  INSTALL_K3S_MIRROR=cn \
  K3S_URL=https://<sv1的Tailscale-IP>:6443 \
  K3S_TOKEN=<node-token> \
  sh -s - agent \
  --vpn-auth="name=tailscale,joinKey=<AUTH_KEY>" \
  --node-external-ip=<新节点Tailscale-IP>
```

### 6.3 移除节点

```bash
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data
kubectl delete node <node-name>

# 在对应机器上卸载
sudo /usr/local/bin/k3s-agent-uninstall.sh
```

### 6.4 卸载

```bash
# Server
sudo /usr/local/bin/k3s-uninstall.sh

# Agent
sudo /usr/local/bin/k3s-agent-uninstall.sh
```

---

## 7. 常见问题

### Q: 为什么用 Tailscale 集成而不是手动配 Flannel？

**官方推荐。** 手动配 Flannel 需要处理 VXLAN/WireGuard 隧道、路由、ACL 等复杂问题。Tailscale 集成让 K3s 自动管理这些，更可靠。

### Q: embedded etcd 支持跨网络部署吗？

**不支持。** 官方文档明确说明：如果使用 embedded etcd，所有 server 节点必须能通过内网 IP 互相访问。

### Q: Pod CIDR 怎么配？

默认 `10.42.0.0/16`。需要在 Tailscale ACL 中配置 `autoApprovers.routes` 允许这个 CIDR 的路由。如果修改了 `--cluster-cidr`，需要同步修改 ACL。

### Q: Agent 安装后 Tailscale 节点没批准怎么办？

访问 https://login.tailscale.com/admin/machines 手动批准。如果配置了 `autoApprovers`，会自动批准。

### Q: `--vpn-auth` 的 auth key 会过期吗？

如果生成时勾选了 **Reusable**，key 不会过期。否则每次安装都需要重新生成。

### Q: 怎么查看 Tailscale 节点状态？

```bash
tailscale status
```

---

## 8. 参考

- [K3s 官方文档 - 分布式/多云部署](https://docs.k3s.io/networking/distributed-multicloud)
- [K3s 网络选项](https://docs.k3s.io/networking/basic-network-options)
- [K3s 安装要求](https://docs.k3s.io/installation/requirements)
- [Tailscale 官方文档](https://tailscale.com/kb/)
- [Tailscale 完全指南](../tools/network/tailscale-完全指南.md)（本仓库）
- [Kubernetes 开发者学习手册](README.md)（本仓库）
