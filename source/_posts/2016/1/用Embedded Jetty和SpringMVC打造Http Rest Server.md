title:  "用Embedded Jetty和SpringMVC打造Http Rest Server"
date: 2016-01-12
updated: 2016-01-15
categories:
- network
tags:
- network
- spring
- web server
---

> Don't deploy your application in Jetty, deploy Jetty in your application!

## $1 是时候抛弃Tomcat了
你是不是一直很讨厌写`web.xml`。其实你就只需要一个简单的`Dynamic HTTP Server`，结果你不得不去下载一个`tomcat`，把你的应用打包成`war`，然后部署到`tomcat`中。如果你只想在你的应用中提供一个`Restful`的`Http Server`给外部，对内还要提供一些`RPC`的逻辑，这时候你是不是还得去Google一下Tomcat如何提供2个`Connectors`，然后如何让不同的`Request`发送到不同的`Connector`。
这个时候Jetty就是你的救星，特指Embeded Jetty。

## $2 你唯一需要的就是一个jar包
`node.js`需要配置文件吗？不需要！需要打包成一个可部署的组件吗？不需要！需要你先启动一个类似容器的玩意吗？不需要！
``` js
http.createServer(function (request, response) {
    	//handle request
    	//make response    
	});
}).listen(port);
```
`node.js`创建一个`HTTP Server`就是这么简单。那么Embeded Jetty呢？
``` java
public class SimplestServer
{
    public static void main( String[] args ) throws Exception
    {
        Server server = new Server(8080);
        server.setHandler(new AbstractHandler(){
        	public void handle( String target,
                        Request baseRequest,
                        HttpServletRequest request,
                        HttpServletResponse response ) throws IOException,
                                                      ServletException
	    	{
		        response.setContentType("text/html; charset=utf-8");
		        response.setStatus(HttpServletResponse.SC_OK);
		        PrintWriter out = response.getWriter();
		        out.println("<h1>" + greeting + "</h1>");
		        if (body != null)
		        {
		            out.println(body);
		        }
		 
		        baseRequest.setHandled(true);
	    	}
        });
        server.start();
        server.join();
    }
}
```
看起来还行。那需要讨厌的配置文件吗？不需要，你只需要在maven中依赖jetty-all就可以啦
``` xml
<dependency>
  <groupId>org.eclipse.jetty.aggregate</groupId>
  <artifactId>jetty-all</artifactId>
  <version>9.3.4.RC1</version>
</dependency>
```
所有配置文件的逻辑在Embeded Jetty中都可以通过参数解决，最关键的是：Jetty定义得很清晰，一目了然。


## $3 Jetty启动和处理Request的过程

### $3.1 Jetty启动

1 创建一个Server 
``` java 
Server server = new Server();
```

2 创建一个或多个Connector
``` java
ServerConnector http = new ServerConnector(server);
http.setPort(8080);
http.setHost("localhost");
server.addConnector(http);
```

3 创建一堆Handler
``` java
HandlerList handlers = new HandlerList();
handlers.setHandlers(new Handler[]{
        requestLogHandler,
        rewrite,
        spingMvcHandler,
//      new DefaultHandler()
});
server.setHandler(handlers);
```

4 Go!
``` java
server.start();
```
代码已经简单到不忍直视了。有同学要问了，那原来那一堆配置文件能改动的参数，现在应该咋弄啊。

