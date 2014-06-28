---
layout: post
title: 移植openwrt到HG255D 
categories: WiFi
tags: 
    - JuBo 
---
OpenWrt是一个嵌入式的 Linux 发行版，OpenWrt不是一个单一、静态的固件，而是提供了一个可添加软件包的可写的文件系统。
>交叉编译的主机是Ubuntu

## 移植OpenWrt
###安装编译依赖包

{% highlight sh %}
apt-get install gcc g++ binutils patch bzip2 flex bison make autoconf gettext texinfo unzip 
apt-get install sharutils subversion libncurses5-dev ncurses-term zlib1g-dev
{% endhighlight %} 

###获取OpenWrt源代码和安装包
这里获取openwrt 12.09分支版本，最新版本源码路径可以去https://dev.openwrt.org/wiki/GetSource 查找

{% highlight sh %}
git clone git://git.openwrt.org/12.09/openwrt.git
{% endhighlight %} 

###修改相关文件

*  复制mach-hg255d.c到trunk/target/linux/ramips/files/arch/mips/ralink/rt305x/
*  复制hg255d.mk到target/linux/ramips/rt305x/profiles/

**编辑`trunk/target/linux/ramips/image/Makefile`**

在mtdlayout_edimax_3g6200n的上面增加:
{% highlight sh %}
mtdlayout_16M=256k(u-boot)ro,128k(u-boot-env)ro,128k(factory)ro,1024k(kernel),14848k(rootfs),15872k@0x80000(firmware),16384k@0x0(fullflash)
kernel_size_16M=1048576
rootfs_size_16M=15204352
define BuildFirmware/GENERIC_16M
$(call BuildFirmware/Generic,$(1),$(2),$(call mkcmdline,$(3),$(4),$(5)) $(call mkmtd/$(6),$(mtdlayout_16M)),$(kernel_size_16M),$(rootfs_size_16M))
endef

define BuildFirmware/GENERIC_16M/initramfs
$(call BuildFirmware/Generic/initramfs,$(1),$(2),$(call mkcmdline,$(3),$(4),$(5)) $(call mkmtd/$(6),$(mtdlayout_16M)))
endef
{% endhighlight %} 

在mtdlayout_nw718的上面增加:
{% highlight sh %}
define BuildFirmware/HG255D
$(call BuildFirmware/GENERIC_16M,$(1),hg255d,HG255D,ttyS1,57600,phys)
endef
{% endhighlight %} 

在define Image/Build/Profile/FREESTATION5的上面增加:
{% highlight sh %}
define Image/Build/Profile/HG255D
$(call Image/Build/Template/$(fs_squash)/$(1),HG255D)
endef
{% endhighlight %} 

在 $(call Image/Build/Profile/FREESTATION5,$(1))的上面增加:
`$(call Image/Build/Profile/HG255D,$(1))`

**编辑`trunk/target/linux/ramips/files/arch/mips/ralink/rt305x/Kconfig`**

在config [[RT305X_MACH_FREESTATION5的上面增加:
{% highlight sh %}
config RT305X_MACH_HG255D
bool "HuaWei HG255D board support"
select RALINK_DEV_GPIO_BUTTONS
select RALINK_DEV_GPIO_LEDS
{% endhighlight %} 

**编辑`trunk/target/linux/ramips/files/arch/mips/ralink/rt305x/Makefile`**

在obj-$(CONFIG_RT305X_MACH_FREESTATION5)上面增加:

`obj-$(CONFIG_RT305X_MACH_HG255D) += mach-hg255d.o`

**编辑`trunk/target/linux/ramips/rt305x/config-3.3`**

在CONFIG_RT305X_MACH_FREESTATION5=y上面增加:

`CONFIG_RT305X_MACH_HG255D=y`

**编辑`trunk/target/linux/ramips/files/arch/mips/include/asm/mach-ralink/machine.h`**

在RAMIPS_MACH_FREESTATION5, /* ARC Freestation5 */上面增加:

`RAMIPS_MACH_HG255D, /* HuaWei HG255D */`

**编辑`trunk/target/linux/ramips/files/arch/mips/include/asm/mach-ralink/rt305x_regs.h`**
将#define RT305X_FLASH0_SIZE (8 * 1024 * 1024)修改成为

`#define RT305X_FLASH0_SIZE (16 * 1024 * 1024)`

###开始编译
* make menuconfig
	* 选择ralink
	* 选择rt305x
	* 选择HG255D
	* 其他的自己随便选
*make V=s
PS:如果menuconfig中没有HG255D选项，删除attitude_adjustment下的tmp目录就可以了。
##烧入
编译完成后烧入,目前我使用的是lintel的uboot所以原版电信u-boot能否使用未知

>因为我只是用HG255D作为测试平台,并不正是使用，因此我不使用led和buttons所以：
>led灯和button我还没有弄,修改mach-hg255d.c那个文件可以实现在trunk下支持。
>有哪个朋友有兴趣继续可以去把那个实现。
>这个修改我将会提交给openwrt的team，但是他们是否加入trunk我不知道。
>作者：hoowa.sun@gmail.com www.freeiris.org
>感谢：
>lintel编写的支持程序
>www.openwrt.org.cn上参与我那篇帖子的无名英雄
>www.right.com.cn上参与我那篇帖子的无名英雄
>www.asterisk-mips.org上的嵌入式工程师

##编译SDK
在执行make menuconfig后，选择目标版本和Bulid the OpenWrt SDK选项进行编译。
{% highlight sh %}
Target System (Ralink RT288x/RT3xxx)
Subtarget (RT305x based boards)
Target Profile (HG255D Profile)
[*]Bulid the OpenWrt SDK
{% endhighlight %} 
所有的产品都会放在编译根目录下的bin/yourtarget/. 例如:我所编译的产物都放在./bin/ramips/下，其中文件主要有几类：

*    .bin/.trx 文件: 这些都是在我们所选的target-system的类别之下，针对不同路由器型号、版本编译的路由器固件。这些不同路由器的型号和版本是openwrt预先设置好的，我们不需要更改。至于.bin和.trx的区别，一种说法是，第一次刷路由器的时候，需要用.bin文件，如果需要再升级，则不能再使用.bin文件，而需要用.trx文件。原因是.bin是将路由器的相关配置信息和.trx封装在一起而生成的封包，也就是说是包含路由器版本信息的.trx。在第一次刷固件的时候，我们需要提供这样的信息，而在后续升级时，则不再需要，用.trx文件即可。

*    packages文件夹: 里面包含了我们在配置文件里设定的所有编译好的软件包。默认情况下，会有默认选择的软件包。

*    OpenWrt-SDK.**.tar.bz2:  这个也就是我们定制编译好的OpenWRT SDK环境。我们将用这个来进行OpenWrt软件包的开发。
目前我们编译出来的是OpenWrt-SDK-ramips-for-linux-i486-gcc-4.6-linaro_uClibc-0.9.33.2.tar.bz2

*    md5sums 文件: 这个文件记录了所有我们编译好的文件的MD5值，来保证文件的完整性。因为文件的不完整，很容易将路由器变成“砖头”。

需要注意的是，编译完成后，一定要将编译好的bin目录进行备份（如果里面东西对你很重要的话），因为在下次编译之前，执行make clean 会将bin目录下的所有文件给清除掉!!


