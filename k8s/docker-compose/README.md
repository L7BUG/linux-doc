# K3s Docker Compose 快速部署

通过 Docker Compose 在开发机上一键启动 K3s Server + Agent。集群 leader 是云服务器 B（sv1，7×24 在线）。

## 前提

1. sv1（云服务器 B）已安装 K3s 并初始化集群（见主文档 Phase 1）
2. 在 Tailscale 后台给每台机器设置好节点名：

| 机器 | Tailscale 节点名 | 说明 |
|------|-----------------|------|
| 云服务器 B | `sv1` | 集群 leader，7×24 在线 |
| 公司开发机 | `pc1` | 第二个 Server |
| 家里开发机 | `pc2` | 第三个 Server |
| 云服务器 A | `sv2` | Agent |

## 文件说明

```
docker-compose/
├── docker-compose.yml    ← 主配置（两台开发机通用，不需要改）
├── .env.company          ← 公司开发机配置（加入 sv1 集群）
├── .env.home             ← 家里开发机配置（加入 sv1 集群）
└── README.md             ← 本文件
```

## 使用步骤

### 公司开发机（pc1）

```bash
mkdir -p ~/k3s-cluster && cd ~/k3s-cluster
cp /path/to/docker-compose.yml .
cp /path/to/.env.company .env
vim .env                    # 确认 K3S_NODE_IP 是 pc1 的 Tailscale IP
docker compose up -d
docker compose logs -f k3s-server  # 等待 Ready
```

### 家里开发机（pc2）

```bash
mkdir -p ~/k3s-cluster && cd ~/k3s-cluster
cp /path/to/docker-compose.yml .
cp /path/to/.env.home .env
vim .env                    # 确认 K3S_NODE_IP 是 pc2 的 Tailscale IP
docker compose up -d
```

> **启动顺序**：sv1 → pc1 → pc2，每个等 Ready 后再启动下一个。

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

- **所有节点的 K3S_TOKEN 必须一致**
- **sv1 必须先启动**（它是集群 leader，其他节点通过它加入）
- `.env` 中 `--node-ip` 必须用 Tailscale IP（K3s 硬限制）
- `--server` 和 `--tls-san` 可以用主机名（sv1, pc1 等）
- `--restart unless-stopped` 保证 Docker 重启后自动恢复
