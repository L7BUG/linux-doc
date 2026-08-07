# Rust 模拟鼠标输入：从软件到硬件的完整方案

## 1. 背景与问题

编写自动化脚本（游戏辅助、办公自动化、测试脚本等）时，需要让程序模拟鼠标点击。在 Linux 下有多层实现方式，从最简单的 X11 API 调用，到用一块 $3 的微控制器模拟真实 USB 鼠标硬件。**核心问题在于：你运行脚本的机器，也是运行反作弊程序的机器——这是一场不对等博弈。**

本文涵盖：
- 反作弊检测的多层机制
- 4 种模拟方案（X11 → uinput → USB Gadget → Arduino/Pi Pico）
- 完整 Rust 代码示例
- 行为随机化技术
- 各方案成本与风险对比

---

## 2. 反作弊检测层级（先了解敌人）

反作弊程序（特别是内核级反作弊：Vanguard、EAC、BattlEye）有多层检测机制：

### 2.1 第一层：用户态 API Hook

反作弊 Hook 了关键输入函数，能直接判断事件来源：

| Hook 目标 | 检测内容 |
|-----------|---------|
| `XSendEvent` | `send_event` 字段：正常设备事件 = `False`，程序模拟 = `True` |
| `XTestFakeInput` | enigo/xdotool 底层调用这个，直接标记为"虚假输入" |
| `libinput` 路径 | 正常鼠标走 `/dev/input/event*` → libinput，模拟的走 X11 协议，路径不同 |

**`XSendEvent` 是最致命的**——反作弊只需检查这一个字段。

### 2.2 第二层：内核级检测

内核反作弊运行在 Ring 0，能看到一切：

```
用户态:     你的 Rust 程序         ← 一切用户态行为都在监控中
────────────────────────────────────
内核态:     反作弊驱动
              ↓ 它可以：
              · 看到你的进程调用了哪些 syscall
              · 拦截 /dev/uinput 的写入
              · 追踪输入事件来自哪个 PID
              · 检测是否有程序在注入 evdev 事件
              · 检查设备驱动路径（真实 USB vs 虚拟 uinput）
```

### 2.3 第三层：行为分析

即使绕过了所有软件检测，反作弊还会分析**行为模式**：

| 检测维度 | 人类行为 | 脚本行为 |
|---------|---------|---------|
| 点击间隔 | 50-200ms 随机抖动 | 精确固定间隔 |
| 移动轨迹 | 贝塞尔曲线，有加速度 | 直线或无规律跳跃 |
| 反应时间 | 200-300ms | < 50ms 或精确固定值 |
| 点击位置 | ±3px 微小偏移 | 永远同一点 |
| 活动时间 | 会累，会休息 | 24 小时不停 |
| 光标停留 | 会犹豫、会在空白处无意义移动 | 永远在做"有目的地"的事 |

---

## 3. 方案一：X11 用户态模拟（enigo）

### 3.1 原理

```
你的 Rust 程序
    ↓ 调用 enigo API
libX11 / XTest
    ↓ XSendEvent / XTestFakeInput
X Server
    ↓ send_event = True ← 致命标记
目标窗口
```

### 3.2 适用场景

- 单机游戏（无人管）
- 模拟器挂机
- 办公自动化
- GUI 测试

绝对不适用于带反作弊的在线竞技游戏。

### 3.3 完整示例

`Cargo.toml`：

```toml
[package]
name = "auto-clicker"
version = "0.1.0"
edition = "2021"

[dependencies]
enigo = "0.3"
screenshots = "0.8"
opencv = "0.92"
rand = "0.8"
```

`src/main.rs`：

