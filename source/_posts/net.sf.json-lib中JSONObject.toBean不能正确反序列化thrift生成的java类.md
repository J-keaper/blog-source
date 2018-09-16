title: net.sf.json-lib中JSONObject.toBean不能正确反序列化thrift生成的java类
tags:
  - JAVA
categories: []
author: Keaper
date: 2018-09-16 18:29:00
---

---

### 现象
在使用json-lib中`JSONObject.toBean`将`JsonObject`转化为由thrift生成的Java类时，发现得到的javabean中的数据均为空（null或者默认值）。


### 复现
测试代码：
pom.xml
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>cn.keaper</groupId>
    <artifactId>thrift-test</artifactId>
    <version>1.0-SNAPSHOT</version>
    
    <dependencies>
        <dependency>
            <groupId>org.apache.thrift</groupId>
            <artifactId>libthrift</artifactId>
            <version>0.11.0</version>
        </dependency>
        <dependency>
            <groupId>net.sf.json-lib</groupId>
            <artifactId>json-lib</artifactId>
            <version>2.4</version>
        </dependency>
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>4.12</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.hamcrest</groupId>
            <artifactId>hamcrest-all</artifactId>
            <version>1.3</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>commons-lang</groupId>
            <artifactId>commons-lang</artifactId>
            <version>2.6</version>
        </dependency>
        <dependency>
            <groupId>net.sf.ezmorph</groupId>
            <artifactId>ezmorph</artifactId>
            <version>1.0.6</version>
        </dependency>
        <dependency>
            <groupId>commons-beanutils</groupId>
            <artifactId>commons-beanutils</artifactId>
            <version>1.9.3</version>
        </dependency>
    </dependencies>

    <properties>
        <thrift.executable.shell>/usr/local/bin/thrift</thrift.executable.shell>
    </properties>
    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.thrift.tools</groupId>
                <artifactId>maven-thrift-plugin</artifactId>
                <version>0.1.10</version>
                <configuration>
                    <thriftExecutable>${thrift.executable.shell}</thriftExecutable>
                    <generator>java</generator>
                    <outputDirectory>src/main/java</outputDirectory>
                </configuration>
                <executions>
                    <execution>
                        <id>thrift-sources</id>
                        <phase>generate-sources</phase>
                        <goals>
                            <goal>compile</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
</project>
```
定义一个thrift struct作为测试的类 `student.thrift`
```thrift
namespace java cn.keaper

struct Student {
    1: string name;
    2: i32 age;
}
```
再自定义一个普通的javaBean，作对比之用`Person.java`
```java
public class Person {

    String name;
    int age;

    public String getName() {
        return name;
    }
    public void setName(String name) {
        this.name = name;
    }
    public int getAge() {
        return age;
    }
    public void setAge(int age) {
        this.age = age;
    }
    @Override
    public String toString() {
        return "Person{" +
                "name='" + name + '\'' +
                ", age=" + age +
                '}';
    }
}

```
测试代码：
```java
@Test
public void testJSON() {
    String json = "{name:\"edward\",age:\"20\"}";
    JSONObject jsonObject = JSONObject.fromObject(json);
    System.out.println(jsonObject.toString());

    Student student = (Student) JSONObject.toBean(jsonObject,Student.class);
    System.out.println(student.toString());

    Person person = (Person) JSONObject.toBean(jsonObject,Person.class);
    System.out.println(person.toString());
}
```
数据结果：
```
{"name":"edward","age":"20"}
Student(name:null, age:0)
九月 16, 2018 4:58:42 下午 net.sf.json.JSONObject toBean
信息: Property 'name' of class cn.keaper.Student has no write method. SKIPPED.
九月 16, 2018 4:58:42 下午 net.sf.json.JSONObject toBean
信息: Property 'age' of class cn.keaper.Student has no write method. SKIPPED.
Person{name='edward', age=20}
```

我们可以看到我们自定义的javaBean使用toBean方法转换是没有问题的，但是thrift生成的java类在使用toBean方法时是不能正确获取到数据的。

### 问题溯源
通过单步跟踪发现：
在JSONObject.toBean方法中执行到以下代码：
```java
PropertyDescriptor pd = PropertyUtils.getPropertyDescriptor(bean, key);
if (pd != null && pd.getWriteMethod() == null) {
    log.info("Property '" + key + "' of " + bean.getClass() + " has no write method. SKIPPED.");
}else{
    ......
}
```
这里的执行对应了上文的日志输出，可见问题的原因是没有找到合适的set方法去对类的属性赋值。


我们来看看thrift生成的java类中setter，getter：
```java
public String getName() {
    return this.name;
}

