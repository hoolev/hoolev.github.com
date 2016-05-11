---
layout: post
title: 在Win10上发布Meteor应用 
categories: Meteor
tags:
    - JavaScript
---

在Win10发布绿色版Meteor应用，发布成功后用户不需要安装Node、Mongodb、Meteor等软件，解压缩就可以运行Meteor应用。
基本思路就是通过demeteorizer打包Meteor应用，然后通过`npm install`安装好依赖的NPM包，最后把所需要的exe和dll文件打包在一起，形成一个解压即可运行的Meteor应用压缩包。

## 环境依赖

* Windows 10
* [Visual Studio 2012](https://download.microsoft.com/download/B/2/8/B2801FEE-9A60-4AFA-8657-0E8AB0A373F0/VS2012_PRO_chs.iso)
* [Python x64 v2.7](https://www.python.org/ftp/python/2.7.11/python-2.7.11.amd64.msi) 
* [Node v0.10.40 x64](https://nodejs.org/dist/v0.10.40/x64/node-v0.10.40-x64.msi)
* demeteorizer("npm install -g demeteorizer")
* Meteor for Windows
* [MongoDB x64 v3.2.6](http://180.173.24.147/mongodb-win32-x86_64-2008plus-ssl-3.2.6-signed.msi?fid=LxnDXh3Jw3Z0QzJRJBOcCkkfrzQAjlsGAAAAAAu4UYixo-XsZ7vAZ5bBeH4jAUjQ&mid=666&threshold=150&tid=1DF2308C1714464C3F3A3DB65457B5B1&srcid=119&verno=1)

## 发布过程

1. 进入需要发布的Meteor应用目录，确认应用能够正常运行
2. 打开`Developer Command Prompt VS2012`，进入需要发布的Meteor应用目录运行`demeteorizer`
3. 进入`.demeteorizer`目录，执行`npm install --registry=https://registry.npm.taobao.org --disturl=https://npm.taobao.org/dist`
4. 执行`npm uninstall bcrypt && npm install bcrypt --registry=https://registry.npm.taobao.org`
5. 新建一个目录，目录名为应用名，目录结构如下：
6. 
{% highlight shell%}
	-/bin
	--node.exe
	--mongod.exe
	--libeay32.dll
	--ssleay32.dll
	--run64.cmd
	-/resources
	--/data
	--/bundle
	---/server
	---/programs
	---main.js
{% endhighlight %}

bin目录下的exe和DLL文件从Node、MongoDB的安装目录下拷贝，run64.cmd是程序启动脚本，下面会给出一个模板。
resources目录由.demeteorizer目录拷贝重命名而来，`/data`目录是新建目录，用来存储应用数据库。

## 启动脚本模板

{% highlight shell%}
@ECHO off
:: Basic bathc file to run a meteor app including mongod
:: Set some common variables

SETLOCAL ENABLEEXTENSIONS
SET me=%~n0
SET parent=%~dp0

:: Step 1 -- Launch mongod since this needs to be running of course
SET MONGODATA=..\resources\data\dbfolder
SET MONGOPORT=20172
SET MONGOIP=127.0.0.1
mkdir %MONGODATA%
echo %me% - Launching Mongo @ %MONGOIP%:%MONGOPORT% Data dir @ %MONGODATA%
START /b %parent%/mongod --nohttpinterface --smallfiles --bind_ip %MONGOIP% --port %MONGOPORT% --dbpath %MONGODATA%
TIMEOUT /t 5 /NOBREAK

cls
echo launch jubo
:: Now launch our application

SET MONGO_URL=mongodb://%MONGOIP%:%MONGOPORT%/jubo
SET PORT=8080
SET ROOT_URL=http://localhost:%PORT%/
cd ..\resources\bundle
%parent%/node main.js
{% endhighlight %}

## 扩展

* 如果Meteor应用需要设置启动参数，那么可以在启动脚本中通过`set METEOR_SETTINGS`来设置。
* 发布成功后，再结合node-webkit就可以发布Windows下Meteor本地应用了。




