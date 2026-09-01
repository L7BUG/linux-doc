# K3s Docker Compose 快速部署

通过 Docker Compose 在开发机上一键启动 K3s Server（自带 Agent 功能）。集群 leader 是云服务器 B（sv1，7×24 在线）。

## 核心概念

**K3s server 自带 agent 功能**，不需要单独跑 agent 容器。一个 server 容器 = 控制面 + 工作节点。

## 前提

1. sv1（云服务器 B）已安装 K3s 并初始化集群（见主文档 Phase 1）
2. 在 Tailscale 后台给每台机器设置好节点名和 IP

## 文件说明

```
docker-compose/
├── docker-compose.yml    ← 主配置（两台开发机通用）
├── .env.company          ← 公司开发机配置
├── .env.home             ← 家里开发机配置
└── README.md             ← 本文件
```

## 使用步骤

### 公司开发机

```bash
mkdir -p ~/k3s-cluster && cd ~/k3s-cluster
cp /path/to/docker-compose.yml .
cp /path/to/.env.company .env
vim .env                    # 确认 IP
docker compose up -d
docker compose logs -f      # 等待 Ready
```

### 家里开发机

```bash
mkdir -p ~/k3s-cluster && cd ~/k3s-cluster
cp /path/to/docker-compose.yml .
cp /path/to/.env.home .env
vim .env                    # 确认 IP
docker compose up -d
```

> **启动顺序**：sv1 → pc1 → pc2，每个等 Ready 后再启动下一个。

## 常用命令

```bash
cd ~/k3s-cluster

docker compose ps                    # 查看状态
docker compose logs -f               # 实时日志
docker compose restart               # 重启
docker compose stop                  # 停止（保留数据）
docker compose start                 # 启动
docker compose down -v               # 销毁（数据丢失！）
```

## 注意事项

- **所有节点的 K3S_TOKEN 必须一致**
- **sv1 必须先启动**（集群 leader）
- K3s server 自带 agent，**不需要单独跑 agent 容器**
- `.env` 中 `--node-ip` 必须用 Tailscale IP
- `--flannel-backend` 和 `--disable` 只能用在 server 上