```rust
use enigo::{Coordinate, Direction, Enigo, Mouse, Settings};
use opencv::{core, imgproc, imgcodecs};
use screenshots::Screen;
use rand::Rng;
use std::{thread, time::Duration};

/// 将 RGBA 缓冲区转为 opencv BGR Mat
fn rgba_to_bgr_mat(buf: &[u8], w: u32, h: u32) -> core::Mat {
    let bgr: Vec<u8> = buf.chunks(4)
        .flat_map(|p| [p[2], p[1], p[0]])
        .collect();
    unsafe {
        core::Mat::new_rows_cols_with_data(
            h as i32, w as i32,
            core::CV_8UC3,
            bgr.as_ptr() as *mut _,
            core::Mat_AUTO_STEP,
        ).unwrap()
    }
}

/// 模板匹配，返回最佳匹配坐标
fn match_template(
    screen: &core::Mat,
    tmpl: &core::Mat,
    threshold: f64,
) -> Option<(i32, i32, f64)> {
    let mut result = core::Mat::default();
    imgproc::match_template(
        screen, tmpl, &mut result,
        imgproc::TM_CCOEFF_NORMED, &core::no_array(),
    ).ok()?;

    let mut min_val = 0.0;
    let mut max_val = 0.0;
    let mut max_loc = core::Point::default();
    core::min_max_loc(
        &result,
        Some(&mut min_val), Some(&mut max_val),
        None, Some(&mut max_loc),
        &core::no_array(),
    ).ok()?;

    if max_val > threshold {
        Some((max_loc.x, max_loc.y, max_val))
    } else {
        None
    }
}

/// 带人类行为特征的点击
fn humanized_click(enigo: &mut Enigo, x: i32, y: i32) {
    let mut rng = rand::thread_rng();
    let fx = x + rng.gen_range(-2..=2);
    let fy = y + rng.gen_range(-2..=2);

    enigo.move_mouse(fx, fy, Coordinate::Abs).unwrap();
    thread::sleep(Duration::from_millis(rng.gen_range(40..=120)));
    enigo.button(enigo::Button::Left, Direction::Click).unwrap();
    thread::sleep(Duration::from_millis(rng.gen_range(80..=300)));
}

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let mut enigo = Enigo::new(&Settings::default())?;
    let screens = Screen::all()?;
    let screen = &screens[0];

    // 加载所有模板
    let template_paths = ["button.png", "icon.png"];
    let mut templates: Vec<(String, core::Mat)> = Vec::new();
    for name in &template_paths {
        if let Ok(mat) = imgcodecs::imread(
            &format!("/path/to/templates/{}", name),
            imgcodecs::IMREAD_COLOR,
        ) {
            templates.push((name.to_string(), mat));
        }
    }

    println!("自动点击器启动 按 Ctrl+C 退出");
    println!("已加载 {} 个模板", templates.len());

    loop {
        let capture = screen.capture()?;
        let mat = rgba_to_bgr_mat(capture.as_ref(), capture.width(), capture.height());

        for (name, tmpl) in &templates {
            if let Some((x, y, confidence)) = match_template(&mat, tmpl, 0.75) {
                let center_x = x + tmpl.cols() / 2;
                let center_y = y + tmpl.rows() / 2;
                println!(
                    "匹配到 [{}] @ ({}, {}) 置信度={:.2}",
                    name, center_x, center_y, confidence
                );
                humanized_click(&mut enigo, center_x, center_y);
            }
        }

        thread::sleep(Duration::from_millis(100));
    }
}
```

### 3.4 检测风险

- **检测等级：必检测**
- X11 `send_event` 字段暴露一切
- 反作弊 Hook `XSendEvent` / `XTestFakeInput` 可直接识别

---

## 4. 方案二：uinput 内核虚拟设备（evdev crate）

### 4.1 原理

绕过 X11 直接在 `/dev/uinput` 注册一个虚拟输入设备，向内核发送 evdev 事件。

```
你的 Rust 程序
    ↓ ioctl + write
/dev/uinput ──→ 内核 input 子系统
    ↓ 创建虚拟设备 /dev/input/event15
libinput / X Server
    ↓ send_event = False ← 比方案一好
目标窗口

但内核能看到：
  /sys/devices/virtual/input/  ← "virtual" 暴露了
  驱动名 = "uinput"
```

### 4.2 完整示例

`Cargo.toml`：

```toml
[dependencies]
evdev = "0.12"
```

`src/main.rs`：

