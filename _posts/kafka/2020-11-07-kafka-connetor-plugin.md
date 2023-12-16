---
layout: post
title:  "kafka-connector-plugin"
date:   2021-11-14 23:40 +0800
categories: kafka-connect
tags: Debezium-Oracle kafka-connect
author: qinghui.guo
mathjax: true
---
* content
{:toc}

## 摘要
>上文已经介绍了如何以插件的方式部署Debezium，可以看到通过kafka connect接口，可以部署和使用丰富的连接器。confluent提供了丰富的connect plugin，包含自研的和一些第三方的plugin，而Debezium开源项目提供针对DB 的connector。所以单纯从功能上来说，通过kafka和connector plugin 实现数据总线，或者数据流成为可能，让数据流动起来。
>上文已经部署过了stanalone 模式，本文会着重介绍分布式模式部署kafka connect，并说明Debezium社区开发的所有连接器，以及支持的数据种类。
>目前只说明kafka connector source ，为下文展开sink 端的数据导出，本文还会部署confluent 公司开发 jdbc connector，能连接所有提供jdbc连接的数据库，实现数据的导入和导出。
>

## kafka connector 分布式架构说明

>Kafka Connect是Kafka 0.9+增加了一个新的特性,提供了API可以更方便的创建和管理数据流管道。它为Kafka和其它系统创建规模可扩展的、可信赖的流数据提供了一个简单的模型，通过connectors可以将大数据从其它系统导入到Kafka中，也可以从Kafka中导出到其它系统。Kafka Connect可以将完整的数据库注入到Kafka的Topic中，或者将服务器的系统监控指标注入到Kafka，然后像正常的Kafka流处理机制一样进行数据流处理。而导出工作则是将数据从Kafka Topic中导出到其它数据存储系统、查询系统或者离线分析系统等，比如数据库、Elastic Search、Apache Ignite等。


## plugin组件介绍
- confluentinc-kafka-connect-jdbc-10.2.5.zip
jdbc 支持source 和sink
debezium-connector-oracle-1.7.0.Final-plugin.tar.gz
只支持source，CDC日志采集
debezium-connector-mysql-1.7.0.Final-plugin.tar.gz
只支持source，CDC日志采集
debezium-connector-postgres-1.7.0.Final-plugin.tar.gz
只支持source，CDC日志采集
debezium-connector-mongodb-1.7.1.Final-plugin.tar.gz
只支持source，CDC日志采集
debezium-connector-sqlserver-1.7.1.Final-plugin.tar.gz
只支持source，CDC日志采集
debezium-connector-db2-1.7.1.Final-plugin.tar.gz
只支持source，CDC日志采集
debezium-connector-vitess-1.7.1.Final-plugin.tar.gz
只支持source，CDC日志采集
debezium-connector-cassandra-1.7.1.Final-plugin.tar.gz
只支持source，CDC日志采集

## kafka部署和 plugin组件配置

> 1、zookeeper 集群部署


```
--所有节点修改配置文件
vi /app/kafka_2.13-2.8.1/config/zookeeper.properties
dataDir=/app/data/zookeeper
clientPort=2181
maxClientCnxns=0
admin.enableServer=false
tickTime=2000
initLimit=5
syncLimit=2
server.1=10.96.80.31:2888:3888
server.2=10.96.80.32:2888:3888
server.3=10.96.80.33:2888:3888

--设置不同节点的myid
echo "1" > /app/data/zookeeper/myid
echo "2" > /app/data/zookeeper/myid
echo "3" > /app/data/zookeeper/myid

--所有节点启动服务
zookeeper-server-start.sh -daemon /app/kafka_2.13-2.8.1/config/zookeeper.properties
```

> 2、kafka集群配置

