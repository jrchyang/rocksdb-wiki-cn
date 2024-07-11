rocksdb 库提供了一个持久的键值存储库。键和值是任意字节数组。键根据用户指定的比较器函数在键值存储区内排序。

## 打开数据库

rocksdb 数据库有一个与文件系统目录相对应的名称，数据库的所有内容都存储在这个目录中。下面的示例展示了如何打开数据库，必要时创建数据库：

```C++
#include <cassert>
#include "rocksdb/db.h"

rocksdb::DB* db;
rocksdb::Options options;
options.create_if_missing = true;
rocksdb::Status status = rocksdb::DB::Open(options, "/tmp/testdb", &db);
assert(status.ok());
```

如果想在数据库已存在的情况下引发错误，请在 `rocksdb::DB::Open` 调用前添加以下一行：

```C++
options.error_if_exists = true;
```

如果你要把代码从 leveldb 移植到 rocksdb，可以使用 `rocksdb::LevelDBOptions` 把 `leveldb::Options` 对象转换成 `rocksdb::Options`，它的功能与 `leveldb::Options` 相同：

```C++
#include "rocksdb/utilities/leveldb_options.h"

rocksdb::LevelDBOptions leveldb_options;
leveldb_options.option1 = value1;
leveldb_options.option2 = value2;
// ...
rocksdb::Options options = rocksdb::ConvertOptions(leveldb_options);
```

## RocksDB 选项

用户可以选择始终在代码中显式设置选项字段，如上所示。或者，也可以通过字符串到字符串映射或选项字符串来设置。请参阅 [[Option-String-and-Option-Map|选项字符串和选项映射]] 。

某些选项可以在 DB 运行时动态更改。例如：

```C++
rocksdb::Status s;
s = db->SetOptions({{"write_buffer_size", "131072"}});
assert(s.ok());
s = db->SetDBOptions({{"max_background_flushes", "2"}});
assert(s.ok());
```

RocksDB 会自动将数据库中使用的选项保存在 DB 目录下的 OPTIONS-xxxx 文件中。用户可以选择从这些选项文件中提取选项，从而在数据库重启后保留选项值。参见 [[Options-File|RocksDB 选项文件]] 。

## 状态返回值

你可能已经注意到上面的 `rocksdb::Status` 类型。 在 RocksDB 中，大多数函数在遇到错误时都会返回这种类型的值。 你可以检查这样的结果是否正常，也可以打印相关的错误信息：

```C++
rocksdb::Status s = ...;
if (!s.ok()) cerr << s.ToString() << endl;
```

## 关闭数据库

使用完数据库后，有 3 种方法可以优雅地关闭数据库：

1. 只需删除数据库对象即可。这将释放数据库打开时保留的所有资源。但是，如果在释放任何资源时遇到任何错误，例如在关闭 info_log 文件时出错，这些资源就会丢失
2. 调用 `DB::Close()`，然后删除数据库对象。`DB::Close()` 返回 `Status`，通过检查 `Status` 可以确定是否有任何错误。无论是否出错，`DB::Close()` 都将释放所有资源，并且是不可逆的
3. 在使用 `WaitForCompactOptions.close_db=true` 时调用 `DB::WaitForCompact()`。 `DB::WaitForCompact()` 将在等待运行中的后台作业结束后内部调用 `DB::Close()`。如果用户希望在关闭前等待后台工作，而不是在重新打开时中止并可能重做一些工作，建议选择此方法

示例：

```C++
// ... open the db as described above ...
// ... do something with db ...
delete db;
```

或者

```C++
// ... open the db as described above ...
// ... do something with db ...
Status s = db->Close();
// ... log status ...
delete db;
```

或者

```C++
// ... open the db as described above ...
// ... do something with db ...
opt = WaitForCompactOptions();
opt.close_db = true;
Status s = db->WaitForCompact(opt);
// ... log status ...
delete db;
```

## 读请求

数据库提供 `Put`、`Delete`、`Get` 和 `MultiGet` 方法来修改/查询数据库。例如，以下代码将存储在 key1 下的值移动到 key2：

```C++
std::string value;
rocksdb::Status s = db->Get(rocksdb::ReadOptions(), key1, &value);
if (s.ok()) s = db->Put(rocksdb::WriteOptions(), key2, value);
if (s.ok()) s = db->Delete(rocksdb::WriteOptions(), key1);
```

