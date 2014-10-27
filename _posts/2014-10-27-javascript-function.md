---
layout: post
title: JavaScript多种函数定义方法及其区别
categories: program
tags:
    - JavaScript
---

在Javascript中定义一个函数一般有函数声明式、函数变量式和Function()构造函数三种方法。

## 函数声明式
所谓的函数声明式就是使用function语句来定义函数：

{% highlight js%}
function funcname([args]){
  statements
}
{% endhighlight %}

funcname 是要定义的函数名，是一个标识符，而不是字符串或者表达式；
紧跟函数名后面的是用括号括起来的参数列表，参数之间用逗号隔开；
最后，函数体，它是由一行或者多行代码组成并且是用大括号括起来的。

## 函数变量式
函数变量式也叫函数直接量，通过JavaScript的表达式创建，而不是通过语句创建。

`var f2 = function(){}`

函数直接量是一个表达式，即可以定义一个匿名函数，也可以定义一个命名函数。
函数直接量对于定义那些只使用一次，而且不需要命名的函数特别方便。

## Function()构造函数
Function()构造函数定义的是匿名函数：

`var myfunc = new Function ('x', 'y', 'alert(x+y)');`

Functino()构造函数可以接受任意多个字符串参数，最后一个参数是函数体。函数体中可以包含任何JavaScript语句，语句之间用分号分隔。
如果没有参数，传一个函数体就行了。

Function()构造函数允许我们动态地建立和编译一个函数，而不是限制在function语句预编译的函数体中。
每次调用Function()定义的函数都需要对它进行编译，因此该方式不宜在循环体中使用，尤其是递归函数，否则会大幅影响性能。

## 区别
从作用域上来说，函数声明式和函数直接量使用的是局部变量，而 Function()构造函数却是全局变量。

从性能上来说，Function()构造函数的效率要低于其他两种方式，尤其是在循环体中。

从加载顺序上来说，函数声明式在JavaScript编译的时候就加载到作用域中,而其他两种方式则是在代码执行的时候加载，如果在定义之前调用，则会返回undefined。




