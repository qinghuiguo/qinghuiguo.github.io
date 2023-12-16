---
layout: post
title:  "原理分析1-统计信息pg_stat_user_tables"
date:   2021-09-25 20:40 +0800
categories: PostgreSQL
tags: dml统计 统计信息 触发策略 
author: qinghui.guo
mathjax: true
---
* content
{:toc}

## 摘要
>上文介绍了Oracle收集统计信息策略，以及统计信息过期的逻辑，并没有介绍统计信息过期后收集的后台任务。本文会介绍一下，PG统计信息的收集策略，以及统计信息收集的触发阈值；另外PG会回收死堆元，死堆元的回收阈值和统计信息阈值类似。本文并没有介绍统计信息过期后的收集频率和收集方法，只是介绍了统计信息过期过程，或者说统计信息到了应该收集的阈值过程。关于后台任务如何收集过期的统计信息，将和Oracle和MySQL的统计信息收集任务做对比。

## 结论假设

>1、表数据变化信息收集策略，dml ,create as select ,truncate 的表现
- 增删改的变化，会记录到n_tup_ins，n_tup_upd，n_tup_del，n_tup_hot_upd
- create as select 的数据变化，会记录n_tup_ins
- truncate 表，n_live_tup和n_dead_tup记录会更新为0


>2、表收集统计信息的判断逻辑，触发统计信息收集的阈值
Autovacuum ANALYZE threshold for a table = autovacuum_analyze_scale_factor * number of tuples + autovacuum_analyze_threshold 
>3、PG比较特殊，出了收集统计信息，还会触发回收死堆元
>表数据的变化量超过阈值之后，autovaccum 就可以进行回收了。
自动回收触发阈值
Autovacuum VACUUM thresold for a table = autovacuum_vacuum_scale_factor * number of tuples + autovacuum_vacuum_threshold
insert 触发阈值
Autovacuum VACUUM thresold for a table = autovacuum_vacuum_insert_threshold* number of tuples  + autovacuum_vacuum_insert_threshold
>PG的统计信息记录在文件中，通过内部函数进行读取，这点也是比较特殊。
>需要特别注意，触发阈值的计算，应该是dml操作后记录的n_live_tup，本文按照dml之前的行数计算是不准确的。


## 14beta3版本模拟测试 

>1、模拟create as select 和truncate 操作。

```
postgres=# drop table test ; 
DROP TABLE
postgres=# create table test as select * from pg_class;  
SELECT 398
postgres=# select now();
-[ RECORD 1 ]----------------------
now | 2021-09-25 20:03:36.708725+08


postgres=# select * from pg_stat_user_tables where relname='test';  
-[ RECORD 1 ]-------+-------
relid               | 16434
schemaname          | public
relname             | test
seq_scan            | 0
seq_tup_read        | 0
idx_scan            | 
idx_tup_fetch       | 
n_tup_ins           | 398
n_tup_upd           | 0
n_tup_del           | 0
n_tup_hot_upd       | 0
n_live_tup          | 398
n_dead_tup          | 0
n_mod_since_analyze | 398
n_ins_since_vacuum  | 398
last_vacuum         | 
last_autovacuum     | 
last_analyze        | 
last_autoanalyze    | 
vacuum_count        | 0
autovacuum_count    | 0
analyze_count       | 0
autoanalyze_count   | 0

--收集万统计信息后，n_mod_since_analyze字段更新为0
postgres=# select now();
-[ RECORD 1 ]----------------------
now | 2021-09-25 20:04:11.683122+08

postgres=# select * from pg_stat_user_tables where relname='test';
-[ RECORD 1 ]-------+------------------------------
relid               | 16434
schemaname          | public
relname             | test
seq_scan            | 0
seq_tup_read        | 0
idx_scan            | 
idx_tup_fetch       | 
n_tup_ins           | 398
n_tup_upd           | 0
n_tup_del           | 0
n_tup_hot_upd       | 0
n_live_tup          | 398
n_dead_tup          | 0
n_mod_since_analyze | 0
n_ins_since_vacuum  | 398
last_vacuum         | 
last_autovacuum     | 
last_analyze        | 
last_autoanalyze    | 2021-09-25 20:04:12.671321+08
vacuum_count        | 0
autovacuum_count    | 0
analyze_count       | 0
autoanalyze_count   | 1

postgres=# select now();
-[ RECORD 1 ]----------------------
now | 2021-09-25 20:08:41.931437+08

---模拟trancate操作
postgres=# truncate table test;
TRUNCATE TABLE
postgres=# select now();
-[ RECORD 1 ]----------------------
now | 2021-09-25 20:08:55.594144+08

--truncate 操作之后，更新n_live_tup和n_dead_tup
postgres=# select * from pg_stat_user_tables where relname='test';
-[ RECORD 1 ]-------+------------------------------
relid               | 16434
schemaname          | public
relname             | test
seq_scan            | 0
seq_tup_read        | 0
idx_scan            | 
idx_tup_fetch       | 
n_tup_ins           | 398
n_tup_upd           | 0
n_tup_del           | 0
n_tup_hot_upd       | 0
n_live_tup          | 0
n_dead_tup          | 0
n_mod_since_analyze | 0
n_ins_since_vacuum  | 0
last_vacuum         | 
last_autovacuum     | 
last_analyze        | 
last_autoanalyze    | 2021-09-25 20:04:12.671321+08
vacuum_count        | 0
autovacuum_count    | 0
analyze_count       | 0
autoanalyze_count   | 1
```


