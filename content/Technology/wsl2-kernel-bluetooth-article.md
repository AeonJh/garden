# WSL2 自定义内核编译与蓝牙直通完整指南

> **环境**: Ubuntu 22.04.5 LTS · WSL2 · Linux 6.6.123.2-microsoft-standard-WSL2+ · usbipd-win

---

## 背景

WSL2 的默认内核是微软预编译的通用镜像，无法支持所有硬件场景。当需要以下功能时，必须自行编译内核：

- 通过 `usbipd-win` 将 Windows 物理 USB 设备（如蓝牙适配器）直通到 WSL2
- 启用特定驱动（如 MediaTek 蓝牙、特定网卡驱动）
- 调整内核参数以适配特殊工作负载

本文记录从零构建 WSL2 自定义内核，并解决过程中遇到的两个核心问题：**内核模块全部丢失**和**蓝牙设备无法初始化**。

---

## 一、构建环境准备

```bash
# 安装编译依赖
sudo apt update && sudo apt install -y \
    build-essential flex bison libssl-dev libelf-dev \
    bc python3 pahole dwarves

# 克隆 WSL2 官方内核仓库
git clone --depth=1 https://github.com/microsoft/WSL2-Linux-Kernel.git \
    ~/projects/WSL2-Linux-Kernel
cd ~/projects/WSL2-Linux-Kernel
```

---

## 二、标准编译流程

WSL2 内核仓库在 `Microsoft/config-wsl` 提供了官方维护的 Kconfig，其中约 939 个驱动配置为可加载模块（`=m`），适合直通场景。

```bash
# Step 1: 编译内核镜像（bzImage）
make KCONFIG_CONFIG=Microsoft/config-wsl -j$(nproc)

# Step 2: 安装内核模块到临时目录
make KCONFIG_CONFIG=Microsoft/config-wsl \
     INSTALL_MOD_PATH="$PWD/modules" \
     modules_install

# Step 3: 打包为 VHDX（WSL2 模块虚拟磁盘格式）
sudo ./Microsoft/scripts/gen_modules_vhdx.sh \
     "$PWD/modules" \
     "$(make -s KCONFIG_CONFIG=Microsoft/config-wsl kernelrelease)" \
     modules.vhdx

# Step 4: 复制到 Windows 文件系统
cp arch/x86/boot/bzImage /mnt/c/WSL2Kernel/bzImage
cp modules.vhdx /mnt/c/WSL2Kernel/modules.vhdx
```

在 `%USERPROFILE%/.wslconfig` 中配置：

```ini
[wsl2]
kernel=C:\\WSL2Kernel\\bzImage
kernelModules=C:\\WSL2Kernel\\modules.vhdx
```

然后 `wsl --shutdown` 重启。

---

## 三、问题一：内核模块全部丢失（kernel/ 目录不存在）

### 症状

启动后执行 `lsmod` 为空，`modprobe` 任意模块均失败。检查模块目录：

```
/lib/modules/6.6.123.2-microsoft-standard-WSL2+/
├── build -> /home/.../WSL2-Linux-Kernel
├── modules.alias        # 45 bytes（几乎为空）
├── modules.dep          # 0 bytes  ← 关键告警
├── modules.order        # 0 bytes  ← 关键告警
└── modules.builtin      # 15 KB（正常）
```

`kernel/` 目录完全不存在，无任何 `.ko` 文件。

### 根本原因分析

WSL2 以 overlay 文件系统挂载 VHDX：

```
none on /usr/lib/modules/6.6.123.2-microsoft-standard-WSL2+
    type overlay (lowerdir=/modules, upperdir=...rw/upper, ...)
```

`/modules` 是 VHDX 内容的挂载点（lowerdir，只读）。VHDX 本身缺少 `kernel/` 树，因此直接原因是 **VHDX 打包时没有 `.ko` 文件**。

进一步溯源，在构建树根目录发现两个 config 文件：

| 文件 | `=m` 数量 | `CONFIG_KVM=` |
|------|-----------|---------------|
| `Microsoft/config-wsl` | **939** | `m` |
| `config`（构建根目录） | **0** | `y` |

内核实际是用 `config` 文件构建的。该文件将所有驱动编译为内建（`=y`），导致构建过程中没有生成任何 `.ko` 文件。随后 `modules_install` 无内容可安装，VHDX 自然为空。

### 为何发生配置错位

`KCONFIG_CONFIG=Microsoft/config-wsl` 是传递给 `make` 的**变量**，而非将文件复制为 `.config`。当 make 在某次历史构建中以其他方式初始化后，构建根目录残留了 `config` 文件（内容为全 `=y`）。此后的增量构建复用了该文件，使 `Microsoft/config-wsl` 实际被绕过。

```
# 验证方式
find . -name "*.ko" | wc -l
# 输出: 0  → 确认构建树中无任何模块文件
wc -c modules.order
# 输出: 0  → make 未注册任何外部模块
```

### 修复方案