```rust
use evdev::{
    uinput::{VirtualDevice, VirtualDeviceBuilder},
    AbsInfo, AttributeSet, Key, UinputAbsSetup, EventType, InputEvent,
};
use std::{thread, time::Duration};

fn create_virtual_mouse() -> Result<VirtualDevice, Box<dyn std::error::Error>> {
    // 配置 ABS_X —— 屏幕宽度范围（以 1920×1080 为例）
    let abs_x = UinputAbsSetup::new(
        evdev::AbsoluteAxisType::ABS_X,
        AbsInfo::new(0, 0, 1920, 0, 0, 1),
    );

    let abs_y = UinputAbsSetup::new(
        evdev::AbsoluteAxisType::ABS_Y,
        AbsInfo::new(0, 0, 1080, 0, 0, 1),
    );

    let mut keys = AttributeSet::<Key>::new();
    keys.insert(Key::BTN_LEFT);
    keys.insert(Key::BTN_RIGHT);
    keys.insert(Key::BTN_MIDDLE);

    let device = VirtualDeviceBuilder::new()?
        .name("我的虚拟鼠标")
        .with_absolute_axis(&abs_x)?
        .with_absolute_axis(&abs_y)?
        .with_keys(&keys)?
        .build()?;

    Ok(device)
}

fn move_and_click(device: &mut VirtualDevice, x: i32, y: i32) {
    // 移动到目标坐标
    device.emit(&[
        InputEvent::new(EventType::ABSOLUTE, evdev::AbsoluteAxisType::ABS_X.0, x),
        InputEvent::new(EventType::ABSOLUTE, evdev::AbsoluteAxisType::ABS_Y.0, y),
        InputEvent::new(EventType::SYNCHRONIZATION, 0, 0), // SYN_REPORT
    ]).unwrap();

    thread::sleep(Duration::from_millis(20));

    // 按下左键
    device.emit(&[
        InputEvent::new(EventType::KEY, Key::BTN_LEFT.code(), 1), // 1 = 按下
        InputEvent::new(EventType::SYNCHRONIZATION, 0, 0),
    ]).unwrap();

    thread::sleep(Duration::from_millis(30));

    // 释放左键
    device.emit(&[
        InputEvent::new(EventType::KEY, Key::BTN_LEFT.code(), 0), // 0 = 释放
        InputEvent::new(EventType::SYNCHRONIZATION, 0, 0),
    ]).unwrap();
}

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let mut mouse = create_virtual_mouse()?;

    println!("虚拟鼠标已创建");
    println!("查看设备: cat /proc/bus/input/devices | grep -A5 '我的虚拟鼠标'");

    // 等待 2 秒让用户准备好
    thread::sleep(Duration::from_secs(2));

    // 模拟点击屏幕中心
    move_and_click(&mut mouse, 960, 540);

    Ok(())
}
```

### 4.3 在系统里的样子

```bash
$ cat /proc/bus/input/devices | grep -A5 "我的虚拟鼠标"

N: Name="我的虚拟鼠标"
H: Handlers=mouse2 event15       # ← 不是 mouse0（那是你的真实鼠标）
S: Sysfs=/devices/virtual/input/input15  # ← "virtual" 标记
```

### 4.4 检测风险

- **检测等级：中高**
- X11 send_event 没问题（=`False`），但内核知道设备来自 uinput
- `/sys/devices/virtual/input/` 路径暴露虚拟设备身份（真实鼠标在 `/sys/devices/pci.../usb/.../input/`）
- 反作弊可以检查 `/proc/bus/input/devices` 中是否存在非标准设备

---

## 5. 方案三：USB Gadget（树莓派 / 独立 Linux 板）

### 5.1 原理

用一台**独立的 Linux 设备**（树莓派 Zero、Android 手机等），通过 USB 线连接到目标电脑。利用内核的 USB Gadget 子系统，把自己伪装成 USB HID 鼠标硬件。

```
[目标电脑] ←──USB 线──→ [树莓派 / Linux 板]
                           ↓
                    你的 Rust 程序 → /dev/hidg0
                           ↓
                    USB Gadget 驱动
                           ↓
                    目标电脑看到：一个"USB 罗技鼠标"
```

### 5.2 硬件准备

- **树莓派 Zero / Zero 2 W**：~$10-15。有两个 micro-USB 口，一个供电一个接目标电脑
- **树莓派 4/5**：可以用 USB-C OTG 模式（需额外配置）

### 5.3 配置 USB Gadget（一次性设置）

在树莓派上执行：

```bash
#!/bin/bash
# 在树莓派上配置 USB Gadget HID 鼠标

# 加载内核模块
sudo modprobe libcomposite
sudo modprobe dwc2        # 树莓派 USB 控制器

# 启用 dwc2 overlay（编辑 /boot/config.txt，加一行）：
# dtoverlay=dwc2

cd /sys/kernel/config/usb_gadget

# 创建 Gadget 配置
sudo mkdir -p mouse_gadget
cd mouse_gadget

# 设置 USB 描述符（伪装成罗技鼠标）
sudo echo 0x046d > idVendor       # Logitech
sudo echo 0xc077 > idProduct      # M105 鼠标
sudo echo 0x0100 > bcdDevice      # 1.0.0

# USB 设备信息
sudo mkdir -p strings/0x409
sudo echo "Logitech" > strings/0x409/manufacturer
sudo echo "USB Optical Mouse" > strings/0x409/product
sudo echo "1234567890" > strings/0x409/serialnumber

# 配置 HID 功能
sudo mkdir -p functions/hid.usb0
sudo echo 0 > functions/hid.usb0/protocol
sudo echo 0 > functions/hid.usb0/subclass
sudo echo 4 > functions/hid.usb0/report_length

# HID 报告描述符（标准鼠标：3 键 + XY 位移 + 滚轮）
sudo echo -ne "\x05\x01\x09\x02\xa1\x01\x09\x01\xa1\x00\x05\x09\x19\x01\x29\
\x03\x15\x00\x25\x01\x95\x03\x75\x01\x81\x02\x95\x01\x75\x05\x81\x03\x05\
\x01\x09\x30\x09\x31\x15\x81\x25\x7f\x75\x08\x95\x02\x81\x06\xc0\xc0" \
  > functions/hid.usb0/report_desc

# 绑定到配置
sudo mkdir -p configs/c.1/strings/0x409
sudo echo "HID Mouse Config" > configs/c.1/strings/0x409/configuration
sudo ln -s functions/hid.usb0 configs/c.1/

# 激活 Gadget（树莓派通过 USB 线连接到目标电脑后执行）
UDC=$(ls /sys/class/udc | head -1)
sudo echo "$UDC" > UDC
```

