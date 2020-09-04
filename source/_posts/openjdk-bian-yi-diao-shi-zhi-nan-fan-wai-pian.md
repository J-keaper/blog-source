---
title: 'OpenJDK 编译调试指南 - 番外篇'
date: 2020-09-03 20:22:51
tags: [JAVA,OpenJDK]
published: true
hideInList: false
feature: 
isTop: false
---
作为[OpenJDK 编译调试指南(Ubuntu 16.04 + MacOS 10.15)](https://jiawanggjia.github.io/post/openjdk-bian-yi-zhi-nan/)的番外篇，本文主要用来记录在调试`OpenJDK`过程过程中遇到的一些问题以及解决办法。

<!-- more -->
# 如何编译"零汇编(Zero-Assembler)"的OpenJDK
在使用`JetBrains CLion`调试`OpenJDK`的过程中，发现有时候`Call Stack`中有一部分是汇编代码，导致我们无法完全探究其内部实现。有没有办法能够不使用这部分汇编代码呢？

首先我们来看下为什么会存在部分汇编代码？
## 为什么存在部分汇编代码？
从 [HotSpot Runtime Overview - Interpreter](https://openjdk.java.net/groups/hotspot/docs/RuntimeOverview.html#Interpreter|outline) 了解到，`HotSpot`为了提高性能，使用了基于模板的解释器（`template based interpreter`）来解释执行`JAVA虚拟机指令`，`JVM`在启动时会根据`TemplateTable`（与每个字节码对应的汇编代码）来生成解释器，所以说这个解释器其实是部分基于汇编代码实现的。除了解释器之外，在`OpenJDK`项目中还有一些部分也是基于汇编代码的，例如`即时编译器JIT`（`C1`编译器，`C2`编译器）等，但汇编代码是依赖于硬件架构的，这种做法在提升了性能的同时却降低了可移植性。

`OpenJDK`社区的`IcedTea`项目提供了一种比较通用的移植方法，使得`OpenJDK`可以在所有的`Linux`系统上构建， 而无需进一步进行移植工作。

## IcedTea项目
`IcedTea`项目最初的出现是因为`Sun`公司在发布`JDK`时，类库的一些部分由于产权原因没有发布其源代码，而是仅以二进制插件的形式提供， 因为这部分源代码属于受版权保护的第三方。`IcedTea`的主要目标就是提供这些二进制插件的免费等效替代品，使得完全使用免费开源软件构建`JDK`成为可能。

此外，`IdeaTea`还做了很多工作以促进用户更容易地构建和部署`JDK`，其中包括将`OpenJDK`移植到更多的平台。

因为`OpenJDK`的代码中除了 `C++` 代码之外还包含许多汇编代码，而`OpenJDK`支持的硬件平台架构有限，远远少于`Linux`系统所支持的。于是`IcedTea`的子项目`Zero`出现了，`Zero`项目旨在通过移除`OpenJDK`项目代码中的与平台相关的汇编代码，使用纯`C++`来替代，从而可以在任何 `Linux` 系统上构建，而无需进一步进行移植工作。

> 这也正是标题中"零汇编"的含义所在，就是指不使用汇编代码来构建`OpenJDK`。

`OpenJDK` 的虚拟机在很大程度上依赖 `JIT（即时编译器）` 编译来提高性能。而`Zero`中只包含一个纯`C++`解释器，`Zero` 在相同的硬件上要比原始（`vanilla`）的 `OpenJDK` 慢得多。于是又一个子项目`Shark`出现了， `Shark`使用 `LLVM` 来即时编译 `Java` 方法，从而在不引入系统特定的代码的情况下提高性能。

## 如何实现"零汇编(Zero-Assembler)"
到这里我们已经明白了，使用`zero`来构建`OpenJDK`就可以实现"零汇编(`Zero-Assembler`)"，那么如何使用呢？目前`Zero`和`Shark`已经集成到了`OpenJDK`的主分支中，只要在配置`OpenJDK`的时候指定参数`--with-jvm-variants=zero`重新编译即可。
```shell
sh ./configure  --with-jvm-variants=zero # 未列出全部参数
make all
```

## 参考资料
1. [HotSpot Runtime Overview - Interpreter](https://openjdk.java.net/groups/hotspot/docs/RuntimeOverview.html#Interpreter|outline)
2. [Sustaining the zero assembler port in OpenJDK: An inside perspective of CPU specific issues](https://jerboaa.fedorapeople.org/presentations/OpenJDK_Zero_FOSDEM_2015-02-01.pdf)
3. [Demystifying the JVM: JVM Variants, Cppinterpreter and TemplateInterpreter](https://metebalci.com/blog/demystifying-the-jvm-jvm-variants-cppinterpreter-and-templateinterpreter/)
4. [OpenJDK: IcedTea Project](https://openjdk.java.net/projects/icedtea/)
5. [IcedTea - Wikipedia](https://en.wikipedia.org/wiki/IcedTea)
6. [Main Page - IcedTea](https://icedtea.classpath.org/wiki/Main_Page)
7. [Zero-Assembler Project](https://openjdk.java.net/projects/zero/)
8. [ZeroSharkFaq - IcedTea](https://icedtea.classpath.org/wiki/ZeroSharkFaq)
9. [The New Hotspot Build System](http://cr.openjdk.java.net/~ihse/docs/new-hotspot-build.html)
