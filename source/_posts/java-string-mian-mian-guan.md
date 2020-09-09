---
title: 'Java String 面面观'
date: 2020-09-08 21:03:24
tags: [JAVA]
published: true
hideInList: false
feature: 
isTop: false
---
本文主要介绍`Java`中与字符串相关的一些内容，主要包括`String`类的实现及其不变性、`String`相关类（`StringBuilder`、`StringBuffer`）的实现 以及 字符串缓存机制的用法与实现。

<!-- more -->
# String类的设计与实现
`String`类的核心逻辑是通过对`char`型数组进行封装来实现字符串对象，但实现细节伴随着`Java`版本的演进也发生过几次变化。
## Java 6
```java
public final class String implements java.io.Serializable, Comparable<String>, CharSequence
{
    /** The value is used for character storage. */
    private final char value[];
    /** The offset is the first index of the storage that is used. */
    private final int offset;
    /** The count is the number of characters in the String. */
    private final int count;
    /** Cache the hash code for the string */
    private int hash; // Default to 0
}
```
在`Java 6`中，`String`类有四个成员变量：`char`型数组`value`、偏移量 `offset`、字符数量 `count`、哈希值 `hash`。`value`数组用来存储字符序列， `offset` 和 `count` 两个属性用来定位字符串在`value`数组中的位置，`hash`属性用来缓存字符串的`hashCode`。

使用`offset`和`count`来定位`value`数组的目的是，可以高效、快速地共享`value`数组，例如`substring()`方法返回的子字符串是通过记录`offset`和`count`来实现与原字符串共享`value`数组的，而不是重新拷贝一份。`substring()`方法实现如下：
```java
String(int offset, int count, char value[]) {
	this.value = value;    // 直接复用原数组
	this.offset = offset;
	this.count = count;
}
public String substring(int beginIndex, int endIndex) {
    // ...... 省略一些边界检查的代码 ......
    return ((beginIndex == 0) && (endIndex == count)) ? this :
        new String(offset + beginIndex, endIndex - beginIndex, value);
}
```
但是这种方式却很有可能会导致内存泄漏。例如在如下代码中：
```java
String bigStr = new String(new char[100000]);
String subStr = bigStr.substring(0,2);
bigStr = null;
```
在`bigStr`被设置为`null`之后，其中的`value`数组却仍然被`subStr`所引用，导致垃圾回收器无法将其回收，结果虽然我们实际上仅仅需要`2`个字符的空间，但是实际却占用了`100000`个字符的空间。

在`Java 6`中，如果想要避免这种内存泄漏情况的发生，可以使用下面的方式：
```java
String subStr = bigStr.substring(0,2) + "";
// 或者
String subStr = new String(bigStr.substring(0,2));
```
在语句执行完之后，`substring`方法返回的匿名`String`对象由于没有被别的对象引用，所以能够被垃圾回收器回收，不会继续引用`bigStr`中的`value`数组，从而避免了内存泄漏。

## Java 7 & Java 8
```java
public final class String implements java.io.Serializable, Comparable<String>, CharSequence {
    /** The value is used for character storage. */
    private final char value[];
    /** Cache the hash code for the string */
    private int hash; // Default to 0
}
```
在`Java 7`-`Java 8`中，`Java` 对 `String` 类做了一些改变。`String` 类中不再有 `offset` 和 `count` 两个成员变量了。`substring()`方法也不再共享 `value`数组，而是从指定位置重新拷贝一份`value`数组，从而解决了使用该方法可能导致的内存泄漏问题。`substring()`方法实现如下：
```java
public String(char value[], int offset, int count) {
    // ...... 省略一些边界检查的代码 ......

    // 从原数组拷贝
    this.value = Arrays.copyOfRange(value, offset, offset+count);   
}
public String substring(int beginIndex, int endIndex) {
    // ...... 省略一些边界检查的代码 ......
    return ((beginIndex == 0) && (endIndex == value.length)) ? this
            : new String(value, beginIndex, subLen);
}
```