>2、模拟dml操作

```
--创建测试表
postgres=# create table test as select * from pg_class;  
SELECT 398
postgres=#  select now(); 
-[ RECORD 1 ]----------------------
now | 2021-09-25 20:30:23.112004+08
--查看统计信息
postgres=# select * from pg_stat_user_tables where relname='test';   
-[ RECORD 1 ]-------+-------
relid               | 16445
schemaname          | public
relname             | test
seq_scan            | 0
seq_tup_read        | 0
idx_scan            | 
idx_tup_fetch       | 
n_tup_ins           | 398
n_tup_upd           | 0
n_tup_del           | 0
n_tup_hot_upd       | 0
n_live_tup          | 398
n_dead_tup          | 0
n_mod_since_analyze | 398
n_ins_since_vacuum  | 398
last_vacuum         | 
last_autovacuum     | 
last_analyze        | 
last_autoanalyze    | 
vacuum_count        | 0
autovacuum_count    | 0
analyze_count       | 0
autoanalyze_count   | 0

postgres=# 
--更新398条记录
postgres=# update test set relname =22; 
UPDATE 398
postgres=# select * from test ; 
-[ RECORD 1 ]-------+-----------------------------------------
oid                 | 3575
relname             | 22
relnamespace        | 11
reltype             | 0
reloftype           | 0
relowner            | 10
relam               | 403
relfilenode         | 3575
reltablespace       | 0
relpages            | 1
reltuples           | 0
relallvisible       | 0
reltoastrelid       | 0
relhasindex         | f
relisshared         | f
relpersistence      | p
relkind             | i
relnatts            | 2
relchecks           | 0
relhasrules         | f
relhastriggers      | f
relhassubclass      | f
relrowsecurity      | f
relforcerowsecurity | f
relispopulated      | t
relreplident        | n
relispartition      | f
relrewrite          | 0
relfrozenxid        | 0
relminmxid          | 0
relacl              | 
reloptions          | 
relpartbound        | 
-[ RECORD 2 ]-------+-----------------------------------------
oid                 | 13714
relname             | 22
relnamespace        | 99

--删除210条记录
postgres=# delete from test where oid >4000;
DELETE 210

--可以看到统计信息，插入了398条，更新了398，删除了210，记录数为188
n_mod_since_analyze和n_ins_since_vacuum记录数为0，是因为统计信息收集了。
postgres=# select * from pg_stat_user_tables where relname='test';   
-[ RECORD 1 ]-------+------------------------------
relid               | 16445
schemaname          | public
relname             | test
seq_scan            | 3
seq_tup_read        | 1194
idx_scan            | 
idx_tup_fetch       | 
n_tup_ins           | 398
n_tup_upd           | 398
n_tup_del           | 210
n_tup_hot_upd       | 2
n_live_tup          | 188
n_dead_tup          | 0
n_mod_since_analyze | 0
n_ins_since_vacuum  | 0
last_vacuum         | 
last_autovacuum     | 2021-09-25 20:33:14.00916+08
last_analyze        | 
last_autoanalyze    | 2021-09-25 20:33:14.011364+08
vacuum_count        | 0
autovacuum_count    | 2
analyze_count       | 0
autoanalyze_count   | 2

```


