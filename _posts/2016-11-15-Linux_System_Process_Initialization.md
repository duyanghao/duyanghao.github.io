---
layout: post
title: Linux System Process Initialization Mechanisms
date: 2016-11-15 19:52:31
category: 技术
tags: Linux
excerpt: this article describes three Linux startup mechanisms……
---

this article simply describes three Linux startup mechanisms.

## [Linux System Process Initialization (SysV)](http://glennastory.net/boot/init.html)

**/sbin/init process:**

1、执行/etc/rc.d/rc.sysinit([initialization tasks.](http://glennastory.net/boot/sysinit.html))

2、执行/etc/rc.d/rc(again driven by entries in /etc/inittab)

工具：chkconfig(eg:/etc/rc.d/init.d/docker,/etc/sysconfig/docker,/etc/rcX(0-6).d/S95docker)

## [Linux System Process Initialization Using Upstart (Upstart)](http://glennastory.net/boot/upstart.html)

**/sbin/init process:**

1、The upstart version of init reads a series of files from the directory /etc/init or its children. Each of these files describes a "service," typically a daemon that runs in the background to perform some system function. Here is an [example.](http://glennastory.net/boot/udevmonitor.conf.txt)

2、The init process monitors a service while it is running and can restart it if it fails.

3、Another startup file looks like [this.](http://glennastory.net/boot/rc.conf.txt) This file runs the rc script. This has the effect of running any traditional startup scripts as supported by the ["SysV" or legacy init process.](http://glennastory.net/boot/init.html)

**The real advantage of upstart over the SysV init process is that the latter mechanism runs strictly sequentially, upstart is designed to run initialization steps in parallel, as defined by the .conf files. This generates a tree of startup tasks. For example here is the [tree for Fedora 14.](http://glennastory.net/boot/upstart-start.png)**

## [Linux System Process Initialization Using Systemd (Systemd)](http://glennastory.net/boot/systemd.html)

CentOS 7 使用systemd替换了SysV。Systemd目的是要取代Unix时代以来一直在使用的init系统，兼容SysV和LSB的启动脚本，而且够在进程启动过程中更有效地引导加载服务

systemd的特性有：

* 支持并行化任务

* 同时采用socket式与D-Bus总线式激活服务

* 按需启动守护进程（daemon）

* 利用 Linux 的 cgroups 监视进程

* 支持快照和系统恢复

* 维护挂载点和自动挂载点

* 各服务间基于依赖关系进行精密控制

Whereas [SysV Init](http://glennastory.net/boot/init.html) is made entirely from scripts, and [Upstart](http://glennastory.net/boot/upstart.html) is a combination of descriptive configuration files and scripits, systemd configuration is described entirely by configuration files.

/sbin/init(In the case of systemd, /sbin/init is actually a symbolic link to another file /usr/lib/systemd/systemd.)

**/sbin/init process:**

1、The systemd version of init reads a series of files from the directories /etc/systemd/system and /usr/lib/systemd/system. Each of these files is called a "unit" and units can be of various types such as service, target, etc., as indicated by the filename suffix (.service, .target, etc.) A "service" is typically a daemon that runs in the background to perform some system function. Here is a [sample.](http://glennastory.net/boot/systemd-udevd.service.txt)

2、The init process monitors a service while it is running and can restart it if it fails.

*Runlevels as implemented in SysV init are replicated in systemd by "targets" (files with a .target suffix). Some systems provide files such as "runlevel5.target" as a symbolic link in order to mimic SysV runlevel behavior.*

**The advantage of either upstart or systemd over the SysV init process is that whereas the SysV mechanism runs strictly sequentially, upstart and systemd are designed to run initialization steps in parallel. Which of upstart or systemd is preferable is a current subject of heated debate. systemd does away with startup script code, which some people applaud since it prevents having to start shell for each service, while others object to the loss of control of the logic used to start a service.**

## [Linux Login](http://glennastory.net/boot/login.html)
……

## 参考

* [chkconfig refer1](http://www.cnblogs.com/wangtao_20/archive/2014/04/04/3645690.html)

* [chkconfig refer2](http://blog.csdn.net/taiyang1987912/article/details/41698817)

* [systemd](https://blog.linuxeye.com/400.html)
