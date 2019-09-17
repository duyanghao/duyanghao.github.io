---
layout: post
title: PostgreSQL备份还原方案
date: 2019-11-1 19:10:31
category: 技术
tags: database bur
excerpt: 本文介绍PostgreSQL的备份还原方案……
---

## 前言

`PostgreSQL`是流行的关系型数据库，本文先介绍`PostgreSQL`的三种备份还原方案，然后针对`SQL dump`给出备份还原（`bur`）步骤……

## PostgreSQL Backup and Restore

`PostgreSQL`官方给出了三种`bur`方案：

* SQL dump
* File system level backup
* Continuous archiving

各有优缺点，对比如下：

### SQL dump

>> The idea behind this dump method is to generate a file with SQL commands that, when fed back to the server, will recreate the database in the same state as it was at the time of the dump. PostgreSQL provides the utility program pg_dump for this purpose

`SQL dump`方案原理：生成构成数据库的SQL文件，利用这些SQL语句可以还原出数据库

#### Pros

* 1、pg_dump is a regular PostgreSQL client application (albeit a particularly clever one). This means that you can perform this backup procedure from any remote host that has access to the database.
* 2、pg_dump's output can generally be re-loaded into newer versions of PostgreSQL, whereas file-level backups and continuous archiving are both extremely server-version-specific.
* 3、pg_dump is also the only method that will work when transferring a database to a different machine architecture, such as going from a 32-bit to a 64-bit server.
* 4、Dumps created by pg_dump are internally consistent, meaning, the dump represents a snapshot of the database at the time pg_dump began running. pg_dump does not block other operations on the database while it is working. (Exceptions are those operations that need to operate with an exclusive lock, such as most forms of ALTER TABLE.)

#### Cons

* pg_dump does not need to dump the contents of indexes for example, just the commands to recreate them. However, taking a file system backup might be faster.

**简单实用，适用于`PostgreSQL`数据量小的情况下**

### File system level backup

>> An alternative backup strategy is to directly copy the files that PostgreSQL uses to store the data in the database

`File system level backup`方案原理：直接拷贝`PostgreSQL`数据文件，然后进行还原

#### Pros

* Taking a file system backup might be faster than SQL dump

#### Cons

There are two restrictions, however, which make this method impractical, or at least inferior to the pg_dump method:

* The database server must be shut down in order to get a usable backup. Half-way measures such as disallowing all connections will not work (in part because tar and similar tools do not take an atomic snapshot of the state of the file system, but also because of internal buffering within the server). Information about stopping the server can be found in Section 18.5. Needless to say, you also need to shut down the server before restoring the data.
* If you have dug into the details of the file system layout of the database, you might be tempted to try to back up or restore only certain individual tables or databases from their respective files or directories. This will not work because the information contained in these files is not usable without the commit log files, pg_xact/*, which contain the commit status of all transactions. A table file is only usable with this information. Of course it is also impossible to restore only a table and the associated pg_xact data because that would render all other tables in the database cluster useless. So file system backups only work for complete backup and restoration of an entire database cluster.
* Note that a file system backup will typically be larger than an SQL dump. (pg_dump does not need to dump the contents of indexes for example, just the commands to recreate them.) However, taking a file system backup might be faster.

**不建议使用**

### Continuous Archiving and Point-in-Time Recovery (PITR)

>> At all times, PostgreSQL maintains a write ahead log (WAL) in the pg_wal/ subdirectory of the cluster's data directory. The log records every change made to the database's data files. This log exists primarily for crash-safety purposes: if the system crashes, the database can be restored to consistency by “replaying” the log entries made since the last checkpoint. However, the existence of the log makes it possible to use a third strategy for backing up databases: we can combine a file-system-level backup with backup of the WAL files. If recovery is needed, we restore the file system backup and then replay from the backed-up WAL files to bring the system to a current state. 

`Continuous Archiving and Point-in-Time Recovery (PITR)`方案原理：利用`WAL`的文件系统备份，对这些`WAL`文件应答进行还原

#### Pros

* We do not need a perfectly consistent file system backup as the starting point. Any internal inconsistency in the backup will be corrected by log replay (this is not significantly different from what happens during crash recovery). So we do not need a file system snapshot capability, just tar or a similar archiving tool.
* Since we can combine an indefinitely long sequence of WAL files for replay, continuous backup can be achieved simply by continuing to archive the WAL files. This is particularly valuable for large databases, where it might not be convenient to take a full backup frequently.
* It is not necessary to replay the WAL entries all the way to the end. We could stop the replay at any point and have a consistent snapshot of the database as it was at that time. Thus, this technique supports point-in-time recovery: it is possible to restore the database to its state at any time since your base backup was taken.
* If we continuously feed the series of WAL files to another machine that has been loaded with the same base backup file, we have a warm standby system: at any point we can bring up the second machine and it will have a nearly-current copy of the database.
* As with the plain file-system-backup technique, this method can only support restoration of an entire database cluster, not a subset. Also, it requires a lot of archival storage: the base backup might be bulky, and a busy system will generate many megabytes of WAL traffic that have to be archived. Still, it is the preferred backup technique in many situations where high reliability is needed.

#### Cons

* This approach is more complex to administer than either of the previous approaches

**复杂但是有效，适用于`PostgreSQL`数据量大的情况下，不需要频繁做全量备份，只需要做增量备份，减少了备份时间**

### SQL dump bur步骤

#### SQL dump backup

>> pg_dump dumps only a single database at a time, and it does not dump information about roles or tablespaces (because those are cluster-wide rather than per-database). To support convenient dumping of the entire contents of a database cluster, the pg_dumpall program is provided. pg_dumpall backs up each database in a given cluster, and also preserves cluster-wide data such as role and tablespace definitions. The basic usage of this command is:

```bash
pg_dumpall > dumpfile
```

#### SQL dump restore

>> The resulting dump can be restored with psql:

```bash
psql -f dumpfile postgres
```

## Refs

* [PostgreSQL bur](https://www.postgresql.org/docs/current/backup-file.html)
* [How to Back Up Your PostgreSQL Database](https://www.linode.com/docs/databases/postgresql/how-to-back-up-your-postgresql-database/)