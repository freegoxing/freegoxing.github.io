---
title: FPGA 学习笔记（一）：Linux 下安路开发环境从安装到ModelSim仿真优化
date: 2026-02-24 10:00:00
categories: [FPGA]
tags: [Anlogic, ModelSim, FPGA]
---

<!-- more -->

#  前言 

在 Linux 系统下进行 FPGA 开发，环境搭建往往是第一道坎。安路（Anlogic）的 Tang Dynasty (TD) 软件与 ModelSim 的配合虽然强大，但在 Linux 下的安装配置以及默认的仿真流转效率仍有优化空间。本文记录了这两个软件在 Linux 下的安装基本流程，并重点介绍如何通过优化 `.do` 脚本解决仿真启动缓慢的问题。

# 1. Linux 环境下的软件安装

## 1.1 Tang Dynasty (TD) 安装
安路 TD 软件通常提供 Linux 版本的压缩包（如 `.tar.gz`）。以 `TD_Release_2026.1_NL_Linux.tar.gz` 为例：

1. **解压与解包**：
   ```bash
   sudo mkdir -p /opt/Anlogic
   sudo tar -xzvf TD_Release_2026.1_NL_Linux.tar.gz -C /opt/Anlogic
   ```

2. **License 配置**：
   将官网下载的 `Anlogic.lic` 放置在软件安装目录下的 `license` 文件夹中：
   ```txt
   /opt/Anlogic/TD_Release_2026.1_NL/license/Anlogic.lic
   ```

3. **软件启动**：
   TD 提供了二进制程序 `td` 和封装脚本 `td.sh`。

   - **推荐方式（使用 wrapper 脚本）**：
     `td.sh` 会自动处理 `LD_LIBRARY_PATH` 等环境变量，是最简单且稳定的启动方式：
     ```bash
     /opt/Anlogic/TD_Release_2026.1_NL/bin/td.sh -gui  # 启动图形界面
     /opt/Anlogic/TD_Release_2026.1_NL/bin/td.sh       # 命令行模式
     ```

   - **手动配置启动（仅限高级调试）**：
     若需直接调用二进制文件 `td`，需在当前终端手动设置环境（以 Bash 为例）：
     
     ```bash
     export TD_HOME=/opt/Anlogic/TD_Release_2026.1_NL
     export LD_LIBRARY_PATH=$TD_HOME/lib:$TD_HOME/lib/Qt/lib:$LD_LIBRARY_PATH
     export QT_PLUGIN_PATH=$TD_HOME/lib/Qt/plugins
     
     $TD_HOME/bin/td -gui
     ```
     
   - **便捷访问**：
     建议建立一个软链接到 `/usr/local/bin`，这样可以直接在终端输入 `td` 启动：
     ```bash
     sudo ln -s /opt/Anlogic/TD_Release_2026.1_NL/bin/td.sh /usr/local/bin/td
     ```

## 1.2 ModelSim 安装

ModelSim (或 Questa) 的安装在 Linux 下相对复杂