### 5.4 Rust 控制端

树莓派上的 Rust 程序（需交叉编译为 `aarch64-unknown-linux-gnu`）：

```rust
use std::fs::OpenOptions;
use std::io::Write;
use std::{thread, time::Duration};

/// HID 鼠标报告：[按钮(1字节), X位移(1字节), Y位移(1字节), 滚轮(1字节)]
fn build_report(buttons: u8, dx: i8, dy: i8, wheel: i8) -> [u8; 4] {
    [buttons, dx as u8, dy as u8, wheel as u8]
}

fn main() -> std::io::Result<()> {
    let mut hid = OpenOptions::new()
        .write(true)
        .open("/dev/hidg0")?;

    // 移动鼠标：向右 10，向下 5（相对移动模式）
    let report = build_report(0, 10, 5, 0);
    hid.write_all(&report)?;

    thread::sleep(Duration::from_millis(50));

    // 左键点击
    let press = build_report(0x01, 0, 0, 0);   // BIT 0 = 左键按下
    let release = build_report(0x00, 0, 0, 0);  // 全释放

    hid.write_all(&press)?;
    thread::sleep(Duration::from_millis(30));
    hid.write_all(&release)?;

    println!("HID 报告已发送");
    Ok(())
}
```

### 5.5 在目标电脑上的效果

```bash
$ lsusb
Bus 001 Device 005: ID 046d:c077 Logitech, Inc. M105 Optical Mouse
#                                ↑        ↑
#                       你设定的 VID   你设定的 PID

$ cat /proc/bus/input/devices
N: Name="Logitech USB Optical Mouse"
S: Sysfs=/devices/pci0000:00/0000:00:14.0/usb1/1-1/1-1:1.0/.../input/input5
#        ↑ 走真实 USB 总线路径，没有 "virtual" 标记！
```

### 5.6 检测风险

- **检测等级：中低**
- 对目标电脑来说，这就是通过 USB 连接的"真实鼠标硬件"
- 走完整的 USB HID 驱动栈，不是 `virtual/input/` 路径
- **但仍可能被检测：**
  - USB 描述符细节可能与真品有细微差异（序列号长度、厂商字符串编码等）
  - USB 数据包时序特征（硬件有微秒级抖动，软件模拟通常太规律）
  - HID 报告频率异常（真实鼠标典型 125Hz/1000Hz，你的循环间隔是固定的）

### 5.7 完整架构：PC 截图识别 + 树莓派点击

```
┌─────────────────┐    串口/Wi-Fi/蓝牙   ┌──────────────────┐
│   目标电脑 (PC)   │ ←─────────────────→ │  树莓派 Zero      │
│                  │                     │                  │
│  Rust 程序：      │   坐标指令：         │  Rust 程序：       │
│  1. 截图         │   "500,300,CLICK"   │  1. 接收坐标      │
│  2. 图像识别     │ ──────────────────→ │  2. 写入 /dev/hidg0│
│  3. 坐标计算     │                     │                  │
│  4. 发送指令     │                     └──────┬───────────┘
└─────────────────┘                            │ USB HID
                                                ↓
                                          目标电脑看到的：
                                          "罗技 USB 鼠标"
```

---

## 6. 方案四：Arduino / Pi Pico 硬件方案（最强）

### 6.1 原理

用微控制器直接模拟 USB HID 鼠标——微控制器是独立芯片，目标电脑无法从硬件层面区分它和真鼠标。

```
目标电脑 ←──USB 线──→ [Arduino/Pi Pico] ←──串口──→ [PC 运行 Rust 程序]
                         ↑
                   独立微控制器
                   运行自己的固件
                   目标电脑看到的：一个真实 USB HID 设备
```

### 6.2 硬件选择