### $3.2 Jetty的配置
在官方文档上有一个[Like Jetty XML例子](http://www.eclipse.org/jetty/documentation/current/embedding-jetty.html)。要注意的是这个例子给出来的是修改jetty本身的默认参数的，对于web.xml里面的那种参数和`Spring`集成相关的参数不一样。也就是说，这个是类似于修改tomcat的参数。

- 设置http的参数
``` java
// HTTP Configuration
HttpConfiguration http_config = new HttpConfiguration();
http_config.setSecureScheme("https");
http_config.setSecurePort(8443);
http_config.setOutputBufferSize(32768);
http_config.setRequestHeaderSize(8192);
http_config.setResponseHeaderSize(8192);
http_config.setSendServerVersion(true);
http_config.setSendDateHeader(false);
ServerConnector http = new ServerConnector(server,
        new HttpConnectionFactory(http_config));
http.setPort(8080);
http.setIdleTimeout(30000);
server.addConnector(http);
```
其他的配置不展开了，因为我也没有好好研究。遇上了参考着改就行。

### $3.3 Jetty处理REQUEST的过程
一言以蔽之，就是你设置的那一堆handler被一个个地调用。
- `HandlerCollection`会每个都调用，哪怕某个Handler抛了Exception，或者`baseRequest.isHandled()==true`
- `HandlerList` 在执行过程中，如果哪个handler挂了就不继续了，或者`baseRequest.isHandled()==true`也不继续了。
上面示例中给出的`HandlerList`包含了
- `RequestLogHandler`，这个就是Request的log记录，当然你也可以用log4j之类的
``` java
NCSARequestLog requestLog = new AsyncNCSARequestLog();
requestLog.setFilename("./jetty-yyyy_mm_dd.log");
requestLog.setExtended(false);
RequestLogHandler requestLogHandler = new RequestLogHandler();
requestLogHandler.setRequestLog(requestLog);
```
- `RewriteHandler`，用做UrlRewrite。如果`RequestLogHandler`放在`RewriteHandler`之后，会看到Http请求的Url被转换了。
``` java
//urlrewrite handler
RewriteHandler rewrite = new RewriteHandler();

RewritePatternRule oldToNew = new RewritePatternRule();
oldToNew.setPattern("/some/old/spingMvcHandler");
oldToNew.setReplacement("/someAction?val1=old&val2=spingMvcHandler");
rewrite.addRule(oldToNew);

RewriteRegexRule reverse = new RewriteRegexRule();
reverse.setRegex("/reverse/([^/]*)/(.*)");
reverse.setReplacement("/reverse/$2/$1");
rewrite.addRule(reverse);
```
- `ServletContextHandler`，这个是我用来接入SpringMVC的Handler，并且排在最后一个。在初始化这个Handler的时候要注意同时初始化好`Spring`。
``` java
ServletContextHandler spingMvcHandler = new ServletContextHandler();
spingMvcHandler.setContextPath("/some");
XmlWebApplicationContext context = new XmlWebApplicationContext();
context.setConfigLocations(new String[]{"classpath:config/spring/appcontext-bean.xml"});
spingMvcHandler.addEventListener(new ContextLoaderListener(context));
spingMvcHandler.addServlet(new ServletHolder(new DispatcherServlet(context)), "/*");
```
用Spring和Tomcat的同学一下子就能看懂了吧，这里的WebApplicationContext传入的配置文件就是spring的配置文件；EventListener就是ContextLoaderListener。
提醒一句，如果是用SpringMVC的注解，千万记得加上`<mvc:annotation-driven />`
``` xml
appcontext-bean.xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:aop="http://www.springframework.org/schema/aop"
       xmlns:mvc="http://www.springframework.org/schema/mvc"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-2.5.xsd
    http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context-2.5.xsd
    http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop-2.5.xsd
    http://www.springframework.org/schema/mvc http://www.springframework.org/schema/mvc/spring-mvc.xsd">

    <context:annotation-config></context:annotation-config>
    <context:component-scan base-package="jetty9"/>
    <mvc:annotation-driven />
</beans>
```
至此，Request进入到SpringMVC的Controller了。

## $4 用SpringMVC处理Http Request

### $4.1 通过一堆Annotation获取到任何你想要的Http参数
```java
@RestController
public class MySpringMVC {
    @RequestMapping(value = "/greeting/{who}", method = {RequestMethod.POST, RequestMethod.GET})
    @ResponseStatus(code = HttpStatus.OK)
    public GreetingVO greeting(@RequestParam String name,
                               @PathVariable String who,
                               @RequestHeader("id") long id,
                               @CookieValue("JSESSIONID") String cookie) {
        return new GreetingVO(id, who, name);
    }
}
```
- `@RequestParam` 用来获取GET的参数，或者HttpHeader中`Content-Type: application/x-www-form-urlencoded`的POST参数
- `@RequestBody` 一般用来获取Post时Content-Type不是application/x-www-form-urlencoded的参数
- `@PathVariable` URI中的参数
- `@RequestHeader`和`@CookieValue`不言而喻
- `@ResponseStatus` 用来标记返回Http的Code，
- `@RestController` 是@Controller和@ResponseBody的组合，默认是json格式返回，记得加上下面的maven依赖

``` xml
<dependency>
  <groupId>com.fasterxml.jackson.core</groupId>
  <artifactId>jackson-core</artifactId>
  <version>2.2.0</version>
</dependency>
<dependency>
  <groupId>com.fasterxml.jackson.core</groupId>
  <artifactId>jackson-databind</artifactId>
  <version>2.2.0</version>
</dependency>
```
### $4.2 有洁癖的，或者喜欢从底层处理的朋友
直接拿到`RequestEntity`，拿Request Header，拿Request Body。构造ResponseEntity, Set Response Header和Response Body。应有尽有，想干嘛干嘛啊
``` java
@RequestMapping(value = "/test", method = {RequestMethod.POST, RequestMethod.GET})
public ResponseEntity<String> handle(HttpEntity<byte[]> requestEntity) throws UnsupportedEncodingException {
    String id = requestEntity.getHeaders().getFirst("id");

    byte[] requestBody = requestEntity.getBody();
    MediaType contentType = requestEntity.getHeaders().getContentType();
    // do something with request header and body
    if (contentType.equals(MediaType.APPLICATION_JSON_UTF8_VALUE)) {
        String requestBodyStr = new String(requestBody, "UTF-8");
    }

    HttpHeaders responseHeaders = new HttpHeaders();
    responseHeaders.set("MyResponseHeader", "MyValue");
    return new ResponseEntity<String>("Hello World", responseHeaders, HttpStatus.CREATED);
}
```

## $5 其他杂项
- 关于Filter，个人理解和Handler很像，但是Handler不会返回来处理。
- 之所以选择SpringMVC是因为一般做Java的都逃不过Spring，一大堆Spring定义好的beans。SpringMVC本身提供的功能很强大，先用熟几个常用的注解，然后理解了Spring处理的流程，其他高级的注解也就会好用了，比如`@ControllerAdvice`
- 个人认为在Java后端生成HTML的方式会慢慢被取代，因为前端和后端联调是一个低效率的事情。Java后端就应该专注于数据处理，把生成HTML和View相关的东西全部都丢给前端，或者现在流行的MVVM结构。

## Reference
[Jetty9文档（英文）](http://www.eclipse.org/jetty/documentation/current/advanced-embedding.html)
[SpringMVC文档（英文）](http://docs.spring.io/spring/docs/current/spring-framework-reference/html/mvc.html)

