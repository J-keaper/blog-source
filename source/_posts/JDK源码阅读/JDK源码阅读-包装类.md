---
title: JDK源码阅读-包装类
author: Keaper
tags:
  - JAVA
categories: 
date: 2019-06-15 15:20:00
---

# 概览
JAVA中的基本类型有八种，分别是：byte,short,int,long,float,double,char,boolean。
在JDK中都有对应的包装类，均在java.lang包中，分别是：Byte,Short,Integer,Long,Float,Double,Boolean,Character。
> 以下源码基于JDK1.8。

这几个类型的继承关系如图：
![https://blog-picture.nos-eastchina1.126.net/picture0011.jpg](https://blog-picture.nos-eastchina1.126.net/picture0011.jpg)

# Number
Byte,Short,Integer,Long,Float,Double这几个表示数字的类均继承自Number类。
Number类是一个 `abstract`类，Numbr类有以下方法：
```java
public abstract int intValue();
public abstract long longValue();
public abstract float floatValue();
public abstract double doubleValue();
public byte byteValue() {
	return (byte)intValue();
}
public short shortValue() {
	return (short)intValue();
}
```
这几个方法分别对应了每种类型向其他类型转换的方法。`byteValue`，`shortValue`这两个方法是直接由`intvalue()`返回值强制转换过来的。

系列文章：
[JDK源码阅读-Integer](../JDK源码阅读-Integer)