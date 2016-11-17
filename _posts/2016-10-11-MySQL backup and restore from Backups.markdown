---
layout: post
title: "MySQL backup and restore from Backups"
date: 2016-10-11
author: Dykin
category: Database
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

# Xtrabackup
## advantages
Hot standby, reliable full backup or portion backup, support append backup, time point restore. No read_only lock requirement, save hard drive space. Restore backup is faster.
All of the advantages only implement at InnoDB engine. MyISAM still doesn't support append backup.

```
mysql> SHOW GLOBAL VARIABLES LIKE 'innodb_file%';
+--------------------------+----------+
| Variable_name | Value |
+--------------------------+----------+
| innodb_file_format | Antelope |
| innodb_file_format_check | ON |
| innodb_file_format_max | Antelope |
| innodb_file_per_table | ON |
+--------------------------+----------+
```

innodb_file_per_table on means innodb sets to the individual table. If it sets off, you have to use mysqldump to do full backup and modify the my.cnf file and initialize all system and restore the data. So I highly recommend you set it to 1 (innodb_file_per_table):

```
# ls
classes.frm coc.MYD courses.MYI scores.MYI teachers.frm testtb.ibd
classes.MYD coc.MYI db.opt students.frm teachers.MYD toc.frm
classes.MYI courses.frm scores.frm students.MYD teachers.MYI toc.MYD
coc.frm courses.MYD scores.MYD students.MYI testtb.frm toc.MYI
```

## Install Xtrabackup

