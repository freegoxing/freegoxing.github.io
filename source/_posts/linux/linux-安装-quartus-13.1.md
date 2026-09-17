---
title: Quartus II 13.1 在现代 Ubuntu 上的安装、补丁与 Docker 兼容
date: 2026-08-10 10:00:00
tags: [linux, ubuntu, quartus, fpga, docker]
categories: [linux]
description: 在现代 Ubuntu 上以 64 位模式运行 Quartus II 13.1，处理 libpng12、libsys_cpt.so、旧式网卡名和 Fitter 卡死问题，并在必要时使用 CentOS 6 Docker 完成编译。
---

# 前言

Quartus II 13.1 是 2013 年的软件。在现代 Ubuntu 上运行它，问题并不只是一项缺失的动态库，而是一串彼此独立的兼容性问题：

```text
quartus: error while loading shared libraries: libpng12.so.0:
cannot open shared object file: No such file or directory
```

- GUI 启动时缺少 `libpng12.so.0`；
- 64 位运行库 `libsys_cpt.so` 需要针对当前文件版本应用补丁；
- 旧代码只会枚举 `eth0` 到 `eth9`，不能识别现代 Linux 的 predictable interface names；
- 即使上述问题已经处理，Fitter 仍可能卡在 Quartus 自带的旧版 `libtbbmalloc.so.2` 中。

最终存在两条使用路径：

```text
准备 64 位运行环境并应用补丁
              ↓
确保宿主机存在 eth0 并测试实际工程
              ├── Fitter 正常完成 → 直接在宿主机使用
              └── Fitter 卡死     → 使用 CentOS 6 Docker 编译
```

{% note info %}

本文先使用 Quartus 自带的 64 位运行环境，仅将现代 Ubuntu 已淘汰的 `libpng12.so.0` 放入用户私有兼容目录。能够正常编译的用户到此即可；只有确认 Fitter 卡死的用户，才需要后面的 CentOS 6 Docker 兼容环境。

二进制补丁只适用于 SHA-256 完全匹配的文件。不要跳过哈希和旧字节校验，也不要把偏移套用到其他 Quartus 版本或构建。使用软件及许可证时，请遵守适用的许可协议和当地法律。

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

# 7. 为 `libsys_cpt.so` 应用补丁

本次实践中的补丁对象是：

```text
/opt/altera/13.1/quartus/linux64/libsys_cpt.so
```

先进入该目录。脚本会校验原文件 SHA-256，复制出新文件，并在写入前逐项核对偏移处的旧字节：

```bash
cd /opt/altera/13.1/quartus/linux64

python3 - <<'PY'
from pathlib import Path
import hashlib
import shutil

src = Path("libsys_cpt.so")
dst = Path("libsys_cpt.so.patched")

expected_sha256 = "8c82dbd61499f61c06e102731aa1bab12cc31aa7b0c9e9d6b9fe4f18d174e7ab"
actual_sha256 = hashlib.sha256(src.read_bytes()).hexdigest()
assert actual_sha256 == expected_sha256, (
    f"libsys_cpt.so 版本不匹配：{actual_sha256}"
)

patches = [
    # offset, old bytes, new bytes
    (0x5A436, bytes.fromhex("87"),          bytes.fromhex("6E")),
    (0x8EEFD, bytes.fromhex("0F 85"),       bytes.fromhex("90 E9")),
    (0x8EF34, bytes.fromhex("F8 FF FF FF"), bytes.fromhex("00 00 00 00")),
    (0x8F177, bytes.fromhex("8D FF FF FF"), bytes.fromhex("00 00 00 00")),
]

shutil.copy2(src, dst)

with dst.open("r+b") as f:
    for offset, old, new in patches:
        f.seek(offset)
        current = f.read(len(old))
        assert current == old, (
            f"偏移 {offset:#x} 不匹配："
            f"期望 {old.hex(' ')}, 实际 {current.hex(' ')}"
        )
        f.seek(offset)
        f.write(new)

print(f"已生成：{dst}")
print("SHA-256:", hashlib.sha256(dst.read_bytes()).hexdigest())
PY
```

脚本成功后，再备份和替换原文件：

```bash
cp -a libsys_cpt.so libsys_cpt.so.original
cp -a libsys_cpt.so.patched libsys_cpt.so
```

如果安装目录不能由当前用户写入，只对最后两条复制命令使用 `sudo`。需要回退时执行：

```bash
cp -a libsys_cpt.so.original libsys_cpt.so
```

# 8. 确保 Quartus 能识别宿主网卡

Quartus II 13.1 的相关旧代码只会尝试 `eth0` 到 `eth9`。现代 Ubuntu 通常使用 `eno1`、`enp5s0` 等名称，因此即使网卡存在，Quartus 也可能无法取得 MAC 地址。

