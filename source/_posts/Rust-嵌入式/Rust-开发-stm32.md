---
title: Rust 开发 stm32
date: 2026-02-23 15:01:25
tags: [ Rust, STM32, Embedded, MCU, Embassy, HAL, defmt, probe-rs ]
categories: [ Rust-嵌入式 ]  

---

<!-- more -->

# 前言

Rust 以其内存安全、零成本抽象和现代化的工具链正在席卷嵌入式开发领域。本文将详细探讨如何从零开始，使用 Rust 语言开发 STM32 单片机。

# 环境搭建

在使用 Rust 开发 STM32 之前，我们需要配置好相关的开发工具链。

## 安装 Rust 工具链
首先，确保已安装 Rust 编译器及其包管理工具：
```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

## 设置 Rust night 版本

Rust 的嵌入式开发环境需要是 nightly 版本

```bash
rustup install nightly
```

## 安装目标平台

你需要根据 MCU 的内核安装对应的交叉编译目标。STM32 常见的内核对应的 target 如下：
- **Cortex-M0/M0+/M1**: `thumbv6m-none-eabi`
- **Cortex-M3**: `thumbv7m-none-eabi`
- **Cortex-M4/M7 (无 FPU)**: `thumbv7em-none-eabi`
- **Cortex-M4/M7 (带 FPU)**: `thumbv7em-none-eabihf`

执行安装命令（以 STM32F103 为例）：
```bash
rustup target add thumbv7m-none-eabi
```

## 必要的辅助工具
- **probe-rs**: 嵌入式开发“瑞士军刀”，支持烧录、RTT 调试。一行 `cargo run` 的核心。
  ```bash
  cargo binstall probe-rs-tools
  ```
- **flip-link**: 增强安全性，防止栈溢出覆盖代码。
  ```bash
  cargo install flip-link
  ```

# 实战：创建 Embassy 异步项目

我们将以 STM32F103C8T6 (Blue Pill) 为例，展示一个现代化的 Embassy 异步项目配置。

## 项目文件结构

典型的基于 Embassy 的 Rust 嵌入式项目目录结构如下：

```text
demo/
├── .cargo/
│   └── config.toml    # 烧录器 runner 与编译目标配置
├── src/
│   └── main.rs         # 异步主程序入口
├── build.rs            # 链接脚本参数传递
├── Cargo.toml          # 依赖管理 (path 或 git)
├── rust-toolchain.toml # 锁定 rustc 版本与组件
└── memory.x            # 可选：定义 MCU Flash 与 RAM 布局 
```

{% note warning %}

在 Embassy 中，如果你在 `Cargo.toml` 的 `embassy-stm32` 特性中启用了 `memory-x`   并且指定了芯片型号，Embassy 会自动为你生成内存布局，此时你不需要手动创建 `memory.x` 文件。

{% endnote %}

## 项目配置 (Cargo.toml)

为了避免频繁的网络下载，我们可以将 Embassy 仓库克隆到本地并使用 `path` 引入。同时，配置合理的编译 Profile：

```toml
[dependencies]
# 使用本地路径引入 Embassy，方便调试和离线开发
embassy-stm32 = { path = "../embassy/embassy-stm32", features = ["defmt", "stm32f103c8", "unstable-pac", "memory-x", "time-driver-any"] }
embassy-executor = { path = "../embassy/embassy-executor", features = ["arch-cortex-m", "executor-thread", "defmt"] }
embassy-time = { path = "../embassy/embassy-time", features = ["defmt", "defmt-timestamp-uptime", "tick-hz-32_768"] }

defmt = "0.3"
defmt-rtt = "0.4"
panic-probe = { version = "0.3", features = ["print-defmt"] }

