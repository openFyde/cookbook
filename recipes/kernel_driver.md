# Add kernel driver

这个recipe描述了如何在x86_64 的体系下通过修改kernel解决类似
`在 openFyde 下我的设备触控板/wifi/蓝牙/触屏/声卡不工作，但是在 ubuntu 下可以。我如何自己修改内核加上这些缺失的驱动`
的问题。

如果是ARM64下的设备硬件请见 [TODO]().

以下以解决 `Intel AX210 网卡在openFyde 14.1下无法工作` 为例。


## Preparation

* 由于openFyde是基于Linux的，所以当然请确保硬件能够在Linux下正常工作。

* 参照 [before_hack](before_hack.md) 进入openFyde 并且将 `/` 目录重现挂载为rw。

## How to cook

### 调查为何openFyde下的硬件无法工作

我们首先确认openFyde的内核版本是否支持该设备：
```
localhost ~ # uname -r
5.10.98-12114-gd38e716283b6-dir
```

对于AX210来说，Intel官网的列表中标注kernel version 大于5.10就可以，通过下载
Intel官网对应的firmware，可以找到对应的文件名为`iwlwifi-ty-a0-gf-a0-59.ucode`。
然后查找openFyde 的对应firmware：

```
localhost ~ # find /lib/firmware -name iwlwifi-ty-a0-gf-a0-59.ucode
/lib/firmware/iwlwifi-ty-a0-gf-a0-59.ucode
```

这个时候如果发现firmware和内核版本都是符合要求的，那么需要从dmesg中查看是驱动加载出现了问题。
如果内核不符合最低版本的要求，那么需要从高版本的内核中backport对应的驱动。

```
localhost ~ # sudo lspci -nn | grep Intel
29:00.0 Network controller [0280]: Intel Corporation Wi-Fi 6 AX210/AX211/AX411 160MHz [8086:2725] (rev 1a)
```

通过以上输出我们可以拿到 `29:00.0`, 

```
localhost ~ # dmesg | grep 29:00.0
[    0.490428] pci 0000:29:00.0: [8086:2725] type 00 class 0x028000
[    0.490460] pci 0000:29:00.0: reg 0x10: [mem 0xfc700000-0xfc703fff 64bit]
[    0.490562] pci 0000:29:00.0: PME# supported from D0 D3hot D3cold
[    0.905161] pci 0000:29:00.0: Adding to iommu group 15
[    5.700251] pci 0000:29:00.0: attach allowed to drvr iwlwifi [internal device]
[    5.700276] iwlwifi 0000:29:00.0: enabling device (0000 -> 0002)
[    5.713053] iwlwifi 0000:29:00.0: api flags index 2 larger than supported by driver
[    5.713068] iwlwifi 0000:29:00.0: TLV_FW_FSEQ_VERSION: FSEQ Version: 93.8.63.28
[    5.713343] iwlwifi 0000:29:00.0: loaded firmware version 59.601f3a66.0 ty-a0-gf-a0-59.ucode op_mode iwlmvm
[    5.803694] iwlwifi 0000:29:00.0: Detected Intel(R) Wi-Fi 6 AX210 160MHz, REV=0x420
[    5.947808] iwlwifi 0000:29:00.0: loaded PNVM version 0x5a8dfca
[    6.200408] iwlwifi 0000:29:00.0: Timeout waiting for PNVM load!
[    6.200412] iwlwifi 0000:29:00.0: Failed to start RT ucode: -110
[    6.200415] iwlwifi 0000:29:00.0: iwl_trans_send_cmd bad state = 0
[    6.212553] iwlwifi 0000:29:00.0: Failed to run INIT ucode: -110
[    6.224424] iwlwifi 0000:29:00.0: retry init count 0
[    6.225218] iwlwifi 0000:29:00.0: Detected Intel(R) Wi-Fi 6 AX210 160MHz, REV=0x420
[    6.624412] iwlwifi 0000:29:00.0: Timeout waiting for PNVM load!
[    6.624417] iwlwifi 0000:29:00.0: Failed to start RT ucode: -110
[    6.624421] iwlwifi 0000:29:00.0: iwl_trans_send_cmd bad state = 0
[    6.636606] iwlwifi 0000:29:00.0: Failed to run INIT ucode: -110
[    6.648835] iwlwifi 0000:29:00.0: retry init count 1
[    6.649974] iwlwifi 0000:29:00.0: Detected Intel(R) Wi-Fi 6 AX210 160MHz, REV=0x420
[    7.049407] iwlwifi 0000:29:00.0: Timeout waiting for PNVM load!
[    7.049411] iwlwifi 0000:29:00.0: Failed to start RT ucode: -110
[    7.049414] iwlwifi 0000:29:00.0: iwl_trans_send_cmd bad state = 0
[    7.061467] iwlwifi 0000:29:00.0: Failed to run INIT ucode: -110
[    7.073682] iwlwifi 0000:29:00.0: retry init count 2
```

