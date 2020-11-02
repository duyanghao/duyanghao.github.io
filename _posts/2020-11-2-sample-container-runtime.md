---
layout: post
title: sample container runtime
date: 2020-11-2 19:10:31
category: 技术
tags: Kubernetes Docker runc container-runtime
excerpt: 本文描述了sample-container-runtime初始版本的核心实现细节
---

## 前言

无论是虚拟化技术还是容器技术都是为了最大程度解决母机资源利用率的问题。虚拟化技术利用Hypervisor(运行在宿主机OS上)将底层硬件进行了虚拟，使得在每台VM看来，硬件都是独占的，并且由VM Guest OS直接操作(具备最高操作权限)；而容器共享母机OS，每个容器只包含应用以及应用所依赖的库和二进制文件；宿主机内核的namespace隔离特性，cgroups(资源控制)，以及联合文件系统使得多个容器之间相互隔离，同时文件系统视图也各不相同。总的来说：容器技术相比虚拟机而言，更加轻量级，同时也具备更高的执行效率

![](/public/img/sample-container-runtime/docker-vs-vm.png)

对于容器来说，最具有代表性的项目就是Docker。Docker自2013年由DotCloud开源后，便席卷整个容器技术圈。它通过设计和封装用户友好的操作接口，使得整个容器技术使用门槛大大降低；同时它也系统地构建了应用打包(Docker build)，分发(Docker pull&push)标准和工具，使得整个容器生命周期管理更加容易和可实施。对于Docker来说，可以简单的认为它并没有创造新的技术，而是将内核的namespace(进程隔离)，cgroups(进程资源控制)，Capabilities，Apparmro以及seccomp(安全防护)，以及联合文件系统进行了组合，并最终呈现给用户一个可操作和管理的容器引擎

![](/public/img/sample-container-runtime/docker-life.png)

这里为了研究容器技术，我在参考了阿里云三位同学编写的《自己动手写Docker》这本书后，基于[mydocker](https://github.com/xianlubird/mydocker/tree/code-6.5)项目开始编写自己的容器运行态，希望能更加贴近容器本质，并计划补充mydocker没有涉及的OCI，CRI等部分以及一些高级命令

## namespace隔离



## cgroups控制

## aufs

## 容器进阶

## 容器网络

## Roadmap

## Conclusion

## Refs

* [sample-container-runtime](https://github.com/duyanghao/sample-container-runtime)