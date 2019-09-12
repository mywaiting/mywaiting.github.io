---
layout: post
title: "Introducing MySQL Change Data Capture"
description: "MySQL Change Data Capture, MySQL 数据库变动数据捕获"
date: "2019-09-12 21:34:13"
categories:
    - Weblogs
tags:
    - MySQL
    - CDC
---

* toc
{:toc}

## 前言

在写 `Change Data Capture` 之前，我想写一下我遇到它的故事。

在进行项目调研的时候，遇到这样一个难题：如何安全地准实时进行 MySQL 全球数据同步。当然啦，我这个项目数据量比较小，跟那些动辄冠以海量数据的项目没有可比性，而且项目对于实时性要求不高，分钟甚至十分钟级别都是可以的。问题其实很简单，既然用了 MySQL，相信稍微有点 MySQL 常识的同学都会想到 MySQL 主从同步（主从复制或者多主复制都是可以的），事实上，我刚刚开始的时候也是这么想的。然而我既然将这么一件事写成了一篇博客，足以说明这件事情没有这么简单。里面关键的问题并非是 MySQL 主从复制的搭建困难等诸如此类的技术问题，而是一个很现实的问题：我的主数据库位于中国大陆。

想必能读懂上面这句话的同学都会第一时间反应过来是什么问题，是的，网络的波动导致上面的 MySQL 主从同步，在跨地区同步的时候完成不可用，或者说，用起来后会有很多不确定的因素。我可不想写那种时不时会出错告警的代码。很简单的人生哲学，不做则已，如果做的话就一次将事情做好，这能少去很多的人生的烦恼。

其实如果 MySQL 数据库的数据量不大，更新不频繁的话，其实跨地区准时更新 MySQL 数据库有个很简单的方法

> 每次从 MySQL 中 dump 出全部的数据打包成 `backup.timestamp.sql` 然后使用 `rsync` 到异地服务器全库 `insert` 即可
    
这样的方法是不是很简单？很粗暴？

是的，确实粗暴有效果。思路灵活的一点的，甚至可以使用 Dropbox 这样的云存储来实现服务器之间的数据同步。然而这样的实现，虽然够直接够粗暴，却不那么优雅，原因很简单，每次同步都是 MySQL 全库重建插入数据，少量数据还好，稍微大量点的数据玩不起这样的实现啊，毕竟线上产品也是时刻有访问的啊。

但是上面这样的思路没有错啊，只是全量更新略为不够优美，那么有没有办法做到 MySQL 的数据增量更新呢？是的，有这样的解决方法，答案就是 `Change Data Capture` 

## 关于 CDC

CDC 全称 `Change Data Capture` 这样的英文很好理解，翻译成中文反而不是很易明白，简单来说就是捕获变动（有变化）的数据。你可以将这个术语理解为跟数据库架构有关的一种设计模式。它的核心思想是，监测并捕获数据库的变动（包括数据或数据表的插入，更新，删除等），将这些变更按发生的顺序完整记录下来，写入到消息中间件中以供其他服务进行订阅及消费。

