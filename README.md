<!-- SPDX-License-Identifier: GPL-2.0-only -->

# Gentoo WSL2 Kernel

为 **Gentoo on WSL2** 维护的精简 Linux 内核配置及补丁集合。仓库不包含完整内核源码或预编译内核；使用 Gentoo 的 `sys-kernel/gentoo-sources` 获取源码，再应用这里维护的配置和补丁。

当前已验证版本：**gentoo-sources 7.2.10（x86_64）**。

## 设计目标

- 面向 WSL2 命令行开发环境，不以 WSLg、GPU 直通或物理硬件支持为目标。
- 内建 Hyper-V、VirtIO PCI、VirtIOFS、9P、EXT4、OverlayFS 等 WSL 所需功能。
- 保留 Docker Engine / Compose 所需的 namespace、cgroup v2、Bridge、VETH、NAT、Seccomp 等功能。
- 关闭内核模块支持（`CONFIG_MODULES=n`），优先使用内建驱动。
- 移植微软 WSL 内核的 **Hyper-V PCI / SWIOTLB 专用 DMA 内存池**支持，以适配 WSL 宿主的 VirtIO 性能优化。

这是面向特定环境的个人配置，不是通用发行版内核，也不是安全加固内核。部分安全缓解和非必需驱动已被关闭，请自行评估风险。

## 仓库结构

```text
.
├── README.md
├── LICENSE
├── configs/
│   └── gentoo-7.2.10.config
└── patches/
    └── gentoo-sources-7.2.10/
        └── 0001-wsl-hv-pci-swiotlb.patch
```

`configs/` 存放内核的完整 `.config`；`patches/` 存放对应源码版本的补丁。**补丁版本必须与源码版本匹配**，不能假定旧补丁可无修改地用于新内核。

> 注意：上面的 `configs/` 和 `patches/` 是预期仓库布局。请先将本地已经验证的配置和补丁放入相应位置，再按以下步骤构建。

## 构建

以下命令在 Gentoo WSL2 内执行，假设仓库位于 `~/.config/kernel`。

### 1. 获取配置与补丁

```bash
REPO="$HOME/.config/kernel"
# 首次使用时：git clone <你的仓库地址> "$REPO"
```

推荐通过 Gentoo 的 [Portage User Patches](https://wiki.gentoo.org/wiki//etc/portage/patches) 自动应用源码补丁。**先安装补丁，再安装对应版本的源码**：

```bash
sudo install -Dm644 \
  "$REPO/patches/gentoo-sources-7.2.10/0001-wsl-hv-pci-swiotlb.patch" \
  /etc/portage/patches/sys-kernel/gentoo-sources-7.2.10/0001-wsl-hv-pci-swiotlb.patch

sudo emerge --oneshot '=sys-kernel/gentoo-sources-7.2.10'
```

如果源码已通过手工方式修改过，**不要对同一份源码重复应用补丁**。重新安装源码前，请备份现有源码树、配置文件和可启动的旧 `bzImage`。

### 2. 配置内核

```bash
SRC=/usr/src/linux-7.2.10-gentoo
BUILD="$HOME/kernel-build/gentoo-7.2.10"
REPO="$HOME/.config/kernel"

mkdir -p "$BUILD"
cp "$REPO/configs/gentoo-7.2.10.config" "$BUILD/.config"
make -C "$SRC" O="$BUILD" olddefconfig
```

可选：使用 `make -C "$SRC" O="$BUILD" nconfig` 调整配置。升级内核版本后，应核对 `olddefconfig` 引起的差异。

### 3. 编译

```bash
make -C "$SRC" O="$BUILD" -j4 bzImage
make -s -C "$SRC" O="$BUILD" kernelrelease
ls -lh "$BUILD/arch/x86/boot/bzImage"
```

将 `bzImage` 复制到 Windows 文件系统中（下例中的 Windows 用户名仅为占位符）：

```bash
cp "$BUILD/arch/x86/boot/bzImage" \
  '/mnt/c/Users/<WindowsUser>/.wsl/bzImage-gentoo-7.2.10'
```

## 启用 WSL2 自定义内核

在 Windows 的 `%USERPROFILE%\.wslconfig` 文件中，修改现有 `[wsl2]` 段落：

```ini
[wsl2]
kernel=C:\\Users\\<WindowsUser>\\.wsl\\bzImage-gentoo-7.2.10
loadDefaultKernelModules=false
guiApplications=false
# 首次启动排障可开启：
debugConsole=true
```

该文件属于 Windows 用户配置，**不要**与 Gentoo 内部的 `/etc/wsl.conf` 混淆。

在 Windows PowerShell 中执行：

```powershell
wsl --shutdown
wsl -d Gentoo
```

发行版名称如有不同，请使用 `wsl --list --verbose` 查看。

## 验证

进入 Gentoo 后：

```bash
uname -r
findmnt /
findmnt /mnt/c
cat /sys/fs/cgroup/cgroup.controllers
getent hosts github.com
```

验证 SWIOTLB 优化：

```bash
dmesg | grep -Ei 'swiotlb|hv_pci'
cat /sys/bus/vmbus/drivers/hv_pci/swiotlb_base
cat /sys/bus/vmbus/drivers/hv_pci/swiotlb_size
```

在已测试的 WSL 宿主环境中，内核启动参数包含 `swiotlb=force` 和 `hv_pci_swiotlb=67108864`；驱动成功预留 **64 MiB** 内存池，sysfs 报告的大小为 `67108864` 字节，并可正常挂载 `/mnt/c` 的 VirtIOFS。

这些测试证明了**内核可启动、内存池已预留、sysfs 已导出且 VirtIOFS 可挂载**；并不构成所有 DMA 路径、长期负载稳定性或实际性能提升的完整验证。

可进一步按需测试 Docker：

```bash
docker info
docker run --rm hello-world
```

## 回滚

务必保留一个已验证可启动的旧版 `bzImage`。如果新内核无法启动，在 Windows 中将 `.wslconfig` 的 `kernel=` 改回旧文件，或者临时注释该项以恢复 WSL 默认内核，然后运行：

```powershell
wsl --shutdown
```

## 补丁来源与版权

SWIOTLB 补丁参考并适配了 [Microsoft WSL2-Linux-Kernel](https://github.com/microsoft/WSL2-Linux-Kernel) 中的 Hyper-V PCI / SWIOTLB 实现，主要参考 `linux-msft-wsl-6.18.54.1`；其目标源码为 **Gentoo Linux 7.2.10**，并非可直接套用的微软原始提交。

- [微软 Hyper-V PCI 驱动](https://github.com/microsoft/WSL2-Linux-Kernel/blob/linux-msft-wsl-6.18.54.1/drivers/pci/controller/pci-hyperv.c)
- [微软 SWIOTLB 实现](https://github.com/microsoft/WSL2-Linux-Kernel/blob/linux-msft-wsl-6.18.54.1/kernel/dma/swiotlb.c)
- [Linux Kernel COPYING](https://github.com/microsoft/WSL2-Linux-Kernel/blob/linux-msft-wsl-6.18.54.1/COPYING)

本仓库所包含的内核源码改动遵循 **GNU GPL version 2 only（SPDX: `GPL-2.0-only`）**，参见 [`LICENSE`](./LICENSE)。保留补丁涉及的上游版权声明和原文件 SPDX 标识；该仓库不代表 Microsoft、Linux 内核项目或 Gentoo 的官方支持。
