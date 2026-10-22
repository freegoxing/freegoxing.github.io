---
title: Quartus II 13.1 在现代 Ubuntu 上的安装、补丁与 Docker 兼容
date: 2026-08-10 10:00:00
tags: [linux, ubuntu, quartus, fpga, docker]
categories: [linux]
description: 在现代 Ubuntu 上以 64 位模式运行 Quartus II 13.1，处理 libpng12、libsys_cpt.so、旧式网卡名和 Fitter 卡死问题，并在必要时使用 CentOS 6 Docker 完成编译。
---

# 先看使用路径

Quartus II 13.1 在现代 Ubuntu 上可能遇到缺少 `libpng12.so.0`、`libsys_cpt.so` 不兼容、旧版许可证工具找不到网卡，以及 Fitter 卡在旧版 `libtbbmalloc.so.2`。本文先处理宿主机运行环境，再用实际工程决定是否需要 Docker：

```text
准备 64 位运行环境并应用补丁
              ↓
核对 Host ID 并测试实际工程
              ├── Fitter 正常完成 → 直接在宿主机使用
              └── Fitter 卡死     → 使用 CentOS 6 Docker 编译
```

补丁只适用于下文 SHA-256 完全匹配的文件；不匹配时停止，不要套用偏移。软件和许可证的使用应遵守适用协议。

# 1. 安装并确认 64 位程序

解压 `Quartus-web-13.1.0.162-linux.tar`，准备安装目录并运行安装器：

```bash
tar -xf Quartus-web-13.1.0.162-linux.tar
cd Quartus-web-13.1.0.162-linux
sudo install -d -o "$USER" -g "$USER" /opt/altera
./setup.sh
```

安装目录选 `/opt/altera/13.1`。安装器可能提示需要 32 位兼容库；如果图形安装器已正常启动，本文的 64 位运行路径无须因此安装 i386 软件包。

## 1.1 确认 64 位程序

`bin/quartus` 是启动脚本，实际的 64 位程序在 `linux64/`。确认其位数：

```bash
QUARTUS_ROOT="/opt/altera/13.1/quartus"
file "$QUARTUS_ROOT/linux64/quartus"
```

输出应包含 `ELF 64-bit`。后续仍从 `bin/quartus` 启动，并传入 `--64bit`。

# 2. 补齐 64 位运行依赖

把旧库放进用户私有目录，避免覆盖系统库：

```bash
mkdir -p ~/.local/lib/quartus-13.1-compat
```

## 2.1 提取 `libpng12.so.0`

从 Debian 归档提取 amd64 版 `libpng12.so.0`，只解包，不把旧软件包安装到系统中：

```bash
cd /tmp
wget http://archive.debian.org/debian/pool/main/libp/libpng/libpng12-0_1.2.50-2+deb8u3_amd64.deb

rm -rf libpng12-amd64
mkdir libpng12-amd64
dpkg-deb -x libpng12-0_1.2.50-2+deb8u3_amd64.deb libpng12-amd64
```

将库文件和符号链接一起复制：

```bash
cp -a \
    /tmp/libpng12-amd64/lib/x86_64-linux-gnu/libpng12.so.0* \
    ~/.local/lib/quartus-13.1-compat/
```

确认位数：

```bash
file ~/.local/lib/quartus-13.1-compat/libpng12.so.0
```

输出应包含 `ELF 64-bit`。若先前放入过同名的 32 位库，确认符号链接现在指向刚提取的 amd64 版本。

## 2.2 检查依赖

检查依赖时，将私有目录与 Quartus 的 `linux64/` 都加入库路径：

```bash
QUARTUS_ROOT="/opt/altera/13.1/quartus"

LD_LIBRARY_PATH="$HOME/.local/lib/quartus-13.1-compat:$QUARTUS_ROOT/linux64" \
ldd "$QUARTUS_ROOT/linux64/quartus" | grep 'not found'
```

没有 `not found` 输出即直接链接依赖已满足。裸 `ldd` 未加载 Quartus 启动脚本配置的环境，可能误报自带库缺失。

## 2.3 启动 GUI

使用官方启动脚本并指定 `--64bit`：

