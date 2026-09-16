---
title: Quartus II 13.1 在现代 Ubuntu 上的安装与 64 位兼容
date: 2026-08-10 10:00:00
tags: [linux, ubuntu, quartus, fpga]
categories: [linux]
description: 在现代 Ubuntu 上以 64 位模式启动 Quartus II 13.1，并通过私有 libpng12 兼容库解决旧 ABI 依赖问题。
---

# 前言

Quartus II 13.1 是较早期的软件，在现代 Ubuntu 上启动时，可能因缺少旧版动态库而失败，例如：

```text
quartus: error while loading shared libraries: libpng12.so.0:
cannot open shared object file: No such file or directory
```

Quartus II 13.1 同时提供 32 位和 64 位程序。现代 64 位 Ubuntu 无须为 Quartus GUI 补装 `libsm6:i386`、`libice6:i386` 等 32 位依赖；只需准备 amd64 版本的 `libpng12.so.0`，并在启动时显式传入 `--64bit`。

{% note info %}

本文采用的原则是：使用 Quartus 自带的 64 位运行环境，仅将现代 Ubuntu 已淘汰的 `libpng12.so.0` 放入用户私有兼容目录，再通过 `LD_LIBRARY_PATH` 加载。Quartus 自带的 Qt、TBB、ICU、Tcl、`libsys_*` 和 `libdb_*` 等库仍使用其自带版本，不修改系统库。

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

安装器最开始可能会输出：

```text
You must have the 32-bit compatibility libraries installed for the Quartus II installer and software to operate properly.
```

这是旧版安装器给出的通用提示，并不表示后续必须以 32 位模式运行 Quartus。若安装器能够正常进入图形界面，可直接继续安装，无须为本文采用的 64 位启动方案补装 i386 软件包。

在安装目录页面填写 `/opt/altera/13.1`；安装完成后，Quartus 根目录为 `/opt/altera/13.1/quartus`。然后按需选择组件与器件系列即可完成常规安装。

# 2. 确认 64 位程序

`/opt/altera/13.1/quartus/bin/quartus` 是 Shell 启动脚本，不能直接用 `file` 判断实际程序的位数。Quartus 的 64 位 ELF 可执行文件位于 `linux64/`：

```bash
QUARTUS_ROOT="/opt/altera/13.1/quartus"
file "$QUARTUS_ROOT/linux64/quartus"
```

输出应包含：

```text
ELF 64-bit LSB executable, x86-64
```

`linux/quartus` 是 32 位程序，但本文不会启动它。后续通过官方脚本传入 `--64bit`，让脚本选择 `linux64/quartus` 及对应的 64 位运行环境。

# 3. 建立私有兼容库目录

```bash
mkdir -p ~/.local/lib/quartus-13.1-compat
```

该目录为：

```text
~/.local/lib/quartus-13.1-compat/
```

它只用于存放现代 Ubuntu 已不再提供、但 Quartus II 13.1 仍需要的旧 ABI 库。将兼容库放在用户目录中，可以避免覆盖或污染系统库。

# 4. 提取 amd64 版本的 `libpng12.so.0`

Quartus II 13.1 的 64 位程序需要：

```text
libpng12.so.0
```

现代 Ubuntu 通常不再提供 `libpng12`，可以从 Debian 归档下载 amd64 软件包。这里只解包并复制动态库，不使用 `dpkg -i` 将旧软件包安装到系统中：

```bash
cd /tmp
wget http://archive.debian.org/debian/pool/main/libp/libpng/libpng12-0_1.2.50-2+deb8u3_amd64.deb

rm -rf libpng12-amd64
mkdir libpng12-amd64
dpkg-deb -x libpng12-0_1.2.50-2+deb8u3_amd64.deb libpng12-amd64
```

查看解包后的库文件：

```bash
find libpng12-amd64 -name 'libpng12.so*' -ls
```

该软件包中的库位于 `lib/x86_64-linux-gnu/`。将版本文件和符号链接一并复制到 Quartus 的私有兼容目录：

```bash
cp -a \
    /tmp/libpng12-amd64/lib/x86_64-linux-gnu/libpng12.so.0* \
    ~/.local/lib/quartus-13.1-compat/
```

随后确认库的位数：

```bash
file ~/.local/lib/quartus-13.1-compat/libpng12.so.0
```

输出应包含：

```text
ELF 64-bit LSB shared object, x86-64
```

兼容目录中不要混用 i386 和 amd64 版本的同名库。如果此前放入过 32 位的 `libpng12.so.0`，应确保该符号链接最终指向刚复制的 amd64 版本。

# 5. 检查 64 位程序的依赖

直接对 `linux64/quartus` 执行裸 `ldd`，可能无法找到 Quartus 自带的库，因为正常启动时 `bin/quartus` 会调用 `adm/qenv.sh` 设置运行环境。检查依赖时，需要同时加入私有兼容目录和 Quartus 的 `linux64/` 目录：

```bash
QUARTUS_ROOT="/opt/altera/13.1/quartus"

LD_LIBRARY_PATH="$HOME/.local/lib/quartus-13.1-compat:$QUARTUS_ROOT/linux64" \
ldd "$QUARTUS_ROOT/linux64/quartus" | grep 'not found'
```

若没有任何输出，说明该程序的直接动态链接依赖已经满足。也可以单独确认 `libpng12` 的加载位置：

```bash
QUARTUS_ROOT="/opt/altera/13.1/quartus"

LD_LIBRARY_PATH="$HOME/.local/lib/quartus-13.1-compat:$QUARTUS_ROOT/linux64" \
ldd "$QUARTUS_ROOT/linux64/quartus" | grep libpng12
```

结果应指向 `~/.local/lib/quartus-13.1-compat/libpng12.so.0` 对应的实际路径。

# 6. 以 64 位模式启动 Quartus

不要直接执行 `linux64/quartus`，仍应使用官方启动脚本，让它完成其余运行环境配置。同时必须传入 `--64bit`：

```bash
QUARTUS_ROOT="/opt/altera/13.1/quartus"

LD_LIBRARY_PATH="$HOME/.local/lib/quartus-13.1-compat${LD_LIBRARY_PATH:+:$LD_LIBRARY_PATH}" \
"$QUARTUS_ROOT/bin/quartus" --64bit
```

# 7. 创建启动 wrapper

如果不希望每次启动时都手动指定 `LD_LIBRARY_PATH` 和 `--64bit`，可创建一个本地启动脚本：

```bash
mkdir -p ~/.local/bin

cat > ~/.local/bin/quartus-13.1 <<'EOF'
#!/usr/bin/env bash

QUARTUS_ROOT="/opt/altera/13.1/quartus"
COMPAT_LIB="$HOME/.local/lib/quartus-13.1-compat"

export LD_LIBRARY_PATH="$COMPAT_LIB${LD_LIBRARY_PATH:+:$LD_LIBRARY_PATH}"

exec "$QUARTUS_ROOT/bin/quartus" --64bit "$@"
EOF

chmod +x ~/.local/bin/quartus-13.1
```

之后可直接运行：

```bash
~/.local/bin/quartus-13.1
```

如果 `~/.local/bin` 已在 `PATH` 中，也可以直接执行：

```bash
quartus-13.1
```