## Java 9
```java
public final class String implements java.io.Serializable, Comparable<String>, CharSequence {
    /** The value is used for character storage. */
    private final byte[] value;
    /**  The identifier of the encoding used to encode the bytes in {@code value}. */
    private final byte coder;
    /** Cache the hash code for the string */
    private int hash; // Default to 0
}
```
为了节省内存空间，`Java 9`中对`String`的实现方式做了优化，`value`成员变量从`char[]`类型改为了`byte[]`类型，同时新增了一个`coder`成员变量。我们知道`Java`中`char`类型占用的是两个字节，对于只占用一个字节的字符（例如，`a-z`，`A-Z`）就显得有点浪费，所以`Java 9`中将`char[]`改为`byte[]`来存储字符序列，而新属性 `coder` 的作用就是用来表示`value`数组中存储的是双字节编码的字符还是单字节编码的字符。`coder` 属性可以有 `0` 和 `1` 两个值，`0` 代表 `Latin-1`（单字节编码），`1` 代表 `UTF-16`（双字节编码）。在创建字符串的时候如果判断所有字符都可以用单字节来编码，则使用`Latin-1`来编码以压缩空间，否则使用`UTF-16`编码。主要的构造函数实现如下：
```java
String(char[] value, int off, int len, Void sig) {
    if (len == 0) {
        this.value = "".value;
        this.coder = "".coder;
        return;
    }
    if (COMPACT_STRINGS) {
        byte[] val = StringUTF16.compress(value, off, len);  // 尝试压缩字符串，使用单字节编码存储
        if (val != null) {   // 压缩成功，可以使用单字节编码存储
            this.value = val;
            this.coder = LATIN1;
            return;
        }
    }
    // 否则，使用双字节编码存储
    this.coder = UTF16;
    this.value = StringUTF16.toBytes(value, off, len);
}
```

# String类的不变性
我们注意到`String`类是用`final`修饰的；所有的属性都是声明为`private`的；并且除了`hash`属性之外的其他属性也都是用`final`修饰。这保证了：
1. `String`类由`final`修饰，所以无法通过继承`String`类改变其语义；
2. 所有的属性都是声明为`private`的， 所以无法在`String`外部**直接**访问或修改其属性；
3. 除了`hash`属性之外的其他属性都是用`final`修饰，表示这些属性在初始化赋值后不可以再修改。

上述的定义共同实现了`String`类一个重要的特性 —— **不变性**，即 `String` 对象一旦创建成功，就不能再对它进行任何修改。`String`提供的方法`substring()`、`concat()`、`replace()`等方法返回值都是新创建的`String`对象，而不是原来的`String`对象。

> `hash`属性不是`final`的原因是：`String`的`hashCode`并不需要在创建字符串时立即计算并赋值，而是在`hashCode()`方法被调用时才需要进行计算。

**为什么String类要设计为不可变的？**
1. 保证 `String` 对象的安全性。`String`被广泛用作`JDK`中作为参数、返回值，例如网络连接，打开文件，类加载，等等。如果 `String` 对象是可变的，那么 `String` 对象将可能被恶意修改，引发安全问题。
2. 线程安全。`String`类的不可变性天然地保证了其线程安全的特性。
3. 保证了`String`对象的`hashCode`的不变性。`String`类的不可变性，保证了其`hashCode`值能够在第一次计算后进行缓存，之后无需重复计算。这使得`String`对象很适合用作`HashMap`等容器的`Key`，并且相比其他对象效率更高。
4. 实现`字符串常量池`。`Java`为字符串对象设计了`字符串常量池`来共享字符串，节省内存空间。如果字符串是可变的，那么字符串对象便无法共享。因为如果改变了其中一个对象的值，那么其他对象的值也会相应发生变化。

# 与String类相关的类
除了`String`类之外，还有两个与`String`类相关的的类：`StringBuffer`和`StringBuilder`，这两个类可以看作是`String`类的可变版本，提供了对字符串修改的各种方法。两者的区别在于`StringBuffer`是线程安全的而`StringBuilder`不是线程安全的。

## StringBuffer / StringBuilder的实现
`StringBuffer`和`StringBuilder`都是继承自`AbstractStringBuilder`，`AbstractStringBuilder`利用可变的`char`数组（`Java 9`之后改为为`byte`数组）来实现对字符串的各种修改操作。`StringBuffer`和`StringBuilder`都是调用`AbstractStringBuilder`中的方法来操作字符串， 两者区别在于`StringBuffer`类中对字符串修改的方法都加了`synchronized`修饰，而`StringBuilder`没有，所以`StringBuffer`是线程安全的，而`StringBuilder`并非线程安全的。

我们以`Java 8`为例，看一下`AbstractStringBuilder`类的实现：
```java
abstract class AbstractStringBuilder implements Appendable, CharSequence {
    /** The value is used for character storage. */
    char[] value;
    /** The count is the number of characters used. */
    int count;
}
```
`value`数组用来存储字符序列，`count`则用来存储`value`数组中已经使用的字符数量，字符串真实的内容是`value`数组中`[0,count)`之间的字符序列，而`[count,length)`之间是**未使用**的空间。需要`count`属性记录已使用空间的原因是，`AbstractStringBuilder`中的`value`数组并不是每次修改都会重新申请，而是会提前预分配一定的多余空间，以此来减少重新分配数组空间的次数。（这种做法类似于`ArrayList`的实现）。