| 硬件 | 价格 | USB HID 支持 | 推荐度 |
|------|------|-------------|--------|
| Arduino Pro Micro / Leonardo | ~$3-5 | 原生 ATmega32U4 | ★★★★ |
| Raspberry Pi Pico | ~$4-5 | RP2040 + TinyUSB | ★★★★★ |
| Teensy 4.0 | ~$20 | NXP iMXRT1062 | ★★★★★ |
| ESP32-S2/S3 | ~$3-5 | 支持 USB OTG | ★★★ |

> **推荐 Pi Pico**：性价比最高，有完善的 Rust 支持（`rp2040-hal` + `usbd-hid`）。

### 6.3 Arduino 端固件

```cpp
// Arduino Pro Micro / Leonardo 固件
// 通过串口接收 PC 发来的坐标并执行鼠标操作

#include <Mouse.h>

struct Command {
  char type;     // 'M'=Move, 'C'=Click, 'W'=Wheel
  int16_t x;
  int16_t y;
};

Command cmd;
byte* cmdBytes = (byte*)&cmd;

void setup() {
  Serial.begin(115200);
  Mouse.begin();
}

void loop() {
  // 等待完整指令（5 字节）
  if (Serial.available() >= sizeof(Command)) {
    Serial.readBytes(cmdBytes, sizeof(Command));

    switch (cmd.type) {
      case 'M':
        Mouse.move(cmd.x, cmd.y, 0);  // 相对移动
        break;
      case 'C':
        Mouse.click(cmd.x ? MOUSE_LEFT : MOUSE_RIGHT);
        break;
      case 'W':
        Mouse.move(0, 0, cmd.x);      // 滚轮
        break;
    }
  }
}
```

### 6.4 PC 端 Rust 程序

```rust
use serialport;
use std::io::Write;
use std::{thread, time::Duration};
use rand::Rng;

#[repr(C, packed)]
struct Command {
    cmd_type: u8,   // b'M'=Move, b'C'=Click
    x: i16,
    y: i16,
}

fn send_command(port: &mut Box<dyn serialport::SerialPort>, cmd: &Command) {
    let bytes: &[u8] = unsafe {
        std::slice::from_raw_parts(
            cmd as *const Command as *const u8,
            std::mem::size_of::<Command>(),
        )
    };
    port.write_all(bytes).unwrap();
}

fn click_at(port: &mut Box<dyn serialport::SerialPort>, x: i32, y: i32) {
    let mut rng = rand::thread_rng();

    let move_cmd = Command {
        cmd_type: b'M',
        x: (x + rng.gen_range(-3..=3)) as i16,
        y: (y + rng.gen_range(-3..=3)) as i16,
    };
    send_command(port, &move_cmd);

    // 随机延迟（模拟人类反应）
    thread::sleep(Duration::from_millis(rng.gen_range(50..=150)));

    // 点击
    let click_cmd = Command {
        cmd_type: b'C',
        x: 1,  // 左键
        y: 0,
    };
    send_command(port, &click_cmd);
}

fn main() {
    let mut port = serialport::new("/dev/ttyACM0", 115200)
        .timeout(Duration::from_secs(1))
        .open()
        .expect("无法打开串口");

    println!("已连接到 Arduino 鼠标");

    // 测试：点击屏幕 (500, 300) 位置
    click_at(&mut port, 500, 300);

    thread::sleep(Duration::from_secs(2));
}
```

### 6.5 检测风险

- **检测等级：低**
- 目标电脑看到的是一个通过 USB 连接的**真实微控制器**
- 设备路径：`/sys/devices/pci.../usb/.../input/`（和真实鼠标一模一样的路径）
- 跨过了所有软件和内核检测层
- **唯一破绽：行为模式分析**

---

## 7. 方案全景对比

```
方案                │ 难度 │ 硬件成本 │ X11检测 │ 内核检测 │ 行为检测 │ 延迟
────────────────────┼──────┼─────────┼─────────┼─────────┼─────────┼─────
enigo (X11)        │ 极低 │ $0      │ ✗ 必检测 │ ✗       │ ✗       │ ~1ms
uinput (evdev)      │ 低   │ $0      │ ✓ 绕过  │ ✗ 可检测│ ✗       │ ~1ms
USB Gadget (另一台) │ 中   │ $10-50  │ ✓ 绕过  │ △ 可能  │ ✗       │ ~5ms
Arduino / Pi Pico  │ 中   │ $3-5    │ ✓ 绕过  │ ✓ 绕过  │ ✗       │ ~5ms
物理机械臂          │ 极高 │ $200+   │ ✓ 绕过  │ ✓ 绕过  │ △       │ ~200ms
```

