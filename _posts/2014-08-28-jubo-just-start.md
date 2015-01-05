---
layout: post
title: Just Start 
categories: WiFi
tags: 
    - JuBo 
    - OpenWrt
---

Meteor终于在openwrt上运行起来了，看到"App running at: http://localhost:3000/",心里不由的感慨“JuBo终于发芽了”。

>JuBo(巨柏)又名雅鲁藏布江柏木，中国珍惜特有树种，亦为基于OpenWrt的JavaScript OS代号。

当前调试环境是VirtualBox，用的是OpenWrt 12.09的vdi文件。虚拟机的内存是128M、虚拟硬盘也是128M。
原来的目标是希望Meteor能够在内存64M、存储空间也是64M的机器上跑，毕竟目前绝大部分无线路由器是这个配置。

但是在64M内存下，Meteor会由于内存不足被操作系统Kill掉。
64M的存储空间也不够用，把Meteor所需要的各种lib、package和module放上去之后空间就所剩无几了。
如果后面经过优化还不能在64M内存、64M存储的机器上跑的话，那么推广就有点麻烦了。

在网上搜索了下极贰和小米路由的参数：
> 极贰：64M内存、16MFlash
> 小米路由：256M内存、1TB硬盘

看来目前JuBO只能在小米路由上运行了，原本还希望JuBo能够在华为HG255D上运行呢。
不过没有关系，毕竟还是Just Start。