> 3、模拟统计信息和autovaccum触发阈值
> Autovacuum VACUUM thresold for a table = autovacuum_vacuum_scale_factor * number of tuples + autovacuum_vacuum_threshold
>Autovacuum VACUUM thresold for a table = autovacuum_vacuum_insert_threshold* number of tuples  + >autovacuum_vacuum_insert_threshold
Autovacuum ANALYZE threshold for a table = autovacuum_analyze_scale_factor * number of tuples + autovacuum_analyze_threshold 
> autovacuum_analyze_threshold 50 
autovacuum_vacuum_threshold  50
autovacuum_vacuum_scale_factor | 0.2
autovacuum_analyze_scale_factor | 0.1
autovacuum_vacuum_insert_scale_factor 0.2
autovacuum_vacuum_insert_threshold | 1000

----


> 3.1统计信息收集阈值模拟
Autovacuum ANALYZE threshold for a table = autovacuum_analyze_scale_factor * number of tuples + autovacuum_analyze_threshold 

```
postgres=# select * from pg_stat_user_tables where relname='test';  
-[ RECORD 1 ]-------+------------------------------
relid               | 16445
schemaname          | public
relname             | test
seq_scan            | 3
seq_tup_read        | 1194
idx_scan            | 
idx_tup_fetch       | 
n_tup_ins           | 799
n_tup_upd           | 398
n_tup_del           | 210
n_tup_hot_upd       | 2
n_live_tup          | 401
n_dead_tup          | 0
n_mod_since_analyze | 0
n_ins_since_vacuum  | 401
last_vacuum         | 
last_autovacuum     | 2021-09-25 20:33:14.00916+08
last_analyze        | 
last_autoanalyze    | 2021-09-25 20:45:14.656733+08
vacuum_count        | 0
autovacuum_count    | 2
analyze_count       | 0
autoanalyze_count   | 3

--触发条件  0.1*401 +50 =90


postgres=# insert into test select * from pg_class limit 91 ; select now();
INSERT 0 91
-[ RECORD 1 ]----------------------
now | 2021-09-25 20:49:51.103724+08
--可以看到n_mod_since_analyze 为91
postgres=# select * from pg_stat_user_tables where relname='test';  select now();
-[ RECORD 1 ]-------+------------------------------
relid               | 16445
schemaname          | public
relname             | test
seq_scan            | 3
seq_tup_read        | 1194
idx_scan            | 
idx_tup_fetch       | 
n_tup_ins           | 890
n_tup_upd           | 398
n_tup_del           | 210
n_tup_hot_upd       | 2
n_live_tup          | 492
n_dead_tup          | 0
n_mod_since_analyze | 91
n_ins_since_vacuum  | 492
last_vacuum         | 
last_autovacuum     | 2021-09-25 20:33:14.00916+08
last_analyze        | 
last_autoanalyze    | 2021-09-25 20:45:14.656733+08
vacuum_count        | 0
autovacuum_count    | 2
analyze_count       | 0
autoanalyze_count   | 3

-[ RECORD 1 ]----------------------
now | 2021-09-25 20:49:53.112268+08



--收集万统计信息后，n_mod_since_analyze更新为0
postgres=# select * from pg_stat_user_tables where relname='test';  select now();
-[ RECORD 1 ]-------+------------------------------
relid               | 16445
schemaname          | public
relname             | test
seq_scan            | 3
seq_tup_read        | 1194
idx_scan            | 
idx_tup_fetch       | 
n_tup_ins           | 890
n_tup_upd           | 398
n_tup_del           | 210
n_tup_hot_upd       | 2
n_live_tup          | 492
n_dead_tup          | 0
n_mod_since_analyze | 0
n_ins_since_vacuum  | 492
last_vacuum         | 
last_autovacuum     | 2021-09-25 20:33:14.00916+08
last_analyze        | 
last_autoanalyze    | 2021-09-25 20:50:14.904217+08
vacuum_count        | 0
autovacuum_count    | 2
analyze_count       | 0
autoanalyze_count   | 4
```

> 3.2 模拟autovacuum 触发条件insert 阈值

Autovacuum VACUUM thresold for a table = autovacuum_vacuum_insert_threshold* number of tuples  + autovacuum_vacuum_insert_threshold

--0.2*641+100=1208.4 -1042 =166.4 
n_ins_since_vacuum 1042值需要大于insert 的触发阈值，所以只需模拟插入166条记录。

