# K3s + Docker + Tailscale 多节点集群部署指南

> **背景**：用 4 台机器组建 K3s 高可用集群——2 台个人开发机（公司+家里）通过 Docker 容器运行 K3s，2 台云服务器原生安装 K3s，所有节点通过 Tailscale 虚拟局域网互联。3 台 Server 节点（公司开发机+家里开发机+4C 云服务器）组成嵌入式 etcd 集群，实现高可用。

---

## 1. 架构概览

### 1.1 集群拓扑

```
┌────────────────────────────────────────────────────────────────┐
│                      Tailscale 虚拟局域网                       │
│                 100.x.x.x 全节点互通                            │
└────────────────────────────────────────────────────────────────┘
      │                            │
┌─────┴───────────────┐   ┌───────┴─────────────────┐
│  公司开发机 (32GB)    │   │  家里开发机 (32GB)        │
│  很少关机 ✅          │   │  可能关机                 │
│                     │   │                          │
│  Docker             │   │  Docker                  │
│  ┌───────────────┐  │   │  ┌────────────────────┐  │
│  │ K3s Server    │  │   │  │ K3s Server         │  │
│  │ + Agent       │  │   │  │ + Agent            │  │
│  └───────────────┘  │   │  └────────────────────┘  │
└─────────────────────┘   └──────────────────────────┘
      │                            │
      └────────────┬───────────────┘
                   │
      ┌────────────┴────────────────────┐
      │                                 │
┌─────┴───────────────┐   ┌────────────┴──────────┐
│  云服务器 B (4C/4G)   │   │  云服务器 A (2C/2G)     │
│  7×24 在线            │   │  7×24 在线              │
│                      │   │                        │
│  原生 K3s             │   │  原生 K3s               │
│  ┌────────────────┐  │   │  ┌──────────────────┐  │
│  │ K3s Server     │  │   │  │ K3s Agent        │  │
│  │ + Agent        │  │   │  │ (轻量工作节点)     │  │
│  └────────────────┘  │   │  └──────────────────┘  │
└──────────────────────┘   └────────────────────────┘
```

### 1.2 角色分配

| 机器 | 配置 | 角色 | 部署方式 | 在线情况 |
|------|------|------|---------|---------|
| 公司开发机 | 6C/12T + 32GB | **Server + Agent** | Docker 容器 | 很少关机 |
| 家里开发机 | 6C/12T + 32GB | **Server + Agent** | Docker 容器 | 可能关机 |
| 云服务器 B | 4C/4GB | **Server + Agent** | 原生 K3s | 7×24 |
| 云服务器 A | 2C/2GB | **Agent** | 原生 K3s | 7×24 |

### 1.3 为什么是 3 server HA

公司开发机很少关机 + 两台云服务器永远在线 → **3 台 server 大部分时间都在线**：

| 场景 | 存活 Server | Quorum (2/3) | 集群状态 |
|------|-----------|-------------|---------|
| 全部在线 | 3/3 | ✅ | 完整可用 |
| 家里关机 | 2/3（公司+云B） | ✅ | 正常可用 |
| 周末家里关机 | 2/3 | ✅ | 正常可用 |
| 公司+家里都关机（极少见） | 1/3 | ❌ | 暂停，开机自动恢复 |

**关键优势**：家里关机时，公司开发机 + 云服务器 B 仍然提供 2/3 quorum，集群正常运行。

### 1.4 资源规划

| 机器 | K3s Server 占用 | K3s Agent 占用 | 剩余可用 |
|------|----------------|---------------|---------|
| 公司开发机（32GB） | ~500MB | ~200MB | ~31GB |
| 家里开发机（32GB） | ~500MB | ~200MB | ~31GB |
| 云服务器 B（4GB） | ~500MB | ~300MB | ~3.2GB |
| 云服务器 A（2GB） | — | ~500MB | ~1.5GB |

---

## 2. 前置准备

### 2.1 Docker 安装（仅开发机）

```bash
# Debian/Ubuntu
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
# 重新登录生效
```

