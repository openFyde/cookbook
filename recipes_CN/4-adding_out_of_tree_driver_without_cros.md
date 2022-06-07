# 在没有 cros 开发环境的情况下，添加主线外的驱动模块

# 目的

本教程试图为解决 openFyde/FydeOS 上的不兼容问题提供一些指导性的意见。

你的设备的某些组件可能在 openFyde/FydeOS 下不工作，但同样的组件在 Chrome OS Flex、Ubuntu 或其他一些主流的传统 Linux 发行版下工作。

还有一种情况是，有些驱动程序只能在主线外模块中找到，甚至在其他主流 Linux 发行版中也没有。

与普通的 Linux 发行版不同，openFyde 和 FydeOS 不提供像 linux-headers 这样的包或文件，而这是大多数主线外模块的依赖。有两种方法来编译主线外的模块。

- 遵循 openFyde [起步走](https://gitee.com/openFyde/getting-started)的步骤。将 chromeos-kernel 的 ebuild 改为你包含主线外模块的 Linux 分支。
- openFyde 的内核是开源的，你可以获取和编译，并加入你想要的模块。把编译好的模块移到你安装 openFyde/FydeOS 的根目录下。

方法一对构建机器的系统要求更高，需要至少 16GB 内存、150GB 可用磁盘空间和一个 4 核 CPU。它需要 git-clone Chromium OS 的全部源代码，通常需要几个小时。

方法二要求机器有 4GB 内存，大约 4GB 的可用磁盘空间来克隆和编译 Linux 内核代码。如果你有足够的耐心，CPU 的速度很慢也没有关系。

下面介绍如何按照方法二编译主线外的模块，以 dkms 模块 [rtl8821ce](https://github.com/tomaspinho/rtl8821ce) 为例。本教程的流程不限于与网卡相关的驱动程序。

# 准备工作

确保你要编译的模块有良好的文档和代码。首先阅读其构建要求。

了解如何在你的 Linux 发行版中编译 Linux 内核并安装依赖项，例如 [ArchLinux](https://wiki.archlinux.org/title/Kernel/Traditional_compilation)、[Debian](https://tools.ietf.org/doc/kernel-package/Kernel.htm)、[Fedora](https://fedoraproject.org/wiki/Building_a_custom_kernel)。

# 具体步骤

## 准备好源代码

openFyde 已经在 [gitee](https://gitee.com/openFyde/kernel) 上开源了内核。[FydeOS For PC](https://fydeos.com/download/pc/) 的内核与 openFyde 相同。

首先克隆内核代码。

```
(build)
$ git clone https://gitee.com/openFyde/kernel
```

在你的 FydeOS/openFyde 设备上运行 `uname -r`。

```
(device)
# uname -r
5.10.98-12114-gd38e716283b6-dir
```

内核是 5.10 版本，所以需要把 openFyde 内核调整为 5.10 版本。

```
(build)
$ git checkout chromeos-5.10
```

然后为 rtl8821ce 准备主线外的模块。

```
(build)
$ git clone https://github.com/tomaspinho/rtl8821ce
```

## 获取内核配置

内核配置位于 openFyde gitee 中。你可以在 openFyde/overlay-$board/kconfig 目录下找到。

对于 APU 变体，配置在 [https://gitee.com/openFyde/overlay-amd64-openfyde_apu/tree/main/kconfig/](https://gitee.com/openFyde/overlay-amd64-openfyde_apu/tree/main/kconfig) 。

对于 rpi4 变体，配置在 [https://gitee.com/openFyde/overlay-rpi4-openfyde/tree/main/kconfig/](https://gitee.com/openFyde/overlay-rpi4-openfyde/tree/main/kconfig) 。

找到与你的内核版本相对应的配置，并将其作为 `.config` 保存到 /path/to/linux，然后运行：

```
(build)
$ cd /path/to/linux
$ make olddefconfig
$ make -j $(nproc)
```

## 编译主线外模块

接下来让我们编译 rtl8821ce 模块。不同模块的 Makefile 可能有所不同，但应该都在 `modules` 部分，看起来像这样：

```
modules:
    $(MAKE) ARCH=$(ARCH) CROSS_COMPILE=$(CROSS_COMPILE) -C $(KSRC) M=$(shell pwd)  modules
```

这一行代码说明，模块使用 $KSRC 作为内核源目录。所以我们可以这么做：

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

上述输出显示模块已经成功编译。

## 安装编译后的模块

在安装之前，请以 root 身份运行，将你的 rootfs 重新挂载为 rw。这样我们就可以直接 scp/rsync 整个模块到你的 FydeOS/openFyde。

然后进入模块并调用 `make install`。

```
(device)
# cd /path/to/copied_module
# make install
```

不要忘记修改 `/etc/modprobe.d/blacklist.conf`，如果你的模块提到了它。

对于 rtl8821ce，必须在 `/etc/modprobe.d/blacklist.conf` 中加入 `blacklist rtw88_8821ce`。

如果上面的操作都很顺利，重启你的 openFyde 设备后该模块就能成功加载。

如果还是不能工作，请检查重启后 `lsmod` 的输出。如果模块没有加载，请尝试调用 `modeprobe $module` 并在 `dmesg` 中检查错误。

# 它是如何工作的

即使 openFyde 和 FydeOS 不提供 linux-headers，主线外的模块也可以用手动编译 openFyde 内核的方式来进行编译。