---
layout: post
title: Redis备份还原方案
date: 2019-11-7 19:10:31
category: 技术
tags: database bur
excerpt: 本文介绍Redis的备份还原方案……
---

## Overview

>> Before starting this section, make sure to read the following sentence: Make Sure to Backup Your Database. Disks break, instances in the cloud disappear, and so forth: no backups means huge risk of data disappearing into /dev/null.

## Backing up Redis data

>> Redis is very data backup friendly since you can copy RDB files while the database is running: the RDB is never modified once produced, and while it gets produced it uses a 
temporary name and is renamed into its final destination atomically using rename(2) only when the new snapshot is complete.

This means that copying the RDB file is completely safe while the server is running. This is what we suggest:

* Create a cron job in your server creating hourly snapshots of the RDB file in one directory, and daily snapshots in a different directory.
* Every time the cron script runs, make sure to call the find command to make sure too old snapshots are deleted: for instance you can take hourly snapshots for the latest 48 * hours, and daily snapshots for one or two months. Make sure to name the snapshots with data and time information.
* At least one time every day make sure to transfer an RDB snapshot outside your data center or at least outside the physical machine running your Redis instance.

按照官方推荐的方式对`Redis`进行备份：

```bash
# generate dump.rdb
$redis-cli
127.0.0.1:6379> auth xxx
OK
127.0.0.1:6379> config get dir
1) "dir"
2) "/var/lib/redis"
127.0.0.1:6379> bgsave
```

## Disaster recovery

>> Disaster recovery in the context of Redis is basically the same story as backups, plus the ability to transfer those backups in many different external data centers. This way data is secured even in the case of some catastrophic event affecting the main data center where Redis is running and producing its snapshots.

>> Since many Redis users are in the startup scene and thus don't have plenty of money to spend we'll review the most interesting disaster recovery techniques that don't have too high costs.

* Amazon S3 and other similar services are a good way for implementing your disaster recovery system. Simply transfer your daily or hourly RDB snapshot to S3 in an encrypted form. You can encrypt your data using gpg -c (in symmetric encryption mode). Make sure to store your password in many different safe places (for instance give a copy to the most important people of your organization). It is recommended to use multiple storage services for improved data safety.
* Transfer your snapshots using SCP (part of SSH) to far servers. This is a fairly simple and safe route: get a small VPS in a place that is very far from you, install ssh there, and generate an ssh client key without passphrase, then add it in the authorized_keys file of your small VPS. You are ready to transfer backups in an automated fashion. Get at least two VPS in two different providers for best results.

按照官方推荐的方式对`Redis`进行还原，分两种场景：

* 一、AOF = no

```bash
# In case it is not AOF mode. ('no'), the process is quite straightforward. First, just stop Redis server (new Redis):

$ /etc/init.d/redis-server stop

# Then, remove the current dumb.rdb file (if there is one) or rename it to dump.rdb.old

$ mv /var/lib/redis/dump.rdb /var/lib/redis/dump.rdb.back

# After confirming there is no dump.rdb file name in the folder, copy the good backup you took earlier into the /var/lib/redis folder (or which folder Redis is using in your env).

$ cp /backup/redis/dump.xxxxxx.rdb /var/lib/redis/dump.rdb

# Remember to set it to be owned by Redis user/group

$ chown redis:redis dump.rdb

# Now, just start the Redis server.

$ /etc/init.d/redis-server start
```

* 二、AOF = yes

```bash
# The first thing to do is the same as previous case, just stop the Redis server:
$ /etc/init.d/redis-server stop

# Then, remove the current dumb.rdb file (if there is one) or rename it to dump.rdb.old:
$ mv /var/lib/redis/dump.rdb /var/lib/redis/dump.rdb.old
$ mv /var/lib/redis/appendonly.aof /var/lib/redis/appendonly.aof.old

# Copy the good backup file in and correct its permission:
$ cp /backup/redis/dump.xxxxxx.rdb /var/lib/redis/dump.rdb
$ chown redis:redis /var/lib/redis/dump.rdb

# Next the important part is to disable AOF by editing /etc/redis/redis.conf file, set appendonly as "no":
#Verify appendonlu is set to "no":
$cat /etc/redis/redis.conf |grep 'appendonly '|cut -d' ' -f2

#Next start the Redis server and run the following command to create new appendonly.aof file:
$ /etc/init.d/redis-server start
$ redis-cli bgrewriteaof
# Check the progress (0 - done, 1 - not yet):
$ redis-cli info | grep aof_rewrite_in_progress

# You should see a new appendonly.aof file.
# Next, stop the server:
$ /etc/init.d/redis-server stop

# After it finished, enable AOF again by changing appendonly in /etc/redis/redis.conf file to yes
# Then start the Redis server again:
$ /etc/init.d/redis-server start
```

## Addition

上述方案适用于单实例`Redis`。如果想要备份还原`Redis`集群（`分片`vs`主从`），则需要另外研究解决方案，会更复杂很多：参考[How to efficiently backup and restore a Redis cluster?](https://www.reddit.com/r/redis/comments/8vpsvr/help_needed_how_to_efficiently_backup_and_restore/e1rms55/)

## Refs

* [persistence](https://redis.io/topics/persistence)
* [how-to-back-up-and-restore-your-redis-data-on-ubuntu-14-04](https://www.digitalocean.com/community/tutorials/how-to-back-up-and-restore-your-redis-data-on-ubuntu-14-04)
* [How-to-Backup-and-Restore-Open-Source-Redis](https://community.pivotal.io/s/article/How-to-Backup-and-Restore-Open-Source-Redis)
* [Restoring redis cluster snapshot rdb files on new setup](https://groups.google.com/forum/#!topic/redis-db/sq7dhuxq6xU)