# 为搭载 ARM SoC 的设备移植 openFyde

## 目标

本指南描述了我为搭载 ARM SoC 的 SBC 移植 openFyde 的步骤。在 ARM 平台上进行移植比在 x86 机器上要复杂得多；这个过程也很大程度上取决于各个设备的设计和配置。

本指南的读者应具备 Linux kernel 调试经验和 ARM 架构及嵌入式系统 [devicetree](https://en.wikipedia.org/wiki/Devicetree) 的基本概念，以及对 openFyde / Chromium OS build systems 的基本熟悉。

请注意，本指南主要关注如何在**特定**的 SBC 上**启动** openFyde。对于类似的硬件设备，你可能会遇到这里未包含的各种问题（声音、DP 输出、摄像头等）。

在本指南中，我将记录为搭载 [RK3399 SoC](https://www.rock-chips.com/a/en/products/RK33_Series/2016/0419/758.html) 的 Radxa [Rock 4C Plus](https://rockpi.org/rockpi4) 进行移植的过程。

## 前提条件

需要注意几点：

* 首先，确保你的 ARM SBC（以下简称"开发板"）能够至少启动一个 Linux distro，并且工作正常。

* **关于 mali 的注意事项**：openFyde 和 Chromium OS 对 mali drivers 有严格的要求。你**必须**拥有带有适当 mali driver 的 Linux kernel（你的开发板上正常工作的 Linux distro 的 kernel）源代码。如果你没有，请咨询开发板厂商以获取。

* 你还需要能够与开发板的 tty console 进行交互。

* 你必须至少在其他（更通用的）平台/架构上成功构建过一次 openFyde，并且熟悉技术术语如 "overlay"、"use flag"、"make.conf"、"Makefiles" 等。如果不熟悉，[Getting started](https://github.com/openFyde/getting-started) 是你应该开始的地方。

## 步骤

### 创建 board overlay

创建你的 overlay 文件夹是第一步。首先检查是否有公开可用的与你的开发板类似的 Chromebook board 会很有帮助。对于 RockChip RK3399 芯片组（又称 OP1），你可以复制或继承现有的 board overlays（在这个例子中是 "kevin"）。

如果现有的公开 board overlays 都不适用，尝试在同一架构中上升一级寻找 board overlay。例如 "arm64"。

### 寻找可重用的 Linux kernel

openFyde 是 Chromium OS 的下游分支；它使用来自上游 kernels 的几个 kernel 版本，这些版本包含了 Chromium 团队移植的 commits。这些移植包括 Chromium OS 所需的功能，如安全增强、对 overlayfs 和 cgroups 的支持等。没有这些功能，openFyde 就无法成为一个功能完整的操作系统。

在实际移植过程之前，让我们为你的开发板选择 openFyde 的 kernel。这里你需要先检查当前的 Linux kernel 版本。你当前的 kernel 版本越新，需要移植的 commits 就可能越少。当然，我们可以重用 Chromium OS 源码树中已经提供的 kernels：

```
$ ls src/third_party/kernel/
upstream  v4.14  v4.19  v4.4  v5.10  v5.10-arcvm  v5.10-manatee  v5.15  v5.4  v5.4-arcvm  v5.4-manatee
```
除 `upstream` 外，上述所有 kernels 都是 LTS 版本。对于 RK3399 SOC，kernel 版本 v4.4 由 RockChip 维护。FydeOS 团队有这个 kernel [repo](https://github.com/FydeOS-for-You-overlays/kernel-rockchip)，其中包含了针对 Chromium OS / openFyde 的一些调整。

- 对于类似 Raspberry Pi 的设备，推荐使用 kernel v5.4；这里是我们一直用于 openFyde 和 Chromium OS 的 kernel [链接](https://github.com/FydeOS/kernel-raspberry_pi)。

- 对于 RK3588 SoC，首选这个 [v5.10 分支](https://github.com/radxa/kernel/tree/stable-5.10-rock5)。此外，openFyde 基于它提供了一些 [patches](https://github.com/openFyde/foundation-rk3588/tree/main/chipset-rk3588/sys-kernel/chromeos-kernel-5_10)。

如果你的开发板的 SoC 不属于上述任何组，你的选择是：

- 1) 使用 Chromium 团队在你的开发板上维护的 kernel，这样你可以通过避免处理复杂的安全和 cgroup 代码来节省时间和潜在的脑细胞损失。

- 2) 将 Chromium OS 需要的代码移植到你开发板的出厂 kernel。它可以重用你开发板的大部分组件，但你可能会陷入一个让 openFyde 启动的打地鼠游戏。

如果你的出厂 Linux kernel 版本 >= 5.10，我们建议你选择方法二；否则，方法一可能是更好的选择。

### U-Boot

与 ARM 领域的其他 Linux distributions 一样，openFyde 需要一个合适的功能性 U-Boot binary。由于你已经有一个可启动的 Linux OS，你可以从存储驱动器导出 U-Boot 和 preloader binaries。

对于 RK3399 SoC，磁盘布局在[这里](https://opensource.rock-chips.com/wiki_Partitions)有文档说明。我们制作了一个简单的[脚本](https://github.com/FydeOS-for-You-overlays/uboot-bin-for-pinebookpro/blob/rock-pi4/export_images.sh)来提取它。

导出 binary image 后，你需要将它们放入 ebuild 文件夹 `files` 中，并修改 [U-Boot ebuild](https://github.com/openFyde/foundation-rk3399/tree/main/baseboard-rockpi4/sys-boot/rockchip-uboot)。

### TTY console

下一步是解决 TTY console 的问题。你必须首先从 `dmesg` 输出中观察你的开发板的 tty port。openFyde 中有一个名为 [tty](https://github.com/openFyde/foundation-rk3399/tree/main/baseboard-rockpi4/chromeos-base/tty) 的包；在你的开发板的 "overlay" 中复制并修改它。不要忘记在你的 `make.conf` 中添加 `TTY_CONSOLE=$TTY_PORT` 的 "use flag"。

tty 的 kernel parameters 也应该调整；详情请参见这个 [ebuild](https://github.com/openFyde/foundation-rk3399/blob/main/chipset-rk3399-openfyde/sys-kernel/rockchip-kernel/rockchip-kernel-4.4.190.ebuild)。

dts 文件的优先级高于 kernel parameters，所以请检查 dts 和 dtsi 文件中的参数。

### 调整 dts 文件

这有点偏离主题，但值得一提。这一步通常涉及复制 dts 和 dtsi 文件，以仔细检查是否满足所有前提条件。你应该关注 dts 文件中的开关、兼容的 kernel drivers 和 kernel configs。

假设缺少任何 device drivers；你必须从上游手动挑选它们或从你当前工作的 kernel 中复制相关目录，将它们添加到上层 Makefile 中，然后解决可能出现的任何编译错误。

在移植 drivers 时要小心；kernel panics 可能会非常令人烦恼。

### Kernel config

由于 Chromium OS 有额外的 kernel 功能，有额外的 kernel configs 来启用它们。在一个[典型的 Chromebook board](https://www.chromium.org/chromium-os/how-tos-and-troubleshooting/kernel-configuration/) 中有三个 kernel config 级别。

openFyde 中的所有 board overlays 都使用在 make.config 中定义的一层 kernel config：

```
CHROMEOS_KERNEL_CONFIG="/mnt/host/source/src/overlays/overlay-rock4cp-openfyde/kconfigs/fydeos-r96-4_4-def"
```

如果你想为 openFyde 添加外部 kernel configs，`src/third_party/kernel/$kver/chromeos/config/base.config` 包含基本配置选项。对于 arm64 boards，`src/third_party/kernel/$kver/chromeos/config/arm64/common.config` 有通用选项，这是一个很好的起点。

注意：简单地将这些选项附加到你的 kernel config 文件可能不起作用；必须启用一些依赖选项；运行 `make menuconfig ARCH=$arch` 来查找。

值得一提的是，一些 SBCs 的电源供应似乎较弱；将非必要的 drivers（mali、wifi、ethernet NIC 等）编译为模块会是理想的选择。

### Frecon

如果以上步骤一切顺利，你应该能够成功构建一个镜像。如果你尝试启动它，kernel 应该会给你输出信息。如果 kernel 崩溃，不要惊慌。检查输出中的堆栈跟踪，我们就可以开始调试过程。

如果你无法通过按键输入任何内容，在 kernel 消息中 grep 关键词 'frecon' 和 'drm'，看看发生了什么。"Frecon"（FREon CONsole）是 Chromium OS graphics stack "freon" 的控制台。要了解更多信息，请查看 [docs](https://chromium.googlesource.com/chromiumos/platform/frecon/+/HEAD/DESIGN-DOC.md) 页面。

frcon 无法启动的最可能原因是 drm driver，所以要特别注意与 drm 相关的错误。

### Mali

虽然 ARM 有其开源的 kernel 级 [mali driver](https://developer.arm.com/downloads/-/mali-drivers)，但用户空间 drivers 通常是二进制的黑盒。

如果在你的 dmesg 中出现以下输出段，kernel mali driver 已成功初始化。否则，你必须检查你的 kernel mali driver 错误并解决所有问题。

```
(device)
[   53.109371] mali ff9a0000.gpu: Looking up mali-supply from device tree
[   53.110191] mali ff9a0000.gpu: GPU identified as 0x0860 r2p0 status 0
[   53.110961] mali ff9a0000.gpu: Protected mode not available
[   53.111980] mali ff9a0000.gpu: Using configured power model mali-simple-power-model, and fallback mali-simple-power-model
[   53.118161] mali ff9a0000.gpu: Probed as mali0
```
如果 openFyde 的 GUI 仍然没有显示，检查 `/var/log/ui/ui.LATEST` 和 `/var/log/chrome/chrome` 是否有明显的错误和提示。UI 进程退出的一个可能原因是 kernel 和用户空间之间的 mali 版本不匹配，所以检查 Chromium OS 中对应的 mali 版本，并相应地在你的 kernel 中进行更改。

### 无法创建用户

在你成功显示 GUI 并发现自己处于 OOBE 过程后，如果你无法创建用户（尝试登录桌面但失败），检查 dmesg 的最后部分。假设有关于 `cryptohome` 或 `recognized mount option` 的错误。在这种情况下，Chromium OS 的 kernel 安全部分工作不正确，包括 selinux 和其他相关组件。现在最好检查相应的 kernel configs 和移植的 commits。

### 音频

一些 Linux distributions 使用 "pulseaudio" 作为前端服务器，alsa 作为后端服务器来播放音乐。openFyde / Chromium OS 有自己的前端服务器，名为 [cras](https://www.chromium.org/chromium-os/chromiumos-design-docs/cras-chromeos-audio-server/)。cras 将在 openFyde 设备启动时作为服务启动。

alsa 将根据 cras 规则扫描输入和输出设备。它通过输入音频设备将输出设备分为不同类型，如 HDMI、headphones、speakers 等。Chromium OS 在 cras 规则中没有大量细节来将输入设备名称映射到正确的类型。不同输出设备的名称应该以 `type string` 开头。对于 HDMI 设备，'HDMI/DP' 应该是其名称前缀。

另一个需要注意的情况是当耳机和麦克风（耳机组）插入 3.5mm 耳机插孔时的检测；这两种类型的设备每个设备应该有一个名为 `$headphone jack` 的事件。否则，当你将耳机组插入 3.5mm 插孔时，Chromium OS 不会将音频输出切换到耳机，并且其麦克风也不会工作。

## 工作原理
一些关键要点：

 - openFyde / Chromium OS kernel 和其他 Linux OS 的 kernel 之间的共同点是 dts 文件和相关的 drivers。
 - 移植的关键是 mali driver 和 graphics stack。它需要 kernel 和 userland 协同工作。
