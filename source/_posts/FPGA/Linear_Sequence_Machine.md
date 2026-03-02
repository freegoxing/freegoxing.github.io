---
title:  FPGA 学习笔记（六）：时序逻辑进阶——线性序列机 (LSM)
date: 2026-03-2 21:00:00
categories: [FPGA]
tags: [Verilog, FPGA, 时序逻辑,线性序列机]

---

在复杂的时序控制中，如果仅使用简单的计数器或复杂的有限状态机（FSM），代码往往会变得难以维护。**线性序列机（Linear Sequence Machine, LSM）** 提供了一种折中且高效的方案：它以“时间”为轴，将复杂的逻辑分解为在特定时间点执行的确定动作。

<!-- more -->

# 1. 什么是线性序列机？

线性序列机是 FPGA 设计中一种非常重要的思想。其核心逻辑是：**利用一个或多个计数器产生一个不断推进的“时间轴”，在时间轴的特定刻度（Count Value）上触发特定的信号翻转或逻辑操作。**

**优点：**

- **结构清晰**：逻辑按时间顺序排列，易于阅读。
- **易于调试**：通过仿真观察计数器值即可快速定位问题点。
- **适用广泛**：常用于 SPI、I2C 驱动协议实现，以及各种需要精确脉冲控制的场景。


# 2. 基础：定时翻转序列 (led_ctrl0)

最简单的线性序列机就是通过一个主计数器，在不同的计数值点对输出信号进行赋值。

## 2.1 设计目标

实现一个周期为 1s 的 LED 控制：前 0.25s 亮，后 0.75s 灭。

## 2.2 RTL 代码实现

```verilog
// rtl/led_ctrl0.v
module led_ctrl0(
    input Clk,
    input Reset_n,
    output reg Led
);
    
reg [25:0] counter;
parameter MCNT0 = 50_000_000 - 1; // 总周期 1s
parameter MCNT1 = 12_500_000;     // 翻转点 0.25s

// 1. 产生时间轴：主计数器
always @(posedge Clk or negedge Reset_n) begin
	if (!Reset_n) 
        counter <= 0; 
	else if (counter == MCNT0) 
        counter <= 0;
	else 
        counter <= counter + 1'd1;
end

// 2. 在特定刻度点操作逻辑
always @(posedge Clk or negedge Reset_n) begin
    if (!Reset_n) 
        Led <= 1'b0;
    else if(counter == 0) 
        Led <= 1'b1;      // 0时刻点亮
    else if(counter == MCNT1) 
        Led <= 1'b0; // 0.25s时刻熄灭
end 
endmodule
```

**知识点解析：单一计数器序列**

- **时间轴映射**：计数器 `counter` 本质上是一个“进度条”。通过 `if(counter == 0)` 和 `if(counter == MCNT1)`，我们将连续的时间轴切割成了离散的动作点。

# 3. 进阶：嵌套计数器与查找表序列 (led_ctrl1)

当序列变得复杂时，直接在主计数器里写 `if-else` 会非常繁琐。此时我们可以将时间轴进行分段：**小计数器计基本刻度，大计数器计状态步骤。**

## 3.1 设计目标

实现一个复杂的闪烁序列，如：0.25s亮 -> 0.25s灭 -> 0.25s灭 -> 0.25s亮...（共10个状态）。

## 3.2 RTL 代码实现

```verilog
// rtl/led_ctrl1.v
module led_ctrl1(     
    input Clk,
    input Reset_n,
    output reg Led
);
// 定义基本时间刻度：0.25s
reg [24:0] counter0;
parameter MCNT = 12_500_000 - 1;

always@(posedge Clk or negedge Reset_n) begin
    if (!Reset_n) 
        counter0 <= 0;
    else if (counter0 == MCNT) 
        counter0 <= 0;
    else 
        counter0 <= counter0 + 1'd1;
end

// 定义步骤计数器：在基本刻度结束时加1
reg [3:0] counter1;
always@(posedge Clk or negedge Reset_n) begin
    if (!Reset_n) 
        counter1 <= 0;
    else if (counter0 == MCNT) begin
    	if (counter1 == 9) 
            counter1 <= 0;
    	else 
            counter1 <= counter1 + 1'd1;
	end
end

// 使用 case 语句构建“线性序列”
always@(posedge Clk or negedge Reset_n) begin
    if (!Reset_n) 
        Led <= 0;
    else begin
        case (counter1)
            0: Led <= 1'd1;
            1: Led <= 1'd0;
            3: Led <= 1'd1;
            4,5: Led <= 1'd1; // 连续状态合并
            7,8, 9: Led <= 1'd0;
            default: Led <= Led;
        endcase
    end 
end 
endmodule
```

