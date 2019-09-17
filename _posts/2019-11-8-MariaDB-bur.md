---
layout: post
title: MariaDB备份还原方案
date: 2019-11-8 19:10:31
category: 技术
tags: database bur
excerpt: 本文介绍MariaDB的备份还原方案……
---

## [MariaDB History](https://softwareengineering.stackexchange.com/questions/120178/whats-the-difference-between-mariadb-and-mysql)

>> MariaDB is a backward compatible, binary drop-in replacement of MySQL. What this means is:

>> * Data and table definition files (.frm) files are binary compatible.
>> * All client APIs, protocols and structs are identical.
>> * All filenames, binaries, paths, ports, sockets, and etc... should be the same.
>> * All MySQL connectors work unchanged with MariaDB.
>> * The mysql-client package also works with MariaDB server.

In most common practical scenarios, MariaDB version 5.x.y will work exactly like MySQL 5.x.y, MariaDB follows the version of MySQL, i.e. it's version number is used to indicate with which MySQL version it's compatible.

MariaDB originated as a fork of MySQL by Michael "Monty" Widenius, one of the original developers of MySQL and co-founder of MySQL Ab. The MariaDB Foundation acts as the custodian of MariaDB.

The main motivation behind MariaDB was to provide a floss version of MySQL, in case Oracle goes all corporate with MySQL. It's worth noting that Monty was vocal against MySQL acquisition (via Sun's acquisition) by Oracle.

Although MariaDB is supposed to be compatible with MySQL, for one reason or the other there are quite a few compatibility issues and different features:

>> * MariaDB includes all popular open source engines,
>> * MariaDB claims several speed improvements over MySQL, and
>> * there are a few new floss extensions that MySQL lacks

Finally, the name comes from Monty's daughter Maria (the other one being My), as MySQL is now a registered trademark of Oracle Corporation.

简单的说，MariaDB 是 MySQL fork出来的一个分支，旨在开源。MariaDB兼容了MySQL很多方面，当然也对MySQL进行了优化，开发了许多特性，这里不详细展开，具体见[MariaDB vs MySQL](https://www.eversql.com/mariadb-vs-mysql/)

## OverView

由于项目中有使用MariaDB，涉及到备份还原，本文对此做一个总结，详细描述MariaDB的备份还原方案

## Backup and Restore

官方提供两种备份机制：Logical vs Physical，如下：

* Logical： Logical backups consist of the SQL statements necessary to restore the data, such as CREATE DATABASE, CREATE TABLE and INSERT.
* Physical: Physical backups are performed by copying the individual data files or directories.

优缺点如下：

* logical backups are more flexible, as the data can be restored on other hardware configurations, MariaDB versions or even on another DBMS, while physical backups cannot be imported on significantly different hardware, a different DBMS, or potentially even a different MariaDB version.
* logical backups can be performed at the level of database and table, while physical databases are the level of directories and files. In the MyISAM and InnoDB storage engines, each table has an equivalent set of files. (In versions prior to MariaDB 5.5, by default a number of InnoDB tables are stored in the same file, in which case it is not possible to backup by table. See innodb_file_per_table.)
* logical backups are larger in size than the equivalent physical backup.
* logical backups takes more time to both backup and restore than the equivalent physical backup.
* log files and configuration files are not part of a logical backup

也即：logical 更加灵活且可控，但是相比 Physical 更加占用空间和耗时

## Logical Backup and Restore

对于MariaDB数据量较小的情况我们采用逻辑备份方式：

>> mysqldump performs a logical backup. It is the most flexible way to perform a backup and restore, and a good choice when the data size is relatively small.

>> For large datasets, the backup file can be large, and the restore time lengthy.

>> mysqldump dumps the data into SQL format (it can also dump into other formats, such as CSV or XML) which can then easily be imported into another database. The data can be imported into other versions of MariaDB, MySQL, or even another DBMS entirely, assuming there are no version or DBMS-specific statements in the dump.

>> mysqldump dumps triggers along with tables, as these are part of the table definition. However, stored procedures, views, and events are not, and need extra parameters to be recreated explicitly (for example, --routines and --events). Procedures and functions are however also part of the system tables (for example mysql.proc).

这里我们介绍`mysqldump`工具进行Logical Backup and Restore：

```bash
# backup
$ mysqldump -uxxx -pxxx db_name > backup-file.sql

# restore-phase-1(drop db)
$ mysqladmin -uxxx -pxxx -f drop db_name
Database "xxx" dropped

# restore-phase-2(recreate db)
$ mysqladmin -uxxx -pxxx create db_name

# restore-phase-3(restore db)
$ mysql -uxxx -pxxx db_name < backup-file.sql
```

具体可参考[mysqldump](https://mariadb.com/kb/en/library/mysqldump/)

## Physical Backup and Restore

对于MariaDB数据量较大的情况我们采用物理备份方式：

>> Mariabackup is an open source tool provided by MariaDB for performing physical online backups of InnoDB, Aria and MyISAM tables. For InnoDB, “hot online” backups are possible. It was originally forked from Percona XtraBackup 2.3.8. It is available on Linux and Windows.

这里我们介绍[Mariabackup](https://mariadb.com/kb/en/library/mariabackup-overview/)工具进行Physical Backup and Restore：

#### 1、Mariabackup Backup

```bash
# install Mariabackup
$ yum install MariaDB-backup

# Backing up the Database Server
$ mariabackup --backup \
   --target-dir=/var/mariadb/backup/ \
   --user=mariabackup --password=mypassword
$ ls /var/mariadb/backup/

aria_log.0000001  mysql                   xtrabackup_checkpoints
aria_log_control  performance_schema      xtrabackup_info
backup-my.cnf     test                    xtrabackup_logfile
ibdata1           xtrabackup_binlog_info
```

#### 2、Mariabackup Restore

* Preparing the Backup

>> Preparing the Backup(Before you can restore from a backup, you first need to prepare it to make the data files consistent.)

```bash
$ mariabackup --prepare \
   --target-dir=/var/mariadb/backup/
```
* Restoring the Backup
  * First, stop the MariaDB Server process.
  * Then, ensure that the datadir is empty.
  * Then, run Mariabackup with one of the options mentioned above:
    ```bash
    $ mariabackup --copy-back \
    --target-dir=/var/mariadb/backup/
    ```
  * Then, you may need to fix the file permissions.
  When Mariabackup restores a database, it preserves the file and directory privileges of the backup. However, it writes the files to disk as the user and group restoring the database. As such, after restoring a backup, you may need to adjust the owner of the data directory to match the user and group for the MariaDB Server, typically mysql for both. For example, to recursively change ownership of the files to the mysql user and group, you could execute:
  ```bash
  $ chown -R mysql:mysql /var/lib/mysql/
  ```
  * Finally, start the MariaDB Server process.

具体可参考[full-backup-and-restore-with-mariabackup](https://mariadb.com/kb/en/library/full-backup-and-restore-with-mariabackup/)

## Refs

* [Backup and Restore Overview](https://mariadb.com/kb/en/library/backup-and-restore-overview/)
* [MariaDB vs MySQL – Comparing MySQL 8.0 with MariaDB 10.3](https://www.eversql.com/mariadb-vs-mysql/)