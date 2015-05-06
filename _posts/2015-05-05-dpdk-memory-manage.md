---
layout: post
title: DPDK内存管理 
categories: SDN+NFV 
tags:
    - DPDK
---

DPDK通过使用hugetlbfs，提高CPU tlb表的命中率，提高性能。

> DPDK-2.0

## hugepage技术
hugepage(2M/1G..)相对于普通的page(4K)来说有两个特点:

* hugepage不受虚拟内存管理影响，不会被替换出内存
* 同样的内存大小，hugepage产生的页表项数目远少于普通page

这两个特点提高了tlb的命中率，从而带来性能的提升。
例如用户进程需要使用4M大小的内存，如果采用普通page，则需要1K页表项存放虚地址到物理地址的映射关系。
而采用hugepage(2M)只需要2条也表项，这样就减少了页表数，同时减少了tlb的冲突率。

> tlb驻留在CPU的1级cache里，是芯片访问速度最快的缓存，一般只能容纳100多条页表项。
> tlb命中，立即就获取到物理地址，否则需要查 rc3->进程页目录表pgd->进程页中间表pmd->进程页框->物理内存，如果这中间pmd或者页框被虚拟内存系统替换到交互区，则还需要交互区load回内存。

Linux使用hugepage的方式比较简单:

* 将hugetlbfs文件系统mount到某个文件，`mount -t hugetlbfs nodev /mnt/huge`
* 通过修改`/sys/kernel/mm/hugepages/hugepages-2048kB/`目录下的文件可以修改hugepage页面的大小和总数目
* 在用户进程里通过mmap映射hugetlbfs mount的目标文件，这个mmap返回的地址就是大页面了

## 内存管理
mmap系统调用可以通过`MAP_SHARED`标志设置为共享的映射，DPDK的内存共享就依赖于此。
在DPDK中进程分为两种角色，主进程(`RTE_PROC_PRIMARY`)和从进程(`RTE_PROC_SECONDARY`)。

主进程只有一个，必须在从进程之前启动，负责执行DPDK库环境的初始化，先mmap hugetlbfs，构建内存管理相关结构，并将这些结构存入共享内存上的配置文件`rte_config`。
从进程attach到主进程初始化的DPDK库环境上，mmap配置文件`rte_config`，获取内存管理结构。

dpdk采用了一定的技巧，使得最终同样的共享物理内存在不同进程内部对应的虚拟地址是完全一样的，意味着一个进程内部的基于dpdk的共享数据和指向这些共享数据的指针，可以在不同进程间通用。

### 全局配置结构rte_config
rte_config是每个进程私有的数据结构，里面的配置都是每个进程的私有配置。

{% highlight c%}
struct rte_config {
	uint32_t master_lcore;       // 主核 
	uint32_t lcore_count;        // 可以使用的核数
	
	// -c参数设置的进程跑在哪几个核上
	enum rte_lcore_role_t lcore_role[RTE_MAX_LCORE]; 
	enum rte_proc_type_t process_type; // 主进程初始化内存，从进程attach
	
	// 指向各个进程共享的内存配置结构
	// 这个结构被mmap到文件/var/run/.rte_config
	// 通过这个方式多进程实现对mem_config结构的共享
	struct rte_mem_config *mem_config; 
};
{% endhighlight %}

### rte\_mem\_config
这个数据结构mmap 到文件/var/run/.rte\_config中，主/从进程通过这个文件访问实现对这个数据结构的共享。
在每个进程内，使用`rte_config.mem_config`访问这个结构。
需要注意的是，访问这个结构需要加锁，访问结构内不同的内容使用不同的锁。

{% highlight c%}
struct rte_mem_config {
	volatile uint32_t magic;   // Magic number - Sanity check. 

	/* memory topology */
	uint32_t nchannel;    // Number of channels (0 if unknown). 
	uint32_t nrank;       // Number of ranks (0 if unknown). 

	/**
	 * current lock nest order
	 *  - qlock->mlock (ring/hash/lpm)
	 *  - mplock->qlock->mlock (mempool)
	 * Notice:
	 *  *ALWAYS* obtain qlock first if having to obtain both qlock and mlock
	 */
	rte_rwlock_t mlock;   // only used by memzone LIB for thread-safe. 
	rte_rwlock_t qlock;   // used for tailq operation for thread safe.
	rte_rwlock_t mplock;  // only used by mempool LIB for thread-safe.

	uint32_t memzone_idx; // Index of memzone 

	/* memory segments and zones */
	struct rte_memseg memseg[RTE_MAX_MEMSEG];    // Physmem descriptors. 
	struct rte_memzone memzone[RTE_MAX_MEMZONE]; // Memzone descriptors.

