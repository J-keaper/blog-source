title: Java 如何处理未捕获异常
author: Keaper
tags:
  - JAVA
categories: []
date: 2019-09-06 17:13:00
---
# 问题
如果代码中发生了异常，但我们没有用`try/catch`捕获，JVM会如何处理?
这是一段肯定会发生异常的代码，但我们没有处理异常，运行这个代码会发生什么？
```java
public class Main {
    public static void main(String[] args) {
        System.out.println(1 / 0);
        System.out.println("test");
    }
}
```
程序结束，并且控制台输出下面的内容：
```
Exception in thread "main" java.lang.ArithmeticException: / by zero
	at com.xxx.xxx.web.Main.main(Main.java:6)
```
这是如何背后的逻辑是什么，为什么会输出这些内容呢？

# JAVA异常体系
首先来复习一下`JAVA`的异常体系。
## 异常分类
异常继承结构大致如下：
![](https://user-gold-cdn.xitu.io/2018/5/16/163684dec571c6a3?imageslim)

JAVA中所有的异常都是继承自`Throwable`类。可以分为以下两类：
1. 非检查异常(`unchecked exceptions`)，包括以下两种：
    - 错误(`Error`)，包括`Error`类及其子类。这种异常是在正常情况下，不大可能出现的情况，绝大部分的`Error`都会导致程序（比如JVM自身）处于非正常的、不可恢复状态。既然是非正常情况，所以不便于也不需要捕获，常见的比如`OutOfMemoryError`类。
    - 运行时异常(`RuntimeException`)，包括`RuntimeException`类及其子类。这种异常通常是可以通过编码避免的逻辑错误，具体根据需要来判断是否需要捕获，并不会在编译期强制要求。
2. 受检查异常(`checked exception`)，除了上面两种（`Error`类及其子类，`RuntimeException`类及其子类），其他异常都属于受检查异常。这种异常通常是外部错误，不是代码逻辑的错误，编译器强制要求对这种异常进行处理，比如网络连接错误会抛出`IOException`,我们应该提前预料这种情况并对其进行处理(比如重试)。

## 异常处理
JAVA中处理异常的方式有两种：
1. 使用`try/catch`捕获异常并进行处理。
2. 使用`throws`关键字，在方法上声明可能会抛出的异常，由外层调用者去处理这个异常。

上面两种异常中，**受检查异常**必须被捕获，否则会编译失败，而**非检查异常**在编译期不强制要求被捕获。

# 未捕获异常
如果一个**非检查异常**没有被捕获处理，那这就是未捕获异常。因为**受检查异常**都必须在代码中捕获进行处理，所以未捕获异常实际上都是在说**非检查异常**。

在探究如何处理未捕获异常之前先来看一个接口，这个接口是处理未捕获异常的关键接口：
## Thread.UncaughtExceptionHandler接口
```java
@FunctionalInterface
public interface UncaughtExceptionHandler {
    void uncaughtException(Thread t, Throwable e);
}
```
这个接口很简单，只有一个方法，用来处理未捕获的异常，参数是线程信息，以及异常信息，

下面从源码层面来看看如何处理未捕获异常。
## 未捕获异常处理流程
Thread类中有一个`dispatchUncaughtException`方法,这个方法的作用是分发异常信息到正确的`UncaughtExceptionHandler`。当线程运行中出现了未捕获的异常，JVM会调用线程的这个方法，来寻找一个`UncaughtExceptionHandler`处理异常。
```java
/**
* Dispatch an uncaught exception to the handler. This method is
* intended to be called only by the JVM.
*/
private void dispatchUncaughtException(Throwable e) {
    getUncaughtExceptionHandler().uncaughtException(this, e);
}
```
`getUncaughtExceptionHandler`的获取逻辑是，如果此线程的`uncaughtExceptionHandler`属性不为`null`,则分发异常到线程自己的`uncaughtExceptionHandler`，否则将异常分发给此线程所在的线程组。
```java
/**
* Returns the handler invoked when this thread abruptly terminates
* due to an uncaught exception. If this thread has not had an
* uncaught exception handler explicitly set then this thread's
* <tt>ThreadGroup</tt> object is returned, unless this thread
* has terminated, in which case <tt>null</tt> is returned.
* @since 1.5
* @return the uncaught exception handler for this thread
*/
public UncaughtExceptionHandler getUncaughtExceptionHandler() {
    return uncaughtExceptionHandler != null ?
        uncaughtExceptionHandler : group;
}
```
分别来看下两种方式：
### 线程自己处理
`Thread`类有一个`uncaughtExceptionHandler`属性，表示这个线程当这个线程发生未捕获异常时的处理器，可以通过`Thread.setUncaughtExceptionHandler`方法来设置这个属性。如果没有显式调用此方法设置，那么`uncaughtExceptionHandler`属性默认为`null`。
```java
// null unless explicitly set
private volatile UncaughtExceptionHandler uncaughtExceptionHandler;

/**
* Set the handler invoked when this thread abruptly terminates
* due to an uncaught exception.
* <p>A thread can take full control of how it responds to uncaught
* exceptions by having its uncaught exception handler explicitly set.
* If no such handler is set then the thread's <tt>ThreadGroup</tt>
* object acts as its handler.
* @param eh the object to use as this thread's uncaught exception
* handler. If <tt>null</tt> then this thread has no explicit handler.
* @throws  SecurityException  if the current thread is not allowed to
*          modify this thread.
* @see #setDefaultUncaughtExceptionHandler
* @see ThreadGroup#uncaughtException
* @since 1.5
*/
public void setUncaughtExceptionHandler(UncaughtExceptionHandler eh) {
    checkAccess();
    uncaughtExceptionHandler = eh;
}
```

### 交给线程组处理
如果没有设置线程的`uncaughtExceptionHandler`属性或者为`null`，则会将异常信息分发给线程所在的线程组。上面代码可以将`group`作为结果返回是因为所有线程组的父类`ThreadGroup`类实现了`Thread.UncaughtExceptionHandler`接口。
如果当前线程的线程组重写了`uncaughtException`方法，会调用重写的`uncaughtException`方法，否则调用`ThreadGroup`类的`uncaughtException`方法。
下面是这个`ThreadGroup`类的`uncaughtException`方法的实现：
```java
/**
* Called by the Java Virtual Machine when a thread in this
* thread group stops because of an uncaught exception, and the thread
* does not have a specific {@link Thread.UncaughtExceptionHandler}
* installed.
* <p>
* The <code>uncaughtException</code> method of
* <code>ThreadGroup</code> does the following:
* <ul>
* <li>If this thread group has a parent thread group, the
*     <code>uncaughtException</code> method of that parent is called
*     with the same two arguments.
* <li>Otherwise, this method checks to see if there is a
*     {@linkplain Thread#getDefaultUncaughtExceptionHandler default
*     uncaught exception handler} installed, and if so, its
*     <code>uncaughtException</code> method is called with the same
*     two arguments.
* <li>Otherwise, this method determines if the <code>Throwable</code>
*     argument is an instance of {@link ThreadDeath}. If so, nothing
*     special is done. Otherwise, a message containing the
*     thread's name, as returned from the thread's {@link
*     Thread#getName getName} method, and a stack backtrace,
*     using the <code>Throwable</code>'s {@link
*     Throwable#printStackTrace printStackTrace} method, is
*     printed to the {@linkplain System#err standard error stream}.
* </ul>
* <p>
* Applications can override this method in subclasses of
* <code>ThreadGroup</code> to provide alternative handling of
* uncaught exceptions.
*
* @param   t   the thread that is about to exit.
* @param   e   the uncaught exception.
* @since   JDK1.0
*/
public void uncaughtException(Thread t, Throwable e) {
    if (parent != null) {
        parent.uncaughtException(t, e);
    } else {
        Thread.UncaughtExceptionHandler ueh =
            Thread.getDefaultUncaughtExceptionHandler();
        if (ueh != null) {
            ueh.uncaughtException(t, e);
        } else if (!(e instanceof ThreadDeath)) {
            System.err.print("Exception in thread \""
                                + t.getName() + "\" ");
            e.printStackTrace(System.err);
        }
    }
}
```
处理流程如下：
1. 首先，如果这个线程组有父线程组(`parent`属性)，将会调用父线程组的`uncaughtException`方法处理。
2. 否则，先调用`Thread.getDefaultUncaughtExceptionHandler()`检查是否有一个默认的`UncaughtExceptionHandler`,如果有，交给这个默认的`UncaughtExceptionHandler`来处理。
3. 否则，如果该异常是`ThreadDeath`的实例，那么直接退出，如果不是，会将线程名字以及异常栈打印至标准错误输出流（控制台）。这是我们没有设置任何处理器时的默认逻辑，开头那段代码就是这种情况，没有设置任何处理器，所以只是在控制台输出了线程名称和异常信息。

#### 默认处理器
上面代码中第二个分支中`Thread.getDefaultUncaughtExceptionHandler()`是什么呢？
```java
// null unless explicitly set
private static volatile UncaughtExceptionHandler defaultUncaughtExceptionHandler;
/**
* Set the default handler invoked when a thread abruptly terminates
* due to an uncaught exception, and no other handler has been defined
* for that thread.
*
* <p>Uncaught exception handling is controlled first by the thread, then
* by the thread's {@link ThreadGroup} object and finally by the default
* uncaught exception handler. If the thread does not have an explicit
* uncaught exception handler set, and the thread's thread group
* (including parent thread groups)  does not specialize its
* <tt>uncaughtException</tt> method, then the default handler's
* <tt>uncaughtException</tt> method will be invoked.
* <p>By setting the default uncaught exception handler, an application
* can change the way in which uncaught exceptions are handled (such as
* logging to a specific device, or file) for those threads that would
* already accept whatever &quot;default&quot; behavior the system
* provided.
*
* <p>Note that the default uncaught exception handler should not usually
* defer to the thread's <tt>ThreadGroup</tt> object, as that could cause
* infinite recursion.
*
* @param eh the object to use as the default uncaught exception handler.
* If <tt>null</tt> then there is no default handler.
*
* @throws SecurityException if a security manager is present and it
*         denies <tt>{@link RuntimePermission}
*         (&quot;setDefaultUncaughtExceptionHandler&quot;)</tt>
*
* @see #setUncaughtExceptionHandler
* @see #getUncaughtExceptionHandler
* @see ThreadGroup#uncaughtException
* @since 1.5
*/
public static void setDefaultUncaughtExceptionHandler(UncaughtExceptionHandler eh) {
    SecurityManager sm = System.getSecurityManager();
    if (sm != null) {
        sm.checkPermission(
            new RuntimePermission("setDefaultUncaughtExceptionHandler")
                );
    }
    defaultUncaughtExceptionHandler = eh;
}
/**
* Returns the default handler invoked when a thread abruptly terminates
* due to an uncaught exception. If the returned value is <tt>null</tt>,
* there is no default.
* @since 1.5
* @see #setDefaultUncaughtExceptionHandler
* @return the default uncaught exception handler for all threads
*/
public static UncaughtExceptionHandler getDefaultUncaughtExceptionHandler(){
    return defaultUncaughtExceptionHandler;
}

```
可以看到这个方法是一个`Thread`类的静态方法，`defaultUncaughtExceptionHandler`也是`Thread`类的静态属性,表示可以供所有线程使用的默认的`UncaughtExceptionHandler`。可以分别通过`setter`方法和`getter`方法设置和获取。

## 处理流程总览
![Uncaught Exception Process](https://blog-picture.nos-eastchina1.126.net/picture0013.png)
> 注：上图中Handler指的是`UncaughtExceptionHandler`

# 扩展知识
## ThreadGroup - 线程组
- 线程组是一组线程的集合
- 线程组中也可以包含其他线程组
- 线程组的组织是一个树结构，除了初始线程组之外每个线程组都有父线程组
- 线程组实现了`Thread.UncaughtExceptionHandler`接口
- 初始线程组是`system`线程组，由系统创建，这个线程组没有父线程组，通过下面这个构造方法创建
    ```java
    /**
    * Creates an empty Thread group that is not in any Thread group.
    * This method is used to create the system Thread group.
    */
    private ThreadGroup() {     // called from C code
        this.name = "system";
        this.maxPriority = Thread.MAX_PRIORITY;
        this.parent = null;
    }
    ```

## ThreadDeath
我们还忽略了一个小细节，就是在`ThreadGroup`类的默认处理逻辑中，如果异常是`ThreadDeath`的实例，是不会进行处理的。
`ThreadDeath`的源码：
```java
/**
 * An instance of {@code ThreadDeath} is thrown in the victim thread
 * when the (deprecated) {@link Thread#stop()} method is invoked.
 *
 * <p>An application should catch instances of this class only if it
 * must clean up after being terminated asynchronously.  If
 * {@code ThreadDeath} is caught by a method, it is important that it
 * be rethrown so that the thread actually dies.
 *
 * <p>The {@linkplain ThreadGroup#uncaughtException top-level error
 * handler} does not print out a message if {@code ThreadDeath} is
 * never caught.
 *
 * <p>The class {@code ThreadDeath} is specifically a subclass of
 * {@code Error} rather than {@code Exception}, even though it is a
 * "normal occurrence", because many applications catch all
 * occurrences of {@code Exception} and then discard the exception.
 *
 * @since   JDK1.0
 */
public class ThreadDeath extends Error {
    private static final long serialVersionUID = -4417128565033088268L;
}
```
当`Thread.stop()`方法被调用时，会抛出一个`ThreadDeath`类的实例。
应用程序只有必须在异步终止后进行清理时才应该捕获该类的实例。如果 ThreadDeath 被一个方法捕获，那么将它重新抛出非常重要，因为这样才能让该线程真正终止。
这就是为什么`ThreadGroup`的`uncaughtException`没有捕获`ThreadDeath`异常。


# 参考链接
1. [Lesson: Exceptions (The Java™ Tutorials > Essential Classes)](https://docs.oracle.com/javase/tutorial/essential/exceptions/index.html)
2. [JVM 如何处理未捕获异常](https://droidyue.com/blog/2019/01/06/how-java-handle-uncaught-exceptions/)
3. [How uncaught exceptions are handled](https://www.javamex.com/tutorials/exceptions/exceptions_uncaught_handler.shtml)
4. [Thread (Java Platform SE 8 )](https://docs.oracle.com/javase/8/docs/api/java/lang/Thread.html)