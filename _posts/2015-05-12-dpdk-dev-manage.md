---
layout: post
title: DPDK设备管理 
categories: SDN+NFV 
tags:
    - DPDK
---

当我们指定网卡给DPDK使用后，Linux系统就失去了对网卡的管理，由DPDK完全接管。
因此这里所说的设备管理，主要是指对网卡的管理。
网卡驱动模型一般包含三层，即，PCI总线设备、网卡设备以及网卡设备的私有数据结构，
即将设备的共性一层层的抽象，PCI总线设备包含网卡设备，网卡设备又包含其私有数据结构。

> DPDK-2.0

## 概述  
在DPDK中，首先会注册设备驱动，然后查找当前系统有哪些PCI设备，并通过PCI_ID为PCI设备找到对应的驱动，最后调用驱动初始化设备。

DPDK完成这些工作需要用到三个链表：

* dev\_driver\_list，全局设备驱动链表，驱动在main函数运行前由PMD\_REGISTER\_DRIVER注册。
* pci\_device\_list，全局PCI设备链表，通过扫描/sys/bus/pci/devices/目录生成，按PCI地址从大到小排序。
* pci\_driver\_list，全局PCI驱动链表，由设备驱动的init回调函数注册

这里需要说明下所谓的PCI设备驱动实际包括PCI设备驱动和设备本身驱动两部分，dev\_driver\_list是设备本身驱动，pci\_driver\_list是PCI设备驱动。

- 首先DPDK在main()函数执行之前会生成设备驱动(struct rte\_driver)链表
- 然后在`rte_eal_pci_init()`中扫描/sys/bus/pci/devices/目录生成PCI设备(struct rte\_pci\_device)链表
- 接着在`rte_eal_dev_init()`中注册生成PCI驱动(struct rte\_pci\_driver)链表
- 最后在`rte_eal_pci_probe()`中使用创建好的三个链表初始化PCI设备

## 网卡驱动注册
DPDK支持的所有网卡驱动都通过PMD\_REGISTER\_DRIVER宏进行注册，PMD\_REGISTER\_DRIVER宏定义如下：

{% highlight c%}
#define PMD_REGISTER_DRIVER(d)\
void devinitfn_ ##d(void);\
void __attribute__((constructor, used)) devinitfn_ ##d(void)\
{\
	rte_eal_driver_register(&d);\
}
{% endhighlight %}

DPDK通过GCC attribute的constructor属性，在MAIN函数执行前，就执行`rte_eal_driver_register()`函数，将驱动挂到全局链表dev\_driver\_list上。

设备驱动结构定义如下，其中rte\_driver.init是设备的初始化函数，在该函数中会注册网卡对应的PCI驱动到全局链表pci\_driver\_list上。
{% highlight c%}
struct rte_driver {
	TAILQ_ENTRY(rte_driver) next;  /**< Next in list. */
	enum pmd_type type;		       /**< PMD Driver type */
	const char *name;              /**< Driver name. */
	rte_dev_init_t *init;          /**< Device init. function. */
	rte_dev_uninit_t *uninit;      /**< Device uninit. function. */
};
{% endhighlight %}

## PCI设备扫描
调用`rte_eal_pci_init()`函数，通过遍历/sys/bus/pci/devices/目录查找当前系统中有哪些PCI设备，分别是什么类型，并将它们挂到全局链表pci\_device\_list上。

遍历工作由`pci_scan()`完成，`pcai_scan()`通过读取/sys/bus/pci/devices/目录下相关PCI设备文件，获取对应的信息，
初始化struct rte\_pci\_device数据结构，并按照PCI地址从大到小的顺序挂到pci\_device\_list链表上。

PCI设备结构定义如下：
{% highlight c%}
struct rte_pci_device {
	TAILQ_ENTRY(rte_pci_device) next; /*< Next probed PCI device. */
	struct rte_pci_addr addr;          /**< PCI location. */
	struct rte_pci_id id;              /**< PCI ID. */
	/**< PCI Memory Resource */
	struct rte_pci_resource mem_resource[PCI_MAX_RESOURCE];  
	struct rte_intr_handle intr_handle;  /**< Interrupt handle */
	struct rte_pci_driver *driver;   /**< Associated driver */
	uint16_t max_vfs;               /**< sriov enable if not zero */
	int numa_node;                  /**< NUMA node connection */
	struct rte_devargs *devargs;    /**< Device user arguments */
	enum rte_kernel_driver kdrv;   /**< Kernel driver passthrough */
};
{% endhighlight %}

* rte\_pci\_addr记录PCI地址
* numa\_node记录PCI设备属于哪个物理CPU
* rte\_pci\_id，记录PCI设备ID，包括device ID、subsystem\_vendor ID和subsystem\_device ID。
* rte\_pci\_resource记录PCI设备的在地址总线上的物理地址，以及物理地址空间的大小。