- **资源获取**：[网盘下载](https://pan.baidu.com/s/1oNHvzMFh9pLcGAX1A3Dqjw?pwd=2301) (提取码: 2301)

### 1. 运行安装程序

建议在 64 位系统（如 Ubuntu 20.04+）上进行如下操作：

1. **准备安装包**：确保安装程序具有执行权限。

   ```bash
   chmod +x modelsim-se-2020.4.aol
   sudo ./modelsim-se-2020.4.aol
   ```
2. **图形界面安装**：在弹出的窗口中选择安装路径（建议 `/opt/mentor/modelsim`）。
3. **组件选择**：

    1. `ModelSim SE 64-bit`
    2. `ModelSim SE Documentation`
    3. `GCC-64-bit`
    
    由于环境是64位，所以组件选择64位的组件

### 2. 配置环境变量

在 `~/.bashrc` 或 `~/.zshrc` 末尾添加以下内容。

将以下内容添加到配置文件末尾（请根据你的实际安装路径修改）：

```bash
# ModelSim 2020.4 Settings
export MTI_VCO_MODE=64 # 强制开启 64 位模式
export MTI_HOME=/opt/mentor/modelsim2020.4/modeltech
export PATH=$MTI_HOME/bin:$PATH

# License 路径 (破解后生成的 license.lic 路径)
export LM_LICENSE_FILE=/opt/mentor/modelsim2020.4/license.lic
```

保存后，在终端执行以下命令使配置生效：
```bash
source ~/.zshrc  # 如果使用 bash，请执行 source ~/.bashrc
```

{% note info %}

虽然现在还没有 `license.lic` 文件，会在后面破解时候生成

{% endnote %}

### 3. 破解 License 验证程序 

**详细参考**：[CSDN - Ubuntu 安装 ModelSim SE 教程](https://blog.csdn.net/weixin_43245577/article/details/140839616)

# 2. 项目目录规范

在我的学习过程中，我采用了如下结构，确保代码、仿真和日志互不干扰：

```txt
Project
├── log                                 # 日志文件夹
├── demo.al                             # TD 项目配置文件
├── demo_Runs                           # 项目运行目录
├── demo_tb_sim                         # 项目仿真目录
│  └── behav_sim                        
│      └── demo_behavioral_sim.do       # TD 生成的 .do 脚本
├── rtl                                 # 项目代码
│  └── demo.v
└── sim                                 # 项目 TestBench 代码
   └── demo_tb.v
```

# 3. 核心优化：预编译库链接

安路 EG4 系列的库文件如果每次都重编，会浪费大量时间。我们可以为 ModelSim 手动建立一个预编译库。

## 3.1 预编译 `eg4_lib` 库

### 1. 文件准备
1. 在 ModelSim 的安装目录下（或任意你喜欢的地方），建立文件夹 `anlogic/src`。
2. 将 TD 安装路径下 `sim_release` 目录内的所有文件复制到上述 `src` 文件夹中。

### 2. 执行编译
在 `anlogic` 目录下打开终端，依次执行以下命令：

```bash
# 1. 创建库目录
vlib eg4

# 2. 映射库名
vmap eg4 eg4

# 3. 编译源文件到 eg4 库
# 注意：路径请根据实际复制后的路径调整
vlog -work eg4 ./src/common/*.v
vlog -work eg4 ./src/common/apm/*.v
vlog -work eg4 ./src/eg4/*.v
```

### 3. 写入全局配置文件 (可选但推荐)
为了让 ModelSim 每次启动都能自动识别 `eg4` 库，建议修改 ModelSim 安装目录下的 `modelsim.ini`：

1. 找到 `modelsim.ini`（通常在 `$MTI_HOME/modelsim.ini`），取消其只读属性。
2. 在 `[Library]` 部分添加：
   ```ini
   eg4 = /opt/mentor/modelsim2020.4/modeltech/anlogic/eg4
   ```
   这样以后在任何项目的 `.do` 脚本中，直接调用 `-L eg4` 即可，无需再手动 `vmap`。

## 3.2 优化后的 `.do` 脚本示例

现在，你的项目仿真脚本可以精简为如下结构（建议命名为 `sim.do`）：

```tcl
# 建立当前项目的工作库
if {![file exists work]}{
	vlib work
}

# (可选) 如果没改 .ini，则需要手动指定映射
# vmap eg4 /opt/mentor/modelsim2020.4/modeltech/anlogic/eg4

# 仅编译当前项目的 RTL 和 TB 
vlog -work work "../../rtl/project.v"
vlog -work work "../../sim/project_tb.v"

# 启动仿真：使用 -L 加载预编译库
vsim -t ps -voptargs="+acc" -L eg4 work.your_tb_name

# 加载波形与运行
view wave signals
add wave *
run 1us
```

### 运行方式
在仿真目录下打开终端，直接执行以下命令即可实现“一键仿真”：

```bash
vsim -do ./sim.do 
```

# 4. 总结
- **自动化思维**：通过修改 `.do` 脚本实现“一键启动”，极大减少了手动点击 UI 的重复劳动。
- **资源解耦**：将厂商库与项目代码分离，利用预编译库（.ini 映射）将原本几分钟的编译过程缩短至秒级。
- **环境规范**：清晰的目录结构（rtl, sim, log）是长期维护大型项目的基础。

