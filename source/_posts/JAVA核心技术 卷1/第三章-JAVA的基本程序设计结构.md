---
title: 第三章 JAVA的基本程序设计结构
author: Keaper
tags:
  - JAVA
categories:
date: 2018-08-11 15:26:00
---
##### 1.字面值常量
>[http://www.cnblogs.com/hwaggLee/p/4507968.html](http://www.cnblogs.com/hwaggLee/p/4507968.html)
>字面值大体上可以分为整型字面值、浮点字面值、字符和字符串字面值、特殊字面值。
>1. 整形字面值
>一般情况下，字面值创建的是int类型，但是int字面值可以赋值给byte short char long int，只要字面值在目标范围以内，Java会自动完成转换，如果试图将超出范围的字面值赋给某一类型（比如把128赋给byte类型），编译通不过。而如果想创建一个int类型无法表示的long类型，则需要在字面值最后面加上L或者l。通常建议使用容易区分的L。所以整型字面值包括int字面值和long字面值两种。
>2. 浮点字面值
>分为float字面值和double字面值，如果在小数后面加上F或者f，则表示这是个float字面值，如11.8F。如果小数后面不加F（f），如10.4。或者小数后面加上D（d），则表示这是个double字面值。
>3. 字符及字符串字面值
>Java中字符字面值用单引号括起来，如‘@’‘1’。
>字符串字面值则使用双引号，字符串字面值中同样可以包含字符字面值中的转义字符序列。

##### 2. char类型大小
16bit，两个字节。
```java
/**
     * The number of bits used to represent a <tt>char</tt> value in unsigned
     * binary form, constant {@code 16}.
     *
     * @since 1.5
     */
    public static final int SIZE = 16;

    /**
     * The number of bytes used to represent a {@code char} value in unsigned
     * binary form.
     *
     * @since 1.8
     */
    public static final int BYTES = SIZE / Byte.SIZE;
```
##### 3. 变量使用之前一定要初始化。
可以不在定义是初始化，但一定要在使用之前初始化

##### 4. &和|用于布尔值
&&，|| 和&，| 用于boolean的差别在于，前者有短路特性，后者没有。
用于整数。
而在C++中&和|都是按照整数处理的。

##### 5. \>>>和>>，C++中的右移位
\>>>表示算术移位，>>表示逻辑移位。
C++中，无符号数逻辑右移，有符号数算术右移。

##### 6. char如何转换int
char转换int返回code point。

##### 7. 类型转换
int转换float，int转换double，long转换double都会有精度丢失。但是可以进行自动转换，无需强制转换。
两个数值类型进行计算时，如果类型不匹配，会将"低类型""转化为"高类型"，然后在进行计算。
>如果有一个为double，转换另一个为double
>否则，如果有一个为float，转换另一个为float
>否则，如果有一个为long，转换另一个为long
>否则，都转换为int。

**不要使用Boolean转换类型**

##### 8. string类对象不可变字符串
String对象创建的时候，
1. 如果直接用字符创字面值常量创建，如：`String s = "abc"`，如果string literal pool（常量池）中不存在"abc"这个对象，那么会创建**一个**对象，如果存在，则不会创建；
2. 如果new String()，那么总共会创建两个对象，一个对象为字面值常量本身，另一个为单独的字符串对象。
参考：[http://www.cnblogs.com/javaminer/p/3923484.html](http://www.cnblogs.com/javaminer/p/3923484.html)

##### 9. code point 和 code unit
String length()返回的是code unit的数量。
codePointCount()可以返回code point 的数量。
charAt(n)返回位置为n的code unit。
codePointAt(n) 返回位置为n的code point。
这种差别会在一些字符上表现出来，[UTF-16 - Wikipedia](https://en.wikipedia.org/wiki/UTF-16)上可以找到两个字符，'𐐷'和'𤭢'都是由两个code unit来表示的。

##### 10. StringBuilder和SttingBuffer
这两个类的对象是可以被修改的，与String不同。两个类 的API相同。
由于 StringBuilder 相较于 StringBuffer 有速度优势，所以多数情况下建议使用 StringBuilder 类。然而在应用程序要求线程安全的情况下，则必须使用 StringBuffer 类。

##### 11. Console类
Console类可以用来从控制台读取密码，输入不可见。
[https://docs.oracle.com/javase/7/docs/api/java/io/Console.html](https://docs.oracle.com/javase/7/docs/api/java/io/Console.html)

##### 12. printf对日期处理，参数下标
JAVA 中增加了printf对日期的格式化。
支持参数下标，实现一个参数多次使用。
这两种方式输出是相同的：
```java
        System.out.printf("%1$s %2$tB %2$te, %2$tY","date:",new Date());

        System.out.printf("%s %tB %<te, %<tY","date:",new Date());
```

##### 13. Scanner读文件PrintWriter写文件
```java
        Scanner scanner = new Scanner(Paths.get("a.txt"));
        
        PrintWriter out = new PrintWriter("a.txt");
```

##### 14. 不能在嵌套的块中定义同名变量
与 C++ 不同，C++ 中定义的同名变量外层会被内层覆盖，而JAVA是不允许的。

##### 15. 带标签的break，continue
JAVA中为了支持goto，break可以带标签，该语句会跳到标签定义处，可以实现跳到外层循环。
continue也可以跳到标签指定处。

##### 16. 命令行参数第一个参数
C++中命令行参数的第一个参数为文件名，而JAVA是不会将文件名计算在参数内的。

##### 17. 数组
1. 数组也是会存放在常量池中的，所以用 = 赋值个另一个数组变量时，两者指向同一个引用。修改一个另一个也会变化。如果想要拷贝一个新的数组，可以用`Arrays.copy()`。
2. for each 不能自动处理多维数组
3. deepToString()快速打印多维数组。
4. JAVA多维数组长度可以不相等，多维数组中，每个元素存的都是另一个数组的引用。