---
layout: post
title:  "TiDB-分布式数据库数据强一致性的实现"
date:   2020-01-30 09:40:18 +0800
categories: TiDB
tags: TiDB TiKV PD RAFT协议 gRPC RocksDB
author: qinghui.guo
mathjax: true
---

* content
{:toc}
本文介绍TiDB的一些基本概念、基本实现原理以及运用的一些开源技术，当然需要说明的是，TiDB不是平常理解的数据库，它是一套架构，通过几个组件的互相调用和一些分布式原理实现。

## 基本概念和相关分布式原理介绍
### gRPC：
（http://www.oschina.net/p/grpc-framework）
gRPC()  是一个高性能、开源和通用的 RPC 框架，面向移动和 HTTP/2 设计。

目前提供 C、Java 和 Go 语言版本，分别是：grpc, grpc-java, grpc-go. 其中 C 版本支持 C, C++, Node.js, Python, Ruby, Objective-C, PHP 和 C# 支持.

gRPC 基于 HTTP/2 标准设计，带来诸如双向流、流控、头部压缩、单 TCP 连接上的多复用请求等特。这些特性使得其在移动设备上表现更好，更省电和节省空间占用。


### RocksDB：
（https://blog.csdn.net/zhufenglonglove/article/details/54286068）
RocksDB项目起源于Facebook的一个实验项目,该项目旨在开发一个与快速存储器（尤其是闪存）存储数据性能相当的数据库软件，以应对高负载服务。

这是一个c++库,可用于存储键和值,可以是任意大小的字节流。它支持原子读和写。

RocksDB具有高度灵活的配置功能,可以通过配置使其运行在各种各样的生产环境,包括纯内存,Flash,硬盘或HDFS。它支持各种压缩算法，并提供了便捷的生产环境维护和调试工具。

RocksDB借鉴了开源项目LevelDB的重要代码和Apache HBase项目的重要思想。最初的代码来源于开源项目leveldb 1.5分叉。它借鉴了了Facebook的代码和思想。

### RAFT协议

Raft是一种共识算法，旨在替代Paxos。 它通过逻辑分离比Paxos更容易理解，但它也被正式证明是安全的，并提供了一些额外的功能。 Raft提供了一种在计算系统集群中分布状态机的通用方法，确保集群中的每个节点都同意一系列相同的状态转换。

## TiDB整体架构 ***官方文档***

要深入了解 TiDB 的水平扩展和高可用特点，首先需要了解 TiDB 的整体架构。TiDB 集群主要包括三个核心组件：TiDB Server，PD Server 和 TiKV Server。此外，还有用于解决用户复杂 OLAP 需求的 TiSpark 组件。

