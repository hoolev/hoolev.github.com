---
layout: post
title: 使用gbus调试DBus 
categories: Linux 
tags:
    - JuBo
---

[DBus](www.freedesktop.org/wiki/Software/dbus)是一个低延迟、低开销、高可用的IPC机制。
经过不断的发展，DBus已经成为主流的IPC机制，并且逐渐被应用到嵌入式系统，被Android、MeeGo、Tizen等移动操作系统所采用。
DBus设计了一套面向对象的API，所有的Services的Object函数都以Methond、Signal、Properties的概念对外展现，配合Introspectable API，第三方可以很容易的使用DBus。

> Ubuntu 14.04
> 以[connman](https://connman.net/)为例

== DBus的启动 ==
如果DBus和connman没有安装，则使用`sudo apt-get`安装。
dbus-daemon是一个后台进程，负责消息的转发，在系统启动时通过dbus-launch自动启动。如果需要手动运行，可以查看`man dbus-daemon`。
一般来说，系统中会有两个dbus-daemon进程，一个属于system bus，一个属于session bus。
session bus处理同一用户启动和运行的不同程序之间的通信。system bus处理不同session bus下程序之间的通信。

一般来说连接system通道是没有问题的，但是连接session通道可能会出点小问题。
dbus-monitor可以监测session bus上的消息，我们也可以使用dbus-monitor来确定session是否可用。
运行dbus-monitor，如果没有错误输出，则表明session bus可用。
如果报`Unable to autolaunch a dbus-daemon without a $DISPLAY for X11`的错误，则需在profile中添加`export DISPLAY=:0`配置并重启。
如果重启之后还不行，那么就需要配置`DBUS_SESSION_BUS_ADDRESS`环境变量了。

有两种方法可以获取`DBUS_SESSION_BUS_ADDRESS`要填写的值：

* 在~/.dbus/session-bus/目录下有文件保存已经建好的session bus信息

{% highlight sh %}
DBUS_SESSION_BUS_ADDRESS=unix:abstract=/tmp/dbus-71yAWPQ98j,guid=f12e562709bef04dbeef54ee53c4c720
DBUS_SESSION_BUS_PID=4528
DBUS_SESSION_BUS_WINDOWID=71303169
{% endhighlight %}

* 运行dbus-launch

{% highlight sh %}
DBUS_SESSION_BUS_ADDRESS=unix:abstract=/tmp/dbus-vBaMOLC5f4,guid=c581607d72d266f920297a2753c5e98a
DBUS_SESSION_BUS_PID=10547
{% endhighlight %}

获取到值后再执行`export DBUS_SESSION_BUS_ADDRESS=unix:abstract=/tmp/dbus-71yAWPQ98j,guid=f12e562709bef04dbeef54ee53c4c720`

== gdbus的使用 ==
** 获取connman的services **

{% highlight sh %}
gdbus call --system --dest net.connman --object-path / --method net.connman.Manager.GetServices
([(objectpath '/net/connman/service/ethernet_080027078793_cable', 
	{'Type': <'ethernet'>, 
	'Security': <@as []>, 
	'State': <'online'>, 
	'Favorite': <true>, 
	'Immutable': <false>, 
	'AutoConnect': <true>, 
	'Name': <'Wired'>, 
	'Ethernet': <{'Method': <'auto'>, 
	'Interface': <'eth0'>, 'Address': <'08:00:27:07:87:93'>, 'MTU': <uint16 1500>}>, 
	'IPv4': <{'Method': <'dhcp'>, 'Address': <'10.0.2.15'>, 'Netmask': <'255.255.255.0'>, 'Gateway': <'10.0.2.2'>}>, 
	'IPv4.Configuration': <{'Method': <'dhcp'>}>, 
	'IPv6': <@a{sv} {}>, 'IPv6.Configuration': <{'Method': <'auto'>, 
	'Privacy': <'prefered'>}>, 
	'Nameservers': <['8.8.8.8', '192.168.0.12', '10.10.0.13', '10.10.0.12']>, 
	'Nameservers.Configuration': <@as []>, 
	'Timeservers': <['10.0.2.2']>, 
	'Timeservers.Configuration': <@as []>, 
	'Domains': <['combatelecom.com']>, 
	'Domains.Configuration': <@as []>, 
	'Proxy': <{'Method': <'direct'>}>, 
	'Proxy.Configuration': <@a{sv} {}>, 
	'Provider': <@a{sv} {}>}
)
{% endhighlight %}





