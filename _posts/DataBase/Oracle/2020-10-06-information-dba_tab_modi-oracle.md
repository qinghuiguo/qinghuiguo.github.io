---
layout: post
title:  "情报分析1-从dba_tab_modifications和dba_tab_statistics说起"
date:   2021-09-08 20:40 +0800
categories: Oracle
tags: dml统计 统计信息 触发策略 
author: qinghui.guo
mathjax: true
---
* content
{:toc}

## 假设结论：
>1、create as 语法插入的数据不会记录到dba_tab_modifications视图，只会记录到dba_tab_statistics,未标记统计信息过期，但是自动任务会收集。
>truncate table 最终会记录到dba_tab_modifications，并标记统计信息为过期。

>2、dbms_stat.FLUSH_DATABASE_MONITORING_INFO 可以刷新内存中的刷新到基表视图，收集统计信息的命令也可以触发该刷新动作
>
>3、dba_tab_modifications和dba_tab_statistics 三小时自动从内存刷新一次，超过10%的变化，标记统计信息为过期。

>4、收集统计信息，会先触发dbms_stat.FLUSH_DATABASE_MONITORING_INFO把内存的数据变化信息刷到基表视图中。
>5、dba_tab_modifications只会记录dml操作，回滚也会记录;同样会记录truncate操作，包含truncate操作删除的记录数。
>6、19C中所有这些异步动作，已经变为同步，或者是直接查询内存中pending的变化信息。
>create as 完成后直接收集一次统计信息，truncate完成后，直接标记统计信息为过期，dml后，通过视图可以直接查询到数据变化量，数据变化量超过10%，同样会同事标记统计信息为过期。


## 11G 模拟测试 

>1、create as 语法插入的数据不会记录到dba_tab_modifications视图，只会记录到dba_tab_statistics,未标记统计信息过期，但是自动任务会收集。


```sql
SYS@ora11db 09:13:45> drop table test ; 

Table dropped.

SYS@ora11db 09:13:47> 
SYS@ora11db 09:13:47> 
---创建新表
SYS@ora11db 09:13:47> create table test as select * from user_objects;

Table created.
--检查数据变化，无数据。
SYS@ora11db 09:13:53> select * from user_tab_modifications where table_name='TEST';

no rows selected
--统计信息没有收集，而且未标记过期，19C create table 会同步收集统计信息。
SYS@ora11db 09:13:59>  select s.table_name,s.NUM_ROWS,s.LAST_ANALYZED,s.STALE_STATS from user_tab_statistics s where table_name='TEST';

TABLE_NAME                       NUM_ROWS LAST_ANA STA
------------------------------ ---------- -------- ---
TEST

```


> 因为11G统计信息和表的变更是异步,需要把内存记录更新到基表中。


```sql

--从内存中刷新表的变化，期望统计信息和表的变更视图会有变化。
SYS@ora11db 09:14:05> exec dbms_stats.FLUSH_DATABASE_MONITORING_INFO;

PL/SQL procedure successfully completed.

SYS@ora11db 09:19:15>  select * from user_tab_modifications where table_name='TEST';

no rows selected

SYS@ora11db 09:19:22>  select s.table_name,s.NUM_ROWS,s.LAST_ANALYZED,s.STALE_STATS from user_tab_statistics s where table_name='TEST';

TABLE_NAME                       NUM_ROWS LAST_ANA STA
------------------------------ ---------- -------- ---
TEST
```
> 可以看到统计信息和表变更视图没有变化，考虑user_tab_modifications只会记录dml操作，create table 不会记录表变化的数据量，推测收集统计信息时会直接收集。


```sql
--调用内部收集统计信息的任务，收集统计信息，可以发现表的统计信息被收集。
SYS@ora11db 09:19:28>  exec dbms_stats.gather_database_stats_job_proc;

PL/SQL procedure successfully completed.

SYS@ora11db 09:20:00>  select s.table_name,s.NUM_ROWS,s.LAST_ANALYZED,s.STALE_STATS from user_tab_statistics s where table_name='TEST';

TABLE_NAME                       NUM_ROWS LAST_ANA STA
------------------------------ ---------- -------- ---
TEST                                 9512 09:20:00 NO

SYS@ora11db 09:20:09> select * from user_tab_modifications where table_name='TEST';

no rows selected
```

