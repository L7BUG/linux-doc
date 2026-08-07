# Caddy 反向代理完全指南

## 1. 为什么选 Caddy 而不是 Nginx

Nginx 是工业标准，但个人服务器上 Caddy 有几个决定性的优势：

| | Caddy | Nginx |
|------|-------|-------|
| HTTPS 证书 | 自动申请、自动续期，零配置 | 需要 certbot + cron |
| 配置文件 | 几行搞定一个站点 | 模板冗长 |
| HTTP/3 (QUIC) | 默认开启 | 需手动编译 / 配置 |
| 内存占用 | ~10-30 MB | ~50-100 MB |
| 配置热重载 | `caddy reload` | `nginx -s reload` |

> **一句话：Caddy 让你忘记 HTTPS 证书这件事。**

---

## 2. 前置条件：域名解析

Caddy 自动申请证书的前提是**域名已经指向服务器 IP**：

```bash
# 验证
dig +short your-domain.com
# 应返回你云服务器的公网 IP
# 或用 ping
ping your-domain.com
```

至少需要：
- 一个域名（`example.com`）
- 在 DNS 管理后台将域名 A 记录指向云服务器 IP

如果需要内网服务也有 HTTPS（如 `vault.example.com`、`notes.example.com`），两种做法：
- 添加多条 A 记录指向同一个 IP
- 或添加一条泛解析：`*.example.com` → 服务器 IP

---

## 3. 安装

```bash
# Arch Linux
sudo pacman -S caddy

# Debian / Ubuntu
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' \
  | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' \
  | sudo tee /etc/apt/sources.list.d/caddy-stable.list
sudo apt update && sudo apt install caddy

# 或官方一键脚本（任意 Linux）
curl -fsSL https://getcaddy.com | bash
```

---

## 4. Caddyfile 基础

Caddy 的配置文件叫 Caddyfile，默认路径 `/etc/caddy/Caddyfile`。

### 4.1 最简静态站点

```
example.com {
    root * /var/www/html
    file_server
}
```

三行：域名、网站根目录、启用文件服务。HTTPS 自动搞定。

### 4.2 反向代理

```
files.example.com {
    reverse_proxy localhost:8080
}
```

这就是一个完整的反向代理——域名 `files.example.com` 的请求自动获得 HTTPS 证书，然后转发到本地 `8080` 端口。

### 4.3 多站点

```
# 网盘
files.example.com {
    reverse_proxy localhost:8080
}

# 密码管理器
vault.example.com {
    reverse_proxy localhost:8222
}

# 代码仓库
git.example.com {
    reverse_proxy localhost:3000
}

# 监控面板
status.example.com {
    reverse_proxy localhost:3001
}
```

每个子域名自动申请独立证书，互不影响。

---

## 5. 常用配置模式

### 5.1 WebSocket 支持

很多自建服务（code-server、Uptime Kuma、Jellyfin）依赖 WebSocket：

```
code.example.com {
    reverse_proxy localhost:8443 {
        header_up X-Forwarded-For {remote_host}
        header_up X-Forwarded-Proto {scheme}
    }
}
```

Caddy 默认支持 WebSocket 的 `Upgrade` 头，不需要像 Nginx 那样额外配置 `proxy_set_header Upgrade`。上面额外加的 `X-Forwarded-For` 和 `X-Forwarded-Proto` 用于让后端服务知道真实来源 IP 和协议。

### 5.2 路径重写与子目录代理

服务跑在子路径时（如 `localhost:8080/app`）：

```
example.com {
    handle_path /app/* {
        reverse_proxy localhost:8080
    }
}
```

`handle_path` 会自动去掉前缀 `/app`，所以请求 `example.com/app/api` → `localhost:8080/api`。

### 5.3 自定义响应头

```
api.example.com {
    reverse_proxy localhost:3000
    header {
        X-Content-Type-Options nosniff
        X-Frame-Options DENY
        Referrer-Policy strict-origin-when-cross-origin
    }
}
```

### 5.4 访问控制

