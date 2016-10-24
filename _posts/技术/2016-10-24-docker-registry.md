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
![](/public/img/docker-registy/1.png)






