---
layout: post
title: redis zset实现排行榜
date: 2018-11-1 16:33:31
category: 技术
tags: 业务开发
excerpt: 本文介绍了redis zset排行榜应用……
---

## 排行榜介绍

在业务开发过程中会遇到很多排行榜类似的需求，比如粉丝排行榜，播放排行榜等，排行榜本身含义非常清晰，就是维护一个列表，其中每个条目有一个score，依据score进行排序，形成排行榜

## redis zset

redis zset由于如下特性天生支持排行榜需求：

* 1. zset是有序集合类型

* 2. zset支持常用的添加，删除，范围读取操作

下文将详细展开介绍：利用`redis zset`实现排行榜功能

### redis zset实现排行榜功能

现在假定有如下排行榜需求：实现一个粉丝TOP10排行榜。简单分析该需求，我们可以将用户id作为member，用户粉丝量作为score，并设置`redis key`为`fans_rank`形成`redis zset`。

按照如下步骤你可以很容易地实现该功能：

* 1.[zset添加member](https://redis.io/commands/zadd)

如果用户粉丝数目发生变动，则需要将该用户重新添加到`zset`中，假定用户id为user_id，粉丝量为score，则命令如下：

```bash
zadd fans_rank user_id score
```

注意算法复杂度：

>> Time complexity: O(log(N)) for each item added, where N is the number of elements in the sorted set.

* 2.zset维持topN操作

* 3.zset删除member

* 4、zset范围读取

## Refs

* [ZREMRANGEBYRANK](http://redisdoc.com/sorted_set/zremrangebyrank.html)
* [Redis 有序集合(sorted set)](http://www.runoob.com/redis/redis-sorted-sets.html)
* [Redis keep only top 50 elements in a sorted set](https://stackoverflow.com/questions/17650240/redis-keep-only-top-50-elements-in-a-sorted-set)