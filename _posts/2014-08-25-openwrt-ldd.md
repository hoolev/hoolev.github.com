---
layout: post
title: OpenWrt中ldd命令coredump的问题 
categories: WiFi
tags: 
    - JuBo 
    - OpenWrt
---

在JuBo的开发过程中需要把Meteor移植到openwrt上，而Meteor中用到了不少NodeJS的C++扩展模块。
这就需要对NodeJS的C++扩展模块进行交叉编译，交叉编译后运行，在加载模块时报`File not found`错误。

> OpenWrt 12.09

## 排查过程

检查了模块名称、路径都没有问题，使用ldd命令查看模块，直接报错`segfault`。
对node执行ldd命令`ldd node`，没报错，详细列出了node依赖的库。
这问题大了，难道是交叉编译有问题，仔细检查Makefile，没发现什么问题。
Google两个错误，没有发现什么有价值的信息。

木有办法了，只有出撒手锏了。于是使用`node debug`对照node源码进行Debug。
断点、监视，分步运行，最终发现`File not found`这个错误是dlopen这个函数报出来的。
通过阅读`man dlopen`初步推断可能是扩展模块某个或多个依赖的库没有找到导致报错。

为了排除是不是编译的有问题，对系统中自带的共享库执行ldd命令发现也会报`segfault`错误。
于是猜测可能该版本的ldd命令有问题，不能对共享库使用。
`openwrt ldd`的搜索结果确定了问题所在，12.09版本的ldd命令存在[Bug](https://dev.openwrt.org/ticket/11482)。

> 11482 closed defect (worksforme)
> ldd and segmentation fault 
>
> ldd command on openwrt gives always segmentation failed when executing on library 
> for example on libmysqlclient.so.16.0.0

## 解决办法

在该Bug的报告页面上可以发现这个Bug在`Chaos Calmer (trunk)`中已经解决了。
openwrt最新的版本是14.07，在[这里](http://downloads.openwrt.org/barrier_breaker/14.07-rc3/)
找到对应平台的ldd包下载并安装，再执行ldd命令就不会报错了。

通过ldd命令找到模块缺少的库，安装完库之后就不会报`File not found`的错误了。

## 总结

其实如果ldd命令没有问题，那么早就可以确定模块缺少依赖的库，解决问题了。
不过"塞翁失马，焉知非福"，在对node模块加载流程的Debug过程中，加深了对node的理解，这也是一大收获。



