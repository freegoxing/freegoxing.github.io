---
title: FPGA 学习笔记（三）：时序逻辑基础——D 触发器与计数器
date: 2026-02-25 10:00:00
categories: [FPGA]
tags: [Verilog, FPGA, 时序逻辑]
math: true
---

与组合逻辑不同，时序逻辑引入了“时间”的概念。通过 D 触发器与时钟信号，我们可以让电路具有“记忆”功能，并实现精确的时间控制。

<!-- more -->

# 1. 时序逻辑的核心：D 触发器 (DFF)

D 触发器（D Flip-Flop）是构成时序逻辑电路的最基本单元，也是 FPGA 内部最主要的资源之一。

## 1.1 工作原理

D 触发器在时钟信号（Clock）的特定边沿（通常是上升沿 `posedge`）捕获输入端 D 的电平，并将其传递到输出端 Q。在两个时钟边沿之间的时间段内，无论 D 端如何变化，输出端 Q 都会保持之前的状态。

## 1.2 Verilog 描述方式

在 Verilog 中，时序逻辑必须编写在 `always` 块中，并使用特定的边沿触发。

```verilog
always @(posedge Clk) begin
    Q <= D; // 在 Clk 上升沿，将 D 的值赋给 Q
end
```

**关键规则：**
- **同步性**：逻辑的变化只发生在时钟边沿。
- **非阻塞赋值 (`<=`)**：在描述时序逻辑时，**必须**使用 `<=`。它意味着块内的赋值是并行发生的，模拟了硬件中寄存器的真实行为。

---

# 2. LED 闪烁灯 (led_twinkle)

通过计数器对高速系统时钟进行计数，我们可以实现分频功能，从而产生人眼可见的低频信号（如 LED 闪烁）。

## 2.1 设计目标
实现一个 LED 闪烁灯：每隔 0.5 秒翻转一次 LED 状态。假设实验板的系统时钟为 50MHz。

## 2.2 RTL 代码实现 (`rtl/led_twinkle.v`)

```verilog
module led_twinkle(
    Clk,
    Reset_n,
    Led
);
    
    input Clk;
    input Reset_n;
    output reg Led;

    // 声明位宽范围：25,000,000 需要 25 位宽 (2^25 > 25,000,000)
    reg [24:0] counter;

    // 计数器逻辑
    // posedge Clk：在时钟上升沿触发
    // negedge Reset_n：在复位信号下降沿触发（异步复位）
    always @(posedge Clk or negedge Reset_n)
    if (!Reset_n) // 当复位信号为低电平时，重置计数器
        counter <= 0; // 非阻塞赋值
    else if (counter == 25_000_000-1) // 当计数器达到 25,000,000 次 (0.5s) 时
        counter <= 0;
    else
        counter <= counter + 1'd1; // 否则，计数器加 1

    // LED 翻转逻辑
    always @(posedge Clk or negedge Reset_n)
    if (!Reset_n) // 当复位信号为低电平时，关闭 LED
        Led <= 1'b0;
    else if(counter == 25_000_000-1)
        Led <= !Led; // 当计数器达到设定值时，翻转 LED 状态
        
endmodule
```

**知识点解析：**

- **异步复位 (`negedge Reset_n`)**：敏感列表中包含了复位信号的边沿。这意味着无论时钟是否到来，只要复位按键按下，电路都会立即重置。

- **计数器原理**：FPGA 中没有 `delay()` 函数。我们通过计算时钟脉冲的个数来衡量时间。对于 50MHz 时钟，周期为 20ns。$0.5s / 20ns = 25,000,000$ 个周期。

    ![](https://img.556756.xyz/PicGo/blogs/2026/02/20260225092307045.png)

{% note info %}
**重点：为什么是 25,000,000 - 1？**

初学者常以为“执行复位语句需要消耗一个时钟周期”，所以要提前一个数减 1。**这是错误的理解！**

在 FPGA 硬件设计中，逻辑是并行的，`counter` 的累加、判定和重置是在时钟边沿同步发生的电路行为。之所以减 1，纯粹是因为**计数是从 0 开始的**：

- **计 1 个数**：范围是 `[0]`，判定条件是 `counter == 0`
- **计 2 个数**：范围是 `[0, 1]`，判定条件是 `counter == 1`
- **计 N 个数**：范围是 `[0, N-1]`，判定条件是 `counter == N-1`

如果在计数器已经达到 `N` 时才去重置，那么计数器实际上经历了 `0, 1, ..., N-1, N` 共 **N+1** 个状态，这会导致最终的时间比预定时间**多出一个时钟周期**。这在精密的时序控制（如串口波特率、时钟分频）中会导致累积误差。

如果没有减去一，则会增加一个计数/时钟周期，这里多 $2 \times 2000 \mathrm{ps} = 4000 \mathrm{ps}$，即对应 $50 \mathrm{MHz}$ 

![](https://img.556756.xyz/PicGo/blogs/2026/02/20260225092910309.png)

{% endnote %}

## 2.3 仿真验证 (`sim/led_twinkle_tb.v`)

为了验证代码逻辑，我们需要在 Testbench 中产生模拟时钟。

```verilog
`timescale 1ns/1ns

module led_twinkle_tb ();
    reg Clk;
    reg Reset_n;
    wire Led;

    // 实例化被测试模块 (Unit Under Test)
    led_twinkle uut (
        .Clk(Clk),
        .Reset_n(Reset_n),
        .Led(Led)
    );

    // 时钟生成：每 20ns 一个周期，即 50MHz
    initial Clk = 1;
    always #10 Clk = ~Clk; // 每 10ns 翻转一次时钟信号

    // 测试序列
    initial begin
        // 初始化信号
        Reset_n = 0; // 复位信号初始为低电平
        #201;        // 等待 201ns，确保避开初始不确定态
        Reset_n = 1; // 释放复位信号，开始正常工作
        
        // 观察一段时间的波形
        #2000;
        // $stop;
    end
    
endmodule
```

---

# 3. 总结

- **核心差异**：组合逻辑是输入决定输出（实时），时序逻辑是时钟决定输出（同步）。
- **赋值习惯**：时序逻辑永远使用 `<=` (Non-blocking)。
- **硬件意识**：在 FPGA 中计数，永远记住“从 0 开始”，因此计 $N$ 个数需判定 $N-1$。