download the lastest Xtrabackup from [here](https://www.percona.com/downloads/XtraBackup/)
here is the [installation manual](https://www.percona.com/doc/percona-xtrabackup/2.3/installation/apt_repo.html)

and then you have to install the perl-DBD-mysql dependent package.

```
sudo apt-get install perl-DBD-mysql
```

## Full backup
Once you use innobackupex to backup your database, it will call xtrabackup to backup all of the InnoDB table automatically. Copy all of the table structure definition related file(.frm), MyISAM,MERGE,CSV and ARCHIVE table related files. In the same time, it still is able to backup the system configure files and trigger related files.all of the files will be saved at an folder named by the time.

```
# innobackupex --user=DBUSER--password=DBUSERPASS /path/to/BACKUP-DIR/
```

the processing is following:

```
# mkdir /innobackup
# innobackupex --user=root --password=mypass /innobackup/ #完全备份
################ if the backup is correct the following information will be shown up###############
xtrabackup: Transaction log of lsn (1604655) to (1604655) was copied. # logs pos（lsn）
130814 07:04:55 innobackupex: All tables unlocked
innobackupex: Backup created in directory '/innobackup/2013-08-14_07-04-49'# the location of backup
innobackupex: MySQL binlog position: filename 'mysql-bin.000003', position 538898
130814 07:04:55 innobackupex: Connection to database server closed
130814 07:04:55 innobackupex: completed OK!
```

switch to backup folder to check out the data and files:

```
# cd /innobackup/2013-08-14_07-04-49/
[2013-08-14_07-04-49]# ls
backup-my.cnf myclass student xtrabackup_binlog_info
hellodb mysql test xtrabackup_checkpoints
ibdata1 performance_schema xtrabackup_binary xtrabackup_logfile
```

xtrabackup_checkpoints:
This file contains a line showing the to_lsn, which is the database’s LSN at the end of the backup. Create the full backup with a command such as the following:

```
xtrabackup --backup --target-dir=/data/backups/base --datadir=/var/lib/mysql/ # just an example from the official website
```

xtrabackup_binlog_info: the position and information of the binary logs

xtrabackup_binary: the executable binary file of backup

backup-my.cnf: mysql configuration backup file

xtrabackup_logfile: xtrabackup log file

## standby a full backup
standby means rollback the unsubmitted transactions and synchronize the submitted transactions.
we can use innobackupex --apply-log to do that:

```
# innobackupex -apply-log /innobackup/2013-08-14_07-04-49/

xtrabackup: starting shutdown with innodb_fast_shutdown = 1
130814 7:39:33 InnoDB: Starting shutdown...
130814 7:39:37 InnoDB: Shutdown completed; log sequence number 1606156
130814 07:39:37 innobackupex: completed OK!
```

## emulate database crashed and restore the backup

1, emulate database crashed

```
[ ~]# service mysqld stop
[ ~]# cd /mydata/data/
[ data]# rm -rf *
```

2, restore the backup
important: you cannot initialize database and service before recovery all data

```
[ ~]# innobackupex --copy-back /innobackup/2013-08-14_07-04-49/

innobackupex: Starting to copy InnoDB log files
innobackupex: in '/innobackup/2013-08-14_07-04-49'
innobackupex: back to original InnoDB log directory '/mydata/data'
innobackupex: Copying '/innobackup/2013-08-14_07-04-49/ib_logfile0' to '/mydata/data'
innobackupex: Copying '/innobackup/2013-08-14_07-04-49/ib_logfile1' to '/mydata/data'
innobackupex: Finished copying back files.
130814 07:58:22 innobackupex: completed OK!
```

3, You need to make sure all the data files belone to the correct group, and the user permissions are correct.
Otherwise you have to modify the data files' permission

```
# chown -R mysql:mysql /mydata/data/
```

4, start the service to check the recovery.

```
[data]# service mysqld start
```

## Use innobackupex to append backup

Every InnoDB file includes an LSN information, every time the file is changed, LSN automatically increases.
Let's change the data first:

```
[ data]# innobackupex --user=root --password=mypass --incremental /innobackup --incremental-basedir=/innobackup/2013-08-14_08-14-12/
```

/innobackup is the full backup folder, after this command, innobackupex will create a new folder to store all the appended backup data named by the time.

change the data second time and backup:

```
[ ~]# innobackupex --user=root --password=mypass --incremental /innobackup --incremental-basedir=/innobackup/2013-08-14_08-29-05/
```

–incremental-basedir: the last time backup folder

change the data third time and without append backup:

```
mysql> delete from coc where id=14;
```

## Use innobackupex to do full bakcup + append backup + binary logs restore backup
1, You'd better split the data folder and logs folder. Or if the system crashed you have to lose data and logs at the same time.

```
mkdir /mybinlog
chown mysql:mysql /mybinlog
vim /etc/my.cnf
log-bin=/mybinlog/mysql-bin
```

copy the binary logs:

```
[data]# cp mysql-bin.000001/innobackup/]
```

2, emulate the system crash

```
[ ~]# service mysqld stop
[ ~]# cd /mydata/data/
[ data]# rm -rf *
```

3, standby backup
the committed transactions need to add to the full backup
the uncommitted transactions need to rollback


```
[ ~]# innobackupex --apply-log --redo-only/innobackup/2013-08-14_08-14-12/
```

add the append backup to the full backup at the first time:

```
[~]# innobackupex --apply-log--redo-only /innobackup/2013-08-14_08-14-12/--incremental-dir=/innobackup/2013-08-14_08-29-05/
```

add the append backup to the full backup at the first time:

```
[~]# innobackupex --apply-log--redo-only /innobackup/2013-08-14_08-14-12/ --incremental-dir=/innobackup/2013-08-14_09-08-39/
```

–redo-only just synchronizes the committed transactions and doesn't rollback the uncommitted transactions.

4, restore data

```
[ ~]# innobackupex --copy-back/innobackup/2013-08-14_08-14-12/
```

5, change the group

```
[ ~]# cd /mydata/data/
[ data]# chown -R mysql:mysql *
```

6, check the result

```
[ ~]# mysql -uroot -pmypas
mysql> select * from coc;
+----+---------+----------+
| ID | ClassID | CourseID |
+----+---------+----------+
| 1| 1 | 2 |
| 2| 1 | 5 |
| 3| 2 | 2 |
| 4| 2 | 6 |
| 5| 3 | 1 |
| 6| 3 | 7 |
| 7| 4 | 5 |
| 8| 4 | 2 |
| 9| 5 | 1 |
| 10 | 5 | 9 |
| 11 | 6 | 3 |
| 12 | 6 | 4 |
| 13 | 7 | 4 |
| 14 | 7 | 3 |
+----+---------+----------+
14 rows in set (0.00 sec)
```

we can see the third time modified information doesn't implement.

7, restore data by binary logs
check the last time backup logs

```
[ data]# cd /innobackup/2013-08-14_09-08-39/
[ 2013-08-14_09-08-39]# cat xtrabackup_binlog_info
mysql-bin.000001 780
```

export the data without backup logs

```
[ innobackup]# mysqlbinlog mysql-bin.000001
# at 780
#130814 9:20:19 server id 1 end_log_pos 851 Query thread_id=7 exec_time=0 error_code=0
SET TIMESTAMP=1376443219/*!*/;
BEGIN
/*!*/;
# at 851
#130814 9:20:19 server id 1 end_log_pos 944 Query thread_id=7 exec_time=0 error_code=0
SET TIMESTAMP=1376443219/*!*/;
delete from coc where id=14
/*!*/;
# at 944
#130814 9:20:19 server id 1 end_log_pos 1016 Query thread_id=7 exec_time=0 error_code=0
SET TIMESTAMP=1376443219/*!*/;
COMMIT
/*!*/;
DELIMITER ;
# End of log file
ROLLBACK /* added by mysqlbinlog */;
/*!50003 SET COMPLETION_TYPE=@OLD_COMPLETION_TYPE*/;
/*!50530 SET @@SESSION.PSEUDO_SLAVE_MODE=0*/;
[ innobackup]# mysqlbinlog --start-position=780 mysql-bin.000001 > ./all.sql # export data
```

restore data

```
[ ~]# mysql -uroot –pmypass
mysql> SET SQL_LOG_BIN=0; # shutdown binary logs file
mysql> source /innobackup/all.sql # import data
mysql> SET SQL_LOG_BIN=1; # start binary logs file
mysql> select * from coc;
+----+---------+----------+
| ID | ClassID | CourseID |
+----+---------+----------+
| 1 | 1 | 2 |
| 2 | 1 | 5 |
| 3 | 2 | 2 |
| 4 | 2 | 6 |
| 5 | 3 | 1 |
| 6 | 3 | 7 |
| 7 | 4 | 5 |
| 8 | 4 | 2 |
| 9 | 5 | 1 |
| 10 | 5 | 9 |
| 11 | 6 | 3 |
| 12 | 6 | 4 |
| 13 | 7 | 4 |
+----+---------+----------+
13 rows in set (0.00 sec)
```

we use hot backup on this backup plan. So I think it is the best way to backup mysql database.

Outcome：All of three backup solutions require binary logs, which means logs are so important for backup and system.
