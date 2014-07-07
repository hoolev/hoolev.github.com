---
layout: post
title: 在Ubuntu 14.04上安装VirtualBox增强功能
categories: Tools 
tags:
---

安装VirtualBox虚拟机的增强功能可以实现如下功能：

* 主机与虚拟机之间的文件共享
* 主机与虚拟机之间的剪切板共享
* 虚拟机窗口随意放大和缩小

> Ubuntu版本：14.04

为了获得上述的便利，按照网上的N多教程，折腾了不少时间，总是遇到内核版本、头文件不匹配，增强功能编译不成功的问题。
秉着Ubuntu下安装软件不应该如此麻烦的信念，不停的Google，终于找到了另外一种安装VirtualBox增强功能的方法。

使用/opt下的脚本移除已经存在的Guest Additions

`/opt/VBoxGuestAdditions-4.3.6/uninstall.sh`

安装Guest Additions ISO

`sudo apt-get install virtualbox-guest-additions-iso`

开启VirtualBox Guest Service驱动

`software-properties-gtk --open-tab=4`
![](/images/tools/wpid-software-properties-gtk-tab4.png)

重启系统就可以看到虚拟机窗口可以随意放大和缩小了，通过设置文件共享也可以在主机和虚拟机之间共享文件了。




