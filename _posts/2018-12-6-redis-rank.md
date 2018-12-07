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

* zset是有序集合类型
* zset支持常用的添加，删除，范围读取操作
* 有序集合的成员是唯一的,但分数(score)却可以重复

下文将详细展开介绍：利用`redis zset`实现排行榜功能

### redis zset实现排行榜功能

现在假定有如下排行榜需求：实现一个粉丝Top10排行榜。简单分析该需求，我们可以将用户id作为member，用户粉丝量作为score，并设置`redis key`为`fans_rank`形成`redis zset`

按照如下步骤你可以很容易地实现该功能：

* 1.[zset添加member](https://redis.io/commands/zadd)

如果用户粉丝数目发生变动，则需要将该用户重新添加到`zset`中进行排序，假定用户id为user_id，粉丝量为score，则命令如下：

```bash
ZADD fans_rank user_id score
```

注意算法复杂度：

>> Time complexity: O(log(N)) for each item added, where N is the number of elements in the sorted set.

* 2.[zset维持TopN操作](https://redis.io/commands/zremrangebyrank)

由于要维持TopN排行榜，所以最好的处理方法是每次`ZADD`后执行`ZREMRANGEBYRANK`，假定要维护TopN榜单，则命令如下：

```bash
ZREMRANGEBYRANK fans_rank 0 -(TopN+1)
```

注意：

* 这里范围是`0`——`-(TopN+1)`，因为`redis zset`是按照从小到大方式排序的，所以需要维持的榜单是倒数TopN，也即从最后一个元素开始，倒数推TopN个
* 时间复杂度：
>> Time complexity: O(log(N)+M) with N being the number of elements in the sorted set and M the number of elements removed by the operation.

* 3.[zset删除member](https://redis.io/commands/zrem)

如果某个用户进行注销操作，或者被封号了，这个时候该用户应该从`redis zset`中被踢掉，如下：

```bash
ZREM fans_rank user_id
```

注意算法复杂度：

>> Time complexity: O(M*log(N)) with N being the number of elements in the sorted set and M the number of elements to be removed.

* 4、[zset范围读取](https://redis.io/commands/zrevrange)

最后是获取排行榜TopN数据，命令如下：

```bash
ZREVRANGE fans_rank 0 (TopN-1) withscores
```

注意：

* 这里范围是`0`——`TopN-1`，原因是：`redis zset`下标从0开始，而`ZREVRANGE`是倒序取数据
* `redis zset` `ZRANGE`与`ZREVRANGE`当元素`score`相同时会默认根据`member`字典序排序
* `withscores`会将`member`与对应`score`一起返回
* 算法复杂度：
>> Time complexity: O(log(N)+M) with N being the number of elements in the sorted set and M the number of elements returned.

通过如上四个步骤就可以顺利完成排行榜核心逻辑开发

### 其他

这里给出`redis zset`与排行榜功能相关的其它几个常用命令：

* 顺序取数据——[ZRANGE](https://redis.io/commands/zrevrange)
* 计算长度——[ZCARD](https://redis.io/commands/zcard)

## Refs

* [ZREMRANGEBYRANK](http://redisdoc.com/sorted_set/zremrangebyrank.html)
* [Redis 有序集合(sorted set)](http://www.runoob.com/redis/redis-sorted-sets.html)
* [Redis keep only top 50 elements in a sorted set](https://stackoverflow.com/questions/17650240/redis-keep-only-top-50-elements-in-a-sorted-set)
* [Redis data-types](https://redis.io/topics/data-types)