![](https://pingcap.com/images/docs-cn/tidb-architecture.png)

### TiDB Server  

TiDB Server 负责接收 SQL 请求，处理 SQL 相关的逻辑，并通过 PD 找到存储计算所需数据的 TiKV 地址，与 TiKV 交互获取数据，最终返回结果。TiDB Server 是无状态的，其本身并不存储数据，只负责计算，可以无限水平扩展，可以通过负载均衡组件（如LVS、HAProxy 或 F5）对外提供统一的接入地址。
### PD Server

Placement Driver (简称 PD) 是整个集群的管理模块，其主要工作有三个：一是存储集群的元信息（某个 Key 存储在哪个 TiKV 节点）；二是对 TiKV 集群进行调度和负载均衡（如数据的迁移、Raft group leader 的迁移等）；三是分配全局唯一且递增的事务 ID。

PD 是一个集群，需要部署奇数个节点，一般线上推荐至少部署 3 个节点。

### TiKV Server

TiKV Server 负责存储数据，从外部看 TiKV 是一个分布式的提供事务的 Key-Value 存储引擎。存储数据的基本单位是 Region，每个 Region 负责存储一个 Key Range（从 StartKey 到 EndKey 的左闭右开区间）的数据，每个 TiKV 节点会负责多个 Region。TiKV 使用 Raft 协议做复制，保持数据的一致性和容灾。副本以 Region 为单位进行管理，不同节点上的多个 Region 构成一个 Raft Group，互为副本。数据在多个 TiKV 之间的负载均衡由 PD 调度，这里也是以 Region 为单位进行调度。

### TiSpark

TiSpark 作为 TiDB 中解决用户复杂 OLAP 需求的主要组件，将 Spark SQL 直接运行在 TiDB 存储层上，同时融合 TiKV 分布式集群的优势，并融入大数据社区生态。至此，TiDB 可以通过一套系统，同时支持 OLTP 与 OLAP，免除用户数据同步的烦恼。


## 事务原理 

### 事务模型

TiKV 的事务采用的是 Percolator 模型，并且做了大量的优化。事务的细节这里不详述，大家可以参考论文以及我们的其他文章。这里只提一点，TiKV 的事务采用乐观锁，事务的执行过程中，不会检测写写冲突，只有在提交过程中，才会做冲突检测，冲突的双方中比较早完成提交的会写入成功，另一方会尝试重新执行整个事务。当业务的写入冲突不严重的情况下，这种模型性能会很好，比如随机更新表中某一行的数据，并且表很大。但是如果业务的写入冲突严重，性能就会很差，举一个极端的例子，就是计数器，多个客户端同时修改少量行，导致冲突严重的，造成大量的无效重试。

### 事务过程

该端引用网址为：http://www.itpub.net/thread-2059684-1-1.html

总体来说，TiKV 的读写事务分为两个阶段：1、Prewrite 阶段；2、Commit 阶段。

客户端会缓存本地的写操作，在客户端调用 client.Commit() 时，开始进入分布式事务 prewrite 和 commit 流程。

Prewrite 对应传统 2PC 的第一阶段：

1、首先在所有行的写操作中选出一个作为 primary row，其他的为 secondary rows
2、PrewritePrimary: 对 primaryRow 写入锁（修改 meta key 加入一个标记），锁中记录本次事务的开始时间戳。上锁前会检查：
    
i.该行是否已经有别的客户端已经上锁 (Locking)

    
ii.是否在本次事务开始时间之后，检查versions ，是否有更新 [startTs, +Inf) 的写操作已经提交 (Conflict)

在这两种种情况下会返回事务冲突。否则，就成功上锁。将行的内容写入 row 中，版本设置为 startTs
3、将 primaryRow 的锁上好了以后，进行 secondaries 的 prewrite 流程：
    
i.类似 primaryRow 的上锁流程，只不过锁的内容为事务开始时间 startTs 及 primaryRow 的信息
    
ii.检查的事项同 primaryRow 的一致
    
iii.当锁成功写入后，写入 row，时间戳设置为 startTs


以上 Prewrite 流程任何一步发生错误，都会进行回滚：删除 meta 中的 Lock 标记 , 删除版本为 startTs 的数据。

当 Prewrite 阶段完成以后，进入 Commit 阶段，当前时间戳为 commitTs，TSO会保证 commitTs> startTs

Commit 的流程是，对应 2PC 的第二阶段：

1、commit primary: 写入 meta 添加一个新版本，时间戳为 commitTs，内容为 startTs, 表明数据的最新版本是 startTs 对应的数据

2、删除 Lock 标记


值得注意的是，如果 primary row 提交失败的话，全事务回滚，回滚逻辑同 prewrite 失败的回滚逻辑。

如果 commit primary 成功，则可以异步的 commit secondaries，流程和 commit primary 一致， 失败了也无所谓。Primary row 提交的成功与否标志着整个事务是否提交成功。

事务中的读操作：
1、检查该行是否有 Lock 标记，如果有，表示目前有其他事务正占用此行，如果这个锁已经超时则尝试清除，否则等待超时或者其他事务主动解锁。注意此时不能直接返回老版本的数据，否则会发生幻读的问题。

2、读取至 startTs 时该行最新的数据，方法是：读取 meta ，找出时间戳为 [0, startTs], 获取最大的时间戳 t，然后读取为于 t 版本的数据内容。


由于锁是分两级的，Primary 和 Seconary row，只要 Primary row 的锁去掉，就表示该事务已经成功提交，这样的好处是 Secondary 的 commit 是可以异步进行的，只是在异步提交进行的过程中，如果此时有读请求，可能会需要做一下锁的清理工作。因为即使 Secondary row 提交失败，也可以通过 Secondary row 中的锁，找到 Primary row，根据检查 Primary row 的 meta，确定这个事务到底是被客户端回滚还是已经成功提交。


## TiDB事务场景漫谈


1、小事务，事务不跨region，如何保证事务强一致性。

> 该情况较为简单，和单实例一样，事务保证性也好，无需复杂的确认过程。多个副本之间的数据一致性，通过raft协议和事务日志应用保证leader和follower region之间的数据一致性。
> 一旦leader 和 follower 之间事务日志传输量较大，传输到副本超时（180ms)，或者多个副本超时，会引发节点踢出，虽然可以通过大多数确认提交，就认为成功，但是还是存在一定问题。

