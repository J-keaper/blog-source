---
title: 'Java中的遍历（遍历集合或数组的几种方式）'
date: 2020-08-19 15:18:17
tags: [JAVA]
published: true
hideInList: false
feature: 
isTop: false
---
本文主要总结了`Java`中遍历集合或数组的几种方式，并介绍了各种遍历方式的实现原理，以及一些最佳实践。最后介绍了`Java`集合类迭代器的快速失败（`fail-fast`）机制。
<!-- more -->

# Java中的循环结构
遍历必然需要使用到循环结构，`Java`中有以下几种循环结构：
1. `while`语句
2. `do...while`语句
3. 基本`for`语句
4. 增强`for`语句

对于`do...while`语句，其第一个循环体是必须会执行的，这对于空集合或者空数组是不适用的。所以我们一般不会使用`do...while`语句来进行遍历。其余三种都是我们经常用来遍历的语法结构。

> `for`语句和`while`语句在一般情况下可以互相转化，下文我们并不将两种语句单独区分，会根据场景选择使用更加简单的方式。

# 遍历数组
## 1. 使用下标遍历
使用循环遍历数组下标范围，在循环体中用下标访问数组元素：
```java
for (int i = 0; i < array.length; i++) {
    System.out.println(array[i]);
}
```
## 2. 增强for循环
使用增强`for`循环语法结构来对数组进行遍历：
```java
for (int value : array) {
    System.out.println(value);
}
```
增强`for`循环 其实只是一种语法糖，使用 增强`for`循环 在遍历数组时，在编译过程会将其转化为 "使用下标遍历" 的方式，在字节码层面其实等价于第一种方式，效率上也没有太大差别。

