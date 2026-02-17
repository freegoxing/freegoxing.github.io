---
title: 通过 Bottles 在 Linux 上安装和运行 Multisim
date: 2026-02-16 19:23:43
tags: [linux, bottles, multisim, wine, flatpak]
categories:
---

# 前言

Multisim 是由美国国家仪器（NI）公司推出的一款功能强大的电路仿真和设计软件。它在学术界和工业界被广泛用于电路教学、电子线路设计、仿真与分析。然而，作为一款专业的 Windows 软件，官方并未提供 Linux 版本。

对于 `Linux` 用户而言，尝试使用 `wine` 来运行 `Multisim` 常常会遇到许多挑战。这不仅因为 `wine` 的兼容性问题，还因为 `Multisim` 自身的复杂性，其安装程序可能会触发 `Windows` 的补丁更新机制，导致在 `wine` 环境下安装失败。手动解包 `nipkg` 和校验文件等操作也相当繁琐。

本文将介绍一种更可靠的方法：通过 `Bottles` 将一个在 `Windows` 下已经安装好的 `Multisim` “移植”过来，从而绕过复杂的安装过程，实现在 `Linux` 上的稳定运行。

# 前置条件

## 1. 安装 Flatpak

`Bottles` 通过 `Flatpak` 进行分发，这使得它可以在几乎所有的 `Linux` 发行版上运行。在开始之前，请确保您的系统已经安装了 `Flatpak`。

