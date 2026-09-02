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
| 云服务器 B | 4C/4GB | **Server（leader）+ Agent** | 原生 K3s | 7×24 |
| 公司开发机 | 6C/12T + 32GB | **Server + Agent** | Docker 容器 | 很少关机 |
| 家里开发机 | 6C/12T + 32GB | **Server + Agent** | Docker 容器 | 可能关机 |
| 云服务器 A | 2C/2GB | **Agent** | 原生 K3s | 7×24 |

### 1.3 为什么云服务器 B 做 leader

集群的第一个 server（leader）负责初始化 etcd，其他节点通过它加入。**选 sv1 做 leader 而不是开发机的原因**：

| | sv1（云服务器 B） | 开发机 |
|---|---|---|
| 在线时间 | 7×24 永远在线 | 可能关机 |
| 网络 | 公网 IP，稳定 | 内网，Tailscale 打洞 |
| API Server 可达性 | 永远可达 | 关机后不可达 |
| kubeconfig 指向 | 始终有效 | 关机后失效 |

**sv1 做 leader 的好处**：开发机关机后，API Server（在 sv1 上）仍然运行，kubectl 仍然可用，只是无法调度新 Pod 到开发机上。

### 1.4 为什么是 3 server HA

公司开发机很少关机 + 两台云服务器永远在线 → **3 台 server 大部分时间都在线**：

| 场景 | 存活 Server | Quorum (2/3) | 集群状态 |
|------|-----------|-------------|---------|
| 全部在线 | 3/3 | ✅ | 完整可用 |
| 家里关机 | 2/3（公司+云B） | ✅ | 正常可用 |
| 周末家里关机 | 2/3 | ✅ | 正常可用 |
| 公司+家里都关机（极少见） | 1/3 | ❌ | 暂停，开机自动恢复 |

**关键优势**：家里关机时，公司开发机 + 云服务器 B 仍然提供 2/3 quorum，集群正常运行。

### 1.5 资源规划

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

### 2.4 配置镜像加速（所有机器）

K3s 使用内置的 containerd，镜像加速配置文件是 `/etc/rancher/k3s/registries.yaml`。

> **注意**：必须写到 `/etc/rancher/k3s/registries.yaml`，写到 `/etc/containerd/config.toml` 对 K3s 无效（那是系统 containerd 的配置，K3s 不认）。

### 2.5 Docker 环境特殊配置

K3s in Docker 必须加 `--disable-network-policy`，否则 kube-router 会误杀 Pod 间流量（CoreDNS 不通、Service 访问不了）。

原因：kube-router 在 Docker 容器内无法正确管理 iptables FORWARD 链，默认 DROP 策略会阻断合法的 Pod→Service 流量。开发/测试环境不需要 Network Policy，直接禁用即可。

docker-compose 的 `.env` 文件中已包含此参数：

```
K3S_SERVER_FLAGS=... --disable=traefik --disable-network-policy
```

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

1. 复制 docker-compose 目录时 `registries.yaml` 已包含在内
2. docker-compose.yml 已挂载 `./registries.yaml` 到容器内
3. 重启生效：

```bash
cd ~/k3s-cluster
docker compose down
docker compose up -d
```

---

## 3. Phase 1：云服务器 B（sv1，集群 leader）

sv1 是 7×24 在线的云服务器，作为集群第一个 server，负责初始化 etcd。

### 3.1 安装 K3s（原生，不用 Docker）

```bash
# 在 sv1 上执行（Debian）
curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn INSTALL_K3S_EXEC="server" sh -s - \
  --cluster-init \
  --tls-san=sv1 --tls-san=pc1 --tls-san=pc2 \
  --node-ip=<sv1的Tailscale-IP> \
  --flannel-backend=host-gw \
  --disable=traefik
```

> **替换占位符**：`<sv1的Tailscale-IP>` 换成实际 IP（`tailscale status` 查看）。

### 3.2 等待 Server 就绪

```bash
# 查看节点状态
sudo k3s kubectl get nodes
# 看到 sv1 状态为 Ready 说明启动成功
```

### 3.3 获取节点令牌

```bash
sudo cat /var/lib/rancher/k3s/server/node-token
# 记下这个令牌，其他节点加入时需要
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

---

## 4. Phase 2：公司开发机（pc1，第二个 Server）

### 4.1 准备 docker-compose 文件

```bash
mkdir -p ~/k3s-cluster && cd ~/k3s-cluster
cp /path/to/docker-compose.yml .
cp /path/to/.env.company .env
```

### 4.2 修改 .env 配置

```bash
vim .env
```

**必须确认的值**：

```bash
K3S_TOKEN=k3s-cluster-token-2026    # 和 sv1 上的令牌一致！
K3S_NODE_IP=100.64.0.1               # pc1 的 Tailscale IP
```

> `.env.company` 模板中预设了 `--server https://sv1:6443`，指向集群 leader。

