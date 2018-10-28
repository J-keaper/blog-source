title: JAVA 日志简介
author: Keaper
tags:
  - JAVA
  - 日志
categories:
  - JAVA
date: 2018-10-28 20:43:00
---
当我们开发应用程序时，它可能不会按照我们预期的运行，这个时候通常我们会DEBUG，来观察程序中的代码分支(if,else)，观察一些变量在运行中的值等等以便确定程序的运行过程。但是，在生产环境，我们通常没有办法，或者不能够很容易去进行DEBUG。因此，更好的方法是使用日志工具。
在生产环境中，日志是查找问题来源的重要依据。应用程序运行时的产生的各种信息，包括记录程序运行时产生的错误信息、状态信息、调试信息和执行时间信息等等，都可以通过日志来进行记录。

在JAVA中，有哪些记录日志的方法呢？
介绍之前我们对下文做如下约定：
>1. 依赖方式均以`maven denpendency`方式给出。
>2. 下文所说的`系统属性`指的是`java`中的`System Property`,即通过`System.setProperty`设置或者通过`java -D`设置的系统属性。
>3. 本文只会简单介绍各个日志库的配置方式及基本用法，详细的配置项及更高级的用法请移步官方文档。

## 最原始的方式
你一定记得在刚开始学习Java时,下面这种写法：
```java
System.out.println("variable is :" + variable);
```
或者这样，
```java
try{
    …………            
}catch (Exception e){
    e.printStackTrace();
}
```
在程序中打印变量的值，在异常发生时打印异常栈，这都有助于分析程序的运行过程，但是这种方式缺乏灵活性，一般也只会在初学时用到。

严格来说，这根本算不上记录日志。。。
我们需要的日志工具库至少应该满足：
1. 输出目标(也就是日志库中`appender`或者`handler`)  
上述方式会将日志打印到控制台，但是在现实的应用场景中，我们可能需要将输出到文件系统，数据库，数据总线等系统中。
2. 环境(可以灵活的设置日志级别`level`)  
我们需要根据环境不同来决定是否输出某些日志信息，生产环境只输出必要的信息，而在非生产环境，更多的调试信息会更有助于我们解决问题。  

## JDKLog(JUL)
JDK在`java.util.logging`包下提供了日志相关的API，不过在JDK1.4才加入，在此之前，JDK中并不包含日志记录相关的API和实现。