```
--修改配置文件，所有节点

vi /app/kafka_2.13-2.8.1/config/server.properties
broker.id=1
listeners=PLAINTEXT://10.96.80.31:9092
num.network.threads=3
num.io.threads=8
socket.send.buffer.bytes=102400
socket.receive.buffer.bytes=102400
socket.request.max.bytes=104857600
log.dirs=/app/data/kafka-logs
num.partitions=1
num.recovery.threads.per.data.dir=1
min.insync.replicas=2
default.replication.factor=3
offsets.topic.replication.factor=3
transaction.state.log.replication.factor=3
transaction.state.log.min.isr=3

log.retention.hours=2

log.segment.bytes=1073741824
log.retention.check.interval.ms=300000
zookeeper.connect=10.96.80.31:2181,10.96.80.32:2181,10.96.80.33:2181

zookeeper.connection.timeout.ms=18000

group.initial.rebalance.delay.ms=0
delete.topic.enable=true

节点2
broker.id=2
listeners=PLAINTEXT://10.96.80.32:9092
节点三
broker.id=3
listeners=PLAINTEXT://10.96.80.33:9092

--启动kafka服务
kafka-server-start.sh -daemon /app/kafka_2.13-2.8.1/config/server.properties
```
> 3、Kafka connect配置和启动

```
--节点一
vi /app/kafka_2.13-2.8.1/config/connect-distributed.properties
bootstrap.servers=10.96.80.31:9092,10.96.80.32:9092,10.96.80.33:9092
group.id=connect-cluster
key.converter=org.apache.kafka.connect.json.JsonConverter
value.converter=org.apache.kafka.connect.json.JsonConverter
key.converter.schemas.enable=false
value.converter.schemas.enable=false

internal.key.converter=org.apache.kafka.connect.json.JsonConverter
internal.value.converter=org.apache.kafka.connect.json.JsonConverter
internal.key.converter.schemas.enable=false
internal.value.converter.schemas.enable=false


offset.storage.topic=connect-offsets
offset.storage.replication.factor=3
offset.storage.partitions=3

config.storage.topic=connect-configs
config.storage.replication.factor=3

status.storage.topic=connect-status
status.storage.replication.factor=3

offset.flush.interval.ms=10000
rest.advertised.host.name=10.96.80.31

offset.storage.file.filename=/app/data/connect.offsets
plugin.path=/app/kafka_2.13-2.8.1/plugin
--节点二
rest.advertised.host.name=10.96.80.32
节点三
rest.advertised.host.name=10.96.80.33
--创建启动必须的topic
kafka-topics.sh --create --zookeeper 10.96.80.31:2181 --topic connect-configs --replication-factor 3 --partitions 1 --config cleanup.policy=compact
kafka-topics.sh --create --zookeeper 10.96.80.31:2181 --topic connect-offsets --replication-factor 3 --partitions 3 --config cleanup.policy=compact
kafka-topics.sh --create --zookeeper 10.96.80.31:2181 --topic connect-status --replication-factor 3 --partitions 1 --config cleanup.policy=compact

--启动分布式的connect服务，所有节点

connect-distributed.sh -daemon /app/kafka_2.13-2.8.1/config/connect-distributed.properties

```

> 4、查询支持 plugins