**必须清理旧构建产物后再重新构建**，`make mrproper` 删除所有 `.o`、`built-in.a` 及配置缓存，确保 `Microsoft/config-wsl` 完整生效：

```bash
# 清理所有构建产物（包括错误的 config 文件）
make mrproper

# 全量重新构建
make KCONFIG_CONFIG=Microsoft/config-wsl -j$(nproc)

# 验证 .ko 文件已生成
find . -name "*.ko" | wc -l
# 应输出几百个

# 重新安装模块并生成 VHDX
rm -rf "$PWD/modules" modules.vhdx
make KCONFIG_CONFIG=Microsoft/config-wsl \
     INSTALL_MOD_PATH="$PWD/modules" \
     modules_install
sudo ./Microsoft/scripts/gen_modules_vhdx.sh \
     "$PWD/modules" \
     "$(make -s KCONFIG_CONFIG=Microsoft/config-wsl kernelrelease)" \
     modules.vhdx
```

---

## 四、问题二：蓝牙设备直通后无法初始化

### 症状

通过 `usbipd-win` 将 Windows 蓝牙适配器（Foxconn `0489:e0cd`，MediaTek MT7922 芯片）绑定到 WSL2 后：

```bash
$ hciconfig -a
hci0:   Type: Primary  Bus: USB
        BD Address: 00:00:00:00:00:00  ACL MTU: 0:0  SCO MTU: 0:0
        DOWN
        RX bytes:0 acl:0 sco:0 events:0 errors:0
```

`dmesg` 中可见：

```
[  58.004300] usbcore: registered new interface driver btusb
[  60.008200] Bluetooth: hci0: Opcode 0x0c03 failed: -110
[  60.010373] vhci_hcd: unlink->seqnum 33
[  60.010377] vhci_hcd: urb->status -104
```

- `Opcode 0x0c03` = `HCI_Reset` 命令（OGF=0x03, OCF=0x03）
- `-110` = `ETIMEDOUT`：HCI Reset 超时

### 根本原因分析

`btusb` 驱动已正常加载，但 MediaTek 蓝牙芯片需要在标准 HCI 握手前执行**厂商专有初始化序列**（上传 RAM 固件补丁），这部分逻辑由 `CONFIG_BT_HCIBTUSB_MTK` 控制。

检查内核配置：

```
# CONFIG_BT_HCIBTUSB_MTK is not set   ← 关键
```

该选项被禁用后，`btusb` 将 MT7922 视为标准 HCI 设备直接发送 `HCI_Reset`，而芯片尚未完成固件加载，无法响应，导致超时。

所需固件文件已存在：

```bash
$ ls /lib/firmware/mediatek/ | grep MT7922
BT_RAM_CODE_MT7922_1_1_hdr.bin   ← 固件已就位
WIFI_RAM_CODE_MT7922_1.bin
```

缺的是内核侧的驱动逻辑，而非固件文件本身。

### 修复方案

在 `Microsoft/config-wsl` 中启用 MediaTek 支持后重新编译：

```bash
# 启用 MediaTek btusb 支持
sed -i 's/# CONFIG_BT_HCIBTUSB_MTK is not set/CONFIG_BT_HCIBTUSB_MTK=y/' \
    Microsoft/config-wsl

# 增量编译（只重编修改涉及的模块）
make KCONFIG_CONFIG=Microsoft/config-wsl -j$(nproc)

# 重新安装模块 + 生成 VHDX
rm -rf "$PWD/modules" modules.vhdx
make KCONFIG_CONFIG=Microsoft/config-wsl \
     INSTALL_MOD_PATH="$PWD/modules" \
     modules_install
sudo ./Microsoft/scripts/gen_modules_vhdx.sh \
     "$PWD/modules" \
     "$(make -s KCONFIG_CONFIG=Microsoft/config-wsl kernelrelease)" \
     modules.vhdx

cp arch/x86/boot/bzImage /mnt/c/WSL2Kernel/bzImage
cp modules.vhdx /mnt/c/WSL2Kernel/modules.vhdx
```

重启 WSL2，重新 attach 蓝牙设备：

```powershell
# Windows 侧
usbipd attach --wsl --busid <busid>
```

```bash
# WSL2 侧验证
hciconfig -a
# BD Address 应显示真实 MAC 地址，状态 UP RUNNING

bluetoothctl
[bluetooth]# scan on
# 应能扫描到周边设备
```

---

## 五、经验总结

### 关键结论

| # | 教训 | 机制 |
|---|------|------|
| 1 | **每次 `make` 均须携带 `KCONFIG_CONFIG=`** | make 变量不持久化；构建树存在历史 `config` 文件时会被优先使用 |
| 2 | **配置变更后必须 `make mrproper`** | 增量构建不会重编已存在的 `.o` 文件，配置错位的产物会被复用 |
| 3 | **厂商专有芯片需单独开启驱动选项** | 通用 `=m` 不足，MediaTek/Intel/Realtek 等各有独立 `CONFIG_BT_HCIBTUSB_*` 开关 |
| 4 | **VHDX 内容由 `make modules_install` 决定** | `gen_modules_vhdx.sh` 只是打包工具，源头没有 `.ko` 则 VHDX 必然为空 |