```sql
-- 模拟truncate 表操作，看最终表现
SYS@ora11db 09:20:23> truncate table test;

Table truncated.

-- 统计信息和表变更是异步的没刷新。
SYS@ora11db 09:20:47> select * from user_tab_modifications where table_name='TEST';

no rows selected

SYS@ora11db 09:20:51> select s.table_name,s.NUM_ROWS,s.LAST_ANALYZED,s.STALE_STATS from user_tab_statistics s where table_name='TEST';

TABLE_NAME                       NUM_ROWS LAST_ANA STA
------------------------------ ---------- -------- ---
TEST                                 9512 09:20:00 NO
``` 
>可以看出来，表的统计信息状态没有更新，表数据的变更也没有更新，需要调用内部刷新过程才能看到。


```sql
-- 调用刷新内存信息函数包
SYS@ora11db 09:20:57> exec dbms_stats.FLUSH_DATABASE_MONITORING_INFO;

PL/SQL procedure successfully completed.

--可以看到视图中记录，表被truncate，删除的记录数为9512
SYS@ora11db 09:21:05> select table_name,inserts,updates,deletes,timestamp,truncated from user_tab_modifications where table_name='TEST';

TABLE_NAME                        INSERTS    UPDATES    DELETES TIMESTAM TRU
------------------------------ ---------- ---------- ---------- -------- ---
TEST                                    0          0       9512 09:21:05 YES

--因为truncate的动作，表的统计信息也标记为过期了。
SYS@ora11db 09:21:13> select s.table_name,s.NUM_ROWS,s.LAST_ANALYZED,s.STALE_STATS from user_tab_statistics s where table_name='TEST';

TABLE_NAME                       NUM_ROWS LAST_ANA STA
------------------------------ ---------- -------- ---
TEST                                 9512 09:20:00 YES
```
>可以看出，truncate 操作，会记录到dba_tab_modifications ,并且更新统计信息为过期状态。

```sql

SYS@ora11db 09:21:20>  exec dbms_stats.gather_database_stats_job_proc;

PL/SQL procedure successfully completed.
--统计信息正常
SYS@ora11db 09:21:35> select s.table_name,s.NUM_ROWS,s.LAST_ANALYZED,s.STALE_STATS from user_tab_statistics s where table_name='TEST';

TABLE_NAME                       NUM_ROWS LAST_ANA STA
------------------------------ ---------- -------- ---
TEST                                    0 09:21:35 NO
--记录表变更的视图被更新，相关记录被删除。
SYS@ora11db 09:21:42> select table_name,inserts,updates,deletes,timestamp,truncated from user_tab_modifications where table_name='TEST';

no rows selected

```
>11G统计信息标记为过期，Oracle收集统计信息的过程 ，会收集统计信息，收集完成后，dba_tab_modifications信息会清空，因为统计信息已经收集。



```
SYS@ora11db 09:21:47> 
SYS@ora11db 09:22:08> 
SYS@ora11db 09:22:08> 
-- 模拟插入数据
SYS@ora11db 09:22:08> insert into TEST select *from user_objects; 

9512 rows created.

SYS@ora11db 09:22:25> 
SYS@ora11db 09:22:26> commit;             

Commit complete.
-- 统计信息没有更新，说明11G这个动作是异步的
SYS@ora11db 09:22:47> select s.table_name,s.NUM_ROWS,s.LAST_ANALYZED,s.STALE_STATS from user_tab_statistics s where table_name='TEST';

TABLE_NAME                       NUM_ROWS LAST_ANA STA
------------------------------ ---------- -------- ---
TEST                                    0 09:21:35 NO

--表的变更记录没有更新，说明11G这个动作是异步的
SYS@ora11db 09:22:57>  select table_name,inserts,updates,deletes,timestamp,truncated from user_tab_modifications where table_name='TEST';

no rows selected
```

