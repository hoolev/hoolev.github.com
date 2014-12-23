---
layout: post
title: 浅析Alljoyn中设备的发现和连接
categories: IoT
tags:
    - Alljoyn
---

AllJoyn开源物联网协议框架，**一个能够使连接设备之间进行互操作的通用软件框架和系统服务核心集，也是一个跨制造商来创建动态近端网络的软件应用**。
*使连接设备之间进行互操作*也就是说使用Alljoyn框架的一个前提条件是设备之间的网络是连通的,
在这个前提下，Alljoyn提供不同设备上应用到应用的长连接安全通信通道进行设备间互操作，从而创建一个近端物联网络。

## 设备连接
设备连接就是设备间的网络连接，是Alljoyn工作的前提条件。设备间的网络连接既可以通过网线、也可以通过WiFi或者蓝牙，甚至可以是电线(PLC)。
如果是在WiFi环境中，设备连接就是各个设备通过无线路由器已经组成了一个无线局域网。

Alljoyn提供了一个onboarding服务帮助设备简单快速的接入WiFi网络。
Onboarding由两部分组成：

* onboardee 需要接入WiFi网络的设备
* onboarder 配置onboardee的设备，可能是手机应用或者PC机。

假设我们目前已经有了一个WiFi网络，那么新的onboardee设备通过Onboarding服务接入目标WiFi网络的流程如下：

* onboardee广播一个SSID，这个SSID一般会加上"AJ_"前缀或者"_AJ"后缀。
* onboarder连接上onboardee广播的SSID
* onboarder监听`AllJoyn About announcements`，然后通过onboarding服务接口发送目标WiFi网络的认证信息(SSID、密码等)给onboardee。
* onboardee使用onboarder提供的认证信息连接上目标WiFi网络

## 应用连接
设备连接完成后，就要进行应用连接，只有应用连接完成后才能提供智能服务，比如PC机上的电影既可以在电视上看也可以在手机上看。
应用连接的目的是获取Alljoyn服务,Alljoyn提供了Name-based和Announcement-based两种方式来通告(advertise)和发现(discovery)Alljoyn服务。

### Name-based
一个Alljoyn服务对应一个唯一well-known name，通过这个名字就可以获取对应的Alljoyn服务。
这种方式简单，但是不灵活不开放，因为统一分类命名不同厂家不同设备的服务名称是个不可能完成的任务。
不过应用在Alljoyn联机游戏中还是很方便高效的。

### Announcement-based
在这种方式下，提供Alljoyn服务的应用向外通告自己所提供的服务信息(服务名称、接口名称等)，服务使用者可以根据服务信息来使用对应的服务。
Alljoyn提供了About Announcements核心服务来帮助用户使用Announcement-based方式。

通过About Announcements服务，应用程序可以向外通告以下服务信息：

* App and Device Friendly Names
* Make, Model, Version, Description
* Supported Languages
* App Icon
* Supported objects and interfaces
* Service Port number
* App and Device unique identifiers

`About Announcements`服务由两部分组成：

* About Server, 服务的提供者，向外通告服务信息并提供服务的设备或应用
* About Client, 服务的消费者，获取服务信息并使用服务的设备或应用

一个设备或应用既可以是About Server也可以是About Client。

## 总结
在Alljoyn框架中，我们可以通过onboarding服务来接入一个新的设备，
可以通过About Announcements服务来获取WiFi网络中已接入设备的服务，
可以通过服务接口来完成具体的功能。
这样，一个可控、可互操作的物联网络就形成了。



