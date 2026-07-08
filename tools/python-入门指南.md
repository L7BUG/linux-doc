# Python 入门指南

## 1. 什么是 Python？

**Python** 是 Guido van Rossum 于 1991 年发布的一门解释型、动态类型的高级语言。设计哲学是**可读性优先**——代码读起来像伪代码一样自然。

> **一句话总结：** 当你需要快速写个工具操作文件、处理数据、发送邮件、爬取网页、或做任何「不想为语言本身花太多精力」的事情时，Python 通常是最合适的选择。

### 1.1 为什么在 Linux 下学 Python？

| 原因 | 说明 |
|------|------|
| **系统自带** | 几乎所有 Linux 发行版都预装了 Python 3 |
| **标准库极全** | 文件操作、邮件、压缩、JSON、CSV、正则——全在标准库里 |
| **零项目结构** | 不需要 Maven/Gradle/pom.xml，一个 `.py` 文件就能跑 |
| **互操作好** | 调用 shell 命令、管道处理、systemd 集成都顺手 |
| **动嘴不动手** | 写个小工具改文件名、批量发邮件，几分钟就完事 |

### 1.2 Python 3 还是 Python 2？

**Python 2 已于 2020 年 1 月 1 日停止维护。** 无论看到什么老教程，一律用 Python 3。

```bash
# 确认系统 Python 版本
python3 --version
# Python 3.12.x（或更高）
```

---

## 2. 安装与环境

### 2.1 Arch Linux 安装

```bash
# Python 3 本身（系统已自带）
sudo pacman -S python

# pip — Python 包管理器
sudo pacman -S python-pip

# 常用开发工具（可选）
sudo pacman -S python-pipx    # 隔离安装 CLI 工具（推荐）
sudo pacman -S python-ipython # 增强交互式终端
```

### 2.2 pip 基本用法

```bash
# 安装包（用户级，不污染系统）
pip install --user <包名>

# 卸载
pip uninstall <包名>

# 列出已安装
pip list

# 查看包信息
pip show <包名>

# 从 requirements.txt 批量安装
pip install -r requirements.txt
```

> **注意：** Arch Linux 中系统级 `pip install` 可能与 pacman 冲突。优先用 `--user`（安装到 `~/.local`），或用 `pipx` 安装 CLI 工具。

### 2.3 虚拟环境（venv）

每个项目应有独立的 Python 环境，避免依赖冲突：

```bash
# 创建虚拟环境
python3 -m venv myproject/

# 激活
source myproject/bin/activate

# 现在 pip install 只影响这个项目
pip install requests

# 退出
deactivate
```

> **提示：** 临时小脚本不需要虚拟环境，用系统 Python 即可。这是为「正经项目」准备的。

---

## 3. 基础语法速览

> 如果你有其他语言经验，这一节就能让你开始写了。

### 3.1 变量与类型

```python
# 变量——不需要声明类型
name = "Linux"
count = 42
price = 3.14
is_arch = True

# 类型检查
print(type(name))   # <class 'str'>
print(type(count))  # <class 'int'>

# 字符串拼接
greeting = f"Hello, {name}!"  # f-string（Python 3.6+，推荐）
print(greeting)  # Hello, Linux!
```

### 3.2 三大数据结构

```python
# 列表（list）—— 有序、可变
pkgs = ["linux", "systemd", "bash"]
pkgs.append("vim")
print(pkgs[0])      # linux
print(pkgs[-1])     # vim（倒数第一个）
print(pkgs[1:3])    # ['systemd', 'bash']（切片）

# 字典（dict）—— 键值对
info = {
    "distro": "Arch",
    "kernel": "6.12",
    "wm": "KDE"
}
print(info["distro"])  # Arch
for k, v in info.items():
    print(f"{k}: {v}")

# 集合（set）—— 无序、不重复
installed = {"bash", "vim", "git"}
wanted = {"git", "zsh", "neovim"}
print(installed & wanted)   # {'git'}（交集）
print(installed | wanted)   # 并集
print(installed - wanted)   # 差集：{'bash', 'vim'}
```

### 3.3 控制流