本次实践使用 systemd `.link` 文件，按物理网卡的 MAC 地址精确匹配并重命名。配置文件为：

```text
/etc/systemd/network/10-quartus-eth0.link
```

内容如下：

```ini
[Match]
MACAddress=40:c2:ba:4c:90:00

[Link]
Name=eth0
```

其中 `MACAddress` 必须换成实际需要提供给 Quartus 的物理网卡地址。使用 MAC 匹配可以避免接口原名称变化后规则失效，也能防止把其他网卡误命名为 `eth0`。

保存配置后重启，使 udev 在网卡初始化时应用新的名称：

```bash
sudo reboot
```

系统重新启动后确认：

```bash
ip link show eth0
cat /sys/class/net/eth0/address
```

然后直接调用 Quartus 附带的 FLEXnet 工具验证识别结果：

```bash
cd /opt/altera/13.1/quartus
./linux64/lmutil lmhostid
```

本次实践得到的输出为：

```text
lmutil - Copyright (c) 1989-2008 Acresso Software Inc. All Rights Reserved.
The FLEXnet host ID of this machine is "40c2ba4c9000"
```

这里的 Host ID 正是网卡 MAC 地址去掉冒号后的形式，说明旧版 FLEXnet 已经通过 `eth0` 正确取得物理地址。如果得到 `000000000000`，应重新检查 `.link` 文件匹配的 MAC、重启后的接口名以及该接口是否真实存在。

许可证文件中的 `HOSTID` 使用同一种格式。下面只将三处 Host ID 替换为 `xxxxxxxxxxxx`，其他字段保持实际文件内容：

```text
FEATURE quartus alterad 2035.12 permanent uncounted 295142B536B3 \
    HOSTID=xxxxxxxxxxxx SIGN="0C8D 31B5 AD64 E1C4 C6F9 1540 5072 \
    C53D 386C 7A5E 09F0 6FE0 EBAB A42C C139 015B 44B1 D3E6 8F4B \
    CD45 FAFF B30C 77BE FA54 955D 022F 0663 87C2 26B0 7305"

FEATURE quartus_partial_reconfig alterad 2035.12 permanent \
    uncounted A78162BD7ADA HOSTID=xxxxxxxxxxxx TS_OK SIGN="1E75 \
    CDEE 12CC B7C3 EFF7 8BA4 1D44 658E 47DF 7650 178C D53B 25A6 \
    70A7 D6E5 021A B84F 77B4 0BA7 9966 9469 74F4 955F 9E2C DA28 \
    CC7E D35F 3C4F 9CBD 146F"

FEATURE 6AF7_00A2 alterad 2035.12 permanent uncounted E75BE809707E \
    VENDOR_STRING="iiiiiiiihdLkhIIIIIIIIUPDuiaaaaaaaa11X38DDDDDDDDpjz5cddddddddtmGzGJJJJJJJJbqIh0uuuuuuuugYYWiVVVVVVVVbp0FVHHHHHHHHBUEakffffffffD2FFRkkkkkkkkWL$84" \
    HOSTID=xxxxxxxxxxxx TS_OK SIGN="1E27 C980 33CD 38BC 5532 368B \
    116D C1F8 34E0 5436 99A0 5A2E 1C8C 8DD0 C9C6 011B A5A9 932B \
    08DE C5ED 9E62 2868 5A32 6397 D9B8 5C3A B8E8 4E4F CEC7 C836"
```

其中 `xxxxxxxxxxxx` 表示 12 位、无冒号的小写或大写十六进制 MAC 地址。以本节的示例网卡为例：

```text
网卡地址：40:c2:ba:4c:90:00
FLEXnet Host ID：40c2ba4c9000
license.dat：HOSTID=40c2ba4c9000
```

同一份许可证中的各个 `FEATURE` 项应绑定到许可证签发时指定的同一个 Host ID。

本文将许可证放在：

```text
/opt/altera/13.1/license.dat
```

宿主机直接运行时可以这样指定：

```bash
export LM_LICENSE_FILE=/opt/altera/13.1/license.dat
export ALTERA_LICENSE_FILE=/opt/altera/13.1/license.dat
```

后文 Docker wrapper 会把相同的路径通过这两个环境变量传入容器。

远程修改网卡名可能立即中断网络，建议在本地控制台操作。若系统还有 Netplan 或 NetworkManager 的接口名相关配置，也要同步检查，避免多个规则互相冲突。

{% note info %}

后文 Docker 使用 `--network host`。它不会自动把 `eno1` 改名为 `eth0`，而是让容器共享宿主网络命名空间。因此，容器能够识别 `eth0` 的前提是宿主机已经完成重命名。

{% endnote %}

