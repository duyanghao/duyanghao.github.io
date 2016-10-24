---
layout: post
title: Docker-registry V1源码分析——整体框架
category: 技术
tags: Docker Registry
keywords: Docker Registry
description: Docker-registry V1源码分析
---

由于Docker-registry涉及的知识和框架比较多，所以有必要对其进行梳理。主要是讲述和Docker-registry密切相关的知识以及它们之间的调用逻辑，达到对Docker-registry整体框架的一个比较清晰的了解

以Docker-registry为核心扩展，涉及的主要知识点包括：docker-registry、python-gunicorn、python-flask、python-gevent等

### 上述技术概述

#### 下面分别介绍一下上述技术：

python-gunicorn采用pre-forker worker model，有一个master进程，它会fork多个worker进程，而这些worker进程共享一个Listener Socket，每个worker进程负责接受HTTP请求并处理，master进程只是负责利用TTIN、TTOU、CHLD等信号与worker进程通信并管理worker进程（例如：TTIN、TTOU、CHLD分别会触发master进程执行增加、减少、重启工作进程），另外每个worker进程在接受到HTTP请求后，需要加载APP对象（这个APP对象具有__call__方法，例如python-flask、django等WEB框架生成的APP对象）或者函数来执行HTTP的处理工作，将HTTP请求提取关键字（例如请求类型：GET、POST、HEAD ； 请求文件路径PATH 等）作为参数传递给APP对象或者函数，由它们来负责具体的HTTP请求处理逻辑以及操作，最后对HTTP请求回应报文进行再加工，并发送给客户端

python-flask是一个WEB框架，利用该框架可以很容易地构建WEB应用

docker-registry是一个典型的python-flask WEB应用。它是Docker hub的核心技术，所有研究的技术都围绕它展开。docker-registry主要是利用python-flask框架写一些HTTP请求处理逻辑。真正的HTTP处理逻辑和操作其实都在docker-registry中完成

python-gevent是一个基于协程的python网络库。如下是其简单描述：

![](/public/img/docker-registry/2016-10-24-docker-registry/1.png)

简单的讲就是利用python-greenlet提供协程支持，利用libevent提供事件触发，共同完成一个高性能的HTTP网络库。作为它的子技术：python-greenlet以及libevent，下面也简单描述一下：

![](/public/img/docker-registry/2016-10-24-docker-registry/2.png)

#### 下面介绍上述技术的关系，上述技术之间的关系可以这样归纳：

docker-registry是利用python-flask写的WEB应用，负责真正的HTTP请求处理逻辑和操作

gunicorn框架中的worker进程会利用python-gevent作为HTTP处理框架，对每个HTTP请求它会加载docker-registry WEB APP对象进行具体处理，并对返回的回应报文加工，最后发送给客户端

#### 下面是上述技术的接口图：

![](/public/img/docker-registry/2016-10-24-docker-registry/3.png)

### 代码层面的理解

下面从代码层面加深上述的理解

首先启动docker-registry，由脚本启动，如下：

service docker-registry start

下面是/etc/init.d/docker-registry中的start函数

![](/public/img/docker-registry/2016-10-24-docker-registry/4.png)

很容易看出入口为/usr/bin/gunicorn，下面是/usr/bin/gunicorn的文件内容：

![](/public/img/docker-registry/2016-10-24-docker-registry/5.png)

python-gunicorn入口是wsgiapp.py的run函数，如下：

![](/public/img/docker-registry/2016-10-24-docker-registry/6.png)

逻辑很清晰：加载WSGIApplication类，生成该对象，并执行run函数。再回去看/etc/init.d/docker-registry启动脚本中的参数：

![](/public/img/docker-registry/2016-10-24-docker-registry/7.png)

可以将这些参数分为三类，分别对应prog、OPTIONS、APP_MODULE，如下：

prog：/usr/bin/gunicorn

OPTIONS：
![](/public/img/docker-registry/2016-10-24-docker-registry/7.png)