`value`数组扩容的策略是：当对字符串进行修改时，如果当前的`value`数组不满足空间需求时，则会重新分配更大的`value`数组，分配的数组大小为`min( 原数组大小×2 + 2 , 所需的数组大小 )`，更加细节的逻辑可以参考如下代码：
```java
private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;

private int newCapacity(int minCapacity) {
    // overflow-conscious code
    int newCapacity = (value.length << 1) + 2;    //原数组大小×2 + 2 
    if (newCapacity - minCapacity < 0) {     // 如果小于所需空间大小，扩展至所需空间大小
        newCapacity = minCapacity;
    }
    return (newCapacity <= 0 || MAX_ARRAY_SIZE - newCapacity < 0)
        ? hugeCapacity(minCapacity)
        : newCapacity;
}

private int hugeCapacity(int minCapacity) {
    if (Integer.MAX_VALUE - minCapacity < 0) { // overflow
        throw new OutOfMemoryError();
    }
    return (minCapacity > MAX_ARRAY_SIZE)
        ? minCapacity : MAX_ARRAY_SIZE;
}
```
当然`AbstractStringBuilder`也提供了`trimToSize`方法去释放多余的空间：
```java
public void trimToSize() {
    if (count < value.length) {
        value = Arrays.copyOf(value, count);
    }
}
```

# String对象的缓存机制
因为`String`对象的使用广泛，`Java`为`String`对象设计了缓存机制，以提升时间和空间上的效率。在`JVM`的运行时数据区中存在一个`字符串常量池`（`String Pool`），在这个常量池中维护了所有已经缓存的`String`对象，当我们说一个`String`对象被缓存（`interned`）了，就是指它进入了`字符串常量池`。

我们通过解答下面三个问题来理解`String`对象的缓存机制：
1. 哪些`String`对象会被缓存进`字符串常量池`？
2. `String`对象被缓存在哪里，如何组织起来的？
3. `String`对象是什么时候进入`字符串常量池`的？

> **说明**： 如未特殊指明，本文中提及的`JVM`实现均指的是`Oracle`的`HotSpot VM`，并且不考虑 逃逸分析（`escape analysis`）、标量替换（`scalar replacement`）、无用代码消除（`dead-code elimination`）等优化手段，测试代码基于不添加任何额外`JVM`参数的情况下运行。

## 预备知识
为了更好的阅读体验，在解答上面三个问题前，希望读者对以下知识点有简单了解：
- `JVM`运行时数据区
- `class文件`的结构
- `JVM`基于栈的字节码解释执行引擎
- 类加载的过程
- `Java`中的几种常量池

为了内容的完整性，我们对下文涉及较多的其中两点做简要介绍。
### 类加载的过程 
类从被加载到虚拟机内存中开始，到卸载出内存为止，它的整个生命周期依次为：加载（`Loading`）、验证（`Verification`）、准备（`Preparation`）、解析（`Resolution`）、初始化（`Initialization`）、使用（`Using`）和卸载（`Unloading`）7个阶段。其中验证、准备、解析3个部分统称为连接（`Linking`）。
加载、验证、准备、初始化和卸载这5个阶段的顺序是确定的，类的加载过程必须按照这种顺序按部就班地开始，而解析阶段则不一定：它在某些情况下可以在初始化阶段之后再开始，这是为了支持Java语言的运行时绑定（也称为动态绑定或晚期绑定）。