	/* Runtime Physmem descriptors. */
	struct rte_memseg free_memseg[RTE_MAX_MEMSEG];

	struct rte_tailq_head tailq_head[RTE_MAX_TAILQ]; //Tailqs for objects 

	/* Heaps of Malloc per socket */
	struct malloc_heap malloc_heaps[RTE_MAX_NUMA_NODES];

	/* address of mem_config in primary process. used to map shared config into
	 * exact same address the primary process maps it.
	 */
	uint64_t mem_cfg_addr;
} __attribute__((__packed__));
{% endhighlight %}

`rte_mem_config.memseg`是存储整体的物理地址到虚地址映射的映射，最终这些地址通过`ret_memzoen_reserve`被分配出去。
`rte_mem_config.free_memseg`记录了当前整个DPDK内存空闲的memseg段，注意，这是对所有进程而言的。
初始化意味着memseg数组里所有的内存都是free的，后面随着分配内存，它越来越小。

对于`rte_mem_config.memseg`中的内存，DPDK以memzone为单位来分配，对于所有的分配情况，都记录在`rte_mem_config.memzone`数组里面。
`rte_mem_config.memzone_idx`记录了当前分配的memzone，每申请一次这个变量+1。

`rte_mem_config.mlock`是用来保护`rte_mem_config.memseg`和`rte_mem_config.memzone`的锁。
### hugepage info数组
这是一个struct hugepage\_info数组，数组每个hugepage\_info保存一个hugetlbfs文件系统中hugepage的数目，大小和挂载点信息。
该数组映射到文件/var/run/.rte_hugepage\_info，主/从进程都能访问它。

> Linux支持多种hugetlbfs文件系统，目录/sys/kernel/mm/hugepages下的一个子目录就代表一种hugetlbfs文件系统。
>
一般一个系统只会使用一种hugetlbfs文件系统，一种hugetlbfs文件系统对应的基础数据包括：页面大小，比如2M，页面数目，比如2K个页面。

{% highlight c%}
struct hugepage_info {
	uint64_t hugepage_sz;   /**< size of a huge page */
	
	// hugetlbfs的挂载点，通过解析/proc/mounts获取
	// 默认是/var/run/.rte_hugepage_info
	const char *hugedir;    
	
	// number of hugepages of that size on each socket
	// 通过读取/sys/kernel/mm/hugepages/hugepages-2048kB/nr_hugepages获取
	uint32_t num_pages[RTE_MAX_NUMA_NODES];
	
	int lock_descriptor;    /**< file descriptor for hugepage dir */
};
{% endhighlight %}

> /sys/kernel/mm/hugepages/hugepages-2048kB/nr\_hugepages 保存了nr\_hugepages内存页数
> hw.contigmem.buffer\_size 保存了hugepage\_sz

### hugepage\_file
一个hugepage\_file都代表一个hugepage页面，存储每个页面的物理地址和进程虚地址的映射关系。
hugetlbfs文件系统配置了多少个hugepage页(nr_hugepages)，在挂载点/mnt/huge下就会有对应的rte\_map\_xx文件，
每个rte\_map\_xx文件会mmap一个hugepage\_sz大小的内存区域。

{% highlight c%}
struct hugepage_file {
	// virtual addr of first mmap() 这个地址在主进程初始化huagepage使用
	void *orig_va;     
	
	// virtual addr of 2nd mmap() 映射到主/从进程中的虚地址，最终使用的地址 
	void *final_va;    
	
	// 这个hugepage页面的物理地址，由rte_mem_virt2phy获取
	uint64_t physaddr; 
	
	// the page size，就是hugepage_sz
	size_t size;       
	
	// NUMA socket ID，由find_numasocket从/proc/self/numa_maps中获取
	int socket_id;     
	
	// 创建rtemap_xx文件的顺序，就是xx的值
	int file_id;       
	
	// 该内存页的物理地址在哪个rte_config.mem_config->memseg[]数组中
	int memseg_id;     
	
	// path to backing file on filesystem = /mnt/huge/rtemap_xx
	char filepath[MAX_HUGEPAGE_PATH]; 
};

{% endhighlight %}

在对各个页面的物理地址份配虚拟地址时，DPDK尽可能把物理地址连续的页面分配连续的虚地址上。
因为CPU/cache/内存控制器的等等看到的都是物理内存，我们在访问内存时，如果物理地址连续的话，性能会高一些。

至于到底哪些地址是连续的，那些不是连续的，DPDK使用结构`rte_mem_config.memseg`来管理。
因为rte_mem_config是映射到文件里面的，所以所有的进程都可见`rte_mem_config.memseg`结构。

