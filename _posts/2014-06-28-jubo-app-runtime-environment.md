---
layout: post
title: JuBo应用运行环境 
categories: JuBo
tags: 
    - JuBo 
    - Meteor
---

JuBo应用的运行环境是一个全新的、实时的、完整的web应用开发部署平台。
类似于一个运行多个服务的web服务器，JuBo应用运行环境的后端代码负责运行主要功能，前端代码负责配置和显示运行结果。

## JuBo应用特点 

* 统一的实现语言(HTML + CSS + JavaScript)
* 远程配置，通过浏览器配置管理应用的行为
* 远程监控，通过浏览器查看应用的运行结果
* 本地运行，主要功能运行在本地
* 多APP同时运行

但是与传统的WEB服务器不一样的是：

- 前端和后端统一，APP开发者只需要掌握一种开发语言并且前端和后端开发方式和思维统一。
- 运行结果实时刷新，不需要用户手动刷新页面才能看到新的结果。
- 一个完整的全栈式开发平台，便于应用的开发和管理。
从上面的特殊要求来看，传统的LMAP解决方案不能满足，我们需要一个全新的、实时的、完整的web应用开发部署平台。
幸运的是我们找到了一个合适的框架——Meteor。

## Meteor 
Meteor是一个基于JavaScript的web框架，它的目的是简化实时web应用程序的开发。

Meteor使用一个名为分布式数据协议 (Distributed Data Protocol, DDP) 的协议来处理实时通信，浏览器到服务器的通信是透明的。
DDP 协议旨在处理 JavaScript Serialized Object Notation (JSON) 文档集合，使 JSON 文档容易创建、更新、删除、查询和访问。

Meteor 提供了两个 MongoDB 数据库：一个客户端缓存数据库和服务器上的一个 MongoDB 数据库。
当一个用户更改一些数据时，在浏览器中运行的JavaScript代码会更新本地 MongoDB 中的相应的数据库项，然后向服务器发出一个DDP请求。该代码立即像操作已获得成功那样继续运行，因为它不需要等待服务器回复。与此同时，服务器在后台更新。如果服务器操作失败或返回一个意外结果，那么客户端 JavaScript 代码会依据从服务器新返回的数据立即进行调整。这种调整称为延迟补偿，向用户提供了更高的认知速度。

### 解决方案 
根据Meteor的官方示例，我们可以很快的实现一个web应用，但是我们需要构建一个应用运行环境，那么我们还需要其他的工具和技术配合。

类似于www.meteor.com的免费应用部署环境，每个web应用都有一个二级域名对应，每个web应用运行在不同的服务端口，通过反向代理机制将域名和服务端口对应起来。

那么，首先我们需要创建一个管理框架，这个管理框架也是一个Meteor应用。
{% highlight sh %}
    mkdir jubo
    cd jubo
	meteor create manager
{% endhighlight %} 

然后创建其他Meteor应用，这里仅仅是举例说明，因此可以直接使用Meteor的例子。
{% highlight sh %}
    cd jubo
    meteor create --example todos
	meteor create --example leaderboard
	meteor create --example wordplay
{% endhighlight %} 

配置反向代理，这里使用Nginx来做反向代理，其实Meteor自带了一个代理服务器node-http-proxy，但是从产品成熟度考虑选择了Nginx。
配置nginx.conf，并启动Nginx。
{% highlight sh %}
user www www;
worker_processes  1;

error_log  logs/error.log;
pid        logs/nginx.pid;


events {
    use epoll;
    worker_connections  1024;
}


http {
    include       mime.types;
    default_type  application/octet-stream;

    sendfile        on;

    keepalive_timeout  65;

    gzip  on;
    client_max_body_size 50m;
    client_body_buffer_size 256k;
    client_header_timeout 3m;
    client_body_timeout 3m;
    send_timeout 3m;
    proxy_connect_timeout 300s;
    proxy_read_timeout 300s;
    proxy_send_timeout 300s;
    proxy_buffer_size 64k;
    proxy_buffers 4 32k;
    proxy_busy_buffers_size 64k;
    proxy_temp_file_write_size 64k;
    proxy_ignore_client_abort on;

    upstream todos {
        server 127.0.0.1:4000;
    }

    upstream jubo {
        server 127.0.0.1:3000;
    }

    upstream lb {
        server 127.0.0.1:5000;
    }

    upstream wordplay {
        server 127.0.0.1:6000;
    }

    server {
        listen 80;
        location / {
            if ($host ~ 'todos.localhost.com') {
                proxy_pass http://todos;
                break;
            }

            if ($host ~ 'lb.localhost.com') {
                proxy_pass http://lb;
                break;
            }

            if ($host ~ 'wordplay.localhost.com') {
                proxy_pass http://wordplay;
                break;
            }

            if ($host ~ 'localhost.com') {
                proxy_pass http://jubo;
                break;
            }

            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
        }
    }
}
{% endhighlight %} 

配置/etc/hosts解析临时域名
{% highlight sh %}
    127.0.0.1 localhost.com
    127.0.0.1 todos.localhost.com
    127.0.0.1 lb.localhost.com
    127.0.0.1 wordplay.localhost.com
{% endhighlight %} 

使多个Meteor应用共用一个MongoDB
{% highlight sh %}
    mongod --dbpath=/data --logpath=/mongodb.log
    export MONGO_URL=mongodb://127.0.0.1:27017/local #local是mongodb的默认db，可以修改
{% endhighlight %} 

运行Meteor应用
{% highlight sh %}
    cd jubo/manager
	meteor --port 3000 &
	cd jubo/todos 
	meteor --port 4000 &
	cd jubo/leaderboard
	meteor --port 5000 &
	cd jubo/wordplay
	meteor --port 6000 &
{% endhighlight %} 

现在就可以通过todos.localhost.com等域名来访问对应的应用了。

    
