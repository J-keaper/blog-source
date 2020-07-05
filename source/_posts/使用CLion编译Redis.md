---
title: '使用CLion调试Redis(Ubuntu 16.04 + MacOS 10.15)'
date: 2020-07-05 10:13:56
tags: [Redis]
---
本文主要介绍如何编译`Redis`项目并在`JetBrains CLion`（以下简称`CLion`）中运行/调试。
<!-- more -->
# 0. 开始之前
## JSON Compilation Database
`CLion`默认支持`CMake`构建的项目，但`Redis`项目是基于`Make`构建的。对于使用`Make`构建的项目，`CLion`仍然可以通过`Compilation Database`来导入项目，而不用将其修改为`CMake`项目。同时也能通过`Compilation Database`实现代码的分析、跳转等功能，这对于我们进行代码调试很有帮助。

有关于`Compilation Database`的介绍可以参考这几篇文章：
 - `Clang`官方的`Compilation Database`介绍页面：[JSON Compilation Database Format Specification — Clang 11 documentation](https://clang.llvm.org/docs/JSONCompilationDatabase.html) 
 - 这篇文章介绍了在不同场景下生成`Compilation database`的各种工具：[Compilation database — Sarcasm notebook](https://sarcasm.github.io/notes/dev/compilation-database.html) 
 - `CLion`的帮助页面简要介绍了`Compilation Database`，以及在`CLion`的使用：[Compilation Database - Help | CLion](https://www.jetbrains.com/help/clion/compilation-database.html)

对于基于`Make`构建的`Redis`项目来说，有一些工具可以生成`Compilation Database`，比如下面几个：
- [https://github.com/rizsotto/Bear](https://github.com/rizsotto/Bear)
- [https://github.com/rizsotto/scan-build](https://github.com/rizsotto/scan-build)
- [https://github.com/nickdiego/compiledb](https://github.com/nickdiego/compiledb)

通常默认生成的`Compilation Database`是一个`compile_commands.json`文件。

# 1. 下载Redis源码
```bash
git clone git@github.com:redis-io/redis.git
cd redis
# 可以切换至指定版本对应分支
git checkout 5.0
```

# 2. 编译构建Redis
`Redis`基于`Make`构建，执行`make`命令即可完成构建。`Redis` 默认使用`-O2`级别优化，可以使用`make noopt`来编译以关闭优化，获得更多的调试信息。

---
我们需要使用工具来生成`Compilation Database`，以便于导入`CLion`。在不同系统上各个工具安装情况可能略有不同，下面分别介绍`Ubuntu 16.04`和`MacOS 10.15`上编译`Redis`并生成`Compilation Database`的方法。
## Ubuntu 16.04
### 使用Bear工具
```bash
# 下载Bear工具
sudo apt-get install bear
# 编译构建，并使用Bear工具生成 Compilation Database
bear make noopt
```
### 使用compiledb工具
```bash
# 安装pip
sudo apt-get install python-pip
# pip安装 compiledb
pip install compiledb
# 编译构建，并使用compiledb工具生成 Compilation Database
compiledb make noopt
```

## MacOS 10.15
### 使用Bear工具
在`MacOS`上使用Bear工具需要关闭SIP，关闭方法是进入恢复模式执行`csrutil disable`命令，然后重启，详细操作方法请自行搜索。

我们需要用到`Homebrew`来安装`Bear`，如果没有安装`Homebrew`需要先安装`Homebrew`，安装方法：
```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install.sh)"
```
开始编译`Redis`：
```bash
# 下载Bear工具
brew install bear
# 编译构建，并使用Bear工具生成 Compilation Database
bear make noopt
```
如果遇到`adlist.c:32:10: fatal error: 'stdlib.h' file not found`报错，可以先执行以下命令后重试：
```bash
sudo mount -uw /
sudo cp -R /Library/Developer/CommandLineTools/SDKs/MacOSX.sdk/usr/include /usr
```
### 使用compiledb工具
我们需要使用`pip`来安装`compiledb`，如果没有安装`pip`需要先安装`pip`，安装方法：
```bash
curl https://bootstrap.pypa.io/get-pip.py -o get-pip.py
python get-pip.py
# 安装pip之后可能还需要设置环境变量
```

开始编译`Redis`：
```bash
# pip安装 compiledb
pip install compiledb
# 编译构建，并使用compiledb工具生成 Compilation Database
compiledb make noopt
```
---
编译完成之后可以检验以下是否成功：
```
$ ./src/redis-server --version
Redis server v=5.0.8 sha=1f7d08b7:1 malloc=libc bits=64 build=3530fd9b48a55c7f
```
同时根目录下应该会有一个不为空的`compile_commands.json`文件。

## 导入CLion并调试
对于如何在`CLion`中管理基于`Make`构建的项目，`CLion`的官方帮助文档中有详细的介绍：
[Managing Makefile Projects](https://www.jetbrains.com/help/clion/managing-makefile-projects.html)。参考上述文档，本小节简要介绍了如何在`CLion`中导入`Redis`项目并运行/调试的方法。

1. 下载安装`CLion`，并安装`Makefile Support`插件。建议使用最新版本，较老的版本是不支持`Make`构建的项目的。
2. 导入项目。打开`CLion`，选择`Open Or Import`，选择项目目录中的`compile_commands.json`文件，弹出框选择`Open as Project`，等待文件索引完成。
    ![](https://keaper-cn-image.oss-cn-beijing.aliyuncs.com/20200705183609.gif)
3. 创建自定义`Build Target`。点击`File`菜单栏，`Settings | Build, Execution, Deployment | Custom Build Targets`，点击`+`新建一个`Target`。
	- `Name`：`Target`的名字，之后在创建`Run/Debug`配置的时候会看到这个名字
	- 点击`Build`或者`Clean`右边的三点，弹出框中点击`+`新建两个`External Tool`配置如下：
        第一个配置如下，用来指定构建指令，Program 和 Arguments 共同构成了所要执行的命令 "make noopt"
        ```bash
		Name: make
		Program: make
		Arguments: noopt
		Working directory: $ProjectFileDir$
        ```
		第二个配置如下，用来清理构建输出，Program 和 Arguments 共同构成了所要执行的命令 "make clean"
        ```bash
        Name: make clean
		Program: make
		Arguments: clean
		Working directory: $ProjectFileDir$
		```
	- `ToolChain`选择`Default`；`Build`选择`make`（上面创建的第一个`External Tool`）；`Clean`选择`make clean`（上面创建的第二个`External Tool`）
    ![](https://keaper-cn-image.oss-cn-beijing.aliyuncs.com/20200705183311.gif)
4. 创建自定义的`Run/Debug configuration`。点击`Run`菜单栏,`Edit Configurations`， 点击`+`，选择`Custom Build Application`，配置如下：
    ```bash
	# Name：Configure 的名称
    Name: redis
	# Target：选择上一步创建的 “Custom Build Target”
    Target: redis
	# Executable：程序执行入口，也就是需要调试的程序
	Executable: 这里我们调试Redis Server，选择`{source_root}/src/redis-server`。
	# Program arguments: 与 “Executable” 配合使用，指定其参数
	Program arguments: 这里我们选择"$ProjectFileDir$/redis.conf"作为配置文件启动Redis服务器
	```
    `Executable`和`Program arguments`可以根据需要调试的信息自行设置。

	如果不想每次运行/调试前都执行`Build`操作（在这里就是`Make`构建过程），可以在编辑页下方`Before launch`框中删除`Build`条目。
    ![](https://keaper-cn-image.oss-cn-beijing.aliyuncs.com/20200705183322.gif)
5. 点击`Run`/`Debug`开始运行/调试。


# 参考
1. [Redis debugging guide](https://redis.io/topics/debugging)
2. [nickdiego/compiledb: Tool for generating Clang's JSON Compilation Database files for make-based build systems.](https://github.com/nickdiego/compiledb)
3. [Managing Makefile Projects](https://www.jetbrains.com/help/clion/managing-makefile-projects.html)
4. [Dealing with Makefile Projects in CLion: Status Update](https://blog.jetbrains.com/clion/2020/02/dealing-with-makefiles/)