### Java中的几种常量池
**1. class文件中的常量池**
我们知道`java`后缀的源代码文件会被`javac`编译为`class`后缀的`class文件`（字节码文件）。在`class文件`中有一部分内容是 [常量池（Constant Pool）](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-4.html#jvms-4.4) ，这个常量池中主要存储两大类常量：
- 代码中的`字面量`或者`常量表达式`的值；
- 符号引用，包括：类和接口的全限定名、字段的名称和描述符、方法的名称和描述符等。

**2. 运行时常量池**
在`JVM`[运行时数据区（Run-Time Data Areas）](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-2.html#jvms-2.5)中，有一部分是[运行时常量池（Run-Time Constant Pool）](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-2.html#jvms-2.5.5)，属于`方法区`的一部分。`运行时常量池`是`class文件`中每个类或者接口的常量池（`Constant Pool` ）的运行时表示形式，`class文件`的常量池中的内容会在类加载后进入方法区的`运行时常量池`。

**3. 字符串常量池**
`字符串常量池`（`String Pool`）也就是我们上文提到的用来缓存`String`对象的常量池。 这个常量池是全局共享的，属于运行时数据区的一部分。

## 哪些String对象会被缓存进字符串常量池？
在`Java`中，有两种字符串会被缓存到`字符串常量池`中，一种是在代码中定义的`字符串字面量`或者`字符串常量表达式`，另一种是程序中主动调用`String.intern()`方法将当前`String`对象缓存到`字符串常量池`中。下面分别对两种方式做简要介绍。
### 1. 隐式缓存 - 字符串字面量 或者 字符串常量表达式
之所以称之为隐式缓存是因为我们并不需要主动去编写缓存相关代码，编译器和`JVM`会帮我们完成这部分工作。

**字符串字面量**
第一种会被隐式缓存的字符串是 **字符串字面量**。`字面量` 是类型为原始类型、`String`类型、`null`类型的值在源代码中的表示形式。例如：
```java
int i = 100;   // int 类型字面量
double f = 10.2;  // double 类型字面量
boolean b = true;   // boolean 类型字面量
String s = "hello"; // String类型字面量
Object o = null;  // null类型字面量
```
`字符串字面量`是由双引号括起来的`0`个或者多个字符构成的。 `Java`会在执行过程中为`字符串字面量`创建`String`对象并加入`字符串常量池`中。例如上面代码中的`"hello"`就是一个`字符串字面量`，在执行过程中会先 创建一个内容为`"hello"`的`String`对象，并缓存到`字符串常量池`中，再将`s`引用指向这个`String`对象。

关于`字符串字面量`更加详细的内容请参阅`Java语言规范`（[JLS - 3.10.5. String Literals](https://docs.oracle.com/javase/specs/jls/se8/html/jls-3.html#jls-3.10.5)）。

**字符串常量表达式**
另外一种会被隐式缓存的字符串是 **字符串常量表达式**。`常量表达式`指的是表示简单类型值或`String`对象的表达式，可以简单理解为`常量表达式`就是在编译期间就能确定值的表达式。`字符串常量表达式`就是表示`String`对象的常量表达式。例如：
```java
int a = 1 + 2;
double d = 10 + 2.01;
boolean b = true & false;
String str1 =  "abc" + 123;

final int num = 456;
String  str2 = "abc" +456;
```
`Java`会在执行过程中为`字符串常量表达式`创建`String`对象并加入`字符串常量池`中。例如，上面的代码中，会分别创建`"abc123"`和`"abc456"`两个`String`对象，这两个`String`对象会被缓存到`字符串常量池`中，`str1`会指向常量池中值为`"abc123"`的`String`对象，`str2`会指向常量池中值为`"abc456"`的`String`对象。

关于`常量表达式`更加详细的内容请参阅`Java语言规范`（[JLS - 15.28 Constant Expressions](https://docs.oracle.com/javase/specs/jls/se8/html/jls-15.html#jls-15.28)）。
### 2. 主动缓存 - String.intern()方法
除了声明为`字符串字面量`/`字符串常量表达式`之外，通过其他方式得到的`String`对象也可以主动加入`字符串常量池`中。例如：
```java
String str = new String("123") + new String("456");
str.intern();
```
在上面的代码中，在执行完第一句后，常量池中存在内容为`"123"`和`"456"`的两个`String`对象，但是不存在`"123456"`的`String`对象，但在执行完`str.intern();`之后，内容为`"123456"`的`String`对象也加入到了`字符串常量池`中。

我们通过`String.intern()`方法的注释来看下其具体的缓存机制：
> When the intern method is invoked, if the pool already contains a string equal to this String object as determined by the equals(Object) method, then the string from the pool is returned. Otherwise, this String object is added to the pool and a reference to this String object is returned.
> It follows that for any two strings s and t, s.intern() == t.intern() is true if and only if s.equals(t) is true.

简单翻译一下：
> 当调用 `intern` 方法时，如果常量池中已经包含相同内容的字符串（字符串内容相同由 `equals (Object)` 方法确定，对于 `String` 对象来说，也就是字符序列相同），则返回常量池中的字符串对象。否则，将此 `String` 对象将添加到常量池中，并返回此 `String` 对象的引用。
> 因此，对于任意两个字符串 `s` 和 `t`，当且仅当 `s.equals(t)`的结果为`true`时，`s.intern() == t.intern()`的结果为`true`。

## String对象被缓存在哪里，如何组织起来的？
`HotSpot VM`中，有一个用来记录缓存的`String`对象的全局表，叫做`StringTable`，结构及实现方式都类似于`Java`中的`HashMap`或者`HashSet`，是一个使用拉链法解决哈希冲突的哈希表，可以简单理解为`HashSet<String>`，注意它只存储对`String`对象的引用，而不存储`String`对象实例。 一般我们说一个字符串进入了`字符串常量池`其实是说在这个`StringTable`中保存了对它的引用，反之，如果说没有在其中就是说`StringTable`中没有对它的引用。

而真正的字符串对象其实是保存在另外的区域中的，在`Java 6`中`字符串常量池`中的`String`对象是存储在`永久代`（`Java 8`之前`HotSpot VM`对`方法区`的实现）中的，而在`Java 6`之后，`字符串常量池`中的`String`对象是存储在`堆`中的。

> `Java 7`中将`字符串常量池`中的对象移动到`堆`中的原因是在 `Java 6`中，`字符串常量池`中的对象在`永久代`创建，而`永久代`代的大小一般不会设置太大，如果大量使用字符串缓存将可能对导致`永久代`发生`OOM`异常。

## String对象是什么时候进入字符串常量池的？
对于通过 在程序中调用`String.intern()`方法主动缓存进入常量池的`String`对象，很显然就是在调用`intern()`方法的时候进入常量池的。

我们重点来研究一下会被隐式缓存的两种值（`字符串字面量`和`字符串常量表达式`），主要是两个问题：
1. 我们并没有主动调用`String`类的构造方法，那么它们是在何时被创建？
2. 它们是在何时进入`字符串常量池`的？

我们以下面的代码为例来分析这两个问题：
```java
public class Main {
    public static void main(String[] args) {
        String str1 = "123" + 123;     // 字符串常量表达式
        String str2 = "123456";         // 字面量
        String str3 = "123" + 456;   //字符串常量表达式
    }
}
```

### 字节码分析
我们对上述代码编译之后使用`javap`来观察一下字节码文件，为了节省篇幅，只摘取了相关的部分：常量池表部分以及`main`方法信息部分：
```java
Constant pool:
  #1 = Methodref          #5.#23         // java/lang/Object."<init>":()V
  #2 = String             #24            // 123123
  #3 = String             #25            // 123456
   // ...... 省略 ......
  #24 = Utf8               123123
  #25 = Utf8               123456
 
 // ...... 省略 ......

  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=1, locals=4, args_size=1
         0: ldc           #2                  // String 123123
         2: astore_1
         3: ldc           #3                  // String 123456
         5: astore_2
         6: ldc           #3                  // String 123456
         8: astore_3
         9: return
      LineNumberTable:
        line 7: 0
        line 8: 3
        line 9: 6
        line 10: 9
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      10     0  args   [Ljava/lang/String;
            3       7     1  str1   Ljava/lang/String;
            6       4     2  str2   Ljava/lang/String;
            9       1     3  str3   Ljava/lang/String;
```
在`常量池`中，有两种与字符串相关的常量类型，`CONSTANT_String`和`CONSTANT_Utf8`。`CONSTANT_String`类型的常量用于表示`String`类型的常量对象，其内容只是一个常量池的索引值`index`，`index`处的成员必须是`CONSTANT_Utf8`类型。而`CONSTANT_Utf8`类型的常量用于存储真正的字符串内容。
例如，上面的常量池中的第`2`、`3`项是`CONSTANT_String`类型，存储的索引分别为`24`、`25`，常量池中第`24`、`25`项就是`CONSTANT_Utf8`，存储的值分别为`"123123"`，`"123456"`。

`class文件`的方法信息中`Code`属性是`class文件`中最为重要的部分之一，其中包含了执行语句对应的虚拟机指令，异常表，本地变量信息等，其中`LocalVariableTable`是本地变量的信息，`Slot`可以理解为本地变量表中的索引位置。`ldc`指令的作用是从`运行时常量池`中提取指定索引位置的数据并压入栈中；`astore_<n>`指令的作用是将一个引用类型的值从栈中弹出并保存到本地变量表的指定位置，也就是`<n>`指定的位置。可以看出三条赋值语句所对应的字节码指令其实都是相同的：
```java
ldc           #<index>   // 首先将常量池中指定索引位置的String对象压入栈中
astore_<n>   // 然后从栈中弹出刚刚存入的String对象保存到本地变量的指定位置
```

### 运行过程分析
还是围绕上面的代码，我们结合 从编译到执行的过程 来分析一下`字符串字面量`和`字符串常量表达式`的**创建**及**缓存**时机。

**1. 编译**
首先，第一步是`javac`将源代码编译为`class`文件。在源代码编译过程中，我们上文提到的两种值 `字符串字面量`（`"123456"`） 和 `字符串常量表达式`（`"123" + 456`）这两类值都会存在编译后的`class文件`的常量池中，常量类型为`CONSTANT_String`。值得注意的两点是：

- `字符串常量表达式`会在编译期计算出真实值存在`class`文件的`常量池`中。例如上面源代码中的`"123" + 123`这个表达式在`class`文件的常量池中的表现形式是`123123`，`"123" + 456`这个表达式在`class`文件的常量池中的表现形式是`123456`；
- 值相同的`字符串字面量`或者`字符串常量表达式`在`class文件`的常量池中只会存在一个常量项（`CONSTANT_String`类型和`CONSTANT_Utf8`都只有一项）。例如上面源代码中，虽然声明了两个常量值分别为`"123456"`和`"123" + 456`，但是最后`class`文件的常量池中只有一个值为`123456`的`CONSTANT_Utf8`常量项以及一个对应的`CONSTANT_String`常量项。

**2. 类加载**
在`JVM`运行时，加载`Main`类时，`JVM`会根据 `class文件`的常量池 创建 `运行时常量池`， `class文件`的常量池 中的内容会在类加载时进入方法区的 `运行时常量池`。对于`class文件`的常量池中的符号引用，会在类加载的`解析(resolve)阶段`，会将其转化为真正的值。但在`HotSpot`中，符号引用的`解析`并不一定是在类加载时立即执行的，而是推迟到第一次执行相关指令（即引用了符号引用的指令，[JLS - 5.4.3. Resolution](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-5.html#jvms-5.4.3) ）时才会去真正进行解析，这就做`延迟解析`/`惰性解析`（`"lazy" or "late" resolution`）。

 - 对于一些基本类型的常量项，例如`CONSTANT_Integer_info`，`CONSTANT_Float_info`，`CONSTANT_Long_info`，`CONSTANT_Double_info`，在类加载阶段会将`class`文件常量池中的值转化为`运行时常量池`中的值，分别对应`C++`中的`int`，`float`，`long`，`double`类型；
 - 对于`CONSTANT_Utf8`类型的常量项，在类加载的解析阶段被转化为`Symbol`对象（`HotSpot VM`层面的一个`C++`对象）。同时`HotSpot`使用`SymbolTable`（结构与`StringTable`类似）来缓存`Symbol`对象，所以在类加载完成后，`SymbolTable`中应该有所有的`CONSTANT_Utf8`常量对应的`Symbol`对象；
 - 而对于`CONSTANT_String`类型的常量项，因为其内容是一个符号引用（指向`CONSTANT_Utf8`类型常量的索引值），所以需要进行解析，在类加载的解析阶段会将其转化为`java.lang.String`对象对应的`oop`（可以理解为`Java`对象在`HotSpot VM`层面的表示），并使用`StringTable`来进行缓存。但是`CONSTANT_String`类型的常量，属于上文提到的`延迟解析`的范畴，也就是在类加载时并不会立即执行解析，而是等到第一次执行相关指令时（一般来说是`ldc`指令）才会真正解析。

**3. 执行指令**
上面提到，`JVM`会在第一次执行相关指令的时候去执行真正的解析，对于上文给出的代码，观察字节码可以发现，`ldc`指令中使用到了符号引用，所以在执行`ldc`指令时，需要进行解析操作。那么`ldc`指令到底做了什么呢？

`ldc`指令会从`运行时常量池`中查找指定`index`对应的常量项，并将其压入栈中。如果该项还未解析，则需要先进行解析，将符号引用转化为具体的值，然后再将其压入栈中。如果这个未解析的项是`String`类型的常量，则先从`字符串常量池`中查找是否已经有了相同内容的`String`对象，如果有则直接将`字符串常量池`中的该对象压入栈中；如果没有，则会创建一个新的`String`对象加入`字符串常量池`中，并将创建的新对象压入栈中。可见，如果代码中声明多个相同内容的`字符串字面量`或者`字符串常量表达式`，那么只会在第一次执行`ldc`指令时创建一个`String`对象，后续相同的`ldc`指令执行时相应位置的常量已经解析过了，直接压入栈中即可。

总结一下：
1. **在编译阶段，源码中`字符串字面量`或者`字符串常量表达式`转化为了`class文件`的常量池中的`CONSTANT_String`常量项。**
2. **在类加载阶段，`class文件`的`常量池`中的`CONSTANT_String`常量项被存入了`运行时常量池`中，但保存的内容仍然是一个符号引用，未进行解析。**
3. **在指令执行阶段，当第一次执行`ldc`指令时，`运行时常量池`中的`CONSTANT_String`项还未解析，会真正执行解析，解析过程中会创建`String`对象并加入`字符串`常量池。**

## 缓存关键源码分析
可以看到，其实`ldc`指令在解析`String`类型常量的时候与`String.intern()`方法的逻辑很相似：
1. `ldc`指令中解析`String`常量：先从`字符串常量池`中查找是否有相同内容的`String`对象，如果有则将其压入栈中，如果没有，则创建新对象加入`字符串常量池`并压入栈中。
2. `String.intern()`方法：先从`字符串常量池`中查找是否有相同内容的`String`对象，如果有则返回该对象引用，如果没有，则将自身加入`字符串常量池`并返回。

实际在`HotSpot`内部实现上，`ldc`指令 与 `String.intern()`对应的`native`方法 调用了相同的内部方法。我们以`OpenJDK 8`的源代码为例，简单分析一下其过程，代码如下（源码位置：`src/share/vm/classfile/SymbolTable.cpp`）：
```java

// String.intern()方法会调用这个方法
// 参数 "oop string"代表调用intern()方法的String对象
oop StringTable::intern(oop string, TRAPS)
{
  if (string == NULL) return NULL;
  ResourceMark rm(THREAD);
  int length;
  Handle h_string (THREAD, string);
  jchar* chars = java_lang_String::as_unicode_string(string, length, CHECK_NULL);    // 将String对象转化为字符序列
  oop result = intern(h_string, chars, length, CHECK_NULL);
  return result;
}

// ldc指令执行时会调用这个方法
// 参数 "Symbol* symbol" 是 运行时常量池 中 ldc指令的参数（索引位置）对应位置的Symbol对象
oop StringTable::intern(Symbol* symbol, TRAPS) {
  if (symbol == NULL) return NULL;
  ResourceMark rm(THREAD);
  int length;
  jchar* chars = symbol->as_unicode(length);   // 将Symbol对象转化为字符序列
  Handle string;
  oop result = intern(string, chars, length, CHECK_NULL);
  return result;
}

// 上面两个方法都会调用这个方法
oop StringTable::intern(Handle string_or_null, jchar* name, int len, TRAPS) {
  // 尝试从字符串常量池中寻找
  unsigned int hashValue = hash_string(name, len);
  int index = the_table()->hash_to_index(hashValue);
  oop found_string = the_table()->lookup(index, name, len, hashValue);

  // 如果找到了直接返回
  if (found_string != NULL) {
    ensure_string_alive(found_string);
    return found_string;
  }

   // ...... 省略部分代码 ......
   
  Handle string;
  // 尝试复用原字符串，如果无法复用，则会创建新字符串
  // JDK 6中这里的实现有一些不同，只有string_or_null已经存在于永久代中才会复用
  if (!string_or_null.is_null()) {
    string = string_or_null;
  } else {
    string = java_lang_String::create_from_unicode(name, len, CHECK_NULL);
  }

  //...... 省略部分代码 ......

  oop added_or_found;
  {
    MutexLocker ml(StringTable_lock, THREAD);
    // 添加字符串到 StringTable 中
    added_or_found = the_table()->basic_add(index, string, name, len,
                                  hashValue, CHECK_NULL);
  }
  ensure_string_alive(added_or_found);
  return added_or_found;
}
```

## 案例分析
> **说明**：因为在`Java 6`之后`字符串常量池`从`永久代`移到了`堆`中，可能在一些代码上`Java 6`与之后的版本表现不一致。所以下面的代码都使用`Java 6`和`Java 7`分别进行测试，如果未特殊说明，表示在两个版本上结果相同，如果不同，会单独指出。

------
```java
final int a = 4;
int b = 4;
String s1 = "123" + a + "567";
String s2 = "123" + b + "567";
String s3 = "1234567";
System.out.println(s1 == s2);
System.out.println(s1 == s3);
System.out.println(s2 == s3);
```
结果：
```java
false
true
false
```
解释：
1. 第三行，因为`a`被定义为常量，所以`"123" + a + "567"`是一个`常量表达式`，在编译期会被编译为`"1234567"`，所以会在`字符串常量池`中创建`"1234567"`，`s1`指向`字符串常量池`中的`"1234567"`；
2. 第四行，`b`被定义为变量，`"123"`和`"567"`是`字符串字面量`，所以首先在`字符串常量池`中创建`"123"`和`"567"`，然后通过`StringBuilder`隐式拼接在堆中创建`"1234567"`，`s2`指向堆中的`"1234567"`；
3. 第五行，`"1234567"`是一个`字符串字面量`，因为此时`字符串常量池`中已经存在了`"1234567"`，所以`s3`指向字符串`字符串常量池`中的`"1234567"`。

------
```java
String s1 = new String("123");
String s2 = s1.intern();
String s3 = "123";
System.out.println(s1 == s2); 
System.out.println(s1 == s3); 
System.out.println(s2 == s3);
```
结果：
```java
false
false
true
```
解释：
1. 第一行，`"123"`是一个`字符串字面量`，所以首先在`字符串常量池`中创建了一个`"123"`对象，然后使用`String`的构造函数在堆中创建了一个`"123"`对象，`s1`指向堆中的`"123"`；
2. 第二行，因为`字符串常量池`中已经有了`"123"`，所以`s2`指向`字符串常量池`中的`"123"`；
3. 第三行，同样因为`字符串常量池`中已经有了`"123"`，所以`s3`指向`字符串常量池`中的`"123"`。

------
```java
String s1 = String.valueOf("123");
String s2 = s1.intern();
String s3 = "123";
System.out.println(s1 == s2); 
System.out.println(s1 == s3); 
System.out.println(s2 == s3); 
```
结果：
```java
true
true
true
```
解释：与上一种情况的区别在于，`String.valueOf()`方法在参数为`String`对象的时候会直接将参数作为返回值，不会在堆上创建新对象，所以`s1`也指向`字符串常量池`中的`"123"`，三个变量指向同一个对象。

------
```java
String s1 = new String("123") + new String("456"); 
String s2 = s1.intern();
String s3 = "123456";
System.out.println(s1 == s2); 
System.out.println(s1 == s3); 
System.out.println(s2 == s3);
```
上面的代码在`Java 6`和`Java 7`中结果是不同的。
在`Java 6`中：
```java
false
false
true
```
解释：
1. 第一行，`"123"`和`"456"`是`字符串字面量`，所以首先在`字符串常量池`中创建`"123"`和`"456"`，`+`操作符通过`StringBuilder`隐式拼接在堆中创建`"123456"`，`s1`指向堆中的`"123456"`；
2. 第二行，将`"123456"`缓存到`字符串常量池`中，因为`Java 6`中`字符串常量池`中的对象是在永久代创建的，所以会在`字符串常量池`（永久代）创建一个`"123456"`，此时在堆中和永久代中各有一个`"123456"`，`s2`指向`字符串常量池`（永久代）中的`"123456"`；
3. 第三行，`"123456"`是`字符串字面量`，因为此时`字符串常量池`（永久代）中已经存在`"123456"`，所以`s3`指向`字符串常量池`（永久代）中的`"123456"`。

在`Java 7`中：
```java
true
true
true
```
解释：与`Java 6`的区别在于，因为`Java 7`中`字符串常量池`中的对象是在`堆`上创建的，所以当执行第二行`String s2 = s1.intern();`时不会再创建新的`String`对象，而是直接将`s1`的引用添加到`StringTable`中，所以三个对象都指向常量池中的`"123456"`，也就是第一行中在堆中创建的对象。

> `Java 7`下，`s1 == s2`结果为`true`也能够用来佐证我们上面`延迟解析`的过程。我们假设如果`"123456"`不是延迟解析的，而是类加载的时候解析完成并进入常量池的，`s1.intern()`的返回值应该是常量池中存在的`"123456"`，而不会将`s1`指向的堆中的`"123456"`对象加入常量池，所以结果应该是`s2`不等于`s1`而等于`s3`。

------
```java
String s1 = new String("123") + new String("456");
String s2 = "123456";
String s3 = s1.intern();
System.out.println(s1 == s2); 
System.out.println(s1 == s3); 
System.out.println(s2 == s3);
```
结果：
```java
false
false
true
```
解释：
1. 第一行，`"123"`和`"456"`是`字符串字面量`，所以首先在`字符串常量池`中创建`"123"`和`"456"`，`+`操作符通过`StringBuilder`隐式拼接在堆中创建`"123456"`，`s1`指向堆中的`"123456"`；
2. 第二行，`"123456"`是字符串字面量，此时字符串常量池中不存在`"123456"`，所以在`字符串常量池`中创建`"123456"`， `s2`指向`字符串常量池`中的`"123456"`；
3. 第三行，因为此时字符串常量池中已经存在`"123456"`，所以`s3`指向`字符串常量池`中的`"123456"`。


# 参考
1. [Java substring() method memory leak issue and fix](https://www.geeksforgeeks.org/java-substring-method-memory-leak-issue-and-fix/)
2. [java - substring method in String class causes memory leak - Stack Overflow](https://stackoverflow.com/questions/15612157/substring-method-in-string-class-causes-memory-leak)
3. [JLS - 3.10.5. String Literals](https://docs.oracle.com/javase/specs/jls/se8/html/jls-3.html#jls-3.10.5)
4. [JLS - 15.28 Constant Expressions](https://docs.oracle.com/javase/specs/jls/se8/html/jls-15.html#jls-15.28)
5. [String.intern in Java 6, 7 and 8 – string pooling](http://java-performance.info/string-intern-in-java-6-7-8/)
6. [(Java 中new String("字面量") 中 "字面量" 是何时进入字符串常量池的? - 木女孩的回答 - 知乎](https://www.zhihu.com/question/55994121/answer/147296098)
7. [深入解析String#intern](https://tech.meituan.com/2014/03/06/in-depth-understanding-string-intern.html)
8. [JLS - 5.4.3. Resolution](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-5.html#jvms-5.4.3)
9. [请别再拿“String s = new String("xyz");创建了多少个String实例”来面试了吧](https://www.iteye.com/blog/rednaxelafx-774673)
10. [JVM Internals](https://blog.jamesdbloom.com/JVMInternals.html)
11. [探秘JVM内部结构（翻译）](https://wu-sheng.github.io/me/articles/JVMInternals)
12. [Java虚拟机原理图解](https://blog.csdn.net/u010349169/category_9263262.html)