```
[kafka@node2 ~]$ curl -s localhost:8083/connector-plugins|jq
[
  {
    "class": "io.confluent.connect.jdbc.JdbcSinkConnector",
    "type": "sink",
    "version": "10.2.5"
  },
  {
    "class": "io.confluent.connect.jdbc.JdbcSourceConnector",
    "type": "source",
    "version": "10.2.5"
  },
  {
    "class": "io.debezium.connector.db2.Db2Connector",
    "type": "source",
    "version": "1.7.1.Final"
  },
  {
    "class": "io.debezium.connector.mongodb.MongoDbConnector",
    "type": "source",
    "version": "1.7.1.Final"
  },
  {
    "class": "io.debezium.connector.mysql.MySqlConnector",
    "type": "source",
    "version": "1.7.0.Final"
  },
  {
    "class": "io.debezium.connector.oracle.OracleConnector",
    "type": "source",
    "version": "1.7.0.Final"
  },
  {
    "class": "io.debezium.connector.postgresql.PostgresConnector",
    "type": "source",
    "version": "1.7.0.Final"
  },
  {
    "class": "io.debezium.connector.sqlserver.SqlServerConnector",
    "type": "source",
    "version": "1.7.1.Final"
  },
  {
    "class": "io.debezium.connector.vitess.VitessConnector",
    "type": "source",
    "version": "1.7.1.Final"
  },
  {
    "class": "org.apache.kafka.connect.file.FileStreamSinkConnector",
    "type": "sink",
    "version": "2.8.1"
  },
  {
    "class": "org.apache.kafka.connect.file.FileStreamSourceConnector",
    "type": "source",
    "version": "2.8.1"
  },
  {
    "class": "org.apache.kafka.connect.mirror.MirrorCheckpointConnector",
    "type": "source",
    "version": "1"
  },
  {
    "class": "org.apache.kafka.connect.mirror.MirrorHeartbeatConnector",
    "type": "source",
    "version": "1"
  },
  {
    "class": "org.apache.kafka.connect.mirror.MirrorSourceConnector",
    "type": "source",
    "version": "1"
  }
]
```

## 配置Oracle connector

**需要特别注意，connectors 的建立通过reset api实现**

> 1、创建连接器

```
curl -X POST http://localhost:8083/connectors -H "Content-Type: application/json" -d '{
"name": "hr-connector",
"config": {
"connector.class" : "io.debezium.connector.oracle.OracleConnector",
"tasks.max" : "1",
"database.server.name" : "DB01",
"database.hostname" : "10.96.80.21",
"database.port" : "1521",
"database.user" : "c##dbzuser",
"database.password" : "dbz",
"database.dbname" : "orcl",
"database.pdb.name":"PDB1",
"table.include.list" : "hr.test", 
"database.history.kafka.bootstrap.servers" : "10.96.80.31:9092,10.96.80.32:9092,10.96.80.33:9092",
"database.history.kafka.topic": "schema-changes.test"
}
}'
```
> 2、检查连接器的状态

```
[kafka@node1 debezium-connector-cassandra]$ curl -s localhost:8083/connectors/hr-connector/status|jq
{
  "name": "hr-connector",
  "connector": {
    "state": "RUNNING",
    "worker_id": "10.96.80.31:8083"
  },
  "tasks": [
    {
      "id": 0,
      "state": "RUNNING",
      "worker_id": "10.96.80.31:8083"
    }
  ],
  "type": "source"
}
```

## 配置MySQLconnector

> 1、创建MySQL 的connector

```
curl -X POST http://localhost:8083/connectors -H "Content-Type: application/json" -d '{
"name" : "mysql-connector",
"config":
{
     "connector.class" : "io.debezium.connector.mysql.MySqlConnector",
     "database.hostname" : "10.96.80.31",
     "database.port" : "3307",
     "database.user" : "dbz_mysql",
     "database.password" : "dbz",
     "database.server.id" : "1234",
     "database.server.name" : "MYSQLDB",
     "database.include.list": "TEST",
     "database.history.kafka.bootstrap.servers" : "10.96.80.31:9092,10.96.80.32:9092,10.96.80.33:9092",
     "database.history.kafka.topic" : "mysql.history",
     "include.schema.changes" : "true" ,
     "database.history.skip.unparseable.ddl" : "true"
}
}'
```
> 查看connector 状态

```
[kafka@node1 debezium-connector-cassandra]$ curl -s localhost:8083/connectors/mysql-connector/status|jq
{
  "name": "mysql-connector",
  "connector": {
    "state": "RUNNING",
    "worker_id": "10.96.80.33:8083"
  },
  "tasks": [
    {
      "id": 0,
      "state": "RUNNING",
      "worker_id": "10.96.80.33:8083"
    }
  ],
  "type": "source"
}

```

## reset api 的常用命令
> 1、查看当前集群的连接器

