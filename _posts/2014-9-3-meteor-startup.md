---
layout: post
title: Meteor应用的启动过程 
categories: Meteor
tags:
    - JavaScript
---

使用Meteor创建和运行一个应用是非常简单的，而简单的背后就是繁杂的细节。
我们希望通过分析源码，抽丝剥茧，来理解这简单背后的细节之美。

> meteor v0.9.0.1

## 运行一个应用
首先我们得创建一个应用`meteor create test`，后面的代码分析都会用到这个应用。
在Meteor中只要在应用目录中执行meteor命令就可以运行这个应用了，应用正常运行之后会有如下打印：

```
[[[[[ ~/WCode/test ]]]]]
=> Started proxy.
=> Started your app. 
=> App running at: http://localhost:3000/
```
从上面的打印看Meteor先启动proxy、然后再启动app，但是在启动proxy之前应该还做了一些事的，
比如校验传入参数、获取环境变量、加载Package和Module等等。
根据上述推测，我们暂时把应用的启动过程分为四个步骤：环境配置、加载、启动proxy和启动app。

## 入口
当我们执行`curl https://install.meteor.com/ | sh`完成meteor安装之后，
系统中会出现一个`/usr/local/bin/meteor`可执行文件和一个`~/.meteor/`目录。
`~/.meteor/`目录中包含了Meteor运行需要的所有脚本、包和模块。
`/usr/local/bin/meteor`这个Shell脚本做了两件事：

* 检查Meteor是否成功安装，没有就重新安装一遍。
* 运行`~/.meteor/meteor`。

其实这个脚本执行了一次之后，就不需要再执行这个脚本了，
完全可以把`~/.meteor/`加入到$PATH中或者创建链接来直接执行`~/.meteor/meteor`。

执行`ls ~/.meteor/meteor -al`这个命令就可以看到其实这只是个链接，
实际的文件是`~/.meteor/packages/meteor-tool/1.0.26/meteor-tool-os.linux.x86_32/meteor`。

这也是个Shell脚本，也做了两件事：检查Meteor版本和运行`exec "$DEV_BUNDLE/bin/node" "$METEOR" "$@" `
其中`DEV_BUNDLE="$SCRIPT_DIR/dev_bundle"`,`METEOR="$SCRIPT_DIR/tools/main.js"`, `$@`是输入的命令和参数
而`SRCIPT_DIR=~/.meteor/packages/meteor-tool/.1.0.26.13pjtg1++os.linux.x86_32+web.browser+web.cordova/meteor-tool-os.linux.x86_32/`

因此，meteor运行的真正入口是`main.js`，这个文件在源码中位于tools目录下。

## main.js
main.js中`Fiber(function(){...}).run()`类似于C语言中的main()函数，是所有函数的入口。
这个函数首先检查Node的版本和ROOT_URL，然后解析并校验传入的命令和参数，最后执行命令。
所有命令的实现在tools/command.js中，默认的命令是run。

