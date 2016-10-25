---
layout: post
title: Docker-registry V1源码分析——整体框架
date: 2016-10-24 23:29:30
category: 技术
tags: Docker-registry
excerpt: Docker-registry V1源码分析
---

由于Docker-registry涉及的知识和框架比较多，所以有必要对其进行梳理。主要是讲述和Docker-registry密切相关的知识以及它们之间的调用逻辑，达到对Docker-registry整体框架的一个比较清晰的了解

以Docker-registry为核心扩展，涉及的主要知识点包括：docker-registry、python-gunicorn、python-flask、python-gevent等

### 上述技术概述

#### **下面分别介绍一下上述技术：**

python-gunicorn采用pre-forker worker model，有一个master进程，它会fork多个worker进程，而这些worker进程共享一个Listener Socket，每个worker进程负责接受HTTP请求并处理，master进程只是负责利用TTIN、TTOU、CHLD等信号与worker进程通信并管理worker进程（例如：TTIN、TTOU、CHLD分别会触发master进程执行增加、减少、重启工作进程），另外每个worker进程在接受到HTTP请求后，需要加载APP对象（这个APP对象具有__call__方法，例如python-flask、django等WEB框架生成的APP对象）或者函数来执行HTTP的处理工作，将HTTP请求提取关键字（例如请求类型：GET、POST、HEAD ； 请求文件路径PATH 等）作为参数传递给APP对象或者函数，由它们来负责具体的HTTP请求处理逻辑以及操作，最后对HTTP请求回应报文进行再加工，并发送给客户端

python-flask是一个WEB框架，利用该框架可以很容易地构建WEB应用

docker-registry是一个典型的python-flask WEB应用。它是Docker hub的核心技术，所有研究的技术都围绕它展开。docker-registry主要是利用python-flask框架写一些HTTP请求处理逻辑。真正的HTTP处理逻辑和操作其实都在docker-registry中完成

python-gevent是一个基于协程的python网络库。如下是其简单描述：

![](/public/img/docker-registry/2016-10-24-docker-registry/1.png)

简单的讲就是利用python-greenlet提供协程支持，利用libevent提供事件触发，共同完成一个高性能的HTTP网络库。作为它的子技术：python-greenlet以及libevent，下面也简单描述一下：

![](/public/img/docker-registry/2016-10-24-docker-registry/2.png)

#### **下面介绍上述技术的关系，上述技术之间的关系可以这样归纳：**

docker-registry是利用python-flask写的WEB应用，负责真正的HTTP请求处理逻辑和操作

gunicorn框架中的worker进程会利用python-gevent作为HTTP处理框架，对每个HTTP请求它会加载docker-registry WEB APP对象进行具体处理，并对返回的回应报文加工，最后发送给客户端

#### **下面是上述技术的接口图：**

![](/public/img/docker-registry/2016-10-24-docker-registry/3.png)

### 代码层面的理解

下面从代码层面加深上述的理解

首先启动docker-registry，由脚本启动，如下：

```sh
service docker-registry start
```

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

![](/public/img/docker-registry/2016-10-24-docker-registry/8.png)

![](/public/img/docker-registry/2016-10-24-docker-registry/9.png)

APP_MODULE：docker_registry.wsgi:application

这些参数将作为WSGIApplication构造函数的参数，如下：

![](/public/img/docker-registry/2016-10-24-docker-registry/10.png)

WSGIApplication类继承Application类，如下：

![](/public/img/docker-registry/2016-10-24-docker-registry/11.png)

所以，WSGIApplication对象会执行Application的run函数，该函数会生成Arbiter对象，并执行该对象的run函数，如下：

![](/public/img/docker-registry/2016-10-24-docker-registry/12.png)

**<font color="#8B0000">而Arbiter类的run函数则是整个python-gunicorn的核心，它会生成worker进程，如下：</font>**

![](/public/img/docker-registry/2016-10-24-docker-registry/13.png)

manage_workers函数用于生成worker进程或者kill worker进程，使工作进程总数保持不变，如下：

![](/public/img/docker-registry/2016-10-24-docker-registry/14.png)

其中spawn_workers用于生成worker进程，kill_worker用于杀死工作进程，稍后详细介绍spawn_workers函数的逻辑

**<font color="#8B0000">同时，Arbiter类的run函数也负责管理worker进程，如下：</font>**

![](/public/img/docker-registry/2016-10-24-docker-registry/15.png)

![](/public/img/docker-registry/2016-10-24-docker-registry/32.png)

master进程会进入无限循环，利用信号与worker进程通信，从而管理worker进程，这里不展开介绍

