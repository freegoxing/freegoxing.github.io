---
title: Nix 安装的应用在终端能运行，却不出现在 GNOME「打开方式」
date: 2026-10-25
tags: [linux, ubuntu, nix, home-manager, gnome]
categories: [Nix/NixOS]
permalink: 2026/10/25/linux/Nix-Home-Manager-应用未出现在-GNOME-打开方式/
description: 记录非 NixOS 上使用 Home Manager 安装 Okular 后，排查 GNOME Files 应用列表、MIME 关联和图形会话环境的过程。
---

# 前言

我在 Ubuntu 26.04 上通过 Nix 和 Home Manager 安装了 Okular。终端输入 `okular` 可以启动，PDF 也能通过 `gio open` 打开，但 GNOME Files 的「打开方式」里找不到它。临时调整会话环境后，列表恢复正常；重启后又复现。

这类问题容易让人反复安装软件或修改文件关联。实际上，终端能执行程序、系统知道 PDF 应由谁打开、文件管理器列出应用，是三个需要分别检查的环节。

# 1. 确认程序和桌面入口

先看当前终端找到的程序，以及 Nix profile 中的桌面入口：

```bash
command -v okular
ls ~/.nix-profile/share/applications/*okular*.desktop
```

本机的 `okular` 位于 `~/.nix-profile/bin/`，桌面入口包括 `okularApplication_pdf.desktop` 和 `org.kde.okular.desktop`。如果你的 Home Manager profile 不在 `~/.nix-profile`，后面的路径也要替换为实际路径。

再检查入口中与发现和启动有关的字段：

```bash
grep -E '^(Name|Exec|TryExec|MimeType|NoDisplay|Hidden|OnlyShowIn|NotShowIn)=' \
  ~/.nix-profile/share/applications/*okular*.desktop
```

本机的 PDF 专用入口包含 `Exec=okular %U`、`MimeType=application/pdf;...` 和 `NoDisplay=true`。`NoDisplay=true` 表示它不应出现在普通应用菜单中，但仍可参与 MIME 关联。因此，搜索列表里没有这个专用入口，不能单凭这一点判断安装失败。另一个入口是否显示，还应看它自己的字段和 GNOME 的实际列表。

GNOME 查找桌面入口时，`XDG_DATA_DIRS` 中应包含 profile 的 `share` 目录，例如 `~/.nix-profile/share`，而不是 `share/applications`。

# 2. 检查 PDF 关联

查看默认应用和 GIO 已识别的处理程序：

```bash
xdg-mime query default application/pdf
gio mime application/pdf
```

本机的默认入口是 `okularApplication_pdf.desktop`，`gio mime` 也能看到 Okular，`gio open /path/to/file.pdf` 可以打开 PDF。这说明 PDF 关联已经可用；没有必要为了修复列表显示而反复设置默认应用。

如果确实需要设置默认应用，可以执行：

```bash
xdg-mime default okularApplication_pdf.desktop application/pdf
```

使用 Home Manager 管理关联时，也可以在**尚未有同名配置**的前提下声明：

```nix
xdg.mimeApps = {
  enable = true;
  defaultApplications."application/pdf" = "okularApplication_pdf.desktop";
};
```

这只负责 MIME 默认值，不能代替后面的会话环境检查。

# 3. 比较终端与图形会话的环境

终端通常读取 Shell 初始化配置，Nautilus 不会因为 `.zshrc` 中添加了 Nix 路径，就自动得到相同的 `PATH`。分别查看当前终端和 systemd 用户管理器的环境：

```bash
printf '%s\n' "$PATH" | tr ':' '\n'
printf '%s\n' "$XDG_DATA_DIRS" | tr ':' '\n'
systemctl --user show-environment | grep -E '^(PATH|XDG_DATA_DIRS)='
```

如果 Nautilus 正在运行，再看它**启动时**继承的环境：

```bash
pid=$(pgrep -u "$USER" -o -x nautilus)
if [ -n "$pid" ]; then
  tr '\0' '\n' < "/proc/$pid/environ" | grep -E '^(PATH|XDG_DATA_DIRS)='
fi
```