### 依赖
不需要任何外部依赖，JDK自身提供的。
### 配置
JDK通过`java.util.logging.config.file`系统属性来读取自定义配置文件的位置，如果不配置，JDK中有默认的配置文件，位置在`[path/to/jre]/lib/logging.properties`。
#### 配置文件示例
```
# 全局的日志handler
handlers= java.util.logging.ConsoleHandler,java.util.logging.FileHandler
# 全局的日志级别
.level= INFO

# ConsoleHandler配置level以及formater
java.util.logging.ConsoleHandler.level = INFO
java.util.logging.ConsoleHandler.formatter = java.util.logging.SimpleFormatter
# SimpleFormatter的格式
java.util.logging.SimpleFormatter.format = %1$tc [%4$s]:%5$s %n

# FileHandler配置
java.util.logging.FileHandler.pattern = %h/java%u.log
java.util.logging.FileHandler.limit = 50000
java.util.logging.FileHandler.count = 1
java.util.logging.FileHandler.formatter = java.util.logging.XMLFormatter
```
`SimpleFormatter.format`如何自定义格式？参见：  
[https://docs.oracle.com/javase/7/docs/api/java/util/logging/SimpleFormatter.html#method_detail](https://docs.oracle.com/javase/7/docs/api/java/util/logging/SimpleFormatter.html#method_detail)

### 用法
```java
import java.util.logging.Logger;

public class App {
    //Logger.getLogger(String name),参数为logger的name，通常为类的全限定名
    private static Logger logger = Logger.getLogger(App.class.getName());
    public static void main( String[] args )
    {
        try{
            logger.info("This is test log.");
            throw new IllegalArgumentException("test Exception");
        }catch (Exception e){
            logger.severe("This is test Exception:" + e);
        }
    }
}
```
## Log4j
Log4j 是Apache的一个用于JAVA的日志库。是出现较早的日志工具，目前已经发展到Log4j2.X，但是这里我们讨论的是Log4j1.X，Log4j2.X下文会单独讨论。

### 依赖
```xml
<dependency>
    <groupId>log4j</groupId>
    <artifactId>log4j</artifactId>
    <version>1.2.17</version>
</dependency>
```
### 配置
Log4j没有任何默认的日志配置，如果没有配置文件。你将会遇到：
```
log4j:WARN No appenders could be found for logger (xx.xx.xx).
log4j:WARN Please initialize the log4j system properly.
log4j:WARN See http://logging.apache.org/log4j/1.2/faq.html#noconfig for more info.
```
我们看下Log4j默认的初始化过程(具体过程在`org.apache.log4j.LogManager`类中)：
1. 设置`log4j.defaultInitOverride`为除了`false`之外的其他值都会导致`Log4j`跳过初始化过程。(但是一般我们不会这么做)
2. 试图将`log4j.configuration`系统属性转换为一个`URL`(`java.net.URL`),从该URL加载配置,如果转换失败，则从`classpath`中寻找`log4j.configuration`系统属性配置的文件,从该文件加载配置
4. 如果没有设置`log4j.configuration`属性，则在`classpath`目录下寻找`log4j.xml`文件，从该文件加载配置。
5. 如果仍然没有，则在`classpath`目录下寻找`log4j.properties`文件，从该文件加载配置。
6. 如果以上均失败了，中止初始化。

从以上流程可以看出，最简单的配置方式就是提供`log4j.xml`或者`log4j.properties`文件。

#### `log4j.xml`文件示例：
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE log4j:configuration PUBLIC "-//APACHE//DTD LOG4J 1.2//EN" "log4j.dtd">
<log4j:configuration xmlns:log4j='http://jakarta.apache.org/log4j/'>
    <!--配置Console Appender-->
    <appender name="stdout" class="org.apache.log4j.ConsoleAppender">
        <layout class="org.apache.log4j.PatternLayout">
            <param name="ConversionPattern" value="[%-5p %d{yyyy-MM-dd HH:mm:ss.SSS}] [%t] %l [%m]%n"/>
        </layout>
    </appender>

    <!--配置File Appender-->
    <appender name="file" class="org.apache.log4j.FileAppender">
        <!--文件位置-->
        <param name="File" value=" out.log" />
        <param name="Append" value="false" />
        <!--日志格式-->
        <layout class="org.apache.log4j.PatternLayout" >
            <param name="ConversionPattern" value="[%-5p %d{yyyy-MM-dd HH:mm:ss.SSS}] [%t] %l [%m]%n" />
        </layout>
    </appender>
    
    <!--配置Logger-->
    <logger name="cn.keaper" additivity="false">
        <level value="debug"/>
        <appender-ref ref="file"/>
    </logger>

    <root>
        <level value="debug"/>
        <appender-ref ref="stdout"/>
    </root>
</log4j:configuration>
```
### 用法：
```java
import org.apache.log4j.Logger;

public class App {
    //Logger.getLogger()可以传递string作为 logger name
    //也可以传递Class，此时则用类的全限定名作为 logger name
    private static final Logger logger = Logger.getLogger(App.class);

    public static void main( String[] args )
    {
        try{
            logger.info("This is test log.");
            throw new IllegalArgumentException("test Exception");
        }catch (Exception e){
            logger.error("This is test Exception:",e);
        }
    }
}
```
## 休息一下
> 除了 `JUL` 和 `Log4J`这样的日志记录库之外，还有一类库用来封装不同的日志记录库。这样的封装库中一开始以 `Apache Commons  Logging`框架最为流行，现在比较流行的是`SLF4J`。这样封装库的API都比较简单，只是在日志记录库的API基础上做了一层简单的封装，屏蔽不同实现之间的区别。

这样做典型的好处是可以在一个公共的项目中，可以通过这类公共的API去操作日志，但是具体的实现由调用方去指定。这样可以避免在依赖多个不同的日志库时需要对多种不同的日志库提供实现及配置。

## commoning-log(JCL)
>JCL provides thin-wrapper Log implementations for other logging tools, including Log4J, Avalon LogKit (the Avalon Framework's logging infrastructure), JDK 1.4, and an implementation of JDK 1.4 logging APIs (JSR-47) for pre-1.4 systems.

JCL内置支持的日志实现工具包括`Log4J`、`Avalon LogKit`(项目目前已经关闭)、JDK 1.4和为JDK1.4之前系统提供的JDK log API的实现(JSR-47)。这几个中使用最多的还是`Log4j`。

### 依赖
```xml
<dependency>
    <groupId>commons-logging</groupId>
    <artifactId>commons-logging</artifactId>
    <version>1.2</version>
</dependency>
```
为了实现具体的日志实现，则需要引入具体的依赖库，如果用`Log4j`，那么需要引入`Log4j`的依赖：
```xml
<dependency>
    <groupId>log4j</groupId>
    <artifactId>log4j</artifactId>
    <version>1.2.17</version>
</dependency>
```
### 配置
`JCL`的配置文件是`classpath`路径中的`commons-logging.properties`。`JCL`中使用`LogFactory`来获取具体的`Log`,`LogFactory`可以通过`org.apache.commons.logging.LogFactory`属性指定，当然`JCL`提供了默认的`LogFactory`,默认的`Logfactory`通过如下流程决定使用哪种具体的日志实现：
1. 寻找`LogFactory`中名为`org.apache.commons.logging.Log`(同时兼容1.0版本之前的`org.apache.commons.logging.log`)的属性指定的`Log`。  
>这里的属性指的是通过`LogFactory`类的`getAttribute`方法获得的，注意定义在`commons-logging.properties`文件中的每个属性都会成为这个默认的`LogFactory`的`Attribte`.
2. 如果没有找到，寻找系统属性中名为`org.apache.commons.logging.Log`(同时兼容1.0版本之前的`org.apache.commons.logging.log`)的属性指定的`Log`。
3. 如果依然没有找到，`Log4j`是可用的(可以找到`Log4j`相关的类),使用`Log4j`。
4. 如果没有找到`Log4j`，JDK版本在1.4以上，那么使用`JDKLog`。
5. 使用JCL内置的`SimpleLog`。

如果我们使用`Log4j`，`JDKLog`，我们可以不用配置`commons-logging`,注意着并不等于不用配置`Log4j`或者`JDKLog`，在使用了`commons-logging`之后，不管是`Log4j`还是`JDKLog`，配置方式都是与单独使用时是相同的，`commoons-logging`和具体的日志实现的配置是分开的。

### 用法
```java
import org.apache.commons.logging.Log;
import org.apache.commons.logging.LogFactory;

public class App {
    private static final Log logger = LogFactory.getLog(App.class);

    public static void main( String[] args )
    {
        try{
            logger.info("This is test log.");
            throw new IllegalArgumentException("test Exception");
        }catch (Exception e){
            logger.error("This is test Exception:",e);
        }
    }
}
```

## slf4j
`slf4j`与`commons-logging`相似，是有一个日志门面框架。
`slf4j`支持`log4j`,`JDKLog`,`logback`同时支持`commons-logging`(相当于两层门面)。

### 依赖
```xml
<dependency>
    <groupId>org.slf4j</groupId>
    <artifactId>slf4j-api</artifactId>
    <version>1.7.25</version>
</dependency>
```
`slf4j-api`提供Log的接口，而具体使用哪种实现则需要提供另一类包：绑定包(`SLF4J bindings`)，每一种绑定包对应一种实现：

|binding jar|支持日志实现|
|:--------:|:--------:|
|`slf4j-log4j12.jar`| log4j 1.2 |
|`slf4j-jdk14.jar`| JDKLog |
|`slf4j-jcl.jar`|commons-logging|

同时`logback`内置slf4j实现，所以使用`logback`并不需要`binding jar`。  
![](http://phb8vtude.bkt.clouddn.com/concrete-bindings.png)

引入了对应的`binding jar`之后，我们还需要引入日志实现库所需要的依赖。  
例如如果我们用`Log4j`作为日志实现，除了`slf4j-api`需要引入，还需要引入
```xml
<dependency>
    <groupId>org.slf4j</groupId>
    <artifactId>slf4j-log4j12</artifactId>
    <version>1.7.21</version>
</dependency>
<dependency>
    <groupId>log4j</groupId>
    <artifactId>log4j</artifactId>
    <version>1.2.17</version>
</dependency>
```

### 配置
`Slf4j`并不需要单独配置，只要引入相应日志的`binding jar`以及其对应依赖并配置即可。

这里的绑定实际上是加载`org.slf4j.impl.StaticLoggerBinder`类，每个`binding jar`中都有这样一个类。如果找到多个绑定，则取决于加载的先后顺序。

### 用法
```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class App {
    private static final Logger logger = LoggerFactory.getLogger(App.class);

    public static void main( String[] args )
    {
        try{
            logger.info("This is test log.");
            throw new IllegalArgumentException("test Exception");
        }catch (Exception e){
            logger.error("This is test Exception:",e);
        }
    }
}
```

### slf4j bridging
如果在你依赖的模块中使用了别的日志库，而在你的项目中你使用的是`slf4j`,这个时候`slf4j` 提供了一些桥接模块可以避免你同时对多个日志库进行配置，这些桥接模块可以重定向对其他日志库的调用到`slf4j`.
官方图表达的很清晰：
![](http://phb8vtude.bkt.clouddn.com/legacy.png)

## Logback
`Logback`是由`Log4j`创始人设计的又一个开源日日志组件。`Logback`当前分成三个模块：`logback-core`,`logback-classic`和`logback-access`。  
`logback-core`是其它两个模块的基础模块。`logback-classic`是`log4j`的一个改良版本。此外`logback-classic`完整实现了`SLF4J  API`，使你可以很方便地更换成其它日记系统如`Log4J`或`JDK14 Logging`。`logback-access`模块可以与Servlet容器集成以提供HTTP日志相关的功能。

### 依赖
```xml
<dependency>
    <groupId>ch.qos.logback</groupId>
    <artifactId>logback-classic</artifactId>
    <version>1.1.7</version>
</dependency>
```
`logback-classic`这个包内部依赖`logback-coer`和`slf4j-api`。所以不需要再额外引入。另外我们在上一节说到，logback提供了对slf4j接口的原生实现，所以不需要提供`bingding jar`。

### 配置
logback通过以下步骤完成初始化：
1. logback尝试在classpath目录下寻找`logback-test.xml`文件。
2. 如果不存在，logback尝试在classpath目录下寻找`logback.groovy`文件。
3. 如果不存在，logback尝试在classpath目录下寻找`logback.xml`文件。
4. 如果都不存在，在classpath目录下寻找`META-INF\services\ch.qos.logback.classic.spi.Configurator`,这个文件的内容是`com.qos.logback.classic.spi.Configurator`接口的实现类的全限定名，logback将使用该类作为配置类。
5. 如果仍然没有找到，那么logback使用默认的`BasicConfigurator`类作为配置类，这个`Configurator`将日志输出至控制台。

`logback.xml`示例：
```xml
<?xml version="1.0"  encoding="UTF-8" ?>
<configuration>
    <!--Console Appender配置-->
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <!-- encoders are  by default assigned the type
             ch.qos.logback.classic.encoder.PatternLayoutEncoder -->
        <encoder>
            <pattern>%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
        </encoder>
    </appender>
    <!--File Appender配置-->
    <appender name="FILE" class="ch.qos.logback.core.FileAppender">
        <file>myApp.log</file>
        <encoder>
            <pattern>%date %level [%thread] %logger{10} [%file:%line] %msg%n</pattern>
        </encoder>
    </appender>

    <logger name="cn.keaper" level="INFO">
        <appender-ref ref="FILE" />
    </logger>

    <root level="DEBUG">
        <appender-ref ref="STDOUT" />
    </root>
</configuration>
```

### 用法
既然logback内置了对slf4j的实现，那么他的使用方式就和slf4j的使用方式一样。
```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class App {
    private static final Logger logger = LoggerFactory.getLogger(App.class);

    public static void main( String[] args )
    {
        try{
            logger.info("This is test log.");
            throw new IllegalArgumentException("test Exception");
        }catch (Exception e){
            logger.error("This is test Exception:",e);
        }
    }
}
```

## Log4j 2
以上我们说的全部都是`Log4j 1`,而`Log4j`现在已经升级到了`2.X`版本。`Log4j 2`已经不仅仅是一个日志实现框架了，他也能够作为一个门面框架来使用，并提供对其他框架丰富的支持。我们只介绍他单纯作为一个日志实现框架的用法，更多的用法请移步官网自行查阅。

### 依赖
```xml
<dependency>
    <groupId>org.apache.logging.log4j</groupId>
    <artifactId>log4j-api</artifactId>
    <version>2.11.1</version>
</dependency>
<dependency>
    <groupId>org.apache.logging.log4j</groupId>
    <artifactId>log4j-core</artifactId>
    <version>2.11.1</version>
</dependency>
```

### 配置
`Log4j 2`通过初始化实现自动配置的流程如下：
1. 检查`log4j.configurationFile`系统属性，如果有设置，将使用
2. 如果没有设置，依次从classpath中寻找配置文件,顺序依次为：
    - `log4j2-test.properties`
    - `log4j2-test.yaml`或者`log4j2-test.yml`
    - `log4j2-test.json`或者`log4j2-test.jsn`
    - `log4j2-test.xml`
    - `log4j2.properties`
    - `log4j2.yaml`或者`log4j2.yml`
    - `log4j2.json`或者`log4j2.jsn`
    - `log4j2.xml`
3.如果这么多文件都没有没有找到，则会有一个默认的配置，这个配置会将日志输出至控制台。


Log4J2.X的xml配置方式上有一些不同,主要的不同可以看这里：[Migrating from Log4j 1.x](https://logging.apache.org/log4j/2.x/manual/migration.html)  

`log4j2.xml`配置文件示例：
```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration>
    <Appenders>
        <!--配置Console Appender-->
        <Console name="STDOUT" target="SYSTEM_OUT">
            <PatternLayout pattern="%d %-5p [%t] %C{2} (%F:%L) - %m%n"/>
        </Console>
        <!--配置File Appender-->
        <File name="FILE" filename="out.log" append="false">
            <PatternLayout pattern="%d %-5p [%t] %C{2} (%F:%L) - %m%n"/>
        </File>
    </Appenders>
    <Loggers>
        <Logger name="cn.keaper" level="info">
            <AppenderRef ref="FILE"/>
        </Logger>
        <Root level="debug">
            <AppenderRef ref="STDOUT"/>
        </Root>
    </Loggers>
</Configuration>
```
### 用法
在使用方式上，也有了变化，主要变化是在Log4J1.X中Logger是具体的类，而在Log4J2.X中，Logger变成了接口(log4j-api包中)，在log4j-core中提供具体实现。这样做的好处是使用log4j的api同时可以使用其他不同日志库的实现。
```java
//注意类的位置不同了
import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;

public class App {
    //Log4j 1.x中可以使用Logger.gerLogger()来获取Logger
    //但是在Log4j 2.X中需要使用LogManager.getLogger()来获取
    private static final Logger logger = LogManager.getLogger(App.class);

    public static void main( String[] args )
    {
        try{
            logger.info("This is test log.");
            throw new IllegalArgumentException("test Exception");
        }catch (Exception e){
            logger.error("This is test Exception:",e);
        }
    }
}
```
## 几种框架的关系
以上介绍的这几种框架可以分为两类(log4j 2 暂时不做区分)：
1. JUL, Log4j 1 和 Logback 是真正的日志实现。
2. commons-logging和sl4j是在真正日志实现之上的门面框架。每个框架都有自己一个简单实现，也可以使用一个另外的日志实现。

commons-logging和slf4j都可以作为JDKLog和Log4j 1的日志门面，此外slf4j还能够作为commons-logging本身和logback的门面框架。
他们之间的关系可以用下图表示：  
![](http://phb8vtude.bkt.clouddn.com/log.png)

## 参考
1. [JDKLog文档](https://docs.oracle.com/javase/8/docs/technotes/guides/logging/overview.html)
2. [Log4J1.X文档](http://logging.apache.org/log4j/1.2/manual.html)
3. [Log4j2.X文档](https://logging.apache.org/log4j/2.x/)
4. [commons-loggging](https://commons.apache.org/proper/commons-logging/guide.html)
5. [logback文档](https://logback.qos.ch/manual/index.html)
6. [Java日志管理最佳实践](https://note.youdao.com/)
7. [The Java Logging Mess](https://www.javacodegeeks.com/2011/09/java-logging-mess.html)
8. [The Java logging quagmire](https://vectorbeta.wordpress.com/2014/08/26/the-java-logging-quagmire/)