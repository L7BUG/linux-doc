# K3s Docker Compose 快速部署

通过 Docker Compose 在开发机上一键启动 K3s Server + Agent。

## 文件说明

```
docker-compose/
├── docker-compose.yml    ← 主配置（两台开发机通用，不需要改）
├── .env.company          ← 公司开发机配置（第一个 server，--cluster-init）
├── .env.home             ← 家里开发机配置（第二个 server，--server 加入）
└── README.md             ← 本文件
```

## 使用步骤

### 第一步：公司开发机（第一个 Server，很少关机）

```bash
# 1. 复制文件到工作目录
mkdir -p ~/k3s-cluster && cd ~/k3s-cluster
cp /path/to/docker-compose.yml .
cp /path/to/.env.company .env

# 2. 修改 .env 中的 Tailscale IP（替换 100.64.0.x 为实际地址）
vim .env

# 3. 启动
docker compose up -d

# 4. 等待启动完成（约30秒）
docker compose logs -f k3s-server
# 看到 "kubectl get nodes" 有 Ready 输出后 Ctrl+C

# 5. 配置 kubectl
mkdir -p ~/.kube
docker cp k3s-server:/etc/rancher/k3s/k3s.yaml ~/.kube/config
sed -i 's/127.0.0.1:6443/<你的Tailscale-IP>:6443/g' ~/.kube/config
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

# 2. 修改 .env 中的 Tailscale IP
vim .env

# 3. 启动（自动加入公司开发机的集群）
docker compose up -d

# 4. 验证（在任意一台开发机上）
kubectl get nodes
# 应该看到 3 个 server 节点
```

### 第三步：云服务器加入

```bash
# 云服务器 B（4C/4G）— 第三个 Server（原生安装）
# 在云服务器 B 上执行：
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="server" \
  sh -s - \
  --server https://<公司Tailscale-IP>:6443 \
  --node-ip=<云服务器B-Tailscale-IP> \
  --flannel-backend=host-gw \
  --disable=traefik

# 云服务器 A（2C/2G）— Agent（原生安装）
# 在云服务器 A 上执行：
curl -sfL https://get.k3s.io | K3S_URL=https://<公司Tailscale-IP>:6443 \
  K3S_TOKEN=<node-token> \
  sh -s - agent \
  --node-ip=<云服务器A-Tailscale-IP> \
  --flannel-backend=host-gw \
  --disable=traefik
```

## 常用命令

```bash
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
- **公司开发机先启动**（`--cluster-init`），等 Ready 后再启动家里开发机
- `.env` 中的 IP 必须替换成实际的 Tailscale IP
- `--restart unless-stopped` 保证 Docker 重启后自动恢复
