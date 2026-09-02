# K3s Docker Compose 快速部署

通过 Docker Compose 在开发机上运行 K3s Agent，加入云服务器的集群。

## 文件说明

```
docker-compose/
├── docker-compose.yml    ← 主配置
├── .env.company          ← 公司开发机
├── .env.home             ← 家里开发机
├── registries.yaml       ← 镜像加速
└── README.md
```

## 使用步骤

```bash
mkdir -p ~/k3s-cluster && cd ~/k3s-cluster
cp /path/to/docker-compose.yml .
cp /path/to/.env.company .env  # 或 .env.home
vim .env                       # 填入 token、server IP、本机 IP
docker compose up -d
```

## .env 配置

```bash
K3S_TOKEN=<sv1的token>           # 从 sv1 获取
K3S_URL=https://<sv1的IP>:6443    # Server 地址
NODE_IP=<本机Tailscale-IP>        # 本机 IP
```

## 常用命令

```bash
docker compose ps / logs -f / restart / stop / start
docker compose down -v  # 销毁（数据丢失）
```