```

postgres=# select * from pg_stat_user_tables where relname='test';  select now();
-[ RECORD 1 ]-------+------------------------------
relid               | 16445
schemaname          | public
relname             | test
seq_scan            | 3
seq_tup_read        | 1194
idx_scan            | 
idx_tup_fetch       | 
n_tup_ins           | 1440
n_tup_upd           | 398
n_tup_del           | 210
n_tup_hot_upd       | 2
n_live_tup          | 1042
n_dead_tup          | 0
n_mod_since_analyze | 0
n_ins_since_vacuum  | 1042
last_vacuum         | 
last_autovacuum     | 2021-09-25 20:33:14.00916+08
last_analyze        | 
last_autoanalyze    | 2021-09-25 21:21:16.407244+08
vacuum_count        | 0
autovacuum_count    | 2
analyze_count       | 0
autoanalyze_count   | 6

-[ RECORD 1 ]----------------------
now | 2021-09-25 21:24:00.909854+08
----模拟插入166
postgres=# insert into test select * from pg_class limit 167 ;select now();
INSERT 0 167
-[ RECORD 1 ]----------------------
now | 2021-09-25 21:24:18.140023+08

--可以看到n_ins_since_vacuum = 1209，达到触发阈值，等待autovacuum回收。
postgres=# select * from pg_stat_user_tables where relname='test';  select now();
-[ RECORD 1 ]-------+------------------------------
relid               | 16445
schemaname          | public
relname             | test
seq_scan            | 3
seq_tup_read        | 1194
idx_scan            | 
idx_tup_fetch       | 
n_tup_ins           | 1607
n_tup_upd           | 398
n_tup_del           | 210
n_tup_hot_upd       | 2
n_live_tup          | 1209
n_dead_tup          | 0
n_mod_since_analyze | 167
n_ins_since_vacuum  | 1209
last_vacuum         | 
last_autovacuum     | 2021-09-25 20:33:14.00916+08
last_analyze        | 
last_autoanalyze    | 2021-09-25 21:21:16.407244+08
vacuum_count        | 0
autovacuum_count    | 2
analyze_count       | 0
autoanalyze_count   | 6

-[ RECORD 1 ]---------------------
now | 2021-09-25 21:24:18.95219+08


postgres=# select * from pg_stat_user_tables where relname='test';  select now();
-[ RECORD 1 ]-------+------------------------------
relid               | 16445
schemaname          | public
relname             | test
seq_scan            | 3
seq_tup_read        | 1194
idx_scan            | 
idx_tup_fetch       | 
n_tup_ins           | 1607
n_tup_upd           | 398
n_tup_del           | 210
n_tup_hot_upd       | 2
n_live_tup          | 1209
n_dead_tup          | 0
n_mod_since_analyze | 167
n_ins_since_vacuum  | 1209
last_vacuum         | 
last_autovacuum     | 2021-09-25 20:33:14.00916+08
last_analyze        | 
last_autoanalyze    | 2021-09-25 21:21:16.407244+08
vacuum_count        | 0
autovacuum_count    | 2
analyze_count       | 0
autoanalyze_count   | 6

-[ RECORD 1 ]----------------------
now | 2021-09-25 21:25:00.312934+08

-- 可以看到autovacuum 执行了，并且同时更新了统计信息，n_ins_since_vacuum变为0

postgres=# select * from pg_stat_user_tables where relname='test';  select now();
-[ RECORD 1 ]-------+------------------------------
relid               | 16445
schemaname          | public
relname             | test
seq_scan            | 3
seq_tup_read        | 1194
idx_scan            | 
idx_tup_fetch       | 
n_tup_ins           | 1607
n_tup_upd           | 398
n_tup_del           | 210
n_tup_hot_upd       | 2
n_live_tup          | 1209
n_dead_tup          | 0
n_mod_since_analyze | 0
n_ins_since_vacuum  | 0
last_vacuum         | 
last_autovacuum     | 2021-09-25 21:25:16.546408+08
last_analyze        | 
last_autoanalyze    | 2021-09-25 21:25:16.572833+08
vacuum_count        | 0
autovacuum_count    | 3
analyze_count       | 0
autoanalyze_count   | 7

-[ RECORD 1 ]----------------------
now | 2021-09-25 21:25:36.791924+08
```

> 3.3 模拟autovacuum 触发条件 ,delete和update 阈值


