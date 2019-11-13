title: 记一次诡异的JVM退出问题
author: Keaper
tags:
  - JAVA
categories: []
date: 2019-11-13 21:17:00
---
# 背景
1. 项目基于`Spring Boot`，会与另外一个项目通过`HTTP`协议进行`RPC`通信，所以项目中启动了`Netty`作为`HTTP Server`。
2. 项目使用`Maven`构建，使用`appassembler-maven-plugin`插件进行打包，使用打包后的脚本启动。

# 问题现象
某次更改代码，本地测试通过，没有问题。
部署测试环境，发现服务刚启动就会退出，重启还是退出，但是日志中没有找到任何任何报错异常信息。

# 问题复现
开始在本地复现：
1. 使用IDEA启动主类运行程序，没有问题，一切正常。
2. 会不会是`appassembler`插件生成的启动脚本的问题？根据测试环境的配置打包，使用启动脚本运行程序，这次问题复现了。
3. 在另外一台电脑上使用启动脚本运行程序，没有问题，一切正常。

问题并不能百分之百复现，与运行环境有着微妙的关系。

# 问题原因
我们从头来思考下这个问题，我们能看到的现象是JAVA应用退出了，也就是JVM关闭了，那么导致JVM关闭的原因有那些呢？
1. 调用Runtim.exit方法或者Runtime.halt方法，并且 SecurityManager允许执行退出操作。
2. 所有的**非守护**线程结束,线程结束原因有两种：
    - run()方法正常返回
    - 线程中抛出了未捕获的异常

JVM退出原因翻译至JDK文档，原文如下：
> When a Java Virtual Machine starts up, there is usually a single non-daemon thread (which typically calls the method named main of some designated class). The Java Virtual Machine continues to execute threads until either of the following occurs:
> - The exit method of class Runtime has been called and the security manager has permitted the exit operation to take place.
> - All threads that are not daemon threads have died, either by returning from the call to the run method or by throwing an exception that propagates beyond the run method.

项目是一个Web Server应用，代码中没有主动调用exit()方法，所以排除第一种情况，重点考虑的是第二种情况。  
不管是项目中LogBack的日志还是JVM的日志，都没有发现异常信息。基本可排除发生了异常。  
那就只有可能是程序真的就是没有任何"问题"，所有线程都正常运行结束。

## 猜想
因为项目并没有用到Spring Boot内嵌的Web容器（Netty Server是在另外一个依赖包中启动的），所以项目启动时指定了`web`选项为`NONE`。
```java
public static void main(String[] args) throws InterruptedException {
    new SpringApplicationBuilder(XXXApplication.class).web(WebApplicationType.NONE).build().run();
}
```
因为我们指定了了`web`选项为`NONE`,所以`SpringBoot`不会去启动Tomcat等Web容器，有没有可能是线程启动顺序问题呢，我们需要的`Netty Server`相关的线程还未启动，`main`线程以及其他**非守护**线程都已经结束，所以程序退出了呢？

## 验证猜想
验证很简单，在主线程执行完毕退出前sleep一段时间:
```java
public static void main(String[] args) throws InterruptedException {
    new SpringApplicationBuilder(XXXApplication.class).web(WebApplicationType.NONE).build().run();
    Thread.sleep(100); // sleep 100ms
}
```
打包编译运行，发现程序正常，没有退出。

继续验证一下是不是因为所有的**非守护线程**都已经执行结束导致程序退出。在主线程执行完毕退出前打印出所有线程的信息：
```java
public static void main(String[] args) throws InterruptedException {
    new SpringApplicationBuilder(XXXApplication.class).web(WebApplicationType.NONE).build().run();
    Thread.getAllStackTraces().keySet().forEach(t -> System.out.println(t.getName() + "," + t.isDaemon()));
}
```
输出：
```
xxl-job, executor JobLogFileCleanThread,true
Reference Handler,true
ZkClient-EventThread-13-tjwqstaging.zk.hadoop.srv:2181,true
xxl-job, executor TriggerCallbackThread,true
main,false
Finalizer,true
Thread-6,true
Signal Dispatcher,true
main-EventThread,true
Timer-0,true
Thread-7,true
main-SendThread(tj-hadoop-staging-zk05.kscn:2181),true
```
可以看到除了main线程之外的其他线程都是daemon线程，所以在main线程执行完后程序会退出。到这里说明我们起初的猜想是没有问题的。