```python
# if/elif/else
size = 1024
if size > 1024 * 1024:
    unit = "MB"
elif size > 1024:
    unit = "KB"
else:
    unit = "B"

# for 循环（遍历一切可迭代对象）
for pkg in ["linux", "systemd", "bash"]:
    print(pkg.upper())

# while 循环
count = 3
while count > 0:
    print(count)
    count -= 1

# 列表推导式（Python 特色）
squares = [x**2 for x in range(10)]  # [0, 1, 4, 9, ..., 81]
evens = [x for x in range(20) if x % 2 == 0]
```

### 3.4 函数

```python
def greet(name, greeting="你好"):
    """返回问候语（这是文档字符串）"""
    return f"{greeting}, {name}！"

print(greet("Arch"))          # 你好, Arch！
print(greet("Btrfs", "👋"))  # 👋, Btrfs！
```

### 3.5 异常处理

```python
# Python 没有受检异常——你想 try 就 try
try:
    result = 10 / 0
except ZeroDivisionError as e:
    print(f"出错: {e}")
else:
    print(f"结果: {result}")  # 没异常才执行
finally:
    print("清理工作")          # 无论如何都执行
```

---

## 4. 文件与目录操作（重点）

这是写小工具最常用的部分。

### 4.1 pathlib — 现代文件路径处理

`pathlib` 是 Python 3.4 引入的面向对象路径库。**忘掉 `os.path.join`，用这个。**

```python
from pathlib import Path

# 路径对象
home = Path.home()           # /home/l
dl = home / "Downloads"      # / 运算符拼接路径
config = home / ".config" / "myapp"

# 基本属性
print(dl.name)               # Downloads
print(dl.parent)             # /home/l
print(dl.exists())           # True/False
print(dl.is_dir())           # True/False

# 遍历目录 —— rglob 是神器
# 递归找所有 .pdf 文件
for pdf in dl.rglob("*.pdf"):
    print(pdf)               # 每个 pdf 的完整路径

# 只看当前目录（不递归）
for f in dl.glob("*.log"):
    print(f)

# 创建目录
new_dir = home / "my-tools"
new_dir.mkdir(exist_ok=True)  # exist_ok=True 表示已存在也不报错

# 读取和写入文件
readme = new_dir / "README.md"
readme.write_text("# Hello\n这是内容")      # 一次性写入
content = readme.read_text()                # 一次性读出
print(content)

# 按行读取
for line in readme.open():
    print(line.strip())
```

### 4.2 shutil — 高级文件操作

```python
import shutil
from pathlib import Path

src = Path("/path/to/source")
dst = Path("/path/to/destination")

# 复制文件（保留权限）
shutil.copy2(src / "config.ini", dst / "config.ini")

# 复制整个目录树
shutil.copytree(src, dst / "backup", dirs_exist_ok=True)

# 移动文件/目录
shutil.move(src / "old.log", dst / "archive/old.log")

# 删除整个目录树
shutil.rmtree(src / "temp")

# 磁盘使用情况
usage = shutil.disk_usage(Path.home())
print(f"总量: {usage.total / 1024**3:.1f} GB")
print(f"已用: {usage.used / 1024**3:.1f} GB")
print(f"可用: {usage.free / 1024**3:.1f} GB")
```

### 4.3 实用场景示例

**场景一：清理超过 30 天的下载文件**

```python
import time
from pathlib import Path

THIRTY_DAYS = 30 * 86400  # 秒
now = time.time()

for f in Path.home().joinpath("Downloads").glob("*"):
    if not f.is_file():
        continue
    age = now - f.stat().st_mtime
    if age > THIRTY_DAYS:
        print(f"删除: {f.name}（{age // 86400:.0f} 天前）")
        f.unlink()  # 删除文件
```

**场景二：按扩展名整理文件**

```python
from pathlib import Path

downloads = Path.home() / "Downloads"

for f in downloads.iterdir():
    if not f.is_file():
        continue

    # 按后缀分到子文件夹
    ext = f.suffix.lstrip(".") or "无后缀"
    target = downloads / ext
    target.mkdir(exist_ok=True)
    f.rename(target / f.name)

print("整理完成！")
```

**场景三：批量重命名**