### rte\_memseg
memseg数组的作用是将物理地址、虚拟地址都连续的hugepage，并且都在同一个物理CPU，pagesize也相同的hugepage页面集合，把它们都划在一个memseg结构里面，这样做的好处就是优化内存。

{% highlight c%}
struct rte_memseg {
	phys_addr_t phys_addr;      // Start physical address.
	union {
		void *addr;        // Start virtual address.
		uint64_t addr_64;  // Makes sure addr is always 64 bits 
	};
#ifdef RTE_LIBRTE_IVSHMEM
	phys_addr_t ioremap_addr; //Real physical address inside the VM
#endif
	size_t len;           // Length of the segment. 
	uint64_t hugepage_sz; // The pagesize of underlying memory 
	int32_t socket_id;    // NUMA socket ID. 
	uint32_t nchannel;    // Number of channels.
	uint32_t nrank;       // Number of ranks.
#ifdef RTE_LIBRTE_XEN_DOM0
	 /**< store segment MFNs */
	uint64_t mfn[DOM0_NUM_MEMBLOCK];
#endif
} __attribute__((__packed__));
{% endhighlight %}

rte\_memseg.phys\_addr、addr、len、hugepage\_sz和socket\_id都是从hugepage\_file里获取的。

## 初始化源码分析
DPDK的内存初始化工作，主要是将hugetlbfs的配置的大内存页，根据其映射的物理地址是否连续、属于哪个Socket等，有效的组织起来，为后续管理提供便利。

`rte_eal_init`是DPDK运行环境初始化入口函数，其中和内存初始化相关的初始化函数有4个:

- eal\_hugepage\_info\_init
- rte\_config\_init
- rte\_eal\_memory\_init
- rte\_eal\_memzone\_init

### eal\_hugepage\_info\_init
这个函数只有主进程会调用，功能实现比较简单，主要是获取hugetlbfs相关的配置信息：

* 从/sys/kernel/mm/hugepages目录下面读取目录名和文件名，获取系统的hugetlbfs文件系统数，以及每个hugetlbfs的内存面大小。
* 从/proc/mounts读取信息，找到hugetlbfs的挂载点。
* 从/sys/kernel/mm/hugepages/hugepages-2048kB/nr\_hugepages读取信息，获取hugepage大页数。

获取的信息存储在/var/run/.rte\_hugepage\_info

### rte\_config\_init
这个函数主要是初始化rte\_config，主进程调用rte\_eal\_config\_create，从进程调用rte\_eal\_config\_attach将/var/run/.config文件mmap到自己进程空间的`rte_config.mem_config`结构上，这样主进程和从进程都可以访问这块内存。

### rte\_eal\_memory\_init
这个函数的主要功能是初始化hugepage，主进程执行rte\_eal\_hugepage\_init，从进程执行rte\_eal\_hugepage\_attach。

rte\_eal\_hugepage\_init函数是DPDK内存初始化核心函数，DPDK对该函数的注释如下所示，对该函数的分析也从这7个方面展开。

{% highlight c%}
/*
* Prepare physical memory mapping: fill configuration structure with
* these infos, return 0 on success.
* 1. map N huge pages in separate files in hugetlbfs
* 2. find associated physical addr
* 3. find associated NUMA socket ID
* 4. sort all huge pages by physical address
* 5. remap these N huge pages in the correct order
* 6. unmap the first mapping
* 7. fill memsegs in configuration with contiguous zones
*/
{% endhighlight %}

函数一开始，将rte\_config\_init函数获取的配置结构放到本地变量mcfg上，然后检查系统是否开启hugetlbfs，如果不开启，则直接通过系统的malloc函数申请配置需要的内存，然后跳出这个函数。

**1. map N huge pages in separate files in hugetlbfs**

首先循环遍历系统所有的hugetlbfs文件系统，一般来说一个系统中只会使用一种hugetlbfs。
创建nr\_hugepages个hugepage\_file数组，即有多少个内存页，就创建多少个hugepage\_file数据结构。
其次，将特定的hugetlbfs的全部页面映射到本进程，放到本进程的hugepage\_file数组管理，这个过程主要由map\_all\_hugepages函数完成。
这次映射称为第一次映射，映射虚地址存放在`hugepage_file.orig_va`。

**2. find associated physical addr**

遍历hugepage\_file数组，找到每个虚地址对应的物理地址，将这些信息记录在`hugepage_file.phyaddr`。
这个过程由find\_physaddrs函数完成，地址查找功能由rte\_mem\_virt2phy函数完成。

rte\_mem\_vir2phy通过读取/proc/self/pagemap页表文件，得到本进程中虚地址与物理地址的映射关系。
首先使用上一步中，每个内存页mmap得到的虚地址，除以操作系统内存页的大小，得到一个偏移量。
然后根据这个偏移量，在/prox/self/pagemap中，得到物理地址的页框，假设为page。
最后通过物理页框page乘以操作系统内存页的大小，再加上虚拟地址的页偏移，得到物理地址。

