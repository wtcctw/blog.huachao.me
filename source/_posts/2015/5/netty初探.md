title:  "netty初探"
date: 2015-05-07
updated: 2015-12-26
categories:
- 后端
tags:
- rpc
- 异步
- 多线程
---

> netty, Java的node.js

## netty怎么工作
netty底层基于epoll，其实无所谓是select, poll, epoll, kqueue，这些都是OS上进行实现的一种通信方式。
回顾一下原来的通信方式，服务端起一个listen线程（进程），有客户端连接过来，listener线程会创建一个worker线程（进程）；这种方式无法适应大规模的链接，因为创建线程和线程间切换会使得服务端资源（Memory，CPU）被快速吃光，即便没有几个客户端在进行通信，但是都是长连接的，所以CPU需要在进程间切换来服务客户端。
select方式是注册一组关心的事件，比如ACCEPT, READ, WRITE，然后发生这些事件的时候，内核来通知服务器进程进行处理。poll, epoll, kqueue则更进一步，具体怎么牛逼没有研究。但是注意，这里是Nio，不是AIO，也就是说不是node.js那种异步IO，而只是非阻塞的IO。
netty有一个boss thread负责epoll相关事宜，也就是维持大量的客户端长连接。在有事件发生时，比如新连接来了，读写Channel之类的，netty会启动一个Thread来负责处理这个事件。在用Thread处理的时候也是非常流程化的，熟悉的同学知道有各种Handler，一些是处理输入的，一些是处理输出的。

### Bootstrp
Netty本身的BootStrap和ServerBootStrap过程都是比较形式化的，所以是一个TemplateMethod模式好的运用场景。
一个Spring配置Client Template的例子
``` xml
<bean id="nettyClientTemplate" class="me.huachao.netty.client.MyNettyClientTemplate">
    <property name="host" value="localhost"/>
    <property name="port" value="8080"/>
    <property name="sslEnabled" value="false"/>
    <property name="handlers">
        <list>
            <bean id="httpClientCodec" class="io.netty.handler.codec.http.HttpClientCodec"/>
            <bean id="httpContentDecompressor" class="io.netty.handler.codec.http.HttpContentDecompressor"/>
            <bean class="me.huachao.netty.client.handler.HttpSnoopClientHandler"/>
        </list>
    </property>
    <property name="runner" ref="httpClientRunner"/>

</bean>

<bean id="httpClientRunner" class="me.huachao.netty.client.runner.HttpClientRunner"/>
```


### ChannelOption
netty中的ChannelOption都是针对TCP/IP的参数设置。如果涉及到HTTP或者其他协议的参数设置，都是需要在上层协议头中进行设置的，换句话说，ChannelOption.SO_KEEPALIVE是TCP层的设置，如果需要在HTTP中设置Connetion:keep-alive，是需要在HTTP头中进行设置的。注意：在ServerBootstrap设置option是，option()是对bossgroup，childoption()才是对真正处理的Handler设置
- TCP_NODELAY, 哪怕没有收到足够多的bit，也会每个msg都给一个ACK
- SO_KEEPALIVE, 会周期性地探测peer是否还存活着
- SO_BROADCAST, 在用UDP时允许广播
- SO_BACKLOG, 队列中的最大请求链接数

### Handler
研究了一下Http相关的几个Handler。其中HttpSnoopClient和HttpSnoopServer是只有encode/decode，其他对象和response都是直接解析的，其过程如下
![直接解析](/images/2015/5/netty初探/http-request-response-process.png)
还有HttpObjectAggreator和ChunkedWriteHandler，前者用来聚合一系列的httpObject，后者用来切割HttpContent
![通过http已有的Handler进行辅助](/images/2015/5/netty初探/http-objectAggregator.png)

- 关于keep-alive, 如果true，那么不要添加CLOSE的listener，这样会关闭这个Channel的。
``` java
boolean keepAlive = HttpHeaderUtil.isKeepAlive(req);
if (!keepAlive) {
    ctx.write(response).addListener(ChannelFutureListener.CLOSE);
} else {
    response.headers().set(CONNECTION, KEEP_ALIVE);
    ctx.write(response);
}
```
- 注意自己创建的ByteBuf需要手动销毁。这个还没有找到一个好的例子来实践一下。
- 在Handler对象中的成员变量，对于每个Channel都是一份。猜测在实现的时候，只有thread出于上个Channel已经unregistered了才会进行复用。
- 由InboundHandler转到OutBoundHandler处理的过程中，需要writeAndFlush，否则数据过不来，还是停留在InBound中。

## Json Server
做了一个练练手的Json Server，模仿的[github上star数很多的json-server](https://github.com/typicode/json-server)

## Code
[MySampleCode@Github](https://github.com/wtcctw/netty-basic)