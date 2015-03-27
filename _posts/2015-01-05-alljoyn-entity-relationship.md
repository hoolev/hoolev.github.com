---
layout: post
title: Alljoyn中实体之间的关系
categories: 物联网
tags:
    - Alljoyn
---

Alljoyn中实体众多，概念一下子也难理解，因此弄清楚各层次实体之间的关系就比较重要了。
![](/images/iot/alljoyn-entity-relationship.jpg)

* 一个device包含一个或多个application
* 一个Well-Known-Name标识一个application 
* 一个application包含一个或多个object
* 同一个application下的每个object都有一个唯一的object path (例如: /MyApp/Refrigerator)
* 一个object可以包含一个或多个interface

* 一个application提供一个或多个service
* 一个service可以包含一个或多个interface

* **一个service可以由一个或多个object实现**
* **一个object也可以实现一个或多个service**

* 每个ineterface都有一个唯一的interface name (例如: org.alljoyn.refrigerator)
* 一个interface包含一个或多个method、signal、property