```bash
QUARTUS_ROOT="/opt/altera/13.1/quartus"

LD_LIBRARY_PATH="$HOME/.local/lib/quartus-13.1-compat${LD_LIBRARY_PATH:+:$LD_LIBRARY_PATH}" \
"$QUARTUS_ROOT/bin/quartus" --64bit
```

# 3. 修补 `libsys_cpt.so`

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

# 4. 检查许可证 Host ID

先让 Quartus 自带的 FLEXnet 工具读取宿主网卡：

```bash
/opt/altera/13.1/quartus/linux64/lmutil lmhostid
```

若输出是许可证绑定的 12 位 Host ID，无须改网卡名。若输出 `000000000000`，而物理网卡使用 `eno1`、`enp5s0` 等名称，可能是 Quartus II 13.1 只枚举 `eth0` 到 `eth9` 所致。可在宿主机创建 `/etc/systemd/network/10-quartus-eth0.link`，以实际物理网卡的 MAC 匹配：

```ini
[Match]
MACAddress=40:c2:ba:4c:90:00

[Link]
Name=eth0
```

将示例 MAC 换成实际地址。在本地控制台重启；远程修改网卡名可能断网，还应检查 Netplan 或 NetworkManager 的接口名配置：

```bash
sudo reboot
```

重启后验证：

```bash
ip link show eth0
/opt/altera/13.1/quartus/linux64/lmutil lmhostid
```

许可证 `HOSTID` 使用签发时指定的无冒号 MAC，例如 `40:c2:ba:4c:90:00` 对应 `40c2ba4c9000`。无需修改签名等其他字段。本文将许可证放在 `/opt/altera/13.1/license.dat`，宿主机可设置：

```bash
export LM_LICENSE_FILE=/opt/altera/13.1/license.dat
export ALTERA_LICENSE_FILE=/opt/altera/13.1/license.dat
```

Docker 启动脚本也会传入这两个变量。`--network host` 共享宿主网络命名空间，不会自行重命名网卡。

# 5. 实测 Fitter，再选择运行环境

用实际工程在宿主机编译，替换示例工程名与 revision：

```bash
LD_LIBRARY_PATH="$HOME/.local/lib/quartus-13.1-compat${LD_LIBRARY_PATH:+:$LD_LIBRARY_PATH}" \
/opt/altera/13.1/quartus/bin/quartus_fit --64bit exp1 -c exp1
```

本次实践中，Fitter 日志停在：

```text
Info (171003): Fitter is performing an Auto Fit compilation, which may decrease Fitter effort to reduce compilation time
Warning (15714): Some pins have incomplete I/O assignments. Refer to the I/O Assignment Warnings report for details
```

跟踪显示程序进入自带的 `libtbbmalloc.so.2` 分配路径后不再推进。上面的 I/O assignment warning 不是卡死原因。

不要仅凭短时间没有日志判断卡死，同时观察进程和工程文件：

```bash
ps aux | grep '[q]uartus_fit'
top
find . -type f -mmin -1 | head
```

若 Fitter 完成编译，停在本节即可；稳定复现卡死时再使用第 6 节的 Docker 环境。

## 5.1 宿主机启动脚本

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

启动时执行：

```bash
~/.local/bin/quartus-13.1
```

以下 Docker 部分只针对宿主机 Fitter 稳定卡死的情况。

# 6. 配置 CentOS 6 容器

Docker 只提供 CentOS 6 用户态，Quartus 安装目录与工程仍在宿主机。只有第 5 节稳定复现 Fitter 卡死时才执行以下步骤。

## 6.1 创建配置容器

```bash
docker pull centos:6

docker run -it \
  --name quartus13-centos6 \
  --network host \
  -v /opt/altera/13.1:/opt/altera/13.1:ro \
  centos:6 \
  /bin/bash
```

配置阶段不加 `--rm`。进入容器后可确认系统版本与网卡：

```bash
cat /etc/centos-release
ip link show eth0
```

`--network host` 让容器共享宿主网络命名空间；若第 4 节配置了 `eth0`，这里应能看到它。

## 6.2 安装运行依赖

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

## 6.3 固化镜像

配置完成后退出并保存镜像：

```bash
exit
docker commit quartus13-centos6 quartus13-centos6:fixed
docker rm quartus13-centos6
```

`docker rm` 删除的是配置容器，已保存的镜像仍在。

# 7. 创建 Docker 启动脚本

创建 `/usr/local/bin/quartus-docker`：