# 9. 用实际工程判断是否需要 Docker

先直接在宿主 Ubuntu 上编译：

```bash
quartus_fit --64bit exp1 -c exp1
```

本次异常发生时，日志停在如下位置：

```text
Info: Running Quartus II 64-Bit Fitter
Info: Version 13.1.0 Build 162 10/23/2013 SJ Web Edition
Info: Command: quartus_fit exp1 -c exp1
Info: Project  = exp1
Info: Revision = exp1
Info (119006): Selected device 5CSEMA5F31C6 for design "exp1"
Info (21077): Low junction temperature is 0 degrees C
Info (21077): High junction temperature is 85 degrees C
Info (171003): Fitter is performing an Auto Fit compilation, which may decrease Fitter effort to reduce compilation time
Warning (15714): Some pins have incomplete I/O assignments. Refer to the I/O Assignment Warnings report for details
```

此时 Fitter 正在创建 Cyclone V 的 I/O / peripheral placement 数据结构，并需要继续申请内存。实际跟踪发现，程序进入 Quartus 自带的 `libtbbmalloc.so.2` 分配路径后不再向前推进。这是 Quartus 13.1 全局代理 malloc 使用的旧 TBB allocator 与当前运行环境之间的兼容性异常，不是上面的 I/O assignment warning 导致的。

不要只凭终端暂时没有新日志判断卡死，可以同时观察 CPU 和工程文件：

```bash
ps aux | grep '[q]uartus_fit'
top
find . -type f -mmin -1 | head
```

如果 Fitter 能继续并完成编译，直接使用宿主 Ubuntu 即可，无须阅读 Docker 部分。如果它稳定复现上述卡死，再使用第 11 节的 CentOS 6 兼容环境。

# 10. 创建宿主机启动 wrapper

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

如果实际工程能够在宿主机完成 Fitter，这就是最终方案。以下内容只针对补丁后仍会卡死的用户。

# 11. 使用 CentOS 6 Docker 兼容环境

Docker 在这里不是用来重新安装 Quartus，而是提供与 2013 年软件更接近的 glibc、Qt4、X11 和 OpenSSL 用户态。Quartus 本体和 FPGA 工程仍保存在宿主机：

```text
Ubuntu 宿主机
├── /opt/altera/13.1/              # Quartus 本体
├── FPGA 工程                      # 源文件和编译结果
├── 已配置为 eth0 的宿主网卡
└── quartus13-centos6:fixed        # 旧用户态兼容镜像
```

## 11.1 创建配置容器

```bash
docker pull centos:6

docker run -it \
  --name quartus13-centos6 \
  --network host \
  -v /opt/altera/13.1:/opt/altera/13.1:ro \
  centos:6 \
  /bin/bash
```

配置阶段不要加 `--rm`。进入容器后确认系统版本和网卡：

```bash
cat /etc/centos-release
ip link show eth0
```

`--network host` 让容器看到宿主机已经重命名好的 `eth0`，而不是另外创建一块 Docker 虚拟网卡。

## 11.2 修复 EOL 软件源并安装依赖

CentOS 6 已停止维护，普通 mirrorlist 已不可用。先备份：

```bash
cp -a /etc/yum.repos.d /etc/yum.repos.d.bak
```

将 `/etc/yum.repos.d/CentOS-Base.repo` 改为可用的 CentOS 6.10 归档源，例如：

```ini
[base]
name=CentOS-6.10 - Base
baseurl=https://mirrors.aliyun.com/centos-vault/6.10/os/$basearch/
gpgcheck=0
enabled=1

[updates]
name=CentOS-6.10 - Updates
baseurl=https://mirrors.aliyun.com/centos-vault/6.10/updates/$basearch/
gpgcheck=0
enabled=1

[extras]
name=CentOS-6.10 - Extras
baseurl=https://mirrors.aliyun.com/centos-vault/6.10/extras/$basearch/
gpgcheck=0
enabled=1
```

重建缓存并安装运行库：

```bash
unset LD_LIBRARY_PATH
yum clean all
yum makecache

yum install -y \
  libpng freetype fontconfig \
  libX11 libXext libXrender libXft libSM libICE libXScrnSaver \
  libgomp xz-libs openssl098e.x86_64

mkdir -p /opt/quartus13-compat
```

若旧 TLS/CA 环境无法访问 HTTPS，可换用归档站实际支持的 HTTP 地址或其他 CentOS 6 Vault。若具体组件仍报告缺库，可这样定位：

```bash
ldd /opt/altera/13.1/quartus/linux64/quartus | grep 'not found'
yum provides '*/缺少的库名.so*'
```

