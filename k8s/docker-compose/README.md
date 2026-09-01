# K3s Docker Compose 快速部署

通过 Docker Compose 在开发机上一键启动 K3s Server + Agent。

## 文件说明

```
docker-compose/
├── docker-compose.yml    ← 主配置（两个机器通用，不需要改）
├── .env.company          ← 公司开发机配置模板
├── .env.home             ← 家里开发机配置模板
└── README.md             ← 本文件
```

## 使用步骤

### 第一步：公司开发机（第一个 Server）

```bash
# 1. 把文件复制到你的工作目录
mkdir -p ~/k3s-cluster && cd ~/k3s-cluster
cp /path/to/docker-compose.yml .
cp /path/to/.env.company .env

# 2. 修改 .env 中的 Tailscale IP
#    把 100.64.0.1 和 100.64.0.2 替换成你实际的 Tailscale IP
vim .env

# 3. 启动
docker compose up -d

# 4. 等待启动完成（约30秒）
docker compose logs -f k3s-server
# 看到 "Wrote kubeconfig" 就可以 Ctrl+C 了

# 5. 配置 kubectl
mkdir -p ~/.kube
docker cp k3s-server:/etc/rancher/k3s/k3s.yaml ~/.kube/config
sed -i 's/127.0.0.1:6443/<你的Tailscale-IP>:6443/g' ~/.kube/config
chmod 600 ~/.kube/config

# 6. 验证
kubectl get nodes
# 应该看到 k3s-server-company 和 k3s-agent-company 都是 Ready
```

### 第二步：家里开发机（第二个 Server）

```bash
# 1. 复制文件
mkdir -p ~/k3s-cluster && cd ~/k3s-cluster
cp /path/to/docker-compose.yml .
cp /path/to/.env.home .env

# 2. 修改 .env 中的 Tailscale IP
vim .env

# 3. 启动（会自动加入公司开发机的集群）
docker compose up -d

# 4. 等待启动
docker compose logs -f k3s-server

# 5. 验证（在任意一台开发机上执行）
kubectl get nodes
# 应该看到 4 个节点（2 server + 2 agent）
```

### 第三步：获取集群 Token（给云服务器用）

```bash
# 在公司开发机上执行
docker exec k3s-server kubectl get nodes
docker exec k3s-server cat /var/lib/rancher/k3s/server/node-token
# 复制输出的 token，云服务器加入时需要
```

## 常用命令

```bash
# 查看状态
docker compose ps

# 查看日志
docker compose logs -f k3s-server
docker compose logs -f k3s-agent

# 重启
docker compose restart

# 停止（保留数据）
docker compose stop

# 启动
docker compose start

# 销毁集群（数据丢失！）
docker compose down -v
```

## 注意事项

- **两个机器的 K3S_TOKEN 必须一致**，否则加入集群会失败
- **公司开发机是第一个 server**（`--cluster-init`），家里开发机是第二个（`--server`）
- `.env` 中的 IP 必须替换成你实际的 Tailscale IP
- 启动顺序：先启动公司开发机，等 Ready 后再启动家里开发机
