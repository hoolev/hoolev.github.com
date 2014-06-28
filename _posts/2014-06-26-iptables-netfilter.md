---
layout: post
title: Netfilter-iptables报文过滤框架
tags: 
    - Linux 
    - Network
---

首先介绍Netfilter/iptables，然后以iptables --> ip_tables --> netfilter的顺序分析整个框架，最后说明不同层次的内核模块的编写方法。
>内核版本是3.0.85。

## What is Netfilter/iptables 
Netfilter/iptables是Linux内核内置的报文过滤框架，程序可以通过该框架完成报文过滤、地址转换(NAT)以及连接跟踪等功能。

Netfilter/iptables由两部分组成，一部分是Netfilter的"钩子(hook)"，这些"钩子"由Linux内核协议栈提供，内核模块可以通过注册"钩子"来完成各种各样的功能。
另一部分是iptables的规则，这些规则规定了"钩子"如何工作。
 
下图很直观的说明了用户空间的iptables和内核空间的ip_tables模块、Netfilter之间的关系。
![](/images/network/netfilter_iptables_relation.jpg)

之所以说Netfilter/iptables是一个框架是因为它提供了最基本的底层支撑，这种底层支撑就是5个"钩子"点——内嵌在内核协议栈的检查点。

当报文经过各个检查点时，就可以通过"钩子"函数对报文进行处理完成相应功能。
![](/images/network/netfilter.jpg)

## iptables 
iptables是一个工作于用户空间的防火墙应用软件，允许系统管理员通过相关的表、链和规则来处理网络数据报文。
>2.4、2.6和3.0内核支持iptables，3.13以后的内核则由nftables取代。

*   表(table)：每个表包含若干条不同的链，iptables包含raw、nat、mangle和filter四个表。
*   链(chain)：每条链包含一系列规则，这些规则会被依次应用到每个遍历该链的数据包上。与Netfilter的5个"钩子"对应，iptables也有5个预先定义的链。
*   规则(rule)：一个或多个匹配及其对应的目标。
*   目标(target)：指定的动作，说明如何处理一个包，比如丢弃，接受，拒绝，或使用自定义目标。
*   策略(police)：对于iptables中某条链，当所有规则都不匹配时其默认的处理动作。
*   匹配(match)：符合指定的条件，比如指定的IP地址和端口。
 
下图列出了iptables中的表，以及每个表中包含的链。
![](/images/network/iptables.jpg)

### 数据包处理过程 
现在，让我们看看当一个数据包到达时它是怎么依次穿过各个链和表的。基本步骤如下：

1.    数据包到达网络接口，比如 eth0。
2.    进入 raw 表的 PREROUTING 链，这个链的作用是赶在连接跟踪之前处理数据包。
3.    如果进行了连接跟踪，在此处理。
4.    进入 mangle 表的 PREROUTING 链，在此可以修改数据包，比如 TOS 等。
5.    进入 nat 表的 PREROUTING 链，可以在此做DNAT，但不要做过滤。
6.    决定路由，看是交给本地主机还是转发给其它主机。

到了这里我们就得分两种不同的情况进行讨论了

**一种情况就是数据包要转发给其它主机，这时候它会依次经过：**

7.    进入 mangle 表的 FORWARD 链，这里也比较特殊，这是在第一次路由决定之后，在进行最后的路由决定之前，  我们仍然可以对数据包进行某些修改。
8.    进入 filter 表的 FORWARD 链，在这里我们可以对所有转发的数据包进行过滤。需要注意的是：经过这里的数据包是转发的，方向是双向的。
9.    进入 mangle 表的 POSTROUTING 链，到这里已经做完了所有的路由决定，但数据包仍然在本地主机，我们还可以进行某些修改。
10.   进入 nat 表的 POSTROUTING 链，在这里一般都是用来做 SNAT ，不要在这里进行过滤。
11.   进入出去的网络接口。完毕。

 **另一种情况是，数据包就是发给本地主机的，那么它会依次穿过：**
 
7.    进入 mangle 表的 INPUT 链，这里是在路由之后，交由本地主机之前，我们也可以进行一些相应的修改。
8.    进入 filter 表的 INPUT 链，在这里我们可以对流入的所有数据包进行过滤，无论它来自哪个网络接口。
9.    交给本地主机的应用程序进行处理。
10.   处理完毕后进行路由决定，看该往那里发出。
11.   进入 raw 表的 OUTPUT 链，这里是在连接跟踪处理本地的数据包之前。
12.   连接跟踪对本地的数据包进行处理。
13.   进入 mangle 表的 OUTPUT 链，在这里我们可以修改数据包，但不要做过滤。
14.   进入 nat 表的 OUTPUT 链，可以对防火墙自己发出的数据做 NAT 。
15.   再次进行路由决定。
16.   进入 filter 表的 OUTPUT 链，可以对本地出去的数据包进行过滤。
17.   进入 mangle 表的 POSTROUTING链，同上一种情况的第9步。注意，这里不光对经过防火墙的数据包进行处理，还对防火墙自己产生的数据包进行处理。
18.   进入 nat 表的 POSTROUTING 链，同上一种情况的第10步。
19.   进入出去的网络接口。完毕。