```
[kafka@node1 ~]$ curl -s localhost:8083/connectors|jq
[
  "mysql-connector",
  "hr-connector"
]
```
> 2、查看连接器详情

```
[kafka@node1 ~]$ curl -s localhost:8083/connectors/mysql-connector |jq
{
  "name": "mysql-connector",
  "config": {
    "connector.class": "io.debezium.connector.mysql.MySqlConnector",
    "database.user": "dbz_mysql",
    "database.server.id": "1234",
    "database.history.kafka.bootstrap.servers": "10.96.80.31:9092,10.96.80.32:9092,10.96.80.33:9092",
    "database.history.kafka.topic": "mysql.history",
    "database.server.name": "MYSQLDB",
    "database.port": "3307",
    "include.schema.changes": "true",
    "database.hostname": "10.96.80.31",
    "database.password": "dbz",
    "name": "mysql-connector",
    "database.history.skip.unparseable.ddl": "true",
    "database.include.list": "TEST"
  },
  "tasks": [
    {
      "connector": "mysql-connector",
      "task": 0
    }
  ],
  "type": "source"
}
```



> 3、查看连接器的状态

```
[kafka@node1 ~]$ curl -s localhost:8083/connectors/mysql-connector/status|jq
{
  "name": "mysql-connector",
  "connector": {
    "state": "RUNNING",
    "worker_id": "10.96.80.33:8083"
  },
  "tasks": [
    {
      "id": 0,
      "state": "RUNNING",
      "worker_id": "10.96.80.33:8083"
    }
  ],
  "type": "source"
}
```

> 4、暂停连接器

```
curl -s -X PUT localhost:8083/connectors/mysql-connector/pause
```
> 5、恢复连接器

```
curl -s -X PUT localhost:8083/connectors/mysql-connector/resume
```

> 6、重启连接器

```
curl -s -X PUT localhost:8083/connectors/mysql-connector/restart
```
> 7、删除连接器

```
curl -X DELETE   http://localhost:8083/connectors/mysql-connector
```

> 8、修改连接器参数

```
curl -i -X PUT -H "Accept:application/json" -H "Content-Type:application/json" localhost:8083/connectors/hr-connector/config -d '
{
"connector.class" : "io.debezium.connector.oracle.OracleConnector",
"tasks.max" : "1",
"database.server.name" : "ORCL",
"database.hostname" : "10.96.80.21",
"database.port" : "1521",
"database.user" : "c##dbzuser",
"database.password" : "dbz",
"database.dbname" : "orcl",
"database.pdb.name":"PDB1",
"schema.include.list" : "hr",
"database.history.kafka.bootstrap.servers" : "10.96.80.31:9092",
"database.history.kafka.topic": "were.wrewrw"
}'
```

## kafak的常用命令
> 1、查看所有的topic

```
[kafka@node1 ~]$ kafka-topics.sh --list --zookeeper localhost:2181
DB01
DB01.HR.TEST
MYSQLDB
MYSQLDB.test.test
MYSQLDB.test.test2
MYSQLDB.test.test3
MYSQLDB.test.test4
__consumer_offsets
connect-configs
connect-offsets
connect-status
mysql.history
schema-changes.test
```
	
> 2、查看topic详情

```
kafka-console-consumer.sh --bootstrap-server 10.96.80.31:9092 --topic MYSQLDB.test.test --from-beginning|jq

```

> 3、查看topic副本信息

```
[kafka@node1 ~]$ kafka-topics.sh --zookeeper 10.96.80.31:2181  --topic MYSQLDB.test.test --describe 
Topic: MYSQLDB.test.test        TopicId: kKZBdzP7SDmRghy9iIlCAg PartitionCount: 1       ReplicationFactor: 3 Configs: 
        Topic: MYSQLDB.test.test        Partition: 0    Leader: 2       Replicas: 2,1,3 Isr: 2,1,3
[kafka@node1 ~]$ 

```

## 分布式架构的kafka配置完成
> 关于connector 具体参数配置和使用，参考官方文档。
> https://debezium.io/