``` sql
--刷新内存数据
SYS@ora11db 09:23:03> exec dbms_stats.FLUSH_DATABASE_MONITORING_INFO;

PL/SQL procedure successfully completed.
--表的统计信息标记为过期
SYS@ora11db 09:23:38> select s.table_name,s.NUM_ROWS,s.LAST_ANALYZED,s.STALE_STATS from user_tab_statistics s where table_name='TEST';

TABLE_NAME                       NUM_ROWS LAST_ANA STA
------------------------------ ---------- -------- ---
TEST                                    0 09:21:35 YES
--插入多少数据条目也有记录
SYS@ora11db 09:23:42> select table_name,inserts,updates,deletes,timestamp,truncated from user_tab_modifications where table_name='TEST';

TABLE_NAME                        INSERTS    UPDATES    DELETES TIMESTAM TRU
------------------------------ ---------- ---------- ---------- -------- ---
TEST                                 9512          0          0 09:23:38 NO
```
> dml操作，异步刷新到dba_tab_modifications中。

```sql

--此时调用收集统计信息的任务，可以更新统计信息
SYS@ora11db 09:23:45> 
SYS@ora11db 09:23:46> exec dbms_stats.gather_database_stats_job_proc;

PL/SQL procedure successfully completed.

SYS@ora11db 09:24:07> 
SYS@ora11db 09:24:08> select s.table_name,s.NUM_ROWS,s.LAST_ANALYZED,s.STALE_STATS from user_tab_statistics s where table_name='TEST';

TABLE_NAME                       NUM_ROWS LAST_ANA STA
------------------------------ ---------- -------- ---
TEST                                 9512 09:24:07 NO

SYS@ora11db 09:24:13> select table_name,inserts,updates,deletes,timestamp,truncated from user_tab_modifications where table_name='TEST';

no rows selected
```

```
--模拟回滚操作
SYS@ora11db 09:24:34> insert into test select * from user_objects ;

9512 rows created.

SYS@ora11db 09:24:44> rollback; 

Rollback complete.

-- 统计信息异步更新
SYS@ora11db 09:24:51> select table_name,inserts,updates,deletes,timestamp,truncated from user_tab_modifications where table_name='TEST';

no rows selected

SYS@ora11db 09:25:01> select s.table_name,s.NUM_ROWS,s.LAST_ANALYZED,s.STALE_STATS from user_tab_statistics s where table_name='TEST';

TABLE_NAME                       NUM_ROWS LAST_ANA STA
------------------------------ ---------- -------- ---
TEST                                 9512 09:24:07 NO
--调用函数，更新统计信息记录
SYS@ora11db 09:25:06> exec dbms_stats.FLUSH_DATABASE_MONITORING_INFO;

PL/SQL procedure successfully completed.


SYS@ora11db 09:25:09> select s.table_name,s.NUM_ROWS,s.LAST_ANALYZED,s.STALE_STATS from user_tab_statistics s where table_name='TEST';

TABLE_NAME                       NUM_ROWS LAST_ANA STA
------------------------------ ---------- -------- ---
TEST                                 9512 09:24:07 YES

---可以看到回滚的数据也会记录到dba_tab_modifications
SYS@ora11db 09:25:16> select table_name,inserts,updates,deletes,timestamp,truncated from user_tab_modifications where table_name='TEST';

TABLE_NAME                        INSERTS    UPDATES    DELETES TIMESTAM TRU
------------------------------ ---------- ---------- ---------- -------- ---
TEST                                 9512          0          0 09:25:09 NO
```

> dml操作，虽然回滚也会记录到dba_tab_modifications中。



