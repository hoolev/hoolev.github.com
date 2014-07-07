---
layout: post
title: 在Ubuntu 14.04上安装VirtualBox增强功能
categories: Tools 
tags:
---

为了获得主机和虚拟机共享文件、共享剪贴板和调整虚拟机窗口大小的便利，决定安装VirtualBox增强功能。
但是按照网上的N多教程，折腾了不少时间，总是遇到内核版本、头文件不匹配，增强功能编译不成功的问题。
秉着Ubuntu下安装软件不应该如此麻烦的信念，不停的Google，终于找到了另外一种安装VirtualBox增强功能的方法。

> Ubuntu版本：14.04

使用/opt下的脚本移除已经存在的Guest Additions

`/opt/VBoxGuestAdditions-4.3.6/uninstall.sh`

安装Guest Additions ISO

`sudo apt-get install virtualbox-guest-additions-iso`

开启VirtualBox Guest Service驱动

`software-properties-gtk --open-tab=4`
![](/images/tools/wpid-software-properties-gtk-tab4.png)

重启系统就可以看到虚拟机窗口可以随意放大和缩小了，通过设置文件共享也可以在主机和虚拟机之间共享文件了。




