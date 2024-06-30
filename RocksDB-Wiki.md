## 欢迎访问 RocksDB

RocksDB 是一个带有键/值接口的存储引擎，其中键和值是任意的字节流。它是一个 C++ 库。它是 Facebook 基于 LevelDB 开发的，为 LevelDB API 提供向后兼容支持。

RocksDB 支持各种存储硬件，最初的重点是速度更快的闪存。它使用日志结构化数据库引擎（Log Structured Database Engine）进行存储，完全使用 C++ 编写，并有一个名为 RocksJava 的 Java 封装器。请参见 [[RocksJava-Basics|RocksJava 基础知识]]。

RocksDB 可以适应各种生产环境，包括纯内存、闪存、硬盘或远程存储。在 RocksDB 无法自动适应的情况下，它提供了高度灵活的配置设置，允许用户自行调整。它支持各种压缩算法，并为生产支持和调试提供了良好的工具。

## 功能

- 专为希望在本地或远程存储系统上存储多达几 TB 数据的应用服务器而设计
- 针对在快速存储设备（闪存设备或内存）上存储中小型键值进行了优化
	- 小型键值：几个字节到几十个字节
	- 中型键值：几十个字节到几百个字节
- 在多核处理器上运行良好

## LevelDB 中没有的功能

RocksDB 引入了数十项新的主要功能，查看 [[Features-Not-in-LevelDB|LevelDB 中没有的功能列表]]。

## 入门

有关完整目录，请参阅 [[Contents|目录]]。大多数读者会希望从开发人员指南的 [[Overview|概述]] 和 [[Basic-Operations|基本操作]] 部分开始阅读。根据 [[Setup-Options-and-Basic-Tuning|设置选项和基本调优]] 来设置你的初始选项。还可以查看 [[RocksDB-FAQ|RocksDB 常见问题]]。此外，我们还为高级RocksDB用户准备了 [[Tuning-Guide|RocksDB调试指南]]。

有关如何构建 Rocksdb 的说明，请查看 [INSTALL.md](https://github.com/facebook/rocksdb/blob/main/INSTALL.md)。

## 发布

RocksDB 在 github 上发布。对于 Java 用户，Maven 工件也会更新。参见 [[Release-Methodology|RocksDB 发布方法]]。

## 为 RocksDB 做贡献

欢迎您发送 PR，为 RocksDB 代码库做出贡献！请查看 [[Contribution-Guide]] 指南。

## 排除故障和寻求帮助

请遵循 [[Troubleshooting-Guide|RocksDB 故障排除指南]] 的指导。

## 博客

请访问我们的博客 rocksdb.org/blog

## 项目历史

- [The History of RocksDB](https://rocksdb.blogspot.com/2013/11/the-history-of-rocksdb.html)
- [Under the Hood: Building and open-sourcing RocksDB](https://www.facebook.com/notes/facebook-engineering/under-the-hood-building-and-open-sourcing-rocksdb/10151822347683920)

## 链接

- [Examples](https://github.com/facebook/rocksdb/tree/main/examples)
- [Official Blog](http://rocksdb.org/blog/)
- [Stack Overflow: RocksDB](https://stackoverflow.com/questions/tagged/rocksdb)
- [Talks](https://github.com/facebook/rocksdb/wiki/Talks)

## 联系方式

- [RocksDB Google Group](https://groups.google.com/forum/#!forum/rocksdb)
- [RocksDB Facebook Group](https://www.facebook.com/groups/rocksdb.dev/)
- [RocksDB Github Issues](https://github.com/facebook/rocksdb/issues)
- [Asking For Help](https://github.com/facebook/rocksdb/wiki/RocksDB-Troubleshooting-Guide#asking-for-help)

我们只使用 github issues 来进行 bug 报告，而使用 RocksDB 的 Google Group 或 Facebook Group 来处理其他问题。用户并不总是很清楚这是否是 RocksDB 的 bug。请根据你的判断选择一个。
