# K3s Docker Compose 快速部署

通过 Docker Compose 在开发机上一键启动 K3s Server + Agent。使用 Tailscale 节点名代替 IP，配置更简洁。

## 前提

在 Tailscale 后台（https://login.tailscale.com/admin/machines）给每台机器设置好节点名：

| 机器 | Tailscale 节点名 | 说明 |
|------|-----------------|------|
| 公司开发机 | `pc1` | 第一个 Server |
| 家里开发机 | `pc2` | 第二个 Server |
| 云服务器 B | `sv1` | 第三个 Server（原生安装） |
| 云服务器 A | `sv2` | Agent（原生安装） |

> 节点名可以自定义，只要能 `ping pc1` 通就行。MagicDNS 会自动解析。

## 文件说明

```
docker-compose/
├── docker-compose.yml    ← 主配置（两台开发机通用，不需要改）
├── .env.company          ← 公司开发机配置（第一个 server，--cluster-init）
├── .env.home             ← 家里开发机配置（第二个 server，--server 加入）
└── README.md             ← 本文件
```

## 使用步骤

### 第一步：公司开发机（第一个 Server）

```bash
# 1. 复制文件
mkdir -p ~/k3s-cluster && cd ~/k3s-cluster
cp /path/to/docker-compose.yml .
cp /path/to/.env.company .env

# 2. 修改 .env（只需要改节点名，确认和 Tailscale 后台一致）
vim .env

# 3. 启动
docker compose up -d

# 4. 等待启动完成（约30秒）
docker compose logs -f k3s-server
# 看到 Ready 输出后 Ctrl+C

# 5. 配置 kubectl
mkdir -p ~/.kube
docker cp k3s-server:/etc/rancher/k3s/k3s.yaml ~/.kube/config
sed -i 's/127.0.0.1:6443/pc1:6443/g' ~/.kube/config
chmod 600 ~/.kube/config

# 6. 验证
kubectl get nodes
```

### 第二步：家里开发机（第二个 Server）

```bash
# 1. 复制文件
mkdir -p ~/k3s-cluster && cd ~/k3s-cluster
cp /path/to/docker-compose.yml .
cp /path/to/.env.home .env

# 2. 修改 .env（确认节点名）
vim .env

# 3. 启动（自动加入 pc1 的集群）
docker compose up -d

# 4. 验证（在任意一台开发机上）
kubectl get nodes
# 应该看到 3 个 server 节点
```

### 第三步：云服务器加入

```bash
# 云服务器 B（sv1）— 第三个 Server（原生安装）
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="server" \
  sh -s - \
  --server https://pc1:6443 \
  --tls-san=pc1 --tls-san=pc2 --tls-san=sv1 \
  --node-ip=sv1 \
  --flannel-backend=host-gw \
  --disable=traefik

# 云服务器 A（sv2）— Agent（原生安装）
curl -sfL https://get.k3s.io | K3S_URL=https://pc1:6443 \
  K3S_TOKEN=<node-token> \
  sh -s - agent \
  --node-ip=sv2 \
  --flannel-backend=host-gw \
  --disable=traefik
```

## 常用命令

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

## 注意事项

- **两台开发机的 K3S_TOKEN 必须一致**
- **pc1 先启动**（`--cluster-init`），等 Ready 后再启动 pc2
- 节点名必须和 Tailscale 后台一致，否则打洞会失败
- `--restart unless-stopped` 保证 Docker 重启后自动恢复
