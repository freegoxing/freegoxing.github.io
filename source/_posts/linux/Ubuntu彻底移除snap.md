---
title: Ubuntu 彻底移除 snap 服务
date: 2026-07-20 20:23:43
tags: [linux, ubuntu, snap, flatpak]
categories: [linux]
---

# 前言

Ubuntu 从较新的版本开始，默认安装了不少 `snap` 应用和相关运行环境，例如 `firefox`、`snap-store`、`gnome` 运行时等。`snap` 的优点是应用分发统一、依赖隔离简单，但也有一些使用上的问题：启动速度偏慢、磁盘占用较高、目录结构不够直观，以及部分应用和系统原生主题的集成体验一般。

如果你更习惯使用 `apt`、`deb` 或 `flatpak` 管理软件，可以把系统里的 `snap` 彻底移除，并通过 `APT Pinning` 防止后续安装软件时又被自动拉回来。

{% note warning %}

删除 `snap` 前请先确认里面没有你正在使用的重要应用。本文会直接移除 `snap` 应用、运行时和 `snapd` 服务，执行前建议先看一遍 `snap list` 的输出。

{% endnote %}

# 查看当前 snap 包

首先查看当前系统已经安装的 `snap` 包：

```bash
snap list
```

![snap list](https://img.556756.xyz/PicGo/blogs/2026/07/20260720202328281.png)

`snap` 包之间存在依赖关系，不能随便从底层组件开始删。一般按照下面的顺序处理：先删除应用，再删除桌面集成和运行时，最后删除基础包和 `snapd` 本体。

# 移除 snap 应用

先删除普通应用。这一部分要根据你自己 `snap list` 的输出调整，如果你的系统里没有某个包，命令报找不到可以直接跳过。

```bash
sudo snap remove firefox
sudo snap remove snap-store
sudo snap remove firmware-updater
sudo snap remove desktop-security-center
sudo snap remove prompting-client
```

然后删除桌面集成和主题相关包：

```bash
sudo snap remove snapd-desktop-integration
sudo snap remove gtk-common-themes
```

继续删除图形运行环境：

```bash
sudo snap remove gnome-46-2404
sudo snap remove mesa-2404
```

最后删除基础包：

```bash
sudo snap remove core24
sudo snap remove bare
```

这些包名会随着 Ubuntu 版本变化。例如有的系统可能是 `core22`、`gnome-42-2204` 或其他运行时名称，所以不要完全照抄，需要以 `snap list` 的实际输出为准。

# 移除 snapd 服务

普通 `snap` 包都清理完成后，再删除 `snapd` 本体：

```bash
sudo snap remove snapd
sudo apt purge snapd
```

如果这里提示 `snapd is not removable`，通常是还有其他 `snap` 包依赖它。可以再次检查：

```bash
snap list
```

如果确认应用已经删完，但服务仍然占用，可以先停止 `snapd` 相关服务：

```bash
sudo systemctl stop snapd.service
sudo systemctl stop snapd.socket
sudo systemctl disable snapd.service
sudo systemctl disable snapd.socket
```

然后再次执行：

```bash
sudo apt purge snapd
```

# 清理残留文件

`snapd` 卸载后，系统里通常还会残留一些目录，可以手动删除：

```bash
sudo rm -rf /snap
sudo rm -rf /var/snap
sudo rm -rf /var/lib/snapd
sudo rm -rf /var/cache/snapd
rm -rf ~/snap
```

再执行一次自动清理：

```bash
sudo apt autoremove --purge
sudo apt clean
```

可以通过下面的命令确认系统里是否还存在 `snapd`：

```bash
dpkg -l | grep snapd
```

如果没有任何输出，说明 `snapd` 已经从 `APT` 的已安装包里移除了。

# 防止 snapd 被重新安装

Ubuntu 有些包可能会把 `snapd` 作为依赖重新拉回来，所以需要给 `snapd` 设置一个较低优先级。

创建配置文件：

```bash
sudoedit /etc/apt/preferences.d/no-snap.pref
```

写入下面的内容：

```text /etc/apt/preferences.d/no-snap.pref
Package: snapd
Pin: release a=*
Pin-Priority: -10
```

保存后更新软件源：

```bash
sudo apt update
```

可以用下面的命令检查优先级是否生效：

```bash
apt policy snapd
```

如果看到 `Pin-Priority` 为负数，之后通过 `apt` 安装软件时就不会自动安装 `snapd`。

# 使用 Flatpak 替代

移除 `snap` 后，桌面应用可以优先考虑 `flatpak`。`flatpak` 同样提供应用隔离和跨发行版分发，常见桌面软件在 `Flathub` 上基本都能找到。

## 安装 Flatpak

大部分主流的 `Linux` 发行版都已经可以直接安装 `Flatpak`。如果你的系统尚未安装，可以访问 [Flatpak 官方安装指南](https://flatpak.org/setup/)，选择对应发行版查看详细步骤。

![Flatpak 安装](https://img.556756.xyz/PicGo/blogs/2026/07/20260720204856512.png)

例如，在 `Ubuntu` 或基于 `Debian` 的系统上：

```bash
sudo apt install flatpak
```

安装完成后，添加 `Flathub` 仓库：

```bash
flatpak remote-add --if-not-exists flathub https://flathub.org/repo/flathub.flatpakrepo
```

添加完成后建议注销并重新登录一次，这样桌面环境可以正常识别 `Flatpak` 应用的启动器。

然后就可以通过 `flatpak` 安装常见应用，例如：

```bash
flatpak install flathub org.mozilla.firefox
flatpak install flathub com.visualstudio.code
```

查看已安装的 `flatpak` 应用：

```bash
flatpak list
```

# 替代安装方式

## Firefox

如果你之前使用的是 Ubuntu 默认的 `snap` 版 `Firefox`，移除 `snap` 后可以选择 `Flatpak` 版本：

```bash
flatpak install flathub org.mozilla.firefox
```

启动：

```bash
flatpak run org.mozilla.firefox
```

`snap` 版和 `flatpak` 版 `Firefox` 的用户数据目录不同。如果需要保留浏览器数据，建议先登录 Firefox Sync 同步书签、密码和扩展；如果要手动迁移配置，可以先备份下面的目录：

```bash
cp -a ~/snap/firefox/common/.mozilla/firefox ~/firefox-snap-profile-backup
```

如果更喜欢 `deb` 包，也可以使用 Mozilla 官方或第三方维护的软件源。但这一类方案会随着发行版版本变化，建议安装前先查看对应源的说明。

## 应用中心

`Flatpak` 也可以配合图形应用中心使用，不需要每一次都通过终端安装软件。这里可以安装 `Bazaar`：

```bash
flatpak install flathub io.github.kolunmi.Bazaar
```

安装完成后就可以使用该应用商店进行安装

![bazaar 界面](https://img.556756.xyz/PicGo/blogs/2026/07/20260720205311828.png)

如果你使用的是 GNOME 桌面，也可以安装 `GNOME Software` 的 `Flatpak` 插件：

```bash
sudo apt install gnome-software gnome-software-plugin-flatpak
```

安装后同样建议注销并重新登录一次。

# 总结

整体流程并不复杂：先根据 `snap list` 删除应用和运行时，再卸载 `snapd`，最后清理残留目录并通过 `APT Pinning` 阻止它被重新安装。

如果只是想减少系统默认软件的占用，做到这里就可以了。后续安装桌面应用时，可以优先考虑 `apt` 和 `flatpak`，这样系统的软件来源会更清晰，也更符合传统 `Linux` 桌面环境的使用习惯。
