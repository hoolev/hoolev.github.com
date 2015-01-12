---
layout: post
title: 使用wrapAsync封装异步接口 
categories: Meteor
tags:
    - JavaScript
---

Meteor提供的wrapAsync接口可以把异步函数封装成同步函数。
封装后的函数在服务端既可以作为异步函数(传入回调函数)也可以作为同步函数使用(不传入回调函数)。
在客户端(浏览器端)还是需要传入回调函数作为异步函数使用。

> Meteor 1.0.2

Node.js中函数是异步的，但是我们习惯了同步方式的编程方法，因此在某些情况我们需要把异步函数封装成同步函数来使用。
wrapAsync就提供了这个功能，wrapAsync的使用很简单，实现一个异步函数，然后直接调用wrapAsync进行封装。

{% highlight js%}
var testAsync = function(name,cb) {
  setTimeout(function() {
	  cb && cb(null,'Hey there, ', + name);
  },2000);
};

var test = Meteor.wrapAsync(testAsync);
{% endhighlight %}

这里需要注意的是需要封装的异步函数(testAsync)里一定得调用回调函数，如果没有调用回调函数，
那么使用同步函数(test)时程序就会一直阻塞住。

