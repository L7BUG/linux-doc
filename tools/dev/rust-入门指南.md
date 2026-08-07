# Rust 入门指南

## 1. 什么是 Rust？

**Rust** 是 Mozilla Research 于 2010 年首次发布、由 Graydon Hoare 最初设计的系统级编程语言，2015 年发布 1.0 稳定版。

> **一句话总结：** 在零开销抽象的前提下，提供 C/C++ 级别的性能和控制力，但通过编译期检查杜绝内存安全漏洞和数据竞争——无需垃圾回收器。

2021 年 Rust 基金会成立（AWS、Google、华为、Microsoft、Mozilla 等联合发起），2022 年 Rust 正式合入 Linux 内核（6.1），2026 年 Linux 7.0 中 Rust 晋升为稳定的一等语言。

### 1.1 为什么在 Linux 下学 Rust？

| 原因 | 说明 |
|------|------|
| **内存安全** | 编译期杜绝 use-after-free、double-free、空指针、缓冲区溢出 |
| **线程安全** | 编译期杜绝数据竞争，无需运行时检查 |
| **零开销** | 无 GC、无运行时，性能与 C/C++ 同级 |
| **现代工具链** | Cargo 构建系统、内置测试、自动格式化、文档生成 |
| **内核级应用** | Linux 内核正式采用，驱动开发的新标准 |
| **生态成熟** | crates.io 超 15 万包，覆盖 CLI、Web、嵌入式、数据库 |

---

## 2. 安装与环境

### 2.1 Arch Linux 安装

```bash
# 通过 rustup 安装（推荐——官方版本管理器）
sudo pacman -S rustup
rustup default stable

# 确认安装
rustc --version    # rustc 1.x.x
cargo --version    # cargo 1.x.x

# 安装额外工具
rustup component add rustfmt      # 自动格式化
rustup component add clippy       # 静态分析 linter
rustup component add rust-analyzer # LSP 服务器（IDE 支持）
```

> **为什么不直接用 pacman 装 rust？** `rustup` 可以管理多个 Rust 版本（stable/beta/nightly）、切换工具链、交叉编译目标。pacman 只能装一个版本。

### 2.2 rustup 常用命令

```bash
rustup update              # 更新 Rust
rustup default stable      # 切换到稳定版
rustup default nightly     # 切换到 nightly（需要时）
rustup show                # 查看当前工具链
rustup target add x86_64-unknown-linux-musl  # 添加交叉编译目标
```

### 2.3 Cargo 基本用法

```bash
cargo new my_project       # 创建新项目（生成目录结构）
cargo init                 # 在已有目录中初始化
cargo build                # 编译（debug 模式）
cargo build --release      # 编译（release 模式，优化全开）
cargo run                  # 编译并运行
cargo test                 # 运行测试
cargo fmt                  # 自动格式化代码
cargo clippy               # 静态分析
cargo doc --open           # 生成文档并在浏览器打开
cargo check                # 检查代码能否通过编译（不生成二进制，速度快）
```

---

## 3. 基础语法速览

> 如果你有 C/C++/Python 经验，这一节能让你开始写 Rust。

### 3.1 变量与可变性

```rust
fn main() {
    // 变量默认不可变
    let x = 5;
    // x = 6;              // ❌ 编译错误！默认不可变

    // 显式声明可变
    let mut y = 5;
    y = 6;                 // ✅

    // 常量（必须标注类型，编译期求值）
    const MAX_POINTS: u32 = 100_000;

    // 变量遮蔽（shadowing）——可以改变类型
    let spaces = "   ";
    let spaces = spaces.len();  // 同名变量，类型从 &str 变成 usize
}
```

### 3.2 基本类型

