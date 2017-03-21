---
layout: post
title: linux内存实践
date: 2017-3-20 18:10:31
category: 技术
tags: Linux Memory
excerpt: 从应用方面讲解了linux内存的含义和实际使用问题……
---

本文从应用方面讲解了linux内存的含义和实际使用问题，不断补充，不断总结……

## free命令

解释一下Linux上free命令的输出

下面是free的运行结果，一共有4行。为了方便说明，我加上了列号。这样可以把free的输出看成一个二维数组FO(Free Output)。例如：

* FO[2][1] = 24677460
* FO[3][2] = 10321516

![](/public/img/resource_monitor/memory-free.png)

__free的输出一共有四行，第四行为交换区的信息，分别是交换的总量（total），使用量（used）和有多少空闲的交换区（free）__

free输出地第二行和第三行是比较让人迷惑的。这两行都是说明内存使用情况的

第一列是总量（total），第二列是使用量（used），第三列是可用量（free）

__第一行的输出是从操作系统（OS）角度来看的。也就是说，从OS的角度来看，计算机上一共有:__

* 24677460KB（缺省时free的单位为KB）物理内存，即FO[2][1]；
* 在这些物理内存中有23276064KB（即FO[2][2]）被使用了；
* 还用1401396KB（即FO[2][3]）是可用的；

这里得到第一个等式：

* FO[2][1] = FO[2][2] + FO[2][3]

FO[2][4]表示被几个进程共享的内存量，现在已经deprecated，其值总是0（当然在一些系统上也可能不是0，主要取决于free命令是怎么实现的）

FO[2][5]表示被OS buffer住的内存。FO[2][6]表示被OS cache的内存。在有些时候buffer和cache这两个词经常混用。不过在一些比较低层的软件里是要区分这两个词的，看老外的洋文:

* _A buffer is something that has yet to be "written" to disk._ 
* _A cache is something that has been "read" from the disk and stored for later use._

也就是说buffer是用于存放要输出到disk（块设备）的数据的，而cache是存放从disk上读出的数据。这二者是为了提高IO性能的，并由OS管理

Linux和其他成熟的操作系统（例如windows），为了提高IO read的性能，总是要多cache一些数据，这也就是为什么FO[2][6]（cached memory）比较大，而FO[2][3]比较小的原因。我们可以做一个简单的测试:

1、释放掉被系统cache占用的数据；

```bash

echo 3>/proc/sys/vm/drop_caches

```

2、读一个大文件，并记录时间；

3、关闭该文件；

4、重读这个大文件，并记录时间；

第二次读应该比第一次快很多

__free输出的第二行是从一个应用程序的角度看系统内存的使用情况__

* 对于FO[3][2]，即-buffers/cache，表示一个应用程序认为系统被用掉多少内存；
* 对于FO[3][3]，即+buffers/cache，表示一个应用程序认为系统还有多少内存；

因为被系统cache和buffer占用的内存可以被快速回收，所以通常FO[3][3]比FO[2][3]会大很多

这里还用两个等式：

* FO[3][2] = FO[2][2] - FO[2][5] - FO[2][6]
* FO[3][3] = FO[2][3] + FO[2][5] + FO[2][6]

补充：free命令的所有输出值都是从/proc/meminfo中读出的

## Refs

* [Setting /proc/sys/vm/drop_caches to clear cache](http://unix.stackexchange.com/questions/17936/setting-proc-sys-vm-drop-caches-to-clear-cache)
* [手工释放linux内存——/proc/sys/vm/drop_caches ](http://www.linuxfly.org/post/320/)