**知识点解析：分级驱动与查找表 (LUT) 思想**

- **时钟使能逻辑**：`counter1` 的自增条件是 `counter0 == MCNT`。这是一种经典的“分频驱动”思想，确保 `counter1` 每一跳的间隔都是精确的 0.25s。
- **状态机简化**：通过 `case(counter1)`，我们将原本复杂的时序逻辑转化为了一个“查找表”。你只需要关心在第几步做什么，而不需要关心具体的毫秒数。


# 4. 灵活运用：外部输入驱动序列 (led_ctrl2)

线性序列机不一定是死代码。我们可以让序列的“内容”由外部输入（如拨码开关）决定。

## 4.1 设计思路

- 以 0.25s 为基本变换间隔。
- 以 8 个小段为一个大周期。
- LED 在每一段的状态由 8 位输入 `sw` 的每一位对应。

## 4.2 RTL 代码实现

```verilog
// rtl/led_ctrl2.v
module led_ctrl2(    
    input Clk,
    input Reset_n,
    input [7:0] sw,
    output reg Led
);
parameter MCNT = 12_500_000 - 1;

// 0.25s 计数(代码同上)

reg [2:0] counter1;  // 8状态循环
always@(posedge Clk or negedge Reset_n) begin
	if (!Reset_n) 
        counter1 <= 0;
	else if (counter0 == MCNT) 
        counter1 <= counter1 + 1'd1; // 3 位寄存器，溢出自动清零
end

always@(posedge Clk or negedge Reset_n) begin
	if (!Reset_n) 
        Led <= 0;
    else begin
        case (counter1)
            3'd0: Led <= sw[0];
            3'd1: Led <= sw[1];
            3'd2: Led <= sw[2];
            3'd3: Led <= sw[3];
            3'd4: Led <= sw[4];
            3'd5: Led <= sw[5];
            3'd6: Led <= sw[6];
            3'd7: Led <= sw[7];
            default: Led <= Led;
        endcase
    end 
end 
endmodule
```

**知识点解析：动态序列映射**

- **数据与逻辑分离**：逻辑部分（计数器）只负责提供“当前的进度”，而数据部分（`sw`）决定了在该进度下的具体表现。这种解耦方式非常适合编写可配置的驱动程序。
- **位选技巧**：在 `case` 中使用 `sw[counter1]` 也是合法的缩写形式，它能让代码更加简洁（本质上是一个多路选择器 MUX）。

# 5. 复杂控制：带使能与空闲态的序列 (led_ctrl3)

在实际应用中，序列往往不是一直循环的。我们需要“开始”和“停止”逻辑。

## 5.1 设计目标

每完成一轮 8 段序列后，强制进入 1s 的空闲（Idle）复位状态，然后再开始下一轮。

## 5.2 RTL 代码实现

