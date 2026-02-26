---
title: FPGA 学习笔记（五）：阻塞赋值和非阻塞赋值的比较
date: 2026-02-26 21:00:00
categories: [FPGA]
tags: [Verilog, FPGA, 阻塞赋值, 非阻塞赋值]
---

在 Verilog 时序逻辑设计中，正确理解 `=`（阻塞赋值）和 `<=`（非阻塞赋值）的区别是避免仿真与综合不一致、杜绝竞争冒险的关键。

<!-- more -->

# 1. 阻塞赋值（Blocking Assignment, `=`）

阻塞赋值的行为类似于 C 语言：语句按顺序执行。在同一个 `always` 块中，后面的语句必须等待前面的语句执行完毕后才能执行。

## 1.1 设计目标
通过两个对比模块 `block_0` 和 `block_1`，观察改变赋值顺序对最终输出逻辑的影响。

## 1.2 RTL 代码实现

```verilog
// 模块 0：先算中间变量 d，再算输出 out
module block_0(
    input Clk,
    input a, b, c,
    output reg[1:0] out
);
    reg [1:0] d;
    always @(posedge Clk) begin
        d = a + b;   // 第一步：更新 d
        out = d + c; // 第二步：使用更新后的 d 计算 out
    end
endmodule

// 模块 1：先算输出 out，再算中间变量 d
module block_1( 
    input Clk,
    input a, b, c,
    output reg[1:0] out
);
    reg [1:0] d;
    always @(posedge Clk) begin
        out = d + c; // 第一步：使用上一个时钟周期的 d 计算 out
        d = a + b;   // 第二步：更新 d
    end
endmodule
```

**知识点解析：**
- **逻辑行为**：在 `block_0` 中，`out` 在同一个时钟上升沿就拿到了 `a+b+c` 的结果；而在 `block_1` 中，`out` 拿到的是“旧的 d”与 `c` 相加的结果。
- **顺序依赖**：在阻塞赋值中，语句的书写顺序直接决定了电路的逻辑功能。

# 2. 非阻塞赋值（Non-blocking Assignment, `<=`）

非阻塞赋值的特点是“并行更新”：在时钟有效沿到来时，所有等号右边的表达式先计算，在时序步结束时统一更新到左边的变量中。

## 2.1 设计目标
通过 `noblock_0` 和 `noblock_1` 验证非阻塞赋值的并行特性和顺序无关性。

## 2.2 RTL 代码实现

```verilog
// 模块 0：先写 d，再写 out
module noblock_0(
    input Clk,
    input a, b, c,
    output reg[1:0] out
);
    reg [1:0] d;
    always @(posedge Clk) begin
        d <= a + b;   // 计划更新 d
        out <= d + c; // 计划更新 out，使用的是 d 的旧值
    end
endmodule

// 模块 1：先写 out，再写 d
module noblock_1(
    input Clk,
    input a, b, c,
    output reg[1:0] out
);
    reg [1:0] d;
    always @(posedge Clk) begin
        out <= d + c; // 计划更新 out，使用的是 d 的旧值
        d <= a + b;   // 计划更新 d
    end
endmodule
```

**知识点解析：**
- **并行更新**：在非阻塞赋值中，所有的赋值动作被视为同时发生。`out` 拿到的永远是时钟沿到来前 `d` 的旧值。
- **顺序无关性**：对比 `noblock_0` 和 `noblock_1` 可以发现，改变语句顺序并不会改变逻辑结果，这对于描述硬件的并行特性至关重要。

# 3. 仿真验证与波形对比

为了直观观察四者的区别，我们可以编写一个统一的 Testbench。

## 3.1 测试脚本 (`sim/block_noblock_tb.v`)

```verilog
`timescale 1ns/1ps

module block_noblock_tb ();
    reg Clk;
    reg a, b, c;
    wire [1:0] out0, out1, out2, out3;

    // 实例化四个对比模块
    noblock_0 noblock_0_inst(.Clk(Clk), .a(a), .b(b), .c(c), .out(out0));
    noblock_1 noblock_1_inst(.Clk(Clk), .a(a), .b(b), .c(c), .out(out1));
    block_0   block_0_inst(  .Clk(Clk), .a(a), .b(b), .c(c), .out(out2));
    block_1   block_1_inst(  .Clk(Clk), .a(a), .b(b), .c(c), .out(out3));

    initial Clk = 1;
    always #10 Clk = ~Clk; // 50MHz 时钟周期

    initial begin
        a = 0; b = 0; c = 0;
        #201;
        a = 1; b = 1; c = 1; // 触发变化
        #200;
        a = 0; b = 1; c = 0;
        #200;
        $stop;
    end
endmodule
```

## 3.2 仿真结果分析

通过波形观察

![不同模块的波形仿真](https://img.556756.xyz/PicGo/blogs/2026/02/20260226215724978.png)

当输入信号 `{a, b, c}` 从 `000` 变为 `111`（发生于第 201ns，第一个生效时钟沿在 210ns），再变为 `010` 时，各模块的输出呈现出显著的序列差异：

*   **`out2` (block_0)**：变化序列为 `0 -> 3 -> 1`。
*   **`out0, out1, out3` (noblock & block_1)**：变化序列为 `0 -> 1 -> 3 -> 2 -> 1`。

{% note info %}
**深度解析：**

1.  **为什么 `block_1` 表现得像非阻塞赋值？**
    在 `block_1` 中，代码先执行 `out = d + c` 后执行 `d = a + b`。在时钟上升沿到来时，`out` 率先读取了 `d` 的**旧值**（上一个周期的结果）进行计算，随后 `d` 才被更新。这种由于“书写顺序”导致的逻辑延迟，在功能上模拟了非阻塞赋值（及触发器级联）的行为。
2.  **为什么所有模块的 `d` 数值都一样？**
    因为无论采用哪种赋值方式，在每个时钟沿处理结束后的“稳定状态”下，`d` 最终都被赋予了当前时刻 `a+b` 的计算结果。**差别不在于 `d` 本身，而在于 `out` 计算时使用的是 `d` 的“更新前”还是“更新后”的值。**
3.  **流水线效应（Pipeline）**：
    *   在 `block_0` 中，`out` 在同一个时钟沿就拿到了 `a+b+c` 的完整结果。
    *   在非阻塞赋值模块中，输入的变化需要经过两个时钟沿的“接力”才能完全传递到 `out`（第一次沿 `d` 变，第二次沿 `out` 变）。这在硬件上对应了两个级联的触发器，即流水线结构。
    {% endnote %}

# 4. 总结：Verilog 建模的“黄金法则”

为了保证设计的可靠性并确保仿真与综合一致，请务必遵守以下原则：

1. **时序逻辑**（`always @(posedge Clk)`）：一律使用 **非阻塞赋值 `<=`**。
2. **组合逻辑**（`always @(*)`）：一律使用 **阻塞赋值 `=`**。
3. **禁止混用**：在同一个 `always` 块中，严禁混用 `=` 和 `<=`。
4. **单一赋值**：同一个变量不应在多个 `always` 块中被赋值。

遵守这些法则，可以确保你的代码在仿真器（Simulation）和综合器（Synthesis）中表现出完全一致的行为。
