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

We have to backup data regularly for the security reason. There are a bunch of methods to backup MySQL database, for some reason, the performance are not the same. Once the database crash or some fatal errors happen, backup is the only way to restore the data and reduce the loss to the minimum. Generally speaking, we have three backup solutions:

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
# mysql -uroot -pmypass < /zhao/database_2013-08-13.sql # restore backup file
# mysql -uroot -pmypass < /zhao/binlog_2013-08-13_19.sql # restore append backup file
# mysql -uroot –pmypass # check the  result
mysql> SET sql_log_bin=1;  # set the new backup Position
```

You have to rebuild all the indexes after this backup. and the backup file is large, so choose this solution as appropriate.

# LVM snapshot backup
### solution conception
1, LVM requires the data of mysql database have to store on the logical volumes.
2, MySQL server have to be added read_only lock(mysql>FLUSH TABLES WITH READLOCK), you cannot quit server
3, Use another session create the snapshot for the volume which data located.

### backup strategy

LVM snapshot full backup and binary logs append backup

### precondition
#### create logical volumes and mount logical volumes


#### initialize mysql and redirect the data folder to /mydata/data

```
# cd /usr/local/mysql/
# scripts/mysql_install_db --user=mysql --datadir=/mydata/data
```

#### Edit and check the my.cnf file, reboot service

```
# vim /etc/my.cnf
datadir = /mydata/data
sync_binlog=1 # add this statement, Once the transaction is commited, the computer will move the logs from the cache to the logs file. and flush the new logs file to the driver.
# service mysqld start
```

### Process running
#### Ensure transaction logs and data file have to locate in the same volume.

```
# ls /mydata/data/
hellodb myclass mysql-bin.000003 stu18.magedu.com.err
ibdata1 mysql mysql-bin.000004 stu18.magedu.com.pid
ib_logfile0 mysql-bin.000001 mysql-bin.index student
ib_logfile1 mysql-bin.000002 performance_schema test
```

the last two files are logs

#### Add the global lock and flush logs

```
mysql> FLUSH TABLES WITH READ LOCK;
mysql> FLUSH LOGS;
```

#### Check and save the binary log and the position of binary log at present(very important).

```
mysql> SHOW MASTER STATUS;
+------------------+----------+--------------+------------------+
| File | Position | Binlog_Do_DB | Binlog_Ignore_DB |
+------------------+----------+--------------+------------------+
| mysql-bin.000004 | 187 | | |
+------------------+----------+--------------+------------------+
# mysql -uroot -pmypass -e 'SHOW MASTER STATUS;' >/zhao/lvmback-2013-08-14/binlog.txt
```

#### Create the snapshot volume.

```
# lvcreate -L 100M -s -p r -n mydata-lvm /dev/vg1/mydata
```

#### switch to another session to release the lock

```
mysql> UNLOCK TABLES;
```

#### Backup the data

```
# cp -a * /backup/lvmback-2013-08-14/
```

#### Create the append backup

```
mysql> use hellodb; # assign the default database
Database changed
mysql> CREATE TABLE testtb (id int,name CHAR(10)); # create table
Query OK, 0 rows affected (0.35 sec)
mysql> INSERT INTO testtb VALUES (1,'tom'); # add data
Query OK, 1 row affected (0.09 sec)
# mysqlbinlog --start-position=187 mysql-bin.000004 > /zhao/lvmlogbin_2013-08-14/binlog.sql # append backup
```

#### emulate database crashed

```
# service mysqld stop
# cd /mydata/data/
# rm -rf *
```

#### restore data

```
# cp /zhao/lvmback-2013-08-14/* /mydata/data/ -a # full backup restore
# cd /mydata/data/ # check the file
# chown -R mysql.mysql * # change the permission
# service mysqld start # start service
# mysql -uroot –pmypass
mysql> SHOW DATABASES;
mysql> SET sql_log_bin=0 # close the log
mysql> source /backup/lvmlogbin_2013-08-14/binlog.sql; # restore log
mysql> SHOW TABLES; # check the tables
+-------------------+
| Tables_in_hellodb |
+-------------------+
| classes |
| coc |
| courses |
| scores |
| students |
| teachers |
| testtb |
| toc |
+-------------------+
mysql> SET sql_log_bin=1; # open the log
```

This method use the closed hot standby backup, so it is fast for backup and restore data.
