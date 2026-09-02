# Kubernetes 开发者学习手册

> 🎯 面向开发人员：从本地搭建到把应用部署到 K8s 的完整路径
> 🖥️ 环境：Arch Linux，单机学习（minikube/k3d）
> 📝 作者：L7-BUG | 更新：2026-09-01

---

## 写在前面

作为开发者，学 K8s 的核心目标：**把自己的应用跑上去、能调试、能迭代**。不追求成为集群运维专家（那是 SRE 的事）。

**你需要掌握的**：kubectl 基本操作 + Deployment/Service/Ingress + Helm 部署 + 本地调试
**不需要深入的**：etcd 调优、集群网络方案选型、K8s 源码

---

## 学习路线（5 天，每天 2 小时）

| 天 | 章节 | 内容 | 重点 |
|---|---|---|---|
| Day 1 | [第1章](ch1-concepts.md) | 概念快速过 + 环境搭建 | 搭好 minikube，kubectl 跑通 |
| Day 2 | [第2章](ch2-core-objects.md) | 核心对象：Pod/Deployment/Service | **最重要**，会写 YAML |
| Day 3 | [第3章](ch3-networking.md) | Ingress + ConfigMap/Secret + Namespace | 应用暴露 + 配置管理 |
| Day 4 | [第4章](ch4-storage.md) | 存储（PV/PVC）+ 运维（健康检查/扩缩容） | 数据持久化 + 自动化 |
| Day 5 | [第5章](ch5-helm.md) | Helm 包管理 + 综合实战 | **能把应用一键部署** |

---

## 各章概要

### [第1章：概念 + 环境搭建](ch1-concepts.md)
K8s 解决什么问题 → 架构速览（5分钟搞定）→ Arch 安装 kubectl + minikube → 跑通第一个集群

### [第2章：核心对象](ch2-core-objects.md) ⭐
Pod/Deployment/Service 三大核心 → 会写 YAML → 会用 kubectl apply/scale/rollout → 部署 Nginx 并访问

### [第3章：网络与配置](ch3-networking.md)
Ingress 域名路由 → ConfigMap/Secret 配置注入 → Namespace 隔离 → 搭配你自己的 Web 应用

### [第4章：存储与运维](ch4-storage.md)
PVC 持久化（重启不丢数据）→ 健康检查（Liveness/Readiness）→ 资源限制 → HPA 自动扩缩

### [第5章：Helm + 综合实战](ch5-helm.md)
Helm 是什么（K8s 的 apt）→ Chart 结构 → 一键部署完整应用 → 你自己的 Node.js/Java 应用 K8s 化

---

## 进阶内容

| 文档 | 简介 | 前置 |
|------|------|------|
| [Istio 服务网格详解](istio详解.md) | K8s 微服务的"智能管家"——流量管理、熔断、安全、可观测性 | 完成 5 章学习 |
| [K8s + Istio + Spring Boot 实战教程](istio-springboot.md) | 在 K8s 上用 Istio 管理 Spring Boot 微服务——流量管理、安全、可观测性 | 完成 Istio 详解 |
| [K3s + Docker + Tailscale 多节点集群部署指南](k3s-docker-tailscale-集群部署指南.md) | 用 Docker 容器运行 K3s Agent，1 台云服务器做 Server，所有节点通过 Tailscale 互联 | 了解 K8s 基础概念 |

---

## 前置知识要求

- ✅ 会用 Linux 终端（你已经有了）
- ✅ 知道 Docker 基础（容器/镜像/Dockerfile）
- ⚠️ 不需要 K8s 基础（从零开始）
- ⚠️ 不需要 Go/Python（只写 YAML，不写代码）

## 参考资料

- K8s 官方文档（中文）：https://kubernetes.io/zh-cn/docs/
- kubectl 命令参考：https://kubernetes.io/docs/reference/kubectl/cheatsheet/
- killerCoda 在线实验：https://killercoda.com/
- Helm 官方文档：https://helm.sh/zh/docs/
- K3s 官方文档：https://docs.k3s.io/

---

## 环境搭建速查

### 安装工具

```bash
sudo pacman -S kubectl helm     # K8s 命令行 + Helm 包管理器
yay -S minikube                  # 单机 K8s 集群
```

### 启用插件

| 插件 | 用途 | 启用命令 |
|---|---|---|
| **ingress** | Ingress 域名路由（Nginx 反向代理） | `minikube addons enable ingress` |
| **headlamp** | Web 控制台（替代已弃用的 Dashboard） | `minikube addons enable headlamp` |
| **metrics-server** | HPA 自动扩缩（收集 CPU/内存指标） | `minikube addons enable metrics-server` |
| **storage-provisioner** | 动态持久化存储（PVC 自动分配） | `minikube addons enable storage-provisioner` |

> `storage-provisioner` minikube 默认已启用，其他 3 个需要手动开。

### 一键启用全部插件

```bash
minikube addons enable ingress
minikube addons enable headlamp
minikube addons enable metrics-server
minikube addons enable storage-provisioner
```