```sql
--模拟收集其他表的统计信息，不相关表的表dml操作记录和统计信息状态会更新到最新，收集统计信息时，会调用FLUSH_DATABASE_MONITORING_INFO把内存中的统计数据更新到基表。
SYS@ora11db 09:25:21> exec dbms_stats.gather_database_stats_job_proc;

PL/SQL procedure successfully completed.

SYS@ora11db 09:25:37>  select s.table_name,s.NUM_ROWS,s.LAST_ANALYZED,s.STALE_STATS from user_tab_statistics s where table_name='TEST';

TABLE_NAME                       NUM_ROWS LAST_ANA STA
------------------------------ ---------- -------- ---
TEST                                 9512 09:25:37 NO

SYS@ora11db 09:25:41>  select table_name,inserts,updates,deletes,timestamp,truncated from user_tab_modifications where table_name='TEST';

no rows selected

SYS@ora11db 09:25:45>  select table_name,inserts,updates,deletes,timestamp,truncated from user_tab_modifications where table_name='TEST';

no rows selected

SYS@ora11db 10:10:10> select s.table_name,s.NUM_ROWS,s.LAST_ANALYZED,s.STALE_STATS from user_tab_statistics s where table_name='TEST';

TABLE_NAME                       NUM_ROWS LAST_ANA STA
------------------------------ ---------- -------- ---
TEST                                 9512 09:25:37 NO

SYS@ora11db 10:10:14> 
SYS@ora11db 10:10:15> 
--test 表插入数据
SYS@ora11db 10:10:15> insert into TEST select * from user_objects;

9512 rows created.

SYS@ora11db 10:10:55> commit; 

Commit complete.

SYS@ora11db 10:10:57> 
SYS@ora11db 10:10:58> 
--新建test_other表
SYS@ora11db 10:10:58> create table test_other as select * from dba_tables;

Table created.

SYS@ora11db 10:11:32> select s.table_name,s.NUM_ROWS,s.LAST_ANALYZED,s.STALE_STATS from user_tab_statistics s where table_name='TEST';

TABLE_NAME                       NUM_ROWS LAST_ANA STA
------------------------------ ---------- -------- ---
TEST                                 9512 09:25:37 NO

SYS@ora11db 10:11:42> select table_name,inserts,updates,deletes,timestamp,truncated from user_tab_modifications where table_name='TEST';

no rows selected

--收集test_other表的统计信息
SYS@ora11db 10:11:47> exec dbms_stats.gather_table_stats(ownname=>'sys',tabname=>'test_other',method_opt=>'FOR ALL COLUMNS SIZE 1',degree=>10,estimate_percent => 10, cascade=>true ,no_invalidate=>false);

PL/SQL procedure successfully completed.

SYS@ora11db 10:12:44> select table_name,inserts,updates,deletes,timestamp,truncated from user_tab_modifications where table_name='TEST_OTHER';

no rows selected

SYS@ora11db 10:12:58> select s.table_name,s.NUM_ROWS,s.LAST_ANALYZED,s.STALE_STATS from user_tab_statistics s where table_name= 'TEST_OTHER';

TABLE_NAME                       NUM_ROWS LAST_ANA STA
------------------------------ ---------- -------- ---
TEST_OTHER                           1219 10:12:44 NO

--可以看到test的统计数据已经更新。
SYS@ora11db 10:13:13> select table_name,inserts,updates,deletes,timestamp,truncated from user_tab_modifications where table_name='TEST';

TABLE_NAME                        INSERTS    UPDATES    DELETES TIMESTAM TRU
------------------------------ ---------- ---------- ---------- -------- ---
TEST                                 9512          0          0 10:12:44 NO

SYS@ora11db 10:13:20> select s.table_name,s.NUM_ROWS,s.LAST_ANALYZED,s.STALE_STATS from user_tab_statistics s where table_name='TEST';

TABLE_NAME                       NUM_ROWS LAST_ANA STA
------------------------------ ---------- -------- ---
TEST                                 9512 09:25:37 YES

SYS@ora11db 10:13:26> 
```

> 11G收集统计信息之前，会调用FLUSH_DATABASE_MONITORING_INFO把内存中的统计信息更新到基表。

## 19C 模拟测试