```rust
// 整数（默认 i32）
let a: i8 = -128;         // 有符号，8 位（-128 ~ 127）
let b: u32 = 42;          // 无符号，32 位
let c: usize = 100;       // 指针大小（64 位系统上即 64 位）

// 浮点数（默认 f64）
let x: f32 = 3.14;
let y = 2.71828;          // 推断为 f64

// 布尔值
let t: bool = true;

// 字符（Unicode 标量值，4 字节）
let heart_eyed_cat: char = '😻';

// 元组（不同类型组合）
let tup: (i32, f64, char) = (42, 6.28, 'A');
let (a, b, c) = tup;      // 解构
println!("{a} {b} {c}");  // 42 6.28 A
println!("{}", tup.0);    // 通过索引访问：42

// 数组（固定长度，栈分配）
let arr: [i32; 5] = [1, 2, 3, 4, 5];
let zeros = [0; 100];     // [0, 0, ..., 0] —— 100 个 0
println!("{}", arr[2]);   // 3
// 越界访问在运行时 panic，不会出现缓冲区溢出
```

### 3.3 函数

```rust
// 最后一个表达式隐含 return（不加分号）
fn add(x: i32, y: i32) -> i32 {
    x + y  // 注意：没有分号 = 返回值
}

// 显式 return（加上分号）
fn max(x: i32, y: i32) -> i32 {
    if x > y {
        return x;
    }
    y
}

// 单元类型 ()
fn print_msg(msg: &str) {
    println!("{msg}");
}
```

### 3.4 控制流

```rust
// if 是表达式，可以赋值
let condition = true;
let number = if condition { 5 } else { 6 };
// 注意：两个分支必须返回相同类型

// loop —— 无限循环（可以返回值）
let mut counter = 0;
let result = loop {
    counter += 1;
    if counter == 10 {
        break counter * 2;  // break 带值 = loop 的返回值
    }
};
println!("{result}");  // 20

// while
let mut n = 3;
while n > 0 {
    println!("{n}");
    n -= 1;
}

// for —— 遍历集合的首选
let a = [10, 20, 30, 40, 50];
for element in a {
    println!("{element}");
}

// Range
for i in 0..5 {       // 0, 1, 2, 3, 4（不含 5）
    println!("{i}");
}
for i in 0..=5 {      // 0, 1, 2, 3, 4, 5（含 5）
    println!("{i}");
}
```

---

## 4. 所有权 —— Rust 的基石

> 这是 Rust 最独特、最重要的概念。理解它，Rust 就掌握了一半。

### 4.1 三条所有权规则

```rust
// 规则 1：Rust 中每个值都有一个所有者（owner）
// 规则 2：同一时刻只能有一个所有者
// 规则 3：当所有者离开作用域，值被自动释放（drop）

fn main() {
    let s1 = String::from("hello");  // s1 是所有者
    let s2 = s1;                      // 所有权移动（move）到 s2
    // println!("{s1}");              // ❌ s1 已失效！

    let s3 = s2.clone();              // 深拷贝（显式）
    println!("{s2} {s3}");           // ✅ 两个独立的值
}
```

### 4.2 借用（Borrowing）—— 用而不取

```rust
fn main() {
    let mut s = String::from("hello");

    // 不可变借用 —— 可以有多个
    let r1 = &s;
    let r2 = &s;
    println!("{r1} {r2}");

    // 可变借用 —— 有且仅有一个，且不能同时有不可变借用
    let r3 = &mut s;
    r3.push_str(" world");
    // println!("{r1}");  // ❌ r3 存在时 r1 不可用
}
```

**这套规则在编译期就杜绝了：**
- Use-after-free（释放后使用）
- Double-free（双重释放）
- 数据竞争（Data race）
- 空指针解引用（Rust 根本没有 null）
- 缓冲区溢出（数组访问有边界检查）

### 4.3 生命周期（Lifetimes）

```rust
// 'a 是生命周期标注：返回值生命周期与较短的那个参数相同
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() { x } else { y }
}

fn main() {
    let s1 = String::from("short");
    let s2 = String::from("loooooong");
    let result = longest(&s1, &s2);
    println!("{result}");  // loooooong
}
```

生命周期参数让编译器能**静态验证所有引用不会悬垂**——这就是 Rust 不需要 GC 的关键。

---

## 5. 枚举与模式匹配

### 5.1 Option —— 没有 null 的世界

