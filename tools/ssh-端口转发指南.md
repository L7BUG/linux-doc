# SSH 端口转发完全指南

SSH 除了远程登录，最实用的功能就是**端口转发**——通过加密隧道把本地和远程的网络端口打通。两个核心参数：`-L`（本地转发）和 `-R`（远程转发），方向相反但互为镜像。

## 1. 快速对比

| | `-L` 本地转发 | `-R` 远程转发 |
|------|------|------|
| 监听端 | 本地 | 远程 |
| 数据流向 | 本地 → 远程 → 目标 | 远程 → 本地 → 目标 |
| 典型场景 | 访问远程内网服务 | 把本地服务暴露给远程 |
| 记忆口诀 | **L**ocal entry（入口在本地） | **R**emote entry（入口在远程） |

---

## 2. `-L` — 本地端口转发

### 2.1 命令格式

```bash
ssh -N -L [本地地址:]本地端口:目标主机:目标端口 user@跳板机
```

### 2.2 工作原理

```
你 ──→ localhost:8080 ──[SSH 加密]──→ 跳板机 ──→ internal-server:80
```

所有发往本地 `8080` 端口的流量，经 SSH 加密后通过跳板机转发到内网的 `internal-server:80`。

### 2.3 常用场景

```bash
# 通过跳板机访问内网数据库
ssh -N -L 3306:db.internal:3306 user@bastion

# 通过跳板机访问内网 Web 服务
ssh -N -L 8080:web.internal:80 user@bastion

# 只绑定本地回环（默认），只有本机能用这个隧道
ssh -N -L 9090:remote-host:22 user@bastion

# 绑定所有网络接口，同网段其他机器也能用你的隧道
ssh -N -L 0.0.0.0:9090:remote-host:22 user@bastion

# 后台运行
ssh -NfL 8080:web.internal:80 user@bastion
```

> `-N` 表示不执行远程命令（不打开 shell），`-f` 表示认证后在后台运行，两者常搭配使用。

### 2.4 典型示例

你有台云服务器 `vps.example.com`，想访问服务器内网里的 Jenkins（`10.0.0.5:8080`）：

```bash
ssh -NfL 9090:10.0.0.5:8080 user@vps.example.com
```

然后浏览器打开 `http://localhost:9090`，就访问到内网 Jenkins 了。

---

## 3. `-R` — 远程端口转发

### 3.1 命令格式

```bash
ssh -N -R [远程地址:]远程端口:本地目标:本地端口 user@远程主机
```

### 3.2 工作原理

```
外部用户 ──→ remote-host:8080 ──[SSH 加密]──→ 你的机器:3000
```

远程服务器上的 `8080` 端口收到的请求，经 SSH 隧道转发到你本地的 `3000` 端口。

### 3.3 常用场景

```bash
# 把本地开发的前端项目暴露到公网服务器
ssh -NfR 8080:localhost:3000 user@vps

# 把本地 SSH 端口映射回去（让远程机器能 SSH 连回来）
ssh -NfR 2222:localhost:22 user@vps

# 暴露给远程所有网络接口
ssh -NfR 0.0.0.0:8080:localhost:3000 user@vps
```

> **注意：** 默认情况下 `-R` 绑定的远程端口只监听 `127.0.0.1`（远程本机回环），外部机器无法直接访问。如需对外开放，必须在远程服务器的 `/etc/ssh/sshd_config` 中设置 `GatewayPorts yes` 并重启 sshd。

### 3.4 典型示例

你在公司内网开发，想让外网的朋友预览你本地的 `localhost:3000`（React 开发服务器）：

```bash
# 在你的开发机上执行
ssh -NfR 8080:localhost:3000 user@your-public-vps
```

朋友访问 `http://your-public-vps:8080` 就能看到你的本地服务。

> **注意：** 如果远程 `GatewayPorts` 没开，朋友访问不到。此时可以在远程服务器上做二次转发：
> ```bash
> # 在远程服务器上执行
> socat TCP-LISTEN:0.0.0.0:8080,fork TCP:127.0.0.1:8080
> ```

---

## 4. 常用辅助参数

| 参数 | 含义 |
|------|------|
| `-N` | 不执行远程命令，纯转发 |
| `-f` | 认证后在后台运行 |
| `-T` | 禁用伪终端分配（纯转发时用） |
| `-C` | 压缩数据传输（慢速网络下有用） |
| `-v` / `-vv` | 详细输出 / 更详细，排查问题用 |
| `-o ServerAliveInterval=60` | 每 60 秒发心跳包，防止连接断开 |
| `-o ExitOnForwardFailure=yes` | 端口转发失败时立即退出，而不是静默继续 |

### 4.1 生产级写法

```bash
# 本地转发（适用于不稳定网络）
ssh -NfTL 8080:web.internal:80 \
    -o ServerAliveInterval=60 \
    -o ExitOnForwardFailure=yes \
    user@bastion

# 远程转发（同上）
ssh -NfTR 8080:localhost:3000 \
    -o ServerAliveInterval=60 \
    -o ExitOnForwardFailure=yes \
    user@vps
```

---

## 5. 动态转发 `-D`（SOCKS 代理）

除了 `-L` 和 `-R`，还有一个 `-D` 能创建 SOCKS 代理：

```bash
# 在本地 1080 端口启动 SOCKS5 代理
ssh -NfD 1080 user@vps
```

然后浏览器或系统设置 SOCKS5 代理为 `localhost:1080`，所有流量走远程服务器。

---

## 6. 管理转发隧道

```bash
# 查看当前 SSH 隧道进程
ps aux | grep "ssh -N"

# 或更精确
pgrep -af "ssh -N"

# 关闭指定端口隧道
# 先找到 PID
lsof -i :8080
# 再 kill
kill <PID>
```

---

## 7. 常见问题

### Q1: 隧道断了怎么办？

用 `autossh` 自动重连：

```bash
# 安装
sudo pacman -S autossh

# 使用（参数和 ssh 完全一样）
autossh -M 0 -NfL 8080:web.internal:80 user@bastion
```

`-M 0` 表示关闭 autossh 自身的监控端口，仅依赖 SSH 的心跳。

### Q2: `-L` 绑定的端口被占用？

换一个本地端口即可，比如 `8081`：

```bash
ssh -NfL 8081:web.internal:80 user@bastion
```

### Q3: 如何让 `-R` 的端口直接对外开放？

在远程服务器 `/etc/ssh/sshd_config` 中添加：

```
GatewayPorts yes
```

然后 `sudo systemctl restart sshd`。之后再执行 `-R` 就会默认绑定 `0.0.0.0`。

### Q4: `-L` 和 `-R` 的中转机能是同一台吗？

可以。`-L` 的场景中，如果目标主机就是跳板机本身，直接用 `localhost`：

```bash
ssh -NfL 8080:localhost:80 user@vps
```

### Q5: 如何通过多层跳板转发？

```bash
# 通过 bastion1 再到 bastion2，最终访问 target:80
ssh -NfL 8080:target:80 -J user1@bastion1 user2@bastion2
```

`-J` 是 ProxyJump 的简写，SSH 7.3 以上版本支持。
