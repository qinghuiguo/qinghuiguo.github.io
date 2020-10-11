---
layout: post
title:  "数据库实例崩溃恢复-MySQL"
date:   2020-10-11 09:40 +0800
categories: MySQL
tags: 崩溃 恢复 数据一致性
author: qinghui.guo
mathjax: true
---

* content
{:toc}


## 什么是crash recover？

> 由于故障导致Instance异常关闭,都会导致数据库实例在重启时执行实例恢复，由后台进程完成，不需要人工干预,恢复的时间检点之间的redo量控制。


## 为什么要进行crash recover

>正常运行期间，里面存在很多脏块，如果实例异常，导致脏块未写入磁盘，如果没有什么手段把崩溃后的脏块找回来，肯定会出现数据不一致的情况。当然关系数据库设计之初，这种情况就已经考虑了，通过重做日志，完全可以实现实例crash不丢数据。
总而言之，所有已提交的事务，都是可以恢复的，这样才能保证数据的一致性。



## 实例恢复的过程

> **LSN（Log Sequence Number**）
一个64位无符号整数，表示Redo Log系统中的时间点，也是事务写入Redo Log的字节总量，从日志初始化开始计数(数据库初始化安装时间点开始且单调递增)

LSN不仅存在于Redo Log中，在每个数据页中都保存着一个LSN，在进行数据恢复时通过LSN做比较运算可以判断出每个数据页是否需要进行恢复操作



> **恢复过程**

1、确定恢复起点，MySQL的恢复起点LSN由ibdata1的第一个page确定，如果该数据也无效，则从dblwr恢复。**很奇怪，该点到底不是检查点，还是最后一个写入的脏块的LSN？检查点是否会记录到ibdata1文件头，没有说明，但是从恢复逻辑看，把这个LSN当做检查点了**
2、恢复truncate操作，保障原子性
3、三次扫描日志
- 第一次根据一号日志头中记录的CHECKPOINT LSN找到MLOG_CHECKPOINT（**记录记录检查点之后space id映射关系，会关联检查点LSN**）
- 第二次从checkpoint点开始扫描解析 redo 日志并存储到hash中；如果hash空间不够用，则再来一轮重新开始，解析一批，应用一批
4、只有LSN大于数据页上LSN的日志才会被apply，完成崩溃恢复过程。**应用日志的过程并没有验证数据文件的版本**
5、回滚，redo log中的操作都apply到数据页中，但是对于prepare状态的事务却还没有进行回滚处理，这个阶段则是针对prepare状态的事务进行处理，需要使用到binlog和undo log。

## 模拟crash recover，验证是否校验数据文件

>1、创建测试表，

```
root@test 10:54:21>create table test as select * from mysql.user; 
Query OK, 3 rows affected (0.01 sec)
Records: 3  Duplicates: 0  Warnings: 0

root@test 10:54:32>create table checksum as select * from mysql.user;
Query OK, 3 rows affected (0.02 sec)
Records: 3  Duplicates: 0  Warnings: 0

```

> 2、运行状态备份checksum的idb文件


```
 cp checksum.ibd checksum.ibd.bak 
```


> 3、开启事务

```
root@test 10:56:35>begin ; 
Query OK, 0 rows affected (0.00 sec)

root@test 10:56:37>
root@test 10:56:37>
root@test 10:56:38>
root@test 10:56:38>insert into test select * from test ;


root@test 10:57:37>begin ; 
root@test 10:57:55>delete from checksum ;
Query OK, 3 rows affected (0.01 sec)
root@test 10:58:07>commit; 
Query OK, 0 rows affected (0.01 sec)
```

> 4、使用老版本的ibd文件覆盖新版本的ibd文件

```
[root@node2 test]# cp checksum.ibd.bak checksum.ibd 
```


> 5、启动数据库观察，
checksum表的虽然使用的是老版本的数据文件，但是能正常读取，所以没有校验数据文件，而且数据也是老的。


```
root@test 11:02:05>select count(*) from test; 
+----------+
| count(*) |
+----------+
|        3 |
+----------+
1 row in set (0.00 sec)

root@test 11:02:11>select count(*) from checksum ; 
+----------+
| count(*) |
+----------+
|        3 |
+----------+
1 row in set (0.00 sec)
```

> 错误日志，可以发现先扫描日志，然后发现数据库异常关闭，开始crash recover，回滚未完成的事务



