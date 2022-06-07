![](https://fydeos.com/content/wp-content/uploads/2022/05/openfyde-cookbook.jpg)

[英语版本](./README.md)



<br>


## 菜单

- [与 OpenFyde 交互](./recipes_CN/1-talking_to_openfyde.md)
- [修改内核以驱动无法工作的AX210 WIFI 网卡](./recipes_CN/2-adding_AX210_kernel_driver_support.md)
- [使用 fydeos-hardware-tuning 对 I²C 触摸板添加内核驱动支持](./recipes_CN/3-addind_touchpad_driver_support.md)
- [在没有 cros 开发环境的情况下，添加主线外的驱动模块](./recipes_CN/4-adding_out_of_tree_driver_without_cros.md)


<br>


# 关于这本 Cookbook

当我们谈到计算机科学或软件工程方面的 Cookbook 时，首先想到的是什么？O'Reilly 应该有很强的存在感，是的，我指的是那种老式厚厚的平装书。这些书通常都太重了，放在背包里不方便携带，而且封面上总是有一只神奇的怪兽。不可否认的是，这些 Cookbook 给从业者带来了宝贵的实践经验，从而激励了几代人。我们也想创造一本这样的书，希望这本书可以带来一些灵感，当然，也可以提供些微的帮助。

这是一本关于 openFyde 和 Chromium OS 的 Cookbook。

本 Cookbook 中提供的教程致力于为你带去灵感，让你知道能用 openFyde 实现什么目标。这些教程可能可以直接满足你的项目需求，也可能不行，但我们希望起码可以提供一些指导性的方向。

openFyde 和 FydeOS 团队的所有人将继续完善 Cookbook 的内容，对每一个帮助实现这一目标的人表示衷心的感谢。

# 本书包括哪些内容

- 开发 Chromium OS 和 openFyde 的技巧和窍门
- 为 Chromium OS 和 openFyde 的代码库做贡献的日常工作
- Chromium OS 的功能增强、优化和定制
- Linux 内核的修补和调整，包括如何添加驱动程序和改善硬件设备的兼容性
- Linux 内核驱动和固件调试，包括如何诊断和解决触控板、摄像头、无线网卡等非功能性硬件组件的问题

# 使用这本 Cookbook 需要什么前提条件？

本书中的教程是在假设读者对 Linux 或其他类似 UNIX 的操作系统有一些基本了解和操作经验的情况下编写的。拥有软件开发经验可能非常有帮助，但也不是必须的。例如，你应该知道如何通过命令行导航、发布命令、编辑文件、应用补丁和调用软件构建程序等。最重要的是，你应该能够正确和有效地使用搜索引擎。

当然，我们还假设你已经阅读并按照 [openFyde 起步走](https://gitee.com/openFyde/getting-started)的流程走过一遍。通过完成这些，你应该对如何从源代码开始构建一个可以进行修改的 openFyde 镜像有了一定的了解。

# 谁是这本 Cookbook 的目标读者？

- 有兴趣使用 openFyde 作为其大型项目一部分的开发者、团体和公司
- 希望移植 openFyde/FydeOS 目前没有的硬件驱动的用户和开发者
- 像你那样拥有好奇心的用户

# 这本 Cookbook 包含哪些内容？

与你在书店最喜欢的角落里找到的任何其他 Cookbook 相似，它有一篇篇的所谓的「食谱」，也就是教程。每一篇教程提供一个模版，一个指南，当你按照教程中的要求进行操作，起码能得到同样的结果，不致于失败。

本手册中的教程都遵循以下结构。

## 目标

当然，在传统「食谱」中，目标可能是一张美食的照片，还配有漂亮的装饰。但在我们提供的「食谱」中，我们会尽量描述清楚一则教程需要实现的目标以及为什么要实现该目标。

## 准备工作

这一章节不提供原料和超市购物清单，而是提供关于软件和硬件要求的信息，你需要准备的工具，以及其他一些你在实际执行任务前可能需要完成的准备工作。

## 操作步骤

在这一章节中，你会看到关于如何实现目标的分步指南，其信息量之大是我们可以想象的。从理论上讲，如果你遵循这些步骤，应该能够实现同样的目标。

## 它是如何运作的？

我们将尝试提供一些关于理论和原则的讨论，让你更好地了解正在发生的事情和幕后的原理。

## 其他需要注意的事项

如果需要的话，可能有一些最后的要点总结。

# 如何阅读这本 Cookbook？

这里有一些你应该知道的有用信息，以便更好地理解和利用这本 Cookbook。

## 前提条件

- 你已经阅读并按照 [openFyde 起步走](https://gitee.com/openFyde/getting-started)的流程走过一遍，成功构建了一个可启动的 openFyde 镜像。
- 对 [openFyde 起步走](https://gitee.com/openFyde/getting-started)以及 [Chromium OS 开发者指南](https://chromium.googlesource.com/chromiumos/docs/+/main/developer_guide.md)中所涉及的程序和命令有基本的了解。
- 愿意学习新的东西，不害怕用 Google 进入未知的领域。

## 排版习惯

命令行 shell 命令以不同的标签显示，以表明它们适用的设备：

- 你的构建计算机（你正在进行开发所用的计算机）
- 你的构建计算机上的 chroot（Chromium OS SDK）。
- 你的 openFyde 设备（用来启动和运行你构建的镜像的设备）



## 系统要求

以下是你在参考本手册时需要拥有的东西的清单。

- 一个 x86_64 系统来执行 openFyde 的构建。64 位硬件和操作系统是必须的。openFyde（和 Chromium OS）是一个非常大的项目，从源头开始构建通常需要几个小时，甚至十几个小时，这取决于系统配置。
    - CPU：我们建议使用 4 核或更高的处理器。openFyde 的构建过程是并行运行的，所以更多的核心可以帮助大大缩短构建时间。
    - 内存：我们建议至少 16GB，加上足够的 SWAP 空间，因为在这个项目中，你需要从源代码构建 Chromium。截至 2017 年 3 月，编译链接（link）Chromium 需要 8-28GB 的内存，所以如果你的内存较少，你会遇到大量的 SWAP 空间调用或内存不足。然而，如果你不需要建立自己的 Chromium 副本，内存要求将大大降低，代价是失去这个项目提供的一些关键功能。
    - 磁盘：至少 150GB 的空余空间，强烈建议 200GB 或以上。SSD 可以明显缩短构建时间，因为有许多千兆字节的文件需要写入和从磁盘中读取。
    - 网络：源代码的总下载量将超过 30GB。快速和稳定的网络连接是非常有帮助的。
- 一个 x86_64 的 Linux 操作系统作为你的主要工作站，它在本文档后面将被称为主机操作系统。openFyde 的构建过程利用 chroot 将构建环境与主机操作系统隔离。所以理论上任何主流 Linux 系统都可以胜任。然而，只有有限的 Linux 发行版已经经过了 Chromium OS 团队和 FydeOS 团队的测试。已知可以工作的 Linux 版本包括：
    - Ubuntu Linux 18.04 LTS
    - Gentoo Linux
    - Arch Linux
- 一个具有 sudo 权限的非 root 用户账户。构建过程应该由这个用户运行，而不是 root 账户。该用户需要有 sudo 权限。为了简单方便，可以为这个用户设置无密码的 sudo。

# 关于 openFyde 项目

openFyde 是由燧炻科技创新（北京）有限责任公司发起并维护的开源操作系统。简而言之，openFyde 是 Chromium OS 的下游分叉，但它并不强制你使用 Google 的云基础设施和服务来支持该操作系统，我们也尽可能地使用开源组件来替代 Chromium OS/Chrome OS 中的二进制或独立模块。

由于基于 Linux 的操作系统的复杂性，openFyde 被组织成许多独立的代码库，每一个代码库都有其独特的用途。你还需要整个 Chromium OS 的源代码才能继续在 openFyde 上工作，因为 openFyde 和 Chromium OS 基本上共享同一个代码库，还有相当多的依赖关系都在里面。由于上述原因，gitee 中的 openFyde 并非一个项目，而是一个「组织」，里面包含了所有必须的代码库。目前这个仓库不包含任何实际的源代码，只用于介绍和指导。

作为 Chromium OS 的下游分叉，openFyde 并不打算成为 Chromium OS 的对手。我们愿意向上游贡献代码，使 Chromium OS 成为更好的操作系统。
