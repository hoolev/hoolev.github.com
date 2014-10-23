---
layout: post
title: Node.js中exec和spawn方法的区别
categories: program
tags:
    - JavaScript
---

Node.js的Child Processes模块(`child_process`)中，有两个类似的方法exec和spawn，都是通过生成子进程去执行指定的命令。两个方法除了使用方法稍有不同外，最大的区别就是二者的返回值不一样。
`child_process.exec`返回一个buffer，而`child_process.spawn`返回一个stream。

## exec方法
`child_process.exec`返回整个子进程处理时产生的buffer，这个buffer默认大小是200K。
当子进程返回的数据超过默认大小时，程序就会产生"Error: maxBuffer exceeded"异常。
调大exec的maxBuffer选项可以解决这个问题，不过当子进程返回的数据太过巨大的时候，这个问题还会出现。
因此当子进程返回的数据超过默认大小时，最好的解决方法是使用spawn方法。

## spawn方法
`child_process.spawn`返回`stdout`和`stderr`流对象。
程序可以通过`stdout`的`data`、`end`或者其他事件来获取子进程返回的数据。
使用spawn方法时，子进程一开始执行就会通过流返回数据，因此spawn适合子进程返回大量数据的情形。

与exec相比，spawn还可以对子进程进行更详细的设置。
例如使子进程在后台运行，成为一个daemon程序，不随着父进程的退出而退出。

{% highlight js%}
var app = spawn('node','main.js' {env:{detached:true}});
app.stderr.on('data',function(data) {
  console.log('Error:',data);
});

app.stdout.on('data',function(data) {
  console.log(data);
});

process.exit(0);
{% endhighlight %}

在执行子进程的时候设置`detached:true`选项，这样在父进程退出时，子进程不会退出继续执行。

## 总结
从实现原理来说，spawn是更底层的接口，exec对spawn进行了再次封装，提供了更简单的API接口。

* exec比spawn易于使用，当子进程返回的数据不超过200K时，exec比spawn更适合。
* 当子进程需要返回大量数据时，spawn更安全。
* spawn提供了更多的选项，可以对子进程进行更详细的设置。

