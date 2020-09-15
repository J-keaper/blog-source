---
title: '如何编译"零汇编(Zero-Assembler)"的OpenJDK'
date: 2020-09-15 08:49:24
tags: [OpenJDK,JAVA]
published: true
hideInList: false
feature: 
isTop: false
---
在使用`JetBrains CLion`调试`OpenJDK`的过程中，有时候会发现`Call Stack`中有一部分是汇编代码，导致无法完全探究其内部实现。本文主要针对此问题给出了如何在不引入汇编代码（零汇编，`Zero-Assembler`）的情况下完成`OpenJDK`项目的编译和调试。
<!-- more -->

> 在[OpenJDK 编译调试指南](http://keaper.cn/2020/07/04/OpenJDK%E7%BC%96%E8%AF%91%E8%B0%83%E8%AF%95%E6%8C%87%E5%8D%97/)一文中，已经详细介绍了编译调试`OpenJDK`的步骤，这里不再详述具体过程。

# 为什么存在部分汇编代码？
首先我们来看下为什么会存在部分汇编代码，以及这部分汇编代码是什么？
从 [HotSpot Runtime Overview - Interpreter](https://openjdk.java.net/groups/hotspot/docs/RuntimeOverview.html#Interpreter|outline) 了解到，`HotSpot`为了提高性能，使用了基于模板的解释器（`template based interpreter`）来解释执行`JAVA虚拟机指令`，`JVM`在启动时会根据`TemplateTable`（与每个字节码对应的汇编代码）来生成解释器，所以说这个解释器其实是部分基于汇编代码实现的。除了解释器之外，在`OpenJDK`项目中还有一些部分也是基于汇编代码的，例如`即时编译器JIT`（`C1`编译器，`C2`编译器）等，但汇编代码是依赖于硬件架构的，这种做法在提升了性能的同时却降低了可移植性。

`OpenJDK`社区的`IcedTea`项目提供了一种比较通用的移植方法，使得`OpenJDK`可以在所有的`Linux`系统上构建， 而无需进一步进行移植工作。

# IcedTea项目
`IcedTea`项目最初的出现是因为`Sun`公司在发布`JDK`时，类库的一些部分由于产权原因没有发布其源代码，而是仅以二进制插件的形式提供， 因为这部分源代码属于受版权保护的第三方。`IcedTea`的主要目标就是提供这些二进制插件的免费等效替代品，使得完全使用免费开源软件构建`JDK`成为可能。

此外，`IdeaTea`还做了很多工作以促进用户更容易地构建和部署`JDK`，其中包括将`OpenJDK`移植到更多的平台。

因为`OpenJDK`的代码中除了 `C++` 代码之外还包含许多汇编代码，而`OpenJDK`支持的硬件平台架构有限，远远少于`Linux`系统所支持的。于是`IcedTea`的子项目`Zero`出现了，`Zero`项目旨在通过移除`OpenJDK`项目代码中的与平台相关的汇编代码，使用纯`C++`来替代，从而可以在任何 `Linux` 系统上构建，而无需进一步进行移植工作。

> 这也正是标题中"零汇编"的含义所在，就是指不使用汇编代码来构建`OpenJDK`。

`OpenJDK` 的虚拟机在很大程度上依赖 `JIT（即时编译器）` 编译来提高性能。而`Zero`中只包含一个纯`C++`解释器，`Zero` 在相同的硬件上要比原始（`vanilla`）的 `OpenJDK` 慢得多。于是又一个子项目`Shark`出现了， `Shark`使用 `LLVM` 来即时编译 `Java` 代码，从而在不引入系统特定的代码的情况下提高性能。

# 如何实现"零汇编(Zero-Assembler)"
到这里我们已经明白了，使用`zero`来构建`OpenJDK`就可以实现"零汇编(`Zero-Assembler`)"，那么如何使用呢？目前`Zero`和`Shark`已经集成到了`OpenJDK`的主分支中，只要在配置`OpenJDK`的时候指定参数`--with-jvm-variants=zero`重新编译即可。

> 虽然在`OpenJDK 8`中提供了`--with-jvm-variants=zeroshark`的配置项，但经过几番尝试发现并不能真正构建成功。根据`OpenJDK`社区的一些资料（[[JDK-8171853] Remove Shark compiler - Java Bug System](https://bugs.openjdk.java.net/browse/JDK-8171853)、[jdk-updates/jdk10u: fb290fd1f9d4](http://hg.openjdk.java.net/jdk-updates/jdk10u/rev/fb290fd1f9d4)）， `Shark`由于长期缺乏维护已经在`JDK 10`版本中被移除了，也不再支持配置`--with-jvm-variants=zeroshark`参数了。所以下文我们只考虑`Zero`而不设计`Shark`相关内容。

> 这篇文章[Sustaining the zero assembler port in OpenJDK: An inside perspective of CPU specific issues](https://jerboaa.fedorapeople.org/presentations/OpenJDK_Zero_FOSDEM_2015-02-01.pdf)比较形象地描述了`模板解释器`和`C++解释器`的区别，以及如何构建`zero`版的`OpenJDK`，推荐阅读。


下面主要记录一下在`MacOS`和`Ubuntu`平台下编译`Zero`版`OpenJDK`所遇到的问题。
## Ubuntu 16.04
在`Ubuntu 16.04`平台上编译`OpenJDK 8`时遇到的一个问题是缺少 `libffi` 依赖。运行`configure`时出现下面的错误信息：
```
checking for LIBFFI... configure: error: Package requirements (libffi) were not met:
No package 'libffi' found
```
使用如下命令安装之后重新配置编译即可：
```bash
sudo apt-get install libffi-dev
```

在`Ubuntu 16.04`平台上编译`OpenJDK 11`时没有问题，很顺利就编译完成了。

## MacOS 10.15
在`MacOS 10.15`平台上编译时遇到较多问题，主要原因是`OpenJDK`项目对`MacOS`平台编译`zero`版本 支持不够完善，导致有一些代码编译不通过。
### OpenJDK 8
1. 运行`configure`出现下面的错误信息：
```
checking for LIBFFI... configure: error: in `/Users/jiajiawang/Workspace/openjdk-jdk8u':
configure: error: The pkg-config script could not be found or is too old.  Make sure it
is in your PATH or set the PKG_CONFIG environment variable to the full
path to pkg-config.
```
同样是因为缺少了`libffi`库，除了`libffi`之外还缺少`pkg-config`，使用`HomeBrew`安装这两个依赖即可：
```bash
brew install libffi pkg-config
```

2. `configure`完成，运行`make`时出现下面的错误信息：
```
/usr/bin/clang++  ...  ... ... openjdk-jdk8u/hotspot/src/share/vm/precompiled/precompiled.hpp -o precompiled.hpp.pch 
clang: error: unsupported option '-gstabs'
```
这是因为`clang`不支持`gcc`的`-gstabs`选项，需要对`hotspot/make/bsd/makefiles/gcc.make`文件进行修改，修改内容如下：
```diff
   DEBUG_CFLAGS/ppc   = -g
   DEBUG_CFLAGS += $(DEBUG_CFLAGS/$(BUILDARCH))
   ifeq ($(DEBUG_CFLAGS/$(BUILDARCH)),)
-  DEBUG_CFLAGS += -gstabs
+      ifeq ($(USE_CLANG), true)
+        # Clang doesn't understand -gstabs
+        DEBUG_CFLAGS += -g
+      else
+        DEBUG_CFLAGS += -gstabs
+      endif
   endif
```

3. 继续运行`make`时出现下面的错误信息：
```
... ... ... /src/os/bsd/vm/os_perf_bsd.cpp:29:10: fatal error: 'vm_version_ext_x86.hpp' file not found
#include "vm_version_ext_x86.hpp"
         ^~~~~~~~~~~~~~~~~~~~~~~~
```
这里引入了错误的文件，需要对`hotspot/src/os/bsd/vm/os_perf_bsd.cpp`做如下修改，使其引入正确的`vm_version_ext_zero.hpp`文件：
```
 #include "memory/resourceArea.hpp"
 #include "runtime/os.hpp"
 #include "runtime/os_perf.hpp"
-#include "vm_version_ext_x86.hpp"
+
+#ifdef TARGET_ARCH_aarch32
+# include "vm_version_ext_aarch32.hpp"
+#endif
+#ifdef TARGET_ARCH_x86
+# include "vm_version_ext_x86.hpp"
+#endif
+#ifdef TARGET_ARCH_sparc
+# include "vm_version_ext_sparc.hpp"
+#endif
+#ifdef TARGET_ARCH_zero
+# include "vm_version_ext_zero.hpp"
+#endif
+#ifdef TARGET_ARCH_arm
+# include "vm_version_ext_arm.hpp"
+#endif
+#ifdef TARGET_ARCH_ppc
+# include "vm_version_ext_ppc.hpp"
+#endif
```

4. 继续运行`make`时出现下面的错误信息：
```
make[2]: *** No rule to make target ` ... ... openjdk-jdk8u/jdk/src/macosx/bin/zero/jvm.cfg', needed by ` ... ... openjdk-jdk8u/build/macosx-x86_64-normal-zero-slowdebug/jdk/lib/jvm.cfg'.  Stop.
```
缺少`jdk/src/macosx/bin/zero/jvm.cfg`文件，我们从其他目录拷贝一份过来：
```bash
mkdir -p jdk/src/macosx/bin/zero/
cp jdk/src/macosx/bin/x86_64/jvm.cfg jdk/src/macosx/bin/zero
```

### OpenJDK 11
1. 运行`configure`出现下面的错误信息：
```
checking for ffi.h... no
configure: error: Could not find libffi! 
```
同样是因为缺少`ligffi`依赖，与`OpenJDK 8`不同的是在`OpenJDK 11`中，安装 `libffi`之后需要通过`--with-libffi=<path>`来指定`libffi`的地址：
```bash
# 安装libffi
brew install libffi
# 配置
sh ./configure ...省略其他参数... --with-jvm-variants=zero --with-libffi=/usr/local/Cellar/libffi/3.3/
```

2. `configure`完成，执行`make`出现下面的错误信息：
```
... ... ... /openjdk-jdk11u/src/hotspot/share/interpreter/bytecodeInterpreter.cpp:2926:13: error: 7 enumeration values not handled in switch: 'T_VOID', 'T_ADDRESS', 'T_NARROWOOP'... [-Werror,-Wswitch]
    switch (istate->method()->result_type()) {
            ^
... ... ... /openjdk-jdk11u/src/hotspot/share/interpreter/bytecodeInterpreter.cpp:2926:13: note: add missing switch cases
    switch (istate->method()->result_type()) {
            ^
```
这是因为源代码中的`switch`语句没有处理所有的分支情况，所有`clang`给出了警告（`Diagnostic`）,导致编译不通过，我们可以通过`clang`的`-w`参数来取消所有的警告（`Diagnostic`），需要在配置时添加如下两个参数，重新编译即可：
```bash
sh ./configure  ...省略其他参数... --with-extra-cflags=-w --with-extra-cxxflags=-w
```

3. 继续运行`make`时出现下面的错误信息：
```
... ... ... /src/os/bsd/vm/os_perf_bsd.cpp:29:10: fatal error: 'vm_version_ext_x86.hpp' file not found
#include "vm_version_ext_x86.hpp"
         ^~~~~~~~~~~~~~~~~~~~~~~~
```
这里引入了错误的文件，需要对`src/hotspot/os/bsd/os_perf_bsd.cpp`做如下修改，使其引入正确的`vm_version_ext_zero.hpp`文件（与`OpenJDK 8`中稍有不同）：
```
 #include "runtime/os.hpp"
 #include "runtime/os_perf.hpp"
 #include "utilities/globalDefinitions.hpp"
-#include "vm_version_ext_x86.hpp"
+
+#include CPU_HEADER(vm_version_ext)
```


# 参考资料
1. [HotSpot Runtime Overview - Interpreter](https://openjdk.java.net/groups/hotspot/docs/RuntimeOverview.html#Interpreter|outline)
2. [Sustaining the zero assembler port in OpenJDK: An inside perspective of CPU specific issues](https://jerboaa.fedorapeople.org/presentations/OpenJDK_Zero_FOSDEM_2015-02-01.pdf)
3. [Demystifying the JVM: JVM Variants, Cppinterpreter and TemplateInterpreter](https://metebalci.com/blog/demystifying-the-jvm-jvm-variants-cppinterpreter-and-templateinterpreter/)
4. [OpenJDK: IcedTea Project](https://openjdk.java.net/projects/icedtea/)
5. [IcedTea - Wikipedia](https://en.wikipedia.org/wiki/IcedTea)
6. [Main Page - IcedTea](https://icedtea.classpath.org/wiki/Main_Page)
7. [Zero-Assembler Project](https://openjdk.java.net/projects/zero/)
8. [ZeroSharkFaq - IcedTea](https://icedtea.classpath.org/wiki/ZeroSharkFaq)
9. [The New Hotspot Build System](http://cr.openjdk.java.net/~ihse/docs/new-hotspot-build.html)
