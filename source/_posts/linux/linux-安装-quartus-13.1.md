---
title: Quartus II 13.1 在现代 Ubuntu 上的安装与 32 位依赖兼容
date: 2026-08-10 10:00:00
tags: [linux, ubuntu, quartus, fpga]
categories: [linux]
description: 在现代 Ubuntu 上为 Quartus II 13.1 配置 i386 依赖与私有 libpng12 兼容库。
---

# 前言

Quartus II 13.1 是较早期的软件，在现代 Ubuntu 上启动时，常因缺少旧版 32 位动态库而失败，例如：

```text
quartus: error while loading shared libraries: libpng12.so.0:
cannot open shared object file: No such file or directory
```

本文以 Quartus II 13.1 为例，说明如何在不污染系统库的前提下补齐运行依赖。

{% note info %}

处理原则如下：现代 Ubuntu 仓库仍提供的依赖，直接通过 `apt` 安装对应的 `:i386` 软件包；已被淘汰的旧 ABI 库（如 `libpng12.so.0`）仅放入用户私有兼容目录，并通过 `LD_LIBRARY_PATH` 加载。Quartus 自带的 Qt、TBB、ICU、Tcl、`libsys_*` 和 `libdb_*` 等库仍使用其自带版本。

{% endnote %}

# 1. 解压并启动安装器

下载 `Quartus-web-13.1.0.162-linux.tar` 后，先在下载目录解压：

```bash
tar -xf Quartus-web-13.1.0.162-linux.tar
cd Quartus-web-13.1.0.162-linux
```

解压后的目录应包含 `setup.sh` 与 `components/`，其中 `components/` 内有 Quartus、ModelSim、帮助文档和各器件系列的安装包。

Quartus 属于第三方开发工具，建议安装在 `/opt`，而不要使用安装器默认的 `$HOME/altera/...` 路径。先创建仅供 Quartus 使用的安装目录，并将其所有权交给当前用户：

```bash
sudo install -d -o "$USER" -g "$USER" /opt/altera
```

随后回到解压后的目录，启动安装器：

```bash
./setup.sh
```

如果在父目录直接执行 `./setup.sh` 并看到“没有那个文件或目录”，只表示当前目录下没有该脚本；进入 `Quartus-web-13.1.0.162-linux` 后再运行即可。

安装器最开始会输出：

```text
You must have the 32-bit compatibility libraries installed for the Quartus II installer and software to operate properly.
```

这是一条正常的通用提示，提醒 Quartus II 13.1 的安装器与软件本体都依赖 32 位兼容库。若仅显示该提示，安装器可继续进入图形安装界面。在安装目录页面填写 `/opt/altera/13.1`；安装完成后，Quartus 根目录为 `/opt/altera/13.1/quartus`。然后按需选择组件与器件系列即可完成常规安装。

真正需要处理的是其后出现的动态链接错误，例如：

```text
tb2_install: error while loading shared libraries: libpng12.so.0: cannot open shared object file: No such file or directory
```

这说明安装器自身无法加载已被现代 Ubuntu 淘汰的 `libpng12.so.0`，会导致安装流程中断。下文先解决该兼容库问题，再以该库启动安装器。

# 2. 确认 Quartus 实际程序的位数

`/opt/altera/13.1/quartus/bin/quartus` 是 Shell 启动脚本，不能直接用 `file` 判断程序位数。应检查实际的 ELF 可执行文件：

```bash
QUARTUS_ROOT="/opt/altera/13.1/quartus"
file "$QUARTUS_ROOT/linux/quartus"
```

本机输出如下：

```text
ELF 32-bit LSB executable, Intel i386
```

因此，这套 Quartus 13.1 需要 **32 位 i386 动态库**。不要使用 amd64/x86-64 版本的 `libpng12.so.0`，否则会出现 ELF 位数不匹配的问题。

# 3. 建立安装前可用的私有兼容库目录

```bash
mkdir -p ~/.local/lib/quartus-13.1-compat
```

该目录为：

```text
~/.local/lib/quartus-13.1-compat/
```

它只用于存放现代 Ubuntu 已不再提供、但 Quartus 13.1 的安装器和软件本体仍需要的旧 ABI 库。由于安装器启动前 Quartus 的安装目录尚不存在，兼容库应先放在这个用户私有目录，而不是系统库目录。

# 4. 处理 `libpng12.so.0`

Quartus 13.1 需要以下库：

```text
libpng12.so.0
```

现代 Ubuntu 通常不再提供 `libpng12`，因此不建议为了它修改系统库。请下载 **i386 版本的旧 `libpng12-0` Debian 或 Ubuntu 软件包**，仅解包使用，不要通过 `dpkg -i` 安装。

例如：

```bash
cd /tmp
wget http://archive.debian.org/debian/pool/main/libp/libpng/libpng12-0_1.2.50-2+deb8u3_i386.deb

mkdir -p libpng12-i386

dpkg-deb -x libpng12-0_*_i386.deb libpng12-i386
```

