---
layout: post
title: shell命令集锦 
categories: Linux
tags: 
    - Tools
---

这里收集整理了一些Linux shell命令，以备不时之需。


**`find dir \( -type f \) -exec dos2unix '{}' \;`**

对某个目录执行dos2unix命令，使用时用实际目录替换掉命令中dir。

{% highlight sh %}
cd `dirname "$0"`
{% endhighlight %}

这个命令写在脚本文件里会返回该脚本文件所在的目录，这样就可以根据这个目录来定位一些程序的相对路径。