{% highlight c%}
physaddr = ((page & 0x7fffffffffffffULL) * page_size) + ((unsigned long)virtaddr % page_size);
{% endhighlight %}

> /proc/self/pagemap 页表文件记录了本进程的页表，即本进程虚拟地址到物理地址的映射关系。

**3. find associated NUMA socket ID**

遍历hugepage\_file数组，找到每个虚地址对应的物理CPU号，将这些信息记录在`hugepage_file.socket_id`，
这个过程由find\_numasocket函数完成。

find_numasocket遍历/proc/self/numa\_maps文件，将非hugepage的虚地址过滤掉，
剩下的虚地址与`hugepage_file.orig_va`比较，得到每个内存页mmap的虚地址对应的物理CPU号。

> /proc/self/numa\_maps 文件记录了本进程的虚拟地址与物理CPU号（多核系统）的对应关系。

**4. sort all huge pages by physical address**

在hugepage\_file数组中，根据物理地址，按从小到大的顺序，将hugepage\_file排序。

**5. remap these N huge pages in the correct order**

使用排序后的物理地址作为mmap的起始地址二次映射，使物理地址等于第二次mmap后的虚地址， 
第二次mmap得到的虚地址保存在`hugepage_file.final_va`。

由于hugepage\_file数组已经基于物理地址排序，这些有序的物理地址可能有2种情况，一种是连续的，另一种是不连续的，
这时候的map_all_hugepages调用会遍历hugepage\_file数组，然后统计连续物理地址的最大内存大小。
在获取了最大连续物理内存大小后，调用get\_virtual\_area函数申请同样大小的虚拟空间，
这样可以保证物理内存连续的其虚拟内存也是连续的。

**6. unmap the first mapping**

munmap释放第一步中内存页mmap的得到的内存，即将`hugepage_file.orig_va`变量对应的虚拟地址空间返回给内核。

*上面几步就完成了hugepage\_file数组的构造，现在这个数组对应了某个hugetlbfs系统的hugepage页，*
*数组的每一个节点是一个hugepage\_file结构，hugepage\_file.phyaddr存放着该页面的物理内存地址，*
*`hugepage_file.final_va`存放着phyaddr映射到进程空间的虚地址，*
*`hugepage_file.socket_id`存放着物理CPU号，*
*如果多个hugepage\_file结构的final\_va虚拟地址是连续的，则其phyaddr物理地址也是连续的。*

**7. fill memsegs in configuration with contiguous zones**

将hugepage\_file数组里属于同一个物理CPU，物理内存连续的多个hugepage用rte\_memseg结构管理起来。
一个rte\_memseg结构维护的内存必然是同一个物理CPU上的，虚拟地址和物理地址都连续的内存。

## 内存分配
经过上面的内存初始化工作，DPDK将hugetlbfs文件系统预留的所有物理内存，统一映射到虚地址空间中，并保存在memseg数组中。
理论上可以通过memseg数组使用这些内存，为了方便使用，DPDK提供了统一的使用接口。

rte\_memzone是DPDK内存管理提供的内存分配基础接口，
通过这些接口程序可以获取基于hugepage的属于同一个物理CPU的物理内存连续的虚拟内存也连续的一块地址。
rte\_ring/rte\_malloc/rte\_mempool等组件都是依赖于rte\_memzone组件实现的。

{% highlight c%}
struct rte_memzone {
#define RTE_MEMZONE_NAMESIZE 32 // Maximum length of memory zone name.

	char name[RTE_MEMZONE_NAMESIZE];  /**< Name of the memory zone. */
	phys_addr_t phys_addr;     /**< Start physical address. */
	union {
		void *addr;            /**< Start virtual address. */
		uint64_t addr_64;      /**< Makes sure addr is always 64-bits */
	};
#ifdef RTE_LIBRTE_IVSHMEM
	phys_addr_t ioremap_addr;  /**< Real physical address inside the VM */
#endif

	size_t len;                /**< Length of the memzone. */
	uint64_t hugepage_sz;      /**< The page size of underlying memory */
	int32_t socket_id;         /**< NUMA socket ID. */
	uint32_t flags;            /**< Characteristics of this memzone. */
	uint32_t memseg_id;        /** <store the memzone is from which memseg. */
} __attribute__((__packed__));
{% endhighlight %}

## 内存分配源码分析
`rte_memzone_reserve`从`rte_mem_config.memseg`中以rte\_memzone为单位分配内存，所有的分配情况都记录在`rte_mem_config.memzone`数组里面。