查找解包后的库文件：

```bash
find libpng12-i386 -name 'libpng12.so*' -ls
```

通常会看到：

```text
libpng12.so.0
libpng12.so.0.xx.x
```

![libpng12.so](https://img.556756.xyz/PicGo/blogs/2026/08/20260810175846691.png)

将其复制到 Quartus 的私有兼容目录：

```bash
cp -a libpng12-i386/lib/i386-linux-gnu/libpng12.so* \
    ~/.local/lib/quartus-13.1-compat/
```

如果实际解包路径不同，请以 `find` 的输出为准。随后确认库的位数：

```bash
file ~/.local/lib/quartus-13.1-compat/libpng12.so.0
```

输出应类似：

```text
ELF 32-bit LSB shared object, Intel 80386
```

# 5. 不要直接相信裸 `ldd` 的大量 `not found`

直接执行：

```bash
QUARTUS_ROOT="/opt/altera/13.1/quartus"
ldd "$QUARTUS_ROOT/linux/quartus"
```

![ldd结果](https://img.556756.xyz/PicGo/blogs/2026/08/20260810180110162.png)

这不代表这些库真的没有安装。Quartus 的正常启动链路为：

```text
bin/quartus
    ↓
adm/qenv.sh
    ↓
设置 Quartus 自己的 LD_LIBRARY_PATH
    ↓
linux/quartus
```

直接运行 `ldd linux/quartus` 会绕过 `qenv.sh`。我们通过

加入 Quartus 自带库和私有兼容库后，再检查真正缺失的依赖：

```bash
QUARTUS_ROOT="/opt/altera/13.1/quartus"

LD_LIBRARY_PATH="$HOME/.local/lib/quartus-13.1-compat:$QUARTUS_ROOT/linux" \
ldd "$QUARTUS_ROOT/linux/quartus" | grep 'not found'
```

如果安装时选择了其他目录，只需将 `QUARTUS_ROOT` 改为实际的 Quartus 根目录。

![缺少的i386依赖](https://img.556756.xyz/PicGo/blogs/2026/08/20260810180828223.png)

而是现代 Ubuntu 仍提供的正常 X11 依赖。

# 7. 通过 `apt` 安装现代 i386 依赖

先启用 i386 multiarch：

```bash
sudo dpkg --add-architecture i386
sudo apt update
```

然后安装所需依赖：

```bash
sudo apt install \
    libsm6:i386 \
    libice6:i386
```

这些库由 Ubuntu 包管理器正常维护，无需放入 Quartus 的 `compat` 目录。

# 8. 再次检查依赖

```bash
QUARTUS_ROOT="/opt/altera/13.1/quartus"

LD_LIBRARY_PATH="$HOME/.local/lib/quartus-13.1-compat:$QUARTUS_ROOT/linux" \
ldd "$QUARTUS_ROOT/linux/quartus" | grep 'not found'
```

若没有任何输出，说明 Quartus GUI 主程序的直接动态链接依赖已满足。也可以单独确认 `libpng12` 的加载位置：

```bash
QUARTUS_ROOT="/opt/altera/13.1/quartus"

LD_LIBRARY_PATH="$HOME/.local/lib/quartus-13.1-compat:$QUARTUS_ROOT/linux" \
ldd "$QUARTUS_ROOT/linux/quartus" | grep png
```

它应指向：

```text
~/.local/lib/quartus-13.1-compat/libpng12.so.0
```

# 9. 启动 Quartus

不要直接执行 `linux/quartus`，仍应使用官方启动脚本：

```bash
QUARTUS_ROOT="/opt/altera/13.1/quartus"

LD_LIBRARY_PATH="$HOME/.local/lib/quartus-13.1-compat${LD_LIBRARY_PATH:+:$LD_LIBRARY_PATH}" \
"$QUARTUS_ROOT/bin/quartus"
```

该命令会先加入私有兼容库目录，再由 `bin/quartus` 调用 `adm/qenv.sh`，继续配置 Quartus 的 `linux/` 等内部运行环境。

# 10. 可选：创建本地启动 wrapper

如果不希望每次启动时都手动指定 `LD_LIBRARY_PATH`，可创建一个本地启动脚本：

```bash
mkdir -p ~/.local/bin

cat > ~/.local/bin/quartus-13.1 <<'EOF'
#!/bin/sh

QUARTUS_ROOT="/opt/altera/13.1/quartus"

export LD_LIBRARY_PATH="$HOME/.local/lib/quartus-13.1-compat${LD_LIBRARY_PATH:+:$LD_LIBRARY_PATH}"

exec "$QUARTUS_ROOT/bin/quartus" "$@"
EOF

chmod +x ~/.local/bin/quartus-13.1
```

之后可直接运行：

```bash
~/.local/bin/quartus-13.1
```