**<font color="#8B0000">下面详细介绍worker进程产生过程，也即函数spawn_workers，该函数首先创建worker_class对象，python-gunicorn中总共有四种类型的worker进程，如下：</font>**

![](/public/img/docker-registry/2016-10-24-docker-registry/16.png)

![](/public/img/docker-registry/2016-10-24-docker-registry/17.png)

**<font color="#8B0000">其中AsyncIO Workers是推荐的worker进程类型，在如下情况要求使用该worker类型：</font>**

![](/public/img/docker-registry/2016-10-24-docker-registry/18.png)

回顾开始，启动脚本中其实已经指定了是使用AsyncIO Workers 进程的，如下：

![](/public/img/docker-registry/2016-10-24-docker-registry/19.png)

**<font color="#8B0000">就是由-k选项指定工作进程类型，gevent表示AsyncIO Workers进程类型</font>**

python-gunicorn中与该worker进程对应的处理类是gunicorn/workers/ggevent.py中GeventWorker类，也即spawn_workers函数会生成GeventWorker类对象

在生成GeventWorker类对象后，spawn_workers函数会fork子进程（也即worker进程），子进程会执行该GeventWorker类对象的init_process函数，如下：

![](/public/img/docker-registry/2016-10-24-docker-registry/20.png)

init_process函数会调用GeventWorker类的run函数，该函数对每个Listener Socket（监听套接字，例如：  0.0.0.0:5000 ）会分别创建一个python-gevent中的StreamServer类对象，并执行start函数，如下：

![](/public/img/docker-registry/2016-10-24-docker-registry/21.png)

由于server_class为None，则会执行else分支，也即会生成python-gevent中的StreamServer类对象，其中pool可以理解为利用python-greenlet生成的协程池。之后启动该对象的start函数

接下来详细介绍一下StreamServer类对象的start函数，如下：

![](/public/img/docker-registry/2016-10-24-docker-registry/22.png)

该函数会调用start_accepting函数，转到start_accepting函数，如下：

![](/public/img/docker-registry/2016-10-24-docker-registry/23.png)

该函数会利用Libevent在监听套接字上创建一个触发事件，只要有客户端请求到来则会触发调用_do_accept函数，转到_do_accept函数，如下：

![](/public/img/docker-registry/2016-10-24-docker-registry/24.png)

![](/public/img/docker-registry/2016-10-24-docker-registry/33.png)

该函数会accept请求，创建Client Socket，并创建线程处理该客户端请求，而这里的self._handle也即python-gunicorn中GeventWorker类的handle函数，跳转到handle函数，如下：

![](/public/img/docker-registry/2016-10-24-docker-registry/25.png)

![](/public/img/docker-registry/2016-10-24-docker-registry/34.png)

该函数负责请求的处理，三个参数分别表示：监听套接字、客户端Socket，客户端地址

处理逻辑为：首先分析请求内容，然后交给handle_request函数处理

转到handle_request函数，如下：

![](/public/img/docker-registry/2016-10-24-docker-registry/26.png)

![](/public/img/docker-registry/2016-10-24-docker-registry/35.png)

![](/public/img/docker-registry/2016-10-24-docker-registry/36.png)

**<font color="#8B0000">该函数会加载Docker-registry app对象，并执行__call__函数，如下：（这个很关键，是python-gunicorn与docker-registry联系所在）</font>**

![](/public/img/docker-registry/2016-10-24-docker-registry/27.png)

self.wsgi也即Docker-registry的App对象，回到最开始的脚本文件，APP模块为：docker_registry.wsgi:application，转到Docker-registry的wsgi.py文件：

![](/public/img/docker-registry/2016-10-24-docker-registry/28.png)

**<font color="#8B0000">其中app为Flask类对象，如下：</font>**

![](/public/img/docker-registry/2016-10-24-docker-registry/29.png)

之后，会调用Flask类对象的__call__函数（在python-flask中），如下：

![](/public/img/docker-registry/2016-10-24-docker-registry/30.png)

该函数会调用wsgi_app函数，其中environ为WSGI HTTP环境，主要包含请求类型（例如：GET、POST等）和请求文件URI；start_response则主要负责重加工请求回应的头部信息，最后返回值即为回应的内容

wsgi_app函数处理逻辑大致为：先从environ中取出请求类型和请求文件URI，然后根据这两个主要参数调用对应的函数处理，并生成回应报文，之后，调用start_response函数重加工回应报文头部，最后将回应内容作为函数返回值返回

最后handle_request函数还要将请求信息写入日志，如下：

![](/public/img/docker-registry/2016-10-24-docker-registry/31.png)

这样整个Docker-registry的框架就大致讲完了，以后会陆续补充，欢迎大家讨论！