现在，值的大小必须小于 4GB。RocksDB 还允许 [[Single-Delete|单次删除]] ，这在某些特殊情况下非常有用。

每次 "获取" 都会导致从源字符串到值字符串的至少一次 memcpy。如果源文件在块缓存中，则可以使用 PinnableSlice 来避免额外的复制，如下：

```C++
PinnableSlice pinnable_val;
rocksdb::Status s = db->Get(rocksdb::ReadOptions(), key1, &pinnable_val);
```

一旦 `pinnable_val` 被析构或调用 `::Reset` 时，源将被释放。 点击 [PinnableSlice: less memcpy with point lookups](https://rocksdb.org/blog/2017/08/24/pinnableslice.html) 了解更多。

从数据库读取多个键时，可以使用 `MultiGet` 。`MultiGet` 有两种实现： 

1. 以更高效的方式从单个列族读取多个键，即比在循环中调用 Get 更快
2. 在多个列族中读取的键值相互一致

例如：

```C++
std::vector<Slice> keys;
std::vector<PinnableSlice> values;
std::vector<Status> statuses;

for ... {
  keys.emplace_back(key);
}
values.resize(keys.size());
statuses.resize(keys.size());

db->MultiGet(ReadOptions(), cf, keys.size(), keys.data(), values.data(),
	     statuses.data());
```

为了避免内存分配的开销，上述 `keys` 、`values` 和 `statuses` 可以是堆栈上的 `std::array` 类型，也可以是任何其他能提供连续存储的类型。

或者

```C++
std::vector<ColumnFamilyHandle*> column_families;
std::vector<Slice> keys;
std::vector<std::string> values;

for ... {
  keys.emplace_back(key);
  column_families.emplace_back(column_family);
}
values.resize(keys.size());

std::vector<Status> statuses = db->MultiGet(ReadOptions(), column_families, keys, 
					    &values);
```

有关使用 `MultiGet` 的性能优势的更深入讨论，请参阅 [[MultiGet-Performance|MultiGet 性能]] 。

## 写请求

### 原子更新

需要注意的是，如果进程在 key2 的 Put 之后、key1 的 delete 之前退出，那么相同的值可能会被保存在多个键下。通过使用 `WriteBatch` 类原子式地应用一组更新，可以避免此类问题：

```C++
#include "rocksdb/write_batch.h"
// ...
std::string value;
rocksdb::Status s = db->Get(rocksdb::ReadOptions(), key1, &value);
if (s.ok()) {
  rocksdb::WriteBatch batch;
  batch.Delete(key1);
  batch.Put(key2, value);
  s = db->Write(rocksdb::WriteOptions(), &batch);
}
```

`WriteBatch` 保存了要对数据库进行的一系列编辑，批处理中的这些编辑将按顺序应用。请注意，我们在 `Put` 之前调用了 `Delete`，这样如果 `key1` 与 `key2` 相同，我们就不会错误地完全丢弃该值。

除了原子性优势外，`WriteBatch` 还可以通过将大量单个突变放入同一批次来加快批量更新的速度。

### 同步写

默认情况下，向 RocksDB 的每次写入都是异步的：将写入从进程推送到操作系统后就返回，从操作系统内存到底层持久化存储的传输是异步进行的。可以为特定的写操作开启同步标志，使写操作在写入的数据被推送到持久性存储之前不会返回。 (在 Posix 系统上，可以通过在写操作返回前调用 `fsync()` 或 `fdatasync()` 或 `msync(..., MS_SYNC)` 来实现）

```C++
rocksdb::WriteOptions write_options;
write_options.sync = true;
db->Put(write_options, ...);
```

### 非同步写

对于非同步写入，RocksDB 只会在操作系统缓冲区或内部缓冲区（当 `options.manual_wal_flush = true` 时）中缓冲 WAL 写入，它们通常比同步写入快得多。非同步写入的缺点是，机器崩溃可能会导致最后几次更新丢失。请注意，仅写入进程崩溃（即未重启）不会导致任何丢失，因为即使 `sync` 为假，更新的数据只有在从进程内存推送到操作系统内存之后才会被认为完成。

非同步写入通常可以安全使用。 例如，在向数据库加载大量数据时，可以通过在崩溃后重新启动批量加载来处理丢失的更新。也可以采用混合方案，即由单独的线程调用 `DB::SyncWAL()` 。

我们还为某些特定写请求提供了一种完全禁用 Write Ahead Log 的方法。如果将 `write_options.disableWAL` 设置为 `true`，写入的内容将完全不进入日志，并可能在进程崩溃时丢失。

RocksDB 默认使用 `fdatasync()` 同步文件，在某些情况下可能比 `fsync()` 更快。 如果你想使用 `fsync()`，可以将 `Options::use_fsync` 设置为 `true`。 在 ext3 等重启后可能丢失文件的文件系统上，应将此设置为 true。

### 进一步

有关写入性能优化和影响性能因素的更多信息，请参阅 [[Pipelined-Write|流水线写入]] 和 [[Write-Stalls|写入延迟]]。

## 并发性

一个数据库一次只能由一个进程打开。 为防止误操作，RocksDB 实现会从操作系统获取一个锁。 在一个进程中，多个并发线程可以安全地共享同一个 `rocksdb::DB` 对象，也就是说，不同的线程可以在同一个数据库中写入或获取迭代器，或调用 `Get`，而无需任何外部同步（RocksDB 实现会自动完成所需的同步）。但其他对象（如 `Iterator` 和 `WriteBatch`）可能需要外部同步，如果两个线程共享这样一个对象，它们必须使用自己的锁协议来保护对该对象的访问。更多详情请查看公共头文件。

## 合并操作符

合并操作符为 读取-修改-写入 操作提供有效支持。 有关接口和实现的更多信息，请访问：

- [[Merge-Operator]]
- [[Merge-Operator-Implementation]]
- [[Merge-Operator#Get Merge Operands|Get Merge Operands]]

## 迭代

下面的示例演示了如何打印数据库中的所有（键、值）对：

```C++
rocksdb::Iterator* it = db->NewIterator(rocksdb::ReadOptions());
for (it->SeekToFirst(); it->Valid(); it->Next()) {
  cout << it->key().ToString() << ": " << it->value().ToString() << endl;
}
assert(it->status().ok()); // Check for any errors found during the scan
delete it;
```

下面的变体演示了如何只处理范围 \[start, limit) 内的键：

```C++
for (it->Seek(start);
  it->Valid() && it->key().ToString() < limit;
  it->Next()) {
  // ...
}
assert(it->status().ok()); // Check for any errors found during the scan
```

您还可以按相反顺序处理条目，需要注意的是，反向迭代可能比正向迭代慢一些，如下：

```C++
for (it->SeekToLast(); it->Valid(); it->Prev()) {
  // ...
}
assert(it->status().ok()); // Check for any errors found during the scan
```

以下示例演示了如何从一个特定键倒序处理范围内的条目（limit，start]：

```C++
for (it->SeekForPrev(start);
  it->Valid() && it->key().ToString() > limit;
  it->Prev()) {
  // ...
}
assert(it->status().ok()); // Check for any errors found during the scan
```

参考：

- 请参见 [[SeekForPrev]] 
- 有关错误处理、不同迭代选项和最佳实践的解释，请参阅 [[Iterator|迭代器]] 
- 要了解实现细节，请参阅 [[Iterator-Implementation|迭代器的实现]]

## 快照

快照为键值存储的整个状态提供一致的只读视图。`ReadOptions::snapshot` 可以是非空的，以表示读取操作应在特定版本的数据库状态上进行。如果 `ReadOptions::snapshot` 为 NULL，读取将在当前状态的隐式快照上进行。

快照由 `DB::GetSnapshot()` 方法创建：

```C++
rocksdb::ReadOptions options;
options.snapshot = db->GetSnapshot();
// ... apply some updates to db ...
rocksdb::Iterator* iter = db->NewIterator(options);
// ... read using iter to view the state when the snapshot was created ...
delete iter;
db->ReleaseSnapshot(options.snapshot);
```

请注意，当不再需要快照时，应使用 `DB::ReleaseSnapshot` 接口释放快照。这将允许实现摆脱为支持读取该快照而维护的状态。

## Slice

上述 `it->key()` 和 `it->value()` 调用的返回值是 `rocksdb::Slice` 类型的实例。`Slice` 是一个简单的结构，包含一个长度和一个指向外部字节数组的指针，返回 `Slice` 比返回 `std::string` 更高效，因为我们不需要复制可能很大的键和值。此外，由于 RocksDB 的键和值允许包含 '\0' 字符，因此 RocksDB 方法不会返回 C 风格的空尾字符串。

C++ 字符串和 以空字符结尾的 C 风格字符串可轻松转换为 Slice：

```C++
rocksdb::Slice s1 = "hello";

std::string str("world");
rocksdb::Slice s2 = str;
```

Slice 可以很容易地转换回 C++ 字符串：

```C++
std::string str = s1.ToString();
assert(str == std::string("hello"));
```

使用 Slice 时要小心，因为调用者必须确保在使用切片时，切片指向的外部字节数组仍然有效。 例如，以下代码就存在错误：

```C++
rocksdb::Slice slice;
if (...) {
  std::string str = ...;
  slice = str;
}
Use(slice);
```

当 `if` 语句退出作用域时，`str` 将被销毁，`slice` 的指向的存储也将消失。

## 事务

RocksDB 现在支持多操作事务，查看 [[Transactions|事务]]

## 比较器

前面的示例使用了 key 的默认排序方法，即按词典顺序对字节进行排序。不过，您可以在打开数据库时提供自定义比较器。例如，假设每个数据库键都由两个数字组成，我们应该按第一个数字排序，按第二个数字打破平局。首先，定义一个合适的 `rocksdb::Comparator` 子类来表达这些规则：

```C++
class TwoPartComparator : public rocksdb::Comparator {
public:
  // Three-way comparison function:
  // if a < b: negative result
  // if a > b: positive result
  // else: zero result
  int Compare(const rocksdb::Slice& a, const rocksdb::Slice& b) const {
    int a1, a2, b1, b2;
    ParseKey(a, &a1, &a2);
    ParseKey(b, &b1, &b2);
    if (a1 < b1) return -1;
    if (a1 > b1) return +1;
    if (a2 < b2) return -1;
    if (a2 > b2) return +1;
    return 0;
}

  // Ignore the following methods for now:
  const char* Name() const { return "TwoPartComparator"; }
  void FindShortestSeparator(std::string*, const rocksdb::Slice&) const { }
  void FindShortSuccessor(std::string*) const { }
};
```

现在使用这个自定义比较器创建一个数据库：

```C++
TwoPartComparator cmp;
rocksdb::DB* db;
rocksdb::Options options;
options.create_if_missing = true;
options.comparator = &cmp;
rocksdb::Status status = rocksdb::DB::Open(options, "/tmp/testdb", &db);
// ...
```

## 列族

[[Column-Families|列族]] 提供了一种对数据库进行逻辑分区的方法。用户可以在多个列族中对多个键进行原子写入，并从中读取一致的视图。

## 批量加载

您可以 [[Creating-and-Ingesting-SST-files|创建和提取 SST 文件]] ，将大量数据直接批量加载到数据库中，对实时流量的影响最小。

## 备份和检查点

[[How-to-Backup-RocksDB|备份]] 允许用户在远程文件系统（如 HDFS 或 S3）中创建定期增量备份，并从其中任何一个备份中恢复。

[[Checkpoints|检查点]] 功能可以在单独的目录中对正在运行的 RocksDB 数据库进行快照。如果可能的话，文件会进行硬链接而不是复制，因此这是一个相对轻量的操作。

## I/O

默认情况下，RocksDB 的 I/O 会经过操作系统的页面缓存。设置 [[Rate-Limiter|速率限制器]] 可以限制 RocksDB 发出文件写入的速度，从而为读取 I/O 腾出空间。

用户还可以选择使用 [[Direct-IO]] 来绕过操作系统的页面缓存。

详见 [[IO]] 。

## 向后兼容性

比较器的 `Name` 方法的结果在创建数据库时附加到数据库，并在每次后续打开数据库时检查。如果名称发生变化，`rocksdb::DB::Open` 调用将失败。因此，当且仅当新 key 格式和比较函数与现有数据库不兼容时才更改名称，并且可以丢弃所有现有数据库的内容。

但是，只要事先稍加规划，您仍可以逐渐改进 key 格式。例如，您可以在每个 key 的末尾存储一个版本号（对于大多数用途，一个字节就足够了）。当您希望切换到新的 key 格式（例如，向 TwoPartComparator 处理的 key 添加可选的第三方）时，(a) 保留相同的比较器名称 (b) 增加新密钥的版本号 (c) 更改比较器函数，使其使用 key 中找到的版本号来决定如何解释它们。

## MemTable and Table factories

默认情况下，我们将内存中的数据保存在 skiplist memtable 中，并将磁盘上的数据保存在以下描述的表格式中：[[Block-based-Table-Format]] 和 [[PlainTable-Format]]

由于 RocksDB 的目标之一是让系统的不同部分都能轻松插入，因此我们支持 memtable 和 table 格式的不同实现。你可以通过设置 `Options::memtable_factory` 来提供自己的memtable工厂，也可以通过设置 `Options::table_factory` 来提供自己的表工厂。关于可用的 memtable 工厂，请参阅 `rocksdb/memtablerep.h`，关于 table 工厂，请参阅 `rocksdb/table.h` 。这些功能都在积极开发中，请注意任何可能会破坏你的应用程序的 API 变动。

您还可以在 [[MemTable|这里]] 了解有关 memtable 的更多信息。

## 性能

从 [[Setup-Options-and-Basic-Tuning|选项设置和基本调整]] 开始。有关 RocksDB 性能的更多信息，请参阅右侧边栏中的 "性能" 部分。

## 块大小

RocksDB 将相邻的键组合到同一个块中，并且这样的块是与持久存储之间传输的单位。默认块大小约为 4096 个未压缩字节。主要对数据库内容进行批量扫描的应用程序可能希望增加此大小。如果性能测量表明有所改善，则对小值进行大量点读取的应用程序可能希望切换到较小的块大小。使用小于一千字节或大于几兆字节的块没有太大好处。另请注意，块大小越大，压缩效果越好。要更改块大小参数，请使用 `Options::block_size` 。

## 写缓冲区

`Options::write_buffer_size` 指定了在转换为磁盘上排序文件之前内存中的数据量。数值越大，性能越高，尤其是在批量加载时。内存中最多可同时容纳 `max_write_buffer_number` 个写缓冲区，因此你可能需要调整这个参数来控制内存的使用。此外，写缓冲区越大，下次打开数据库时的恢复时间就越长。

与此相关的选项是 `Options::max_write_buffer_number` ，它是内存中写入缓冲区的最大数量。默认值为 2，这样当一个写缓冲区被刷新到存储空间时，新的写入可以继续到另一个写缓冲区。刷新操作在 [[Thread-Pool|线程池]] 中执行。

`Options::min_write_buffer_number_too_merge` 是在写入存储之前合并在一起的写缓冲区的最小数量。如果设置为 1，那么所有写缓冲区都会作为单独文件刷新到 L0，这会增加读取放大，因为获取请求必须检查所有这些文件。此外，如果每个单独的写缓冲区中都有重复的记录，那么内存中的合并可能会导致写入存储的数据减少。默认值：1

## 压缩

在写入持久性存储之前，每个数据块都会被单独压缩。由于默认的压缩方法速度非常快，因此默认情况下压缩是开启的，对于无法压缩的数据，压缩会自动关闭。在极少数情况下，应用程序可能希望完全禁用压缩，但只有在基准测试表明性能有所提高的情况下才可以这样做：

```C++
rocksdb::Options options;
options.compression = rocksdb::kNoCompression;
// ... rocksdb::DB::Open(options, name, ...) ....
```

此外，还提供 [[Dictionary-Compression|字典压缩]] 功能。

## 缓存

数据库的内容存储在文件系统中的一组文件中，每个文件都存储一系列压缩块。如果 `options.block_cache` 非 NULL，则用于缓存经常使用的未压缩块内容。我们使用操作系统文件缓存来缓存经过压缩的原始数据。因此，文件缓存充当压缩数据的缓存。

```C++
#include "rocksdb/cache.h"
rocksdb::BlockBasedTableOptions table_options;
table_options.block_cache = rocksdb::NewLRUCache(100 * 1048576); // 100MB uncompressed cache

rocksdb::Options options;
options.table_factory.reset(rocksdb::NewBlockBasedTableFactory(table_options));
rocksdb::DB* db;
rocksdb::DB::Open(options, name, &db);
// ... use the db ...
delete db
```

执行批量读取时，应用程序可能希望禁用缓存，以便批量读取处理的数据不会取代大多数缓存内容。可以使用每个迭代器选项来实现这一点：

```C++
rocksdb::ReadOptions options;
options.fill_cache = false;
rocksdb::Iterator* it = db->NewIterator(options);
for (it->SeekToFirst(); it->Valid(); it->Next()) {
  // ...
}
```

您还可以通过将 `options.no_block_cache` 设置为 `true` 来禁用块缓存。

请参阅 [[Block-Cache|块缓存]] 以了解更多详细信息。

## 键布局

请注意，磁盘传输和缓存的单位是块。相邻的键（根据数据库排序顺序）通常会放在同一个块中。因此，应用程序可以通过将一起访问的键放在彼此附近，并将不常使用的键放在键空间的单独区域中来提高其性能。

例如，假设我们在 RocksDB 上实现一个简单的文件系统。我们可能希望存储的条目类型是：

```C++
filename -> permission-bits, length, list of file_block_ids
file_block_id -> data
```

我们可能希望在文件名键前加上一个字母（如"/"），而在 `file_block_id` 键前加上一个不同的字母（如 "0"），这样在只扫描元数据时就不会强迫我们获取和缓存庞大的文件内容。

## 过滤器

由于 rocksdb 数据在磁盘上的组织方式，单个 Get() 调用可能涉及多次磁盘读取。可选的 FilterPolicy 机制可用于大幅减少磁盘读取次数。

```C++
rocksdb::Options options;
rocksdb::BlockBasedTableOptions bbto;
bbto.filter_policy.reset(rocksdb::NewBloomFilterPolicy(
   10 /* bits_per_key */,
   false /* use_block_based_builder*/));
options.table_factory.reset(rocksdb::NewBlockBasedTableFactory(bbto));
rocksdb::DB* db;
rocksdb::DB::Open(options, "/tmp/testdb", &db);
// ... use the database ...
delete db;
delete options.filter_policy;
```

上述代码将基于 Bloom Filter 的过滤策略与数据库关联起来。基于 Bloom Filter 的过滤依赖于每个键在内存中保留一定数量的数据位（在本例中每个键 10 位，因为这是我们传递给 NewBloomFilter 的参数）。此过滤器将 Get() 调用所需的不必要磁盘读取次数减少约 100 倍。增加每个键的位数将导致更大的减少，但代价是更多的内存使用量。我们建议那些工作集无法容纳在内存中且需要进行大量随机读取的应用程序设置过滤器策略。

如果使用自定义比较器，则应确保所使用的过滤策略与比较器兼容。例如，比较器在比较键时忽略尾部空格。NewBloomFilter 不能与这种比较器一起使用。相反，应用程序应提供同样忽略尾部空格的自定义过滤策略。

例如：

```C++
class CustomFilterPolicy : public rocksdb::FilterPolicy {
private:
  FilterPolicy* builtin_policy_;
public:
  CustomFilterPolicy() : builtin_policy_(NewBloomFilter(10, false)) { }
  ~CustomFilterPolicy() { delete builtin_policy_; }

  const char* Name() const { return "IgnoreTrailingSpacesFilter"; }

  void CreateFilter(const Slice* keys, int n, std::string* dst) const {
    // Use builtin bloom filter code after removing trailing spaces
    std::vector<Slice> trimmed(n);
    for (int i = 0; i < n; i++) {
      trimmed[i] = RemoveTrailingSpaces(keys[i]);
    }
    return builtin_policy_->CreateFilter(&trimmed[i], n, dst);
  }

  bool KeyMayMatch(const Slice& key, const Slice& filter) const {
    // Use builtin bloom filter code after removing trailing spaces
    return builtin_policy_->KeyMayMatch(RemoveTrailingSpaces(key), filter);
  }
};
```

高级应用程序可以提供一种不使用 bloom 过滤器，而是使用其他机制来汇总一组键值的过滤策略。详见 `rocksdb/filter_policy.h` 。

## 校验和

RocksDB 会对文件系统中存储的所有数据进行校验。对于如何积极验证这些校验和，有两种不同的控件：

- `ReadOptions::verify_checksums` 会强制对代表特定读取行为从文件系统读取的所有数据进行校验和验证，默认情况下处于开启状态
- 在打开数据库之前，可以将 `Options::paranoid_checks` 设置为 `true`，以便数据库实现在检测到内部损坏时立即引发错误。错误可能在打开数据库时发生，也可能在之后的其他数据库操作中发生，具体取决于数据库的哪个部分被损坏。默认情况下，偏执检查处于开启状态

还可以通过调用 `DB::VerifyChecksum()` 手动触发校验和验证。此 API 遍历所有列族的所有级别的所有 SST 文件，并针对每个 SST 文件验证嵌入在元数据和数据块中的校验和。目前，它仅支持 BlockBasedTable 格式。文件是按顺序验证的，因此 API 调用可能需要大量时间才能完成。此 API 可用于主动验证分布式系统中的数据完整性，例如，如果发现数据库已损坏，则可以创建新的副本。

如果数据库已损坏（可能在开启偏执检查时无法打开），可以使用 `rocksdb::RepairDB` 函数尽可能恢复数据。

## 压实

RocksDB 会不断重写现有的数据文件，这样做是为了清除陈旧版本的键值，并使数据结构在读取时保持最佳状态。

关于压实的信息被移到了 [[Compaction|压实]] 中，用户在操作 RocksDB 之前不必了解压实的内部信息。

## 大致尺寸

`GetApproximateSizes` 方法可用于获取一个或多个键范围所使用的文件系统空间的大致字节数。

```C++
rocksdb::Range ranges[2];
ranges[0] = rocksdb::Range("a", "c");
ranges[1] = rocksdb::Range("x", "z");
uint64_t sizes[2];
db->GetApproximateSizes(ranges, 2, sizes);
```

上述调用将 size\[0] 设置为键范围 \[a..c) 使用的文件系统空间的近似字节数，将 size\[1] 设置为键范围 \[x..z) 使用的近似字节数。

## 环境变量

由 RocksDB 实现发出的所有文件操作（以及其他操作系统调用），都是通过 `rocksdb::Env` 对象进行的。复杂的客户端可能希望提供自己的 `Env` 实现，以获得更好的控制。例如，应用程序可能会在文件 IO 路径中引入人为延迟，以限制 RocksDB 对系统中其他活动的影响。

```C++
class SlowEnv : public rocksdb::Env {
  // .. implementation of the Env interface ...
};

SlowEnv env;
rocksdb::Options options;
options.env = &env;
Status s = rocksdb::DB::Open(options, ...);
```

## 导出

RocksDB 可以通过提供 `rocksdb/port/port.h` 导出的类型/方法/函数的平台特定实现来移植到新平台。有关更多详细信息，请参阅 `rocksdb/port/port_example.h` 。

此外，新平台可能需要一个新的默认 `rocksdb::Env` 实现。示例请参见 `rocksdb/util/env_posix.h` 。

## 可管理性

为了有效地调整应用程序，最好能获得使用统计数据。你可以通过设置 `Options::table_properties_collectors` 或 `Options::statistics` 来收集这些统计数据。更多信息，请参阅 `rocksdb/table_properties.h` 和 `rocksdb/statistics.h` 。这些功能不会给你的应用程序增加大量开销，我们建议将它们导出到其他监控工具，参见 [[Statistics|统计数据]] 。你还可以使用 [[Perf-Context-and-IO-Stats-Context]] 对单个请求进行剖析。用户可以注册 [[EventListener]] 来回调某些内部事件。

## 清除 WAL 文件

默认情况下，旧的预写日志超出范围且应用程序不再需要时会自动删除。有一些选项可让用户归档日志，然后以 TTL 方式或基于大小限制延迟删除它们。

选项包括 `Options::WAL_ttl_seconds` 和 `Options::WAL_size_limit_MB` 。它们的使用方法如下：

- 如果都设置为 0，日志将被尽快删除并且永远不会进行归档
- 如果 `WAL_ttl_seconds` 为 0 且 `WAL_size_limit_MB` 不为 0，则每 10 分钟检查一次 WAL 文件，如果总大小大于 `WAL_size_limit_MB`，则从最早的文件开始删除，直到满足 `size_limit` 为止，所有空文件都将被删除
- 如果 `WAL_ttl_seconds` 不为 0 且 `WAL_size_limit_MB` 为 0，则每 `WAL_ttl_seconds / 2` 检查一次 WAL 文件，并且将删除早于 `WAL_ttl_seconds` 的文件
- 如果两者都不为 0，则每 10 分钟检查一次 WAL 文件，并且两次检查都将以 ttl 为优先进行

## 其他信息

设置 RocksDB 选项：

- [[Setup-Options-and-Basic-Tuning]]
- 一些详细的 [[Tuning-Guide|调优指南]]

关于 RocksDB 实现的详细信息可以在以下文档中找到：

- [[Overview]]
- [[Block-based-Table-Format]]
- [[PlainTable-Format]]
- [[Write-Ahead-Log-File-Format]]
