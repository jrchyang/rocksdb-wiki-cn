我们详细介绍了编译 RocksDB 和 RocksJava 以及运行通过 Maven Central 提供的 RocksJava 二进制文件的最低要求。

## 编译

### 所有平台

- Java：[OpenJDK 1.7+](https://adoptopenjdk.net/) (required only for RocksJava)
- Tools：
	- curl (recommended; required only for RocksJava)
- Libraries:
	- [gflags](https://gflags.github.io/gflags/) 2.0+ (required for testing and benchmark code)
	- [zlib](http://www.zlib.net/) 1.2.8+ (optional)
	- [bzip2](http://www.bzip.org/) 1.0.6+ (optional)
	- [lz4](https://github.com/lz4/lz4) r131+ (optional)
	- [snappy](http://google.github.io/snappy/) 1.1.3+ (optional)
	- [zstandard](http://www.zstd.net/) 0.5.1+ (optional)

### Linux

- Architecture: x86 / x86_64 / arm64 / ppc64le / s390x
- C/C++ Compiler: GCC 4.8+ or Clang
- Tools:
	- GNU Make or [CMake](https://cmake.org/download/) 3.14.5+

### macOS

- Architecture: x86_64 / arm64 (Apple Silicon)
- OS: macOS 10.12+
- C/C++ Compiler: Apple XCode Clang
- Tools:
	- GNU Make or [CMake](https://cmake.org/download/) 3.14.5+

### Windows

- Architecture: x86_64
- OS: Windows 7+
- C/C++ Compiler: Microsoft Visual Studio 2015+
- Java: [OpenJDK 1.7+](https://java.oracle.com/) (required only for RocksJava)
- Tools:
	- [CMake](https://cmake.org/download/) 3.14.5+

## RocksJava Binaries

从 [Maven Central](https://search.maven.org/search?q=g:org.rocksdb%20a:rocksdbjni) 运行 RocksJava 官方二进制文件的最低要求。

在所有平台上，二进制文件的本地组件都是静态链接的，因此所需的资源很少。Java 组件需要 [OpenJDK 1.7+](https://adoptopenjdk.net/) 以上。

二进制文件是在真实硬件上使用 Docker 容器构建的，你可以在这里找到我们的 Docker 构建容器：[https://github.com/evolvedbinary/docker-rocksjava](https://github.com/evolvedbinary/docker-rocksjava)

## Linux

对于 Linux，我们提供基于 GNU Lib C 或 Musl 平台（自 RocksJava 6.5.2 起）的二进制文件。

| Architecture | glibc version (minimum) | muslc version (minimum) |
| ------------ | ----------------------- | ----------------------- |
| x86          | 2.12                    | 1.1.16                  |
| x86_64       | 2.12                    | 1.1.16                  |
| aarch64      | 2.17                    | 1.1.16                  |
| ppc64le      | 2.17                    | 1.1.16                  |
| s390x (z10)  | 2.17                    | 1.1.16                  |

## macOS

- Architecture: x86_64
- OS: macOS 10.12+

## Windows

Windows 二进制文件是在 Windows Server 2012 上使用 Visual Studio 2015 构建的。

- Architecture: x86_64
- OS: Windows 7+
- libc: Microsoft Visual C++ 2015 Redistributable (x64) - 14.0.24215