```python
from pathlib import Path

# 将某个目录下所有 .jpeg 改为 .jpg
for f in Path("/path/to/photos").glob("*.jpeg"):
    new_name = f.with_suffix(".jpg")
    f.rename(new_name)
    print(f"{f.name} → {new_name.name}")
```

---

## 5. 邮件处理

### 5.1 读取并删除邮件（IMAP）

```python
import imaplib
import email
from email.header import decode_header

# 登录
mail = imaplib.IMAP4_SSL("imap.gmail.com")
mail.login("yourname@gmail.com", "应用专用密码")
mail.select("INBOX")

# 搜索所有邮件
status, messages = mail.search(None, "ALL")
if status != "OK":
    print("搜索失败")
    mail.logout()
    exit(1)

ids = messages[0].split()
print(f"共 {len(ids)} 封邮件")

# 按主题关键字删除
KEYWORDS = ["垃圾广告", "unsubscribe", "促销"]

for num in ids:
    _, data = mail.fetch(num, "(RFC822)")
    msg = email.message_from_bytes(data[0][1])

    # 解码主题（需处理编码）
    subject_raw = msg["Subject"] or ""
    subject_parts = decode_header(subject_raw)
    subject = ""
    for part, charset in subject_parts:
        if isinstance(part, bytes):
            subject += part.decode(charset or "utf-8", errors="ignore")
        else:
            subject += part

    # 匹配关键字
    if any(kw.lower() in subject.lower() for kw in KEYWORDS):
        mail.store(num, "+FLAGS", "\\Deleted")
        print(f"已标记删除: {subject[:50]}")

# 真正删除
mail.expunge()
mail.logout()
print("完成！")
```

> **Gmail 用户注意：** 需要先开启两步验证，然后生成「应用专用密码」，IMAP 登录时用这个密码而非你的 Gmail 密码。

### 5.2 发送邮件（SMTP）

```python
import smtplib
from email.mime.text import MIMEText
from email.mime.multipart import MIMEMultipart

msg = MIMEMultipart()
msg["From"] = "sender@gmail.com"
msg["To"] = "receiver@example.com"
msg["Subject"] = "Python 测试邮件"

body = "这是一封来自 Python 脚本的测试邮件。"
msg.attach(MIMEText(body, "plain", "utf-8"))

with smtplib.SMTP_SSL("smtp.gmail.com", 465) as server:
    server.login("sender@gmail.com", "应用专用密码")
    server.send_message(msg)

print("发送成功！")
```

---

## 6. 命令行工具

### 6.1 argparse — 标准命令行参数解析

```python
import argparse
from pathlib import Path

parser = argparse.ArgumentParser(description="清理过期文件")
parser.add_argument("directory", type=Path, help="要清理的目录")
parser.add_argument("--days", "-d", type=int, default=30, help="保留天数（默认 30）")
parser.add_argument("--dry-run", action="store_true", help="仅显示，不实际删除")
parser.add_argument("--pattern", default="*", help="匹配模式（默认所有文件）")

args = parser.parse_args()

if not args.directory.exists():
    print(f"错误：目录 {args.directory} 不存在")
    exit(1)

for f in args.directory.glob(args.pattern):
    if args.dry_run:
        print(f"[模拟] 将删除: {f}")
    else:
        f.unlink()
        print(f"已删除: {f}")
```

```bash
# 使用方式
python3 cleaner.py ~/Downloads --days 60 --dry-run
python3 cleaner.py ~/Downloads --pattern "*.log" -d 7
```

### 6.2 让脚本变成"命令"

```bash
# 1. 在脚本第一行加 shebang
#    #!/usr/bin/env python3

# 2. 添加可执行权限
chmod +x cleaner.py

# 3. 移动到 PATH 中
mv cleaner.py ~/.local/bin/cleaner

# 现在可以在任何地方直接调用
cleaner ~/Downloads --days 60
```

---

## 7. 常用标准库速查