Autovacuum VACUUM thresold for a table = autovacuum_vacuum_scale_factor * number of tuples + autovacuum_vacuum_threshold
0.2*1209+50=291.8

with t1 as (select ctid from test limit 292)  delete from test where ctid in (select ctid from t1);
select * from pg_stat_user_tables where relname='test';  select now();

```
postgres=#  select * from pg_stat_user_tables where relname='test';  select now();
-[ RECORD 1 ]-------+------------------------------
relid               | 16445
schemaname          | public
relname             | test
seq_scan            | 3
seq_tup_read        | 1194
idx_scan            | 
idx_tup_fetch       | 
n_tup_ins           | 1607
n_tup_upd           | 398
n_tup_del           | 210
n_tup_hot_upd       | 2
n_live_tup          | 1209
n_dead_tup          | 0
n_mod_since_analyze | 0
n_ins_since_vacuum  | 0
last_vacuum         | 
last_autovacuum     | 2021-09-25 21:25:16.546408+08
last_analyze        | 
last_autoanalyze    | 2021-09-25 21:25:16.572833+08
vacuum_count        | 0
autovacuum_count    | 3
analyze_count       | 0
autoanalyze_count   | 7

-[ RECORD 1 ]----------------------
now | 2021-09-25 21:35:10.474517+08
--删除292条记录
postgres=# with t1 as (select ctid from test limit 292)  delete from test where ctid in (select ctid from t1);select now();
DELETE 292
              now              
-------------------------------
 2021-09-25 21:40:58.620179+08

--可以看到n_dead_tup=292
postgres=# select * from pg_stat_user_tables where relname='test';  select now();
-[ RECORD 1 ]-------+------------------------------
relid               | 16445
schemaname          | public
relname             | test
seq_scan            | 7
seq_tup_read        | 2715
idx_scan            | 
idx_tup_fetch       | 
n_tup_ins           | 1607
n_tup_upd           | 398
n_tup_del           | 502
n_tup_hot_upd       | 2
n_live_tup          | 917
n_dead_tup          | 292
n_mod_since_analyze | 292
n_ins_since_vacuum  | 0
last_vacuum         | 
last_autovacuum     | 2021-09-25 21:25:16.546408+08
last_analyze        | 
last_autoanalyze    | 2021-09-25 21:25:16.572833+08
vacuum_count        | 0
autovacuum_count    | 3
analyze_count       | 0
autoanalyze_count   | 7

-[ RECORD 1 ]--------------------
now | 2021-09-25 21:41:02.4531+08
 
--可以看到触发autovacuum，先更新回收死堆元，然后收集统计信息，n_dead_tup变为0
postgres=# select * from pg_stat_user_tables where relname='test';  select now();
-[ RECORD 1 ]-------+------------------------------
relid               | 16445
schemaname          | public
relname             | test
seq_scan            | 7
seq_tup_read        | 2715
idx_scan            | 
idx_tup_fetch       | 
n_tup_ins           | 1607
n_tup_upd           | 398
n_tup_del           | 502
n_tup_hot_upd       | 2
n_live_tup          | 917
n_dead_tup          | 0
n_mod_since_analyze | 0
n_ins_since_vacuum  | 0
last_vacuum         | 
last_autovacuum     | 2021-09-25 21:41:17.383646+08
last_analyze        | 
last_autoanalyze    | 2021-09-25 21:41:17.395919+08
vacuum_count        | 0
autovacuum_count    | 4
analyze_count       | 0
autoanalyze_count   | 8

-[ RECORD 1 ]----------------------
now | 2021-09-25 21:41:33.879876+08

---模拟未触发的场景

Autovacuum VACUUM thresold for a table = autovacuum_vacuum_scale_factor * number of tuples + autovacuum_vacuum_threshold
0.2*917+50=233.4

postgres=#  select * from pg_stat_user_tables where relname='test';  select now();
-[ RECORD 1 ]-------+------------------------------
relid               | 16445
schemaname          | public
relname             | test
seq_scan            | 7
seq_tup_read        | 2715
idx_scan            | 
idx_tup_fetch       | 
n_tup_ins           | 1607
n_tup_upd           | 398
n_tup_del           | 502
n_tup_hot_upd       | 2
n_live_tup          | 917
n_dead_tup          | 0
n_mod_since_analyze | 0
n_ins_since_vacuum  | 0
last_vacuum         | 
last_autovacuum     | 2021-09-25 21:41:17.383646+08
last_analyze        | 
last_autoanalyze    | 2021-09-25 21:41:17.395919+08
vacuum_count        | 0
autovacuum_count    | 4
analyze_count       | 0
autoanalyze_count   | 8

-[ RECORD 1 ]----------------------
now | 2021-09-25 21:48:48.883104+08

postgres=# 
postgres=# 
postgres=# with t1 as (select ctid from test limit 232)  delete from test where ctid in (select ctid from t1);select now();
DELETE 232
-[ RECORD 1 ]----------------------
now | 2021-09-25 21:49:01.016617+08

postgres=# select * from pg_stat_user_tables where relname='test';  select now();
-[ RECORD 1 ]-------+------------------------------
relid               | 16445
schemaname          | public
relname             | test
seq_scan            | 9
seq_tup_read        | 3864
idx_scan            | 
idx_tup_fetch       | 
n_tup_ins           | 1607
n_tup_upd           | 398
n_tup_del           | 734
n_tup_hot_upd       | 2
n_live_tup          | 685
n_dead_tup          | 232
n_mod_since_analyze | 232
n_ins_since_vacuum  | 0
last_vacuum         | 
last_autovacuum     | 2021-09-25 21:41:17.383646+08
last_analyze        | 
last_autoanalyze    | 2021-09-25 21:41:17.395919+08
vacuum_count        | 0
autovacuum_count    | 4
analyze_count       | 0
autoanalyze_count   | 8

-[ RECORD 1 ]----------------------
now | 2021-09-25 21:49:01.835688+08


postgres=# select * from pg_stat_user_tables where relname='test';  select now();
-[ RECORD 1 ]-------+------------------------------
relid               | 16445
schemaname          | public
relname             | test
seq_scan            | 9
seq_tup_read        | 3864
idx_scan            | 
idx_tup_fetch       | 
n_tup_ins           | 1607
n_tup_upd           | 398
n_tup_del           | 734
n_tup_hot_upd       | 2
n_live_tup          | 685
n_dead_tup          | 232
n_mod_since_analyze | 0
n_ins_since_vacuum  | 0
last_vacuum         | 
last_autovacuum     | 2021-09-25 21:41:17.383646+08
last_analyze        | 
last_autoanalyze    | 2021-09-25 21:49:17.877156+08
vacuum_count        | 0
autovacuum_count    | 4
analyze_count       | 0
autoanalyze_count   | 9

-[ RECORD 1 ]----------------------
now | 2021-09-25 21:49:57.084457+08
```

