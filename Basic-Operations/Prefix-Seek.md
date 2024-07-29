# 前缀搜索

## 为什么需要前缀搜索

通常，当执行迭代器查找（seek）操作时，RocksDB 需要将每个有序运行（内存表、0 层文件、其他层级等）的位置放置在查找位置，并合并这些有序运行。这有时会涉及多个 I/O 请求。对于多个数据块的解压缩以及其他 CPU 开销来说，这也是非常耗费 CPU 的。

前缀查找是一种用于减轻某些用例的这些开销的功能。基本思想是，如果用户知道迭代将在一个键前缀内，则可以使用公共前缀来降低成本。最常用的前缀迭代技术是前缀布隆过滤器。如果许多有序运行不包含此前缀的任何条目，则可以通过布隆过滤器将其过滤掉，并且可以忽略有序运行的一些 I/O 和 CPU。

前缀查找的一个典型场景是表示多映射，例如 [MyRocks 中的二级索引](https://github.com/facebook/mysql-5.6/wiki/MyRocks-record-format#secondary-index-c) ，其中 RocksDB 前缀是多映射的键，RocksDB 迭代器会查找与该前缀（多映射键）相关的所有条目。

前缀查找是否可以提高性能取决于工作负载。它对非常短的迭代器查询比对较长的迭代器查询更有效。

## 定义一个“前缀”

“前缀”由 `options.prefix_extractor` 定义，它是 `SliceTransform` 实例的共享指针。通过对某个键调用 `SliceTransform.Transform()` ，我们可以提取表示该 `Slice` 子字符串的 `Slice`，通常是前缀部分。在此 wiki 页面中，我们使用“前缀”来指代某个键的 `options.prefix_extractor.Transform()` 的输出。您可以通过调用 `NewFixedPrefixTransform(prefix_len)` 使用固定长度前缀转换器，通过调用 `NewCappedPrefixTransform(prefix_len)` 使用上限长度前缀转换器，或者您可以按照自己想要的方式实现自己的前缀转换器并将其传递给 `options.prefix_extractor` 。前缀提取器与比较器相关，前缀提取器需要与比较器一起使用，其中具有相同前缀的键在比较器定义的全序中彼此接近。

建议尽可能使用带有逐字比较器或反向逐字比较器的 `CappedPrefixTransform` ，这将以相对较好的性能最大化所支持的功能。

## 配置前缀布隆过滤器

虽然这不是用户利用前缀迭代的唯一方法，但在基于块的 SST 文件中使用前缀布隆过滤器是最常用、最有效的方法。下面举例说明如何在 SST 文件中设置前缀迭代过滤器：

```C++
Options options;

// Set up bloom filter
rocksdb::BlockBasedTableOptions table_options;
table_options.filter_policy.reset(rocksdb::NewBloomFilterPolicy(10, false));
// If you also need Get() to use whole key filters, leave it to true.
table_options.whole_key_filtering = false;
// For multiple column family setting, set up specific column family's ColumnFamilyOptions.table_factory instead.
options.table_factory.reset(rocksdb::NewBlockBasedTableFactory(table_options));  

// Define a prefix. In this way, a fixed length prefix extractor. A recommended one to use.
options.prefix_extractor.reset(NewCappedPrefixTransform(3));

DB* db;
Status s = DB::Open(options, "/tmp/rocksdb",  &db);
```

如何使用布隆过滤器取决于读取查询中的 `ReadOptions` 设置，详见下文。

`options.prefix_extractor` 可以在数据库重启时更改。通常，最终结果是现有 SST 文件中的布隆过滤器将在读取时被忽略。在某些情况下，旧的布隆过滤器仍然可以使用。下文将对此进行说明。

## 在设置前缀布隆过滤器时配置读数

### 如何在读取时忽略前缀布隆过滤器

用户只能使用前缀布隆过滤器来读取特定前缀的内容。如果要查找的内容位于前缀之外，则需要对特定迭代器禁用该功能以防止错误结果：

```C++
ReadOptions read_options;
read_options.total_order_seek = true;
Iterator* iter = db->NewIterator(read_options);
Slice key = "foobar";
iter->Seek(key);  // Seek "foobar" in total order
```

设置 `read_options.total_order_seek = true` 将确保查询返回的结果与没有前缀布隆过滤器时相同。

### 自适应前缀模式

从 6.8 版本开始，引入了新的自适应模式：

```C++
ReadOptions read_options;
read_options.auto_prefix_mode = true;
Iterator* iter = db->NewIterator(read_options);
// ......
```

这将始终生成与 `read_options.total_order_seek = true` 相同的结果，而根据查找键和迭代器上限，可能会使用前缀布隆过滤器。这是使用前缀布隆过滤器的推荐方式，因为它不易被误用。一旦证明该选项稳定，我们将默认启用该选项。

以下是如何使用此功能的示例：对于 3 的固定长度前缀提取器，以下查询将利用布隆过滤器：

```C++
options.prefix_extractor.reset(NewCappedPrefixTransform(3));
options.comparator = BytewiseComparator();  // This is the default
// ......
ReadOptions read_options;
read_options.auto_prefix_mode = true;
std::string upper_bound;
Slice upper_bound_slice;

// "foo2" and "foo9" share the same prefix "foo".
upper_bound = "foo9";
upper_bound_slice = Slice(upper_bound);
read_options.iterate_upper_bound = &upper_bound_slice;
Iterator* iter = db->NewIterator(read_options);
iter->Seek("foo2");

// "foobar2" and "foobar9" share longer prefix than "foo".
upper_bound = "foobar9";
upper_bound_slice = Slice(upper_bound);
read_options.iterate_upper_bound = &upper_bound_slice;
Iterator* iter = db->NewIterator(read_options);
iter->Seek("foobar2");

// "foo2" and "fop" doesn't share the same prefix,
// but "fop" is the successor key of prefix "foo" which is the prefix of "foo2".
upper_bound = "fop";
upper_bound_slice = Slice(upper_bound);
read_options.iterate_upper_bound = &upper_bound_slice;
Iterator* iter = db->NewIterator(read_options);
iter->Seek("foo2");
```

该功能有一定的局限性：

- 只支持 `FixedPrefixTransform` 和 `CappedPrefixTransform`
- 比较器需要实现 `IsSameLengthImmediateSuccessor()` 功能，内置的字节比较器和反向字节比较器已经实现了这一功能
- 目前，只有 `Seek()` 可以通过将查找键与迭代器上限进行比较来自动利用该功能。`SeekForPrev()` 从不使用前缀布隆过滤器 （没有根本原因不能实现，欢迎您实现）
- 检查布隆过滤器有一定额外的 CPU 开销

### 手动前缀迭代

有了这个选项，用户就可以确定自己的用例是否符合前缀迭代器的使用条件，并负责永远不在迭代器之外进行迭代。当前缀迭代器被滥用时，RocksDB 不会返回错误信息，而迭代结果将是未定义的。需要注意的是，在迭代器范围之外迭代时，结果可能不仅限于缺少键。一些未定义的结果可能包括：已删除的键可能会显示出来、未遵循键排序或查询速度非常慢。还要注意的是，如果迭代器移出前缀范围后又返回，即使是前缀范围内的数据也可能不正确。

目前，出于向后兼容的原因，该模式为默认模式，但用户在使用该模式时应格外小心。

如何使用手动模式：

```C++
options.prefix_extractor.reset(NewCappedPrefixTransform(3));

ReadOptions read_options;
read_options.total_order_seek = false;
read_options.auto_prefix_mode = false;
Iterator* iter = db->NewIterator(read_options);

iter->Seek("foobar");
// Iterate within prefix "foo"

iter->SeekForPrev("foobar");
// iterate within prefix "foo"
```

### 更改前缀提取器

如果前缀提取器在数据库重启时发生变化，一些 SST 文件可能会包含使用与当前选项中不同的前缀提取器生成的前缀布隆过滤器。打开这些文件时，RocksDB 会比较存储在 SST 文件属性中的前缀提取器名称。如果前缀提取器的名称与选项中提供的前缀提取器不同，那么该文件将不会使用前缀盛放过滤器，但以下情况除外：如果之前 SST 文件使用的是 `FixedPrefixTransform` 或 `FixedPrefixTransform` ，那么如果查找键和上限表明迭代器在前一个前缀提取器指定的前缀范围内，则该过滤器有时可能会被使用。SST 文件中的行为与自动前缀模式相同。

## 其他前缀迭代特性

除了前缀布隆过滤器，有以下几个前缀迭代功能：

- 在 memtable 中使用前缀布隆过滤器，通过 `options.memtable_prefix_bloom_size_ratio` 打开它
- 前缀 memtable，支持 hash linked list 和 hash skip list 两种类型的 memtable，详见：[[MemTable]]
- block based table format 的前缀散列索引，详见：[[Data-Block-Hash-Index]]
- PlainTable forma，详见：[[PlainTable-Format]]

## 通用前缀查找 API

上面我们详细介绍了如何使用前缀布隆过滤器，其他前缀迭代功能的用法几乎相同，唯一的区别是，某些选项可能不受支持。以下是一般工作流程：

当指定了数据库或列族的 `options.prefix_extractor` 时，RocksDB 就会进入 "前缀查找" 模式。使用示例如下：

```C++
Options options;

// <---- Enable some features supporting prefix extraction
options.prefix_extractor.reset(NewFixedPrefixTransform(3));

DB* db;
Status s = DB::Open(options, "/tmp/rocksdb",  &db);

//......

Iterator* iter = db->NewIterator(ReadOptions());
iter->Seek("foobar"); // Seek inside prefix "foo"
iter->Next(); // Find next key-value pair inside prefix "foo"
```

当 `options.prefix_extractor` 不是 `nullptr` 并且使用默认的 `ReadOptions` 时，迭代器不能保证所有键都是按顺序迭代的，只能保证相同前缀的键是按顺序迭代的。在执行 `Iterator.Seek(lookup_key)` 时，RocksDB 会从 `lookup_key` 中提取前缀。如果数据库中有一个或多个键与 `lookup_key` 的前缀相匹配，RocksDB 就会像全排序模式一样，把迭代器放到与 `lookup_key` 前缀相同或大于 `lookup_key` 的键上。如果没有前缀等于或大于 `lookup_key` 的前缀，或者在调用了一次或多次 `Next()` 之后，我们处理完了相同前缀的所有键，我们可能会返回 `Valid()=false`，或者任何大于前一个键的键（可能是不存在的）。设置 `ReadOptions.prefix_same_as_start=true` 可以保证上述两种行为中的第一种。

从 4.11 版开始，我们支持前缀模式下的 `Prev()`，但仅限于迭代器仍在该前缀的所有键的范围内时。当迭代器超出范围时，`Prev()` 的输出不保证正确。

启用前缀查找模式后，RocksDB 将自由组织数据或建立查找数据结构，以便快速定位特定前缀的键或排除不存在的前缀。以下是前缀查找模式支持的一些优化：基于块的表和 memtable 的 prefix bloom、基于哈希的 memtable 以及 [[PlainTable-Format|PlainTable 格式]] 。设置示例：

```C++
Options options;

// Enable prefix bloom for mem tables
options.prefix_extractor.reset(NewFixedPrefixTransform(3));
options.memtable_prefix_bloom_size_ratio = 0.1;

// Enable prefix hash for SST files
BlockBasedTableOptions table_options;
table_options.index_type = BlockBasedTableOptions::IndexType::kHashSearch;

DB* db;
Status s = DB::Open(options, "/tmp/rocksdb",  &db);

// ......

auto iter = db->NewIterator(ReadOptions());
iter->Seek("foobar"); // Seek inside prefix "foo"
```

从 3.5 版开始，我们支持一个读取选项，使得 RocksDB 即使给定了 `options.prefix_extractor `（前缀模式）也允许 RocksDB 使用全顺序。要启用该功能，需要在执行 `NewIterator()` 时将 `ReadOption.total_order_seek=true` 设置为读取选项，例如：

```C++
ReadOptions read_options;
read_options.total_order_seek = true;
auto iter = db->NewIterator(read_options);
Slice key = "foobar";
iter->Seek(key);  // Seek "foobar" in total order
```

在这种模式下，性能可能会变差。请注意，并非所有前缀查找的实现都支持该选项。例如，[[PlainTable-Format|PlainTable]] 的某些实现并不支持该选项，当你尝试使用它时，会在状态代码中看到一个错误。如果使用它，基于哈希值的内存表可能会进行非常昂贵的在线排序。基于块的表的前缀 bloom 和哈希索引支持这种模式。

从 6.8 版开始，我们支持类似的选项，同时允许底层实施在不影响结果的情况下利用前缀特定信息。

```C++
ReadOptions read_options;
read_options.auto_prefix_mode = true;
auto iter = db->NewIterator(read_options);
Slice key = "foobar";
iter->Seek(key);  // Seek "foobar" in total order
```

目前该功能仅支持使用前缀布隆过滤器的前向查找。

## 局限

`SeekToLast()` 不能很好地支持前缀迭代。`SeekToFirst()` 仅受某些配置支持。如果您要针对迭代器执行这些类型的查询，则应使用全序模式。

使用前缀迭代的一个常见错误是使用前缀模式以反向顺序迭代，但目前尚不支持。如果反向迭代是您的常见查询模式，您可以重新排序数据以将迭代顺序改为正向。您可以通过实现自定义比较器来实现，或者以不同的方式对 key 进行编码。

## API 变化（2.8 -> 3.0）

在本节中，我们将解释 2.8 版本中的应用程序接口以及 3.0 中的变化。

### 变化之前

截至 RocksDB 2.8，共有 3 种查找模式：

#### 全序查找

这就是你所期望的传统查找行为。查找在整个有序键空间上执行，将迭代器定位到大于或等于你所查找的目标键的键上。

```C++
auto iter = db->NewIterator(ReadOptions());
Slice key = "foo_bar";
iter->Seek(key);
```

并非所有表格格式都支持全序查找。例如，新引入的 [[PlainTable-Format|PlainTable]] 格式只支持基于前缀的 `seek()`，除非以总序模式打开（`Options.prefix_extractor==nullptr`）。

#### 使用 ReadOptions.prefix

这是最不灵活的查找方式，创建迭代器时需要提供前缀。

```C++
Slice prefix = "foo";
ReadOptions ro;
ro.prefix = &prefix;
auto iter = db->NewIterator(ro);
Slice key = "foo_bar"
iter->Seek(key);
```

`Options.prefix_extractor `是前提条件。`Seek()` 受限于 `ReadOptions` 提供的前缀，这意味着你需要创建一个新的迭代器来查找不同的前缀。这种方法的好处是，在创建新迭代器时，无关文件会被过滤掉。因此，如果要查找具有相同前缀的多个键，它的性能可能会更好。不过，我们认为这种情况很少见。

#### 使用 ReadOptions.prefix_seek

这种模式比 `ReadOption.prefix` 更灵活。在创建迭代器时不进行预过滤，因此，同一个迭代器可以重复用于不同键/前缀的搜索。

```C++
ReadOptions ro;
ro.prefix_seek = true;
auto iter = db->NewIterator(ro);
Slice key = "foo_bar";
iter->Seek(key);
```

与 `ReadOptions.prefix` 相同，`Options.prefix_extractor` 是前提条件。

### 变化

很明显，3 种查找模式会造成混淆：

- 一种模式需要设置另一个选项（如 `Options.prefix_extractor`）
- 对于我们的用户来说，后两种模式在不同情况下该使用哪种并不明显

这一变更试图解决这一问题，并使事情变得简单明了：默认情况下，如果定义了 `Options.prefix_extractor`，`Seek()` 将在前缀模式下执行，反之亦然。这样做的动机很简单：如果提供了 `Options.prefix_extractor`，就意味着底层数据可以被分片，而前缀查找是一个非常适合的信号。使用方法变得统一：

```C++
auto iter = db->NewIterator(ReadOptions());
Slice key = "foo_bar";
iter->Seek(key);
```

### 迁移到新用法

过渡到新用法应该很简单：移除对 `Options.prefix` 或 `Options.prefix_seek` 的赋值，因为它们已被弃用。现在，直接使用目标键或前缀进行查找。由于 `Next()` 可以跨越边界进入不同的前缀，因此需要检查结束条件：

```C++
auto iter = DB::NewIterator(ReadOptions());
for (iter.Seek(prefix); iter.Valid() && iter.key().starts_with(prefix); iter.Next()) {
   // do something
}
```
