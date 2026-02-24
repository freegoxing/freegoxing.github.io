---
title: FPGA 学习笔记（二）：组合逻辑基础——译码器实现
date: 2026-02-24 21:00:00
categories: [FPGA]
tags: [Verilog, FPGA, 组合逻辑]
---

组合逻辑是数字电路的基础。通过 `01max2`（二选一选择器）和 `02decoder_3_8`（3-8 译码器）两个小项目，我们来看看 Verilog 的基本描述方式。

<!-- more -->
# 1. 2选1多路选择器（max2）

作为 FPGA 设计的“Hello World”，2选1多路选择器（MUX）是理解组合逻辑逻辑和 Verilog 模块化思维的最佳起点。

## 1.1 设计目标

实现一个逻辑选择器：当选择信号 `sel` 为 0 时，输出端口 `a` 的信号；当 `sel` 为 1 时，输出端口 `b` 的信号。

## 1.2 RTL 代码实现 (`rtl/max2.v`)
在 Verilog 中，一切逻辑都封装在 `module` 中。以下是标准的模块定义方式：

```verilog
// rtl/max2.v
module max2(
    input  a,    // 输入信号 a
    input  b,    // 输入信号 b
    input  sel,  // 选择信号
    output out   // 输出信号
);

    // 组合逻辑最常用的描述方式：连续赋值语句 assign
    // 使用三目运算符实现逻辑判断，简单高效
    assign out = (sel == 1'b1) ? b : a;

endmodule
```

**知识点解析：**
- **端口声明**：现代 Verilog 推荐直接在 `module` 括号内定义 `input/output` 及其类型，使代码更简洁。例如 `input a, output wire out;`。对于通过 `assign` 语句赋值的输出，通常隐式为 `wire`，但显式声明 `output wire` 更清晰；对于在 `always` 块中赋值的输出，则需声明为 `output reg`。
- **连续赋值 (`assign`)**：用于描述组合逻辑，它像焊线一样将等号右边的逻辑结果持续“驱动”到左边的 `wire` 变量上。
- **三目运算符**：`(条件) ? 真值 : 假值`，是实现简单选择逻辑的最佳方式。

## 1.3 仿真验证 (`sim/max2_tb.v`)

为了验证逻辑是否正确，我们需要编写测试脚本（Testbench）来模拟输入信号。

```verilog
// timescale <时间单位>/<精度>
`timescale 1ns/1ns 

module max2_tb();
    // 1. 激励信号定义
    // 在 TB 中，需要输入给被测模块的信号定义为 reg（因为要在 initial 块中手动修改）
    reg s0;
    reg s1;
    reg s_sel;

    // 2. 响应信号定义
    // 接收被测模块输出的信号定义为 wire
    wire max2_out;

    // 3. 模块实例化
    // 按照引脚名称（按名关联）将信号连接到被测模块
    max2 max2_inst0(
        .a(s0),
        .b(s1),
        .sel(s_sel),
        .out(max2_out)
    );

    // 4. 产生激励
    // initial 块里面的内容是仿真开始时最先执行的内容
    initial begin
        // 初始化信号
        s0 = 0; s1 = 0; s_sel = 0;
        #20; // 延时 20 个时间单位（20ns）
        s0 = 1; s1 = 0; s_sel = 0; // 预期输出 out = s0 = 1
        #20;
        s0 = 1; s1 = 0; s_sel = 1; // 预期输出 out = s1 = 0
        #20;
        s0 = 1; s1 = 1; s_sel = 1; // 预期输出 out = s1 = 1
        #20;
        ......
        $stop; // 停止仿真
    end

endmodule
```

**仿真要点总结：**
- **`timescale`**：例如 `1ns/1ps` 表示代码中的 `#1` 代表 1ns，而仿真器内部计算精度可以达到 1ps。
- **`reg` vs `wire`**：在仿真环境中，**输入信号**通常在 `initial` 块中赋值，必须定义为 `reg` 类型；而**输出信号**只是接收反馈，定义为 `wire`。
- **`initial` 块与 `begin-end`**：用于书写测试过程，按时间顺序执行。

# 2. 3-8 译码器（decoder_3_8）

3-8 译码器是组合逻辑中的经典案例，其核心逻辑是将 3 位二进制输入转换为 8 位互斥的输出（独热码）。

## 2.1 设计目标

根据输入信号 `{A2, A1, A0}` 的二进制值，选中对应的输出通道（Y0~Y7）。例如当输入为 `3'b000` 时，`Y0` 输出为 1，其余输出均为 0。

## 2.2 RTL 代码实现 (`rtl/decoder_3_8.v`)

在处理多分支逻辑时，使用 `always` 块配合 `case` 语句比 `assign` 更加直观。

