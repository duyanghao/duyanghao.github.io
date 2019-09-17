---
layout: post
title: MongoDB备份还原方案
date: 2019-11-1 20:10:31
category: 技术
tags: database bur
excerpt: 本文介绍MongoDB的备份还原方案……
---

## Overview

When deploying MongoDB in production, you should have a strategy for capturing and restoring backups in the case of data loss events.

## MongoDB Backup and Restore Methods

`MongoDB`官方给出了四种`bur`方案：

* Back Up with Atlas

MongoDB Atlas, the official MongoDB cloud service

* Back Up with MongoDB Cloud Manager or Ops Manager

MongoDB Cloud Manager is a hosted back up, monitoring, and automation service for MongoDB. MongoDB Cloud Manager supports backing up and restoring MongoDB replica sets and sharded clusters from a graphical user interface.

* [Back Up by Copying Underlying Data Files](https://docs.mongodb.com/manual/tutorial/backup-with-filesystem-snapshots/)

You can create a backup of a MongoDB deployment by making a copy of MongoDB’s underlying data files.

These filesystem snapshots, or “block-level” backup methods, use system level tools to create copies of the device that holds MongoDB’s data files. These methods complete quickly and work reliably, but require additional system configuration outside of MongoDB.

* [Back Up with mongodump](https://docs.mongodb.com/manual/tutorial/backup-and-restore-tools/)

mongodump reads data from a MongoDB database and creates high fidelity BSON files which the mongorestore tool can use to populate a MongoDB database. mongodump and mongorestore are simple and efficient tools for backing up and restoring small MongoDB deployments, but are not ideal for capturing backups of larger systems.(For resilient and non-disruptive backups, use a file system or block-level disk snapshot function)

When connected to a MongoDB instance, mongodump can adversely affect mongod performance. If your data is larger than system memory, the queries will push the working set out of memory, causing page faults.

Use these tools for backups if other backup methods, such as MongoDB Cloud Manager or file system snapshots are unavailable.

> NOTE
> 
> mongodump and mongorestore cannot be part of a backup strategy for 4.2+ sharded clusters that have sharded transactions in progress as these tools cannot guarantee a atomicity guarantees of data across the shards.
> 
> For 4.2+ sharded clusters with in-progress sharded transactions, for coordinated backup and restore processes that maintain the atomicity guarantees of transactions across shards, see:
> 
> * [MongoDB Atlas](https://www.mongodb.com/cloud/atlas?jmp=docs),
> * [MongoDB Cloud Manager](https://www.mongodb.com/cloud/cloud-manager?jmp=docs), or
> * [MongoDB Ops Manager](https://www.mongodb.com/products/ops-manager?jmp=docs).

> For MongoDB 4.0 and earlier deployments, refer to the corresponding versions of the manual. For example:
> 
> * https://docs.mongodb.com/v4.0
> * https://docs.mongodb.com/v3.6
> * https://docs.mongodb.com/v3.4

## Conclusion

* 1、`MongoDB Atlas`与`MongoDB Cloud Manager or Ops Manager`需要特定使用场景，不通用，这里舍弃
* 2、`Underlying Data Files`是官方推荐的`bur`方案，适用于大数据量的`MongoDB`备份与还原，但是需要额外的系统配置，比如：[LVM](https://docs.mongodb.com/manual/reference/glossary/#term-lvm)
* 3、`mongodump`简单有效，适用于`MongoDB`数据量较小的情况；另外这种方式会影响`mongod`性能，而且数据量越大影响越大；`mongodump`不能用于`Sharded Cluster`，可用于`Replica-Set`([Difference between Sharding And Replication on MongoDB](https://dba.stackexchange.com/questions/52632/difference-between-sharding-and-replication-on-mongodb))

这里由于我们的`MongoDB`数据量较小，最多100M，所以采用`mongodump`方案进行备份与还原

## Back Up with mongodump

### mongodump

>> The mongodump utility backs up data by connecting to a running mongod.

>> The utility can create a backup for an entire server, database or collection, or can use a query to backup just part of a collection.

```bash
#mongodump --host=mongodb1.example.net --port=3017 --username=user --password="pass" --out=/opt/backup/mongodump-2013-10-24
```

### mongorestore 

>> The mongorestore utility restores a binary backup created by mongodump. By default, mongorestore looks for a database backup in the dump/ directory.

```bash
#mongorestore --host=mongodb1.example.net --port=3017 --username=user  --authenticationDatabase=admin /opt/backup/mongodump-2013-10-24
```

## Refs

* [Back Up and Restore with MongoDB Tools](https://docs.mongodb.com/manual/tutorial/backup-and-restore-tools/)
* [MongoDB Backup Methods](https://docs.mongodb.com/manual/core/backups/)
* [Back Up and Restore with Filesystem Snapshots](https://docs.mongodb.com/manual/tutorial/backup-with-filesystem-snapshots/)
* [difference-between-sharding-and-replication-on-mongodb](https://dba.stackexchange.com/questions/52632/difference-between-sharding-and-replication-on-mongodb)
