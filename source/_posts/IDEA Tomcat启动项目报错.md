title: >-
  IDEA Tomcat启动项目报错
  java.lang.ClassNotFoundException:org.springframework.web.context.ContextLoaderListener
author: Keaper
tags:
  - JAVA
  - IDEA
  - Tomcat
categories: []
date: 2018-01-21 01:34:00
---
IDEA Tomcat启动项目报错

```
java.lang.ClassNotFoundException:org.springframework.web.context.ContextLoaderListener
```

maven项目配置web项目，配置spring时启动报错，发现找不到ContextLoaderListener，查看输出路径WEB-INF/目录下没有lib目录，所以找不到类

解决方案：
这是由于pom.xml中下载的jar包未被部署。我们先ctrl+shift+alt+s打开Project Structure窗口，选择Artifacts，选择要打包部署的项目，在Output Layout –> Web-INF查看是否有lib目录，如果右边Available Elements窗口还显示有jar包，说明这些jar包未添加，则应右击选择Put into Output Root，这样就OK啦~


参考链接：[https://www.jianshu.com/p/18d068f47b09](https://www.jianshu.com/p/18d068f47b09)
