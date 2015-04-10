---
layout: post
title: 如何消除gcc编译警告 warning "unused parameter xxxx"   
categories: 见识
tags:
    - gcc 
    - c/c++
---

"unused parameter xxxx"，这个警告一般容易出现在回调函数的实现中。
因为回调函数的声明是固定的，带了一些参数，但是在实现某个回调函数时，可能某些参数没有用到，这个时候就会出现这个警告。那么如何消除这个编译警告呢？

## 通用方法
这个方法是通用的方法，与编译器无关，移植性很好。

{% highlight c %}
#define UNUSED(x) (void)x 

int foo (int bar) 
{
	UNUSED(bar);
    return 0;
}
{% endhighlight %} 

在UNUSED(param)语句不产生任何目标代码，消除对未使用的变量的警告，并明确文件，不要使用变量的代码。

## 使用gcc unused属性
gcc提供的`__attribute__ ((unused))`属性可以消除这个警告。

{% highlight c %}
int foo (__attribute__((unused)) int bar) 
{
    return 0;
}
{% endhighlight %} 