```rust
// Rust 没有 null！用 Option<T> 表示"可能有值"
enum Option<T> {
    Some(T),
    None,
}

fn divide(x: f64, y: f64) -> Option<f64> {
    if y == 0.0 {
        None
    } else {
        Some(x / y)
    }
}

fn main() {
    match divide(10.0, 2.0) {
        Some(v) => println!("结果: {v}"),
        None    => println!("除零错误"),
    }
    // 编译器强制处理所有情况 —— 不会漏掉 None
}
```

### 5.2 Result —— 无异常的错误处理

```rust
use std::fs::File;
use std::io::Read;

fn read_file(path: &str) -> Result<String, std::io::Error> {
    let mut f = File::open(path)?;  // ? 运算符：出错就提前返回 Err
    let mut s = String::new();
    f.read_to_string(&mut s)?;
    Ok(s)
}

fn main() {
    match read_file("/etc/hostname") {
        Ok(content) => println!("文件内容:\n{content}"),
        Err(e)      => println!("读取失败: {e}"),
    }
}
```

`?` 是 Rust 最精巧的设计之一：错误自动向上传播，无需手动 `if err != nil`。

### 5.3 强大的模式匹配

```rust
let x = 5;

match x {
    1 => println!("一"),
    2 | 3 => println!("二或三"),       // 多值匹配
    4..=10 => println!("四到十"),      // 范围匹配
    _ => println!("其他"),             // 通配符（必须处理所有情况）
}

// 解构
let point = (3, 5);
match point {
    (0, 0)     => println!("原点"),
    (0, y)     => println!("在 y 轴上: {y}"),
    (x, 0)     => println!("在 x 轴上: {x}"),
    (x, y)     => println!("坐标: ({x}, {y})"),
}
```

---

## 6. 结构体与方法

```rust
// 结构体定义
struct User {
    username: String,
    email: String,
    active: bool,
}

// 方法（impl 块）
impl User {
    // 构造函数（惯例叫 new）
    fn new(username: String, email: String) -> Self {
        Self { username, email, active: true }
    }

    // 不可变方法
    fn display(&self) -> String {
        format!("{} <{}>", self.username, self.email)
    }

    // 可变方法
    fn deactivate(&mut self) {
        self.active = false;
    }
}

fn main() {
    let mut user = User::new(
        String::from("rustacean"),
        String::from("rust@example.com"),
    );
    println!("{}", user.display());  // rustacean <rust@example.com>
    user.deactivate();
    println!("活跃: {}", user.active);  // 活跃: false
}
```

---

## 7. Trait —— 共享行为（接口）

```rust
// trait 定义（类似 Go 的 interface，但更强大）
trait Summary {
    fn summarize(&self) -> String;

    // 默认实现
    fn author(&self) -> String {
        String::from("未知")
    }
}

// 为任意类型实现 trait
struct Article {
    title: String,
    author_name: String,
    content: String,
}

impl Summary for Article {
    fn summarize(&self) -> String {
        format!("{} — {}", self.title, self.author_name)
    }

    fn author(&self) -> String {
        self.author_name.clone()
    }
}

// trait bound：泛型约束
fn notify<T: Summary>(item: &T) {
    println!("{}", item.summarize());
}

// 语法糖：impl Trait
fn notify_short(item: &impl Summary) {
    println!("{}", item.summarize());
}
```

---

## 8. 泛型与零成本抽象

```rust
// 泛型函数 —— 编译期单态化（monomorphization）
// 对每种具体类型生成独立代码，无运行时开销
fn largest<T: PartialOrd>(list: &[T]) -> &T {
    let mut largest = &list[0];
    for item in list {
        if item > largest {
            largest = item;
        }
    }
    largest
}

// 泛型结构体
struct Point<T> {
    x: T,
    y: T,
}

fn main() {
    let nums = vec![34, 50, 25, 100, 65];
    println!("最大值: {}", largest(&nums));

    let chars = vec!['a', 'z', 'm', 'k'];
    println!("最大字符: {}", largest(&chars));
}
```

---

## 9. 集合类型

