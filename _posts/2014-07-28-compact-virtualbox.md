---
layout: post
title: 压缩VirtualBox VDI磁盘镜像 
categories: Tools 
tags:
    - VirtualBox
---

VirtualBox使用一个VDI文件来为虚拟机提供一个虚拟硬盘，在使用过程中这个文件会一直增长，而且坑爹的是这个文件不会自己变小，只会越来越大。
也就是说，你往虚拟机里增加一个文件，然后删掉这个文件，VDI文件不会和之前保持一样大小只会增大。

这样下去肯定不行，磁盘空间不能白白浪费。
在网上搜索了下，找到的方法都比较麻烦，不是要装其他软件就是要使用启动盘，而且试下来都不确定能不能生效。
最后，抽取各种方法中容易的部分，组合起来试了下竟然成功了。

> VirtualBox版本  4.2.0

## 清零垃圾文件
在Linux虚拟机里面输入

'sudo dd if=/dev/zero of=/fillerup.zero bs=1024k count=40960'

命令中bs=block size，count=block number,这里是创建了一个40多G的文件，这个大小可根据实际情况调整。
删除创建的文件‘'sudo rm -rf /filerup.zero'，关闭虚拟机。

> 不过网上有人说该方法有时不生效，终极方法是使用LiveCD。

## 压缩VDI磁盘文件
使用VBoxManager压缩VDI文件'VBoxManage modifyhd /the-path-of-VDI.vdi --compact'
这里的路径就是需要压缩的VDI文件的绝对路径。