> 1、11G中所有的异步动作，都变成同步（表现形式为同步）。
> 2、11G中表现为异步，是因为只差基表视图信息，不访问内存，19C不止查基表信息，同样会访问内存结构，所以统计信息显得更实时（[Asktom](https://asktom.oracle.com/pls/apex/f?p=100:11:0::::P11_QUESTION_ID:9530337900346145148)。
> 3、create table as 完成后会先收集统计信息(sys表空间下表现形式和11G类似)
> 4、FLUSH_DATABASE_MONITORING_INFO命令还是可以刷新内存数据到基表




**Asktom 关于 12.2版本统计信息更新策略的描述**


I got some feedback from the Optimizer PM.

In 11g, to decide on stale, we only looked at the dictionary tables, so we had to flush out monitoring info *before* deciding if something was stale.

In 12c, thats been improved - ***we also look at the memory structures*** that hold the "pending" information (ie, that data that *would* be flushed), so if someone asks us to gather stale, we dont need to flush out the info.

So in your case, you should be able to just do a "gather stale" which will be a no-op if appropriate.



### 普通用户模拟操作

```
HR@pdb1 11:43:42> 
HR@pdb1 11:43:42> create table test as select * from user_objects; 

HR@pdb1 11:44:02> select table_name,inserts,updates,deletes,timestamp,truncated from user_tab_modifications where table_name='TEST';

no rows selected

--可以看出创建表的时候，直接收集了统计信息，注意sys 和system 表现会有差异，用户可以自己测试。
HR@pdb1 11:44:11> select s.table_name,s.NUM_ROWS,s.LAST_ANALYZED,s.STALE_STATS from user_tab_statistics s where table_name='TEST';

TABLE_NAME                       NUM_ROWS LAST_ANA STALE_S
------------------------------ ---------- -------- -------
TEST                                   34 11:43:42 NO

--插入数据测试
HR@pdb1 11:44:16>  insert into test select * from user_objects ;

35 rows created.

HR@pdb1 11:44:32> commit; 

Commit complete.

--可以看到直接可以查到insert 的条数，TIMESTAMP没变化，很可能是直接读内存结构，这点待验证，参考asktom 解答。
HR@pdb1 11:44:35> select table_name,inserts,updates,deletes,timestamp,truncated from user_tab_modifications where table_name='TEST'; 

TABLE_NAME                        INSERTS    UPDATES    DELETES TIMESTAM TRU
------------------------------ ---------- ---------- ---------- -------- ---
TEST                                   35          0          0          NO

-- 因为插入数据超过10%，统计信息也直接标记为过期，表现为同步。
HR@pdb1 11:44:39> select s.table_name,s.NUM_ROWS,s.LAST_ANALYZED,s.STALE_STATS from user_tab_statistics s where table_name='TEST';

TABLE_NAME                       NUM_ROWS LAST_ANA STALE_S
------------------------------ ---------- -------- -------
TEST                                   34 11:43:42 YES

--按照11G的逻辑，调用FLUSH_DATABASE_MONITORING_INFO刷新内存，可以看到TIMESTAM字段更新了，说明还是执行了刷新内存数据的动作。
HR@pdb1 11:44:53> ---exec dbms_stats.FLUSH_DATABASE_MONITORING_INFO;
HR@pdb1 11:45:52> select table_name,inserts,updates,deletes,timestamp,truncated from user_tab_modifications where table_name='TEST'; 

TABLE_NAME                        INSERTS    UPDATES    DELETES TIMESTAM TRU
------------------------------ ---------- ---------- ---------- -------- ---
TEST                                   35          0          0 11:45:35 NO

HR@pdb1 11:45:59> select s.table_name,s.NUM_ROWS,s.LAST_ANALYZED,s.STALE_STATS from user_tab_statistics s where table_name='TEST';

TABLE_NAME                       NUM_ROWS LAST_ANA STALE_S
------------------------------ ---------- -------- -------
TEST                                   34 11:43:42 YES

--执行truncate 操作
HR@pdb1 11:46:11> truncate table test ; 

Table truncated.

-- truncate 操作，也表现出统计信息实时更新
HR@pdb1 11:46:24> select table_name,inserts,updates,deletes,timestamp,truncated from user_tab_modifications where table_name='TEST'; 

TABLE_NAME                        INSERTS    UPDATES    DELETES TIMESTAM TRU
------------------------------ ---------- ---------- ---------- -------- ---
TEST                                   35          0         69 11:45:35 YES

HR@pdb1 11:46:29> select s.table_name,s.NUM_ROWS,s.LAST_ANALYZED,s.STALE_STATS from user_tab_statistics s where table_name='TEST';

TABLE_NAME                       NUM_ROWS LAST_ANA STALE_S
------------------------------ ---------- -------- -------
TEST                                   34 11:43:42 YES

---执行内存刷新动作，TIMESTAMP字段更新，表现出从内存中刷新统计信息
HR@pdb1 11:46:37> -- exec dbms_stats.FLUSH_DATABASE_MONITORING_INFO; 
HR@pdb1 11:46:52> select table_name,inserts,updates,deletes,timestamp,truncated from user_tab_modifications where table_name='TEST'; 

TABLE_NAME                        INSERTS    UPDATES    DELETES TIMESTAM TRU
------------------------------ ---------- ---------- ---------- -------- ---
TEST                                   35          0         69 11:46:55 YES

HR@pdb1 11:47:01> insert into test select * from test; 

0 rows created.

HR@pdb1 11:47:25> commit; 

Commit complete.
-- insert 操作也表现为同步更新视图
HR@pdb1 11:47:42> select table_name,inserts,updates,deletes,timestamp,truncated from user_tab_modifications where table_name='TEST'; 

TABLE_NAME                        INSERTS    UPDATES    DELETES TIMESTAM TRU
------------------------------ ---------- ---------- ---------- -------- ---
TEST                                   35          0         69 11:46:55 YES

-- 回滚操作也表现为同步更新视图
HR@pdb1 11:47:47> insert into test select * from user_objects ; 

35 rows created.

HR@pdb1 11:47:55> rollback; 

Rollback complete.

HR@pdb1 11:48:00> select table_name,inserts,updates,deletes,timestamp,truncated from user_tab_modifications where table_name='TEST'; 

TABLE_NAME                        INSERTS    UPDATES    DELETES TIMESTAM TRU
------------------------------ ---------- ---------- ---------- -------- ---
TEST                                   70          0         69 11:46:55 YES

HR@pdb1 11:48:05> 
```


## 结论

>1、create table as 插入的数据不会记录到dba_tab_modifications,也不会标记统计信息状态为过期，但是收集统计信息的任务调用时，肯定会收集。
>2、dbms_stat.FLUSH_DATABASE_MONITORING_INFO 可以刷新内存中的刷新到基表视图，收集统计信息命令调用时，会先调用FLUSH_DATABASE_MONITORING_INFO，把内存中的统计信息刷新到基表视图。
>3、dba_tab_modifications只记录表的dml和truncate操作，dml操作回滚了也会记录，表的变化量超过10%，统计信息为标记为过期，自动收集任务会根据过期状态收集统计信息。
>19C 统计信息表现为同步模式，同时查数据字典和内存结构，执行刷新函数，还是会把内存中的信息刷新到基表。



## 参考文档

- 19C 统计信息查内存结构
https://asktom.oracle.com/pls/apex/f?p=100:11:0::::P11_QUESTION_ID:9530337900346145148

- 11G  dba_tab_modifications介绍和实时刷新函数FLUSH_DATABASE_MONITORING_INFO
https://docs.oracle.com/cd/E11882_01/server.112/e40402/statviews_5048.htm#REFRN23280

- 12C dba_tab_modifications介绍，此时开始没有说明FLUSH_DATABASE_MONITORING_INFO，因为12C已经变成同步
https://docs.oracle.com/en/database/oracle/oracle-database/12.2/refrn/DBA_TAB_MODIFICATIONS.html#GUID-05CB1867-44F3-41AB-9D5D-CA43798593FD
- 12C FLUSH_DATABASE_MONITORING_INFO函数说明，虽然统计信息记录变成同步的，该函数还是存在。
https://docs.oracle.com/en/database/oracle/oracle-database/12.2/arpls/DBMS_STATS.html#GUID-CA79C291-B7B4-4B35-8507-454366D83A03




