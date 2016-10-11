---
layout: post
title: "MySQL backup and restore from Backups"
subtitle: ""
date: 2016-10-11
author: Dykin
category: Database
tags: Database MySQL
finished: true
---
# MySQL backup and restore from Backups

We have to backup data regularly for the security reason. There are a bunch of method to backup MySQL database, for some reason, the performance are not the same. Once the database crash or some fatal errors happen, backup is the only way to restore the data and reduce to the minimum loss. Generally speaking, we have three backup solutions:

1, mysqldump + binary logs backup
2, LVM snapshot + binary logs backup
3, Xtrabackup

## Lab environment:
System requirement: Ubuntu 16.06 server
Database: Mysql 5.7

## use mysqldump tool backup
### solution conception
mysqldump is an logic backup tool, which means it will backup the data from database to a text file. In another word, it will store the table structure and data to a text file.
The backup method is not difficult. First of all, mysqldump resolve the table's structure. secondly, it will add a CREATE statement to the text file. The record in the table will be transfered to the insert statement for recovering.  mysqldump advantages include the convenience and flexibility of viewing or even editing the output before restoring. Mysqldump is able to record the binary logs' location and the content. This backup solution could be hot standby.

### backup plan
mysqldump full backup(differential backup) and binary logs append backup

### backup procedure
#### mysqldump full backup
We need to add an read_only lock for the database before we start backup. So this backup is an warm standby.

```
#mysqldump -uroot -pmypass --lock-all-tables --master-data=2 --events --routines--all-databases > /backup/database_`date +%F`.sql
```

--lock-all-tables add the read_only lock for all tables.
--master-data=2  mark the binary logs location at present
--events  Include Event Scheduler events for the dumped databases in the output.
--routines  Dump stored routines (procedures and functions) from dumped databases
--all-databases   backup all database.

here is the reference manual [mysqldump — A Database Backup Program](http://dev.mysql.com/doc/refman/5.7/en/mysqldump.html)

```
# less database_2013-08-13.sql
-- #comment
-- Position to start replication or point-in-time recovery from
--
-- CHANGE MASTER TO MASTER_LOG_FILE='mysql-bin.000001', MASTER_LOG_POS=14203; #present location is at binary log mysql-bin.000001， --master-data=2 procession
--
-- Current Database: `hellodb`
--
CREATE DATABASE /*!32312 IF NOT EXISTS*/ `hellodb` /*!40100 DEFAULT CHARACTER SET utf8 */;
```

#### binary logs full backup

first: add the data

```
mysql> use hellodb;
mysql> INSERT INTO students(Name,Age,Gender,ClassID,TeacherID) values ('Steve',22,'M',3,3);
```

second: binary log append backup

```
# mysqlbinlog --start-position=14203 --stop-position=14527 mysql-bin.000001 > /backup/binlog_`date +%F_%H`.sql
```

--start-position=14203  the last time backup log Position
--stop-position=14527   today's log Position

### emulate data damaged. recovery all data.

```
mysql> DROP DATABASE hellodb; # delete DB
############ you have to implement at the offline ############
mysql> SET sql_log_bin=0; # shutdown binary logs
mysql> flush logs;
[root@stu18 ~]# mysql -uroot -pmypass < /zhao/database_2013-08-13.sql # restore backup file
[root@stu18 ~]# mysql -uroot -pmypass < /zhao/binlog_2013-08-13_19.sql # restore append backup file
[root@stu18 ~]# mysql -uroot –pmypass # check the  result
mysql> SET sql_log_bin=1;  # set the new backup Position
```

You have to re build the all indexes after this backup. and the backup file is large, so choose this solution as appropriate.