2、小事务，事务跨region，涉及分布式事务，如何保证数据强一致性。
> 通过二阶段提交实现，primary row 和seconary row 异步提交，只要primary row 提交，全事务提交成功。 如果异步提交的seconary row失败，会重新尝试，笔者个人认为这点是primary row成功之后，会记录seconary row的信息，但是官方解释和一些分享比没有说明primary row提交记录的版本信息具体会记录seconary row的哪些信息，数据强一致性有待确认。

3、小事务，多会话同时更新同一条记录，如何实现数据强一致性。
>该情况涉及分布式事务，不同rockdb实例之间分布式事务实现，由于通过乐观锁解决锁冲突，如果失败会话会重试该事务。如果业务层逻辑没有控制好，多个会话更新相同记录，造成锁冲突，引起多会话重试事务，严重影响性能。

4、大事务执行时间长，提交前，其他会话已经更新大事务操作的数据，如何实现数据强一致性。
> 由于乐观锁定，后提交的事务会回滚，大事务回滚后然后重新尝试，中间在有小事务更新其中的数据且优先提交，大事务还是会失败。

## TiDB事务强一致性实现可能存在的问题

1. 业务压力超大的情况下，raft group的各个region日志应用延时，是否会导致整个集群不可用。
2. TiDB事务的异步提交模式是否会造成seconary row 的事务丢失，如果异步提交失败，seconary row的信息记录在哪里，异步提交的seconary row如何恢复。
3.  TiDB对大事务的支持一般，大失误小事务交叉操作，乐观锁造成的大事务频繁回滚，严重影响数据库性能。
4. 乐观锁冲突后，为保证数据强一致性，后提交的事务会回滚，然后重新尝试。一旦业务逻辑存在问题（多个会话同时操作同一行记录），失败的事务不断重新尝试，造成严重的性能问题。



## 引用参考文档

TiDB的基本实现原理是根据谷歌的三篇论文实现和一篇raft 协议论文，以下是四篇论文的地址。

1、F1: A Distributed SQL Database That Scales ***--对应TiDB，SQL层，计算层***

https://storage.googleapis.com/pub-tools-public-publication-data/pdf/41344.pdf

2、Spanner: Google’s Globally Distributed Databas***--对应TiKV，存储层，事务实现***

https://storage.googleapis.com/pub-tools-public-publication-data/pdf/65b514eda12d025585183a641b5a9e096a3c4be5.pdf

3、Large-scale Incremental Processing Using Distributed Transactions and Notifications  ***--分布式事务实现原理***

https://storage.googleapis.com/pub-tools-public-publication-data/pdf/36726.pdf

4、raft 协议算法


https://web.stanford.edu/~ouster/cgi-bin/papers/raft-atc14

5、raft 协议实现动画

http://thesecretlivesofdata.com/raft/

6、TiDB事务实现源原理

http://www.itpub.net/thread-2059684-1-1.html

7、、grpc 

http://www.oschina.net/p/grpc-framework

8、官方文档

https://pingcap.com/docs-cn/
