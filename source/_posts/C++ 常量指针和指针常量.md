---
title: C++ 常量指针和指针常量
author: Keaper
tags:
  - C++
categories: 
date: 2017-09-28 21:31:00
---
原文链接：
[http://www.cnblogs.com/beanmoon/archive/2012/09/23/2698987.html](http://www.cnblogs.com/beanmoon/archive/2012/09/23/2698987.html)
先看一段代码：
```cpp
char greeting[] = “Hello”;  
char* p = greeting; //non-const pointer,non-const data  
const char* p = greeting; //non-const pointer,const data;  
char* const p = greeting;//const pointer,non-const data;  
const char* const p = greeting; //const pointer,const data; 
```
关于定义的阅读，一直以为这段解释很不错了已经，即以*和const的相对位置来判断：
const语法虽然变化多端，但并不莫测高深。如果关键字const出现在*左边，表示被指物是常量；如果出现在*右边，表示指针自身是常量；如果出现在*两边，表示被指物和指针两者都是常量。
如果被指物是常量，有些程序员会将关键字const写在类型之前，有些人会把它写在类型之后、*之前。两种写法的意义相同，所以下列两个定义相同：
```cpp
const int* w;  
int const* w;  
```
后来在百度知道上看到另一段挺有意思的解释：
其实很简单，const和*的优先级相同
且是从右相左读的，即“右左法则”
其实const只是限定某个地址存储的内容不可修改
比如:

```cpp
int *p;         //读作p为指针，指向int，所以p为指向int的指针
int *const p;   //读作p为常量，是指针，指向int，所以p为指向int的常量指针， p不可修改
int const *p;   //p为指针，指向常量，为int，所以p为指向int常量的指针， *p不可修改
int ** const p; //p为常量，指向指针，指针指向int，所以p为指向int型指针的常量指针，p不可修改
int const**p;   //p为指针，指向指针，指针指向常量int，所以p为指针，指向一个指向int常量的指针， **p为int，不可修改
int * const *p; //p为指针，指向常量，该常量为指针，指向int，所以p为指针，指向一个常量指针，*p为指针，不可修改
int ** const *p;//p为指针，指向常量，常量为指向指针的指针，p为指针，指向常量型指针的指针，*p为指向指针的指针，不可修改
int * const **p;//p为指针，指向一个指针1，指针1指向一个常量，常量为指向int的指针，即p为指针，指向一个指向常量指针的指针， **p为指向一个int的指针，不可修改

```