> 每个PCI外设由一个PCI域(PCI domain)、一个总线编号(bus number)、
一个设备编号(device number)及一个功能编号(function number)来标识。
> 每个PCI域可以拥有最多256个总线，每个总线上可支持32个设备，每个设备最多可有8种功能。

## PCI驱动注册
PCI驱动注册由`rte_eal_dev_init()`完成，主要功能就是遍历dev\_driver\_list链表，
执行网卡驱动对应的init的回调函数，注册PCI驱动，并挂到全局链表pci\_driver\_list上。

PCI驱动结构定义如下：

{% highlight c%}
struct rte_pci_driver {
	TAILQ_ENTRY(rte_pci_driver) next;  /**< Next in list. */
	const char *name;             /**< Driver name. */
	pci_devinit_t *devinit;       /**< Device init. function. */
	pci_devuninit_t *devuninit;   /**< Device uninit function. */
	struct rte_pci_id *id_table;  /**< ID table, NULL terminated. */
	uint32_t drv_flags;  /**< Flags contolling handling of device. */
};
{% endhighlight %}

网卡驱动对应的init回调函数会调用`rte_eth_driver_register(struct eth_driver *eth_drv)`注册devinit和devuninit两个回调函数，
所有以太网网卡设备的devinit都会注册为rte\_eth\_dev\_init，devuninit注册为rte\_eth\_dev\_uninit。

struct eth\_driver定义如下：

{% highlight c%}
struct eth_driver {
	struct rte_pci_driver pci_drv; /**< The PMD is also a PCI driver. */
	eth_dev_init_t eth_dev_init;   /**< Device init function. */
	eth_dev_uninit_t eth_dev_uninit; /**< Device uninit function. */
	unsigned int dev_private_size; /**< Size of device private data. */
};
{% endhighlight %}

eth\_driver定义了PCI网卡设备驱动，既包含了PCI设备驱动(pci\_drv)又包含了设备本身驱动(eth\_dev\_init)。
每个类型的以太网卡都会定义一个静态的eth\_driver数据结构，eth\_dev\_init和eth\_dev\_uninit在定义结构时初始化，
pci\_drv在PCI驱动注册时赋值。

## 网卡初始化
调用`rte_eal_pci_probe()`函数，遍历pci\_device\_list和pci\_driver\_list链表，
根据rte\_pci\_id，将rte\_pci\_device与rte\_pci\_driver绑定，
并调用rte\_pci\_driver的devinit回调函数初始化PCI设备。

在`rte_eal_pci_probe_one_driver()`函数中，

* 首先通过比对PCI\_ID的vendor\_id、device\_id、subsystem\_vendor\_id、subsystem\_device\_id四个字段判断pci设备和pci驱动是否匹配。

* PCI设备和PCI驱动匹配后，调用`pci_map_device()`函数为该PCI设备创建map resource。具体如下：

	- 首先读取/sys/bus/pci/devices/PCI/uio目录，获取uio设备的ID，该ID就是uio目录名最后几位的数字。
	当igb_uio模块与网卡设备绑定的时候，会在/sys/bus/pci/devices/对应的PCI设备目录下创建uio目录。
	- 初始化PCI设备的中断句柄。
	- 读取/sys/bus/pci/devices/PCI/maps/map0/目录下的文件，获取UIO设备的map resource。并将其记录在struct pci\_map数据结构中。
	- 检查PCI设备和UIO设备在内存总线上的物理地址是否一致。
	如果一致，对/dev/uioID文件mmap一段内存空间，并将其记录在pci\_map.addr和rte\_pci\_device.mem_resource[].addr中。
	- 将所有UIO设备的resource信息都记录在struct mapped\_pci\_resource数据结构中，并挂到全局链表pci\_res\_list上。
* 调用devinit回调函数初始化PCI设备，这里就是调用`rte_eth_dev_init`
	- 首先，调用`rte_eth_dev_allocate()`在全局数组rte\_eth\_devices[]中分配一个网卡设备。
	并在全局数组rte\_eth\_dev_data[]中为网卡设备的数据域分配空间。
	- 调用`rte_zmalloc()`为网卡设备的私有数据结构分配空间
	- 调用`eth_dev_init()`回调函数初始化网卡设备，设置网卡设备的操作函数集，以及收包、发包函数，
	初始化网卡设备的硬件相关数据，包括设备ID、硬件操作函数集、在内存地址总线上映射的地址、MAC地址等等。
	- 注册中断处理函数


**参考资料**

* DPDK收发包处理流程-----（一）网卡初始化(http://www.cnblogs.com/MerlinJ/p/4108021.html)
* 《Linux设备驱动程序》