大部分主流的 `Linux` 发行版都已经预装了 `Flatpak`。如果您的系统尚未安装，可以访问 [Flatpak 官方安装指南](https://flatpak.org/setup/)，选择您的发行版并按照说明进行安装。

例如，在 `Ubuntu` 或基于 `Debian` 的系统上，您可以使用以下命令安装：

```bash
sudo apt install flatpak
```

在 `Fedora` 上：

```bash
sudo dnf install flatpak
```

在 `Arch Linux` 上：

```bash
sudo pacman -S flatpak
```

安装完成后，建议添加 `Flathub` 仓库，这是 `Flatpak` 应用最主要的来源：

```bash
flatpak remote-add --if-not-exists flathub https://flathub.org/repo/flathub.flatpakrepo
```

## 2. 安装 Bottles 和 Flatseal

接下来，我们通过 `flatpak` 安装 `Bottles` 和 `Flatseal`。`Bottles` 用来创建和管理独立的 `wine` 环境，而 `Flatseal` 则可以方便地管理 `flatpak` 应用的权限。

```bash
flatpak install flathub com.usebottles.bottles
flatpak install flathub com.github.tchx84.Flatseal
```

#  Flatseal 权限设置

由于 `Flatpak` 应用运行在沙箱环境中，默认情况下，`Bottles` 无法访问您主目录下的文件。为了将我们从 `Windows` 复制的文件放入 `bottle` 中，需要使用 `Flatseal` 为其授权。

1.  打开 `Flatseal` 应用。
2.  在左侧列表中找到并选择 `Bottles`。
3.  在右侧的权限设置中，找到 “文件系统 (Filesystem)” 部分。
4.  把这些选项全部勾选，避免 wine 环境出现文件读取问题

![在 Flatseal 中为 Bottles 授予文件系统访问权限](https://img.556756.xyz/PicGo/blogs/2026/02/20260216200701659.png)

{% note info %}

选项出现 ⚠️  的表明是用户修改过的选项

{% endnote %}

#  Bottles 基本使用

`Bottle` 是一个独立的、包含了 `wine` 环境的“容器”。为不同的应用创建不同的 `bottle` 是一个好习惯，可以避免环境配置的相互干扰。

安装 `Multisim` 前，我们需要先创建一个新的 `bottle`：
1.  打开 `Bottles` 应用。
2.  点击主界面右上角的 “+” 号或 “创建新 Bottle” 按钮。
3.  为您的 `bottle` 命名，例如 `Multisim`。
4.  选择 “应用 (Application)” 作为环境类型。
5.  点击 “创建” 按钮，`Bottles` 会自动为您配置好一个新的 `wine` 环境。

![在 Bottles 中为 Multisim 创建一个新的应用环境](https://img.556756.xyz/PicGo/blogs/2026/02/20260216201951670.png)

{% note info %}

第一次打开Bottles 会加载许久，安装对应的依赖和环境

{% endnote %}

# 安装 Multisim

此方法的核心思想是，我们不再尝试在 `Linux` 环境中从零开始安装 `Multisim`，而是将一个在 `Windows` 环境下已经安装好的 `Multisim` “移植”过来，从而绕过 `wine` 环境下因安装程序兼容性问题导致的失败。

## 1. 准备 Windows 端文件

首先，您需要在一个正常的 `Windows` 系统（或 `KVM`、`VMware` 等虚拟机）中安装好 `Multisim`。然后，将以下文件和注册表项准备好，这是我们进行“移植”的原材料。

### a. 复制文件
将以下目录的文件完整地复制出来：

*   `C:\ProgramData\National Instruments`：*注意：`ProgramData` 是隐藏文件夹。压缩或复制此文件夹时若出现部分文件因权限无法访问的提示，可以直接跳过，后续步骤会通过修复工具来解决。*
*   `C:\Program Files (x86)\National Instruments`
*   `C:\Program Files (x86)\Common Files\Microsoft Shared\DAO\dao360.dll` :  `JET 引擎` 关键组件
*   `C:\Program Files\National Instruments`

### b. 导出注册表
在 Windows 的“运行”中输入 `regedit` 打开注册表编辑器，找到以下两个位置，分别右键点击并选择“导出”，保存为 `.reg` 文件。

*   `计算机\HKEY_LOCAL_MACHINE\SOFTWARE\National Instruments`
*   `计算机\HKEY_LOCAL_MACHINE\SOFTWARE\WOW6432Node\National Instruments`

![从 Windows 系统中准备好的 Multisim 文件和注册表](https://img.556756.xyz/PicGo/blogs/2026/02/20260216211624790.png)

## 2. 配置 Bottle 环境

回到 `Bottles`，为 `Multisim` 创建一个纯净的运行环境。

### a. 删除默认的 .NET Framework

`Multisim` 对 `.NET Framework` 版本很敏感，`wine` 环境默认自带的 `mono` 框架可能会导致兼容性问题，需要先将其卸载，后续替换为“原生”的框架。

1.  进入创建好的 `Bottle`，点击 `依赖` 标签页。
2.  搜索 `mono`，点击其右侧的三个点，选择 `卸载`。

![卸载与 Multisim 不兼容的默认 mono 依赖](https://img.556756.xyz/PicGo/blogs/2026/02/20260217105848335.png)

### b. 安装所需依赖

`Multisim` 的正常运行依赖 `.NET Framework`、`Visual C++` 运行时和 `JET` 数据库引擎。

1.  **.NET Framework 4.6.2**: 在依赖页面搜索 `dotnet462` 并点击安装。
    ![搜索并安装 .NET Framework 4.6.2 (dotnet462)](https://img.556756.xyz/PicGo/blogs/2026/02/20260216202934971.png)
2.  **Microsoft Visual C++**: 在依赖页面搜索 `vcredist2022` 并点击安装。
    ![安装 Microsoft Visual C++ 2022 运行时](https://img.556756.xyz/PicGo/blogs/2026/02/20260217123857682.png)
3.  **JET 数据库引擎**: 在依赖页面搜索 `jet40` 并点击安装。缺少此引擎将导致无法从库中拖出元器件。
    ![安装 JET 4.0 数据库引擎以支持访问数据库](https://img.556756.xyz/PicGo/blogs/2026/02/20260216203755628.png)

在安装过程中，如果出现弹窗，保持默认选项并点击 `继续` 或 `Install` 即可。

![依赖安装过程中自动弹出的确认窗口](https://img.556756.xyz/PicGo/blogs/2026/02/20260216203212587.png)

## 3. 移植文件与注册表

现在，将我们在步骤 1 中准备好的文件和注册表信息放入 `Bottle` 中。

### a. 导入文件
1.  在 `Bottle` 的主界面，点击 `浏览C盘` 按钮可以快速打开该 `Bottle` 对应的 `drive_c` 文件夹。
2.  将步骤 1 中从 `Windows` 复制出来的所有文件夹，按照原有的目录结构，放入 `drive_c` 中（如果已存在同名文件夹，直接合并）。

{% note info %}
`Bottle` 的 `C盘` 路径通常位于：
```bash
# 请将 <YOUR_USER_NAME> 和 <YOUR_Bottle_Name> 替换为你的实际信息
/home/<YOUR_USER_NAME>/.var/app/com.usebottles.bottles/data/bottles/bottles/<YOUR_Bottle_Name>/drive_c
```
![通过 Bottles 界面快速浏览 C:/ 盘](https://img.556756.xyz/PicGo/blogs/2026/02/20260216204314554.png)
{% endnote %}

### b. 导入注册表
1.  回到 `Bottle` 界面，进入 `工具` 标签页，点击 `注册表编辑器`。
2.  在注册表编辑器窗口中，点击左上角的 `注册表` -> `导入注册表文件...`。
3.  依次选择并导入你在步骤 1 中导出的两个 `.reg` 文件。

![使用 Bottles 内置的注册表编辑器](https://img.556756.xyz/PicGo/blogs/2026/02/20260216212459397.png)

## 4. 修复安装并运行 Multisim

文件和注册表就位后，我们还需要利用 `NI Package Manager` 修复一下文件路径和配置，以适应新的 `wine` 环境。

### a. 运行 NI Package Manager 并修复
1.  回到 `Bottle` 的 `程序` 列表，点击 `添加快捷方式...`，找到并选择 `NI Package Manager` 的可执行文件。其路径通常是：
    ```
    /drive_c/Program Files/National Instruments/NI Package Manager/NIPackageManager.exe
    ```
2.  运行 `NI Package Manager`。
    ![启动 NI Package Manager 准备修复安装](https://img.556756.xyz/PicGo/blogs/2026/02/20260217114904023.png)
3.  在打开的窗口中，切换到 `已安装` 标签页。
4.  勾选所有列出的 National Instruments 相关软件，然后点击 `修复` 按钮。
    ![在 NI Package Manager 中选择所有组件进行修复](https://img.556756.xyz/PicGo/blogs/2026/02/20260217115034242.png)
5.  等待修复过程完成。
    ![NI Package Manager 成功完成修复操作](https://img.556756.xyz/PicGo/blogs/2026/02/20260217115849635.png)

### b. 运行 Multisim
修复完成后，就可以运行 `Multisim` 主程序了。

1.  同样在 `程序` 列表中 `添加快捷方式...`，这次选择 `Multisim` 的主程序：
    ```
    /drive_c/Program Files (x86)/National Instruments/Circuit Design Suite 14.3/multisim.exe
    ```
2.  在运行之前，建议进行一项优化：点击 `multisim.exe` 快捷方式旁边的三个点，进入 `首选项`，关闭 `DXVK` 和 `VKD3D`。这有助于避免某些图形渲染问题。
3.  现在，点击三角形运行按钮！
    ![在 Bottles 程序列表中管理和运行 multisim.exe](https://img.556756.xyz/PicGo/blogs/2026/02/20260216213158809.png)
4.  如果一切顺利，你将看到 `Multisim` 的启动界面。无论是选择试用还是激活，软件都应该可以正常打开并进行仿真了。
    ![成功启动 Multisim 后的激活窗口](https://img.556756.xyz/PicGo/blogs/2026/02/20260217124208289.png)
    ![Multisim 正常运行并可进行电路仿真](https://img.556756.xyz/PicGo/blogs/2026/02/20260217124524471.png)

### c. 创建桌面快捷方式 (可选)

为了方便日常启动，可以让 `Multisim` 像原生应用一样显示在你的系统程序菜单中。

1.  **自动添加**: 在 `Bottle` 的程序列表中，找到 `multisim.exe` 这一行，点击其右侧的“添加桌面条目”图标（一个指向外部的方框箭头）。在多数桌面环境下，这会自动将快捷方式添加到你的应用程序菜单中，之后便可直接搜索并启动 `Multisim`。

    ![添加桌面条目](https://img.556756.xyz/PicGo/blogs/2026/02/20260217131337830.png)

2.  **手动创建**: 如果点击图标后程序没有自动出现在菜单中，你可以进行手动操作。`Bottles` 会在以下路径生成一个 `.desktop` 启动文件：
    
    ```bash
    ~/.var/app/com.usebottles.bottles/data/applications
    ```
    你可以将这个 `*.desktop` 文件复制或链接到以下通用路径，以确保它被你的系统识别：
    * **为当前用户的所有程序菜单添加**: `~/.local/share/applications`

    * **仅为桌面创建快捷方式**: `~/Desktop`
    
# 总结
通过 `Bottles` 的“移植”方法，我们成功绕过了 `Multisim` 在 `wine` 环境下最困难的安装环节，大大提高了在 `Linux` 上成功运行它的几率。虽然前期需要在 Windows 环境中准备文件，但这是一劳永逸的。希望这篇教程能帮助到同样希望在 Linux 上进行电路设计的你。