[profile.release]
debug = true            # 裸机开发建议保留调试信息，无运行时开销
opt-level = "z"         # 极致优化体积
lto = "fat"             # 开启全量链接时优化
```

## 烧录配置 (.cargo/config.toml)

配置 `probe-rs` 作为 runner，可以实现一行命令完成“编译+烧录+日志打印”：

```toml
[target.'cfg(all(target_arch = "arm", target_os = "none"))']
runner = "probe-rs run --chip STM32F103C8"

[build]
target = "thumbv7m-none-eabi" # Cortex-M3 目标

[env]
DEFMT_LOG = "trace" # 设置日志级别
```

## 链接脚本处理 (build.rs)

在 `build.rs` 中通过命令行向编译器传递链接参数，确保 `defmt` 和 `link.x` 脚本生效：

```rust
fn main() {
    println!("cargo:rustc-link-arg-bins=--nmagic");
    println!("cargo:rustc-link-arg-bins=-Tlink.x");
    println!("cargo:rustc-link-arg-bins=-Tdefmt.x");
}
```

## 核心代码实现 (src/main.rs)

利用 Embassy 的 `async/await` 编写一个带日志输出的 LED 闪烁程序：

```rust
#![no_std]
#![no_main]

use defmt::*;
use embassy_executor::Spawner;
use embassy_stm32::gpio::{Level, Output, Speed};
use embassy_time::Timer;
use {defmt_rtt as _, panic_probe as _};

#[embassy_executor::main]
async fn main(_spawner: Spawner) {
    let p = embassy_stm32::init(Default::default());
    info!("MCU 初始化完成，Hello Embassy!");

    // 初始化 PA0 引脚为输出
    let mut led = Output::new(p.PA0, Level::High, Speed::Low);

    loop {
        info!("LED: High");
        led.set_high();
        Timer::after_millis(300).await;

        info!("LED: Low");
        led.set_low();
        Timer::after_millis(300).await;
    }
}
```

# 烧录与调试

现在，你只需要连接好 ST-Link，然后在项目根目录下执行：

```bash
cargo run --release
```

`probe-rs` 会自动下载固件，并直接在终端显示 `defmt` 的彩色日志。这种体验非常接近高级语言的开发流程。

# 官方示例与参考

Embassy 官方仓库提供了极为丰富的示例项目，涵盖了从基础 GPIO 到复杂的 USB、TCP/IP 网络协议栈的应用。

- **GitHub 示例地址**: [embassy-rs/embassy/examples](https://github.com/embassy-rs/embassy/tree/main/examples)

以 `examples/stm32f1` 为例，你可以看到各种常见外设的使用方法：

```text
examples/stm32f1/src/bin/
├── adc.rs           # 模数转换
├── blinky.rs        # 基础 LED 闪烁
├── can.rs           # CAN 总线通信
├── hello.rs         # 简单的 Hello World
├── input_capture.rs # 输入捕获
├── pwm_input.rs     # PWM 输入频率/占空比检测
└── usb_serial.rs    # USB 虚拟串口
```

Embassy 的支持范围非常广泛，不仅包括你正在使用的 STM32 系列（如 F0, F1, F3, F4, G0, G4, H7, L0, L4, U5, WL 等），还支持 nRF 系列、RP2040 (Raspberry Pi Pico) 等。

如果你在开发中遇到某个外设不知道如何配置，直接在 `examples/stm32xx` 目录下搜索对应的代码，通常都能找到最标准、性能最优的实现参考。

以及现在存在许多关于外设的 rust 库如( ssd1306 屏幕驱动等) 可以在[crates.io](https://crates.io/)里面搜索

# 总结

通过 Embassy + probe-rs + defmt，Rust 为 STM32 开发提供了一套安全、高效且现代化的工具链。虽然配置阶段略显繁琐，但一旦跑通，其带来的异步并发优势和调试便利性是传统 C 语言开发难以比拟的。

虽然 Rust 的学习曲线较高，但其在编译阶段捕获错误的能力和极佳的代码可读性，能显著提升嵌入式开发效率。
