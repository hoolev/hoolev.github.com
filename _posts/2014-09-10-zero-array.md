---
layout: post
title:  结构体零长度数组的作用
categories: program
tags:
    - C/C++
---


在一些 C 语言编写的代码中，有时可以看到如下定义的结构：

{% highlight c%}
typedef struct
{
    char * name;
    int length;
    char data[0];
} UserDefine;
{% endhighlight %}

这里的data是什么意思？我们知道 0 == sizeof(data)，那么data仅仅是为了定义结构的尾地址吗？不是的。这里的data是作为扩展数组用的。这个常用技巧常用来构成缓冲区：数组名就代表了该结构体后面数据的起始地址（而且无需初始化，不占空间）。请看如下代码：

{% highlight c%}
int AllocUserDefine(UserDefine * p, int length)
{
     p = (UserDefine)malloc(sizeof(UserDefine) + length);
     p->name = NULL;
     p->length = length;
     memset(p->data, 0, length);
	 return 0;
}
{% endhighlight %}

在结构体中，我们定义了0长度的数组，按理清空p->data属于越界访问，但是我们把结构体后面的length个长度的空间也一起申请了，所以该访问是合法的。</br>

总结：在某一结构末尾如定义类似 char data[0] 的零长数组，表示该结构不定长，可通过数组的方式进行扩展。结构中必包含一个长度信息。结构本身类似于一个信息头。同时，此结构只能通过堆方式分配内存。</br>

**PS**:对于char data[0]的定义，某些编译器不支持长度为0的数组的定义，在这种情况下，只需将其定义为char data[1]就行了，使用方法相似。
