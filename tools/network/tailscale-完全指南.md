# Tailscale 完全指南

Tailscale 是基于 WireGuard 的零配置 VPN，专治异地组网、内网穿透、跨 NAT 互联。和传统 VPN 的核心区别：**它不走中心服务器中转流量，而是帮你打洞建立直连，安全、低延迟、几乎零配置。**

## 1. 安装

### 1.1 Arch Linux

```bash
sudo pacman -S tailscale
sudo systemctl enable --now tailscaled
```

### 1.2 其他系统

| 系统 | 安装命令 |
|------|----------|
| Debian / Ubuntu | `curl -fsSL https://tailscale.com/install.sh \| sh` |
| macOS | `brew install tailscale` 或 App Store |
| Windows | [tailscale.com/download](https://tailscale.com/download) |
| Android / iOS | 应用商店搜索 Tailscale |

安装完成后都需要执行：

```bash
sudo tailscale up
```

浏览器会弹出登录页面，用 GitHub / Google / Microsoft 账号授权即可。

---

## 2. 基本概念

| 概念 | 说明 |
|------|------|
| **tailnet** | 你的私有网络，所有设备的 `100.x.x.x` 地址都在这个网里 |
| **100.x.x.x** | Tailscale 分配的内网 IP（CGNAT 地址段），每个设备一个 |
| **MagicDNS** | 自动给每个设备分配 `<hostname>.tailad6c50.ts.net` 域名 |
| **DERP** | 打洞失败时的中转服务器（relay），全球分布 |
| **Exit Node** | 出口节点，让其他设备借用本机的网络上网 |
| **Subnet Router** | 子网路由，让 tailnet 访问物理局域网的其他设备 |

---

## 3. 日常使用

### 3.1 查看状态

```bash
tailscale status
```

输出示例：

```
100.112.53.4    server               l@github        linux   active; direct
100.98.23.17    my-arch              l@github        linux   active; direct
100.76.11.2     phone                l@github        android active; direct
```

### 3.2 直连测试

```bash
# Ping 另一台设备的 tailnet IP
tailscale ping 100.112.53.4

# 或用 hostname（需开启 MagicDNS）
tailscale ping server
```

### 3.3 通过 Tailscale 访问服务

加入 tailnet 后，直接使用 `100.x.x.x` 地址即可：

```bash
# SSH
ssh user@100.112.53.4

# Web 服务
curl http://100.112.53.4:8080

# SCP 传文件
scp file.tar.gz user@100.112.53.4:~/
```

> 不再需要公网 IP、端口映射、SSH 隧道。所有设备就像在同一个局域网。

---

## 4. MagicDNS — 用名字代替 IP

### 4.1 开启

去 [Admin Console](https://login.tailscale.com/admin/dns) → 勾选 **Enable MagicDNS**。

### 4.2 使用

开启后每个设备自动获得域名：

```bash
# 之前
ssh l@100.112.53.4

# 之后
ssh l@server.tailad6c50.ts.net
```

> 域名需要完整 FQDN（`hostname.tailnet-name.ts.net`），在 Admin Console 的 DNS 页面能看到你的 tailnet 名称。

---

## 5. Exit Node（出口节点）— 借别人的网

这是 Tailscale 最有用的功能之一：让设备 A 的所有流量经过设备 B 的出口，等于 B 把自己的网络"借"给 A。

### 5.1 声明出口节点（在**有外网访问能力**的机器上）

```bash
# 本机（有 Clash 代理、能正常上网）
sudo tailscale up --advertise-exit-node

# 开启 IP 转发
echo 'net.ipv4.ip_forward = 1' | sudo tee /etc/sysctl.d/99-tailscale.conf
echo 'net.ipv6.conf.all.forwarding = 1' | sudo tee -a /etc/sysctl.d/99-tailscale.conf
sudo sysctl --system
```

### 5.2 授权出口节点

去 [Admin Console → Machines](https://login.tailscale.com/admin/machines)，找到刚才那台机器：
- 点击 `...` → **Edit route settings**
- 勾选 **Use as exit node**

### 5.3 使用出口节点（在**需要上网**的机器上）

```bash
# 临时使用
sudo tailscale up --exit-node=<hostname 或 100.x.x.x>

# 例如
sudo tailscale up --exit-node=my-arch

# 恢复不走出口
sudo tailscale up --exit-node=
```

### 5.4 实际场景

```
场景：服务器没有公网出站，npm / pip / apt 全废了

解决：
1. 本地机器（能上网）→ 设为 Exit Node
2. 服务器 → 走本地的 Exit Node
3. 服务器现在能正常 npm install、apt update 了
```

> **注意：** 出口节点会看到你所有流量。只把自己控制的机器设为出口，不要随便连陌生人的出口。

---

## 6. Subnet Router（子网路由）— 访问物理局域网

云服务器内网有个数据库 `10.0.0.5:3306`，你在家想访问，但懒得给每台机器装 Tailscale。

### 6.1 在云服务器上宣告子网

```bash
sudo tailscale up --advertise-routes=10.0.0.0/24
```

去 Admin Console 批准这条路由。

### 6.2 本地开启子网路由

```bash
sudo tailscale up --accept-routes
```

现在你本机直接访问 `10.0.0.5:3306` 就能连到云服务器内网数据库了，无需端口映射或 SSH 隧道。

### 6.3 组合技：Exit Node + Subnet Router

```bash
# 云服务器同时做子网路由和出口
sudo tailscale up --advertise-routes=10.0.0.0/24 --advertise-exit-node
```

一台机器既暴露内网，又提供网络出口。

---

## 7. SSH 集成 — 免密码免端口

### 7.1 开启 Tailscale SSH

```bash
# 服务端
sudo tailscale up --ssh
```

去 Admin Console → **Access Controls**，确保有 SSH 规则。然后任何 tailnet 内的设备都能：

```bash
ssh <用户名>@<hostname>
```

不需要配 `~/.ssh/authorized_keys`，不需要知道端口，Tailscale 帮你搞定鉴权和加密。

### 7.2 ACL 控制谁可以 SSH

在 Admin Console → Access Controls 编辑：

```jsonc
{
  "acls": [
    // 允许所有设备互相访问
    {"action": "accept", "src": ["*"], "dst": ["*:*"]}
  ],
  "ssh": [
    {
      // 允许所有用户以 root 身份 SSH 到所有设备
      "action": "accept",
      "src": ["autogroup:member"],
      "dst": ["tag:all"],
      "users": ["root"]
    }
  ]
}
```

---

## 8. 常用命令速查

| 命令 | 说明 |
|------|------|
| `tailscale status` | 查看所有在线设备及连接方式 |
| `tailscale ip` | 查看本机 Tailscale IP |
| `tailscale ping <host>` | 测试直连延迟 |
| `tailscale netcheck` | 查看 NAT 类型和 DERP 延迟 |
| `tailscale up --ssh` | 开启 SSH 功能 |
| `tailscale up --exit-node=<host>` | 使用指定出口节点 |
| `tailscale up --exit-node=` | 停止使用出口节点 |
| `tailscale up --advertise-exit-node` | 宣告自己为出口节点 |
| `tailscale up --advertise-routes=10.0.0.0/24` | 宣告子网路由 |
| `tailscale up --accept-routes` | 接受其他设备宣告的子网路由 |
| `tailscale up --reset` | 重置所有配置 |
| `tailscale logout` | 退出登录 |
| `tailscale set --auto-update` | 开启自动更新 |
| `tailscale bugreport` | 生成诊断报告（排查问题用） |
| `tailscale file cp <file> <host>:` | 直接传文件给另一台设备 |

---

## 9. 开机自启

```bash
sudo systemctl enable --now tailscaled
```

配置（Exit Node、Subnet Router 等）是持久化的，重启后自动生效，不需要再执行 `tailscale up`。

---

## 10. 对比其他方案

| | Tailscale | WireGuard 裸配 | ZeroTier | `ssh -L/-R` |
|------|------|------|------|------|
| 配置难度 | 极低（一条命令） | 高（手动密钥分配） | 低 | 低 |
| UDP 支持 | ✅ 原生 | ✅ 原生 | ✅ | ❌ 仅 TCP |
| 打洞能力 | 极强 | 需手配 NAT | 强 | 依赖 SSH 服务端 |
| 免费额度 | 3 用户 + 100 设备 | 免费 | 25 节点 | 免费 |
| 中心依赖 | 协调服务器（开源 Headscale） | 无 | 中心控制器 | 无 |
| 文件传输 | `tailscale file cp` | 需额外工具 | 无内置 | scp |
| ACL 访问控制 | ✅ 图形化 | ❌ | ✅ | ❌ |

---

## 11. 常见问题

### Q1: 怎么知道连接是直连还是中转？

```bash
tailscale status
```

看输出中的连接类型：
- `direct` — 直连（最优）
- `relay` — 通过 DERP 中转

中转延迟一般较高，说明 NAT 太严格导致打洞失败。可以试着重启路由器、打开 UPnP。

### Q2: 不想用 Tailscale 官方的协调服务器？

用开源替代 **Headscale**：

```bash
# 自建协调服务器
# 详见 https://github.com/juanfont/headscale
```

> 自己搭 Headscale 后，`tailscale up --login-server=https://your-server.com`

### Q3: 如何传文件？

```bash
# 发送
tailscale file cp ./report.pdf server:

# 接收方会在默认下载目录收到文件
# 或用 --target-dir 指定目录
tailscale file cp ./data.json server:/home/l/projects/
```

### Q4: MagicDNS 不生效？

```bash
# 确认已开启
tailscale status | head -1

# 测试 DNS
nslookup server.tailad6c50.ts.net

# 排查 DNS 配置
cat /etc/resolv.conf
```

确保 `100.100.100.100` 在 DNS 服务器列表中（`tailscale up` 会自动写入）。

### Q5: Exit Node 不通？

检查出口节点：

```bash
# 确认 IP 转发已开启
sudo sysctl net.ipv4.ip_forward

# 确认 iptables/nftables 没有拦截转发
sudo nft list ruleset | grep -i forward
```

### Q6: 能通过 Tailscale 做游戏联机吗？

可以。Tailscale 原生支持 UDP，所有设备在同一个虚拟局域网，游戏直接通过 Tailscale IP 互联。比 SSH 隧道方案靠谱得多。

### Q7: 服务器用 Exit Node 后需要持久化配置吗？

不需要特意持久化。`tailscale up --exit-node=<host>` 执行一次就够了，重启后仍然生效。想看当前配置：

```bash
tailscale debug prefs 2>/dev/null | grep -i exit
```
