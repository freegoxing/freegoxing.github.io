---
title: waydroid模拟器使用
date: 2026-05-09 10:42:38
tags: [linux, waydroid, 模拟器, Android]
categories: [linux]
---

# 前言

一般而言要在电脑上运行 Android 引用，很多人的选择是虚拟机或模拟器。可是 Android 系统本身就是基于 Linux 系统开发的，所以我们应该是可以在 Linux 上运行一个完整的 Android 系统。我们借助 waydroid 可以允许我们在 Wayland 桌面环境上，通过容器化技术，完整的启动一个 Android 系统

# 环境准备

## 桌面环境是 Wayland

waydroid 依赖于 Wayland 桌面环境，使用 X11 的无法正常使用

```shell
echo $XDG_SESSION_TYPE
```

确保输出的内容是 `Wayland`

## 安装 waydroid

这里以 Ubuntu 为例，安装 waydroid 本体和相关依赖

```shell
sudo apt install waydroid
```

## 启动服务

在安装完成后可以启动 waydroid 容器服务

```shell
sudo waydroid container start
```

## 安装 Android 镜像

waydroid 会根据命令拉取是否带有 GApps 的系统镜像

```shell
waydroid init           # 不带 GApps 
waydroid init -s GAPPS  # 带 GApps
```

# waydroid 基础优化

waydroid 安装的 Android 系统是和宿主机的 CPU 架构是一样的。对于常规的 Android 应用，他们是运行再 arm 架构上所以我们要安装翻译层用来在 x86_64 上支持 arm

我们使用的是 [waydroid_script](https://github.com/casualsnek/waydroid_script) 可以根据 github 上的指南进行操作

```shell
git clone https://github.com/casualsnek/waydroid_script
cd waydroid_script
python3 -m venv venv
venv/bin/pip install -r requirements.txt
```

## 系统依赖

waydroid_scirpt 脚本依赖系统的 `lzip`。没有该内容的可以通过系统包管理器进行安装

```shell
sudo apt install lzip
```

## 脚本运行

然后进入终端交互式界面

```shell
sudo venv/bin/python3 main.py
```

### 1. 选择安卓版本

新版本的 waydroid 的 Android 镜像一般是 Android 13（视自己的情况而定）

![选择安卓版本](https://img.556756.xyz/PicGo/blogs/2026/05/20260510105538568.png)

### 2.选择运行模式

我们选择里面的 `Install` 来安装我们需要的内容

![选择运行模式](https://img.556756.xyz/PicGo/blogs/2026/05/20260510105817379.png)

### 3. 选择要安装的组件

这里提供一些脚本作者的推荐的组件

![选择要安装的组件](https://img.556756.xyz/PicGo/blogs/2026/05/20260510110407415.png)

-   gapps: Google 全家桶与 Google 服务框架，提供 GMS 服务但是会提高占用，视需求安装
-   microg: 是 Google Play Services 的开源替代实现，更加轻量，但是可能存在兼容性问题
-   libndk: 提供 Android Native Development Kit 相关运行库。
-   libhoudini：这是 Intel 的 ARM 转译层。
-   magisk：Android Root 管理工具。
-   smartdock：用于桌面模式增强，修改 dock 栏等
-   fdroidpriv：这是 F-Droid 的特权扩展，配合 F-Droid 应用商店
-   widevine：Google 的 DRM 解密组件，播放受版权保护的视频

一般而言推荐安装 libndk，magisk，smartdock 三个组件。分别提供 arm 应用运行，root 管理和桌面的基础美化

{% note warning %} 

一般而言，在 AMD 平台上，libndk 的性能似乎比 libhoudini 更好。而 libhoudini 适用于英特尔/AMD x86 CPU。但是不是绝对的，所以遇到兼容性问题可以两者均尝试

{% endnote %}

# waydroid 使用

确保 `waydroid-container.service` 已经启动，然后 就可以执行

```shell
waydroid session start
```

来启动 waydroid 会话，下面给出一些 waydroid 交互命令

-   启动 GUI

```shell
waydroid show-full-ui
```

![waydroid GUI 启动界面](https://img.556756.xyz/PicGo/blogs/2026/05/20260510112027397.png)

-   启动 shell

```shell
waydroid shell
```



-   安装程序

我们通过 adb 调试的方式进行安装。

首先我们要安装 adb

```shell
sudo apt install adb
```

然后 adb 连接 waydroid

我们首先要获取 waydroid 网络 ip 可以通过 

```shel
waydroid status
```

![waydroid status](https://img.556756.xyz/PicGo/blogs/2026/05/20260510112746563.png)

发现 ip 为 `192.168.240.112` 然后 adb 连接其 `5555` 端口就可以

```shell
adb connect 192.168.240.112:5555
```

这里必须已经启动了 GUI 界面，因为连接时候会有弹窗要求我们进行确定。然后就可以安装应用了

```shell
adb install <package path>
```

