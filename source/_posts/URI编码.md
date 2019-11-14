# 问题
使用`Spring RestTemplate`发送HTTP请求时请求参数中的`'+'`发送到服务端(`Tomcat`)被解析成了`' '`。
网上查找一番，问题原因大概明了了。简单来说就是`Spring RestTemplate`在对参数编码时没有处理`'+'`，而`Tomcat`在处理参数时会将`'+'`替换`' '`。

# 解决方法
`Spring RestTemplate`提供了可以定制的编码模式。
```java
DefaultUriBuilderFactory factory = new DefaultUriBuilderFactory(baseUrl)
factory.setEncodingMode(EncodingMode.TEMPLATE_AND_VALUES);

// Customize the RestTemplate..
RestTemplate restTemplate = new RestTemplate();
restTemplate.setUriTemplateHandler(factory);
```

# 溯源
先来复习几个知识点：
1. [URI - wikipedia](https://en.wikipedia.org/wiki/Uniform_Resource_Identifier)
2. 两个与URI相关的rfc规范 
    - [Uniform Resource Locators (URL) - rfc1738](https://tools.ietf.org/html/rfc1738) 
    - [Uniform Resource Identifier (URI): Generic Syntax - rfc3986](https://tools.ietf.org/html/rfc3986)
3. [百分号编码 - wikipedia](https://en.wikipedia.org/wiki/Percent-encoding)


一个URI由一个包含数字，字母，和一些图形符号的有限集合的字符组成。百分号编码用于当字符在允许的字符集之外或者与URI语法中的分隔符有冲突时，需要对其进行百分号编码。
所有的的`HTTP Client`（浏览器或者各种程序库）和`HTTP Server`(如`Tomcat`，`Jetty`)等都必须有相应的编码和解码的实现，否则不能够正确处理请求URI，比如URI参数中的中文必须被编码解码。

而我们上面看到的问题，其实与百分号编码关系不大。
URI规范中并没有规定`'+'`需要进行百分号编码，而且上面的问题中`Spring RestTemplate`没有对`'+'`编码，而Tomcat在处理参数时却将`'+'`替换为了空格。这也不是百分号解码的逻辑，只是简单的字符替换。为什么？

[When to encode space to plus (+) or %20?](https://stackoverflow.com/questions/2678551/when-to-encode-space-to-plus-or-20)
[URL encoding the space character: + or %20?](https://stackoverflow.com/questions/1634271/url-encoding-the-space-character-or-20)

参考这两个回答，将`' '`编码为`'+'`的做法是在早期HTML规范中定义的。
早期HTML2.0规范([RFC1866](https://tools.ietf.org/html/rfc1866))定义中定义了对于`application/x-www-form-urlencoded`内容编码时需要将`空格`替换为`'+'`，同时对于GET方法URL中携带的QUERY参数需要按照同样的编码方式处理。
但是在之后版本的HTML规范(HTML2.0之后HTML规范一直由万维网联盟（W3C）维护,[HTML 4 Specification](https://www.w3.org/TR/html4/)以及[HTML 5 Specification](https://www.w3.org/TR/html5/))中，都只定义了`application/x-www-form-urlencoded`内容的编码方式，但是对于URL中的Query部分没有提及。所以对于是否将空格编码为加号这一行为，客户端与服务端在具体实现的时候行为如果不一致就会导致编码解码不一致，出现问题。

# 总结


# 扩展
看下不同的客户端和服务端是如何处理空格加号编码这个问题的：
## JAVA URLEncoder/URLDecoder:
URLDecoder.decode("a+b%2Bc")
URLEncoder.encode("a b+c")

## Apache Http Client
Apache Http Client  URLEncodedUtils：
URLEncodedUtils.format(Lists.newArrayList(new BasicNameValuePair("a b+c","a b+c")),"UTF-8")
URLEncodedUtils.parse("a+b%2Bc", Charsets.UTF_8)

## Spring RestTemplate
RestTemplate在没有自定义uriTemplateHandler属性时的默认行为是不会编码`'+'`的。
可以自定义uriTemplateHandler属性来改变其编码行为：
```java
String baseUrl = "http://example.com";
DefaultUriBuilderFactory factory = new DefaultUriBuilderFactory(baseUrl)
factory.setEncodingMode(EncodingMode.TEMPLATE_AND_VALUES);

// Customize the RestTemplate..
RestTemplate restTemplate = new RestTemplate();
restTemplate.setUriTemplateHandler(factory);
```
## Tomcat
org.apache.catalina.connector.RequestFacade#getParameterMap
org.apache.catalina.connector.Request#getParameterMap
org.apache.catalina.connector.Request#getParameterValues
org.apache.catalina.connector.Request#parseParameters
org.apache.tomcat.util.http.Parameters#handleQueryParameters
org.apache.tomcat.util.http.Parameters#processParameters
org.apache.tomcat.util.http.Parameters#processParameters
org.apache.tomcat.util.http.Parameters#urlDecode
org.apache.tomcat.util.buf.UDecoder#convert


## Jetty 
org.eclipse.jetty.servlet.ServletHolder#handle
org.eclipse.jetty.server.request#getParameterMap
org.eclipse.jetty.server.request#getParameters
org.eclipse.jetty.server.request#extractQueryParameters
org.eclipse.jetty.http.HttpURI#decodeQueryTo
org.eclipse.jetty.util.UrlEncoded#decodeUtf8To

Tomcat和Jetty解码时都会将`'+'`替换为`' '`.

# 参考链接：
1. [HTML - wikipedia](https://zh.wikipedia.org/wiki/HTML)
2. [Uniform Resource Locators (URL) - rfc1738](https://tools.ietf.org/html/rfc1738) 
3. [Uniform Resource Identifier (URI): Generic Syntax - rfc3986](https://tools.ietf.org/html/rfc3986)
4. [Hypertext Markup Language - 2.0 - RFC1866](https://tools.ietf.org/html/rfc1866)
5. [HTML 4 Specification](https://www.w3.org/TR/html4/)
6. [HTML 5 Specification](https://www.w3.org/TR/html5/)