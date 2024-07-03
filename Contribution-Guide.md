## 贡献方式

除了修改代码之外，还有很多其他方式可以为 RocksDB 社区做出贡献，例如

- 在 [user group](https://groups.google.com/g/rocksdb)、[stackoverflow](https://stackoverflow.com/questions/tagged/rocksdb) 和 [facebook group](https://www.facebook.com/groups/rocksdb.dev/) 上回答问题
- 报告问题并提供详细信息，如有可能，附上测试结果
- 调查并修复问题；如果您能帮助重现问题，我们将不胜感激，这通常是最难的部分。如果您正在寻找一个开放的问题来解决，这里是 [待解决问题](https://github.com/facebook/rocksdb/issues?q=is%3Aissue+is%3Aopen+sort%3Aupdated-desc+label%3Aup-for-grabs+no%3Aassignee) 列表
- 审核和测试 PR，我们有一些 PR 尚待审核，您的审核或测试可以减轻我们的审核负担
- 提交新功能请求或实现，我们欢迎任何功能建议，如果要实现新功能，请先创建一个设计提案，这有助于在审核您的实施之前获得更多的参与和讨论
- 帮助正在进行开发的功能，以下是正在积极开发的 [功能](https://github.com/facebook/rocksdb/wiki/Projects-Being-Developed) 列表，请帮助我们完成开发

注意：修复单个明确的错字不会被接受为贡献。澄清 API 注释或尝试修复版本库中所有错字的 PR 可以接受。

## 代码贡献前

在为 RocksDB 做出贡献之前，请确保您能够签署 CLA。除非您签署了适当的 CLA，否则您的修改将不会被合并。更多信息请参见 https://code.facebook.com/cla

## 基本开发工作流程

与 github 上的大多数开源项目一样，RocksDB 的贡献者们在自己的 fork 上工作，并向 RocksDB 的 facebook repo 发送拉取请求。在审核员批准拉取请求后，Facebook 的 RocksDB 团队成员就会将其合并。

## 如何运行单元测试

### 构建系统

RocksDB 使用 gtest。GNU make 使用的 makefile 有一些支持功能，可以帮助开发者并行运行所有单元测试，下文将对此进行介绍。如果使用 cmake，则可以使用 ctest 运行测试。

### 并行运行单元测试

要并行运行单元测试，首先要在主机上安装 GNU parallel，然后运行：

```Shell
make all check [-j] 
```

例如，您可以使用环境变量 `J=1` 来指定要运行的并行测试次数：

```Shell
make J=64 all check [-j]
```

如果在 release 和 debug 版本、正常或精简版本、编译器或编译器选项之间切换，请先调用 make clean。因此，这里有一个运行所有测试的安全例程：

```Shell
make clean
make J=64 all check [-j]
```

### 调试单个单元测试失败

RocksDB 使用 gtest。你可以通过运行测试二进制文件来运行特定的单元测试。如果你使用 GNU make，测试二进制文件就会在你的检查点之下。例如，测试 DBBasicTest.OpenWhenOpen 就在二进制 db_basic_test 中，因此只需运行：

```Shell
./db_basic_test
```

将运行二进制文件中的所有测试。

gtest 提供了一些有用的命令行参数，调用 `--help` 可以查看这些参数：

```Shell
./db_basic_test --help
```

下面是一些经常使用的例子：

使用 `--gtest_filter` 运行测试子集。如果只想运行 `DBBasicTest.OpenWhenOpen`，请调用：

```Shell
./db_basic_test --gtest_filter=“*DBBasicTest.OpenWhenOpen*”
```

默认情况下，即使测试失败，测试创建的测试数据库也会被清除。你可以尝试使用 `--gtest_throw_on_failure` 来保留它。如果想在断言失败时停止调试器，请指定 `--gtest_break_on_failure`。`KEEP_DB=1` 环境变量是另一种保存测试数据的方法，无论测试是否失败，都不会在单元测试运行结束时删除测试数据：

```Shell
KEEP_DB=1 ./db_basic_test --gtest_filter=DBBasicTest.Open
```

默认情况下，临时测试文件位于 `/tmp/rocksdbtest-<number>/` 目录下（除非并行运行时位于 /dev/shm）。您可以使用环境变量 `TEST_TMPDIR` 来更改位置。例如：

```Shell
TEST_TMPDIR=/dev/shm/my_dir ./db_basic_test
```

### Java 单元测试

有时我们也需要运行 Java 测试。运行：

```Shell
make jclean rocksdbjava jtest
```

您可以添加 `-j`，但有时会导致问题。如果发现问题，请尝试删除 `-j`。

### 其他一些构建风格

对于更复杂的代码修改，我们会要求贡献者在提交代码审查之前运行更多的构建版本。GNU make 的 makefile 对此有更好的预定义支持，不过也可以在 CMake 中手动完成。

要使用 AddressSanitizer (ASAN)，请设置环境变量 `COMPILE_WITH_ASAN`：

```Shell
COMPILE_WITH_ASAN=1 make all check -j
```

要使用 ThreadSanitizer (TSAN)，请设置环境变量 `COMPILE_WITH_TSAN`：

```Shell
COMPILE_WITH_TSAN=1 make all check -j
```

运行所有 `valgrind` 测试：

```Shell
make valgrind_test -j
```

要运行 \_UndefinedBehaviorSanitizer (UBSAN)，请设置环境变量 `COMPILE_WITH_UBSAN`：

```Shell
COMPILE_WITH_UBSAN=1 make all check -j
```

要运行 `llvm` 分析器，请运行：

```Shell
make analyze
```

## Folly 集成

RocksDB 支持与 Folly 集成，以提供某些功能，例如使用 `folly::DistributedMutex` 实现更好的 `LRUCache` 锁定，使用 `folly::F14FastMap` 实现更快的哈希表，使用 folly coroutines 实现 `MultiGet` 中的异步 IO 等。目前这只是实验性的，可以通过以下方式启用：

### 安装依赖项

这份清单可能并不完整，但在 Ubuntu 上我做到了：

```Shell
sudo apt-get install openssl libssl-dev autoconf
```

### 检出并构建 folly

```Shell
make checkout_folly
make build_folly
```

### 启用 folly 集成构建 RocksDB

Folly 集成有多种类型。coroutines 功能需要 gcc 10 或更新版本和 cmake。下面的 cmake 命令行用于优化编译。如果想要调试编译，请搜索 cmake 文档了解 CMAKE_BUILD_TYPE。

构建时完全集成了 Folly，包括例行程序，但没有压缩库：

```Shell
CC=gcc-10 CXX=g++-10 mkdir -p build && cd build && cmake -DCMAKE_BUILD_TYPE=RelWithDebInfo -DUSE_COROUTINES=1 -DWITH_GFLAGS=1 -DROCKSDB_BUILD_SHARED=0 .. && make -j
```

以完全集成的方式编译，包括例程、lz4 和 zstd 压缩库，但不设置 CC 或 CXX 路径。这只会构建 db_bench 二进制文件，以避免编译所有二进制文件所需的时间和空间。

```Shell
mkdir -p build && cd build && cmake -DCMAKE_BUILD_TYPE=RelWithDebInfo -DWITH_LZ4=1 -DWITH_ZSTD=1 -DUSE_COROUTINES=1 -DWITH_GFLAGS=1 -DROCKSDB_BUILD_SHARED=0 .. && make -j db_bench
```

使用部分 Folly 集成进行构建（无 coroutines）（需要 gcc 7 或更新版本）

```Shell
USE_FOLLY=1 make -j
```

你可以在 RocksDB LOG 中找到这个，以确认是否支持 Folly

```Shell
DMutex implementation: folly::DistributedMutex
```

## 代码风格

RocksDB 遵循 [Google C++ 风格](https://google.github.io/styleguide/cppguide.html)， 注意：在现有的 RocksDB 代码中，一个常见的模式是在 [the old Google C++ Style](https://stackoverflow.com/questions/26441220/googles-style-guide-about-input-output-parameters-as-pointers) 中为输出参数使用不可为空的 Type*，但这一指导原则 [has changed](https://github.com/google/styleguide/commit/7a7a2f510efe7d7fc#diff-bcadcf8be931ffdd5d6a65c60c266039cf1f96b7f35bfb772662db811214c5a0R1713)。[The new guideline](https://google.github.io/styleguide/cppguide.html#Inputs_and_Outputs) 更倾向于使用（非 const）引用作为输出参数。

在格式化方面，我们将每行的字符数限制为 80 个字符。通过运行：

```Shell
build_tools/format-diff.sh
```

如果使用 GNU make，则只需 make 格式即可。如果缺少运行所需的依赖项，脚本会打印出安装说明。

## 发送拉取请求前的要求

### HISTORY.md

考虑更新 HISTORY.md，以提及您的更改，特别是如果它是一个错误修复、公共 API 更改或一个很棒的新功能。

### PR 摘要

我们建议在 PR 摘要中加入 "测试计划：" 部分，介绍为验证变更的质量和性能而进行的测试。

### 添加单元测试

几乎所有的代码变更都需要与单元测试的变更一起进行验证。对于新功能，即使已通过手动验证，也需要添加新的单元测试或测试场景。这样做是为了确保未来的贡献者可以重新运行测试，以验证他们的更改不会给功能带来问题。

### 简单变更

在运行了所有单元测试后，如果发现所有测试都通过了，就可以发送简单变更的 PR。如果更改了任何公共接口，或涉及 Java 代码，也需要运行 Java 测试。

### 复杂变更

如果变更足够复杂，在发送拉取请求之前，需要在本地环境中运行 ASAN、TSAN 和 valgrind。如果您在运行 ASAN 时使用了更高版本的 llvm（几乎涵盖了 valgrind 的所有功能），则可以跳过 valgrind 测试。这对使用 Windows 的开发人员来说可能有些困难。请尽量使用您环境中可用的最佳等价工具。

### 风险较高或存在一些未知因素的变化

对于风险较高的变更，除了用多种方法运行所有测试外，还需要执行崩溃测试循环（见 [压力测试](https://github.com/facebook/rocksdb/wiki/Stress-test)），并确保不会出现故障。如果崩溃测试未涵盖新功能，可考虑将其添加到崩溃测试中。要运行所有崩溃测试，请运行：

```Shell
make crash_test -j
make crash_test_with_atomic_flush -j
```

如果无法使用 GNU make，可以手动构建 db_stress 二进制文件，然后运行脚本：

```Shell
python -u tools/db_crashtest.py whitebox
python -u tools/db_crashtest.py blackbox
python -u tools/db_crashtest.py --simple whitebox
python -u tools/db_crashtest.py --simple blackbox
python -u tools/db_crashtest.py --cf_consistency blackbox
python -u tools/db_crashtest.py --cf_consistency whitebox 
```

### 性能优化变更

对于可能影响性能的变更，我们建议运行正常基准，以确保不会出现性能下降。根据实际性能，您可以选择运行磁盘支持的数据库或内存支持的文件系统。如果不明显，请在拉取请求摘要中解释选择性能环境的原因。如果更改是为了提高性能，请至少提供一个有利于提高性能的基准测试用例，并展示提高的效果。