```
# 仅允许指定 IP 访问
admin.example.com {
    @allowed remote_ip 你的家庭公网IP
    handle @allowed {
        reverse_proxy localhost:9090
    }
    handle {
        respond "Forbidden" 403
    }
}
```

### 5.5 Basic Auth 快速加锁

```
# 给内部工具加一道密码
internal.example.com {
    basicauth {
        admin $2a$14$xxx
    }
    reverse_proxy localhost:5000
}
```

生成密码哈希：

```bash
caddy hash-password
# 输入密码，复制输出的哈希值
```

---

## 6. 泛域名证书（通配符）

如果有很多子域名服务，逐个申请证书太麻烦。泛域名一张证书覆盖 `*.example.com` 和 `example.com`。

但 Let's Encrypt 泛域名证书**必须用 DNS-01 验证**（HTTP-01 不支持通配符），需要 Caddy 能操作你的 DNS 记录。

### 6.1 安装 DNS 插件

以 Cloudflare 为例：

```bash
# 如果用官方脚本安装，构建时指定插件
curl -fsSL https://getcaddy.com | bash -s personal \
    github.com/caddy-dns/cloudflare

# Arch Linux 的 caddy 包不含 DNS 插件，
# 需要安装 caddy-with-plugins 或手动编译
```

> **建议**：DNS 托管在 Cloudflare（免费），配合 `caddy-dns/cloudflare` 插件。

### 6.2 获取 DNS API Token

以 Cloudflare 为例：

1. 登录 Cloudflare → 点击域名 → 获取 Zone ID
2. 右上角头像 → My Profile → API Tokens → Create Token
3. 选「Edit zone DNS」模板，权限选 Zone:DNS:Edit
4. 记下 Token

### 6.3 配置泛域名证书

```
# Caddyfile
*.example.com, example.com {
    tls {
        dns cloudflare {env.CF_API_TOKEN}
    }

    @files host files.example.com
    handle @files {
        reverse_proxy localhost:8080
    }

    @vault host vault.example.com
    handle @vault {
        reverse_proxy localhost:8222
    }

    @git host git.example.com
    handle @git {
        reverse_proxy localhost:3000
    }

    # 兜底：未匹配的子域名
    handle {
        respond "No service here" 404
    }
}
```

环境变量设置（`/etc/systemd/system/caddy.service.d/override.conf`）：

```ini
[Service]
Environment=CF_API_TOKEN=你的Cloudflare_Token
```

```bash
sudo systemctl daemon-reload
sudo systemctl restart caddy
```

### 6.4 其他 DNS 服务商

| DNS 服务商 | Caddy 插件 |
|-----------|-----------|
| Cloudflare | `github.com/caddy-dns/cloudflare` |
| 阿里云 DNS | `github.com/caddy-dns/alidns` |
| 腾讯云 DNSPod | `github.com/caddy-dns/dnspod` |
| Route53 | `github.com/caddy-dns/route53` |

---

## 7. 内网服务映射实战：完整 Caddyfile 示例

假设你搭了这些服务，全部通过 Caddy 统一暴露：

```
# /etc/caddy/Caddyfile

# ===== 泛域名 =====
*.internal.example.com, internal.example.com {
    tls {
        dns cloudflare {env.CF_API_TOKEN}
    }

    # 密码管理器
    @vault host vault.internal.example.com
    handle @vault {
        reverse_proxy localhost:8222
    }

    # 网盘
    @files host files.internal.example.com
    handle @files {
        reverse_proxy localhost:8080
    }

    # 监控面板
    @status host status.internal.example.com
    handle @status {
        reverse_proxy localhost:3001
    }

    # AdGuard Home
    @adguard host adguard.internal.example.com
    handle @adguard {
        reverse_proxy localhost:8081
    }
}

# ===== 需要外网访问的静态站点 =====
blog.example.com {
    root * /var/www/blog
    file_server
    encode gzip zstd
}
```

以后加新服务只需加一个 `@xxx` + `handle` 块。

---

## 8. 日志与调试

### 8.1 启用访问日志

