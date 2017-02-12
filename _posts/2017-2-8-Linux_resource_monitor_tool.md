---
layout: post
title: Linux Resource Monitor Tools
date: 2017-2-8 19:10:31
category: 技术
tags: Linux
excerpt: 本文介绍了几种实用的Linux系统资源监控工具……
---

## 前言

Linux本文介绍了几种实用的Linux系统资源监控工具，主要用于监控CPU、内存、磁盘、和网卡……

## 资源及对应工具

### CPU

#### top

##### top输出

top命令如下：

![](/public/img/resource_monitor/top.png)

下面详细讲解`top`命令各参数含义：

* 第一行（CPU负载）

10:01:23 — 当前系统时间

126 days, 14:29 — 系统已经运行了126天14小时29分钟（在这期间没有重启过）

2 users — 当前有2个用户登录系统（用`w`命令验证）

load average: 1.15, 1.42, 1.44 — load average后面的三个数分别是1分钟、5分钟、15分钟的负载情况(load average数据是每隔5秒钟检查一次活跃的进程数，然后按特定算法计算出的数值。如果这个数除以逻辑CPU的数量，结果高于5的时候就表明系统在超负荷运转了)

* 第二行（进程状态）

Tasks — 任务（进程），系统现在共有183个进程，其中处于运行中的有1个，182个在休眠（sleep），stoped状态的有0个，zombie状态（僵尸进程）的有0个

* 第三行（CPU使用比例）

6.7% us — 用户空间占用CPU的百分比

0.4% sy — 内核空间占用CPU的百分比

0.0% ni — 改变过优先级的进程占用CPU的百分比

92.9% id — 空闲CPU百分比

0.0% wa — IO等待占用CPU的百分比

0.0% hi — 硬中断（Hardware IRQ）占用CPU的百分比

0.0% si — 软中断（Software Interrupts）占用CPU的百分比

* 第四行（内存使用）

8306544k total — 物理内存总量（8.3GB）

7775876k used — 使用中的内存总量（7.7GB）

530668k free — 空闲内存总量（530M）

79236k buffers — 缓存的内存量 （79M）

* 第五行（swap交换分区）

2031608k total — 交换区总量（2GB）

2556k used — 使用的交换区总量（2.5M）

2029052k free — 空闲交换区总量（2GB）

4231276k cached — 缓冲的交换区总量（4GB）

* 第七行以下：各进程（任务）的状态监控

PID — 进程id

USER — 进程所有者

PR — 进程优先级

NI — nice值。负值表示高优先级，正值表示低优先级

VIRT — 进程使用的虚拟内存总量，单位kb。VIRT=SWAP+RES

RES — 进程使用的、未被换出的物理内存大小，单位kb。RES=CODE+DATA

SHR — 共享内存大小，单位kb

S — 进程状态。D=不可中断的睡眠状态 R=运行 S=睡眠 T=跟踪/停止 Z=僵尸进程

%CPU — 上次更新到现在的CPU时间占用百分比

%MEM — 进程使用的物理内存百分比

TIME+ — 进程使用的CPU时间总计，单位1/100秒

COMMAND — 进程名称（命令名/命令行）

##### top视图

* top视图1（多U多核CPU监控）

在top基本视图中，按键盘数字“1”，可监控每个逻辑CPU的状况：

![](/public/img/resource_monitor/top-1.png)

观察上图，服务器有16个逻辑CPU，实际上是4个物理CPU

* top视图2（打开/关闭加亮效果）

敲击键盘“b”（打开/关闭加亮效果），top的视图变化如下：

![](/public/img/resource_monitor/top-y.png)

* top视图3（运行态进程加亮显示）

可以通过敲击“y”键关闭或打开运行态进程的加亮效果：

![](/public/img/resource_monitor/top-y.png)

我们发现进程id为10704的“top”进程被加亮了，top进程就是视图第二行显示的唯一的运行态（runing）的那个进程

* top视图4（排序列的加亮显示）

敲击键盘“x”（打开/关闭排序列的加亮效果），top的视图变化如下：

![](/public/img/resource_monitor/top-x.png)

可以看到，top默认的排序列是“%CPU”

* top视图5（向右或左改变排序列）

通过”shift + >”或”shift + <”可以向右或左改变排序列，下图是按一次”shift + >”的效果图：

![](/public/img/resource_monitor/top-shift.png)

视图现在已经按照%MEM来排序了

* top视图6（改变进程显示字段）

敲击“f”键，top进入另一个视图，在这里可以编排基本视图中的显示字段：

![](/public/img/resource_monitor/top-f.png)

这里列出了所有可在top基本视图中显示的进程字段，有”*”并且标注为大写字母的字段是可显示的，没有”*”并且是小写字母的字段是不显示的。如果要在基本视图中显示“CODE”和“DATA”两个字段，可以通过敲击“r”和“s”键：

![](/public/img/resource_monitor/top-f-r_s.png)

##### top命令补充

1、top命令是Linux上进行系统监控的首选命令，但有时候却达不到我们的要求，比如当前这台服务器，top监控有很大的局限性。这台服务器运行着websphere集群，有两个节点服务，就是【top视图 01】中的老大、老二两个java进程，top命令的监控最小单位是进程，所以看不到我关心的java线程数和客户连接数，而这两个指标是java的web服务非常重要的指标，通常我用ps和netstate两个命令来补充top的不足

* 监控java线程数：

`ps -eLf | grep java | wc -l`

* 监控网络客户连接数：

`netstat -n | grep tcp | grep port | wc -l`

上面两个命令，可改动grep的参数，来达到更细致的监控要求

2、在Linux系统“一切都是文件”的思想贯彻指导下，所有进程的运行状态都可以用文件来获取。系统根目录/proc中，每一个数字子目录的名字都是运行中的进程的PID，进入任一个进程目录，可通过其中文件或目录来观察进程的各项运行指标，例如task目录就是用来描述进程中线程的，因此也可以通过下面的方法获取某进程中运行中的线程数量（PID指的是进程ID）：

`ls /proc/PID/task | wc -l`

3、在linux中还有一个命令pmap，来输出进程内存的状况，可以用来分析线程堆栈：

`pmap PID`

### Memory

#### free

##### free输出

![](/public/img/resource_monitor/memory-free.png)

### IO

#### iostat

### Network

#### sar

##### 查看网卡命令

`ethtool eth1`

##### sar输出

`sar -n DEV 1`输出如下：

![](/public/img/resource_monitor/sar-dev.png)

## 参考

* [TOP命令](http://www.jb51.net/article/40807.htm)




