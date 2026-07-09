---
title: 利用 SSH 端口转发实现远程代理访问
date: 2026-07-08 22:15:00
tags: [ SSH, 代理, Linux, 端口转发 ]
categories: [linux]
---

# 前言

在日常使用中，我们经常会在远程服务器上部署 Clash 或其他代理工具（例如 mixed port 设为 `17897`）。为了在本地也能安全、便捷地使用这个远端代理，直接开放服务器防火墙端口并不是一个明智的选择。

通过 SSH 端口转发（SSH Tunneling），我们可以将远端的代理端口安全地映射到本地。结合 Systemd 用户服务和 Zsh 配置，就能实现优雅的无缝代理体验。本文将分享我目前的方案。

# SSH 客户端配置

首先，我们需要在本地生成 SSH 密钥对（如果还没有的话）。推荐使用安全性更高的 ed25519 算法：

```bash
ssh-keygen -t ed25519 -C "你的邮箱"
```

一路回车即可。接着，将生成的公钥复制到远程服务器，以实现免密登录：

```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub 用户名@远程服务器地址
```

完成免密登录后，为了让后续连接更加简便，我们需要配置一下 `~/.ssh/config`，设置好远程主机的别名：

```text ~/.ssh/config
Host ubuntu
    HostName 远程服务器地址
    User free
    IdentityFile ~/.ssh/id_ed25519
    Port 22
```

这样我们之后只需使用 `ssh ubuntu` 就能直接连上远程服务器。

# Systemd 用户服务配置

为了让代理通道能在后台稳定运行并在断线时自动重连，我们可以把它配置为一个 Systemd User Service。

在本地机器上创建文件 `~/.config/systemd/user/ssh-proxy.service`：

```ini ~/.config/systemd/user/ssh-proxy.service
[Unit]
Description=SSH tunnel to remote proxy
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
# 启动前检查本地 27897 端口是否已被占用
ExecStartPre=/usr/bin/bash -c '! ss -lnt | grep -q ":27897 "'
# 建立 SSH 隧道，将本地 27897 转发到远端 17897
ExecStart=/usr/bin/ssh \
-N \
-L 27897:127.0.0.1:17897 \
-o ExitOnForwardFailure=yes \
-o ServerAliveInterval=30 \
-o ServerAliveCountMax=3 \
ubuntu
Restart=always
RestartSec=5

[Install]
WantedBy=default.target
```

这里的关键参数解释：
- `-N`：不执行远程命令，仅用于端口转发。
- `-L 27897:127.0.0.1:17897`：将本地的 `27897` 端口转发到远端的 `127.0.0.1:17897` 代理端口。
- `ExitOnForwardFailure=yes`：如果转发失败则退出，配合 `Restart=always` 实现自动重试。

加载配置并让它开机自启（按需）：
```bash
systemctl --user daemon-reload
# 如果需要开机自启可执行：systemctl --user enable ssh-proxy
```

# Zsh 终端代理配置

有了后台隧道，我们可以利用 Bash/Zsh 的函数来快捷开关终端代理，同时联动 Systemd 服务的启停。将以下代码加入你的 `~/.zshrc`：

```bash ~/.zshrc
proxy-on() {
    # 启动 SSH 隧道服务
    systemctl --user start ssh-proxy

    # 设置环境变量
    export http_proxy=http://127.0.0.1:27897
    export https_proxy=http://127.0.0.1:27897
    export HTTP_PROXY=$http_proxy
    export HTTPS_PROXY=$https_proxy
}

proxy-off() {
    # 清除环境变量
    unset http_proxy https_proxy HTTP_PROXY HTTPS_PROXY
    # 停止 SSH 隧道服务
    systemctl --user stop ssh-proxy
}
```

之后在终端中只需要输入 `proxy-on` 或 `proxy-off` 即可优雅地开启/关闭代理环境。

# 常见应用的代理设置

部分应用并不遵循系统的环境变量，需要我们单独设置代理。

## Chrome 浏览器

为 Chrome 设置一个 alias，强制其通过我们本地映射出来的端口走代理：

```bash ~/.zshrc
alias chrome-proxy="google-chrome --proxy-server=\"http://127.0.0.1:27897\""
```

## Steam

Steam 客户端自带代理设置，但其入口隐藏得比较深：
- 窗口模式下是找不到代理设置的。
- 必须进入 **大屏幕模式（Big Picture Mode）**。
- 依次点击 **设置** -> **高级设置** -> **HTTP 代理**。

![HTTP代理](https://img.556756.xyz/PicGo/blogs/2026/07/20260708221722528.png)

- 将代理地址修改为 `127.0.0.1`，端口填写对应的 `27897` 即可。

![HTTP代理设置](https://img.556756.xyz/PicGo/blogs/2026/07/20260708221810407.png)

# 总结

借助 SSH 强大的端口转发功能与 Systemd 用户服务的结合，我们成功将远程的代理端口安全地映射至本地，且具备自动重连机制。配合 Zsh 函数实现了无缝启停，避免了长期后台占用，也保障了各类应用的网络畅通。
