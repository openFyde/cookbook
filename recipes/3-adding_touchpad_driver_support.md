# Adding kernel driver support for non-functioning I²C toucpad using fydeos-hardware-tuning


## Objectives

This recipe attempts to provide some pointers for resolving incompatibility issues on openFyde/FydeOS. 

It happens when you find certain components of your device do not function under openFyde/FydeOS, but you find the same component works under Chrome OS Flex, Ubuntu or some other mainstream conventional Linux distributions. 

Touchpad can use either USB or I²C for host connectivity. In this topic, we are talking about touchpad connected to host by I²C.

This recipe includes examples how to debug touchpad issue using [fydeos-hardware-tuning](https://github.com/openFyde/fydeos-hardware-tuning) and add a quirk to kernel and make it work under openFyde images produced in Apr 2022 (equivalent to FydeOS v14.1).

Not only touchpad is connected to host by I²C but also other device types are using I²C. Here we intend to illustrate how to add a I²C device to openFyde kernel.

## Preparation

* Make sure that your touchpad is working under another Linux distro.


## Procedures

### Poke around openFyde to find out why the hardware component has failed and find out your touchpad model

In FydeOS and openFyde, fydeos-hardware-tuning was installed by default. We can call it directly:

```
(device)
hwtuner --info
```


The output of aboved command contains all information of your hardwares. It includes many sections like MEMORY, WIRELESS and so on.


For touchpad, section `HID DEVICES` is the key information to slove issue.


There are many HID devices in your personal device. Normally, lines about touchpad contain a word `ELAN`. So call `grep` to filter:


```
(device)
hwtuner --info | grep -i ELAN
[    6.463416] i2c_hid i2c-ELAN0638:00: i2c_ELAN0638:00 supply vdd not found, using dummy regulator
[    6.463416] i2c_hid i2c-ELAN0638:00: Linked as a consumer to regulator.0
[    6.463516] i2c_hid i2c-ELAN0638:00: i2c_ELAN0638:00 supply vdd not found, using dummy regulator
```


The segment `ELAN0638` indicates the touchpad ACPI ID. Then do a quik grep:

```
(device)
hwtuner --info | grep -i ELAN0638 | grep input

```

Empty output means the cause is that the touchpad device is unrecognized as an INPUT device.

We are gonna add the ACPI ID `ELAN0638` to kernel and patch openFyde.



### Generate a kernel patch for openFyde

Now we move our focus away from the openFyde device to a development machine that you wish to work the magic with.

```
(device)
localhost ~ # uname -r

5.10.98-12114-gd38e716283b6-dir
```

It's now no secret that openFyde has a 5.10 kernel.


We need a directory to host the Linux kernel source code and pin it to v5.10. Then we can add the HID ACPI ID to 5.10 kernel.

For touchpad ids, `linux/input/elan-i2c-ids.h` is the file to be edited. There are many ACPI IDs already:

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

So let's add one new line '{ "ELAN0638", 0 },' after the the line '{ "ELAN0637", 0 },'.

Once the file is saved, do the following:

```
$ git add input/elan-i2c-ids.h
$ git commit -s
# Then input the commit subject and message e.g. "A quirk for touchpad ELAN0638 ..."
# Save and quit

```

Then we can use `git format-patch` to generate a patch for this:
```
# git format-patch HEAD~1

0001-A-quirk-for-touchpad-ELAN0638.patch
```



### Apply the generated patch to openFyde

In the openFyde (as well as Chromium OS) project structure under the cros_sdk, its kernel package is located at `<path-to-cros_sdk>/src/overlays/project-openfyde-patches/sys-kernel/`, you now need to find the corresponding kernel version number (v5.10) and appy the patch:
```
(outside)
$ cd src/overlays/project-openfyde-patches/sys-kernel/
$ cd chromeos-kernel-5_10
$ cp 0001-A-quirk-for-touchpad-ELAN0638.patch files/
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



### Replace kernel or update kernel module on openFyde device

The objective here is to replace the kernel on the openFyde device with the one that you've just built, hoping that it could resolve the driver compatibility issue for you.

- Use `update_kernel.sh` script available in the scripts folder under your cros_sdk:

    ```
    (inside)
    $ cd <path-to-cros_sdk>/src/scripts/
    $  ./update_kernel.sh --remote=<IP-or-hostname of the openFyde instance >

    ```

If all of the above went well, the touchpad on your openFyde device should now work after reboot.



## How it works

Evidently the kernel side of openFyde follows the same gist as Gentoo Linux. If you find yourself bugged by some driver issues, it's best to give the Gentoo community a visit. The steps and procedures to resolve driver issues are not much different than what you would have to do in Gentoo.