> **行为检测是必须面对的终局。** 无论硬件层多完美，只要你做的事看起来不像人类，就会被 AI 模型揪出来。

---

## 8. 设备切换检测——你忽略的致命破绽

即使你用了 Arduino 硬件方案，反作弊还有一个往往被忽略的检测维度：**你是不是在两个设备之间来回切换？**

### 8.1 典型的脚本使用模式

```
正常游戏时：真鼠标移动、瞄准
检测到目标时：脚本接管，Arduino 点击
然后又切换回真鼠标操作
```

**这种模式本身就是最显眼的检测信号。**

### 8.2 Linux input 子系统如何暴露设备源

每个输入事件都带有设备标识。反作弊不需要理解你做了什么操作，它只需追踪事件的"来源设备"：

```bash
# 真实鼠标
/dev/input/event3  →  Logitech G502  →  USB 总线

# 你的 Arduino
/dev/input/event18 →  Arduino Micro  →  USB 总线（另一条）
```

每次事件到来时，内核都会报告是哪个 `eventX` 发出的。反作弊可以通过以下途径获取：

| 接口 | 能获取到什么 |
|------|------------|
| `/dev/input/event*` 直接读取 | 每个事件的完整设备路径 |
| `libinput` API | `libinput_event_get_device()` 返回设备指针 |
| XInput2 扩展 | `XIEvent` 结构含 `sourceid`（设备 ID） |
| `/proc/bus/input/devices` | 所有注册设备列表 |
| `evtest` 等价逻辑 | 监控哪些 eventX 有事件流 |

### 8.3 检测手法

#### 手法一：设备源追踪

```c
// 反作弊伪代码——追踪每个输入事件的来源
void on_input_event(struct input_event *ev, int device_fd) {
    const char *dev_name = get_device_name(device_fd);
    
    // 记录：这个事件来自哪个设备
    if (strstr(dev_name, "Arduino") || 
        strstr(dev_name, "Teensy") ||
        dev_name 不在白名单中) {
        // 标记为可疑设备
        suspicious_events[device_fd]++;
    }
}
```

反作弊不需要知道你的 Arduino 在做什么——它只需要知道**存在两个鼠标并且都在活动**。

#### 手法二：操作模式和设备切换的时序关联

更致命的检测方式——分析设备切换和操作结果之间的**时序关联**：

```
时间轴（毫秒级）：
─────────────────────────────────────────────────────
event3  (真鼠标)   │████████████│            │██████████│
event18 (Arduino)  │            │████████████│          │
─────────────────────────────────────────────────────
游戏事件            │   移动    │  精准点击  │   移动   │
                   │           │  ↑ 每次精准操作恰好来自Arduino
```

反作弊 AI 模型分析的是：

| 特征 | 人类使用两个鼠标 | 脚本切换 |
|------|---------------|---------|
| 切换频率 | 几乎不切换 | 每次目标出现时切换 |
| 操作类型分布 | 两个设备做的事差不多 | 一个只做复杂移动，一个只做"关键点击" |
| 切换时机 | 随机 | 总是在"检测到目标"之后 |
| 事件间隔 | 无规律 | Arduino 事件总在截图之后 0-50ms 出现 |

#### 手法三：设备能力指纹不匹配

```bash
# 真实鼠标的 evdev 能力位
Event: type 2 (REL), code 0 (REL_X), value 450
Event: type 2 (REL), code 1 (REL_Y), value -200
Event: type 2 (REL), code 8 (REL_WHEEL), value 0
Event: type 4 (MSC), code 4 (MSC_SCAN), value 90003  # 有扫描码

# Arduino 的 evdev 能力位（可能缺少某些特性）
Event: type 2 (REL), code 0 (REL_X), value 500
Event: type 2 (REL), code 1 (REL_Y), value 300
# 缺少 MSC_SCAN、REL_WHEEL_HI_RES 等高级特性
```

不同设备的能力集不同——一个支持高精度滚轮 + 横向滚轮 + 可编程按键，另一个只有基础 XY 移动 + 左右键。反作弊看到**两个能力集差异巨大的鼠标在交替活动**，这就是信号。

### 8.4 解决方案

#### 方案 A：全时 Arduino（最简单可靠）

```
不切换了——全程只用 Arduino 一个设备。

真鼠标 → 不连接，或只作为"传感器"读取位置，
        所有点击和移动都通过 Arduino 发出。
```

**问题：** 你需要实时把自己的鼠标位置同步给 Arduino，增加延迟。

#### 方案 B：Arduino 作为 USB 中继（进阶）

