# Adding out of tree driver without cros develop environment


## Objectives

This recipe attempts to provide some pointers for resolving incompatibility issues on openFyde/FydeOS. 

It happens when you find certain components of your device do not function under openFyde/FydeOS, but you find the same component works under Chrome OS Flex, Ubuntu or some other mainstream conventional Linux distributions. 

There is a scenario some drivers are only avaiable in out of tree module, non even in popular other linux distros.

Unlike normal linux distribution, openFyde and FydeOS don't provide package or files like linux-headers which is dependency of most out of tree modules. There are two ways to compile out of tree modules:

* Follow steps in openFyde [getting-started](https://github.com/openFyde/getting-started). Change ebuild of chromeos-kernel to your linux tree which contains modules out of tree.  

* Since kernel of openFyde is now open-sourced, get it, compile it and compile your wanted module. Move the module compiled to your installed openFyde/FydeOS rootfs.


The method 1 have higher system requirements of build machine: 16GB RAM, 150GB free disk space and a 4-core CPU. It needs to git-clone full source code of ChromiumOS and takes about several hours. 

The method 2 requires that a machine has 4GB RAM, about 4GB free disk space to clone and compile linux kernel code. The CPU speed can be as much as slow if you have much patience.

This recipe describes how to compile out of tree module in the method 2.  It takes the dkms module [rtl8821ce](https://github.com/tomaspinho/rtl8821ce) as an example. The flow of this recipe is not limited to NIC related driver.


## Preparation

* Make sure that the module you wanted to compile is well documented and coded. Read its build requirements first.

* Get to know how to compile Linux kernel in your Linux distro and install dependencies.Some references:
  [ArchLinux](https://wiki.archlinux.org/title/Kernel/Traditional_compilation)
  [Debian](https://tools.ietf.org/doc/kernel-package/Kernel.htm)
  [Fedora](https://fedoraproject.org/wiki/Building_a_custom_kernel)



## Procedures

### Get the source code ready


openFyde has open sourced it's kernel on [github](https://github.com/openFyde/kernel). Kernel of [FydeOS For PC](https://fydeos.io/download/pc) is same as openFyde.

First clone the kernel code:
```
(build)
$ git clone https://github.com/openFyde/kernel linux
```

Run `uname -r` on your FydeOS/openFyde device:

```
(device)
# uname -r
5.10.98-12114-gd38e716283b6-dir
```


The kernel is v5.10, so we checkout our openFyde kernel to v5.10:
```
(build)
$ git checkout chromeos-5.10
```

Then prepare your out of tree module, for rtl8821ce:
```
(build)
$ git clone https://github.com/tomaspinho/rtl8821ce
```


### Get the kernel config

The kernel configs are located in openFyde github. You can find it in openFyde/overlay-$board/kconfig directory.

For APU variant, configs are in https://github.com/openFyde/overlay-amd64-openfyde_apu/tree/main/kconfig/ .

For rpi4 variant, configs are in https://github.com/openFyde/overlay-rpi4-openfyde/tree/main/kconfig/ .

Find corresponding config to your kernel version and save it as `.config` to /path/to/linux. Run:

```
(build)
$ cd /path/to/linux
$ make olddefconfig
$ make -j $(nproc)
```



### Compile the out of tree module


Let's compile the rtl8821ce module next. The Makefile may differ from your module but there should be section `modules` which looks like:
```
modules:
    $(MAKE) ARCH=$(ARCH) CROSS_COMPILE=$(CROSS_COMPILE) -C $(KSRC) M=$(shell pwd)  modules
```
The line says that the module are using $KSRC as kernel source directory. So we can do:
```
(build)

$ cd /path/to/module
$ make KSRC=/path/to/linux/ -j $(nproc)

make ARCH=x86_64 CROSS_COMPILE= -C /root/linux/ M=/root/rtl8821ce  modules
make[1]: Entering directory '/path/to/linux'
…
  LD [M]  /root/rtl8821ce/8821ce.o
  MODPOST /root/rtl8821ce/Module.symvers
  CC [M]  /root/rtl8821ce/8821ce.mod.o
  LD [M]  /root/rtl8821ce/8821ce.ko
make[1]: Leaving directory '/path/to/linux'
```

Above output shows the module was built successfuly.


### Install the compiled module

Before the installation, please run as root, remount your rootfs as rw. So we can scp/rsync the whole module to your FydeOS/openFyde directly.


Then enter the module and call `make install`:

```
(device)
# cd /path/to/copied_module
# make install
```

Don't forget to modify `/etc/modprobe.d/blacklist.conf` if your module mentioned it.


For rtl8821ce, `blacklist rtw88_8821ce` must be added to `/etc/modprobe.d/blacklist.conf`.

If all of the above went well, the module on your openFyde device should now work after reboot.


If it doesn't work, please check output of  `lsmod` after reboot. If the module is not loaded, try to call `modeprobe $module` and check errors in `dmesg`.


## How it works

Even openFyde and FydeOS don't provide linux-headers, out of tree module can be compiled with manual compiled openFyde kernel.