### WSL2 内核模块挂载原理

```
VHDX 文件
  └─ /modules (WSL 内部挂载点, 只读)
       └─ lowerdir (overlay)
            └─ /usr/lib/modules/<kver>/   ← 最终可见路径
                 upperdir (可写层, 用于 dkms 等运行时安装)
```

`/lib/modules/<kver>` 通常是 `/usr/lib/modules/<kver>` 的符号链接，两者指向同一 overlay。

### 推荐的完整构建脚本

```bash
#!/usr/bin/env bash
set -euo pipefail

KERNEL_DIR="$HOME/projects/WSL2-Linux-Kernel"
WINDOWS_OUT="/mnt/c/WSL2Kernel"
KCONFIG="Microsoft/config-wsl"

cd "$KERNEL_DIR"

echo "[1/5] Clean"
make mrproper

echo "[2/5] Build"
make KCONFIG_CONFIG="$KCONFIG" -j"$(nproc)"

echo "[3/5] Install modules"
rm -rf "$KERNEL_DIR/modules"
make KCONFIG_CONFIG="$KCONFIG" \
     INSTALL_MOD_PATH="$KERNEL_DIR/modules" \
     modules_install

KVER="$(make -s KCONFIG_CONFIG="$KCONFIG" kernelrelease)"
echo "[4/5] Generate VHDX for $KVER"
rm -f modules.vhdx
sudo ./Microsoft/scripts/gen_modules_vhdx.sh \
     "$KERNEL_DIR/modules" "$KVER" modules.vhdx

echo "[5/5] Copy to Windows"
mkdir -p "$WINDOWS_OUT"
cp arch/x86/boot/bzImage "$WINDOWS_OUT/bzImage"
cp modules.vhdx "$WINDOWS_OUT/modules.vhdx"

echo "Done. Run 'wsl --shutdown' to apply."
```

---

## 六、参考资料

### 官方文档与仓库

- [microsoft/WSL2-Linux-Kernel](https://github.com/microsoft/WSL2-Linux-Kernel) — WSL2 官方内核源码，含 `Microsoft/config-wsl` 与 `gen_modules_vhdx.sh`
- [dorssel/usbipd-win](https://github.com/dorssel/usbipd-win) — Windows 侧 USB/IP 服务，支持将 USB 设备直通到 WSL2
- [Microsoft Learn: WSL 中的高级设置配置](https://learn.microsoft.com/zh-cn/windows/wsl/wsl-config) — `.wslconfig` 各字段说明，含 `kernel=`、`kernelModules=` 配置项
- [Gentoo Wiki: Bluetooth](https://wiki.gentoo.org/wiki/Bluetooth/zh-cn) — 蓝牙相关配置与调试指南
- [usbipd-win 讨论](https://github.com/dorssel/usbipd-win/discussions/310#discussioncomment-9712477) — 无法应用 firmware 的解决方式

### Linux 内核文档

- [Linux Kernel Kconfig Language](https://www.kernel.org/doc/html/latest/kbuild/kconfig-language.html) — `=y` / `=m` / `=n` 语义与 `KCONFIG_CONFIG` 变量说明
- [Linux Kernel Makefiles](https://www.kernel.org/doc/html/latest/kbuild/makefiles.html) — `make mrproper` / `modules_install` / `INSTALL_MOD_PATH` 等目标文档
- [Linux Kernel Source: `drivers/bluetooth/btusb.c`](https://elixir.bootlin.com/linux/v6.6/source/drivers/bluetooth/btusb.c) — `btusb` 驱动实现，含 `CONFIG_BT_HCIBTUSB_MTK` 条件分支
- [Linux Kernel Source: `drivers/bluetooth/btmtk.c`](https://elixir.bootlin.com/linux/v6.6/source/drivers/bluetooth/btmtk.c) — MediaTek 蓝牙初始化序列实现（固件上传、HCI 握手）

### HCI 协议规范

- [Bluetooth Core Specification 5.4 — Vol 4, Part E: HCI](https://www.bluetooth.com/specifications/specs/core54-html/) — HCI 命令格式定义，Opcode `0x0c03` 对应 `HCI_Reset`（OGF=0x03, OCF=0x03）
- [Linux `errno` 错误码](https://www.kernel.org/doc/html/latest/userspace-api/audit/index.html) — `-110` = `ETIMEDOUT`

### 相关工具

- [linux-firmware](https://git.kernel.org/pub/scm/linux/kernel/git/firmware/linux-firmware.git) — 内核固件仓库，含 `mediatek/BT_RAM_CODE_MT7922_1_1_hdr.bin` 等蓝牙固件文件
- [bluez/hciconfig](http://www.bluez.org/) — BlueZ 蓝牙协议栈，`hciconfig -a` 用于查看 HCI 设备状态

---

*构建日期：2026-04-09 · 内核版本：6.6.123.2-microsoft-standard-WSL2+*