| 库 | 用途 | 示例 |
|----|------|------|
| `pathlib` | 现代文件路径操作 | `Path.home() / "dir" / "file.txt"` |
| `shutil` | 高级文件操作 | `shutil.copy2()`, `shutil.rmtree()` |
| `os` | 底层系统调用 | `os.environ`, `os.getcwd()` |
| `subprocess` | 运行外部命令 | `subprocess.run(["ls", "-la"])` |
| `re` | 正则表达式 | `re.findall(r"\d+", text)` |
| `json` | JSON 处理 | `json.load(f)`, `json.dump(d, f)` |
| `csv` | CSV 读写 | `csv.reader(f)`, `csv.DictWriter(f)` |
| `imaplib` | IMAP 邮件读取 | `mail.search(None, "ALL")` |
| `smtplib` | SMTP 邮件发送 | `server.send_message(msg)` |
| `zipfile` | ZIP 压缩解压 | `zipfile.ZipFile("a.zip").extractall()` |
| `tarfile` | tar 归档操作 | `tarfile.open("a.tar.zst").extractall()` |
| `argparse` | 命令行参数 | `parser.add_argument("--verbose")` |
| `datetime` | 日期时间 | `datetime.now()`, `timedelta(days=30)` |
| `sqlite3` | 轻量数据库 | `conn.execute("SELECT * FROM t")` |
| `configparser` | INI 配置文件 | `config.read("settings.ini")` |
| `logging` | 日志记录 | `logging.info("程序启动")` |
| `hashlib` | 哈希校验 | `hashlib.sha256(data).hexdigest()` |

---

## 8. 推荐第三方库

标准库不够用时，这些是「大家都在用」的选择：

| 库 | 用途 | 安装 |
|----|------|------|
| `requests` | HTTP 请求（比 urllib 好用 10 倍） | `pip install --user requests` |
| `rich` | 终端美化（彩色输出、进度条、表格） | `pip install --user rich` |
| `click` | 更优雅的命令行参数 | `pip install --user click` |
| `httpx` | 现代 HTTP 客户端（支持异步） | `pip install --user httpx` |
| `pydantic` | 数据校验（类型驱动） | `pip install --user pydantic` |
| `watchdog` | 监视文件系统变化 | `pip install --user watchdog` |
| `pillow` | 图片处理 | `pip install --user Pillow` |
| `tqdm` | 进度条 | `pip install --user tqdm` |
| `python-dotenv` | 从 `.env` 文件加载环境变量 | `pip install --user python-dotenv` |

### 8.1 实用示例：用 rich 美化输出

```python
from rich.console import Console
from rich.table import Table
from rich.progress import track
from pathlib import Path
import time

console = Console()

# 彩色打印
console.print("[green]成功[/green] 处理完成")
console.print("[red]错误[/red] 文件不存在")

# 表格
table = Table(title="磁盘使用情况")
table.add_column("分区")
table.add_column("已用")
table.add_column("可用")
table.add_row("/", "120 GB", "380 GB")
table.add_row("/home", "340 GB", "160 GB")
console.print(table)

# 带进度条的操作
for f in track(list(Path.home().joinpath("Downloads").glob("*")),
               description="处理文件中..."):
    time.sleep(0.01)  # 模拟耗时操作
```

---

## 9. 学习路径建议

```
第一阶段（1-2 周）
├── 变量、类型、控制流、函数
├── list / dict / set 三大结构
└── 文件读写（pathlib）

第二阶段（1 周）
├── shutil 文件操作（复制、移动、删除）
├── argparse 命令行参数
├── 异常处理
└── 虚拟环境（venv）与 pip

第三阶段（边用边学）
├── 邮件处理（imaplib / smtplib）
├── subprocess 调用外部命令
├── 正则表达式（re）
├── JSON / CSV 数据处理
└── 按需学习第三方库（requests、rich、click）
```

---

## 10. Python vs 其他语言：小工具场景对比

