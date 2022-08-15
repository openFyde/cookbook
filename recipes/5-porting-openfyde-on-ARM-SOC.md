# Porting openFyde on ARM SOC 


## Objectives

This recipe aims to describe how to port openFyde for your ARM SBC. Porting on ARM platform is more complicated than x86 architecture machines.

Readers of this recipe are expected to have experience in Linux kernel debugging have basic concepts of ARM architecture and Device tree, familiar with openFyde/chromiumOS build systems. 

Please notice that this article focuses on how to **booting** openFyde on ARM SBC. IOW, you may run into various troubles which are not included here(sound issues, DP output, camera etc.).

In next chapters, we'll take Raxdxa [Rock 4C Plus](https://rockpi.org/rockpi4) with [RK3399 SOC](https://www.rock-chips.com/a/en/products/RK33_Series/2016/0419/758.html) as an example.



## Preparation


* First make sure that your ARM SBC can boot Linux and work as expected.

* **MALI DRIVER**. Considering for GUI stack efficiency, openFyde and chromiumOS have strict requirements of mali drivers. You **MUST** have the source code of linux kernel with proper mali drivers.

* Ability to interact with TTY console of the device.



## Procedures

### Creating board overlay

Creating your overlay folder is the first step. Go check if there is Chromebook similar to your SBC.
For RockChip 3399 chipset, you can copy or inherit from existing board overlays.

If none of SOC overlays is satisfied, reduce the requirement to board overlay in the same architecture. e.g. arm64.

### Find reusable Linux kernel 

openFyde, as a downstream of chromiumOS, it uses several kernel versions from upstream kernels with commits backported by Chromium team.
Those backports include some features required by chromiumOS like security, overlayfs, cgoups and other miscellaneous things. 
Without above features, openFyde can't behave as a full-functional operating system.

Before the start of porting, let's choose the kernel for openFyde on your ARM device.

So please check your linux kernel version first. The newer your kernel version is, the fewer commits should be backported.
Of course, we can reuse kernels chromiumOS already contains:

```
$ ls src/third_party/kernel/
upstream  v4.14  v4.19  v4.4  v5.10  v5.10-arcvm  v5.10-manatee  v5.15  v5.4  v5.4-arcvm  v5.4-manatee
```