### 4.3 启动

```bash
docker compose up -d
```

### 4.4 等待就绪 + 配置 kubectl

```bash
# 等待启动
docker compose logs -f k3s-server
# 看到 Ready 后 Ctrl+C

# 配置 kubectl
mkdir -p ~/.kube
docker cp k3s-server:/etc/rancher/k3s/k3s.yaml ~/.kube/config
sed -i 's/127.0.0.1:6443/sv1:6443/g' ~/.kube/config
chmod 600 ~/.kube/config

# 验证（应该看到 sv1 和 pc1 都 Ready）
kubectl get nodes
```

---

## 4. Phase 2：公司开发机（pc1，第二个 Server）

### 4.1 准备 docker-compose 文件

```bash
mkdir -p ~/k3s-cluster && cd ~/k3s-cluster
cp /path/to/docker-compose.yml .
cp /path/to/.env.company .env
```

### 4.2 修改 .env 配置

```bash
vim .env
```

**必须确认的值**：

```bash
K3S_TOKEN=k3s-cluster-token-2026    # 和 sv1 上的令牌一致！
K3S_NODE_IP=100.64.0.1               # pc1 的 Tailscale IP
```

> `.env.company` 模板中预设了 `--server https://sv1:6443`，指向集群 leader。

### 4.3 启动

```bash
docker compose up -d
```

### 4.4 等待就绪 + 配置 kubectl

```bash
# 等待启动
docker compose logs -f k3s-server
# 看到 Ready 后 Ctrl+C

# 配置 kubectl
mkdir -p ~/.kube
docker cp k3s-server:/etc/rancher/k3s/k3s.yaml ~/.kube/config
sed -i 's/127.0.0.1:6443/sv1:6443/g' ~/.kube/config
chmod 600 ~/.kube/config

# 验证（应该看到 sv1 和 pc1 都 Ready）
kubectl get nodes
```

---

## 5. Phase 3：家里开发机（pc2，第三个 Server）

### 5.1 准备 docker-compose 文件

```bash
mkdir -p ~/k3s-cluster && cd ~/k3s-cluster
cp /path/to/docker-compose.yml .
cp /path/to/.env.home .env
```

### 5.2 修改 .env 配置

```bash
vim .env
```

**必须确认的值**：

```bash
K3S_TOKEN=k3s-cluster-token-2026    # 和 sv1 上的令牌一致！
K3S_NODE_IP=100.64.0.2               # pc2 的 Tailscale IP
```

> `.env.home` 模板中预设了 `--server https://sv1:6443`，指向集群 leader。

### 5.3 启动

```bash
docker compose up -d
```

> **启动顺序**：sv1 → pc1 → pc2，每个等 Ready 后再启动下一个。

---

## 6. Phase 4：云服务器 A（sv2，Agent）

### 6.1 安装 K3s Agent

```bash
# 在 sv2 上执行（Debian）
# 注意：agent 不认 --flannel-backend 和 --disable，不要加
curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn K3S_URL=https://<sv1的Tailscale-IP>:6443 K3S_TOKEN=<node-token> sh -s - agent \
  --node-ip=<sv2的Tailscale-IP>
```

### 6.2 最终验证

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
# --node-ip 必须用 IP，--server 可以用主机名
# 注意：agent 不认 --flannel-backend 和 --disable，不要加
curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn K3S_URL=https://<sv1的Tailscale-IP>:6443 K3S_TOKEN=<node-token> sh -s - agent \
  --node-ip=<新节点Tailscale-IP>
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

### Q: `--cluster-init` 每次重启都会重新初始化吗？

**不会。** 如果 volume 数据还在，K3s 会检测到已有 etcd 数据并跳过初始化。但**建议集群跑起来后去掉 `--cluster-init`**，避免 volume 丢失时意外创建空集群。编辑 `.env` 删除该参数，`docker compose down && docker compose up -d` 重启生效。

### Q: 嵌入式 etcd 初始化失败？

**确保第一个 server 使用了 `--cluster-init`。** 只有公司开发机需要这个参数，其他 server 用 `--server` 加入。

### Q: `--node-ip` 能用主机名吗？

**不能。** K3s 的 `--node-ip` 参数只接受 IP 地址格式，不接受主机名。能用主机名的参数只有 `--server` 和 `--tls-san`。`--node-ip` 必须填 Tailscale 分配的 IP（`tailscale status` 查看）。

### Q: `--flannel-backend` 和 `--disable` 能用在 agent 上吗？

**不能。** 这两个都是 server 参数，agent 不认。agent 的网络配置跟随 server，不需要单独指定。docker-compose 的 `.env` 文件中，`K3S_AGENT_FLAGS` 只能包含 `--node-ip` 和 `--server`，不要加其他参数。

### Q: kubeconfig 地址是 127.0.0.1？

```bash
sed -i 's/127.0.0.1:6443/sv1:6443/g' ~/.kube/config
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
