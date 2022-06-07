# 使用 fydeos-hardware-tuning 对 I²C 触摸板添加内核驱动支持

## 目标

本教程试图对在 openFyde/FydeOS 上出现的硬件不兼容问题提供一些指导性思路。

更准确地说，是为你发现设备的某些组件在 Ubuntu 等传统 Linux 发行版系统中能够正常工作，但不能在 openFyde/FydeOS 环境中工作时的情况提供我们的见解。

正常情况下，触控板可以使用 USB 或 I²C 与设备主机进行连接。在本教程中，我们主要讨论的是触控板通过 I²C 连接到设备主机。

本教程包括如何使用 fydeos-hardware-tuning 调试触控板问题的实操案例，并在内核中添加一个补丁，使其能够在 2022 年 4 月制作的 openFyde 系统（相当于 FydeOS v14.1）下工作。

除了触控板之外，其他支持 I²C 连接的外接设备都可参考此教程。

## 准备工作

确认设备上的触控板可以在其他主流 Linux 发行版下正常工作。

## 操作流程

### 在 openFyde 中自行尝试，找出部分硬件失效的原因，找出你的触控板型号

fydeos-hardware-tuning 会默认安装在 FydeOS 和 openFyde 中，你可以通过输入以下指令直接调用它：

```
(device)
hwtuner --info
```

键入上述指令后，系统会列出设备硬件的所有信息，例如内存、无线组件等。

对于触控板，HID 设备是解决这个问题的关键。

我们的硬件中存在有许多的 HID 设备。一般来说，触控板在系统里会包含「ELAN」这个词 。因此，你可以调用 grep 来进行搜索：

```
(device)
hwtuner --info | grep -i ELAN
[    6.463416] i2c_hid i2c-ELAN0638:00: i2c_ELAN0638:00 supply vdd not found, using dummy regulator
[    6.463416] i2c_hid i2c-ELAN0638:00: Linked as a consumer to regulator.0
[    6.463516] i2c_hid i2c-ELAN0638:00: i2c_ELAN0638:00 supply vdd not found, using dummy regulator
```

ELAN0638 段表示触控板的 ACPI ID, 我们来grep这个型号。

```
(device)
hwtuner --info | grep -i ELAN0638 | grep input
```

如果返回结果空空如也，原因可能是触控板设备没有被硬件所识别。

我们要在内核中加入 ACPI ID ELAN0638，并打上 openFyde 补丁。

### 为 openFyde 生成一个内核补丁

现在，让我们把注意力从 openFyde 设备转向到你想要使用的开发机器上来。

```
(device)
localhost ~ # uname -r

5.10.98-12114-gd38e716283b6-dir
```

你需要一个目录来存放 Linux 内核的源代码，并把它固定在 5.10 版本。然后我们就可以在 5.10内核中加入 HID ACPI ID。对于触控板的 ID，linux/input/elan-i2c-ids.h 是需要编辑的文件。而且我们有很多 ACPI ID：

```
static const struct acpi_device_id elan_acpi_id[] = {
        { "ELAN0000", 0 },
        { "ELAN0100", 0 },
        { "ELAN0600", 0 },
        { "ELAN0601", 0 },
        { "ELAN0602", 0 },
        { "ELAN0603", 0 },
        { "ELAN0604", 0 },
        { "ELAN0605", 0 },
        { "ELAN0606", 0 },
        { "ELAN0607", 0 },
        { "ELAN0608", 0 },
        ...
        { "ELAN0633", 0 }, /* Lenovo S145 */
        { "ELAN0634", 0 }, /* Lenovo V340 Ice lake */
        { "ELAN0635", 0 }, /* Lenovo V1415-IIL */
        { "ELAN0636", 0 }, /* Lenovo V1415-Dali */
        { "ELAN0637", 0 }, /* Lenovo V1415-IGLR */
        { "ELAN1000", 0 },
        { }
```

因此，让我们在「{ "ELAN0638", 0 }」一行之后添加一新行「{ "ELAN0637", 0 }」。

修改完文件后保存，并执行以下操作：

```
$ git add input/elan-i2c-ids.h
$ git commit -s
# Then input the commit subject and message e.g. "A quirk for touchpad ELAN0638 ..."
# Save and quit
```

然后我们通过 git format-patch 生成一个补丁。

```
# git format-patch HEAD~1

0001-A-quirk-for-touchpad-ELAN0638.patch
```

### 将生成的补丁应用于 openFyde

在 cros_sdk 下的 openFyde 项目结构中，其内核包位于<path-to-cros_sdk>/src/overlays/project-openfyde-patches/sys-kernel/，你现在需要找到相应的内核版本号 v5.10 应用该补丁。

```
(outside)
$ cd src/overlays/project-openfyde-patches/sys-kernel/
$ cd chromeos-kernel-5_10
$ cp 0001-A-quirk-for-touchpad-ELAN0638.patch files/
```

### 用补丁构建内核

现在让我们来做一些真正的 Chromium OS 开发，首先我们需要进入 chroot。

```
(outside)$ cros_sdk --nouse-image
```

推送 emerge 指令，启动 chromeos-kernel-5_10 软件包的构建过程，即内核 v5.10。

```
(inside)
$ emerge-amd64-openfyde sys-kernel/chromeos-kernel-5_10
```

emerge 的过程理论上会快速完成，但万一出了什么问题，你需要检查一下 backport 是否正常，或者参考 Portage 手册。当 emerge 过程完成后，内核包将自动更新在上一步生成的补丁。如果你想制作一个包含升级后补丁的新镜像，你可以跳过这个教程的其余部分，而是根据[这篇指南](https://gitee.com/openFyde/getting-started)执行 build_image 过程。

### 替换或更新 openFyde 设备上的内核及其模块

通过替换内核，我们希望它能为你解决 openFyde 设备上驱动兼容性问题。使用 cros_sdk 下的 scripts 文件夹中的 update_kernel.sh 脚本来执行此操作：

```
(inside)
$ cd <path-to-cros_sdk>/src/scripts/
$  ./update_kernel.sh --remote=<IP-or-hostname of the openFyde instance >
```

如果一切顺利，重启后你的 openFyde 设备触控板应该可以正常工作了。

## 工作原理

openFyde 在内核方面与 Gentoo Linux 遵循同样的要领。如果你在后续使用中发现其他的设备驱动兼容性问题，我们建议可以去 Gentoo 社区查看相关帖文和经验分享。在 openFyde 中解决驱动问题的流程与步骤和你在 Gentoo 中需要做的没有本质区别。