```rust
// Vec —— 动态数组（堆分配）
let mut v: Vec<i32> = Vec::new();
v.push(1);
v.push(2);
let v2 = vec![1, 2, 3, 4, 5];          // vec! 宏创建

for i in &v2 {
    println!("{i}");
}

// String —— UTF-8 编码的可变字符串
let mut s = String::from("hello");
s.push_str(" world");
s.push('!');
println!("{s}");  // hello world!

// HashMap —— 键值对
use std::collections::HashMap;
let mut scores = HashMap::new();
scores.insert(String::from("Blue"), 10);
scores.insert(String::from("Red"), 50);

// 安全取值（返回 Option）
let blue_score = scores.get("Blue").copied().unwrap_or(0);

// 遍历
for (key, value) in &scores {
    println!("{key}: {value}");
}
```

---

## 10. 迭代器与闭包

```rust
// 迭代器链 —— 编译后和手写循环一样快（零成本抽象）
let sum: i32 = (1..=1000)
    .filter(|x| x % 2 == 0)    // 只取偶数
    .map(|x| x * x)             // 平方
    .sum();                     // 求和
println!("{sum}");

// 闭包
let add_one = |x| x + 1;
println!("{}", add_one(5));    // 6

// 捕获环境变量
let factor = 10;
let multiply = |x| x * factor;
println!("{}", multiply(5));   // 50
```

---

## 11. 模块系统

```rust
// 项目结构
my_project/
├── Cargo.toml
└── src/
    ├── main.rs          // 入口
    ├── lib.rs           // 库根
    ├── config.rs        // 模块文件
    └── utils/
        └── mod.rs       // 目录 = 模块

// main.rs 中声明模块
mod config;
mod utils;

use config::load_config;
use utils::format_output;

fn main() {
    let cfg = load_config();
    println!("{}", format_output(&cfg));
}
```

---

## 12. 错误处理进阶

### 12.1 自定义错误类型

```rust
use std::fmt;
use std::io;

#[derive(Debug)]
enum AppError {
    IoError(io::Error),
    ParseError(String),
    ConfigError { field: String, reason: String },
}

// 实现 Display（用户可见的错误信息）
impl fmt::Display for AppError {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        match self {
            Self::IoError(e)  => write!(f, "IO 错误: {e}"),
            Self::ParseError(s) => write!(f, "解析错误: {s}"),
            Self::ConfigError { field, reason } => {
                write!(f, "配置错误 [{field}]: {reason}")
            }
        }
    }
}

// 如果使用 anyhow/thiserror 这两个库，上面的代码可以大幅简化
// 但标准库的做法如上所示，理解原理很重要
```

### 12.2 优雅地处理多种错误

```rust
use std::fs;
use std::io;

fn read_config() -> Result<String, io::Error> {
    // ? 自动转换错误类型并向上传播
    let content = fs::read_to_string("/etc/myapp/config.toml")?;
    Ok(content)
}

// 组合错误
fn load_and_parse() -> Result<(), Box<dyn std::error::Error>> {
    let raw = read_config()?;       // io::Error → Box<dyn Error>
    let value: i32 = raw.trim().parse()?;  // ParseIntError → Box<dyn Error>
    println!("配置值: {value}");
    Ok(())
}
```

---

## 13. Cargo 与 crates.io

### 13.1 Cargo.toml

```toml
[package]
name = "my-tool"
version = "0.1.0"
edition = "2024"

[dependencies]
serde = { version = "1", features = ["derive"] }
serde_json = "1"
clap = { version = "4", features = ["derive"] }
tokio = { version = "1", features = ["full"] }

[dev-dependencies]
tempfile = "3"

[profile.release]
opt-level = 3       # 最大优化
lto = true          # 链接时优化（二进制更小更快）
codegen-units = 1   # 更好的优化（但编译更慢）
```

### 13.2 关键 crate 生态

| 类别 | crate | 用途 |
|------|-------|------|
| CLI | `clap` | 命令行参数解析 |
| 序列化 | `serde` + `serde_json` | JSON/TOML/YAML 序列化 |
| 异步 | `tokio` | 异步运行时（事实标准） |
| HTTP 客户端 | `reqwest` | HTTP 请求 |
| HTTP 服务端 | `axum` / `actix-web` | Web 框架 |
| 数据库 | `sqlx` / `diesel` / `sea-orm` | 数据库操作 |
| 日志 | `log` + `env_logger` | 日志记录 |
| 错误处理 | `anyhow` / `thiserror` | 简化错误处理 |
| 正则 | `regex` | 正则表达式 |
| 随机数 | `rand` | 随机数生成 |
| 哈希 | `sha2` / `md-5` | 密码学哈希 |
| 文件系统 | `walkdir` | 递归遍历目录 |
| 日期时间 | `chrono` | 日期时间处理 |
| 测试 | `proptest` | 属性测试 |
| 桌面 GUI | `egui` / `tauri` / `slint` | 图形界面 |