从上述dmesg中可以看到问题出现在`Timeout waiting for PNVM load!`，通过搜索
发现问题出现在Firmware 62版本引入了一个 PVNM 文件导致AX210初始化失败。

[Gentoo bug](https://bugs.gentoo.org/777324#c6)中提到，手动bump内核中的max fw version可以
解决这个问题。因此下一步我们需要手动将[patch](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/drivers/net/wireless/intel/iwlwifi/cfg/22000.c?id=000735e5dbbb739ca3742413858c1d9cac899e10)
放到openFyde对应位置。



### 生成openFyde的kernel patch

首先需要一个v5.10的Linux git目录。然后手动cherry-pick上述所提到的commit 000735e5dbbb739ca3742413858c1d9cac899e10
```
$ git cherry-pick 000735e5dbbb739ca3742413858c1
Auto-merging drivers/net/wireless/intel/iwlwifi/cfg/22000.c
CONFLICT (content): Merge conflict in drivers/net/wireless/intel/iwlwifi/cfg/22000.c
error: could not apply 000735e5dbbb... iwlwifi: bump FW API to 62 for AX devices
hint: After resolving the conflicts, mark them with
hint: "git add/rm <pathspec>", then run
hint: "git cherry-pick --continue".
hint: You can instead skip this commit with "git cherry-pick --skip".
hint: To abort and get back to the state before "git cherry-pick",
hint: run "git cherry-pick --abort".
```

以上cherry-pick出现了冲突，需要手动编辑`drivers/net/wireless/intel/iwlwifi/cfg/22000.c`

从
```
/* Highest firmware API version supported */
<<<<<<< HEAD
#define IWL_22000_UCODE_API_MAX 59
=======
#define IWL_22000_UCODE_API_MAX 62
>>>>>>> 000735e5dbbb (iwlwifi: bump FW API to 62 for AX devices)
```
修改为
```
/* Highest firmware API version supported */
#define IWL_22000_UCODE_API_MAX 62
```

解决冲突之后运行
```
$ git add drivers/net/wireless/intel/iwlwifi/cfg/22000.c
$ git cherry-pick --continue
[detached HEAD 30146612133e] iwlwifi: bump FW API to 62 for AX devices
 Author: Luca Coelho <luciano.coelho@intel.com>
 Date: Wed Feb 10 17:23:55 2021 +0200
 1 file changed, 4 insertions(+)
```

再使用`git format-patch`生成patch：
```
# git format-patch HEAD~1
0001-iwlwifi-bump-FW-API-to-62-for-AX-devices.patch
```



### 将patch应用到openFypde


openFyde的kernel package所在的目录为`openfyde/src/overlays/project-openfyde-patches/sys-kernel/`，找到对应的kernel版本并且放入patch
```
$ cd src/overlays/project-openfyde-patches/sys-kernel/
$ cd chromeos-kernel-5_10
$ cp 0001-iwlwifi-bump-FW-API-to-62-for-AX-devices.patch files/
```


### 编译kernel

进入chroot
```
$ cros_sdk --nouse-image
```

```
(inside)
$ emerge-amd64-openfyde sys-kernel/chromeos-kernel-5_10
```

如果编译错误那么请检查backport是否正确。
如果你想要全新的image的话请执行`build_image`,忽略 第五步。



### 替换 kernel/module
openFyde 有以下几种方式替换kernel：

- 使用update_kernel.sh
```
(inside)
(cr) (release-R96-14268.B/(f993911...)) yue@flintboy ~/chromiumos/src/scripts $  ./update_kernel.sh --remote=<IP-or-hostname of the Chromium OS instance >

```

- 手动替换kernel/module 
以AX210为例，之前修改的文件为drivers/net/wireless/intel/iwlwifi/cfg/22000.c，那么改动后发生变动的只有`lib/modules/5.10.98-12114-gd38e716283b6-dirty/kernel/drivers/net/wireless/intel/iwlwifi/iwlwifi.ko`
在build的机器上找到对应的ko文件，

```
(outside)
$ sudo find chroot  -name iwlwifi.ko
$ chroot/build/amd64-fydeos_apu/lib/modules/5.10.98-12114-gd38e716283b6-dirty/kernel/drivers/net/wireless/intel/iwlwifi/iwlwifi.ko
```

拷贝到对应的ChromiumOS机器上即可。
```
$ cp iwlwifi.ko /lib/modules/5.10.98-12114-gd38e716283b6-dirty/kernel/drivers/net/wireless/intel/iwlwifi/
```

之后重启，AX210就可以正常工作。



## How it works

openFyde是基于Gentoo Linux的派生版本，大多数驱动问题和普通Linux的解决方案是一致的。driver
由Linux kernel提供。
