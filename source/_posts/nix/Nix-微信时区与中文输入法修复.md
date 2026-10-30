---
title: Nix 安装的微信显示 UTC 时间、无法使用中文输入法
date: 2026-10-30
tags: [linux, nix, home-manager, wechat, ibus, appimage]
categories: [Nix/NixOS]
description: 记录通过 Home Manager 安装微信后，分别修复 AppImage 时区数据和 IBus 中文输入的问题。
---

# 前言

我用 Home Manager 安装了 Nixpkgs 中的 `wechat`，遇到了两个互不相同的问题：微信里的时间按 UTC 显示，中文输入法也无法在微信输入框中使用。系统本身的时区和输入法正常，问题出在微信的运行环境与桌面会话之间的衔接。

下面是我目前使用的 `home/programs/wechat.nix`。它先调整微信的 AppImage FHS 环境，再给微信的启动命令增加 IBus 环境变量。这里的 `wechat` 指 Nixpkgs 中的 Linux 微信包；`wechat-uos` 等其他包不一定采用相同的封装方式。

# 问题一：微信显示 UTC 时间

先确认系统时间没有设错：

```bash
timedatectl
date '+%Y-%m-%d %H:%M:%S %Z %z'
readlink -f /etc/localtime
```

如果系统和其他应用都显示本地时间，只有微信显示 UTC，就要检查微信自身能否读取时区数据。Linux 程序通常通过 `/etc/localtime` 和时区数据库确定本地时间；`TZDIR` 可以指定时区数据库目录。主机上有这些文件，不意味着 AppImage 的运行环境里也能找到它们。[`tzset(3)` 手册](https://man7.org/linux/man-pages/man3/tzset.3.html)解释了时区文件和 `TZDIR` 的作用。

## 为什么可以在打包时补入时区数据

Nixpkgs 中的微信不是从源码编译的：它下载官方 AppImage，提取内容，再由 `appimageTools.wrapAppImage` 调用 `buildFHSEnv`，为程序拼出它预期的 `/usr` 等目录。启动时，`buildFHSEnv` 使用 `bwrap` 挂载这些目录和选定的主机文件。因此，这里调整的是**打包函数的参数**，让生成的运行环境多包含一个 Nix 包，并没有修改微信二进制文件。[微信包定义](https://github.com/NixOS/nixpkgs/blob/master/pkgs/by-name/we/wechat/package.nix)、[Linux 打包文件](https://github.com/NixOS/nixpkgs/blob/master/pkgs/by-name/we/wechat/linux.nix)和 [`wrapAppImage` 实现](https://github.com/NixOS/nixpkgs/blob/master/pkgs/build-support/appimage/default.nix)展示了这条调用链。

这里需要区分“挂载了文件”和“文件可用”。[Nixpkgs 的 FHS 启动器](https://github.com/NixOS/nixpkgs/blob/master/pkgs/build-support/build-fhsenv-bubblewrap/default.nix)会尝试引入 `/etc/localtime`，但它只引入选定的 `/etc` 项，清单中没有 `/etc/timezone`。如果主机的 `/etc/localtime` 是指向 `/usr/share/zoneinfo/...` 的符号链接，而 FHS 环境中的 `/usr` 没有对应的时区文件，微信仍可能读不到有效的本地时区。这是这类 UTC 现象的一种可能原因，具体还要结合本机链接目标和运行结果判断。

我的处理是在 `wrapAppImage` 的原有 `extraPkgs` 后追加 `tzdata`，并把 `TZDIR` 指向它的 `zoneinfo` 目录：

```nix
extraPkgs =
  p:
  (params.extraPkgs or (_: [ ])) p
  ++ [ p.tzdata.out ];

profile = (params.profile or "") + ''
  export TZDIR="${pkgs.tzdata.out}/share/zoneinfo"
'';
```

这里保留了原包已有的 `extraPkgs` 和 `profile`，只补入时区数据及其查找路径：`extraPkgs` 把 `tzdata` 放进 FHS 环境，`TZDIR` 为按时区名称查找数据的程序指定明确路径。`pkgs.wechat.override` 重新提供 `callPackage`，使微信的 Linux 打包文件收到修改后的 `appimageTools.wrapAppImage`。单纯设置 `TZ` 不会把缺少的时区数据库放进 FHS 环境。

如果你使用的微信包已经改变了打包方式，应先检查包定义，再决定是否继续使用这段覆盖。重新应用 Home Manager 配置并完全退出、重启微信后，比较微信消息时间与系统时间；这比只看 `date` 更能验证微信侧的问题是否消失。

# 问题二：微信无法输入中文

另一个现象是 IBus 在其他程序中可以输入中文，在微信里却不能。为此，我只给微信的启动命令设置输入法变量：

```nix
wechatWrapped = pkgs.symlinkJoin {
  name = "wechat-ibus";
  paths = [ wechatFixed ];
  nativeBuildInputs = [ pkgs.makeWrapper ];

  postBuild = ''
    wrapProgram $out/bin/wechat \
      --unset QT_IM_MODULES \
      --set QT_IM_MODULE ibus \
      --set GTK_IM_MODULE ibus \
      --set XMODIFIERS '@im=ibus' \
      --set IBUS_USE_PORTAL 1
  '';
};
```

`QT_IM_MODULE=ibus` 指定 Qt 输入法模块，`GTK_IM_MODULE` 和 `XMODIFIERS` 一起随微信进程传入，`IBUS_USE_PORTAL=1` 是我这套桌面环境所用的设置。`--unset QT_IM_MODULES` 用来清除可能与 `QT_IM_MODULE` 冲突的已有值。这些变量只影响由此包装器启动的微信，不会改动整个会话。Qt 文档也说明应用可以通过 [`QT_IM_MODULE`](https://doc.qt.io/qt-6.8/qtvirtualkeyboard-deployment-guide.html)选择输入法插件。

这段配置以 **IBus 已安装、中文输入源已经配置且在其他应用中可用** 为前提。若其他应用同样无法输入中文，应先排查 IBus 本身；若只有微信有问题，再检查是否从新生成的 `wechat` 启动，而不是仍在运行的旧进程或另一份微信安装。应用配置后，完全退出微信并重新启动，在聊天输入框里切换中文输入法验证。

# 完整的 Home Manager 配置

```nix
{ pkgs, ... }:

let
  # 为微信的 AppImage FHS 环境补充时区数据库
  wechatFixed = pkgs.wechat.override {
    callPackage =
      path: args:
      pkgs.callPackage path (
        args
        // {
          appimageTools = pkgs.appimageTools // {
            wrapAppImage =
              params:
              pkgs.appimageTools.wrapAppImage (
                params
                // {
                  extraPkgs =
                    p:
                    (params.extraPkgs or (_: [ ])) p
                    ++ [ p.tzdata.out ];

                  profile = (params.profile or "") + ''
                    export TZDIR="${pkgs.tzdata.out}/share/zoneinfo"
                  '';
                }
              );
          };
        }
      );
  };

  # 只包装微信的启动命令，传入 IBus 环境变量
  wechatWrapped = pkgs.symlinkJoin {
    name = "wechat-ibus";
    paths = [ wechatFixed ];
    nativeBuildInputs = [ pkgs.makeWrapper ];

    postBuild = ''
      wrapProgram $out/bin/wechat \
        --unset QT_IM_MODULES \
        --set QT_IM_MODULE ibus \
        --set GTK_IM_MODULE ibus \
        --set XMODIFIERS '@im=ibus' \
        --set IBUS_USE_PORTAL 1
    '';
  };
in
{
  home.packages = [ wechatWrapped ];
}
```

将文件导入 Home Manager 配置后，按你原有的方式运行 `home-manager switch` 或相应的 flake 命令。配置生效后分别验证两件事：微信消息时间是否与本地时区一致，以及聊天框能否输入中文。时区与输入法属于两条独立路径；其中一项恢复，不代表另一项也已经解决。
