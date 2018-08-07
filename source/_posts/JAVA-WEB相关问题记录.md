title: JAVA WEB相关问题记录
author: Keaper
date: 2018-07-15 16:51:04
tags:
---
### 添加了@ResponseBody注解但是返回500错误

原因：springmvc默认的json转换器需要jack依赖 
```
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
    <version>2.9.4</version>
</dependency>

<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-annotations</artifactId>
    <version>2.9.4</version>
</dependency>

<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-core</artifactId>
    <version>2.9.4</version>
</dependency>
```

### 添加依赖没有生效

可能是新添加的依赖没有打包到war包中，在project structure 中 artifacts 中将未包含的包加入到output中即可


### spring mvc @ResponseBody注解返回枚举类型

[http://www.baeldung.com/jackson-serialize-enums](http://www.baeldung.com/jackson-serialize-enums)




### spring mvc 返回400 

将org.springframework.web 包中的log级别调整为DEBUG,可以看到错误原因

### mbatis 动态更新

set 元素可以用于动态包含需要更新的列，而舍去其它的。比如：
```xml
<update id="updateAuthorIfNecessary">
  update Author
    <set>
      <if test="username != null">username=#{username},</if>
      <if test="password != null">password=#{password},</if>
      <if test="email != null">email=#{email},</if>
      <if test="bio != null">bio=#{bio}</if>
    </set>
  where id=#{id}
</update>
```
这里，set 元素会动态前置 SET 关键字，同时也会删掉无关的逗号，因为用了条件语句之后很可能就会在生成的 SQL 语句的后面留下这些逗号。（译者注：因为用的是“if”元素，若最后一个“if”没有匹配上而前面的匹配上，SQL 语句的最后就会有一个逗号遗留）


### mybatis 报错 org.apache.ibatis.executor.ExecutorException: No constructor found

问题是由于自己定义了一个构造函数,所以没有默认的构造函数,添加默认构造函数之后就可以了。
原因暂时不明白。

### react项目 ngnix静态资源服务器配置

```
server {
        listen       80;
        server_name  localhost;

        location / {
            root   E:\\workspace\\IdeaProjects\\classroomsys\\front\\dist;
            index  index.html index.htm;
            
            try_files $uri $uri/ /index.html;
        }
        
		location /api {
			proxy_pass http://127.0.0.1:8080;
		}
        
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
    }
```


### maven项目中读取properties文件


```java
PROPERTIES.load(×××.class.getClassLoader().getResourceAsStream("config.properties"));
```
[java getResourceAsStream的使用方法
](https://blog.csdn.net/winwill2012/article/details/39641401)


### java mysql 时区问题

[如何规避mysql的url时区的陷阱](https://www.jianshu.com/p/7e9247c0b81a)


### mysql 8.0升级问题
1. 报错
```java
java.sql.SQLException: Unknown system variable 'query_cache_size'
```

解决办法：
升级connector jar包

```xml
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>8.0.11</version>
</dependency>
```

2. 报错
```
Unable to load authentication plugin 'caching_sha2_password'.
```
解决办法：
```sql
ALTER USER 'username'@'ip_address' IDENTIFIED WITH mysql_native_password BY 'password';
```

[mysql - Authentication plugin 'caching_sha2_password' cannot be loaded - Stack Overflow](https://stackoverflow.com/questions/49194719/authentication-plugin-caching-sha2-password-cannot-be-loaded)



### 静态资源处理 与 拦截器、异常处理器

问题:springMVC配置了静态资源处理
```xml
    <mvc:resources mapping="/file/**" location="/static/"/>
```

但是访问静态资源报错：
```
java.lang.ClassCastException: org.springframework.web.servlet.resource.ResourceHttpRequestHandler cannot be to org.springframework.web.method.HandlerMethod
```

报错位置在全局异常处理器

ExceptionResolver.java
```java

public ModelAndView resolveException(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) {
        ModelAndView mv = new ModelAndView();
        
        HandlerMethod handlerMethod = (HandlerMethod) handler; //报错在这一行
        Method method = handlerMethod.getMethod();
        …………………
        …………………
        return mv;
    }
```

ResourceHttpRequestHandler不能够被强制转换成HandlerMethod所以报错

解决办法便是在强制转换之前判断一下：
```java
public ModelAndView resolveException(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) {
        ModelAndView mv = new ModelAndView();
        //先判断
        if(!(handler instanceof HandlerMethod)){
            return mv;
        }
        
        HandlerMethod handlerMethod = (HandlerMethod) handler; //报错在这一行
        Method method = handlerMethod.getMethod();
        …………………
        …………………
        return mv;
    }
```

同样在拦截器中也有类似的处理：
```java
public class TokenInterceptor implements HandlerInterceptor {
    private static final Logger logger = LoggerFactory.getLogger(TokenInterceptor.class);


    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        
        //先判断
        if(!(handler instanceof HandlerMethod)){
            return true;
        }
        
        HandlerMethod handlerMethod = (HandlerMethod) handler;
        Method method = handlerMethod.getMethod();
        /**
         * 注解的类或方法需要校验
         */
        if(method.getDeclaringClass().getAnnotation(TokenValidate.class) == null &&
                method.getAnnotation(TokenValidate.class) == null){
            return true;
        }

        ……………………
        ……………………
    }
}
```

回过头来看下ResourceHttpRequestHandler和HandlerMethod都是什么？为什么不能强制转?


简单的说，ResourceHttpRequestHandler是用来处理静态资源的；而HandlerMethod则是springMVC中用@Controller声明的一个bean及对应的处理方法。

[SpringMVC源码研究之 mvc:resources](https://blog.csdn.net/lqzkcx3/article/details/78601545)

[SpringMVC处理静态文件源码分析](https://my.oschina.net/pingpangkuangmo/blog/388208)



### typehandler 


### maven pom配置


### jenkins 项目部署

git,maven,tomcat,jdk,nginx


### IDEA 突然找不到所有引入的包

File -> Invalidate Caches/Restarts
