---
layout: post
title: DPDK编译成动态库，应用程序检测不到端口的问题
categories: SDN+NFV 
tags:
    - DPDK
---

把DPDK由静态库方式改为编译成动态库后，原来正常的应用程序就不能运行了。
在初始化的时候，`rte_eth_dev_count()`总是返回0，而用`dpdk_nic_bind.py --status`查看端口是绑定成功的。

DPDK默认是编译成静态库的，改成动态库只需要把`common_linuxapp`文件中`CONFIG_RTE_BUILD_SHARED_LIB=n`修改成`CONFIG_RTE_BUILD_SHARED_LIB=y`就行了。

DPDK编译成动态库后，PMD的各个驱动就单独编译成了一个个的.so文件，而在应用程序中没有指定需要链接的.so文件，因此就导致了检测不到端口的问题。
PMD有好几种驱动，为了更好的移植性，建议在Makefile中指定链接所有驱动的.so文件,
`LDLIBS += -lrte_pmd_e1000 -lrte_pmd_i40e -lrte_pmd_ixgbe`

如果应用运行在虚拟机环境的话，还需要指定`librte_pmd_virtio_uio.so`和`librte_pmd_vmxnet3_uio.so`。