在本机，终端的 `PATH` 包含 `~/.nix-profile/bin`，systemd 用户管理器和 Nautilus 的 `PATH` 却没有；而 `XDG_DATA_DIRS` 已包含 `~/.nix-profile/share`。这使会话 `PATH` 成为值得验证的线索，但还不能只凭环境差异断定它就是唯一原因。尤其要检查桌面入口是否设置了 `TryExec`：若它使用相对命令名，桌面环境会通过自己的 `PATH` 查找该命令。

# 4. 临时验证会话 PATH

下面的操作只影响当前登录会话。先取出 systemd 用户管理器的 `PATH`，补入实际的 Nix profile 目录，再同步到 D-Bus 激活环境：

```bash
user_path=$(systemctl --user show-environment | sed -n 's/^PATH=//p')
new_path="$HOME/.nix-profile/bin:$user_path"
systemctl --user set-environment "PATH=$new_path"
dbus-update-activation-environment --systemd "PATH=$new_path"
nautilus -q
```

之后从桌面重新打开 GNOME Files，检查「打开方式」。本机这样操作后，Okular 重新出现在列表中。由于这一步同时更新了 systemd、D-Bus 环境并重启 Nautilus，结果只能说明**会话环境及进程重启与现象有关**，不能证明其中某一条命令单独解决了问题。已经运行的程序也不会因 `set-environment` 自动获得新环境。

# 5. 检查重启后为何失效

我最初在 `~/.config/environment.d/20-nix.conf` 中写了：

```ini
PATH=${HOME}/.nix-profile/bin:${PATH}
```

但重启后，systemd 用户管理器的 `PATH` 仍没有 Nix 目录。检查本机配置发现，系统还有 `/usr/lib/environment.d/99-environment.conf`，它指向 `/etc/environment`，其中重新赋值了 `PATH`。

`environment.d` 中不同文件按**文件名排序**处理；同一个变量后面的赋值会覆盖前面的结果。因此，在这台机器上，`20-nix.conf` 的设置可能被 `99-environment.conf` 覆盖。先查看自己机器上的文件和内容，不要直接照搬文件名：

```bash
ls -l ~/.config/environment.d/ /etc/environment.d/ /usr/lib/environment.d/
cat /etc/environment
```

如果确认存在这种覆盖关系，可以将用户配置改为排序更靠后的 `~/.config/environment.d/zz-nix-path.conf`，内容仍为：

```ini
PATH=${HOME}/.nix-profile/bin:${PATH}
```

`environment.d` 不是 Shell 脚本，`${PATH}` 会按 systemd 的配置格式展开。若你的 Nix profile 使用其他目录，应替换上面的路径。修改后重载用户管理器并检查新环境：

```bash
systemctl --user daemon-reload
systemctl --user show-environment | grep '^PATH='
```

最后**注销并重新登录**，再次查看 Nautilus 的 `PATH` 和「打开方式」。这一步才足以检验修改是否跨登录生效。本次记录只验证了临时操作有效，以及旧配置在重启后没有生效；没有改名后重新登录的测试结果，因此这里不把持久化写成已完成的修复。

# 6. Home Manager 还需要检查什么

Home Manager 可以用于非 NixOS 发行版。配置中的 `targets.genericLinux.enable = true;` 用于改善这类系统的集成，但是否启用应先看现有配置和使用的版本，不必重复声明。本机已有 `~/.config/environment.d/10-home-manager.conf`，其中设置了 `XDG_DATA_DIRS`，所以无需再向 `.zshrc` 反复追加同一目录。

遇到相同问题时，我会按这个顺序查：先确认 `okular` 和 `.desktop` 文件，再看 MIME 关联；如果这些都正常，就比较终端、systemd 用户管理器和 Nautilus 的环境。临时操作有效后，再检查 `environment.d` 的覆盖顺序，并以重新登录后的结果判断是否真正修好。

# 参考资料

- [systemd `environment.d` 手册](https://www.freedesktop.org/software/systemd/man/latest/environment.d.html)
- [Desktop Entry Specification](https://specifications.freedesktop.org/desktop-entry/latest-single/)
- [Home Manager `targets.genericLinux` 选项](https://nix-community.github.io/home-manager/options/home-manager/targets.html)
- [Home Manager `xdg.mimeApps` 选项](https://nix-community.github.io/home-manager/options/home-manager/xdg.html)