用一张图总结上面的所有的步骤：
![](/images/network/iptables_traverse.jpg)

## ip_tables内核模块 
ip_tables内核模块是防火墙的核心模块，负责维护防火墙的规则表，和与用户空间的iptables应用程序通信。

通过这些规则实现防火墙的四个核心功能：报文过滤(filter)、地址转换(NAT)、报文处理(mangle)和连接跟踪(conntrack)。

### 规则的存储和遍历机制 
规则是顺序存储的,一条规则主要包括三个部分:ipt_entry、ipt_entry_matchs和xt_entry_target。

ipt_entry_matchs由多个xt_entry_match组成。

*   标准匹配结构(ipt_entry)，主要包含数据包的源、目的IP，掩码等。
*   扩展匹配结构(xt_entry_match)，一条规则可以有零个或多个xt_entry_match结构。
*   规则的动作(xt_entry_target)，一条规则有且只有一个target动作。只有标准匹配和扩展匹配都匹配时才执行target动作。
 
在ipt_entry中还保存有与遍历规则相关的变量target_offset和next_offset，通过target_offset可以找到规则中xt_entry_target的位置，通过next_offset可以找到下一条规则的位置。
![](images/network/rules_storage.jpg)

函数ipt_do_table()实现了规则的遍历，该函数根据传入的参数table和hook找到相应的规则起点，即第一个ipt_entry的位置。
标准匹配通过函数ipt_packet_match()来实现，该函数主要对报文的五元组信息进行匹配，扩展匹配通过宏xt_ematch_foreach来实现。
在对数据包进行匹配后，接着通过函数ipt_get_target()获取规则动作ipt_entry_target，进行相应的动作处理。

### 表、匹配、目标存储及管理机制 
**struct xt_af xt[]结构数组用于挂载各个协议的match和target。**
该数组在Netfilter初始化或匹配模块扩展时进行更新，在初始化时，默认的表和动作会添加到相应的链表中。

xt[]是一个一维数组，其按照协议的不同分别存储，目前我们常用的协议主要是IPV4。

*   xt_register_match(struct xt_match *match)与xt_unreginster_match(struct xt_match *match)
    用于在xt[]数组上挂载或卸载对应协议的match
*   xt_register_target(struct xt_target *target)与xt_unregister_target(struct xt_target *target)
    用于在xt[]数组上挂载或卸载对应协议的target
*   struct xt_match *xt_find_match()与struct xt_target *xt_find_target()
    用于在xt[]数组中查找对应协议的match或target与对应规则相关联，并增加match和target所在模块的引用计数。

**net->xt.tables[]网络命名空间协议链表用于将不同协议的表挂载到对应协议链表中。**

* struct xt_table *xt_register_table(struct net *net, const struct xt_table *input_table, struct xt_table_info *bootstrap, struct xt_table_info *newinfo)
  主要是复制input_table到table表，并将newinfo（由调用该函数模块提供的私有数据xt_table_info）与该表的table->private指针相关联，然后根据该表指定的协议挂入对应的net->xt.table[table->af]链表中。
*   void *xt_unregister_table(struct xt_table *table)
    主要是将table从net.xt.table[table->af]链表中取下来，并返回table->private指针指向的xt_table_info数据。
*   struct nf_hook_ops *xt_hook_link(const struct xt_table *table, nf_hookfn *fn)与void xt_hook_unlink(const struct xt_table *table, struct nf_hook_ops *ops)

  主要是利用xt_table结构和钩子函数构造出nf_hook_ops钩子项，然后调用nf_register_hooks()或nf_unregisgter_hooks()函数来注册或注销协议对应点的钩子函数。

添加表操作一定要先通过xt_register_table()添加一个表，然后再通过xt_hook_link()使HOOK能够引用这些表；
删除表操作一定要先通过xt_hook_unlink()去掉HOOK对表的引用，然后再通过xt_unregister_table()删除一个表。

