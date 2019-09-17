---
layout: post
title: RabbitMQ备份还原方案
date: 2019-11-6 19:10:31
category: 技术
tags: database bur
excerpt: 本文介绍RabbitMQ的备份还原方案……
---

## 前言

`RabbitMQ`包含两种数据类型：Definitions (Topology)+Messages：

* Definitions (Topology)：Nodes and clusters store information that can be thought of schema, metadata or topology. Users, vhosts, queues, exchanges, bindings, runtime parameters all fall into this category.
* Messages：Each node has its own data directory and stores messages for the queues that have their master hosted on that node. Messages can be replicated between nodes using queue mirroring. Messages are stored in subdirectories of the node's data directory.

它们的生命周期不同：`Definitions`通常是静态的，生命周期长；而`messages`通常是动态的，频繁地从生产者流向消费者，生命周期短：

>> When performing a backup, first step is deciding whether to back up only definitions or the message store as well. Because messages are often short-lived and possibly transient, backing them up from under a running node is highly discouraged and can lead to an inconsistent snapshot of the data.
>> Definitions can only be backed up from a running node.

本文针对`Definitions`备份还原进行叙述

## Definitions 备份

`Definitions`备份一般有两种方式：

* 1、definition export/import: 将`Definitions`导出为JSON file
* 2、backed up manually: 直接copy `Definitions` 数据目录

官方推荐采用第一种方式，也即：`definition export/import`，而这种方式也有三种具体实现：

* 1、There's a definitions pane on the Overview page
* 2、rabbitmqadmin provides a command that exports definitions
* 3、The GET /api/definitions API endpoint can be invoked directly

这里我们用`API`方式进行备份：

```bash
curl -u xxx:xxx http://xxx/api/definitions -o definitions.json
```

## Definitions 还原

`Definitions`还原和备份一样，也有两种方式，这里用`definition export/import`方式：

* 1、There's a definitions pane on the Overview page
* 2、rabbitmqadmin provides a command that imports definitions
* 3、The POST /api/definitions API endpoint can be invoked directly

这里我们用`API`方式进行备份：

```bash
curl -H "Content-Type: application/json" -X POST -u xxx:xxx http://xxx/api/definitions -d @definitions.json
```

## Refs

* [Backup and Restore](https://www.rabbitmq.com/backup.html)
* [Management Command Line Tool](https://www.rabbitmq.com/management-cli.html)