---

## 14. 两个完整示例

### 14.1 命令行工具：批量重命名文件

```rust
use clap::Parser;
use std::fs;
use std::path::PathBuf;

#[derive(Parser)]
#[command(about = "批量修改文件扩展名")]
struct Args {
    /// 目标目录
    directory: PathBuf,
    /// 原扩展名（如 .jpeg）
    from: String,
    /// 新扩展名（如 .jpg）
    to: String,
}

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let args = Args::parse();

    for entry in fs::read_dir(&args.directory)? {
        let entry = entry?;
        let path = entry.path();

        if !path.is_file() {
            continue;
        }

        if path.extension().and_then(|e| e.to_str()) ==
           Some(args.from.trim_start_matches('.'))
        {
            let mut new_path = path.clone();
            new_path.set_extension(args.to.trim_start_matches('.'));
            fs::rename(&path, &new_path)?;
            println!("{} → {}", path.display(), new_path.display());
        }
    }

    println!("完成！");
    Ok(())
}
```

### 14.2 读取配置文件并处理

```rust
use serde::Deserialize;
use std::fs;

#[derive(Deserialize, Debug)]
struct Config {
    server: ServerConfig,
    database: Option<DatabaseConfig>,
}

#[derive(Deserialize, Debug)]
struct ServerConfig {
    host: String,
    port: u16,
}

#[derive(Deserialize, Debug)]
struct DatabaseConfig {
    url: String,
    max_connections: u32,
}

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let content = fs::read_to_string("config.toml")?;
    let config: Config = toml::from_str(&content)?;
    println!("服务器: {}:{}", config.server.host, config.server.port);

    if let Some(db) = config.database {
        println!("数据库: {}（最大连接: {}）", db.url, db.max_connections);
    }

    Ok(())
}
```

---

## 15. Rust vs 其他语言

| 维度 | Rust | C | C++ | Go | Python |
|------|------|---|-----|----|--------|
| 内存安全 | ✅ 编译期保证 | ❌ 手动 | ⚠️ 智能指针 | ✅ GC | ✅ GC |
| 数据竞争 | ✅ 编译期杜绝 | ❌ | ❌ | ✅ | ⚠️ GIL 限制 |
| 运行时开销 | 零（无 GC） | 零 | 低 | GC 开销 | GC 开销 |
| 编译速度 | 🔴 慢 | 🟢 快 | 🟡 中等 | 🟢 快 | —（解释型） |
| 学习曲线 | 🔴 陡峭 | 🟡 中等 | 🔴 陡峭 | 🟢 简单 | 🟢 简单 |
| 包管理器 | Cargo ✅ | 无 | vcpkg/Conan | Go Modules | pip |
| 适合场景 | 系统软件/CLI/Web/嵌入式 | 内核/嵌入式 | 游戏/桌面/GUI | 网络服务/CLI | 脚本/自动化/ML |

> **选择建议：** 写脚本 → Python。写网络服务且团队需要简单 → Go。写内核驱动、高性能系统、或想彻底告别内存 bug → Rust。既想要 C 的性能又想要现代工具链 → Rust。

---

## 16. 学习路径建议

```
第一阶段（2-3 周）—— "和编译器打架期"
├── 变量、类型、控制流、函数
├── 所有权、借用、生命周期（核心难点）
├── 枚举与 match
└── struct + impl

第二阶段（2-3 周）—— "开始能用"
├── Vec / String / HashMap
├── 迭代器与闭包
├── Trait 与泛型
├── Result / Option 错误处理
└── 模块系统与 Cargo

第三阶段（边用边学）
├── 一个 CLI 小工具（clap + serde）
├── 文件操作（std::fs / walkdir）
├── 异步编程（tokio）
├── Web 服务（axum）
└── 阅读标准库源码
```

