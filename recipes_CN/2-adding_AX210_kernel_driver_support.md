# 修改内核以驱动无法工作的AX210 WIFI 网卡

## 目标

本教程试图对在 openFyde/FydeOS 上出现的 WIFI 网卡不兼容问题时提供一些指导性思路。

更准确地说，是为你发现设备的某些组件在 Ubuntu 等传统 Linux 发行版系统中能够正常工作，但不能在 openFyde/FydeOS 环境中工作时的情况提供参考。

在这篇教程中，我们将解决导致英特尔 AX210 网卡拒绝在 2022 年 4 月的 openFyde/FydeOS 14.1 版本中工作的问题。

## 准备工作

确认设备的 AX210 网卡能够在其他主流 Linux 发行版下工作。

## 操作流程

### 在 openFyde 中找到部分硬件不工作的原因

这需要关于你设备的网卡对当前 openFyde 内核版本的支持状态信息。要找出 openFyde 的内核版本：

```
(device)
localhost ~ # uname -r

5.10.98-12114-gd38e716283b6-dir
```

从上方信息可以看出，现在 openFyde 的内核版本为 5.10。

根据英特尔 [AX210 产品网页](https://www.intel.com/content/www/us/en/support/articles/000005511/wireless.html)上对 Linux 系统的支持信息可以看出，5.10+ 版本的内核完全受支持，并向用户提供了相应的[可下载固件](https://wireless.wiki.kernel.org/_media/en/users/drivers/iwlwifi-ty-59.601f3a66.0.tgz)。解压缩后，用户能够得到一个署名为 iwlWIFI-ty-a0-gf-a0-59.ucode 的二进制文件。

现在，我们可以用这个文件名作为关键词，在你的 openFyde 设备上的文件系统中进行搜索，检查它是否被包含在内：

```
(device)
localhost ~ # find /lib/firmware -name iwlWIFI-ty-a0-gf-a0-59.ucode

/lib/firmware/iwlWIFI-ty-a0-gf-a0-59.ucode
```

该死，不是这问题！？目前内核版本和固件都已确认到位，看来问题不出在这里，我们需要进一步的调查。再来看看 dmesg，如果是因为内核和这个文件有冲突，我们应该能找到一些相关的信息。

首先我们需要通过一些命令来抓取：

```
(device)
localhost ~ # sudo lspci -nn | grep Intel

29:00.0 Network controller [0280]: Intel Corporation Wi-Fi 6 AX210/AX211/AX411 160MHz [8086:2725] (rev 1a)
```

从上述信息中我们可以通过「29:00.0」这条关键词来搜索 dmesg。

```
(device)
localhost ~ # dmesg | grep 29:00.0

[    0.490428] pci 0000:29:00.0: [8086:2725] type 00 class 0x028000
[    0.490460] pci 0000:29:00.0: reg 0x10: [mem 0xfc700000-0xfc703fff 64bit]
[    0.490562] pci 0000:29:00.0: PME# supported from D0 D3hot D3cold
[    0.905161] pci 0000:29:00.0: Adding to iommu group 15
[    5.700251] pci 0000:29:00.0: attach allowed to drvr iwlWIFI [internal device]
[    5.700276] iwlWIFI 0000:29:00.0: enabling device (0000 -> 0002)
[    5.713053] iwlWIFI 0000:29:00.0: api flags index 2 larger than supported by driver
[    5.713068] iwlWIFI 0000:29:00.0: TLV_FW_FSEQ_VERSION: FSEQ Version: 93.8.63.28
[    5.713343] iwlWIFI 0000:29:00.0: loaded firmware version 59.601f3a66.0 ty-a0-gf-a0-59.ucode op_mode iwlmvm
[    5.803694] iwlWIFI 0000:29:00.0: Detected Intel(R) Wi-Fi 6 AX210 160MHz, REV=0x420
[    5.947808] iwlWIFI 0000:29:00.0: loaded PNVM version 0x5a8dfca
[    6.200408] iwlWIFI 0000:29:00.0: Timeout waiting for PNVM load!
[    6.200412] iwlWIFI 0000:29:00.0: Failed to start RT ucode: -110
[    6.200415] iwlWIFI 0000:29:00.0: iwl_trans_send_cmd bad state = 0
[    6.212553] iwlWIFI 0000:29:00.0: Failed to run INIT ucode: -110
[    6.224424] iwlWIFI 0000:29:00.0: retry init count 0
[    6.225218] iwlWIFI 0000:29:00.0: Detected Intel(R) Wi-Fi 6 AX210 160MHz, REV=0x420
[    6.624412] iwlWIFI 0000:29:00.0: Timeout waiting for PNVM load!
[    6.624417] iwlWIFI 0000:29:00.0: Failed to start RT ucode: -110
[    6.624421] iwlWIFI 0000:29:00.0: iwl_trans_send_cmd bad state = 0
[    6.636606] iwlWIFI 0000:29:00.0: Failed to run INIT ucode: -110
[    6.648835] iwlWIFI 0000:29:00.0: retry init count 1
[    6.649974] iwlWIFI 0000:29:00.0: Detected Intel(R) Wi-Fi 6 AX210 160MHz, REV=0x420
[    7.049407] iwlWIFI 0000:29:00.0: Timeout waiting for PNVM load!
[    7.049411] iwlWIFI 0000:29:00.0: Failed to start RT ucode: -110
[    7.049414] iwlWIFI 0000:29:00.0: iwl_trans_send_cmd bad state = 0
[    7.061467] iwlWIFI 0000:29:00.0: Failed to run INIT ucode: -110
[    7.073682] iwlWIFI 0000:29:00.0: retry init count 2
```

从 dmesg 中，我们有一个来自内核的具体报错信息——「Timeout waiting for PNVM load!」，怎么办呢？那当然是请出搜索引擎啦！通过报错信息的关键词搜索后发现，这是由于 fw62 版本导致的一个常见问题；根据 [Gentoo bug 777324](https://bugs.gentoo.org/777324#c6) 的解释，它通过引入一项 PVMN 文件，在加载过程中由于某种原因打乱了 AX210 的初始化进程。

上述帖子里还提到，目前可行的一种解决方法是手动升级目前机器上运行的 fw 版本。所以，你需要将这个[补丁](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/drivers/net/wireless/intel/iwlwifi/cfg/22000.c?id=000735e5dbbb739ca3742413858c1d9cac899e10)正确安装到 openFyde 上。

### 为 openFyde 生成一个内核补丁

现在，让我们把注意力从 openFyde 设备转向到你想要使用的开发机器上来。

你需要一个目录来存放 Linux 内核的源代码，并把它固定在 5.10 版本。然后我们就可以开始cherry-pick上面提到的 000735e5dbbb739ca3742413858c1d9cac899e10 代码提交。

```
$ cd /path/to/kernel/5.10/
$ git cherry-pick 000735e5dbbb739ca3742413858c1

Auto-merging drivers/net/wireless/intel/iwlWIFI/cfg/22000.c
CONFLICT (content): Merge conflict in drivers/net/wireless/intel/iwlWIFI/cfg/22000.c
error: could not apply 000735e5dbbb... iwlWIFI: bump FW API to 62 for AX devices
hint: After resolving the conflicts, mark them with
hint: "git add/rm <pathspec>", then run
hint: "git cherry-pick --continue".
hint: You can instead skip this commit with "git cherry-pick --skip".
hint: To abort and get back to the state before "git cherry-pick",
hint: run "git cherry-pick --abort".
```

从上述的信息中我们得知，有一个错误阻止了cherry-pick，现在你需要手动编辑drivers/net/wireless/intel/iwlWIFI/cfg/22000.c 文件来解决这项问题。我想聪明的你也大概猜到了，这又是 fw 的版本号问题引起的！所以，通过打开冲突的文件，你会发现 git 已经给你提供了一个选择。

```
/* Highest firmware API version supported */
<<<<<<< HEAD
#define IWL_22000_UCODE_API_MAX 59
=======
#define IWL_22000_UCODE_API_MAX 62
>>>>>>> 000735e5dbbb (iwlWIFI: bump FW API to 62 for AX devices)
```

你只需要把版本号保留为 62，删除前面几行，如下图：

```
/* Highest firmware API version supported */
#define IWL_22000_UCODE_API_MAX 62
```

当报错得到解决后，请进行如下操作：

```
$ git add drivers/net/wireless/intel/iwlWIFI/cfg/22000.c
$ git cherry-pick --continue

[detached HEAD 30146612133e] iwlWIFI: bump FW API to 62 for AX devices
 Author: Luca Coelho <luciano.coelho@intel.com>
 Date: Wed Feb 10 17:23:55 2021 +0200
 1 file changed, 4 insertions(+)
```

看来 git 对我们的工作很满意，接下来我们可以用 git format-patch 来生成一个补丁。

```
# git format-patch HEAD~1

0001-iwlWIFI-bump-FW-API-to-62-for-AX-devices.patch
```

### 将生成的补丁应用于 openFyde

在 cros_sdk 下的 openFyde 项目结构中，其内核位于 <path-to-cros_sdk>/src/overlays/project-openfyde-patches/sys-kernel/，你现在需要做的就是找到相应的内核版本号并将补丁打上去。

```
(outside)
$ cd src/overlays/project-openfyde-patches/sys-kernel/
$ cd chromeos-kernel-5_10
$ cp 0001-iwlWIFI-bump-FW-API-to-62-for-AX-devices.patch files/
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

接下来将讨论如何将你设备中的内核与你刚刚建立的这个内核进行交互。

### 替换或更新 openFyde 设备上的内核及其模块

通过替换内核，我们希望它能为你解决 openFyde 设备上驱动兼容性问题。你有以下选择：

- 使用 cros_sdk 下 scripts 文件夹中的 update_kernel.sh 脚本。

```
(inside)
$ cd <path-to-cros_sdk>/src/scripts/
$  ./update_kernel.sh --remote=<IP-or-hostname of the openFyde instance >
```

- 手动替换内核模块

因为我们在前面的步骤中修改的文件是 drivers/net/wireless/intel/iwlWIFI/cfg/22000.c，所以我们知道只有 lib/modules/5.10.98-12114-gd38e716283b6-dirty/kernel/drivers/net/wireless/intel/iwlWIFI/iwlWIFI.ko 会在 emerge 过程完成后获得更新。我们可以在此之上找到这个受影响的 .ko 文件。

```
(outside)
$ sudo find chroot  -name iwlWIFI.ko

chroot/build/amd64-fydeos_apu/lib/modules/5.10.98-12114-gd38e716283b6-dirty/kernel/drivers/net/wireless/intel/iwlWIFI/iwlWIFI.ko
```

你现在需要做的就是把这个 .ko 文件复制到你的 openFyde 设备上，并替换掉原来的同名文件。

```
(device)
$ cp iwlWIFI.ko /lib/modules/5.10.98-12114-gd38e716283b6-dirty/kernel/drivers/net/wireless/intel/iwlWIFI/
```

如果一切顺利，重启后你的 openFyde 设备上的 AX210 WIFI 网卡应该可以正常工作了。

## 工作原理

openFyde 在内核方面与 Gentoo Linux 遵循同样的要领。如果你在后续使用中发现其他的设备驱动兼容性问题，我们建议可以去 Gentoo 社区查看相关帖文和经验分享。在 openFyde 中解决驱动问题的流程与步骤和你在 Gentoo 中需要做的没有本质区别。
