# 第1章：K8s 概念 + 环境搭建（30分钟）

> 🎯 目标：搞懂 K8s 是什么、装好 kubectl + minikube、跑通第一个集群

---

## 1. K8s 一句话解释

**Kubernetes = 你的"服务器管家"。**

你写好一个 Docker 镜像，告诉 K8s："给我跑 3 个副本，内存限制 512MB，对外端口 8080"——K8s 自动帮你：
- 把容器分配到某台机器上运行
- 挂了自动重启（健康检查）
- 流量大了自动扩容
- 不用的自动缩容/删除

**你不再关心"这台机器够不够"——K8s 帮你管。**

## 2. 核心架构（5分钟搞懂）

```
┌─────────────────── 集群 ───────────────────┐
│                                            │
│  ┌──── Master 节点（大脑）────┐            │
│  │  API Server  ← 你在这里下令│            │
│  │  etcd         ← 存状态数据│            │
│  │  Scheduler    ← 决定容器去哪│           │
│  │  Controller   ← 保持期望状态│           │
│  └────────────────────────────┘            │
│                                            │
│  ┌── Worker 节点（干活）──┐  ┌── Worker ──┐│
│  │  kubelet      ← 跟 Master 汇报│  │  ...   ││
│  │  kube-proxy   ← 处理网络转发  │  │        ││
│  │  Pod/容器      ← 你的应用跑这里│ │        ││
│  └─────────────────────┘  └─────────┘│
│                                            │
└────────────────────────────────────────────┘
```

**开发者视角：** 你只需要跟 API Server 打交道（kubectl 命令），Master 和 Worker 是 K8s 内部的。

## 3. 必须知道的 4 个概念

| 概念 | 一句话 | 你操作它的命令 |
|---|---|---|
| **Pod** | 一个/多个容器的最小单位（≈ 一台"虚拟机"） | `kubectl get pod` |
| **Deployment** | 管理 Pod 的副本数（你要几个，挂了自动重启） | `kubectl get deploy` |
| **Service** | 把 Pod 暴露给外部访问（相当于"端口转发"） | `kubectl get svc` |
| **Node** | 集群里的物理/虚拟机 | `kubectl get node` |

**记住这 4 个就够应付 80% 的场景了。**

## 4. Arch Linux 环境搭建（15分钟）

### 4.1 安装 kubectl（K8s 命令行工具）

```bash
# Arch 官方源直接装
sudo pacman -S kubectl

# 验证
kubectl version --client
```

### 4.2 安装 minikube（单机版 K8s）

```bash
# AUR 仓库
yay -S minikube

# 启动集群（会自动用 Docker 当"虚拟机"）
minikube start

# 验证
kubectl get nodes
# 应该看到: minikube   Ready   control-plane   ...
```

### 4.3 常用命令速查

```bash
minikube dashboard     # 打开 Web 控制台
minikube status        # 集群状态
minikube stop          # 停止集群
minikube delete        # 删除集群（重建用）
```

## 5. 动手：跑通第一个应用

```bash
# 1. 创建 Deployment（跑一个 Nginx）
kubectl create deployment nginx --image=nginx:latest

# 2. 暴露为 Service（端口 80→30000）
kubectl expose deployment nginx --port=80 --type=NodePort

# 3. 获取访问地址
minikube service nginx --url
# 输出类似: http://127.0.0.1:30000

# 4. 浏览器打开 → 看到 Nginx 欢迎页 ✅
```

**看到 Nginx 欢迎页了？恭喜，你的第一个 K8s 应用跑起来了。**

## 6. 本章小结

| 你学到了 | 关键点 |
|---|---|
| K8s 是什么 | 服务器管家，自动管理容器 |
| 核心架构 | Master（大脑）+ Worker（干活） |
| 4 个核心概念 | Pod / Deployment / Service / Node |
| 环境搭建 | `pacman -S kubectl` + `yay -S minikube` |
| 跑通第一个应用 | nginx → 暴露 → 浏览器访问 ✅ |

## 7. 常见问题

**Q: `minikube start` 报错 "No hypervisor found"？**
→ 确保 Docker 正在运行（`sudo systemctl start docker`）

**Q: minikube 太慢/内存不够？**
→ `minikube start --memory=2048 --cpus=2`（降低分配）

**Q: kubectl 命令报 "connection refused"？**
→ minikube 没启动，先 `minikube start`

---

> 📝 下一步：[第2章：核心对象详解](ch2-core-objects.md) —— 会写 YAML，会部署自己的应用