```
真鼠标 ──USB──→ [树莓派/Pi Pico 中继] ──USB──→ 目标电脑
                         ↑
                  你的 Rust 程序
                  可以拦截、修改、注入 HID 报告
                  但目标电脑始终只看到一个 USB 设备
```

这样目标电脑永远只看到一个 USB 鼠标设备，你可以在中间层注入额外的点击/移动。但实现复杂度大幅上升。

#### 方案 C：模拟真鼠标的完整能力集

如果你非要用双设备，至少让两个设备看起来"像是一个牌子"：

- Arduino 固件模拟真实鼠标的 USB 描述符（VID/PID/产品字符串完全一致）
- 尽可能实现相同的能力集（HI_RES_SCROLL、横向滚轮等）
- 关键：**不要让 Arduino 只在"关键操作"时活动——即使不需要它点击，也让它持续发空白事件（心跳信号），让两个设备在时间轴上都有连续的活动。**

```rust
// 在 Rust 端让 Arduino 持续发"无操作"的心跳
fn keep_alive_loop(port: &mut Box<dyn serialport::SerialPort>) {
    loop {
        // 发送空移动（位移 0,0），保持设备活动
        let heartbeat = Command { cmd_type: b'H', x: 0, y: 0 };
        send_command(port, &heartbeat);
        thread::sleep(Duration::from_millis(10));  // 100Hz 心跳
    }
}
```

#### 方案 D：单设备 + evdev 拦截（最隐蔽但最复杂）

```
只用一个真实鼠标，但在内核层拦截/修改其 evdev 事件流。

正常操作 → 放行原始事件
需要脚本点击 → 拦截真实鼠标事件，注入修改后的事件
               从同一个 eventX 发出
```

这需要内核模块或 eBPF 级别的操作，实现难度极高，但**目标电脑永远只看到一个鼠标，不存在"设备切换"。**

### 8.5 各方案对设备切换检测的抵御能力

| 方案 | 设备数 | 检测难度 | 说明 |
|------|--------|---------|------|
| enigo + 真鼠标混用 | 软件模拟+真实 | **必检测** | send_event 标记 + 设备来源差异 |
| uinput + 真鼠标混用 | 2 个不同设备 | **易检测** | event3 vs event15，路径不同 |
| Arduino + 真鼠标混用 | 2 个 USB 设备 | **易检测** | 不同 VID/PID + 切换模式分析 |
| 全时 Arduino（不用真鼠标） | 1 个设备 | **难检测** | 无切换可追踪，仅剩行为分析 |
| Arduino 中继（真鼠标→Arduino→PC） | 1 个设备 | **极难检测** | 目标电脑始终只看到一个设备 |
| 单设备 evdev 拦截 | 1 个设备 | **接近完美** | 事件源头完全一致 |

> **核心教训：两个设备的切换模式，比单个假设备更容易被检测。** 如果你决定用硬件方案，最好的做法不是"真鼠标 + 假鼠标混用"，而是**全程只用一个设备**。

---

## 9. 行为随机化技术

无论用哪个方案，都**必须加入行为随机化**。

### 8.1 点击间隔随机化

```rust
use rand::Rng;
use std::{thread, time::Duration};

fn random_sleep(rng: &mut impl Rng, base_ms: u64, jitter_ms: u64) {
    let ms = base_ms + rng.gen_range(0..=jitter_ms);
    thread::sleep(Duration::from_millis(ms));
}
```

### 8.2 鼠标轨迹贝塞尔曲线

```rust
/// 生成二次贝塞尔曲线上的点
fn bezier_point(
    p0: (f64, f64),
    p1: (f64, f64),
    p2: (f64, f64),
    t: f64,
) -> (f64, f64) {
    let x = (1.0 - t).powi(2) * p0.0 + 2.0 * (1.0 - t) * t * p1.0 + t.powi(2) * p2.0;
    let y = (1.0 - t).powi(2) * p0.1 + 2.0 * (1.0 - t) * t * p1.1 + t.powi(2) * p2.1;
    (x, y)
}

/// 生成自然的鼠标路径
fn generate_human_path(
    from: (f64, f64),
    to: (f64, f64),
    steps: usize,
) -> Vec<(f64, f64)> {
    let mut rng = rand::thread_rng();

    // 控制点加随机偏移，让曲线不完美
    let cp_x = (from.0 + to.0) / 2.0 + rng.gen_range(-100.0..=100.0);
    let cp_y = (from.1 + to.1) / 2.0 + rng.gen_range(-100.0..=100.0);

    (0..=steps)
        .map(|i| bezier_point(from, (cp_x, cp_y), to, i as f64 / steps as f64))
        .collect()
}
```

### 8.3 点击位置微扰