| 维度 | Python | Bash | Go | Java | Rust |
|------|--------|------|----|------|------|
| 启动速度 | ⚠️ 中等 | ✅ 即时 | ✅ 二进制 | ❌ JVM 启动 | ✅ 二进制 |
| 代码量 | ✅ 极少 | ✅ 极少 | ⚠️ 中等 | ❌ 啰嗦 | ⚠️ 中等 |
| 文件操作 | ✅ 顺滑 | ⚠️ 特殊字符噩梦 | ⚠️ 尚可 | ⚠️ 尚可 | ⚠️ 严格 |
| 字符串处理 | ✅ 享受 | ❌ 痛苦 | ✅ 不错 | ⚠️ 还行 | ⚠️ 严格 |
| 邮件处理 | ✅ 标准库 | ❌ 难 | ✅ 标准库 | ⚠️ 第三方 | ⚠️ 第三方 |
| JSON/CSV | ✅ 标准库 | ❌ 需 jq | ✅ 标准库 | ⚠️ 啰嗦 | ⚠️ serde |
| 依赖管理 | ✅ pip | — | ✅ go mod | ⚠️ Maven | ✅ cargo |
| 分发给别人 | ❌ 需安装 Python | ✅ 系统自带 | ✅ 单文件 | ❌ 需 JVM | ✅ 单文件 |
| 适合场景 | 个人工具、自动化 | 简单胶水脚本 | 需分发的 CLI | 企业应用 | 系统软件 |

> **结论：** 给自己写工具 → Python。要给没有 Python 的人用 → Go（编译成单文件）。写内核驱动 → Rust。别用 Java 写小工具。

---

## 11. 常见问题

### Q1: 我的脚本报 `PermissionError` 怎么办？

```python
# 检查文件是否存在且你有权限
from pathlib import Path
f = Path("/etc/some-config")
print(f.exists())   # True/False
print(f.is_file())  # True/False
# 如果返回 False，检查路径拼写

# 需要权限的操作在执行时提权
# sudo python3 my_script.py
# （脚本内不要写 sudo，在运行时提权）
```

### Q2: `pip install` 报 `externally-managed-environment` 错误？

Arch Linux 的保护机制。解决方式三选一：

```bash
# 1. 用户级安装（推荐）
pip install --user <包名>

# 2. 用 pipx 安装 CLI 工具
pipx install <工具名>

# 3. 创建虚拟环境
python3 -m venv myenv && source myenv/bin/activate
```

### Q3: `Path.rglob()` 和 `Path.glob()` 有什么区别？

```python
# glob()     —— 仅当前目录
# rglob()    —— 递归所有子目录（r = recursive）

# 只找 Downloads 下的 pdf
Path.home().joinpath("Downloads").glob("*.pdf")

# 找 Downloads 下所有子目录中的 pdf
Path.home().joinpath("Downloads").rglob("*.pdf")
```

### Q4: Python 脚本能和 shell 命令混用吗？

```python
import subprocess

# 运行命令，获取输出
result = subprocess.run(
    ["pacman", "-Q", "linux"],
    capture_output=True, text=True
)
print(result.stdout)  # linux 6.12.1-arch1-1

# 管道逻辑也可以用 Python 替代
# 不用: find ~/Downloads -name "*.log" -mtime +30 | xargs rm
# 而用:
from pathlib import Path
import time

for f in Path.home().joinpath("Downloads").glob("*.log"):
    if f.stat().st_mtime < time.time() - 30 * 86400:
        f.unlink()
```

### Q5: 我需要懂面向对象才能写 Python 脚本吗？

**不需要。** 写小工具用函数就够了。但 `pathlib.Path` 本身就是对象，你已经在用了：

```python
p = Path("/tmp/test")     # p 是 Path 类的实例
p.mkdir(exist_ok=True)    # 调用它的方法
```

面向对象是组织大型项目的工具，不是写脚本的前提。

### Q6: 代码乱码怎么办？

Python 3 默认就是 UTF-8。如果遇到编码问题，检查终端 locale：

```bash
echo $LANG          # 应该是 zh_CN.UTF-8 或 en_US.UTF-8
localectl status    # 查看系统 locale
```

也可以脚本里显式声明（通常不需要）：

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
```

---

## 参考

- [Python 官方教程](https://docs.python.org/zh-cn/3/tutorial/) — 零基础友好，有中文版
- [Python 标准库文档](https://docs.python.org/zh-cn/3/library/) — 所有自带模块
- [Automate the Boring Stuff with Python](https://automatetheboringstuff.com/) — 自动化小工具实战，免费在线阅读
- [Arch Wiki — Python](https://wiki.archlinux.org/title/Python)
- [pathlib 官方文档](https://docs.python.org/zh-cn/3/library/pathlib.html)
