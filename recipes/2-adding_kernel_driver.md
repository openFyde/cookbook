# Adding missing kernel driver for non-functioning components


## Objectives

This recipe attempts to provide some pointers for resolving incompatibility issues on openFyde/FydeOS. 

It happens when you find certain components of your device do not function under openFyde/FydeOS, but you find the same component works under Chrome OS Flex, Ubuntu or some other mainstream conventional Linux distributions. 

In this recipe, we will tackle the issue causing Intel AX210 NIC to refuse to work under openFyde images produced in Apr 2022 (equivalent to FydeOS v14.1).



## Preparation

* Make sure that your AX210 NIC is working under another Linux distro.



## Procedures

### Poke around openFyde to find out why the hardware component has failed

We need to find more information about the support status of your NIC to the current kernel version of openFyde. To find out the kernel version of openFyde:

```
(device)
localhost ~ # uname -r

5.10.98-12114-gd38e716283b6-dir
```

It's now no secret that openFyde has a 5.10 kernel.

According to the [Intel AX210](https://www.intel.com/content/www/us/en/support/articles/000005511/wireless.html) Linux support webpage, you find that kernel 5.10+ is well supported and it does also give you a corresponding [downloadable firmware file](https://wireless.wiki.kernel.org/_media/en/users/drivers/iwlwifi-ty-59.601f3a66.0.tgz). Upon unarchiving this you get the firmware binary file called `iwlwifi-ty-a0-gf-a0-59.ucode`.

Now we can use this filename as the keyword to search the filesystems on your openFyde device to see if it's included:

```
(device)
localhost ~ # find /lib/firmware -name iwlwifi-ty-a0-gf-a0-59.ucode

/lib/firmware/iwlwifi-ty-a0-gf-a0-59.ucode
```

Bugger! Both kernel version and the firmware files are in place. We will just have to poke even further. Moving onto the `dmesg`, if the kernel is complaining about this device, we should be able to find some clues about it.

First we need some handlers to grab on:

```
(device)
localhost ~ # sudo lspci -nn | grep Intel

29:00.0 Network controller [0280]: Intel Corporation Wi-Fi 6 AX210/AX211/AX411 160MHz [8086:2725] (rev 1a)
```

From the above we have `29:00.0` to search the `dmesg`:

```
(device)
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


From the dmesg we have a more specific complaint from the kernel: `Timeout waiting for PNVM load!`, now it's time to ask Google about it. It's not difficult to find that it is a common issue caused by fw version 62 where it introduces a PVMN file and somehow the loading process causes the initialisation of AX210 to go haywire, per [Gentoo bug 777324](https://bugs.gentoo.org/777324#c6). 

Also mentioned by this thread, a workaround is to manually bump the max fw version. Consequently, we now need to apply this [patch](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/drivers/net/wireless/intel/iwlwifi/cfg/22000.c?id=000735e5dbbb739ca3742413858c1d9cac899e10) to the correct location of openFyde.



### Generate a kernel patch for openFyde

Now we move our focus away from the openFyde device to a development machine that you wish to work the magic with.

We need a directory to host the Linux kernel source code and pin it to v5.10. Then we can start cherry-picking the abovementioned commit `000735e5dbbb739ca3742413858c1d9cac899e10`.

```
$ cd /path/to/kernel/5.10/
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

The screen output is basically telling that there is a conflict preventing the cherry-pick to go through, now you will need to manually edit the `drivers/net/wireless/intel/iwlwifi/cfg/22000.c` file and resolve the conflict before we can move on. What's the conflict? You guessed it, the fw version number, exactly as what we expected. So, by opening the conflicted file you will find git has already given you a choice:

```
/* Highest firmware API version supported */
<<<<<<< HEAD
#define IWL_22000_UCODE_API_MAX 59
=======
#define IWL_22000_UCODE_API_MAX 62
>>>>>>> 000735e5dbbb (iwlwifi: bump FW API to 62 for AX devices)
```


You just need to leave the version number to be 62, deleting the previous lines, like below:

```
/* Highest firmware API version supported */
#define IWL_22000_UCODE_API_MAX 62
```


Once the conflict is resolved, do the following:

```
$ git add drivers/net/wireless/intel/iwlwifi/cfg/22000.c
$ git cherry-pick --continue

[detached HEAD 30146612133e] iwlwifi: bump FW API to 62 for AX devices
 Author: Luca Coelho <luciano.coelho@intel.com>
 Date: Wed Feb 10 17:23:55 2021 +0200
 1 file changed, 4 insertions(+)
```

Seems git is now happy, so we can use `git format-patch` to generate a patch for this:
```
# git format-patch HEAD~1

0001-iwlwifi-bump-FW-API-to-62-for-AX-devices.patch
```



### Apply the generated patch to openFyde

In the openFyde (as well as Chromium OS) project structure under the cros_sdk, its kernel package is located at `<path-to-cros_sdk>/src/overlays/project-openfyde-patches/sys-kernel/`, you now need to find the corresponding kernel version number (v5.10) and appy the patch:
```
(outside)
$ cd src/overlays/project-openfyde-patches/sys-kernel/
$ cd chromeos-kernel-5_10
$ cp 0001-iwlwifi-bump-FW-API-to-62-for-AX-devices.patch files/
```


### Build kernel with the patch

Now let's do some real Chromium OS developement, first we need to enter the chroot:
```
(outside)$ cros_sdk --nouse-image
```

Issue the `emerge` command to kick off the build process of the `chromeos-kernel-5_10` package, i.e. the kernel v5.10
```
(inside)
$ emerge-amd64-openfyde sys-kernel/chromeos-kernel-5_10
```

The emerge process should finish without error, if things do go wrong, you will need to check if backport looks fine, or refer to the Portage manual. Once the emerge process finishes, the kernel package will be automatically updated with the patch you generated in the previous step. If you wish to produce a new bootable image containing this change, you can skip the rest of this recipe and instead perform a `build_image` process, per the [Guide](https://github.com/openFyde/getting-started).

The following discusses ways to swap the kernel from your device with this one that you've just built.



### Replace kernel/module on openFyde device

The objective here is to replace the kernel on the openFyde device with the one that you've just built, hoping that it could resolve the driver compatibility issue for you. Basically, you have the following options:

- Use `update_kernel.sh` script available in the scripts folder under your cros_sdk:

    ```
    (inside)
    $ cd <path-to-cros_sdk>/src/scripts/
    $  ./update_kernel.sh --remote=<IP-or-hostname of the openFyde instance >

    ```

- Manually replace kernel/module

    Because the file we have modified in previous steps is `drivers/net/wireless/intel/iwlwifi/cfg/22000.c`, therefore we can confidently deduce that only `lib/modules/5.10.98-12114-gd38e716283b6-dirty/kernel/drivers/net/wireless/intel/iwlwifi/iwlwifi.ko` will be updated after emerging(rebuilding) the kernel package. We can find this affected `.ko` file on the build machine:

    ```
    (outside)
    $ sudo find chroot  -name iwlwifi.ko

    chroot/build/amd64-fydeos_apu/lib/modules/5.10.98-12114-gd38e716283b6-dirty/kernel/drivers/net/wireless/intel/iwlwifi/iwlwifi.ko
    ```

    All you need to do now is to copy this (dirty) `.ko` file to your openFyde machine and replace the original:

    ```
    (device)
    $ cp iwlwifi.ko /lib/modules/5.10.98-12114-gd38e716283b6-dirty/kernel/drivers/net/wireless/intel/iwlwifi/
    ```

If all of the above went well, the AX210 wifi on your openFyde device should now work after reboot.



## How it works

Evidently the kernel side of openFyde follows the same gist as Gentoo Linux. If you find yourself bugged by some driver issues, it's best to give the Gentoo community a visit. The steps and procedures to resolve driver issues are not much different than what you would have to do in Gentoo.