```
example.com {
    reverse_proxy localhost:3000
    log {
        output file /var/log/caddy/access.log
        format json
    }
}
```

JSON 格式方便用 `jq` 分析或用 Loki 聚合。

### 8.2 测试配置

```bash
# 检查语法
caddy validate --config /etc/caddy/Caddyfile

# 格式化
caddy fmt --overwrite /etc/caddy/Caddyfile

# 重载（不中断服务）
caddy reload --config /etc/caddy/Caddyfile
```

### 8.3 调试证书问题

```bash
# 查看已管理的证书
caddy list-certificates

# 查看某张证书详情
caddy list-certificates --domain example.com

# 手动续期
caddy renew
```

---

## 9. 运维要点

### 9.1 证书自动续期

Caddy 在证书到期前 30 天自动续期，无需手动干预。检查状态：

```bash
sudo journalctl -u caddy --since "1 day ago" | grep -i cert
```

### 9.2 防火墙

```bash
sudo ufw allow 80/tcp    # HTTP（用于证书验证 + 自动跳转 HTTPS）
sudo ufw allow 443/tcp   # HTTPS
sudo ufw allow 443/udp   # HTTP/3 (QUIC)
```

### 9.3 自动启动

```bash
sudo systemctl enable --now caddy
```

### 9.4 备份

```bash
# Caddyfile + 证书数据
tar czf caddy-backup-$(date +%Y%m%d).tar.gz \
  /etc/caddy/Caddyfile \
  /var/lib/caddy/
```

---

## 10. 常见问题

### Q1: "too many certificates already issued" 报错？

Let's Encrypt 有速率限制：同一域名每周最多 5 张重复证书。刚折腾配置时容易触发。

**解决**：先用 `caddy validate` 确认配置无误再启动，避免反复重载触发申请。也可以在测试阶段临时用自签证书：

```
example.com {
    tls internal   # 自签证书，不调用 Let's Encrypt
    reverse_proxy localhost:3000
}
```

确认全部调通后去掉 `tls internal` 改回自动。

### Q2: 服务器在内网（没有公网 80/443），能用 Caddy 吗？

可以，用 DNS-01 验证（见第 6 节）。DNS 验证不要求服务器对外暴露端口，只要 Caddy 能调用 DNS API 加一条 TXT 记录即可。

但证书到手后，用户还是需要能访问服务器——所以最终 HTTPS 端口（443）仍需要可达。

### Q3: Caddy 和 Nginx 能并存吗？

可以，但没必要。如果确实有遗留 Nginx 配置不想迁移，让 Caddy 做 HTTPS 终止：

```
example.com {
    reverse_proxy localhost:8443 {
        transport http {
            tls_insecure_skip_verify
        }
    }
}
```

不过多了一层跳转，建议直接迁移到 Caddy。

### Q4: 怎么让局域网设备也信任 Caddy 的证书？

泛域名公网证书在局域网里直接用，前提是局域网 DNS 能解析域名到内网 IP。

以 AdGuard Home 为例，添加 DNS 重写规则：

```
files.internal.example.com → 192.168.1.100（服务器内网 IP）
```

客户端 DNS 指向 AdGuard Home 后，访问 `files.internal.example.com` 走内网直连 + 公网有效证书，浏览器不会弹不安全警告。

### Q5: 多个后端服务在同一端口但不同路径怎么配？

```
example.com {
    handle_path /api/* {
        reverse_proxy localhost:3001
    }
    handle_path /admin/* {
        reverse_proxy localhost:3002
    }
    handle {
        reverse_proxy localhost:3000    # 前端
    }
}
```

`handle_path` 会自动去掉匹配的前缀再转发，`handle` 是兜底。

---

## 11. 参考资源

- [Caddy 官方文档](https://caddyserver.com/docs/)
- [Caddyfile 指令参考](https://caddyserver.com/docs/caddyfile/directives)
- [Caddy DNS 插件列表](https://caddyserver.com/docs/modules/)
- [Let's Encrypt 速率限制](https://letsencrypt.org/docs/rate-limits/)
