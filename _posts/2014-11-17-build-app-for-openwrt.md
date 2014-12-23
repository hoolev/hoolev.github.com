---
layout: post
title: OpenWrt APP开发就是这么简单
categories: WiFi
tags:
    - JuBo
    - OpenWrt
---

开源项目JuBo提供了一个极简的OpenWrt APP开发环境，只需要使用JavaScript一种语言就可以完成APP的开发。
下面我们就使用JuBo来开发一个简单的OpenWrt应用(jubo-shell)，体验下愉悦的OpenWrt应用开发。

## 创建应用

使用下面的命令来创建应用：

{% highlight bash %}
~ $ jubo create jubo-shell
{% endhighlight %}

这个命令会创建一个名字为jubo-shell的目录，目录里包含了应用所需的所有文件。

{% highlight bash %}
jubo-shell.css   # a CSS file to define your app's styles
jubo-shell.html  # an HTML file that defines view templates
jubo-shell.js    # a JavaScript file loaded on both client and server
.meteor          # internal Meteor files
{% endhighlight %}

打开浏览器输入`http:/localhost:3000`就可以访问应用了，
你可以修改`jubo-shell.html`里的内容，比如`h1`标题，修改完成保存后，不需要刷新就可以在页面看到修改内容了。

对JuBo应用了有了初步体验之后，让我们开始`jubo-shell`的开发工作吧。
`jubo-shell`的源码发布在GitHub上，你可以获取[全部代码](https://github.com/jubolin/jubo-shell)。

## 使用模板
开始编写代码之前，我们必须要正确的设置项目。
为了保证项目整洁，我们首先删除`jubo-shell.html`、`jubo-shell.js`和`jubo-shell.css`。

然后在`jubo-shell`目录下新建两个子文件夹:`client`和`server`。
在`client`目录下创建文件`jubo-shell.html`，并写入以下代码：

{% highlight html %}
  <!-- jubo-shell.html -->
  <head>
    <title>jubo-shell</title>
  </head>

  <body>
    <div class="container">
      {% raw %}{{> input}}{% endraw %}
      {% raw %}{{> buffer}}{% endraw %}
    </div>
  </body>

  <template name="input">
    <div class="row">
      <div class="input-group">
        <input type="text" class="form-control" id="cmd">
        <span class="input-group-btn">
          <button class="btn btn-primary" type="button">Run</button>
        </span>
      </div>
    </div>
  </template>

  <template name="buffer">
    <div class="row" id="buffer">
      <pre>{% raw %}{{ output }}{% endraw %}</pre>
    </div>
  </template>
{% endhighlight %}

关于文件，  JuBo有以下几条规则：

* 在 /server 文件夹中的代码只会在服务器端运行。
* 在 /client 文件夹中的代码只会在客户端运行。
* 其它代码则将同时运行于服务器端和客户端上。
* 在 /lib 文件夹中的文件将被优先载入。
* 所有以 main.* 命名的文件将在其他文件载入后载入。
* 请将所有的静态文件（字体，图片等）放置在 /public 文件夹中。

> 这里的服务器端指的是OpenWrt设备，客户端指的是浏览器。

### HTML模板
Meteor会解析应用程序文件夹中的所有HTML文件，确定`<head>`，`<body>`和`<template>`三个顶级标记，并组成一个标准的HTML文件。

任何`<head>`标记包含的所有内容将被添加到HTML文件的`head`段，
同样，任何`<body>`标记包含的所有内容将被添加到HTML文件的`body`段。

Meteor模板将对`<template>`标记包含的代码进行编译，HTML可以通过`{% raw %}{{> templateName}}{% endraw %}`包含相关代码，
JavaScript文件可以通过`Template.templateName`进行引用。

#### 往模板中添加数据
我们可以通过定义`helpers`从JavaScript中传递数据给模板。
在这里我们定义了`Template.buffer.output`，并且从会话中获取数据或者使用默认数据。
这样我们就可以在HTML文件的`<template>`标记中通过`{% raw %}{{output}}{% endraw %}`来输出数据了。

#### 添加CSS
由于指南手册主要关注HTML和JavaScript ，所以为了避免在CSS细节上花费过多时间，这里提供完整的CSS文件。

{% highlight css %}
/* jubo-shell.css */
/* CSS declarations go here */
body {
  padding-top: 20px;
}

#buffer {
  margin-top: 20px;
}

#cmd {
  font-weight:bold;
  font-size: 1.5em;
  padding-left: 20px;
}

pre {
  font-size: 1.5em
}
{% endhighlight %}

## 添加事件
在client目录下创建文件`jubo-shell.js`，并写入以下代码：

{% highlight js %}
// client/jubo-shell.js
Template.buffer.output = function () {
  if (Session.get('output'))
    return Session.get('output');
  else
    return "JuBo Shell 0.0.1\n\n"
          +"This is a basic terminal, it will run Linux/Unix commands on OpenWrt device\n"
          +"and will print on the screen the output."
          +"\n\n"
          +"Notes: Can't support cd command\n"
          +"       Max output buffer is 200*1024."

}

Template.input.events({
  'click button': function(evt) {
    var cmd  = $('#cmd').val();
    Meteor.call('jsh', cmd, function (err, data) {
      $('#cmd').val('');
      Session.set('output', data );
    });
  },

  'keypress input#cmd': function (evt) {
    if (evt.which === 13) { // enter key
      var cmd  = $('#cmd').val();
      Meteor.call('jsh', cmd, function (err, data) {
        $('#cmd').val('');
        Session.set('output', data );
      });
    }
  }
});
{% endhighlight %}

我们可以通过Template.templateName.events来监听事件，这里我们监听了输入框的回车键和按钮的单击事件。
当用户按下回车或者按下按钮时，就会调用服务器端提供的`jsh`函数执行Shell命令，并通过Session输出结果。
Session 是一个全局的响应式数据存储。它全局性的意思是全局的单例对象：这个Session对象在全局都是可被访问到的。

在server目录下创建文件`jubo-shell.js`，并写入以下代码：

{% highlight js %}
// server/jubo-shell.js
var future = Npm.require('fibers/future');
var exec = Npm.require('child_process').exec;

Meteor.methods({
  jsh: function (command) {
    var shell = new future();

    exec(command,function(error,stdout,stderr){
      if(error)
        shell.return('' + error);
      else
        shell.return('' + stdout);
    });

    return shell.wait();
  }
});
{% endhighlight %}

## 发布应用
到目前为止我们已经完成了jubo-shell应用的开发，进入应用目录输入`jubo`在本地运行jubo-shell应用，
然后通过浏览器访问`127.0.0.1:3000`就可以使用本地的jubo-shell应用了。

我们也可以通过下面的命令把应用发布到无线路由器,这样就可以在PC、手机和平板上通过Shell命令来控制路由器了。

{% highlight sh %}
~ $ jubo deploy root@devip
{% endhighlight %}

`devip`是路由器的管理IP，应用发布成功后，我们就可以访问`devip:22786`来体验应用了。
应用默认是发布到`/root`目录下的，你也可以使用`root@devip:path`来指定发布路径。