```rust
fn perturb_click(x: i32, y: i32) -> (i32, i32) {
    let mut rng = rand::thread_rng();
    (
        x + rng.gen_range(-3..=3),
        y + rng.gen_range(-3..=3),
    )
}
```

### 8.4 完整集成示例

```rust
fn humanized_interaction(
    port: &mut Box<dyn serialport::SerialPort>,
    target_x: i32,
    target_y: i32,
    current_x: i32,
    current_y: i32,
) {
    let mut rng = rand::thread_rng();

    // 1. 生成自然的移动路径（20-40 步）
    let path = generate_human_path(
        (current_x as f64, current_y as f64),
        (target_x as f64, target_y as f64),
        rng.gen_range(20..=40),
    );

    // 2. 沿路径发送移动指令（每步 2-8ms 随机间隔）
    for point in &path {
        send_move(port, point.0 as i16, point.1 as i16);
        thread::sleep(Duration::from_millis(rng.gen_range(2..=8)));
    }

    // 3. 到达后停顿（模拟"确认目标"）
    thread::sleep(Duration::from_millis(rng.gen_range(80..=250)));

    // 4. 点击（带位置微扰）
    let (cx, cy) = perturb_click(target_x, target_y);
    send_move(port, cx as i16, cy as i16);
    send_click(port);
}
```

---

## 10. 实践建议总结

| 场景 | 推荐方案 |
|------|---------|
| 单机游戏 / 模拟器 | enigo + opencv，零成本 |
| MMO 采集 / 钓鱼脚本 | uinput + 行为随机化，低风险 |
| 网页游戏 / 放置游戏 | enigo 或直接用 JavaScript |
| 竞技游戏（FPS/MOBA） | 任何软件方案都有封号风险，硬件方案仅有行为分析破绽 |
| 办公自动化 / 测试 | enigo 或 autopilot，这是正经用途 |

---

## 11. 常见问题

### Q1: Wayland 下用哪个方案？

Wayland 禁用了 `XSendEvent` / `XTest` 等 X11 机制，也限制了第三方截图。推荐：
- **截图**：PipeWire（`xdg-desktop-portal`）或 `screenshots` crate 的 Wayland 后端
- **输入模拟**：`libei` 协议（需要 compositor 支持），或退回到 uinput / 硬件方案
- **最省心**：在 Wayland 下直接上硬件方案（Arduino/Pi Pico）

### Q2: enigo 的 Relative 和 Absolute 坐标有什么区别？

| 模式 | 含义 | 适用 |
|------|------|------|
| `Coordinate::Abs` | 屏幕绝对坐标 (0,0) = 左上角 | 知道目标确切位置 |
| `Coordinate::Rel` | 相对当前位置偏移 (dx, dy) | 模拟鼠标自然移动 |

### Q3: 如何让程序不被反作弊检测到？

**在运行反作弊的同一台机器上运行脚本，本质上没有完美方法。** 反作弊在内核态，能看到一切。可操作的优化方向：
1. 用硬件方案（Arduino/Pi Pico）隔离两个系统——这是最接近"不被检测"的方法
2. 行为随机化不能省——即使硬件方案也需要
3. 不要 24 小时运行

### Q4: 能不能用 QEMU 虚拟机 + USB 透传？

可以。在虚拟机里运行脚本，把 Arduino/Pi Pico 通过 USB 透传进虚拟机。但这引入了额外延迟（~10-30ms），且虚拟机本身也可能被检测（Hyper-V/VirtualBox 等虚拟化特征）。

### Q5: 怎么判断目标是否使用了内核级反作弊？

```bash
# 检查加载的内核模块
lsmod | grep -iE "vanguard|vgk|easyanticheat|battleye|faceit|equ8"

# 检查运行中的反作弊进程
ps aux | grep -iE "vanguard|vgk|easyanticheat|battleye|faceit"

# 检查 /dev 中的反作弊设备
ls /dev | grep -iE "vgk|vanguard|eac|faceit"
```

---

## 12. 参考资源

- [enigo crate](https://crates.io/crates/enigo) — 跨平台输入模拟
- [evdev crate](https://crates.io/crates/evdev) — Linux evdev 接口
- [screenshots crate](https://crates.io/crates/screenshots) — 跨平台截图
- [opencv-rust](https://github.com/twistedfall/opencv-rust) — OpenCV Rust 绑定
- [Linux USB Gadget 文档](https://www.kernel.org/doc/Documentation/usb/gadget_configfs.rst)
- [usbd-hid](https://crates.io/crates/usbd-hid) — 嵌入式 USB HID 实现
- [rp2040-hal](https://crates.io/crates/rp2040-hal) — Pi Pico Rust HAL