建议参考 [维基百科 - Change data capture](https://en.wikipedia.org/wiki/Change_data_capture) 还真是写得挺详细的。

## CDC 架构实现

实现一个完整的 `Change Data Capture` 架构需要 数据源、发布服务（变动数据生产者）、数据总线（消息中间件）、数据消费 共四个部分。

### 数据源

数据库（即数据源）需要能完整地记录或输出数据库的事件（数据或数据表的变动）。例如 MySQL binlog，MySQL 通过 binlog（二进制日志）记录数据库的变动事件，而 MySQL 本身可以借助 binlog 来实现主从复制。其实 CDC 本身也只是把这种机制扩展了一下，使之能够作为更广泛的用途。

在进行 MySQL 主从复制时候，MySQL 会将写入主库的数据（或者说执行过程）通过生成 MySQL binlog 同步到其他的 MySQL 从库再执行一次，以达到主从一致的目的。其设计的思路是相当的原始的。可以简单理解 MySQL binlog 是 MySQL 自身状态一次 `diff` 抓住了这样的 `diff` 过程就能理解 MySQL 更新了什么样的数据。其实也不只是 MySQL，其他的数据库也有这样的实现的

- MySQL: [binlog](https://dev.mysql.com/doc/en/mysqlbinlog.html)
- PostgeSQL: [Streaming Replication Protocol](https://www.postgresql.org/docs/current/static/logicaldecoding-walsender.html)
- MongoDB: [oplog](https://docs.mongodb.com/manual/core/replica-set-oplog)
- Microsoft SQLServer: [SQLServer Replication](https://docs.microsoft.com/en-us/sql/relational-databases/replication/types-of-replication)
- Oracle: Oracle Replication

**注**：其实 Microsoft SQLServer 和 Oracle 有原生的 CDC 实现

### 发布服务（变动数据生产者）

数据源生产了数据，那么通过各种方式得到数据源的更新数据后（比如通过模拟读取并解析 MySQL binlog），需要将这些数据发布出来，供大家进行订阅和调用。作为变动数据的生产者，其主要功能是解析数据源输出的流式数据，序列化成统一的封装格式，并输出到数据总线中。

这里列出部分可以解析 MySQL binlog 然后输出到 kafka 的实现

Project                      | Site                                                   | Description
-----------------------------|--------------------------------------------------------|-------------
aesop                        | https://github.com/Flipkart/aesop                      | Built on top of Databus. In production use at http://www.flipkart.com/. Allows you to plug in your own code to transform/process the MySQL events.
databus                      | https://github.com/linkedin/databus                    | Precursor to Kafka. Reads from MySQL and Oracle, and replicates to its own log structure. In production use at LinkedIn. No Kafka integration. Uses Open Replicator.
Lapidus                      | https://github.com/JarvusInnovations/lapidus           | Streams data from MySQL, PostgreSQL and MongoDB as newline delimited JSON. Can be run as a daemon or included as a Node.js module.
Maxwell                      | https://github.com/zendesk/maxwell                     | Reads MySQL event stream, output events as JSON.  Parses ALTER/CREATE TABLE/etc statements to keep schema in sync.  Written in java. Well maintained.
Canal                        | https://github.com/alibaba/canal                       | Alibaba Open Source Solution for MySQL binlog event parse and consumer

如果你打算自己编程实现，这里有一些前人造好的轮子供你参考。

Project                      | Site                                                   | Description
-----------------------------|--------------------------------------------------------|-------------
python-mysql-replication     | https://github.com/noplay/python-mysql-replication     | Pure python library that parses MySQL binary logs and lets you process the replication events. Basically, the python equivalent of mysql-binlog-connector-java
Debezium                     | http://debezium.io                                     | Replicates from MySQL to Kafka. Uses mysql-binlog-connector-java. Kafka Connector. A funded project supported by Redhat with employees working on it full time.
php-mysql-replication        | https://github.com/krowinski/php-mysql-replication     | Pure PHP Implementation of MySQL replication protocol. This allow you to receive event like insert, update, delete with their data and raw SQL queries.


### 数据总线（消息中间件）

数据总线这部分是一个消息中间件，一般会选 Kafka，RabbitMQ，Redis 之类的，但大部分做 CDC 都会选用 Kafka。

Kafka 官方并不把自己称为消息队列，而是叫 `distributed streaming platform` 的确，目前 kafka 的能力以及演化方向更像是一个数据处理的平台了，而不仅仅是消息中间件，你可以利用 Kafka Streams 来进行一些数据处理。

在整个 CDC 里，Kafka 会作为核心的数据交换组件，或者你可以把它称为数据总线，kafka 集群的健壮性和吞吐量能够支撑海量数据的 pub/sub，并且能够将写入的数据持久化一段时间，发布服务将数据库任何数据变动写入 Kafka，由不同的消费者在上面同时进行订阅和消费，如果有需要，消费者随时可以把自己的 offset 往前调，对以往的消息重新消费。

### 数据消费

消费者部分根据场景而不同，通常只要使用相应消息队列的客户端库实现消费者，根据生产者生成的消息格式进行解析处理即可


## CDC 适用场景

由于 CDC 具备捕获变化数据的能力，你可以简单将 CDC 理解为数据变化（事件）驱动数据再分发系统。其独特的捕获变化数据的能力，让其可以颠覆很多已有的编程的思维

### 异构数据库同步或者备份

由于 CDC 具备捕获变化数据的能力，只需要在数据库 A 集群中通过 CDC 捕获变动数据，清洗、整形和处理数据后，即可导入数据库集群 B 中。

### 解耦异构数据分析计算

在导出整个数据库全量数据后，数据分析系统可以通过订阅感兴趣的数据表的变更，来获取所需要的分析数据进行处理，不需要把分析流程嵌入到已有系统中，以实现解耦。这样的思路可以极大地解耦底层的数据库设计，从而分离底层的数据实现，让异构数据分析成为可能

### 动态缓存更新

在应对高并发、大量数据查询的情况下，除了并发非堵塞基于事件的编程模型外，业界几乎都会毫不犹豫地提及到缓存。将要计算的结果缓存起来或者将高频访问的数据提前缓存，等有请求到来时直接从缓存中读取，这几乎是不可能摆脱的编程实现。

但是，但是。缓存这事情，一直都是计算机界的大难题，缓存缓存，什么时候清理什么时候更新，那可真是写一本书都写不完的事情。

由于 CDC 具备捕获变化数据的能力，考虑这样的一种实现，在系统启动的时候，就能将数据库的所有内容都写入缓存内，然后通过 CDC 来不断更新缓存内容即可。这样的缓存架构实现，仅仅是底层的数据库、缓存之间的同步和实现，与上层的应用无关，能够极大地解耦编程中 `Data Access Layer` 部分的实现，编程逻辑中无须考虑缓存更新和失效问题，其带来的代码迭代升级和数据库维护的好处是显而易见的。

### CQRS 的 Query 视图更新

CQRS 最早来自于 Betrand Meyer（Eiffel语言之父，开-闭原则 OCP 提出者）在 Object-Oriented Software Construction 这本书中提到的一种 命令查询分离 (Command Query Separation, CQS) 的概念。不过分介绍 CQRS 这概念，简单来说，你可以将 CQRS 理解为一种高配版的数据读写分离的设计。

其实在 CQRS 的概念体系中，已经依稀可以看到 CDC 的影子了，只是没有单独将其单独抽象为可以应用在所有数据变动的情况加以讨论。其思想上是很简单的，但是其实现，动辄就是 DDD 和 EventBus/EventSourcing 简直能把小白（比如我这样的）绕到万丈深坑里面去。

如果你用 CDC 的做法来看待 CQRS，你会发现 CQRS 无比简单。抄袭个例子

> 举个例子，我们前面讲了可以利用 CDC 将 MySQL 的数据同步到 Elasticsearch 中以供搜索，在这样的架构里，所有的查询都用 ES 来查，但在想修改数据时，并不直接修改 ES 里的数据，而是修改上游的 MySQL 数据，使之产生数据更新事件，事件被消费者消费来更新 ES 中的数据，这就基本上是一种 CQRS 模式。而在其他 CQRS 的系统中，也可以利用类似的方式来更新查询视图。 [^1]

## MySQL CDC 实战

文章的开头讲述了一个我遇到 CDC 的故事，在整理完（抄袭完）所有的关于 CDC 的概念、架构和场景，这里写一个关于我自己编程实现简单 CDC 的实战的例子。

### 配置 MySQL 生成二进制日志

MySQL reolication 的例子，随便网上搜索一下一抓一大把，只需要修改 MySQL 的配置[^2]，然后重启一下即可


	[mysqld]
	server-id         = 1
	log-bin           = /var/log/mysql/mysql-bin.log
	expire_logs_days  = 10
	max_binlog_size   = 100M
	binlog-format     = row
    
注意一下：`binlog-format = row` 这一行非常重要，这个非常重要 events，一定要设置，只有设置了才能监听到 insert\delete\updates 等事件！

### 导出 MySQL binlog

编程实现导出 MySQL 二进制日志其实是件很简单的事情。简单来说，你只要编程模拟一个 MySQL reolication slave 即可，这个模拟的 slave 不断拿到 MySQL binlog 然后自己按照 MySQL binlog 的格式解析即可。

我编程中是使用 [python-mysql-replication](https://github.com/noplay/python-mysql-replication) 这样一个现成的客户端，直接可以模拟出一个 MySQL replication slave 来连接到 MySQL 服务器，然后不断生成解析出来的二进制日志。

### 清洗并转换  MySQL binlog 成为可执行的 SQL 语句

直接 Github 搜索 `binlog2sql` 有无数这样的实现。其实 [python-mysql-replication](https://github.com/noplay/python-mysql-replication) 也有直接输出实际执行 SQL 语句的能力的，具体的实现就请自行看代码啦

### 最后总结

那么这个`如何安全地准实时进行 MySQL 全球数据同步` 难题，最后就剩下同步转换出来的 `sql statement` 的问题了。直接使用 `rsync` 全球同步生成出来的 `mysql.diff.sql` 应该不是什么大问题。

## 总结

完整的 `Change Data Capture` 其实是思维的转变，让数据真正成为数据，解耦逻辑和数据的实现。我觉得这样的思维转变，其成效是非常巨大的，其指导的思想可以发散到系统架构实现的方方面面。


## 参考引用

[^1]: CQRS 的 Query 视图更新 [例子来源](https://farer.org/2018/07/27/change-data-capture/)
[^2]: [MySQL到Redis同步方案之基于Python的CDC变更数据捕获](http://www.evenvi.com/index.php/archives/63/)