> 社区名言：**"如果你能让 Rust 代码通过编译，它通常就是正确的。"**
>
> 和编译器打架的阶段是必经之路——编译器不是在刁难你，它在阻止你写出有 bug 的代码。通常 1-3 个月后，所有权/借用变成第二本能。

---

## 17. 常见问题

### Q1: Rust 编译为什么这么慢？

```bash
# 原因：Rust 做大量编译期检查 + 单态化泛型 + LLVM 优化

# 加速技巧：
# 1. 开发时用 cargo check 代替 cargo build（不生成二进制，快很多）
cargo check

# 2. 使用 sccache 缓存编译结果
sudo pacman -S sccache
# .cargo/config.toml:
# [build]
# rustc-wrapper = "/usr/bin/sccache"

# 3. 使用 mold 链接器
sudo pacman -S mold
# .cargo/config.toml:
# [target.x86_64-unknown-linux-gnu]
# linker = "clang"
# rustflags = ["-C", "link-arg=-fuse-ld=mold"]
```

### Q2: Rust 适不适合写小工具？

**适合。** 编译成单个二进制文件，分发方便。虽然写的时候比 Python 多花时间，但一旦编译通过，几乎不会出运行时错误。如果分发对象没有装 Python，Rust 编译出的独立二进制远比 `pip install` 友好。

### Q3: `unwrap()` 是什么意思？为什么到处都能看到？

```rust
let x: Option<i32> = Some(5);
let val = x.unwrap();     // 如果是 Some，取出值；如果是 None，panic

let r: Result<i32, _> = "42".parse();
let num = r.unwrap();     // 如果是 Ok，取出值；如果是 Err，panic
```

`unwrap()` 是"我知道这里不可能出错，如果出错就崩溃吧"的语义。**生产代码里少用**，用 `match` 或 `?` 替代。

### Q4: String 和 &str 有什么区别？

```rust
let s1: String = String::from("hello");   // 拥有所有权，在堆上，可变
let s2: &str = "world";                   // 借用，字符串字面量，不可变
let s3: &str = &s1;                       // 从 String 借用

// String → &str（自动转换，因为 String 实现了 Deref<Target=str>）
fn takes_str(s: &str) { println!("{s}"); }
takes_str(&s1);    // ✅
takes_str(s1);     // ❌ 所有权会移动，通常不这样用
```

### Q5: Rust 的异步（async/await）难吗？

比 Python 和 JavaScript 的异步要复杂，因为 Rust 的 `Future` 是惰性的——必须由运行时（tokio）驱动才会执行。入门阶段先避开异步，专注同步代码。

### Q6: 我该不该第一时间学 unsafe Rust？

**不要。** `unsafe` 是给知道自己在做什么的人准备的逃生舱。你的目标是用 Rust 写出不需要 `unsafe` 的代码。把 `unsafe` 留给写内核驱动和 C FFI 的时候。

### Q7: 怎么调用 C 库？

```rust
// 通过 FFI（Foreign Function Interface）调用 C 函数
extern "C" {
    fn getpid() -> i32;
}

fn main() {
    unsafe {
        println!("PID: {}", getpid());
    }
}
```

通常使用 `bindgen` 从 C 头文件自动生成 Rust 绑定，或直接用社区封装好的 crate（如 `libc`）。

---

## 参考

- [Rust 官方教程（The Book）](https://doc.rust-lang.org/book/) — 必读，有[中文翻译版](https://kaisery.github.io/trpl-zh-cn/)
- [Rust by Example](https://doc.rust-lang.org/rust-by-example/) — 通过示例学 Rust
- [Rustlings](https://github.com/rust-lang/rustlings) — 交互式小练习，修复编译错误的习题集
- [标准库文档](https://doc.rust-lang.org/std/)
- [crates.io](https://crates.io/) — Rust 包注册中心
- [Arch Wiki — Rust](https://wiki.archlinux.org/title/Rust)
- [Rust in Linux Kernel](https://rust-for-linux.com/)
