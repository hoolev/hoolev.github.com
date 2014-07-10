---
layout: post
title: 使用Laika测试Meteor APP 
categories: Meteor
tags:
    - JavaScript
---

Meteor目前有[RTD](http://xolvio.github.io/rtd/)、[Laika](http://arunoda.github.io/laika/)和[Safety-Harness](http://safety-harness.meteor.com)三个测试框架。
RTD功能齐全，既可以做单元测试，还可以做验收测试，并且还有测试覆盖率报告。Laika是个轻量级的框架，简单易用，对集成测试支持很好。Safety-Harness可以认为是RTD和Laika功能的集合体，作者的目的是构建一个Web化、持续集成、报告清晰对业务友好的测试框架。[这里](http://safety-harness.meteor.com/comparison)是这三个测试框架的详细比较。

从功能上看，Safety-Harness是最佳的选择，但是Safety-Harness版本较新，文档也不多。
Laika简单易用，文档丰富，相较而言RTD就没有那么快上手，综合考虑选择Laika。

## 创建应用
安装应用需要用到的代码包和Laika所需的组件：

{% highlight sh %}
sudo npm install -g meteorite
mrt add bootstrap-3

sudo npm install -g laika
wget https://bitbucket.org/ariya/phantomjs/downloads/phantomjs-1.9.7-linux-i686.tar.bz2
cp phantomjs-1.9.7-linux-i686.tar.bz2 /opt
tar xvf phantomjs-1.9.7-linux-i686.tar.bz2
ln phantomjs /opt/bin/phantomjs
{% endhighlight %}

使用`meteor create tes下texample`命令创建一个应用，这个命令会创建一个名为testexample的目录，删除目录中的文件和目录，重新创建`client`、`server`和`tests`三个目录。

在Meteor应用中，`client`目录中的文件只会在客户端执行(浏览器)中运行，`server`目录中的文件只会在服务端执行。
`tests`目录用来保存测试文件，该目录下的文件在发布应用时不会包含进应用中。

### 添加HTML页面

{% highlight html %}
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <meta name="description" content="">
  <meta name="author" content="">
  <title>One Question. Several Answers.</title>
  <link rel="stylesheet" type="text/css" href="http://netdna.bootstrapcdn.com/bootswatch/3.0.3/yeti/bootstrap.min.css">
</head>

<body>
  <div class="container">
    <h1>Add an answer. Or vote.</h1>
    <h3><em>Question</em>: Is the world getting warmer?</h3>
    <br>
    <div>
      <!-- if there is an answer, append it to the DOM -->
      {{> addAnswer}}
    </div>
  </div>
</body>

<template name="addAnswer">
  <textarea class="form-control" rows="3" name="answerText" id="answerText" placeholder="Add Your Answer .."></textarea>
  <br>
  <input type="button" class="btn-primary add-answer btn-md" value="Add Answer"/>
</template>

{% endhighlight %}

![](/images/jubo/test_with_meteor_html.png)

### 添加客户端脚本

{% highlight JavaScript %}
Answers = new Meteor.Collection("answers");

Template.addAnswer.events({
  'click input.add-answer' : function(e){
    e.preventDefault();
    var answerText = document.getElementById("answerText").value;
    Meteor.call("addAnswer",answerText,function(error , answerId){
      console.log('Added answer with ID: '+answerId);
    });
    document.getElementById("answerText").value = "";
  }
});
{% endhighlight %}

我们监听一个click事件，从输入框中获取答案，然后通过`Meteor.call`调用服务端提供的`addAnswer`方法把答案添加到数据库中。

### 添加服务端脚本

{% highlight JavaScript %}
Answers = new Meteor.Collection("answers");

Meteor.methods({
  addAnswer : function(answerText){
    console.log('Adding Answer ...');
    var answerId = Answers.insert({
      'answerText' : answerText,
      'submittedOn': new Date()
    });
    console.log(answerId)
    return answerId;
  }
})
{% endhighlight %}

服务端提供了`addAnswer`方法把客户端传来的答案添加MongoDB中，并返回`answerID`给客户端。

## 自动测试
在`tests`目录下创建一个名为`TestAnswers.js`的文件

{% highlight JavaScript %}
var assert = require('assert');

suite('submitAnswers', function() {

  // ensure that -
  // (1) the "Answers" collection exists
  // (2) we can connect to the collection
  // (3) the collection is empty
  test('server initialization', function(done, server) {
    server.eval(function() {
      var collection = Answers.find().fetch();
      emit('collection', collection);
    }).once('collection', function(collection) {
      assert.equal(collection.length, 0);
      done();
    });
  });

  // ensure that -
  // (1) we can add data to the collection
  // (2) after data is added, we can retrieve it
  test('server insert : OK', function(done, server, client) {
    server.eval(function() {
      Answers.insert({answerText: "whee!"  });
      var collection = Answers.find().fetch();
      emit('collection', collection);
    }).once('collection', function(collection) {
      assert.equal(collection.length, 1);
      done();
    });

    client.once('collection', function(collection) {
      assert.equal(Answers.find().fetch().length, 1);
      done();
    });
  });

});
{% endhighlight %}

* 我们首先加载了NodeJS的assert模块
* 接着定义了一个`submitAnswers`测试套件
* 套件里包含了两个测试用例'server initialization'和'server insert : OK'
* 我们使用`test()` 方法创建测试用例，该方法接收测试用例名和一个回调函数两个参数
* 回调函数中的server和client参数可以认为分别代表了服务端和客户端
* `.eval`可以用来添加评估代码，通俗点说就是设置测试的输入
* `emit`用来发送设置好的测试输入
* `.once`和`.on`收到`emit`发送的测试输入进行判断是否满足测试要求

以测试用例'server initialization'举例来说，该用例通过'server.eval'在服务端的数据库中查找了answer并把结果通过`emit`发送出来,
最后通过`.once`接收并判断结果是否满足测试条件。

**运行测试**
{% highlight JavaScript %}
laika

  injecting laika...
  loading phantomjs...
  loading initial app pool...


  submitAnswers
     server initialization (1517ms)
     server insert : OK


  2 passing (2s)

  cleaning up injected code
{% endhighlight %}