### 2.2 Tailscale 安装（所有机器）

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
# 浏览器授权登录
```

**验证互通**：

```bash
tailscale ping <其他节点Tailscale-IP>
# 输出 "direct connection" 表示直连成功
```

> 完整 Tailscale 使用指南见 [Tailscale 完全指南](../tools/network/tailscale-完全指南.md)。

### 2.3 防火墙放行端口

#### Tailscale（所有机器）

```
UDP 41641 → WireGuard 隧道（打洞用，仅云服务器需要开放）
```

```bash
sudo ufw allow 41641/udp
```

> 开发机在内网（NAT 后面），不需要开放端口，Tailscale 会主动往外打洞。云服务器有公网 IP，需要开放 UDP 41641 让开发机连进来。**云厂商安全组也要放行此端口**。

#### K3s（集群内部）

| 端口 | 协议 | 用途 | 哪些节点需要 |
|------|------|------|------------|
| 6443 | TCP | Kubernetes API Server | Server 节点 |
| 8472 | UDP | Flannel VXLAN | 所有节点 |
| 2379-2380 | TCP | 嵌入式 etcd | Server 节点间 |
| 10250 | TCP | Kubelet API | 所有节点 |

```bash
sudo ufw allow 6443/tcp
sudo ufw allow 8472/udp
sudo ufw allow 2379:2380/tcp
sudo ufw allow 10250/tcp
```

> 如果 Flannel 使用 `host-gw` 后端（本文推荐），可以不开放 8472/UDP。K3s 端口可以只放 Tailscale 内网，用 ACL 限制访问，更安全。

---

## 3. Phase 1：公司开发机（第一个 Server）

### 3.1 准备 docker-compose 文件

```bash
# 创建工作目录
mkdir -p ~/k3s-cluster && cd ~/k3s-cluster

# 复制 docker-compose 文件（从 linux-doc 仓库获取）
# 文件位置：k8s/docker-compose/
cp /path/to/docker-compose.yml .
cp /path/to/.env.company .env
```

### 3.2 修改 .env 配置

```bash
vim .env
```

**必须确认的值**：

```bash
K3S_TOKEN=k3s-cluster-token-2026    # 自定义一个复杂字符串
K3S_NODE_HOST=pc1                    # 公司开发机的 Tailscale 节点名
```

> `.env.company` 模板中已预设了 `--cluster-init` 参数，公司开发机是第一个 server，负责初始化集群。Tailscale 节点名直接代替 IP，MagicDNS 自动解析。

### 3.3 启动集群

```bash
docker compose up -d
```

### 3.4 等待 Server 就绪

```bash
docker compose logs -f k3s-server
# 看到 "kubectl get nodes" 有 Ready 输出后 Ctrl+C
```

### 3.5 获取节点令牌

```bash
docker exec k3s-server cat /var/lib/rancher/k3s/server/node-token
# 记下这个令牌，云服务器加入时需要
```

### 3.6 配置 kubectl

```bash
mkdir -p ~/.kube
docker cp k3s-server:/etc/rancher/k3s/k3s.yaml ~/.kube/config
sed -i 's/127.0.0.1:6443/pc1:6443/g' ~/.kube/config
chmod 600 ~/.kube/config

# 验证
kubectl get nodes
```

---

## 4. Phase 2：家里开发机（第二个 Server）

### 4.1 准备 docker-compose 文件

```bash
mkdir -p ~/k3s-cluster && cd ~/k3s-cluster
cp /path/to/docker-compose.yml .
cp /path/to/.env.home .env
```

### 4.2 修改 .env 配置

```bash
vim .env
```

**必须确认的值**：

```bash
K3S_TOKEN=k3s-cluster-token-2026    # 和公司开发机保持一致！
K3S_SERVER_HOST=pc1                  # 公司开发机的 Tailscale 节点名
K3S_NODE_HOST=pc2                    # 家里开发机自己的 Tailscale 节点名
```

> `.env.home` 模板中预设了 `--server https://pc1:6443` 参数，家里开发机作为第二个 server 加入已有集群。

### 4.3 启动

```bash
docker compose up -d
```

