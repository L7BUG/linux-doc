# K3s Docker Compose 快速部署

通过 Docker Compose 在开发机上运行 K3s Agent，加入云服务器 B（sv1）的集群。

## 架构

```
sv1 (云服务器 4C/4G) ─── Server only（仅控制面）
    │
    ├── 你的开发机 ─── Agent（Docker，跑业务 Pod）
    └── 其他节点 ─── Agent（原生 K3s）
```

## 前提

1. sv1 已安装 K3s server 并初始化集群
2. 在 Tailscale 后台给每台机器设置好节点名和 IP

## 文件说明

```
docker-compose/
├── docker-compose.yml    ← 主配置
├── .env.company          ← 公司开发机配置
├── .env.home             ← 家里开发机配置
├── registries.yaml       ← 镜像加速配置
└── README.md
```

## 使用步骤

```bash
mkdir -p ~/k3s-cluster && cd ~/k3s-cluster
cp /path/to/docker-compose.yml .
cp /path/to/.env.company .env  # 或 .env.home
vim .env                    # 确认 IP
docker compose up -d
docker compose logs -f      # 等待 Ready
```

## 常用命令

```bash
docker compose ps
docker compose logs -f
docker compose restart
docker compose stop / start
docker compose down -v      # 销毁（数据丢失！）
```

## 注意事项

- 所有节点的 `K3S_TOKEN` 必须一致
- sv1 必须先启动，Agent 才能加入
- `.env` 中 `--node-ip` 必须用 Tailscale IP
- K3s 默认使用 vxlan 后端，不需要指定 `--flannel-backend`