Quartus GUI 使用旧 Qt4。如果请求的文件名与 CentOS 6 中实际文件名不同，先用下面的命令定位；只有确认 ABI 对应后，才在 `/opt/quartus13-compat` 中建立软链接：

```bash
find /usr/lib64 /usr/lib -name 'libQt*.so*' 2>/dev/null
```

不要把不同 major ABI 的库仅凭文件名相似强行链接。

## 11.3 固化镜像

配置完成后退出容器并保存：

```bash
exit
docker commit quartus13-centos6 quartus13-centos6:fixed
docker images | grep quartus13
```

确认镜像存在后，可以删除配置容器：

```bash
docker rm quartus13-centos6
```

这里删除的只是配置容器，不要删除 `quartus13-centos6:fixed` 镜像。

# 12. 创建 Docker 启动 wrapper

创建 `/usr/local/bin/quartus-13.1-docker`：

```bash
sudo editor /usr/local/bin/quartus-13.1-docker
```

写入：

```bash
#!/usr/bin/env bash
set -e

QUARTUS_ROOT="/opt/altera/13.1/quartus"
IMAGE="quartus13-centos6:fixed"

xhost +SI:localuser:"$USER" >/dev/null

exec docker run --rm -it \
    --user "$(id -u):$(id -g)" \
    --network host \
    -e DISPLAY="$DISPLAY" \
    -e HOME="$HOME" \
    -e LANG=C \
    -e LC_ALL=C \
    -e QT_X11_NO_MITSHM=1 \
    -e QT_X11_NO_XRENDER=1 \
    -e LD_LIBRARY_PATH=/opt/quartus13-compat:/opt/altera/13.1/quartus/linux64 \
    -e LM_LICENSE_FILE=/opt/altera/13.1/license.dat \
    -e ALTERA_LICENSE_FILE=/opt/altera/13.1/license.dat \
    -v /tmp/.X11-unix:/tmp/.X11-unix:rw \
    -v /usr/share/fonts:/usr/share/fonts/host:ro \
    -v /opt/altera/13.1:/opt/altera/13.1:ro \
    -v "$HOME":"$HOME":rw \
    -w "$PWD" \
    "$IMAGE" \
    "$QUARTUS_ROOT/bin/quartus" --64bit "$@"
```

添加执行权限并从工程目录启动：

```bash
sudo chmod +x /usr/local/bin/quartus-13.1-docker
cd /path/to/fpga-project
quartus-13.1-docker
```

其中：

- `--user` 避免生成属于 root 的工程文件；
- `--network host` 传递宿主网络命名空间及已经配置好的 `eth0`；
- Quartus 安装目录只读挂载，`$HOME` 中的工程可读写；
- `-w "$PWD"` 使 GUI 从当前工程目录启动；
- 两个 `QT_X11_*` 变量用于规避旧 Qt4 与现代 X11/XWayland 的部分兼容问题；
- `--rm` 只删除本次临时容器，不删除镜像、工程或编译结果。

# 13. 验证、排错和方案边界

在容器版 Quartus 中执行 `Start Compilation`。成功后应在宿主工程目录看到：

```text
output_files/*.sof
```

运行时可以在宿主机观察容器和输出更新：

```bash
docker ps
find . -type f -mmin -1 | head
```

GUI 无法打开时，检查并重新授权 X11：

```bash
echo "$DISPLAY"
ls -l /tmp/.X11-unix
xhost +SI:localuser:"$USER"
```

如果工程中出现 root 所有权的文件，先确认 wrapper 保留了 `--user`，再按实际路径修复已有文件：

```bash
sudo chown -R "$USER":"$(id -gn)" /path/to/fpga-project
```

本文中的 Docker 只负责 GUI 和编译环境。生成的 `.sof` 位于宿主机，可继续在宿主 Ubuntu 上使用 USB-Blaster 烧录。若要从容器直接烧录，还需另外处理 `/dev/bus/usb`、udev 规则和设备权限。

需要备份兼容镜像时执行：

```bash
docker save quartus13-centos6:fixed -o quartus13-centos6-fixed.tar
```

镜像只包含 CentOS 6 用户态和兼容库；通过 bind mount 提供的 `/opt/altera/13.1` 仍需单独备份。

# 14. 总结

最终结论不是“Quartus II 13.1 必须运行在 Docker 中”，而是先处理 64 位依赖、补丁和传统网卡名，再用实际工程测试：

- Fitter 能正常完成时，直接使用宿主 Ubuntu；
- Fitter 确认卡在旧 `libtbbmalloc.so.2` 分配路径时，才使用 CentOS 6 Docker 兼容用户态。

Docker 方案中，Quartus 本体、工程和编译结果始终保存在宿主机。容器只负责提供更接近 2013 年的运行环境，并通过 `--network host` 使用宿主机已经配置好的 `eth0`。