> **注意**：必须先启动公司开发机并等 Ready 后，再启动家里开发机。

---

## 5. Phase 3：云服务器 B（第三个 Server）

云服务器 B 使用原生 K3s 安装（不用 Docker），作为第三个 server 节点：

```bash
# 在云服务器 B（sv1）上执行（Debian）
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="server" \
  sh -s - \
  --server https://pc1:6443 \
  --tls-san=pc1 --tls-san=pc2 --tls-san=sv1 \
  --node-ip=sv1 \
  --flannel-backend=host-gw \
  --disable=traefik
```

> 原生安装的 K3s 用 systemd 管理，kubeconfig 在 `/etc/rancher/k3s/k3s.yaml`。

### 验证 3 个 Server

```bash
kubectl get nodes -o wide
# 应该看到 3 个 server 节点全部 Ready
```

---

## 6. Phase 4：Agent 节点加入

### 6.1 开发机 Agent（已包含在 docker-compose 中）

开发机的 agent 已经在 `docker compose up -d` 时一起启动了，不需要额外操作。

> `docker-compose.yml` 中的 `k3s-agent` 服务会等 server 健康检查通过后自动加入集群。

### 6.2 云服务器 A（轻量 Agent）

```bash
# 云服务器 A（sv2）— Agent（原生安装）
curl -sfL https://get.k3s.io | K3S_URL=https://pc1:6443 \
  K3S_TOKEN=<node-token> \
  sh -s - agent \
  --node-ip=sv2 \
  --flannel-backend=host-gw \
  --disable=traefik
```

### 6.3 最终验证

```bash
kubectl get nodes -o wide
```

**预期输出**：

```
NAME                    STATUS   ROLES                       AGE   VERSION        INTERNAL-IP
k3s-server-company      Ready    control-plane,master,etcd   1h    v1.31.x+k3s1   100.64.0.1
k3s-server-home         Ready    control-plane,master,etcd   30m   v1.31.x+k3s1   100.64.0.2
k3s-server-cloud-b      Ready    control-plane,master,etcd   15m   v1.31.x+k3s1   100.64.0.10
k3s-agent-company       Ready    <none>                      5m    v1.31.x+k3s1   100.64.0.1
k3s-agent-home          Ready    <none>                      3m    v1.31.x+k3s1   100.64.0.2
k3s-agent-cloud-a       Ready    <none>                      1m    v1.31.x+k3s1   100.64.0.11
```

---

## 7. 集群验证

### 7.1 系统 Pod 检查

```bash
kubectl get pods -A
# 所有 Pod 应该 Running 或 Completed
```

### 7.2 部署测试应用

```bash
kubectl create deployment test-nginx --image=nginx:alpine --replicas=5
kubectl rollout status deployment test-nginx
kubectl get pods -o wide -l app=test-nginx
# Pod 应分布在不同节点

kubectl expose deployment test-nginx --port=80 --type=NodePort
NODE_PORT=$(kubectl get svc test-nginx -o jsonpath='{.spec.ports[0].nodePort}')
curl http://<任意节点Tailscale-IP>:$NODE_PORT
```

### 7.3 高可用测试

```bash
# 模拟家里开发机关机
docker stop k3s-server k3s-agent

# 验证集群仍可用（公司+云B 提供 quorum）
kubectl get nodes -o wide
# 家里节点 NotReady，但集群正常

kubectl create deployment ha-test --image=nginx:alpine
kubectl rollout status deployment ha-test

# 恢复
docker start k3s-server k3s-agent
```

### 7.4 清理

```bash
kubectl delete deployment test-nginx ha-test
kubectl delete svc test-nginx
```

---

## 8. Tailscale 网络集成

### 8.1 流量路径

```
Pod A (公司) → Pod B (云服务器A)
     │                    │
     └── K3s Flannel ────┘
          (host-gw 直接路由)
               │
        Tailscale WireGuard 隧道
               │
     100.64.0.1 ←→ 100.64.0.11
```

Flannel `host-gw` 模式直接通过 Tailscale 的 WireGuard 隧道路由 Pod 流量，无需额外封装。

