---
title: FPGA 学习笔记（四）：模块化思维与参数化设计
date: 2026-02-26 10:00:00
categories: [FPGA]
tags: [Verilog, FPGA, 模块化, 参数化]
---

在复杂的 FPGA 工程中， **模块化（Modularity）** 与 **参数化（Parameterization）** 是衡量代码质量的核心指标。本文通过两个核心案例，重点展示如何通过参数传递提升模块的可复用性。

<!-- more -->

# 1. 多方案实现流水灯设计

作为时序逻辑的进阶实验，流水灯设计是理解“参数化驱动硬件”的最佳实践。我们将探讨三种不同的实现思路。

## 1.1 设计目标

实现 8 位 LED 流水灯，每隔 0.5s（50MHz 时钟下）亮灯位置移动一位，循环往复。

## 1.2 RTL 代码实现

### 方案一：线性位移 (Linear Shift)

利用位移运算符 `<<` 实现灯光的依次点亮。

```verilog
module led_run #(
    parameter MCNT = 25_000_000 - 1  // 默认 0.5s (50MHz)
)(
    input            Clk,
    input            Rst_n,
    output reg [7:0] Led
);
    reg [24:0] counter;

    always @(posedge Clk or negedge Rst_n) 
    if (!Rst_n)
        counter <= 0;
    else if (counter == MCNT) 
        counter <= 0;
    else
        counter <= counter + 1;

    always @(posedge Clk or negedge Rst_n)
    if (!Rst_n)
        Led <= 8'b0000_0001; 
    else if (counter == MCNT) begin
        if (Led == 8'b1000_0000)
            Led <= 8'b0000_0001; 
        else
            Led <= Led << 1; 
    end
endmodule
```

### 方案二：循环位移 (Circular Shift)
利用位拼接符实现环形移位，代码更简洁。

```verilog
always @(posedge Clk or negedge Rst_n)
if (!Rst_n)
    Led <= 8'b0000_0001;
else if (counter == MCNT)
    Led <= {Led[6:0], Led[7]}; // 将最高位拼接到最低位
```

### 方案三：模块化复用（译码器法）
实例化一个通用的 3-8 译码器，通过状态计数器驱动地址。

```verilog
// 顶层模块
module led_run_decoder #(parameter MCNT = 25_000_000-1)(
    input Clk, Rst_n,
    output [7:0] Led
);
    reg [24:0] counter;
    reg [2:0]  state; 
    // ... 计数逻辑 ...
    
    // 实例化译码器
    decoder_3_8 u_decoder_0 (.Addr(state), .Y(Led));
endmodule
```

**知识点解析：**
- **Verilog-2001 端口声明**：直接在 `module` 括号内定义 `input/output` 及其类型，简洁高效。
- **参数 (`parameter`)**：使用参数定义计数值，方便在不同频率的时钟下快速适配。
- **位拼接运算符 `{}`**：这是实现循环移位、数据重组的最强武器。

## 1.3 仿真验证
在仿真中，我们通过修改 `MCNT` 参数来缩短仿真时间。

```verilog
`timescale 1ns/1ns
module led_run_tb();
    reg Clk, Rst_n;
    wire [7:0] Led;

    // 参数覆写实例化
    led_run #(.MCNT(25_000-1)) uut (
        .Clk(Clk), .Rst_n(Rst_n), .Led(Led)
    );

    initial begin
        Clk = 1; Rst_n = 0; #201; Rst_n = 1;
        #1_000_000; $stop;
    end
    always #10 Clk = ~Clk;
endmodule
```

---

# 2.  不同速度闪烁灯（参数层层传递）

本案例展示如何利用同一个底层模块，通过参数传递实现差异化的硬件行为。

## 2.1 设计目标
实现 4 路 LED 以不同的频率闪烁。通过顶层模块对底层 `Led_twinkle` 模块进行 4 次实例化，并分别赋予不同的计数值。

## 2.2 RTL 代码实现

### 底层模块：通用闪烁灯 (`Led_twinkle.v`)
```verilog
module Led_twinkle (
    input Clk,
    input Rst_n,
    output reg Led
);
    reg [25:0] cnt;
    parameter MCNT = 25_000_000-1;

    always @(posedge Clk or negedge Rst_n)         
    if (!Rst_n)
        cnt <= 0;
    else if (cnt == MCNT)
        cnt <= 0;
    else
        cnt <= cnt + 1'd1;

    always @(posedge Clk or negedge Rst_n) 
    if (!Rst_n)
        Led <= 0;
    else if (cnt == MCNT)
        Led <= ~Led;
endmodule
```

### 顶层模块：参数分发器 (`led_twinkle_diffirent_speed.v`)
```verilog
module led_twinkle_diffirent_speed (
    input Clk,
    input Rst_n,
    output [3:0] Led
);
    parameter MCNT0 = 25_000_000-1;
    parameter MCNT1 = 12_500_000-1;  
    parameter MCNT2 = 6_250_000-1;
    parameter MCNT3 = 2_500_000-1;

    // 方法一：使用 #() 进行实例化传递（推荐）
    Led_twinkle #(.MCNT(MCNT0)) led_inst0 (
        .Clk(Clk), 
        .Rst_n(Rst_n), 
        .Led(Led[0])
    );
    
    Led_twinkle #(.MCNT(MCNT1)) led_inst1 (
        .Clk(Clk), 
        .Rst_n(Rst_n), 
        .Led(Led[1])
    );

    // 方法二：使用 defparam 进行路径传递
    Led_twinkle led_inst2 (
        .Clk(Clk), 
        .Rst_n(Rst_n), 
        .Led(Led[2])
    );
    defparam led_inst2.MCNT = MCNT2;

    Led_twinkle led_inst3 (
        .Clk(Clk), 
        .Rst_n(Rst_n), 
        .Led(Led[3])
    );
    defparam led_inst3.MCNT = MCNT3;
endmodule
```

**知识点解析：**
- **参数传递的层级性**：参数可以从 Testbench 传递到顶层，再由顶层传递到底层，形成参数链。
- **`#()` 语法 vs `defparam`**：
    - **`#()`**：在实例化时直接绑定，可读性好，是现代设计的主流。
    - **`defparam`**：通过绝对层级路径修改，适合大型工程中的全局参数调整或仿真调试。

## 2.3 仿真验证

编写 Testbench 验证 4 路信号的翻转频率是否符合预期。

```verilog
`timescale 1ns/1ns
module led_twinkle_diffirent_speed_tb ();
    reg Clk, Reset_n;
    wire [3:0] Led;

    // 再次通过层层传递，将计数值改为仿真级大小
    led_twinkle_diffirent_speed #(
        .MCNT0 (25_000-1),
        .MCNT1 (12_500-1),
        .MCNT2 (6_250-1),
        .MCNT3 (2_500-1)   
    ) inst (.Clk(Clk), .Rst_n(Reset_n), .Led(Led));

    initial begin
        Clk = 1; Reset_n = 0; #201; Reset_n = 1;
        #200_000_000; $stop;
    end
    always #10 Clk = ~Clk;
endmodule
```

---

# 3. 总结

- **模块化**：将通用逻辑（如闪烁、译码）封装，通过实例化构建复杂系统。
- **参数化**：通过 `parameter` 隔离逻辑与具体的硬件物理常数（如频率、位宽）。
- **传递技巧**：熟练掌握 `#()` 传递方式是编写高质量、可复用 Verilog 代码的关键。
