---
layout: post
title:  "数据库实例崩溃恢复-PostgreSQL"
date:   2020-10-09 09:40 +0800
categories: PostgreSQL
tags: 崩溃 恢复 数据一致性
author: qinghui.guo
mathjax: true
---

* content
{:toc}


## 什么是crash recover？

> 由于故障导致Instance异常关闭或由于执行了pg_ctl stop -m immediate都会导致数据库实例在重启时执行实例恢复，由后台进程完成，不需要人工干预,恢复的时间通过max_wal_size控制。


## 为什么要进行crash recover

>正常运行期间，里面存在很多脏块，如果实例异常，导致脏块未写入磁盘，如果没有什么手段把崩溃后的脏块找回来，肯定会出现数据不一致的情况。当然关系数据库设计之初，这种情况就已经考虑了，通过重做日志，完全可以实现实例crash不丢数据。
总而言之，所有已提交的事务，都是可以恢复的，这样才能保证数据的一致性。


## 实例恢复的过程

> **LSN（Log Sequence Number**
wal日志为了replay的有序性需要加上编号。实现的时候，是按日志的产生的顺序写入磁盘的，即使是写到磁盘缓冲区中，也是按产生的顺序一次写到日志缓冲区中，再将日志顺序写到磁盘。
因此采用日志在日志文件中的偏移来代替这个日志编号，可以通过日志编号迅速定位到日志。这个日志编号就叫做lsn。



> **恢复过程**
1、初始化内存，启动后台进程。
2、pg在启动时读取pg_control文件内容。如果state为’in production’，PostgreSQL将进入恢复模式，因为这意味着数据库没有正常停止；如果为’shutdown’，将进入正常启动模式。
3、pg从相应的WAL段文件中读取最新的检查点记录（位于pg_control文件中），并从记录中获取重做点。如果最新的检查点记录无效（invalid），pg将读取前一个检查点的记录。如果两个记录都不可读，将放弃恢复。注意，从11版本开始不会再存储前一个检查点的记录信息。
4、使用合适的资源管理器从重做点开始按顺序读取和重放XLOG记录，直到最新WAL文件的最后位置。当遇到备份块时，无论其LSN如何，都会将覆盖相应表的页面。否则仅当此xlog记录LSN>相应页面的pd_lsn时，才会重放该XLOG记录。

**强制关闭实例**
 pg_ctl stop -m immediate

```
2020-10-10 21:25:08.995 CST [2172] LOG:  received immediate shutdown request
2020-10-10 21:25:09.008 CST [2300] WARNING:  terminating connection because of crash of another server process
2020-10-10 21:25:09.008 CST [2300] DETAIL:  The postmaster has commanded this server process to roll back the current transaction and exit, because another server process exited abnormally and possibly corrupted shared memory.
2020-10-10 21:25:09.008 CST [2300] HINT:  In a moment you should be able to reconnect to the database and repeat your command.
2020-10-10 21:25:09.008 CST [2178] WARNING:  terminating connection because of crash of another server process
2020-10-10 21:25:09.008 CST [2178] DETAIL:  The postmaster has commanded this server process to roll back the current transaction and exit, because another server process exited abnormally and possibly corrupted shared memory.
2020-10-10 21:25:09.008 CST [2178] HINT:  In a moment you should be able to reconnect to the database and repeat your command.
2020-10-10 21:25:09.023 CST [2172] LOG:  database system is shut down
```
> 检查控制文件状态
检查点位置为:1/7F8D4B10

```
[postgres@qinghui-pc ~]$ pg_controldata 
pg_control version number:            1100
Catalog version number:               201809051
Database system identifier:           6842561910620900147
Database cluster state:               in production
pg_control last modified:             Sat 10 Oct 2020 09:23:00 PM CST
Latest checkpoint location:           1/7F8D4B10
Latest checkpoint's REDO location:    1/7F8D4B10
Latest checkpoint's REDO WAL file:    00000001000000010000007F
Latest checkpoint's TimeLineID:       1
Latest checkpoint's PrevTimeLineID:   1
Latest checkpoint's full_page_writes: on
Latest checkpoint's NextXID:          0:749
Latest checkpoint's NextOID:          33179
Latest checkpoint's NextMultiXactId:  1
Latest checkpoint's NextMultiOffset:  0
Latest checkpoint's oldestXID:        561
Latest checkpoint's oldestXID's DB:   1
Latest checkpoint's oldestActiveXID:  0
Latest checkpoint's oldestMultiXid:   1
Latest checkpoint's oldestMulti's DB: 13287
Latest checkpoint's oldestCommitTsXid:0
Latest checkpoint's newestCommitTsXid:0
Time of latest checkpoint:            Sat 10 Oct 2020 06:01:16 PM CST
Fake LSN counter for unlogged rels:   0/1
Minimum recovery ending location:     0/0
Min recovery ending loc's timeline:   0
Backup start location:                0/0
Backup end location:                  0/0
End-of-backup record required:        no
wal_level setting:                    replica
wal_log_hints setting:                off
max_connections setting:              100
max_worker_processes setting:         8
max_prepared_xacts setting:           0
max_locks_per_xact setting:           64
track_commit_timestamp setting:       off
Maximum data alignment:               8
Database block size:                  8192
Blocks per segment of large relation: 131072
WAL block size:                       8192
Bytes per WAL segment:                16777216
Maximum length of identifiers:        64
Maximum columns in an index:          32
Maximum size of a TOAST chunk:        1996
Size of a large-object chunk:         2048
Date/time type storage:               64-bit integers
Float4 argument passing:              by value
Float8 argument passing:              by value
Data page checksum version:           0
Mock authentication nonce:            b76ef4cc1642cbf88e6fff44c8a02a341834729c30f8ae683fd563312244c061
```
> 启动数据库
redo应用的LSN为：1/7F8D4B80(**大于1/7F8D4B10（检查点LSN），现象表明日志应用不一定从检查开始，也可能延后一些**)
```
2020-10-10 21:28:36.812 CST [2433] LOG:  database system was interrupted; last known up at 2020-10-10 21:23:00 CST
2020-10-10 21:28:36.857 CST [2433] LOG:  database system was not properly shut down; automatic recovery in progress
2020-10-10 21:28:36.858 CST [2433] LOG:  redo starts at 1/7F8D4B80
2020-10-10 21:28:37.883 CST [2433] LOG:  invalid record length at 1/89375790: wanted 24, got 0
2020-10-10 21:28:37.883 CST [2433] LOG:  redo done at 1/89375518
2020-10-10 21:28:37.883 CST [2433] LOG:  last completed transaction was at log time 2020-10-10 21:25:05.16881+08
2020-10-10 21:28:38.060 CST [2431] LOG:  database system is ready to accept connections
```

>**注意**：
recover过程非备份区块重做不是幂等的，多次重放将会导致不一致，***另外recover过程不会做数据文件一致性校验，只要LSN大于数据块的LSN就需要应用日志，如果把一个老版本的数据文件拷贝到一个crash的实例下，检查点之后的日志都会重做一遍。***




>>注：在数学与计算机学中，幂等操作是指其执行任意多次所产生的影响均与执行一次相同，即f(f(x))=f(x)。

## 总结

1、由于recover过程并不是全部幂等的，多次应用日志会导致数据不一致
2、恢复的起点是检查点的LSN,但是恢复日志中表现出有时比检查点的LSN大一些。
3、PG启动没有验证数据文件的一致性，较老的数据文件也可以正常读取。