```

2020-10-12T15:01:16.559376Z 0 [Warning] The syntax '--log_warnings/-W' is deprecated and will be removed in a future release. Please use '--log_error_verbosity' instead.
2020-10-12T15:01:16.559653Z 0 [Warning] TIMESTAMP with implicit DEFAULT value is deprecated. Please use --explicit_defaults_for_timestamp server option (see documentation for more details).
2020-10-12T15:01:16.559705Z 0 [Note] --secure-file-priv is set to NULL. Operations related to importing and exporting data are disabled
2020-10-12T15:01:16.559745Z 0 [Note] /usr/local/mysql/bin/mysqld (mysqld 5.7.30-log) starting as process 21137 ...
2020-10-12T15:01:16.572172Z 0 [Warning] InnoDB: Using innodb_file_format is deprecated and the parameter may be removed in future releases. See http://dev.mysql.com/doc/refman/5.7/en/innodb-file-format.html
2020-10-12T15:01:16.572239Z 0 [Warning] InnoDB: innodb_open_files should not be greater than the open_files_limit.

2020-10-12T15:01:16.572304Z 0 [Note] InnoDB: PUNCH HOLE support available
2020-10-12T15:01:16.572317Z 0 [Note] InnoDB: Mutexes and rw_locks use GCC atomic builtins
2020-10-12T15:01:16.572323Z 0 [Note] InnoDB: Uses event mutexes
2020-10-12T15:01:16.572328Z 0 [Note] InnoDB: GCC builtin __sync_synchronize() is used for memory barrier
2020-10-12T15:01:16.572333Z 0 [Note] InnoDB: Compressed tables use zlib 1.2.11
2020-10-12T15:01:16.574640Z 0 [Note] InnoDB: Number of pools: 1
2020-10-12T15:01:16.574777Z 0 [Note] InnoDB: Using CPU crc32 instructions
2020-10-12T15:01:16.589403Z 0 [Note] InnoDB: Initializing buffer pool, total size = 128M, instances = 1, chunk size = 128M
2020-10-12T15:01:16.601308Z 0 [Note] InnoDB: Completed initialization of buffer pool
2020-10-12T15:01:16.605075Z 0 [Note] InnoDB: If the mysqld execution user is authorized, page cleaner thread priority can be changed. See the man page of setpriority().
2020-10-12T15:01:16.622145Z 0 [Note] InnoDB: Opened 3 undo tablespaces
2020-10-12T15:01:16.622171Z 0 [Note] InnoDB: 3 undo tablespaces made active
2020-10-12T15:01:16.622444Z 0 [Note] InnoDB: Highest supported file format is Barracuda.
2020-10-12T15:01:16.626225Z 0 [Note] InnoDB: Log scan progressed past the checkpoint lsn 864148092
2020-10-12T15:01:16.626253Z 0 [Note] InnoDB: Doing recovery: scanned up to log sequence number 864148101
2020-10-12T15:01:16.626284Z 0 [Note] InnoDB: Database was not shutdown normally!
2020-10-12T15:01:16.626293Z 0 [Note] InnoDB: Starting crash recovery.
2020-10-12T15:01:16.726099Z 0 [Note] InnoDB: 1 transaction(s) which must be rolled back or cleaned up in total 3 row operations to undo
2020-10-12T15:01:16.726126Z 0 [Note] InnoDB: Trx id counter is 12800
2020-10-12T15:01:16.727336Z 0 [Note] InnoDB: Last MySQL binlog file position 0 1654402, file name mysql-bin.000018
2020-10-12T15:01:16.836548Z 0 [Note] InnoDB: Removed temporary tablespace data file: "ibtmp1"
2020-10-12T15:01:16.836570Z 0 [Note] InnoDB: Creating shared tablespace for temporary tables
2020-10-12T15:01:16.837426Z 0 [Note] InnoDB: Starting in background the rollback of uncommitted transactions
2020-10-12T15:01:16.837488Z 0 [Note] InnoDB: Rolling back trx with id 12349, 3 rows to undo
2020-10-12T15:01:16.837799Z 0 [Note] InnoDB: Setting file './ibtmp1' size to 12 MB. Physically writing the file full; Please wait ...
2020-10-12T15:01:16.841944Z 0 [Note] InnoDB: Rollback of trx with id 12349 completed
2020-10-12T15:01:16.841982Z 0 [Note] InnoDB: Rollback of non-prepared transactions completed
2020-10-12T15:01:16.850446Z 0 [Note] InnoDB: File './ibtmp1' size is now 12 MB.
2020-10-12T15:01:16.851162Z 0 [Note] InnoDB: 96 redo rollback segment(s) found. 96 redo rollback segment(s) are active.
2020-10-12T15:01:16.851183Z 0 [Note] InnoDB: 32 non-redo rollback segment(s) are active.
2020-10-12T15:01:16.852133Z 0 [Note] InnoDB: Waiting for purge to start
2020-10-12T15:01:16.903629Z 0 [Note] InnoDB: 5.7.30 started; log sequence number 864148101
2020-10-12T15:01:16.904053Z 0 [Note] Plugin 'FEDERATED' is disabled.
2020-10-12T15:01:16.909881Z 0 [Note] InnoDB: Loading buffer pool(s) from /data/mysql/data/ib_buffer_pool
2020-10-12T15:01:16.911895Z 0 [ERROR] Function 'rpl_semi_sync_master' already exists
2020-10-12T15:01:16.911913Z 0 [Warning] Couldn't load plugin named 'rpl_semi_sync_master' with soname 'semisync_master.so'.
2020-10-12T15:01:16.911922Z 0 [ERROR] Function 'rpl_semi_sync_slave' already exists
2020-10-12T15:01:16.911925Z 0 [Warning] Couldn't load plugin named 'rpl_semi_sync_slave' with soname 'semisync_slave.so'.
2020-10-12T15:01:16.912114Z 0 [Note] Semi-sync replication initialized for transactions.
2020-10-12T15:01:16.912123Z 0 [Note] Semi-sync replication enabled on the master.
2020-10-12T15:01:16.912550Z 0 [Note] Recovering after a crash using /data/mysql/log/mysql-bin
2020-10-12T15:01:16.915516Z 0 [Note] Starting crash recovery...
2020-10-12T15:01:16.915627Z 0 [Note] Crash recovery finished.
2020-10-12T15:01:16.916955Z 0 [Note] Starting ack receiver thread
2020-10-12T15:01:16.925903Z 0 [Note] Skipping generation of RSA key pair as key files are present in data directory.
2020-10-12T15:01:16.926134Z 0 [Note] Server hostname (bind-address): '*'; port: 3306
2020-10-12T15:01:16.927098Z 0 [Note] IPv6 is available.
2020-10-12T15:01:16.927120Z 0 [Note]   - '::' resolves to '::';
2020-10-12T15:01:16.927196Z 0 [Note] Server socket created on IP: '::'.
2020-10-12T15:01:16.951916Z 0 [Warning] Error during --relay-log-recovery: Could not locate rotate event from master in relay log file.
2020-10-12T15:01:16.951937Z 0 [Warning] Server was not able to find a rotate event from master server to initialize relay log recovery for channel ''. Skipping relay log recovery for the channel.
2020-10-12T15:01:16.953140Z 0 [Note] Failed to start slave threads for channel ''
2020-10-12T15:01:16.959966Z 0 [Note] Event Scheduler: Loaded 0 events
2020-10-12T15:01:16.960547Z 0 [Note] /usr/local/mysql/bin/mysqld: ready for connections.
Version: '5.7.30-log'  socket: '/usr/local/mysql/mysql.sock'  port: 3306  MySQL Community Server (GPL)
2020-10-12T15:01:17.289312Z 0 [Note] InnoDB: Buffer pool(s) load completed at 201012 23:01:17
```

> 6、模拟未完成的事务

```
root@test 11:25:46>begin ; 
Query OK, 0 rows affected (0.00 sec)

root@test 11:26:01>delete from checksum ;
Query OK, 3 rows affected (0.01 sec)

```

> 7、异常关闭MySQL实例，把checksum的数据文件替换为老版本。

```
cp checksum.ibd.bak  checksum.ibd 

```


> 8、启动数据检查日志，发现未完成事务回滚，但是没产生任何影响。

```

root@test 11:29:20>select count(*) from checksum;
+----------+
| count(*) |
+----------+
|        3 |
+----------+
1 row in set (0.01 sec)
```

## 总结

>1、由于MySQL的检查点较快，无法像PG那样模拟老版本数据文件应用检查点之后的redo日志，但是可以确定，MySQL使用老版本的数据文件，crash recover时回滚无效，也没有报错。
>2、恢复的起点是ibdata1第一个page记录的LSN或者dblwr中LSN,该LSN不一定是检查点。
>3、MySQL启动没有验证数据文件的一致性，较老的数据文件也可以正常读取。




