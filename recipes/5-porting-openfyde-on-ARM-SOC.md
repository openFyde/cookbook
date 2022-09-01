# Porting openFyde for a device with an ARM SoC


## Objectives

This recipe describes my steps to port openFyde for an SBC with an ARM SoC. Porting on the ARM platform is considerably more complicated than on x86  machines; the process also heavily depends on the design and configuration of individual devices.

Readers of this recipe are expected to have experience in Linux kernel debugging and basic concepts of ARM architecture and embedded systems [devicetree](https://en.wikipedia.org/wiki/Devicetree), as well as basic familiarity with openFyde / Chromium OS build systems. 

Please note that this recipe focuses on how to **bring up** openFyde on **a particular** SBC. For similar hardware devices, you may run into various issues which are not included here(sound, DP output, camera, etc.).

In this recipe, I will document the porting process for Radxa [Rock 4C Plus](https://rockpi.org/rockpi4) with [RK3399 SoC](https://www.rock-chips.com/a/en/products/RK33_Series/2016/0419/758.html).



## Prerequisite

A few things to note:

* First, make sure that your ARM SBC(referred to as "dev board" hereafter) is able to boot at least one Linux distro and that it is working as expected.

* **Notes on mali**: openFyde and Chromium OS have strict requirements for mali drivers. You **MUST** have the source code of the Linux kernel(of the working Linux distro on your board) with the proper mali driver. If you don't have it, consult the dev board vendor to obtain this.

* You also need to be able to interact with tty console of the dev board.

* You must have done at least one successful build of the openFyde on other (more generic) platforms/architectures and are familiar with the technical jargon like "overlay", "use flag", "make.conf", "Makefiles", etc. If you don't, the [Getting started](https://github.com/openFyde/getting-started) is where you want to begin.



## Procedures

### Creating board overlay

Creating your overlay folder is the first step. It'd be helpful first to check if there is a publically available Chromebook board similar to your dev board. For the RockChip RK3399 chipset(aka OP1), you can copy or inherit from existing board overlays("kevin" in this case).

If none of the existing public board overlays is applicable, try going up one level for a board overlay in the same architecture. e.g. "arm64".



### Find a reusable Linux kernel 

openFyde is a downstream fork of Chromium OS; it uses several kernel versions from upstream kernels with commits backported by the Chromium team. Those backport include features that Chromium OS require, such as security enhancements, support for overlayfs, cgoups, etc. Without these features, openFyde won't be a fully functional operating system.

Before the actual porting process, let's choose the kernel for openFyde on your dev board. Here you need to check the current Linux kernel version first. The newer your current kernel version is, it'd be more likely that fewer commits need to be backported. Of course, we can reuse kernels that the Chromium OS source tree has already offered:

```
$ ls src/third_party/kernel/
upstream  v4.14  v4.19  v4.4  v5.10  v5.10-arcvm  v5.10-manatee  v5.15  v5.4  v5.4-arcvm  v5.4-manatee
```

All of the above kernels except `upstream` are LTS versions. For RK3399 SOC, kernel version v4.4 is maintained by RockChip. FydeOS team has this kernel [repo](https://github.com/FydeOS-for-You-overlays/kernel-rockchip) with some adjustments for Chromium OS / openFyde.

 - For Raspberry Pi-like devices, kernel v5.4 is recommended; here is the [link](https://github.com/FydeOS/kernel-raspberry_pi) to the kernel we have been using for openFyde and Chromium OS.

 - For RK3588 SoC, this [v5.10 branch](https://github.com/radxa/kernel/tree/stable-5.10-rock5) is prefered. Furthermore, openFyde provides some [patches](https://github.com/openFyde/foundation-rk3588/tree/main/chipset-rk3588/sys-kernel/chromeos-kernel-5_10) based on it. 

If the SoC of your dev board does not belong to any group above, your options are:

 - 1) Use the kernel maintained by the Chromium team on your dev board, this way, you can save time and potential loss of brain cells by avoiding working on complicated security and cgroup code.

 - 2) Backport the code Chromium OS requires to the factory kernel of your dev board. It can reuse most components of your dev board, but you may fall into a whack-a-mole game to let openFyde boot.

If your factory Linux kernel version is >= 5.10, we suggest you choose approach two; otherwise, approach one would be a better way.



### U-Boot
Like other Linux distributions in the ARM realm, openFyde requires a suitable functional U-Boot binary. Since you already have a bootable Linux OS, you can export the U-Boot and the preloader binaries from the storage drive.

For RK3399 SoC, disk layout is documented (here)[https://opensource.rock-chips.com/wiki_Partitions]. We have made a simple [script](https://github.com/FydeOS-for-You-overlays/uboot-bin-for-pinebookpro/blob/rock-pi4/export_images.sh) to extract it.

After exporting the binary image, you need to put them into the ebuild folder `files` and modify the [U-Boot ebuild](https://github.com/openFyde/foundation-rk3399/tree/main/baseboard-rockpi4/sys-boot/rockchip-uboot).



### TTY console
The next step is to sort out the TTY console. You must first observe the tty port of your dev board from the `dmesg` output. There is a package named [tty](https://github.com/openFyde/foundation-rk3399/tree/main/baseboard-rockpi4/chromeos-base/tty) in openFyde; copy and modify it in the "overlay" of your dev board. Don't forget to add the "use flag" of `TTY_CONSOLE=$TTY_PORT` to your `make.conf`.

The kernel parameters of tty should be adjusted too; for details, see this [ebuild](https://github.com/openFyde/foundation-rk3399/blob/main/chipset-rk3399-openfyde/sys-kernel/rockchip-kernel/rockchip-kernel-4.4.190.ebuild).

The priority of the dts file is higher than kernel parameters, so please check the parameters in the dts and dtsi files. 



### Adjust the dts file
This is a little off-topic but worth mentioning. This step generally involves copying the dts and dtsi files to double-check if all preconditions are met. You should pay concerns to the switches in the dts file, compatible kernel drivers and kernel configs.

Suppose any device drivers are missing; you have to hand-pick them from the upstream or copy the relevant directories from your current working kernel, add them to the upper Makefile and then solve any compilation error that may have been raised. 

Be careful while porting drivers; kernel panics can get very annoying.



### Kernel config
Since Chromium OS has additional kernel features, there are additional kernel configs to enable them. There are three kernel config levels in one [typical Chromebook board](https://www.chromium.org/chromium-os/how-tos-and-troubleshooting/kernel-configuration/).

All board overlays in openFyde use one layer kernel config defined in make.config:

```
CHROMEOS_KERNEL_CONFIG="/mnt/host/source/src/overlays/overlay-rock4cp-openfyde/kconfigs/fydeos-r96-4_4-def"
```


If you want to add external kernel configs for openFyde, `src/third_party/kernel/$kver/chromeos/config/base.config` contains base configure options. For arm64 boards, `src/third_party/kernel/$kver/chromeos/config/arm64/common.config` has the common options, it's a good place to begin.

Note: simply appending those options to your kernel config file may not work; some dependency options must be enabled; run `make menuconfig ARCH=$arch` to find out.

It's also worth mentioning that the power supply of some SBCs seems weak; compiling non-essential drivers(mali, wifi, ethernet NIC, etc.) as modules would be ideal.




### Frecon
If everything goes well from the above steps, you should be able to build a successfully built image. If you attempt to boot it, the kernel should give you output messages. No need to panic if the kernel crashes. Check stack trace in output, and we can begin the debug process.

If you cannot input anything by keystrokes, grep keywords 'frecon' and 'drm' in kernel messages and check what was going on. "Frecon"(FREon CONsole) is the console of Chromium OS graphics stack "freon". To learn more check the [docs](https://chromium.googlesource.com/chromiumos/platform/frecon/+/HEAD/DESIGN-DOC.md) page.

The most likely reason why frcon fails to start is the drm driver, so pay extra attention to the drm-related errors.



### Mali
Although ARM has its kernel level [mali driver](https://developer.arm.com/downloads/-/mali-drivers) in open source, user-space drivers are usually black boxes in binary.

If the following output segment occurs in your dmesg, the kernel mali driver has been initialised successfully. Otherwise, you must check your kernel mali driver errors and resolve all issues.
```
(device)
[   53.109371] mali ff9a0000.gpu: Looking up mali-supply from device tree
[   53.110191] mali ff9a0000.gpu: GPU identified as 0x0860 r2p0 status 0
[   53.110961] mali ff9a0000.gpu: Protected mode not available
[   53.111980] mali ff9a0000.gpu: Using configured power model mali-simple-power-model, and fallback mali-simple-power-model
[   53.118161] mali ff9a0000.gpu: Probed as mali0
```

If the GUI of openFyde still does not show up, check `/var/log/ui/ui.LATEST` and `/var/log/chrome/chrome` for any obvious errors and hints. One likely reason for the UI process to exit is the unmatched mali versions between the kernel and the user space, so check the corresponding mali version in Chromium OS and change it in your kernel accordingly.



### Can't create user
After you have managed to have the GUI shown up and found yourself in the OOBE process, if you can't create a user(by attempting to sign in to the desktop but failed), check the last part of the dmesg. Suppose there are errors about the `cryptohome` or `recognized mount option`. In that case, the kernel's security parts of Chromium OS are not working correctly, including selinux and other relevant components. It's now better to check the corresponding kernel configs and backported commits.




### Audio
Some Linux distributions use "pulseaudio" as a frontend server and alsa as a backend server to play music. openFyde / Chromium OS has its own frontend server named [cras](https://www.chromium.org/chromium-os/chromiumos-design-docs/cras-chromeos-audio-server/). cras will start as a service when the openFyde device is booted.

alsa will scan input and output devices in cras rules. It divides output devices into types, such as HDMI, headphones, speakers, etc., by input audio devices. Chromium OS does not have extensive detail in the cras rules to map input device names into the correct types. The name of different output devices should begin with `type string`. The 'HDMI/DP' should be its name prefix for HDMI devices.

Another case to note is the headphone and microphone(headset) detection when it's inserted into the 3.5mm headphone jack; those two types of devices should have an event called `$headphone jack` per device. Otherwise, Chromium OS won't switch the audio output to headphones, and its microphone won't work when you insert your headset into the 3.5mm jack.



## How it works

Some key takeaway:
- The commonplace between openFyde / Chromium OS kernel and the kernel of other Linux OS is the dts files and related drivers. 
- The key to porting is the mali driver and the graphics stack. It requires the kernel and the userland to work together.
