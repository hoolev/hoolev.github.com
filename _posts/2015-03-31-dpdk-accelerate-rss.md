---
layout: post
title: 使用RSS提升DPDK应用的性能
categories: SDN + NFV
tags:
    - DPDK
---

本文描述了RSS以及在DPDK中如何配置RSS达到性能提升和统一分发。

> DPDK 1.8.0

## 什么是RSS
RSS(Receive Side Scaling)是一种能够在多处理器系统下使接收报文在多个CPU之间高效分发的网卡驱动技术。

* 网卡对接收到的报文进行解析，获取IP地址、协议和端口五元组信息
* 网卡通过配置的HASH函数根据五元组信息计算出HASH值,也可以根据二、三或四元组进行计算。
* 取HASH值的低几位(这个具体网卡可能不同)作为RETA(redirection table)的索引
* 根据RETA中存储的值分发到对应的CPU

下图描述了完整的处理流程：
![](/images/network/rss.jpg)

基于RSS技术程序可以通过硬件在多个CPU之间来分发数据流，并且可以通过对RETA的修改来实现动态的负载均衡。

## 在DPDK中配置RSS
DPDK支持设置静态hash值和配置RETA。
不过DPDK中RSS是基于端口的，并根据端口的接收队列进行报文分发的。
例如我们在一个端口上配置了3个接收队列(0,1,2)并开启了RSS，那么
中就是这样的:
 
 {0,1,2,0,1,2,0.........}
 
运行在不同CPU的应用程序就从不同的接收队列接收报文，这样就达到了报文分发的效果。

在DPDK中通过设置`rte_eth_conf`中的`mq_mode`字段来开启RSS功能，
`rx_mode.mq_mode = ETH_MQ_RX_RSS`。

当RSS功能开启后，报文对应的`rte_pktmbuf`中就会存有RSS计算的hash值，可以通过`pktmbuf.hash.rss`来访问。
这个值可以直接用在后续报文处理过程中而不需要重新计算hash值，如快速转发，标识报文流等。

RETA是运行时可配置的，这样应用程序就可以动态改变CPU对应的接收队列，从而动态调节报文分发。
具体通过PMD模块的驱动进行配置，例如`ixgbe_dev_rss_reta_update `和`ixgbe_dev_rss_reta_query`。

## 对称RSS
在网络应用中，如果同一个连接的双向报文在开启RSS之后被分发到同一个CPU上处理，这种RSS就称为对称RSS。
对于需要为连接保存一些信息的网络应用来说，对称RSS对性能提升有很大帮助。
如果同一个连接的双向报文被分发到不同的CPU，那么两个CPU之间共享这个连接的信息就会涉及到锁，而锁显然是会影响性能的。

RSS一般使用Toeplitz哈希算法，该算法有两个输入：一个默认的hash key和从报文中提取的五元组信息。
DPDK使用的默认hash key是微软推荐的，具体定义见lib/librte_pmd_e1000/igb_rxtx.c:1539，
同一个连接的不同方向使用这个默认值计算出来的hash值是不一样的。

具体讲就是{src: 1.1.1.1, dst: 2.2.2.2, srcport: 123, dstport: 456}和{src: 2.2.2.2, dst: 1.1.1.1, srcport: 456, dstport: 123}
计算出来的hash值是不一样的，hash值不一样就会导致两个方向的报文被分发到不同的接收队列，由不同的CPU进行处理。

如果想达到对称RSS的效果，那么需要使用其他hash key替换掉DPDK目前使用的。
在论文《Scalable TCP Session Monitoring with Symmetric Receive-side Scaling》中提到了一个hash key可以做到对称RSS

这里给出hash key的值，具体原理可以参考论文。

`static uint8_t rss_intel_key[40] = { 0x6D, 0x5A, 0x6D, 0x5A, 0x6D, 0x5A, 0x6D, 0x5A, 0x6D, 0x5A, 0x6D, 0x5A, 0x6D, 0x5A, 0x6D, 0x5A, 0x6D, 0x5A, 0x6D, 0x5A, 0x6D, 0x5A, 0x6D, 0x5A, 0x6D, 0x5A, 0x6D, 0x5A, 0x6D, 0x5A, 0x6D, 0x5A, 0x6D, 0x5A, 0x6D, 0x5A, 0x6D, 0x5A, 0x6D, 0x5A, }; `