>4、实例运行的时候统计信息存储在pg_stat_tmp

postgres=# select pg_backend_pid();
-[ RECORD 1 ]--+------
pg_backend_pid | 46073

可以看到回到pg_stat_tmp目录下读取统计信息

 strace 46073
``` 
 [root@master-node ~]# strace -p 46073 
strace: Process 46073 attached
epoll_wait(4, [{EPOLLIN, {u32=43557608, u64=43557608}}], 1, -1) = 1
recvfrom(9, "Q\0\0\0<select * from pg_stat_user_"..., 8192, 0, NULL, NULL) = 61
lseek(6, 0, SEEK_END)                   = 106496
lseek(7, 0, SEEK_END)                   = 32768
lseek(12, 0, SEEK_END)                  = 49152
lseek(13, 0, SEEK_END)                  = 40960
lseek(14, 0, SEEK_END)                  = 32768
lseek(15, 0, SEEK_END)                  = 16384
lseek(16, 0, SEEK_END)                  = 16384
lseek(17, 0, SEEK_END)                  = 8192
lseek(18, 0, SEEK_END)                  = 16384
lseek(19, 0, SEEK_END)                  = 16384
brk(NULL)                               = 0x2b00000
brk(0x2b40000)                          = 0x2b40000
lseek(14, 0, SEEK_END)                  = 32768
lseek(17, 0, SEEK_END)                  = 8192
open("pg_stat_tmp/global.stat", O_RDONLY) = 49
fstat(49, {st_mode=S_IFREG|0600, st_size=1335, ...}) = 0
mmap(NULL, 4096, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f855d37c000
read(49, "\242\274\245\1I\345\3264\321o\2\0W\0\0\0\0\0\0\0\1\0\0\0\0\0\0\0003\332\0\0"..., 4096) = 1335
close(49)                               = 0
munmap(0x7f855d37c000, 4096)            = 0
sendto(8, "\1\0\0\0 \0\0\0\361\305M7\321o\2\0\321$F7\321o\2\0D6\0\0\0\0\0\0", 32, 0, NULL, 0) = 32
select(0, NULL, NULL, NULL, {tv_sec=0, tv_usec=10000}) = 0 (Timeout)
open("pg_stat_tmp/global.stat", O_RDONLY) = 49
fstat(49, {st_mode=S_IFREG|0600, st_size=1335, ...}) = 0
mmap(NULL, 4096, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f855d37c000
read(49, "\242\274\245\1\372\306M7\321o\2\0W\0\0\0\0\0\0\0\1\0\0\0\0\0\0\0003\332\0\0"..., 4096) = 1335
close(49)                               = 0
munmap(0x7f855d37c000, 4096)            = 0
open("pg_stat_tmp/global.stat", O_RDONLY) = 49
fstat(49, {st_mode=S_IFREG|0600, st_size=1335, ...}) = 0
mmap(NULL, 4096, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f855d37c000
read(49, "\242\274\245\1\372\306M7\321o\2\0W\0\0\0\0\0\0\0\1\0\0\0\0\0\0\0003\332\0\0"..., 4096) = 1335
open("pg_stat_tmp/db_13892.stat", O_RDONLY) = 50
fstat(50, {st_mode=S_IFREG|0600, st_size=40705, ...}) = 0
mmap(NULL, 4096, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f855d37b000
read(50, "\242\274\245\1T\25\v\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0"..., 4096) = 4096
read(50, "\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0"..., 4096) = 4096
read(50, "\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0"..., 4096) = 4096
read(50, "\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\2"..., 4096) = 4096
brk(NULL)                               = 0x2b40000
brk(0x2b67000)                          = 0x2b67000
read(50, "\0\0\0\0\0\236\1\0\0\0\0\0\0\226\1\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0"..., 4096) = 4096
read(50, "\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0"..., 4096) = 4096
read(50, "\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0"..., 4096) = 4096
read(50, "\0\0\0\0\0\0\0T\16\v\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0"..., 4096) = 4096
read(50, "\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0"..., 4096) = 4096
read(50, "\0\0\0\0\226\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0S\v\0\0\0\0\0\0\226\0\0\0"..., 4096) = 3841
close(50)                               = 0
munmap(0x7f855d37b000, 4096)            = 0
open("pg_stat_tmp/db_0.stat", O_RDONLY) = 50
fstat(50, {st_mode=S_IFREG|0600, st_size=7960, ...}) = 0
mmap(NULL, 4096, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f855d37b000
read(50, "\242\274\245\1T\37\v\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0"..., 4096) = 4096
read(50, "\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0"..., 4096) = 3864
close(50)                               = 0
munmap(0x7f855d37b000, 4096)            = 0
close(49)                               = 0
munmap(0x7f855d37c000, 4096)            = 0
brk(NULL)                               = 0x2b67000
brk(NULL)                               = 0x2b67000
brk(0x2b00000)                          = 0x2b00000
brk(NULL)                               = 0x2b00000
sendto(8, "\31\0\0\08\0\0\0D6\0\0\0\0\0\0\0\0\0\0\0\0\0\0\253\312I\3\0\0\0\0"..., 56, 0, NULL, 0) = 56
sendto(8, "\2\0\0\0\250\3\0\0D6\0\0\10\0\0\0\1\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0"..., 936, 0, NULL, 0) = 936
sendto(8, "\2\0\0\0\230\0\0\0D6\0\0\1\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0"..., 152, 0, NULL, 0) = 152
sendto(9, "T\0\0\2\313\0\27relid\0\0\0000V\0\1\0\0\0\32\0\4\377\377\377\377\0\0s"..., 936, 0, NULL, 0) = 936
recvfrom(9, 0xd7e400, 8192, 0, NULL, NULL) = -1 EAGAIN (Resource temporarily unavailable)
epoll_wait(4, ^Cstrace: Process 46073 detached
 <detached ...>

```
## 结论

1、表的增删改产生的数据条目数变化，通过视图pg_stat_user_tables可以查看，记录在文件中。
2、表的统计信息，超过阈值后台进程会重新收集统计信息。
3、比较特殊，表的死堆元达到阈值后，会触发死堆元回收，也会更新统计信息。