### 8.2 远程管理

```bash
# 从任意设备（包括笔记本）配置 kubectl
scp pc1:~/.kube/config ~/.kube/config
sed -i 's/127.0.0.1:6443/pc1:6443/g' ~/.kube/config
kubectl get nodes
```

---

## 9. 日常运维

### 9.1 Docker Compose 管理（开发机）

```bash
cd ~/k3s-cluster

docker compose ps                    # 查看状态
docker compose logs -f k3s-server    # Server 日志
docker compose logs -f k3s-agent     # Agent 日志
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

### 9.3 etcd 备份

```bash
# 在任意 server 节点上
# 开发机：
docker exec k3s-server k3s etcd-snapshot save --name backup-$(date +%Y%m%d)

# 云服务器 B：
sudo k3s etcd-snapshot save --name backup-$(date +%Y%m%d)

# 查看快照
docker exec k3s-server k3s etcd-snapshot list
```

### 9.4 添加新节点

```bash
# 新 Agent（原生安装）
curl -sfL https://get.k3s.io | K3S_URL=https://pc1:6443 \
  K3S_TOKEN=<node-token> \
  sh -s - agent \
  --node-ip=<新节点Tailscale节点名> \
  --flannel-backend=host-gw \
  --disable=traefik
```

### 9.5 移除节点

```bash
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data
kubectl delete node <node-name>
# 对应机器上停止服务并卸载
```

### 9.6 卸载 K3s（云服务器）

```bash
# Server
sudo /usr/local/bin/k3s-uninstall.sh

# Agent
sudo /usr/local/bin/k3s-agent-uninstall.sh
```

---

## 10. 常见问题

### Q: 节点 NotReady，INTERNAL-IP 显示 Docker IP 而不是 Tailscale IP？

**`--node-ip` 没有正确设置。** 删除容器/卸载 K3s 后重新启动，确保 `--node-ip` 指向正确的 Tailscale IP。

### Q: 家里关机后集群还能用吗？

**能。** 公司开发机（很少关机）+ 云服务器 B（7×24）= 2/3 quorum，集群正常运行。家里开机后自动恢复。

### Q: 3 个 server 都挂了怎么办？

etcd quorum 丢失，集群暂停。但**数据不丢失**——etcd 数据在 volume/磁盘上。任意 2 个 server 恢复后集群自动恢复。

### Q: 嵌入式 etcd 初始化失败？

**确保第一个 server 使用了 `--cluster-init`。** 只有公司开发机需要这个参数，其他 server 用 `--server` 加入。

### Q: kubeconfig 地址是 127.0.0.1？

```bash
sed -i 's/127.0.0.1:6443/pc1:6443/g' ~/.kube/config
```

### Q: Flannel host-gw 下 Pod 通信不通？

检查 IP 转发：

```bash
sysctl net.ipv4.ip_forward
# 如果是 0：
sudo sysctl -w net.ipv4.ip_forward=1
echo "net.ipv4.ip_forward = 1" | sudo tee -a /etc/sysctl.conf
```

### Q: 云服务器 4C/4G 跑 server 够吗？

**够用。** K3s server 组件约占 500MB~1GB，剩余 3GB+ 可以跑业务 Pod。不适合跑 Elasticsearch 等内存密集型应用。

### Q: 怎么从外网访问集群 Service？

1. **Tailscale 直接访问**：你的设备加入 tailnet，用 NodePort 访问
2. **kubectl port-forward**：`kubectl port-forward svc/my-app 8080:80`
3. **将来加 Ingress**：安装 Traefik 或 Nginx Ingress Controller

---

## 11. 参考

- [K3s 官方文档](https://docs.k3s.io/)
- [K3s HA 安装指南](https://docs.k3s.io/installation/ha)
- [K3s 嵌入式 etcd](https://docs.k3s.io/datastore)
- [K3s Docker 镜像](https://hub.docker.com/r/rancher/k3s)
- [Tailscale 完全指南](../tools/network/tailscale-完全指南.md)（本仓库）
- [Kubernetes 开发者学习手册](README.md)（本仓库）