## 溯源
那么我们想要的提供HTTP服务的相关线程为什么没有启动呢？我们Sleep 100ms之后再打印一下所有线程信息：
```java
public static void main(String[] args) throws InterruptedException {
    new SpringApplicationBuilder(XXXApplication.class).web(WebApplicationType.NONE).build().run();
    Thread.sleep(100);
    Thread.getAllStackTraces().keySet().forEach(t -> System.out.println(t.getName() + "," + t.isDaemon()));
}
```
```
xxl-job, executor JobLogFileCleanThread,true
main,false
Signal Dispatcher,true
main-EventThread,true
Timer-0,true
Thread-7,true
nioEventLoopGroup-2-1,false
nioEventLoopGroup-4-1,false
Reference Handler,true
ZkClient-EventThread-13-tjwqstaging.zk.hadoop.srv:2181,true
xxl-job, executor ExecutorRegistryThread,true
xxl-job, executor TriggerCallbackThread,true
Finalizer,true
Thread-6,true
main-SendThread(tj-hadoop-staging-zk03.kscn:2181),true
```
可以看到多出来的两个非守护线程是`nioEventLoopGroup-x-x`，这组线程正是`Netty`中用来处理HTTP请求的工作线程。为什么这组线程没有来得及创建呢？
我们找到创建它的地方，查看他启动Web Server的过程，看到这样一段代码：
```java
@Override
public void start(final XxlRpcProviderFactory xxlRpcProviderFactory) throws Exception {
    thread = new Thread(new Runnable() {
        @Override
        public void run() {
            // param
            final ThreadPoolExecutor serverHandlerPool = ThreadPoolUtil.makeServerThreadPool(NettyHttpServer.class.getSimpleName());
            EventLoopGroup bossGroup = new NioEventLoopGroup();
            EventLoopGroup workerGroup = new NioEventLoopGroup();
            try {
                // start server
                ServerBootstrap bootstrap = new ServerBootstrap();
                bootstrap.group(bossGroup, workerGroup)
                        .channel(NioServerSocketChannel.class)
                        .childHandler(new ChannelInitializer<SocketChannel>() {
                            @Override
                            public void initChannel(SocketChannel ch) throws Exception {
                                /*ch.pipeline().addLast(new HttpResponseEncoder());
                                ch.pipeline().addLast(new HttpRequestDecoder());*/
                                /*ch.pipeline().addLast(new ChunkedWriteHandler());*/
                                ch.pipeline().addLast(new HttpServerCodec());
                                ch.pipeline().addLast(new HttpObjectAggregator(5*1024*1024));  // merge request & reponse to FULL
                                ch.pipeline().addLast(new NettyHttpServerHandler(xxlRpcProviderFactory, serverHandlerPool));
                            }
                        })
                        .childOption(ChannelOption.SO_KEEPALIVE, true);
                // ...
            } catch (InterruptedException e) {
                // ...
            } finally {
                // ...
            }
        }
    });
    thread.setDaemon(true);	// daemon, service jvm, user thread leave >>> daemon leave >>> jvm leave
    thread.start();
}
```
可以看到在这个方法中启动了一个线程来创建Web Server，但是这个线程是`daemon`的，所以如果`NioEventLoopGroup`线程还未启动，其他**非守护线程**(包括`main`线程)已经执行完的话，那么程序就会退出。
而在我们上面`sleep` 100ms 的这段时间，给了这组线程启动的机会，所以程序能够继续运行。

至于有时能复现有时复现不了的原因猜测，应该是机器环境或者是JAVA启动参数不同导致线程启动先后顺序有微妙的差异。

# 总结
1. JVM退出的原因：
    - 调用Runtim.exit方法或者Runtime.halt方法，并且 SecurityManager允许执行退出操作。
    - 所有的**非守护**线程结束,线程结束原因有两种：要么是run()方法正常返回，要么是线程中抛出了未捕获的异常


# 参考文档
1. [https://docs.oracle.com/javase/8/docs/api/java/lang/Thread.html](https://docs.oracle.com/javase/8/docs/api/java/lang/Thread.html)