## 编写Netfilter target模块 
Netfilter/iptables框架可以让我们向其中添加功能，要添加功能需要自己写一个内核模块并向这个框架注册。
通过编写自定义的扩展模块，我们可以匹配、修改、跟踪任何指定的包。这个自定义的扩展模块既可以是match模块，也可以是target模块。

这里只讲述如何编写一个target模块，如何编写match模块可以参考[注3]。
这个模块对报文不做任何处理，编写这个模块的目的是为了说明如何搭建一个target模块的骨架。

该模块在用户空间的用法是：

{% highlight c %}
iptables -t mangle -A INPUT -p udp -j TEST --action tran
{% endhighlight %} 
这条规则在mangle表的INPUT链上对UDP报文执行TEST动作，其中TEST是自定义的target模块，--action tran是自定义的参数。

自定义的扩展模块分为用户态程序和内核态程序两部分，用户态程序以so共享库的形式存在，内核态程序以ko内核模块的形式存在。
下面会从用户态和内核态两个方面说明如何搭建target模块骨架。

### 用户态开发 
iptables库的基本用途就是和用户交互，解析并传送用户输入的参数给内核态程序。
我们首先初始化xtables_target结构中常用的字段：

{% highlight c%}
static struct xtables_target test_target_reg =` 
{
    .name			= "TEST",          
    .version        = XT_TEST_VERSION,
    .family         = NFPROTO_IPV4,
    .size           = XT_ALIGN(sizeof(struct test_target_info)),
    .userspacesize	= XT_ALIGN(sizeof(struct test_target_info)),
    .help			= test_help,
    .parse			= test_parse,
    .final_check	= test_check,
    .print			= test_print,
    .extra_opts		= test_opts,
};
{% endhighlight %}

其中:

*   name是模块名，用于自动加载共享库。
*   size和userspacesize用于确保用户态和内核态共享结构的大小一致，该共享结构由用户根据具体需求自定义。
*   parse在用户输入一条新规则的时候调用，用于验证参数的合法性。
*   final_check最后一次参数合法性检查，在用户输入完规则后，参数解析刚刚完成的时候调用。
*   print用于"iptables -L"显示添加的规则。
*   extra_opts是struct option类型的结构体，用于将每个参数映射到一个值。

iptables架构能够支持多个共享库，每个共享库需要使用xtables_register_target向iptables注册，这个函数将在模块被iptables加载的时候调用。

{% highlight c%}
void _init(void)
{
    xtables_register_target(&test_target_reg);
}
{% endhighlight %}

下面是用户态需要实现函数的声明，用户可以根据具体需求实现对应的函数。
这些函数的参数是固定的，函数名称由用户自定义。

{% highlight c%}
static void test_help(void);
static int test_parse(int c, char **argv, int invert, unsigned int *flags,const void *entry, 
                           struct xt_entry_target **target);
                           
static void test_check(unsigned int flags);
static void test_print(const void *ip, const struct xt_entry_target *target,int numeric)；
{% endhighlight %}

### 内核态开发 
内核态开发的主要工作是实例化一个xt_target对象，然后对其进行必要的初始化设置，最后通过xt_register_target将其注册到net->xt.tables[AF_INET].target全局链表中。我们首先初始化xt_target结构中常用字段：

{% highlight c %}
static struct xt_target test_reg  = 
{
    .name           = "TEST",
    .family         = AF_INET,
    .table          = "mangle",
    .target         = test_target,
    .targetsize     = sizeof(struct test_target_info),
    .me             = THIS_MODULE,
};
{% endhighlight %}

其中：

*   name是模块名，该名称必须得与用户态so共享库的名称一致。
*   family协议族，我们常用的是AF_INET。
*   target需要注册的回调函数，具体功能就在该函数里实现。
*   table指明target注册的表名称。

然后在模块加载和退出函数中注册和移除自定义模块：

{% highlight c%}
int init_module(void)
{
	xt_register_target(&test_req);
}
void cleanup_module() 
{
	xt_unregister_target(&test_reg);
}
{% endhighlight %}

下面是回调函数的声明，其中targinfo是由用户态传入的参数。

{% highlight c%}
static unsigned int test_target(struct sk_buff **pskb, 
		const struct net_device *in, const struct net_device *out,
    	unsigned int hooknum, const struct xt_target *target, 
		const void *targinfo, void *userdata)；
{% endhighlight %}

### 总结 
加载target模块后，net->xt.tables[AF_INET].target链表中就会存储我们的自定义模块TEST。

当用户输入'iptables -t mangle -A INPUT -p udp -j TEST --action tran'命令的时候，用户态的so共享库就会加载，我们实现的parse、final_check函数就会被调用解析用户输入的参数'--action tran'。

同时Netfilter/iptables框架会把对应的表、链、匹配规则和target模块对应起来，这里对应的表是mangle表、链是INPUT链，匹配规则是UDP报文。

当有符合上述条件的包时，就会回调自定义模块TEST，并传入由用户态输入的自定义参数。 

## Netfilter 
Netfilter是嵌入Linux内核协议栈的，设置在报文处理路径上的一系列调用入口。
从上文我们已经知道，Netfilter一共有5个"钩子"设置在IP协议栈的报文处理路径上。
那么内核是如何管理这些"钩子"的呢？

### "钩子"的存储及管理机制 
"钩子"函数由一个全局二维数组nf_hooks按照协议族归类存储，在每个协议族中，根据钩子点顺序排列，在钩子点内则根据钩子函数的优先级排列。
![](/images/network/netfilter_hooks.jpg)

*   这个二维数组的每一项代表了一个钩子被调用的点，NF_PROTO代表协议栈，NF_HOOK代表协议栈中某个路径点。
*   所有模块都可以通过nf_register_hook()函数将一个钩子函数挂入想要被调用点的链表中(通过Protocol和hook指定一个点)。
    这样，该钩子函数就能够处理从指定Protocol和指定hook点流过的数据包。
*   Netfilter在不同协议栈的不同点上放置钩子函数，当数据包经过某个协议栈(NF_PROTO)的某个点(NF_HOOK)时，该协议栈会通过NF_HOOK()函数调用对应钩子链表(nf_hooks\[NF_PROTO][NF_HOOK])中注册的每一个钩子项来处理该数据包。

Netfilter定义了每个钩子函数的返回值，每个钩子函数只能返回下面的返回值，而不能自定义返回值。

*   NF_DROP(0)：数据包被丢弃，即不被下一个钩子函数处理，同时也不再被协议栈处理，并释放数据包。
*   NF_ACCEPT(1)：数据包被接受，即交给下一个钩子或协议栈继续处理。
*   NF_STOLEN(2)：数据包被停止处理，即不被下一个钩子函数处理，同时也不再被协议栈处理，但不释放数据包。
*   NF_QUEUE(3)：将数据包交给nf_queue子系统处理，即不被下一个钩子函数处理，同时也不再被协议栈处理，也不释放数据包。
*   NF_REPEAT(4)：数据包将被该返回值的钩子函数再次处理一遍。
*   NF_STOP(5): 数据包停止被该HOOK点的后续钩子函数处理，交给协议栈继续处理。

### "钩子"的使用方法 
与target内核态程序编写类似，"钩子"的使用首先实例化一个nf_hook_ops对象，然后对其进行必要的初始化设置，最后通过nf_register_hook()函数将其注册到二维数组nf_hooks中。
我们首先初始化nf_hook_ops中的常用字段：

{% highlight c %}
static struct nf_hook_ops nf_hook_test_ops = 
{
    .hook     = test_hook_func;
    .hooknum  = NF_INET_PRE_ROUTING;
    .pf       = PF_INET;
    .owner    = THIS_MODULE;
    .priority = NF_IP_PRI_FIRST;
}
{% endhighlight %}
其中：

*   hook是钩子函数
*   hooknum是钩子点
*   pf是协议栈
*   priority是钩子函数的优先级

然后在模块加载和退出函数中注册和移除钩子函数：

{% highlight c %}
int init_module(void)
{
    nf_register_hook(&nf_hook_test_ops);
}
void cleanup_module() 
{
    nf_unregister_hook(&nf_hook_test_ops);
}
{% endhighlight %}
下面是回调函数的声明：

{% highlight c %}
static unsigned int test_hook(unsigned int hooknum, struct sk_buff *skb,
		const struct net_device *in, const struct net_device *out,
		int (*okfn)(struct sk_buff*)
{% endhighlight c%}
从上述过程可以看出，钩子函数的使用与iptables没有任何关系，也就是说如果某个模块需要对协议栈的报文进行处理，但不需要用户空间的参数，那么完全可以只注册钩子函数，而不需要编写iptables的模块。

即使需要用户空间的参数，也可以通过proc等其他用户态和内核态通信方式来传递参数，这样就可以更灵活的使用钩子函数了。

**注：**
----------

* [学习使用iptables](http://wangcong.org/articles/learning-iptables.cn.html)
* [Netfilter实现机制分析](http://bbs.chinaunix.net/thread-2008344-1-1.html)
* [自己写Netfilter匹配器](http://www.linuxfocus.org/ChineseGB/February2005/article367.shtml)
* [linux-2.6.35.6 xtables&iptables&hipac](http://bbs.chinaunix.net/thread-3749229-1-1.html)