```verilog
// rtl/decoder_3_8.v
module decoder_3_8(
    input A0,
    input A1,
    input A2,
    output reg Y0,
    output reg Y1,
    output reg Y2,
    output reg Y3,
    output reg Y4,
    output reg Y5,
    output reg Y6,
    output reg Y7
);

    // 过程赋值语句 always
    // @(*) 表示敏感列表包含块内所有涉及的输入信号
    always @(*) begin
        case({A2, A1, A0}) // 位拼接：将三个 1 位信号组合成 3 位矢量
            // 语法格式：位宽'进制代码 数值
            3'b000: {Y7,Y6,Y5,Y4,Y3,Y2,Y1,Y0} = 8'b0000_0001;    
            3'b001: {Y7,Y6,Y5,Y4,Y3,Y2,Y1,Y0} = 8'b0000_0010;    
			......
            default: {Y7,Y6,Y5,Y4,Y3,Y2,Y1,Y0} = 8'b0000_0000;
        endcase
    end

endmodule
```

{% note info %}

这里模块定义可以更加简明一些，我们如果使用总线来代替8个独立的输出信号，就有

```verilog
module decoder_3_8(
    input A0,
    input A1,
    input A2,
    output reg [7:0] Y
);

    // 过程赋值语句 always
    // @(*) 表示敏感列表包含块内所有涉及的输入信号
    always @(*) begin
        case({A2, A1, A0}) // 位拼接：将三个 1 位信号组合成 3 位矢量
            // 语法格式：位宽'进制代码 数值
            3'b000: Y = 8'b0000_0001;    
            3'b001: Y = 8'b0000_0010;    
			......
            default: Y = 8'b0000_0000;
        endcase
    end

endmodule
```

这种写法在实际工程中更常见，因为：
- 便于参数化扩展
- 便于约束文件绑定
- 易于与总线接口对接
- 更符合综合器优化习惯

{% endnote %}

**知识点解析：**

- **`always @(*)`**：组合逻辑的过程描述方式。这里的 `*` 是通配符，意味着任何输入信号的变化都会触发块内逻辑的重新计算。
- **`reg` 型变量**：在 `always` 块中被赋值的信号**必须**定义为 `reg` 类型。需要注意，在组合逻辑中，`reg` 并不代表触发器（Flip-Flop），它仅仅是 Verilog 的语法要求。
- **位拼接运算符 `{}`**：将多个独立信号按顺序“捆绑”在一起。这在处理总线或并列的输出端口时非常高效。
- **数值表达方式**：Verilog 中数值的格式为 `位宽'进制符号数值`。
    - `8'b0000_0100`：8 位二进制数。`b` 代表二进制 (binary)。
    - `4'd10`：4 位十进制数 10 (等同于 `4'b1010`)。`d` 代表十进制 (decimal)。
    - `12'hFAB`：12 位十六进制数 FAB (等同于 `12'b1111_1010_1011`)。`h` 代表十六进制 (hexadecimal)。
    - `3'o7`：3 位八进制数 7 (等同于 `3'b111`)。`o` 代表八进制 (octal)。
    - 下划线 `_` 仅用于提高可读性，在编译时会被忽略。
- **`case` 语句**：用于根据表达式的值，执行不同的代码块。
    - **基本语法**：`case (表达式) ... endcase`。
    - **匹配项**：每个 `case` 项的表达式必须与 `case` 关键字后的表达式类型和位宽匹配。
    - **`default`**：可选，当所有 `case` 项都不匹配时执行 `default` 后的代码。对于组合逻辑，强烈建议包含 `default` 项，以避免产生锁存器（latch）。

## 2.3 仿真验证 (`sim/decoder_3_8_tb.v`)

通过 Testbench 遍历 000 到 111 的所有组合，验证译码逻辑的正确性。

```verilog
`timescale 1ns/1ns

module decoder_3_8_tb();
    reg A0, A1, A2;
    wire Y0, Y1, Y2, Y3, Y4, Y5, Y6, Y7;

    // 实例化被测模块
    decoder_3_8 decoder_3_8_inst0(
        .A0(A0), .A1(A1), .A2(A2),
        .Y0(Y0), .Y1(Y1), .Y2(Y2), .Y3(Y3),
        .Y4(Y4), .Y5(Y5), .Y6(Y6), .Y7(Y7)
    );
    
    initial begin
        // 依次测试 8 种可能的输入组合
        A2=0; A1=0; A0=0; #20; 
        A2=0; A1=0; A0=1; #20;
        A2=0; A1=1; A0=0; #20;
        A2=0; A1=1; A0=1; #20;
        A2=1; A1=0; A0=0; #20;
        A2=1; A1=0; A0=1; #20;
        A2=1; A1=1; A0=0; #20;
        A2=1; A1=1; A0=1; #20;
        $stop; // 暂停仿真
    end 
    
endmodule
```

# 3. 总结

- **赋值选择**：在 `assign` 中赋值的变量定义为 `wire`，在 `always` 块中赋值的必须定义为 `reg`。
- **灵活性**：组合逻辑优先考虑 `assign` + 三目运算符；当逻辑复杂（分支多）时，使用 `always` + `case` 更利于维护。
- **验证意识**：每一个模块都应对应一个 Testbench，覆盖所有可能的逻辑边界。