关于增强`for`循环语法更详细的介绍，请移步：[Java Language Specification - 14.14.2. The enhanced for statement](https://docs.oracle.com/javase/specs/jls/se8/html/jls-14.html#jls-14.14.2)

# JAVA中的迭代器
在 面向对象编程里，迭代器模式是一种设计模式，是一种最简单也最常见的设计模式。迭代器模式提供了一种方法顺序访问一个集合对象中的各个元素，而又不暴露其内部的表示。`Java`中也提供了对迭代器模式的支持，主要是针对`Java`的各种集合类进行遍历。

`Iterator`接口是`Java`中对迭代器的抽象接口定义，其定义如下：
```java
public interface Iterator<E> {
    // 是否还有下一个元素
    boolean hasNext();
    // 返回下一个元素
    E next();
    // 删除迭代过程中最近访问的一个元素
    // 也就是在next()之后调用remove()删除刚刚next()返回的元素
    default void remove() {
        throw new UnsupportedOperationException("remove");
    }
}
```
`Iterable`接口是在`Java 1.5`中引入的，为了用来支持增强型`for`循环，只有实现了`Iterable`接口的对象才可以使用增强型`for`循环。`Iterable`接口定义如下：
```java
public interface Iterable<T> {
    // 返回迭代器
    Iterator<T> iterator();
    // 使用函数式接口对增强型for循环进行包装，可以方便地使用lambda表达式来进行遍历
    default void forEach(Consumer<? super T> action) {
        Objects.requireNonNull(action);
        for (T t : this) {
            action.accept(t);
        }
    }
}
```
可以看到`Iterable`接口提供了`forEach`方法的默认实现，函数参数是一个函数式接口`action`参数来表示对遍历到的每个元素的操作行为，实现逻辑是使用 增强`for`循环 遍历自身，循环中对每个元素都应用 参数`action`所表示的操作行为。

# 遍历List
## List接口的定义
`List`表示的是一个有序的元素集合。`List`接口继承了`Collection`接口，`Collection`接口又继承了`Iterable`接口，其定义如下：
```java
public interface Iterable<T> {
    // 返回Iterator迭代器
    Iterator<T> iterator();
}
public interface Collection<E> extends Iterable<E> {
}
public interface List<E> extends Collection<E> {
    // 获取指定位置的元素
    E get(int index);
    // 获取ListIterator迭代器
    ListIterator<E> listIterator();
    // 从指定位置获取ListIterator迭代器
    ListIterator<E> listIterator(int index);
}
```
可以看到`List`除了可以通过`iterator()`方法获得`Iterator`迭代器之外，还可以通过`listIterator()`方法获得`ListIterator`迭代器。`ListIterator`迭代器相比于`Iterator`迭代器之外，访问和操作元素的方法更加丰富：
1. `Iterator`只能向后迭代，而`ListIterator`可以向两个方向迭代
2. `Iterator`只能在迭代过程中删除元素，而`ListIterator`可以添加元素、删除元素、修改元素。
```java
public interface ListIterator<E> extends Iterator<E> {
    // Query Operations
    // 是否还有后一个元素
    boolean hasNext();
    // 访问后一个元素
    E next();
    // 是否还有前一个元素
    boolean hasPrevious();
    // 访问前一个元素
    E previous();
    // 后一个元素的下标
    int nextIndex();
    // 前一个元素的下标
    int previousIndex();

    // Modification Operations
    // 删除元素
    void remove();
    // 修改元素
    void set(E e);
    // 添加元素      
    void add(E e);
}
```

## List的遍历方法
### 1. 使用下标遍历
`List`接口提供了`get`方法来访问指定位置的元素，所以与遍历数组一样，`List`也可以通过遍历`List`下标并使用`get`方法访问元素来遍历`List`：
```java
for (int i = 0; i < list.size(); i++) {
    System.out.println(list.get(i));
}
```
### 2. 使用Iterator迭代器
`List`可以使用继承自`Iterable`接口的`iterator()`方法来获得`Iterator`迭代器，使用`Iterator`迭代器来遍历`List`：
```java
Iterator<Integer> iterator = list.iterator();
while (iterator.hasNext()){
    System.out.println(iterator.next());
}
```
### 3. 使用ListIterator迭代器
`List`还可以使用`listIterator()`方法来获得`ListIterator`迭代器，使用`ListIterator`迭代器来遍历`List`：
```java
ListIterator<Integer> listIterator = list.listIterator();
while (listIterator.hasNext()){
    System.out.println(listIterator.next());
}
// ListIterator 可以向前遍历
listIterator = list.listIterator(list.size() - 1);
while (listIterator.hasPrevious()){
    System.out.println(listIterator.previous());
}
```
### 4. 增强for循环
我们前面说到，实现了`Iterable`接口的对象可以使用增强`for`循环遍历。`List`接口继承自`Iterable`接口，所以可以使用增强`for`循环来遍历`List`：
```java
for (Integer value : list) {
    System.out.println(value);
}
```
增强性`for`循环是一种语法糖，但是与遍历数组不一样的是，使用增强型`for`循环在遍历实现了`Iterable`接口的对象时，会在编译过程中将其转化为使用`Iterator`迭代器进行遍历的方式。所以这种方式本质上与上一种方式是一样的。

关于增强`for`循环语法更详细的介绍，请移步：[Java Language Specification - 14.14.2. The enhanced for statement](https://docs.oracle.com/javase/specs/jls/se8/html/jls-14.html#jls-14.14.2)

### 5. Iterable接口的forEach方法
List接口实现了`Iterable`接口，`Iterable`接口中提供了`forEach`方法来更加方便的遍历集合。其参数是一个函数式接口`action`，来表示对遍历到的每个元素的操作行为。并且因为参数是一个函数式接口，所以我们可以使用`lamdba`表达式更简洁的表达遍历过程。

`Iterable`接口的`forEach`方法的默认实现是使用增强`for`循环来遍历自身。所以如果没有重写`forEach`方法的话，这种方式本质上与上一种方式是一样的。
```java
list.forEach(value -> System.out.println(value));
list.forEach(System.out::println);
```

## 最佳实践
上面几种方式其实本质上来讲只有两种方式：
1. 使用循环遍历集合的下标范围，配合`get`方法获取集合元素 来遍历`List`。
2. 使用迭代器（`Iterator`或`ListIterator`）来遍历`List`。

其余方式都只不过是第二种方式的语法糖或其变种。
那么到底应该使用哪种方式更好呢？这取决于`List`的内部实现方式。

`List`的常用实现数据结构有两种，`数组`和`链表`：
1. 对于数组实现的`List`及其对应的`Iterator`/`ListIterator`实现来说，比如`ArrayList`、`Vector`。`List.get()`方法、`Iterator.next()`方法、`ListIterator.next()`方法、`ListIterator.previous()`方法的时间复杂度都为`O(1)`。所以使用下标遍历所有元素的时间复杂度为`O(N)`，使用迭代器遍历所有元素的时间复杂度也为`O(N)`。但是下标遍历的方式执行的代码更少更简单，所以效率稍高。
2. 而对于链表实现的`List`其对应的`Iterator`/`ListIterator`实现来说，`List.get()`方法的时间复杂度为`O(N)`，`Iterator.next()`方法、`ListIterator.next()`方法、`ListIterator.previous()`方法的时间复杂度都为`O(1)`。所以使用下标遍历所有元素的时间复杂度为`O(N*N)`，使用迭代器遍历所有元素的时间复杂度为`O(N)`。所以使用迭代器遍历效率更高。

`Java`集合框架中，提供了一个`RandomAccess`接口，该接口没有方法，只是一个标记。通常被`List`接口的实现类使用，用来标记该`List`的实现类是否支持`Random Access`。一个集合类实现了该接口，就意味着它支持`Random Access`，按位置读取元素的平均时间复杂度为`O(1)`，比如`ArrayList`。而没有实现该接口的，就表示不支持`Random Access`，比如`LinkedList`。所以推荐的做法就是，如果想要遍历一个`List`，**如果其实现了`RandomAccess`接口，那么使用下标遍历效率更高，否则的话使用迭代器遍历效率更高**。

# 遍历Set
## Set接口的定义
```java
public interface Iterable<T> {
    // 返回Iterator迭代器
    Iterator<T> iterator();
}
public interface Collection<E> extends Iterable<E> { }
public interface Set<E> extends Collection<E> { }
```
## Set的遍历方法
相较于`List`，`Set`是无序的，所以`Set`没有通过下标获取元素的`get`方法，也就没办法使用下标来遍历。`Set`也没有类似`ListIterator`一样特殊的迭代器。所以遍历`Set`只能使用`Iterator`迭代器来遍历。下面三种方式其实本质上都是使用`Iterator`迭代器来遍历，后两种方式只是第一种方式的语法糖或者变种。
### 1. 使用Iterator迭代器
`Set`接口同样继承了`Iterable`的`iterator()`方法，可以使用其返回的`Iterator`迭代器来遍历`Set`：
```java
Iterator<Integer> iterator = set.iterator();
while (iterator.hasNext()){
    System.out.println(iterator.next());
}
```
### 2. 增强for循环
`Set`接口实现了`Iterable`接口，所以也可以使用增强型`for`循环来遍历`Set`：
```java
for (Integer value : set) {
    System.out.println(value);
}
```

### 3. Iterable接口的forEach方法
`Set`接口实现了`Iterable`接口，所以也可以使用`forEach`方法来遍历`Set`：
```java
set.forEach(value -> System.out.println(value));
set.forEach(System.out::println);
```

# 遍历Map
不同于`List`，`Map`并不是一组元素的集合，而是一组键值对，所以`Map`没有继承`Collection`、`Iterable`等其他接口。
## Map接口的定义
```java
public interface Map<K,V> {
    // 返回Map中键的集合
    Set<K> keySet();
    // 返回Map中值的集合
    Collection<V> values();
    // 返回Map中键值对的集合
    Set<Map.Entry<K, V>> entrySet();
    // 类似于Iterable接口的forEach方法
    default void forEach(BiConsumer<? super K, ? super V> action) {
        Objects.requireNonNull(action);
        for (Map.Entry<K, V> entry : entrySet()) {
            K k;
            V v;
            try {
                k = entry.getKey();
                v = entry.getValue();
            } catch(IllegalStateException ise) {
                // this usually means the entry is no longer in the map.
                throw new ConcurrentModificationException(ise);
            }
            action.accept(k, v);
        }
    }
}
```
`Map`提供了`keySet()`、`values()`、`entrySet()`方法来分别获取`Map`的 键集合、值集合、键值对集合。并且提供了类似于`Iterable`的`forEach`方法及其默认实现。

## Map的遍历方法
遍历`Map`可以通过先获取其 键集合、值集合、键值对集合，然后根据返回的集合类型选择不同的遍历方式。

同时，`Map`也提供了类似于`Iterable`的`forEach`方法，参数`action`是一个函数式接口，指定了对于每一个键值对的操作行为，实现逻辑是使用增强`for`循环遍历`Map`的`entrySet()`方法的返回值，对于每一个遍历到的每一个键值对，应用参数`action`代表的操作行为。

# Java集合中的快速失败机制
 ## 何为快速失败
[wikipedia](https://en.wikipedia.org/wiki/Fail-fast)上对`快速失败（fail-fast）`的介绍是：
 >In systems design, a fail-fast system is one which immediately reports at its interface any condition that is likely to indicate a failure. Fail-fast systems are usually designed to stop normal operation rather than attempt to continue a possibly flawed process. 

简单来说就是系统运行中，如果有错误发生，那么系统立即结束，而不是继续冒不确定的风险继续执行，这种设计就是"**快速失败**"。

 ## Java集合迭代器的快速失败机制
`Java`集合框架中的一些集合类的迭代器也是被设计为`快速失败`的。集合迭代器中的快速失败机制是说：
> 在使用迭代器遍历集合时，如果迭代器创建之后，通过 除了迭代器提供的修改方法之外 的其他方式对集合进行了结构性修改（添加、删除元素等），那么迭代器应该抛出一个`ConcurrentModificationException`异常，表示在此次遍历中集合发生了"并发修改"，应该提前终止迭代过程。因为在迭代器遍历集合的过程中，如果有别的行为改变了集合本身的结构，那么迭代器之后的行为可能就是不符合预期的，可能会出现错误的结果，所以提前检测并抛出异常是一个更好的做法。

`Java`中大部分基本的集合类的迭代器都实现了快速失败机制，包括`ArrayList`，`LinkedList`，`Vector`，`HashMap`，`HashSet`等等。但是对于并发集合类，例如`ConcurrentHashMap`，`CopyOnWriteArrayList`等，这些类本身就是设计来支持并发的，是线程安全的，所以也不存在快速失败这一说。

以`ArrayList`为例，[ArrayList (Java Platform SE 8 )](https://docs.oracle.com/javase/8/docs/api/java/util/ArrayList.html)中对`fail-fast`的介绍如下：
> The iterators returned by this class's iterator and listIterator methods are fail-fast: if the list is structurally modified at any time after the iterator is created, in any way except through the iterator's own remove or add methods, the iterator will throw a ConcurrentModificationException. Thus, in the face of concurrent modification, the iterator fails quickly and cleanly, rather than risking arbitrary, non-deterministic behavior at an undetermined time in the future.

> `ArrayList`的`iterator`和`listIterator`方法返回的迭代器都是`fail-fast`的：如果在迭代器创建之后，通过除了迭代器自身的`add`/`remove`方法之外的其他方式，对`ArrayList`进行了结构性修改（添加、删除元素等），那么该迭代器应该抛出一个`ConcurrentModificationException`异常。所以，迭代器在面对并发修改时，迭代器将快速而干净地失败，而不是冒着在未来不确定的时间发生不确定行为的风险继续执行。

 ## 典型实践
如果看过阿里的《JAVA开发手册》，会知道里面有这一条规范：
>【强制】不要在 `foreach` 循环里进行元素的 `remove`/`add` 操作。`remove` 元素请使用 `Iterator`方式，如果并发操作，需要对 `Iterator` 对象加锁。

上面说的`foreach`循环指的上面我们提到的使用增强`for`循环进行遍历的方式。这种方式本质上使用的是迭代器方式。所以更明确一点，这个规范其实就是在说：
> 在使用`Iterator`迭代器遍历集合过程中，不要通过集合本身的`remove`/`add`方法来进行元素的 `remove`/`add` 操作，`remove`请使用`Iterator`提供的`remove`方法。

**正确**方式：
```java
List<Integer> list = new ArrayList<>();
list.add(1);
list.add(2);
list.add(3);
Iterator<Integer> iterator = list.iterator();
while (iterator.hasNext()) {
    Integer item = iterator.next();
    if (item == 1) {
        iterator.remove();
    }
}
list.forEach(System.out::println); // 输出 2 3 
```
**错误**方式：
```java
List<Integer> list = new ArrayList<>();
list.add(1);
list.add(2);
list.add(3);
for (Integer item : list) {  // 会在这里抛出异常
    if (item == 1) {
        list.remove(item);  // 使用 list.remove 删除
    }
}
list.forEach(System.out::println);
```
上述代码会在第五行抛出`java.util.ConcurrentModificationException`异常。因为其本质还是使用迭代器遍历，所以为了方便理解其原因，我们将上述方式改写为：
```java
List<Integer> list = new ArrayList<>();
list.add(1);
list.add(2);
list.add(3);
Iterator<Integer> iterator = list.iterator();
while (iterator.hasNext()) {
    Integer item = iterator.next();  // 会在这里抛出异常
    if (item == 1) {
        list.remove(item);  // 使用 list.remove 删除
    }
}
list.forEach(System.out::println);
```
这段代码会在第`7`行抛出`java.util.ConcurrentModificationException`异常。与第一种方式的差别仅仅在于使用了`List.remove`方法而不是`Iterator.remove`方法。

## 快速失败的实现原理
以`ArrayList`为例来说明快速失败机制的实现原理，其他类的实现方式也是大致相同的。
1. `ArrayList`中有一个属性`modCount`，是一个记录`ArrayList`修改次数的计数器。
2. `ArrayList`的`iterator`/`listIterator`方法被调用时，会创建出`Iteartor`/`ListIterator`的实现类 `ArrayList.Itr`/`ArrayList.ListItr`迭代器对象，其中有一个属性`expectedModCount`。当迭代器被创建时，`ArrayList`本身的`modCount`将被复制给`Itr`/`ListItr`中的`expectedModCount`属性。
3. 当使用迭代器的`remove`/`add`方法增删元素时，在修改`ArrayList`的`modCount`之后，还会将其值复制给迭代器自身的`expectedModCount`。而通过`ArrayList`的`remove`/`add`方法增删元素时，仅仅修改了`ArrayList`的`modCount`。
4. 当迭代器执行`next`/`remove`/`add`/`set`操作时是会检查迭代器自身的`expectedModCount`与`ArrayList`的`modCount`是否相等，如果不相等则会抛出`ConcurrentModificationException`异常。

所以在迭代过程中，如果只使用迭代器的`remove`/`add`方法增删元素，是不会出现问题的，因为在增删元素之后迭代器始终会将`ArrayList`的`modCount`值赋值给迭代器自身的`expectedModCount`，所以下次迭代两者一定相等。而如果迭代过程中使用了`ArrayList`的`remove`/`add`方法增删元素，或者有另外一个迭代器进行了增删元素，就会造成`ArrayList`中的`modCount`与迭代器中的`expectedModCount`不一致，抛出`ConcurrentModificationException`异常。

## 快速失败的"bug"
快速失败机制也不能够保证百分之百生效，例如，在下面这段代码中，使用迭代器遍历`ArrayList`过程中，使用`ArrayList`的`remove`方法删除倒数第二个元素，程序能够正确删除，并不会像我们上面所说的抛出`ConcurrentModificationException`异常。
```java
List<Integer> list = new ArrayList<>();
list.add(1);
list.add(2);
list.add(3);
Iterator<Integer> iterator = list.iterator();
while (iterator.hasNext()) {
    Integer item = iterator.next();
    if (item == 2) {
        list.remove(item);
    }
}
list.forEach(System.out::println);  // 输出 1 3
```
这个"`bug`"发生的原因在于，在第二次执行`iterator.next()`后，迭代器记录的下一次将要访问的下标应该是`2`，而在执行`list.remove()`删除元素后，`list`的`size`变为了`2`，所以在下次执行`iterator.hasNext()`时认为已经没有元素要继续迭代了，返回`false`，结束循环。

所以`fail-fast`机制并不能够完全保证所有的并发修改的情况都抛出`ConcurrentModificationException`异常，在程序中也不应该依赖于这个异常信息。
在[ArrayList (Java Platform SE 8 )](https://docs.oracle.com/javase/8/docs/api/java/util/ArrayList.html)中也指出了`fail-fast`这一性质：
> Note that the fail-fast behavior of an iterator cannot be guaranteed as it is, generally speaking, impossible to make any hard guarantees in the presence of unsynchronized concurrent modification. Fail-fast iterators throw ConcurrentModificationException on a best-effort basis. Therefore, it would be wrong to write a program that depended on this exception for its correctness: the fail-fast behavior of iterators should be used only to detect bugs.

# 参考
1. [The Java Language Specification, Java SE 8 Edition](https://docs.oracle.com/javase/specs/jls/se8/html/jls-14.html#jls-14.14.2)
2. [Java遍历集合的几种方法分析（实现原理、算法性能、适用场合）](https://blog.csdn.net/HJF_HUANGJINFU/article/details/51220253#t23)
3. [ArrayList (Java Platform SE 8 )](https://docs.oracle.com/javase/8/docs/api/java/util/ArrayList.html)
