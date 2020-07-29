---
title: 'MySQL 5.7 编译调试指南(Ubuntu 16.04 + MacOS 10.15)'
date: 2020-07-30 00:44:37
tags: [MySQL]
published: true
hideInList: false
feature: 
isTop: false
---
本文主要介绍在`MacOS 10.15`和`Ubuntu 16.04`系统下编译构建`MySQL 5.7.30`并使用`JetBrains CLion`(以下简称`CLion`)进行运行调试的方法。
<!-- more -->
  
# 下载源码
可以从两种方式下载`MySQL`源代码：
## 从官方代码库下载最新源代码
```bash
# clone过程可能耗时比较长
git clone https://github.com/mysql/mysql-server.git
# 可以切换至指定版本对应分支
git checkout 5.7
```
## 下载源代码分发包
`MySQL`提供了各个版本的源码分发包供下载。下载地址：[MySQL Product Archives](https://downloads.mysql.com/archives/community/)，选择指定版本下载解压即可。

# 安装依赖
对于各个依赖或者工具，安装前可以先验证一下是否已经安装，一些工具系统已经预装。
## 1. 构建工具 CMake + make
在`Ubuntu`上：
```bash
sudo apt install cmake make
```
在`MacOS`上：
```bash
brew install cmake make
```
[MySQL 5.7 Reference Manual](https://dev.mysql.com/doc/refman/5.7/en/source-installation-prerequisites.html)建议的`make`为`GNU make 3.75`或者更高版本。

## 2. 编译工具 GCC/Clang
在`Ubuntu`上，`MySQL`可以使用`GCC`或者`Clang`来编译，这里我们使用`apt`来安装`GCC`：
```bash
sudo apt install gcc
```
在`MacOS`上，`MySQL`会使用`Clang`编译器来编译，一般`MacOS`系统已经安装好了`Clang`，如果没有的的话可以通过安装`Xcode`来方便地安装`Clang`。可以在`App Store`中下载完整`Xcode`，或者使用如下命令来安装`Xcode command line tools`：
```bash
xcode-select --install
```
根据[MySQL 5.7 Reference Manual](https://dev.mysql.com/doc/refman/5.7/en/source-configuration-options.html#option_cmake_force_unsupported_compiler)，对于`MySQL 5.7`，支持的`GCC`最低版本是`4.4`，支持的`Clang`最低版本是`3.3`。

## 3. OpenSSL
在`Ubuntu`下：
```bash
sudo apt install openssl libssl-dev
```
在`MacOS`下：
```bash
brew install openssl
```
根据[MySQL 5.7 Reference Manual](https://dev.mysql.com/doc/refman/5.7/en/source-ssl-library-configuration.html)，对于`MySQL 5.7`，支持的最小`OpenSSL`版本是`1.0.1`。

## 4. Boost
根据[MySQL 5.7 Reference Manual](https://dev.mysql.com/doc/refman/5.7/en/source-installation-prerequisites.html)，对于`MySQL 5.7`，必须要使用`1.59.0`版本。在[Boost Version History](https://www.boost.org/users/history/)下载`Boost 1.59.0`并解压。
> 因为`1.59.0`是比较老的版本，所以并不推荐直接使用包管理器在全局安装一个旧版本的`Boost`，所以我们这里采用下载后在构建时指定其路径的方法。

## 5. ncurses
在`Ubuntu`下：
```bash
sudo apt-get install libncurses5-dev
```
在`MacOS`下：
```bash
brew install ncurses
```
## 6. Bison
在`Ubuntu`下：
```bash
sudo apt-get install bison
```
在`MacOS`下：
```bash
brew install bison
```
根据[MySQL 5.7 Reference Manual](https://dev.mysql.com/doc/refman/5.7/en/source-installation-prerequisites.html)，对于`MySQL 5.7`，支持的`bison`最低版本是`2.1`，


# 编译构建
为了不影响机器上原本已经安装的`MySQL`，在配置构建过程中，我们自定义了一些配置避免与机器上另外的`MySQL`实例冲突。
```bash
# 进入源码目录
cd mysql-server
# 创建构建文件的目录及数据目录、临时文件目录
mkdir -p bld/data bld/tmp
# 配置，"/home/jiajiawang/workspace/mysql-server/"为源码目录的绝对路径
cmake -DCMAKE_BUILD_TYPE=Debug -DCMAKE_INSTALL_PREFIX=/home/jiajiawang/workspace/mysql-server/bld -DMYSQL_DATADIR=/home/jiajiawang/workspace/mysql-server/bld/data  -DMYSQL_UNIX_ADDR=/home/jiajiawang/workspace/mysql-server/bld/data/mysql.sock -DTMPDIR=/home/jiajiawang/workspace/mysql-server/bld/tmp/ -DMYSQL_TCP_PORT=3336 -DWITH_BOOST=/home/jiajiawang/software/boost_1_59_0

# 构建
make
# 安装MySQL
make install
```
上面配置的目录及文件位置最好都配置成**绝对路径**。

`CMake`选项说明：

| 参数 | 含义 | 默认值 |
| --- | --- | --- |
| CMAKE_BUILD_TYPE | 生成的构建类型，可选值有：`RelWithDebInfo`，`Debug` ，使用`Debug`可以禁用优化，更有助于调试 | `RelWithDebInfo` |
| CMAKE_INSTALL_PREFIX |  指定`MySQL`安装路径 | `/usr/local/mysql` |
| MYSQL_DATADIR | 指定`MySQL`数据目录 |  |
| MYSQL_UNIX_ADDR | 指定`Unix socket`文件目录 | `/tmp/mysql.sock` |
| TMPDIR | 指定临时文件目录 |  |
| MYSQL_TCP_PORT | `MySQL`启动`TCP`端口号 | `3306` |
| WITH_BOOST | 指定`Boost`依赖的路径 |  |

更多选项说明请参考：[MySQL 5.7 Reference Manual](https://dev.mysql.com/doc/refman/5.7/en/source-configuration-options.html)

# 配置并启动MySQL
## 1. 数据库初始化
```bash
cd bld
```
使用如下命令可以用来初始化数据库。两个参数的区别是`--initialize`会为`root@localhost`用户生成一个随机密码，而`--initialize-insecure`会设置`root@localhost`用户密码为空：
```bash
bin/mysqld --initialize
# 或者
bin/mysqld --initialize-insecure
```
创建加密连接需要的文件：
```bash
bin/mysql_ssl_rsa_setup
```
## 2. 启动 MySQL Server
启动`MySQL Server`：
```bash
bin/mysqld
```
关于其他启动方式，参考：[MySQL 5.7 Reference Manual](https://dev.mysql.com/doc/refman/5.7/en/programs-server.html)
## 3. 使用 MySQL Client 连接 Server
使用`MySQL Client`连接`Server`：
```bash
# 如果没有生成密码的话可以不指定-p
bin/mysql -uroot -p
```
使用初始化时生成的随机密码登录后需要修改密码才可以进行其他操作：
```bash
alter user 'root'@'localhost' identified by 'root';
```
## 4. 关闭MySQL Server
使用`ps`命令找到`mysqld`的进程ID，使用`kill -9 <pid>`杀掉进程。

# 导入`CLion`并运行/调试
1. 导入`CLion`
2. 设置`CMake`参数。点击`File`菜单栏，`Settings | Build, Execution, Deployment | CMake`，在`CMake options`输入框中输入上文执行`cmake`命令时的参数。
![](https://keaper-cn-image.oss-cn-beijing.aliyuncs.com/20200726235304.png)
3. 运行/调试`MySQL`。点击`Run`菜单栏，`Edit Configurations`，左侧`CMake Application`列出了`MySQL`中的各个程序，例如`mysqld`为`MySQL Server`，`mysql`为`MySQL CLient`，可以在对应的`Program arguments`中配置各种参数。

# 参考
1. [Installing MySQL from Source](https://dev.mysql.com/doc/refman/5.7/en/source-installation.html)
2. [Postinstallation Setup and Testing](https://dev.mysql.com/doc/refman/5.7/en/postinstallation.html)
3. [Server Command Options](https://dev.mysql.com/doc/refman/5.7/en/server-options.html)
4. [Running Multiple MySQL Instances on One Machine](https://dev.mysql.com/doc/refman/5.7/en/multiple-servers.html)