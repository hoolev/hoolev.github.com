---
layout: post
title: DPDK HowTo
categories: Linux
tags:
    - DPDK
---

DPDK是Intel公司发布的数据包转发处理软件库，可以将数据包处理性能最多提高十倍。
>用户可以在单个英特尔® 至强® 处理器上获得超过 80 Mpps 的吞吐量适合网络数据包分析，处理等操作！

## Quick Start
这里使用的是`DPDK-1.7.1`版本，相关的源码和文档可以去www.dpdk.org下载。
安装的详细过程请参考www.dpdk.org/doc/quick-start，这里只补充说明一些的细节。
系统环境需求：

* kernel >= 2.6.33
* glibc >= 2.7
* gcc >= 4.5.x
* 与内核版本匹配的内核源码

我使用的系统是RedHat 6.5，然后在上面编译了3.12.8的内核，并升级了GCC，最终安装环境是这样的。
升级GCC 4.8.2的脚本，如有需要可以[参考]()。

* kernel = 3.12.8
* glibc = 2.12
* gcc = 4.8.2

如果glibc不符合要求(使用ldd --version可以查看glibc版本)，那么最好的建议是换装更新的系统。
当然如果你是不折腾不舒服斯基，那么也可以考虑升级glibc。

官方给出的quick-start手册中的例子使用的是64位机器，如果你的机器是32位的，那么需要`make config T=i686-native-linuxapp-gcc`。

在编译的过程中，遇到了下面的错误，不知道你会不会遇到。
> error "SSSE3 instruction set not enabled"

搜索的结果好像跟CPU体系结构有关，在make时增加参数`make TOOLCHAIN_CFLAGS="-msse4"`可以解决该问题。

编译完成后，就可以运行`testpmd`了，手册中的命令可能在你的机器上运行不了，调整下命令中的`-c -n`参数值就可以了。

## Compile APP
在DPDK源码的examples目录下提供了很多例子程序，我们可以通过这些例子程序来学习和了解DPDK。
编译这些examples需要设置两个环境变量:

* RTE_SDK - Points to the Intel® DPDK installation directory.
* RTE_TARGET - Points to the Intel® DPDK target environment directory.

在前面编译的时候，我们没有指定DPDK的安装目录，因此两个环境变量是这样指定的
`export RTE_SDK="DPDK source directory"`和`export RTE_TARGET=build`。

设置好环境变量后，进入到相应的examples目录下执行make命令就可以编译对应的例子了。

## NIC Bind
在手册中`tools/dpdk_nic_bind.py --bind=igb_uio $(tools/dpdk_nic_bind.py --status | sed -rn 's,.* if=([^ ]*).*igb_uio *$,\1,p')`命令会把系统中没有使用的网卡都绑定到`igb_uio`驱动下。

网卡绑定到`igb_uio`后，使用`ifconfig`,`ethtool`等系统命令是看不到对应网卡的信息的，操作系统也不能使用绑定的网卡。

我们可以使用`tools/dpdk_nic_bind.py`释放绑定的网卡给操作系统使用，首先执行`tools/dpdk_nic_bind.py --status`

{% highlight sh%}
Network devices using DPDK-compatible driver

============================================
0000:03:00.0 'I350 Gigabit Network Connection' drv=igb_uio unused=igb
0000:03:00.1 'I350 Gigabit Network Connection' drv=igb_uio unused=igb
0000:03:00.2 'I350 Gigabit Network Connection' drv=igb_uio unused=igb
0000:03:00.3 'I350 Gigabit Network Connection' drv=igb_uio unused=igb

Network devices using kernel driver
===================================
0000:02:00.0 'NetXtreme BCM5754 Gigabit Ethernet PCI Express' if=eth0 drv=tg3 unused=igb_uio *Active*
{% endhighlight %}

结果中的第1列是网卡的设备号，第3列是当前使用的驱动，第4列是网卡之前的驱动也就是操作系统使用的网卡驱动。

执行`tools/dpdk_nic_bind.py -u 0000:03:00.1`就可以释放选定的网卡了，不过该网卡此时操心系统还不能使用。

我们还需要把网卡绑定系统使用的驱动`tools/dpdk_nic_bind.py -b igb 0000:03:00.1` ，这样系统就可以使用该网卡了。


