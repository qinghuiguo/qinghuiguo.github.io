---
layout: post
title:  "PostgreSQL的内部结构---第一章数据库集群，数据库和表"
date:   2020-02-01 09:40:18 +0800
categories: PostgreSQL
tags: 据库集群 数据库 表
author: Hironobu SUZUKI
mathjax: true
---

* content
{:toc}

***声明***：该系列文章为Hironobu SUZUKI的私人项目，为促进PostgreSQL的技术分享，现翻译出来供大家学习。

**版权**：所有版权归 Hironobu SUZUKI所有，本系列文章只是负责翻译和分享工作，再次感谢大神。

**概述**：本系列文章，旨在分享PostgreSQL技术，推广PostgreSQL。由于该系列文章原作者还在更新中，中文版将会同步更新

[原文链接](http://www.interdb.jp/pg/pgsql01.html)


# 第1章数据库集群，数据库和表

本章和下一章总结了PostgreSQL的基本知识，以帮助阅读后续章节。 在本章中，描述了以下主题：

- 数据库集群的逻辑结构
- 数据库集群的物理结构
- 堆表文件的内部布局
- 将数据写入和读取到表的方法 

如果您已经熟悉它们，可以跳过本章。
## 1.1。 数据库集群的逻辑结构


数据库集群是PostgreSQL服务器管理的数据库集合。 如果你是第一次听到这个定义，你可能会对此感到疑惑，但PostgreSQL中的术语“数据库集群” 并不意味着“一组数据库服务器”。 PostgreSQL服务器在单个主机上运行并管理单个数据库集群。

图1.1显示了数据库集群的逻辑结构。 数据库是数据库对象的集合。 在关系数据库理论中， 数据库对象是用于存储或引用数据的数据结构。 （堆） 表是它的典型示例，还有更多像索引，序列，视图，函数等。 在PostgreSQL中，数据库本身也是数据库对象，并且在逻辑上彼此分离。 所有其他数据库对象（例如，表，索引等）属于它们各自的数据库。
**图1.1。 数据库集群的逻辑结构。**
![tupian](http://www.interdb.jp/pg/img/fig-1-01.png)

PostgreSQL中的所有数据库对象都由相应的对象标识符（OID）进行内部管理，这些标识符是无符号的4字节整数。 数据库对象与相应OID之间的关系存储在适当的系统目录中 ，具体取决于对象的类型。 例如，数据库和堆表的OID分别存储在pg_database和pg_class中 ，因此您可以通过发出以下查询来查找您想要知道的OID：

```sql
sampledb =＃SELECT datname，oid FROM pg_database WHERE datname ='sampledb';
  datname |  OID  
 ---------- + -------
  sampledb |  16384
 （1排）
 
 sampledb = #SELECT relname，oid FROM pg_class WHERE relname ='sampletbl';
   relname |  OID  
 ----------- + -------
  sampletbl |  18740 
 （1排）
```


## 1.2。 数据库集群的物理结构

数据库集群基本上是一个称为基本目录的目录 ，它包含一些子目录和大量文件。 如果执行initdb实用程序以初始化新数据库集群，则将在指定目录下创建基目录。 虽然它不是必须的，但基本目录的路径通常设置为环境变量PGD​​ATA 。

图1.2显示了PostgreSQL中数据库集群的一个示例。 数据库是base子目录下的子目录，每个表和索引（至少）一个文件存储在它所属的数据库的子目录下。 还有几个包含特定数据和配置文件的子目录。 虽然PostgreSQL支持表空间 ，但该术语的含义与其他RDBMS不同。 PostgreSQL中的表空间是一个包含基本目录之外的数据的目录。
**图1.2。 数据库集群的一个示例。**
![enter image description here](http://www.interdb.jp/pg/img/fig-1-02.png)

在以下小节中，描述了数据库集群的布局，数据库，与表和索引关联的文件以及PostgreSQL中的表空间。
### 1.2.1。 数据库集群的布局

数据库集群的布局已在官方文档中描述。 表1.1中列出了文档一部分中的主要文件和子目录：
表1.1：基本目录下的文件和子目录的布局（来自官方文档）


| 文件  |	描述  |
|  :---------------- | :-----------|
|PG_VERSION|包含PostgreSQL主版本号的文件
|pg_hba.conf| 	用于控制PosgreSQL客户端身份验证的文件
|pg_ident.conf| 	用于控制PostgreSQL用户名映射的文件
|postgresql.conf| 	用于设置配置参数的文件
|postgresql.auto.conf| 	用于存储在ALTER SYSTEM（版本9.4或更高版本）中设置的配置参数的文件
|postmaster.opts| 	记录服务器上次启动的命令行选项的文件

|   子目录  | 	描述  |
|  :----------------  | :-----------|
|base/| 	包含每个数据库子目录的子目录。
|global/| 	包含群集范围表的子目录，例如pg_database和pg_control。
|pg_commit_ts/| 	包含事务提交时间戳数据的子目录。 9.5或更高版本
|pg_clog/|（版本9.6或更早版本	包含事务提交状态数据的子目录。 它在版本10中重命名为pg_xact.CLOG将在5.4节中描述。
|pg_dynshmem/|	子目录，包含动态共享内存子系统使用的文件。 版本9.4或更高版本。
|pg_logical/| 	子目录，包含逻辑解码的状态数据。 版本9.4或更高版本。
|pg_multixact/ |	包含多次事务状态数据的子目录（用于共享行锁）
|pg_notify/| 	包含LISTEN / NOTIFY状态数据的子目录
|pg_repslot/| 	包含复制槽数据的子目录。 版本9.4或更高版本。
|pg_serial/ |	包含有关已提交的可序列化事务（版本9.1或更高版本）的信息的子目录
|pg_snapshots/| 	包含导出快照的子目录（版本9.2或更高版本）。 PostgreSQL的函数pg_export_snapshot在此子目录中创建快照信息文件。
|pg_stat/| 	包含统计子系统永久文件的子目录。
|pg_stat_tmp/| 	子目录，包含统计子系统的临时文件。
|pg_subtrans/| 	包含子事务状态数据的子目录
|pg_tblspc关联/| 	包含指向表空间的符号链接的子目录
|pg_twophase/| 	子目录，包含准备好的事务的状态文件
|pg_wal/|（版本10或更高版本） 	包含WAL（Write Ahead Logging）段文件的子目录。 它在版本10中从pg_xlog重命名。
|pg_xact/|（版本10或更高版本） 	包含事务提交状态数据的子目录。 它在版本10中从pg_clog重命名.CLOG将在5.4节中描述。
|pg_xlog/|（版本9.6或更早版本） 	包含WAL（Write Ahead Logging）段文件的子目录。 它在版本10中重命名为pg_wal 。


### 1.2.2。 数据库的布局

数据库是base子目录下的子目录; 并且数据库目录名称与相应的OID相同。 例如，当数据库sampledb的OID为16384时，其子目录名称为16384。

``` bash
 $ cd $ PGDATA
 $ ls -ld base / 16384
 drwx ------ 213 postgres postgres 7242 8 26 16:33 16384
```

### 1.2.3。 与表和索引关联的文件的布局

大小小于1GB的每个表或索引是存储在其所属的数据库目录下的单个文件。 作为数据库对象的表和索引由各个OID在内部管理，而这些数据文件由变量relfilenode管理 。 表和索引的relfilenode值基本上但不总是与相应的OID匹配，详细信息如下所述。

让我们展示表sampletbl的OID和relfilenode ：
```sql
 sampledb = #SELECT relname，oid，relfilenode FROM pg_class WHERE relname ='sampletbl';
   relname |  oid |  relfilenode
 ----------- + ------- + -------------
  sampletbl |  18740 |  18740 
 （1排）
```

从上面的结果中，您可以看到oid和relfilenode值都相等。 您还可以看到表sampletbl的数据文件路径是'base / 16384/18740' 。
```bash
 cd $ PGDATA
 $ ls -la base / 16384/18740
 -rw ------- 1 postgres postgres 8192 Apr 21 10:21 base / 16384/18740
```

通过发出一些命令（例如，TRUNCATE，REINDEX，CLUSTER）来更改表和索引的relfilenode值。 例如，如果我们截断表sampletbl ，PostgreSQL会为表分配一个新的relfilenode（18812），删除旧的数据文件（18740），并创建一个新的（18812）。
```sql
 sampledb =＃TRUNCATE sampletbl;
 TRUNCATE TABLE

 sampledb = #SELECT relname，oid，relfilenode FROM pg_class WHERE relname ='sampletbl';
   relname |  oid |  relfilenode
 ----------- + ------- + -------------
  sampletbl |  18740 |  18812 
 （1排）
```
在9.0或更高版本中，内置函数pg_relation_filepath非常有用，因为此函数返回具有指定OID或名称的关系的文件路径名。
```sql
 sampledb =＃SELECT pg_relation_filepath（'sampletbl'）;
  pg_relation_filepath 
 ----------------------
 碱/一万八千八百十二分之一万六千三百八十四
 （1排）
```


当表和索引的文件大小超过1GB时，PostgreSQL会创建一个名为relfilenode.1的新文件并使用它。 如果新文件已填满，则将创建名为relfilenode.2的下一个新文件，依此类推。

```bash
 $ cd $ PGDATA
 $ ls -la -h base / 16384/19427 *
 -rw ------- 1 postgres postgres 1.0G Apr 21 11:16 data / base / 16384/19427
 -rw ------- 1 postgres postgres 45M Apr 21 11:20 data / base / 16384 / 19427.1
```


在构建PostgreSQL时，可以使用配置选项--with-segsize更改表和索引的最大文件大小。

仔细查看数据库子目录，您会发现每个表都有两个相关文件，后缀分别为'_fsm'和'_vm'。 这些被称为自由空间映射和可见性映射 ，分别存储表文件中每个页面上的可用空间容量和可见性的信息（参见第5.3.4 节和第6.2 节中的更多细节）。 索引仅具有单独的可用空间映射，并且没有可见性映射。

具体示例如下所示：

```bash
 $ cd $ PGDATA
 $ ls -la base / 16384/18751 *
 -rw ------- 1 postgres postgres 8192 Apr 21 10:21 base / 16384/18751
 -rw ------- 1 postgres postgres 24576 Apr 21 10:18 base / 16384 / 18751_fsm
 -rw ------- 1 postgres postgres 8192 Apr 21 10:18 base / 16384 / 18751_vm
```
它们也可以在内部被称为每种关系的叉子 ; 可用空间映射是表/索引数据文件的第一个分支（fork编号为1），可见性映射表的数据文件的第二个分支（fork编号为2）。 数据文件的分叉号为0。
### 1.2.4。 表空间

PostgreSQL中的表空间是基本目录之外的附加数据区域。 此功能已在8.0版中实现。

图1.3显示了表空间的内部布局，以及与主数据区的关系。
**图1.3。 数据库群集中的表空间。**
![enter image description here](http://www.interdb.jp/pg/img/fig-1-03.png)

在发出CREATE TABLESPACE语句时指定的目录下创建表空间，并在该目录下创建特定于版本的子目录（例如，PG_9.4_201409291）。 版本特定的命名方法如下所示。

 PG _'主要版本'_'目录版本号'

例如，如果在'/ home / postgres / tblspc'中创建一个表空间'new_tblspc' ，其oid为16386，则会在表空间下创建一个子目录，例如'PG_9.4_201409291' 。
```sql
 $ ls -l / home / postgres / tblspc /
总共4
 drwx ------ 2 postgres postgres 4096 Apr 21 10:08 PG_9.4_201409291
```
表空间目录由pg_tblspc子目录中的符号链接寻址 ，链接名称与表空间的OID值相同。
```sql
 $ ls -l $ PGDATA / pg_tblspc /
总共0
 lrwxrwxrwx 1 postgres postgres 21 Apr 21 10:08 16386  - > / home / postgres / tblspc
```
如果在表空间下创建新数据库（OID为16387），则会在特定于版本的子目录下创建其目录。
```sql
 $ ls -l /home/postgres/tblspc/PG_9.4_201409291/
总共4
 drwx ------ 2 postgres postgres 4096 Apr 21 10:10 16387
```
如果创建属于在基本目录下创建的数据库的新表，首先，在特定于版本的子目录下创建名称与现有数据库OID相同的新目录，然后放置新表文件在创建的目录下。
```sql
 sampledb = #CREATE TABLE newtbl（.....）TABLESPACE new_tblspc;

 sampledb =＃SELECT pg_relation_filepath（'newtbl'）;
              pg_relation_filepath             
 ----------------------------------------------
  pg_tblspc关联/ 16386 / PG_9.4_201409291 /18894分之16384
```
## 1.3。 堆表文件的内部布局

在数据文件（堆表和索引，以及可用空间映射和可见性映射）内部，它被分为固定长度的页 （或块 ），默认为8192字节（8 KB）。 每个文件中的那些页面从0开始按顺序编号，这些数字称为块编号 。 如果文件已填满，PostgreSQL会在文件末尾添加一个新的空页以增加文件大小。

页面的内部布局取决于数据文件类型。 在本节中，将描述表格布局，以下章节将要求提供信息。
**图1.4。 堆表文件的页面布局。**
![enter image description here](http://www.interdb.jp/pg/img/fig-1-04.png)
图1.4。堆表文件的页面布局。
	
表中的页面包含如下所述的三种数据：

**1、heap tuple（s）** - 堆元组本身就是一个记录数据。 它们从页面底部按顺序堆叠。 元组的内部结构在第5.2节和第9章中描述，因为需要知道PostgreSQL中的并发控制（CC）和WAL。
**2、行指针** - 行指针长4个字节，并保存指向每个堆元组的指针。 它也被称为项目指针 。
    行指针形成一个简单的数组，它扮演元组索引的角色。 每个索引从1开始按顺序编号，并称为偏移号 。 当向页面添加新元组时，新的行指针也会被推到数组上以指向新的元组。
    **3、标头数据** - 由结构PageHeaderData定义的标头数据在页面的开头分配。 它长24个字节，包含有关页面的一般信息。 该结构的主要变量如下所述。
- pd_lsn - 此变量存储由此页面的最后一次更改写入的XLOG记录的LSN。 它是一个8字节无符号整数，与WAL（预写日志记录）机制相关。 细节在第9章中描述。
- pd_checksum - 此变量存储此页面的校验和值。 （请注意，版本9.3或更高版本支持此变量;在早期版本中，此部分已存储页面的timelineId。）
- pd_lower，pd_upper - pd_lower指向行指针的末尾，pd_upper指向最新堆元组的开头。
- pd_special - 此变量用于索引。 在表格中的页面中，它指向页面的末尾。 （在索引中的页面中，它指向特殊空间的开头，它是仅由索引保存的数据区域，并根据索引类型的类型包含特定数据，如B-tree，GiST，GiN等） 

行指针末尾和最新元组开头之间的空白空间称为空闲空间或空洞 。

为了识别表中的**元组**，内部使用**元组标识符（TID）** 。 TID包括一对值：包含元组的页面的块编号 ，以及指向元组的行指针的偏移编号 。 其用法的典型示例是索引。 请参见第1.4.2节中的更多细节。

```sql
PageHeaderData在src / include / storage / bufpage.h中定义 。
```
另外，使用称为TOAST （超大属性存储技术）的方法来存储和管理其大小大于约2KB（约为8KB的1/4）的堆元组。 有关详细信息，请参阅PostgreSQL文档 。
## 1.4。 写作和阅读元组的方法

在本章的最后，描述了编写和读取堆元组的方法。
### 1.4.1。 写堆堆元组

假设一个表由一个页面组成，该页面只包含一个堆元组。 此页面的pd_lower指向第一行指针，行指针和pd_upper都指向第一个堆元组。 见图1.5（a）。

插入第二个元组时，将其放在第一个元组之后。 第二行指针被推到第一行，它指向第二个元组。 pd_lower更改为指向第二行指针，pd_upper更改为第二个堆元组。 见图1.5（b）。 此页面中的其他标题数据（例如，pd_lsn，pg_checksum，pg_flag）也被重写为适当的值; 更多细节在第5.3节和第9章中描述。
**图1.5。 编写堆元组。**
![enter image description here](http://www.interdb.jp/pg/img/fig-1-05.png)
图1.5。编写堆元组。
###1.4.2。 阅读堆元组

这里概述了两种典型的访问方法，顺序扫描和B树索引扫描：

- **顺序扫描** - 通过扫描每页中的所有行指针顺序读取所有页面中的所有元组。 见图1.6（a）。
- **B树索引扫描** - 索引文件包含索引元组，每个索引元组由索引键和指向目标堆元组的TID组成。 如果找到了您正在查找的键的索引元组，PostgreSQL将使用获取的TID值读取所需的堆元组。 （这里没有解释在B树索引中找到索引元组的方法的描述，因为它很常见，这里的空间有限。参见相关资料。）例如，在图1.6（b）中，TID获得的索​​引元组的值是'（block = 7，Offset = 2）'。 这意味着目标堆元组是表中第7页的第二元组，因此PostgreSQL可以读取所需的堆元组，而不会在页面中进行不必要的扫描。 

**图1.6。 顺序扫描和索引扫描。**
![enter image description here](http://www.interdb.jp/pg/img/fig-1-06.png)


PostgreSQL还支持TID-Scan， Bitmap-Scan和Index-Only-Scan。

TID-Scan是一种通过使用所需元组的TID直接访问元组的方法。 例如，要在表中找到第0个页面中的第一个元组，请发出以下查询：
```sql
 sampledb = #SELECT ctid，data FROM sampletbl WHERE ctid ='（0,1）';
  ctid | 数据    
 ------- + -----------
  （0,1）|  AAAAAAAAA
 （1排）
```
Index-Only-Scan将在第7章中详细介绍。

***©版权所有 2015-2018 Hironobu SUZUKI版权所有。***