```verilog
// rtl/led_ctrl3.v
module led_ctrl3(
input Clk,
input Reset_n,
input [7:0] sw,
output reg Led
);
parameter MCNT0 = 12_500_000 - 1; // 0.25s
parameter MCNT2 = 50_000_000 - 1; // 1s 空闲
reg en_counter0, en_counter2;

// 基础计时：仅在 en_counter0 为高时工作
reg [24:0] counter0;
always@(posedge Clk or negedge Reset_n) begin
    if (!Reset_n) 
        counter0 <= 0;
    else if (en_counter0) begin
        if(counter0 == MCNT0) 
            counter0 <= 0;
        else 
            counter0 <= counter0 + 1'd1;
    end else 
        counter0 <= 0;
end

// 8 位计数器（代码同上）

// 空闲计时：仅在 en_counter2 为高时工作
reg [26:0] counter2;
always@(posedge Clk or negedge Reset_n) begin
    if (!Reset_n) 
        counter2 <= 0;
    else if (en_counter2) begin
        if(counter2 == MCNT2) 
            counter2 <= 0;
        else 
            counter2 <= counter2 + 1'd1;
    end else 
        counter2 <= 0;
end

// 核心：使能信号的切换逻辑
always @(posedge Clk or negedge Reset_n) begin
    if(!Reset_n) begin
        en_counter2 <= 1'd1; // 初始进入空闲态
        en_counter0 <= 1'd0;
    end else if ((counter1 == 7) && (counter0 == MCNT0)) begin
        en_counter2 <= 1'd1; // 序列结束，开启空闲
        en_counter0 <= 1'd0;
    end else if (counter2 == MCNT2) begin
        en_counter2 <= 1'd0; // 空闲结束，开启序列
        en_counter0 <= 1'd1;
    end
end

// LED 输出
always@(posedge Clk or negedge Reset_n) begin
    if (!Reset_n)
        Led <= 0;
    else if (en_counter2 == 1)
        Led <= 0;
    else begin
        case (counter1)
            3'd0: Led <= sw[0];
            3'd1: Led <= sw[1];
            3'd2: Led <= sw[2];
            3'd3: Led <= sw[3];
            3'd4: Led <= sw[4];
            3'd5: Led <= sw[5];
            3'd6: Led <= sw[6];
            3'd7: Led <= sw[7];
            default: Led <= Led;
        endcase
    end 
end 
endmodule
```

**知识点解析：有限状态机 (FSM) 与 LSM 的结合**

- **运行控制**：通过 `en_counter` 信号，我们给线性序列机加上了“离合器”。这在处理非连续触发的场景（如按键触发一次发送一个 SPI 包）时非常关键。
- **状态互斥**：代码中通过逻辑控制确保了 `en_counter0` 和 `en_counter2` 在理想状态下是互斥的。这种“工作-休眠”轮转模式是低功耗设计和复杂协议控制的基础。

# 6. 仿真代码

以 `led_ctrl3` 的仿真代码为例子： 

```verilog
`timescale 1ns/1ns

module led_ctrl3_tb();
    reg Clk;
    reg Reset_n; 
    reg [7:0] sw;
    wire Led;
    
    // 实例化时参数名要匹配
    led_ctrl3 #(
        .MCNT0 (12_500 - 1),    // 仿真加速：250us
        .MCNT2 (50_000 - 1)
    ) 
    led_ctrl3_inst(
        .Clk(Clk),
        .Reset_n(Reset_n),
        .sw(sw),
        .Led(Led)
    );
    
    initial Clk = 1;
    always #10 Clk = ~Clk;  // 50MHz
    
    initial begin
        $dumpfile("led_ctrl3.vcd");
        $dumpvars(0, led_ctrl3_tb);
        
        Reset_n = 0; 
        sw = 8'b1010_1010;
        #201; 
        Reset_n = 1;
        
        #4_000_000;  // 等待4ms，观察软复位效果
        sw = 8'b1111_0000;
        
        #4_000_000;  // 再等待4ms
        sw = 8'b1001_0110;
        
        #4_000_000;
        $stop;
    end
endmodule
```

# 7. 总结

1.  **定义时间单位**：根据需求，用 `counter0` 产生基础的 Tick。
2.  **构建步骤轴**：用 `counter1` 记录当前处于序列的哪一步。
3.  **描述动作逻辑**：用 `case(counter1)` 描述每一步要干什么。

这种思想在编写 SPI 驱动时极其有用：`counter0` 产生时钟频率，`counter1` 记录发送到了第几个 bit，然后在 `case` 里根据 `counter1` 改变数据线和时钟线的电平。