```bash
sudo editor /usr/local/bin/quartus-docker
```

写入：

```bash
#!/usr/bin/env bash
set -e

QUARTUS_ROOT="/opt/altera/13.1/quartus"
IMAGE="quartus13-centos6:fixed"

USB_GROUP="$(getent group plugdev | cut -d: -f3)"
USB_GROUP_ARGS=()
if [[ -n "$USB_GROUP" ]]; then
    USB_GROUP_ARGS=(--group-add "$USB_GROUP")
fi

xhost +SI:localuser:"$USER" >/dev/null

exec docker run --rm -it \
    --user "$(id -u):$(id -g)" \
    "${USB_GROUP_ARGS[@]}" \
    --network host \
    -v /dev/bus/usb:/dev/bus/usb:rw \
    --device-cgroup-rule='c 189:* rmw' \
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
sudo chmod +x /usr/local/bin/quartus-docker
cd /path/to/fpga-project
quartus-docker
```

`--user` 保持工程文件归属；`--network host` 保持宿主网卡视图；`/opt/altera/13.1` 只读，`$HOME` 可写。工程应位于 `$HOME` 下，否则还需单独挂载工程目录。`/dev/bus/usb` 和 [Docker 设备规则](https://docs.docker.com/reference/cli/docker/container/run/#using-dynamically-created-devices---device-cgroup-rule)提供 USB 节点访问；如果节点属于 `plugdev` 且权限为 `0660`，脚本会把该组的数字 GID 加入容器。`c 189:* rmw` 中的 `*` 不需要转义。

# 8. 可选：在容器内使用 USB-Blaster

容器版 Programmer 若已能正常烧录，跳过本节。宿主机现有的设备权限通常已经足够；只有遇到节点权限错误时才调整 udev。

## 8.1 设备权限不足时配置 udev

先用 `lsusb` 找到设备的 Bus 和 Device 编号，再查看对应节点的权限。例如 Bus `001`、Device `004`：

```bash
lsusb -d 09fb:
stat -c '%a %G %n' /dev/bus/usb/001/004
```

若节点确实不允许当前 UID 或其附加组读写，可按[传统 USB-Blaster 下载线文档](https://www.intel.com/content/dam/support/us/en/programmable/support-resources/bulk-container/pdfs/literature/ug/archives/ug-usb-blstr-15.1.pdf)中的 VID `09fb`、PID `6001`、`6002`、`6003` 建规则；不同设备需替换为实际 VID/PID。

确保 `plugdev` 组存在，并在宿主机创建 `/etc/udev/rules.d/51-usbblaster.rules`，只为匹配的设备节点授予该组读写权限：

```bash
getent group plugdev || sudo groupadd plugdev
```

```udev
SUBSYSTEM=="usb", ENV{DEVTYPE}=="usb_device", ATTR{idVendor}=="09fb", ATTR{idProduct}=="6001", MODE="0660", GROUP="plugdev"
SUBSYSTEM=="usb", ENV{DEVTYPE}=="usb_device", ATTR{idVendor}=="09fb", ATTR{idProduct}=="6002", MODE="0660", GROUP="plugdev"
SUBSYSTEM=="usb", ENV{DEVTYPE}=="usb_device", ATTR{idVendor}=="09fb", ATTR{idProduct}=="6003", MODE="0660", GROUP="plugdev"
```

保存后重新加载规则并拔插 USB-Blaster。如果还要在宿主机以普通用户直接烧录，将该用户加入 `plugdev`，然后重新登录：

```bash
sudo udevadm control --reload-rules
sudo usermod -aG plugdev "$USER"
```

重新运行 `lsusb` 和 `stat`，确认新节点权限为 `660`、组为 `plugdev`。udev 规则只需配置在宿主机。

# 9. 验证与排错

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

生成的 `.sof` 位于宿主机，也可在宿主机烧录。若在容器内烧录，打开 **Tools → Programmer → Hardware Setup** 确认 USB-Blaster 可用；权限错误参见第 8 节。

需要备份兼容镜像时执行：

```bash
docker save quartus13-centos6:fixed -o quartus13-centos6-fixed.tar
```

镜像只包含 CentOS 6 用户态和兼容库；通过 bind mount 提供的 `/opt/altera/13.1` 仍需单独备份。
