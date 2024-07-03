## Facebook

在 Facebook，我们将 RocksDB 用作多种数据管理服务的存储引擎以及多种有状态服务的后台，其中包括：

1. [MyRocks](https://github.com/MySQLOnRocksDB/mysql-5.6)
2. [MongoRocks](https://github.com/mongodb-partners/mongo-rocks)
3. [ZippyDB](https://www.youtube.com/watch?v=DfiN7pG0D0khtt)
	1. Facebook 的分布式键值存储，采用 Paxos 式复制，构建于 RocksDB 之上
	2. [https://research.facebook.com/publications/realtime-data-processing-at-facebook/](https://research.facebook.com/publications/realtime-data-processing-at-facebook/)
4. Laser
	1. Laser 是一种高查询吞吐量、低（毫秒级）延迟、基于 RocksDB 的键值存储服务
	2. [https://research.facebook.com/publications/realtime-data-processing-at-facebook/](https://research.facebook.com/publications/realtime-data-processing-at-facebook/)
5. [Dragon](https://code.facebook.com/posts/1737605303120405/dragon-a-distributed-graph-query-engine/)
	1. 分布式图查询引擎
6. Stylus
	1. 是一个用 C++ 编写的低级流处理框架
	2. [https://research.facebook.com/publications/realtime-data-processing-at-facebook/](https://research.facebook.com/publications/realtime-data-processing-at-facebook/)
7. LogDevice
	1. 日志的分布式数据存储
	2. [https://code.facebook.com/posts/357056558062811/logdevice-a-distributed-data-store-for-logs/](https://code.facebook.com/posts/357056558062811/logdevice-a-distributed-data-store-for-logs/)
8. Libra
	1. 区块链
	2. https://libra.org/

## 领英

领英的两个不同用例都使用 RocksDB 作为存储引擎：

1. LinkedIn 用于存储用户活动的关注源。查看博文：[https://engineering.linkedin.com/blog/2016/03/followfeed--linkedin-s-feed-made-faster-and-smarter](https://engineering.linkedin.com/blog/2016/03/followfeed--linkedin-s-feed-made-faster-and-smarter)
2. Apache Samza，用于流处理的开源框架 在 Ankit Gupta 和 Naveen Somasundaram 的技术讲座中了解有关这些用例的更多信息：[http://www.youtube.com/watch?v=plqVp_OnSzg](http://www.youtube.com/watch?v=plqVp_OnSzg)

## 雅虎

雅虎正在使用 RocksDB 作为其最大的分布式数据存储 Sherpa 的存储引擎。点击此处了解更多信息：[http://yahooeng.tumblr.com/post/120730204806/sherpa-scales-new-heights](http://yahooeng.tumblr.com/post/120730204806/sherpa-scales-new-heights)

## CockroachDB

CockroachDB 是一个开源的地理复制事务数据库，使用 RocksDB 作为存储引擎，查看他们的 github：[https://github.com/cockroachdb/cockroach](https://github.com/cockroachdb/cockroach)

## DNANexus

DNANexus 正在使用 RocksDB 加快基因组学数据的处理速度。您可以从 Mike Lin 的这篇精彩博文中了解更多信息：[http://devblog.dnanexus.com/faster-bam-sorting-with-samtools-and-rocksdb/](http://devblog.dnanexus.com/faster-bam-sorting-with-samtools-and-rocksdb/)

## Iron.io

Iron.io 将 RocksDB 用作其分布式队列系统的存储引擎。从 Reed Allman 的技术演讲中了解更多信息：[http://www.youtube.com/watch?v=HTjt6oj-RL4](http://www.youtube.com/watch?v=HTjt6oj-RL4)

## Tango Me

Tango 使用 RocksDB 作为图存储来存储所有用户的连接数据和其他社交活动数据。

## Turn

Turn 使用 RocksDB 作为其键/值存储的存储层，在不同的数据中心提供峰值 2.4MM QPS 的服务。在以下网站查看我们的 RocksDB Protobuf 合并运算符：[https://github.com/vladb38/rocksdb_protobuf](https://github.com/vladb38/rocksdb_protobuf)

## Santanader UK/Cloudera Profession Services

查看他们的博文：[http://blog.cloudera.com/blog/2015/08/inside-santanders-near-real-time-data-ingest-architecture/](http://blog.cloudera.com/blog/2015/08/inside-santanders-near-real-time-data-ingest-architecture/)

## Airbnb

Airbnb 正在使用 RocksDB 作为其个性化搜索服务的存储引擎。您可以在此了解更多信息：[https://www.youtube.com/watch?v=ASQ6XMtogMs](https://www.youtube.com/watch?v=ASQ6XMtogMs)

## Alluxio

Alluxio 使用 RocksDB 提供文件系统元数据服务并将其扩展到超过 10 亿个文件。本工程博客将介绍详细的设计和实施过程：[https://www.alluxio.io/blog/scalable-metadata-service-in-alluxio-storing-billions-of-files/](https://www.alluxio.io/blog/scalable-metadata-service-in-alluxio-storing-billions-of-files/)

## Pinterest

Pinterest 的对象检索系统使用 RocksDB 进行存储：[https://www.youtube.com/watch?v=MtFEVEs_2Vo](https://www.youtube.com/watch?v=MtFEVEs_2Vo)

## Smyte

Smyte 使用 RocksDB 作为其核心键值存储、高性能计数器和时间窗口 HyperLogLog 服务的存储层。

## Rakuten Marketing

Rakuten Marketing 将 RocksDB 用作其 Performance DSP 中实时竞价服务的磁盘缓存层。

## VWO, Wingify

VWO 的智能代码检查器和 URL 助手使用 RocksDB 来存储所有安装了 VWO 智能代码的 URL。

## quasardb

quasardb 是一种高性能、分布式、事务型键值数据库，能与 Apache Spark 等内存分析引擎很好地集成。

## Netflix

Netflix Netflix 在带有本地 SSD 驱动器的 AWS EC2 实例上使用 RocksDB 来缓存应用数据。

## TiKV

TiKV 是一个 GEO 复制、高性能、分布式、事务型键值数据库。TiKV 由 Rust 和 Raft 支持。TiKV 使用 RocksDB 作为持久层。

## Apache Flink

Apache Flink 使用 RocksDB 在机器上本地存储状态。

## Dgraph

Dgraph 是一个开源、可扩展、分布式、低延迟、高吞吐量的图形数据库。他们使用 RocksDB 在机器上本地存储状态。

## Uber

Uber 使用 RocksDB 作为持久的、可扩展的任务队列。

## 360 Pika

360 Pika 是与 redis 兼容的 nosql。由于存储的数据量巨大，redis 可能会遭遇容量瓶颈，而 Pika 就是为解决这一问题而诞生的。它已广泛应用于许多公司

## LzLabs

LzLabs 在其多数据库分布式框架中使用 RocksDB 作为存储引擎，以存储应用配置和用户数据。

## ProfaneDB

ProfaneDB 是用于协议缓冲区的数据库，使用 RocksDB 进行存储。它可通过 gRPC 访问，模式直接使用 .proto 文件定义。

## IOTA Foundation

IOTA 基金会正在 IOTA 参考实现（IRI）中使用 RocksDB 来存储 Tangle 的本地状态。Tangle 是首个开源分布式账本，将为未来的物联网提供动力。

## Avrio Project

Avrio 项目在 Avrio 中使用 RocksDB 来存储区块、账户余额和数据以及其他区块链发布的数据。Avrio 是一种多区块链去中心化加密货币，赋予货币交易权力。

## XTDB

XTDB（前身为 Crux）是一个文档数据库，使用 RocksDB 进行本地 EAV 索引存储，以实现时间点位时态 Datalog 查询。非捆绑式 "架构使用 Kafka 提供横向扩展能力。

## Nebula Graph

Nebula Graph 是一个分布式、可扩展、快如闪电的开源图形数据库，能够托管拥有数百亿个顶点（节点）和数万亿条边的超大规模图形，延迟时间仅为几毫秒。

## Apache Hadoop Ozone

Ozone 是一个用于 Hadoop 的可扩展、冗余和分布式对象存储。除了可扩展到数十亿个不同大小的对象外，Ozone 还能在 Kubernetes 和 YARN 等容器化环境中有效运行：[https://blog.cloudera.com/apache-hadoop-ozone-object-store-architecture/](https://blog.cloudera.com/apache-hadoop-ozone-object-store-architecture/)

## Apache Doris

[http://doris.apache.org/master/en/administrator-guide/operation/tablet-meta-tool.html](http://doris.apache.org/master/en/administrator-guide/operation/tablet-meta-tool.html)

## Apache Pegasus

Apache Pegasus 是一种可横向扩展、一致性强且高性能的键值存储：[https://github.com/apache/incubator-pegasus](https://github.com/apache/incubator-pegasus)
