---
title: 第六章 接口、lambda 表达式与内部类
author: Keaper
tags:
  - JAVA
categories:
date: 2018-08-12 00:37:00
---
#### 接口
1. 接口中的方法默认属于`public`，声明接口时，不必提供`public`关键字，但是实现接口时必须指定`public`关键字。
2. 接口中不能包含成员变量，但是可以包含常量，接口中的域自动为`public static final`,不必指定关键字。
3. 接口不是类，尤其不能使用`new`运算符实例化一个接口，尽管不能构造接口的对象，却能声明接口的变量，接口变量必须引用实现了该接口的对象。可以使用`instanceof`检查一个对象是否实现了某个接口。
4. 一个类可以实现多个接口，但只能继承一个类。JAVA不可以多重继承。
5. java 8 中允许在接口中增加静态方法，还可以为接口提供一个默认实现，用`default`修饰符标记方法即可。
6. 默认方法冲突。如果先在一个接口中将一个方法定义为默认方法 , 然后又在子类或另一个接口中定义了同样的方法, 会发生什么情况?
	- 超类优先原则。如果子接口中定义了与父接口中同名且参数类型相同的默认方法，那么父接口中的默认方法会被忽略。
    - 接口冲突。如果一个接口提供了一个默认方法 , 另一个接口提供了一个同名而且参数类型 ( 不论是否是默认参数 ) 相同的方法，而一个类同时实现这两个接口，那么这个类必须覆盖这个方法。**注意这里只要有一个方法是默认方法，就会存在冲突，如果两个都不是默认方法，则不存在冲突。**
7. 一个类继承了一个父类,同时实现了一个接口 , 并从父类和接口继承了相同的方法。这种情况遵循“类优先”原则，接口中的默认方法会被忽略。


#### 对象克隆、Cloneable接口
1. Java默认的克隆操作时浅拷贝，它并没有克隆包含在对象中的内部对象。
2. `Cloneable`是一个标记接口，接口中不包含方法，使用它的唯一目的是可以用`instanceof`进行类型检查。`clone()`方法是从`Object`中继承而来的，并不是`Cloneable`接口的方法。
3. 所有数组类型都有一个`public`的`clone`方法，可以用这个方法建立一个新数组,包含原数组所有元素的副本。
```java
int [] num = {1,2,3,4};
int [] clone = num.clone();
clone[3] = 5;
System.out.println(num[3]); // output:4
```
4. 如果一个类想要实现深拷贝，需要实现`Cloneable`接口并重新定义`clone`方法，并指定public访问修饰符。指定`public`修饰符的原因是`Object`类中的`clone()`是`protected`的，只有子类才可以调用，所以必须重新定义`clone()`为`public`才能允许所有方法克隆对象。

#### lambda表达式
1. lambda表达式语法
```java
String [] ss = new String []{"54321","9876","123"};
Arrays.sort(ss,(a,b) -> a.length() - b.length());
System.out.println(Arrays.toString(ss));//output：[123, 9876, 54321]
```
2. 如果lambda 表达式没有参数， 仍然要提供空括号。
3. 如果可以推导出一个lambda表达式的参数类型，则可以忽略其类型。
```java
Comparator<String> comp = (first, second) // Same as (String first, String second)
        -> first.length() - second.length();
```
4. 如果方法只有一参数，而且这个参数的类型可以推导得出，那么甚至还可以省略小括号。
```java
ActionListener listener =  event ->
System.out.println("The time is " + new Date());
// Instead of (event) -> . . . or (ActionEvent event) -> . . .
```
5. 对于只有一个抽象方法的接口，需要这种接口的对象时就可以提供一个lambda表达式。这种接口称为函数式接口(functional interface)。
6. Java中,lambda表达式可以转换为函数式接口，需要函数式接口的地方就可以用lambda表达式。
7. 方法引用。主要有三种情况：

> object::instance Method  
> Class::static Method  
> Class::instance Method  

    在前2 种情况中,方法引用等价于提供方法参数的lambda表达式。
    
    `System.out::println`等价于`x ->System.out.println(x)`。 
    
    `Math::pow`等价于`(x，y) ->Math.pow(x, y)`。
    
    对于第3 种情况，第1个参数会成为方法的目标。
    `String::compareToIgnoreCase` 等同于`(x, y) -> x.compareToIgnoreCase(y)`

8. 构造器引用。构造器引用与方法引用很类似，只不过方法名为new。
9. lambda 表达式有3个部分：
    - 一个代码块；
    - 参数;
    - 自由变量的值，这是指非参数而且不在代码中定义的变量。

10. lambda 表达式中捕获的变量必须实际上是`final`变量(`effectively final`),意思是这个变量初始化之后就不会再为它赋新值。
11. lambda 表达式的代码块与嵌套块有相同的作用域。


#### 内部类
1. 内部类既可以访问自身的数据域，也可以访问创建它的外围类对象的数据域.
2. 内部类的对象总有一个隐式引用，它指向了创建它的外部类对象。
3. 内部类中声明的所有静态域都必须是`final`。原因很简单。我们希望一个静态域只有一个实例，不过对于每个外部对象， 会分别有一个单独的内部类实例。如果这个域不是final , 它可能就不是唯一的。
4. 内部类不能有`static`方法。
5. 编译器将会把内部类翻译成用$分隔外部类名与内部类名的常规类文件。
6. 可以在方法汇中定义局部内部类，作用域仅限于声明这个局部类的中，对外部不可见，同时也不可以用`public`或者`private`修饰。
7. 局部内部类不仅可以访问外围类的变量，还可以访问局部变量。不过，那些局部变量必须事实上为final。事实上为`final`的意思与上面lambda表达式访问捕获的变量意思相同。
8. 匿名内部类。语法格式为
```java
new SuperType(construction parameters)
{
inner class methods and data
}
```
`SuperType`可以是`ActionListener`这样的接口，内部类就要实现这个接口。`SuperType`也可以是一个类，内部类就要扩展它。
9. 匿名内部类不能有构造器。而是要将构造方法传递给父类构造器。
```
Person queen = new Person("Mary") ;
// 一个Person对象
Person count = new Person("Dracula") { . . .
// 一个继承自Person的匿名内部类
    }
```
10. 双括号初始化实际是内部类语法。
```
invite(new ArrayList<String>0 {{ add("Harry") ; add("Tony") ; }}) ;
```
外层括号建立了`ArrayList`的一个匿名子类。内层括号则是一个对象构造块。
11. 使用内部类只是为了把一个类隐藏在另外一个类的内部，并不需要内部类引用外围类对象。为此，可以将内部类声明为`static`,以便取消产生的引用。静态内部类的对象除了没有对生成它的外围类对象的引用特权外，与其他所冇内部类完全一样。