All of the above kernels except `upstream` are LTS versions. 
For rk3399 SOC, kernel version v4.4 is maintained by RockChip and we FydeOS team has open-sourced (repo)[https://github.com/FydeOS-for-You-overlays/kernel-rockchip] with some adjustments for openFyde.

For raspberry-pi like device, kernel v5.4 is recommended, here is the [link](https://github.com/FydeOS/kernel-raspberry_pi) for openFyde and chromiumOS.

For rk3588 SOC, this [v5.10 branch](https://github.com/radxa/kernel/tree/stable-5.10-rock5) is prefered, openFyde provides some [patches](https://github.com/openFyde/foundation-rk3588/tree/main/chipset-rk3588/sys-kernel/chromeos-kernel-5_10) based on it. 

If SOC of your device does not belong  to any group above, you have to choose 

1) Port your ARM board to kernel which is maintained by Chromium team. Save time and brain with complicated security and cgroup code.

2) Backport code chromiumOS required. It can make most components of your board work but you may fall into a whack-a-mole game to let openFyde boot.

If your workable linux kernel version is >= 5.10, we suggest you choose method 2. It should consumes little time from you.
Otherwise, method 1 is a better way because the module dependency hell in kernel code.

### U-Boot
Like other linux distributions, openFyde only requires a suitable aka bootable U-boot binary. Since you already have a bootable linux OS, your can export U-Boot and preloader binaries from emmc storage.

For rk3399 SOC, disk layout is documented on the (webpage)[https://opensource.rock-chips.com/wiki_Partitions]. We have made a simple [script](https://github.com/FydeOS-for-You-overlays/uboot-bin-for-pinebookpro/blob/rock-pi4/export_images.sh) for it.

After exporting binary images, put them into ebuild folder `files` and modify the u-boot [ebuild](https://github.com/openFyde/foundation-rk3399/tree/main/baseboard-rockpi4/sys-boot/rockchip-uboot).

### TTY console
The next step is to make TTY console right. Observe tty port of your device in dmesg output of your linux device. There is a package named [tty](https://github.com/openFyde/foundation-rk3399/tree/main/baseboard-rockpi4/chromeos-base/tty) in openFyde, copy and modify it in overlay of your SBC.
Don't forget to add use flag "TTY_CONSOLE=$TTY_PORT" to your make.conf.

And kernel parameter tty should be adjusted too. See the [ebuild](https://github.com/openFyde/foundation-rk3399/blob/main/chipset-rk3399-openfyde/sys-kernel/rockchip-kernel/rockchip-kernel-4.4.190.ebuild).

The priority of dts file is high than kernel parameter so please check parameters in dts and dtsi file. 

### Adjust dts file

This is little off-topic. In general, It's about that copy dts and dtsi files then double check whether all preconditions are ready.
Switches in dts, compatible kernel drivers and kernel configs should be concerned.
If device drivers are missing, you have to pick them from upstream or copy folders from your kernel, add them to upper Makefile then solve some compilation errors.
Be careful while porting drivers, or kernel panic will bite you. 

### Kernel config
Since chromiumOS has additional code, there are additional kernel configs to enable them.
There are three config levels of one (typical ChromeBook board)[https://www.chromium.org/chromium-os/how-tos-and-troubleshooting/kernel-configuration/].

All board overlays in openFyde are using one layer kernel config appointed in make.config:

```
CHROMEOS_KERNEL_CONFIG="/mnt/host/source/src/overlays/overlay-rock4cp-openfyde/kconfigs/fydeos-r96-4_4-def"
```


If you want to add external kernel configs for openFyde, `src/third_party/kernel/$kver/chromeos/config/base.config` contains base configure options.
For arm64 boards, `src/third_party/kernel/$kver/chromeos/config/arm64/common.config` has common options.

Note: simple appending those options to your kernel config file may not work, some dependency options must be enabled, run `make menuconfig ARCH=$arch` then check.

Power supply of some SBCs seems weak, compile unnecessary drivers(mali, wifi, LAN cards etc) as modules as possible.

### Frecon
If everything goes well, build image and kernel starts to output messages.
Do not panic if kernel crashed. Check stack trace in output then debug.

If you can not input anything by keystrokes, check keywords 'frecon' and 'drm' in kernel messages.

Frecon is a console of chromiumos. To see more please check the [documentation](https://chromium.googlesource.com/chromiumos/platform/frecon/+/HEAD/DESIGN-DOC.md).

The most reason why frcon failed to start is the drm drvier, check drm status.


### Mali
Even ARM has open-sourced its [mali driver](https://developer.arm.com/downloads/-/mali-drivers) of linux kernel. However, user-space drivers are black boxes in binary.

If following segement occurs in dmesg, it means kernel mali driver initialized successfully. Otherwise, go check your kernel mali driver.
```
(device)
[   53.109371] mali ff9a0000.gpu: Looking up mali-supply from device tree
[   53.110191] mali ff9a0000.gpu: GPU identified as 0x0860 r2p0 status 0
[   53.110961] mali ff9a0000.gpu: Protected mode not available
[   53.111980] mali ff9a0000.gpu: Using configured power model mali-simple-power-model, and fallback mali-simple-power-model
[   53.118161] mali ff9a0000.gpu: Probed as mali0
```

If GUI of openFyde still doesn't not show up, check `/var/log/ui/ui.LATEST` and `/var/log/chrome/chrome`.
The UI process may exit due to unmatched mali versions between kernel and user-space.

Check the corresponding mali version in chromiumOS and change version in your kernel. The solution seems rough but do solve the error reported. 

### Can't create user
If you can't create user, check the last part of dmesg. If there are something about `crypted home` or `recognized mount option`, that means security part about chromiumOS in kernel is not working now.
Better to check corresponding kernel configs and backported commits.

### Audio
Some linux distributions use pluseaudio as frontend server and alsa as backend to play music. openFyde/chromiumOS has its own frontend server named [cras](https://www.chromium.org/chromium-os/chromiumos-design-docs/cras-chromeos-audio-server/).
Cras server started as a service when device booted.

It scans input devices and output devices in cras rules. It divides output devices into HDMI, headphone, speaker and so on by input audio devices.
chromiumOS has no such details about rules mapping name to type only code says.  Different output devices should start with `type string`. For HDMI device, 'HDMI/DP' should be its name prefix.

Another case that should be taken care of is that headphones and microphone detections, those two types devices should have a event called `$headphone jack` per device. otherwise, chromiumOS won't switch audio output to headphones automaticly and microphone
won't work.


## How it works

The common part of openFyde/chromiumOS kernel and other linux is dts file and realted drivers. The key part of porting is the mali driver, it requires that kernel part and client part co-works without error.