public Student setName(String name) {
    this.name = name;
    return this;
}
.....

public int getAge() {
    return this.age;
}

public Student setAge(int age) {
    this.age = age;
    setAgeIsSet(true);
    return this;
}
```

这里可以看到其中是有set方法的。
那么为什么没找到呢？继续跟踪，中间过程略长,已略过，最后我们来到了`Introspector.getTargetPropertyInfo()`
```java
try {
    if (argCount == 0) {
        if (name.startsWith(GET_PREFIX)) {
            // Simple getter
            pd = new PropertyDescriptor(this.beanClass, name.substring(3), method, null);
        } else if (resultType == boolean.class && name.startsWith(IS_PREFIX)) {
            // Boolean getter
            pd = new PropertyDescriptor(this.beanClass, name.substring(2), method, null);
        }
    } else if (argCount == 1) {
        if (int.class.equals(argTypes[0]) && name.startsWith(GET_PREFIX)) {
            pd = new IndexedPropertyDescriptor(this.beanClass, name.substring(3), null, null, method, null);
        } else if (void.class.equals(resultType) && name.startsWith(SET_PREFIX)) {
            // Simple setter
            pd = new PropertyDescriptor(this.beanClass, name.substring(3), null, method);
            if (throwsException(method, PropertyVetoException.class)) {
                pd.setConstrained(true);
            }
        }
    } else if (argCount == 2) {
        if (void.class.equals(resultType) && int.class.equals(argTypes[0]) && name.startsWith(SET_PREFIX)) {
            pd = new IndexedPropertyDescriptor(this.beanClass, name.substring(3), null, null, null, method);
            if (throwsException(method, PropertyVetoException.class)) {
                pd.setConstrained(true);
            }
        }
    }
} catch (IntrospectionException ex) {
    // This happens if a PropertyDescriptor or IndexedPropertyDescriptor
    // constructor fins that the method violates details of the deisgn
    // pattern, e.g. by having an empty name, or a getter returning
    // void , or whatever.
    pd = null;
}
```
可见其中判断set方法的依据是**参数个数为1**,**返回类型为void**，**方法名以set开头**，所以回头看thrift生成的java类
```java
public Student setName(String name) {
    this.name = name;
    return this;
}
.....

public Student setAge(int age) {
    this.age = age;
    setAgeIsSet(true);
    return this;
}
```
其中的set方法与我们自定义javaBean的set方法不同的是它返回void，而是返回对象本身，所以它不能够被识别为一个setter，所以导致最后没有序列化成功。


### 总结
1. `Introspector`这个类是jdk中用来处理javaBean的类。
>The Introspector class provides a standard way for tools to learn about the properties, events, and methods supported by a target Java Bean.
所以其中对setter的处理是符合Java Bean规范的。

2. json-lib中使用的是`commons-beanutils`包，这个包中使用的就是jdk中`Introspector`来处理Java Bean，所以导致按照Java Bean 规范并不能正确的得到一个Java Bean。
3. fastJson
>fastjson是阿里巴巴的开源JSON解析库，它可以解析JSON格式的字符串，支持将Java Bean序列化为JSON字符串，也可以从JSON字符串反序列化到JavaBean。

经测试，使用fastjson的`JSON.parseObject`方法是可以正确反序列化出thrift生成的JAVA类的，而且直接传入json字符串即可。