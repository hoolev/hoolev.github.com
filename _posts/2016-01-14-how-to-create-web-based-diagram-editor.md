---
layout: post
title: 如何创建一个基于WEB的图形编辑器
categories: 物联网
tags:
    - D3
    - JavaScript
---

本系列文章将会说明如何使用D3.js创建一个基于WEB的，具备添加、删除、移动、复制和粘贴节点及连接线，调整布局功能的工作流编辑器。
![](/images/jubo/diagram-editor.png)

## 基础知识

* JavaScript + HTML + CSS基础知识
* [D3.js入门教程](http://www.ourd3js.com/wordpress/?cat=2)，一系列简单易懂的D3.js教程，分为入门和进阶两个部分。
* [D3.js的文档](https://github.com/mbostock/d3/wiki)，必不可少的官方文档，权威但是也比较难理解。
* 简单的Meteor知识，推荐阅读[《Discover Meteor》](http://zh.discovermeteor.com/)

## 实现
主要从以下这几个方面来收集资料，实现功能：

* 画布，即如何把多个图形渲染在指定的位置。参考 http://bl.ocks.org/explunit/4659227
* 连接，如何把各个图形之前通过曲线连接起来。参考 http://bl.ocks.org/explunit/5603250
* 冲突检测，如何在添加图形时使图形不重叠或覆盖，参考 http://bl.ocks.org/mbostock/3231298
* 画笔区域选择，参考 http://bl.ocks.org/musically-ut/4747894
* 限定范围的图形拖动，参考 http://bl.ocks.org/mbostock/1557377

后续文章会围绕这5个方面进行说明和实现，最终实现一个简单的用于物联网工作流编辑器。

