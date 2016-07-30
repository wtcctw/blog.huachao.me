title:  "说说我用netty的经验和教训"
date: 2016-07-29
categories:
- network
tags:
- network
---

> netty真好用，后悔没有早点沉下心来好好学学

# 我理解的netty概略图
![](/images/2016/7/说说我用netty的经验和教训/netty.png)

# 简单梳理图中的红线过程
1. 客户端连接
2. boss channel接受连接，创建或者找到client channel，并且将处理过程delegate到child handlers
3. 所谓delegate到child handlers，意思是child Eventloop会用这个channel的handlers来进行业务逻辑处理
4. 事件是netty用来通知的方式，如果client的通道上有事件发生，比如通道上有数据了，那么netty会触发channelRead方法；如果handler在处理过程抛了异常，那么netty会触发exceptionCaught()方法执行
5. 调用write方法往通道上发送数据


# ServerBootStrap
- 理清ServerBootStrap要干的几件事情
	- 2个线程池，很重。
	- channel类型指定，NioServerChannel
	- childHandler的指定
	- 一些参数的指定
	- bind
- 为啥要干这些事情
	- 线程池是用来执行方法的
	- Handler用来执行业务逻辑
- 注意点
	- 一般我们把整个Server的逻辑拆成2块，1）start；2）stop；
	- start最终目的是要执行bind，并且等待bind成功，所以一般我们会在ChannelFuture上加上sync()方法进行同步等待。
	- stop最终要释放所有的资源，主要是2个线程池，close所有的child channels和boss channel
	- 可以用`Runtime.getRuntime().addShutdownHook(new ShutdownThread(this));`结束server，注意ShutdownHook接收的是15信号，所以优雅关闭的命令是`kill -15 pid`

# EventLoop
- Eventloop就是一个运行时的载体，它会执行每个channel上的handlers
- Eventloop扩展自`ScheduledExecutorService`，对于在Chile通道上执行一些定时任务很合适
- inEventLoop()方法可以用来判断是否当前线程可以马上执行，如果true，那就直接do()；如果false，那就`executors().schedule(new Runnable(){do();}, delayTime)`
- 在做代理的时候，new BootStrap().group()应该要复用当前channel.eventloop()，这样效率是最高的

# Channel & ChannelContext
- 每个连接用一个Channel来抽象，有10w个连接那么就有10w个Channel实体，并且每个Channel有一个对应的context
- 上面这句话的意思是，Channel和ChannelContext都是线程安全的，因为每个通道一份，你不会在不同的通道上还要共享啥东西吧。。。
- channel.write()会导致从最后一个outboundhandler开始往前write；ctx.write()会从当前这个outboundhandler开始往前write，所以如果你在inboundhandler里面write，那么效果和channel.write()是一样的

# Handler & Pipeline
- 我一开始一直理解错了，以为内存中只有一份Handlers和Pipeline；其实不是，每个Channel都有一份自己的Handlers和Pipeline。这样就可以理解为啥在netty中不需要去处理共享资源了，因为每次Eventloop都是去拿到一个channel，及其handlers，pipeline。然后你在业务逻辑中也不用去考虑锁之类的东西，因为每个通道都是独立的。
- 每个inboundhandler都有如下方法需要实现
	- `void channelRegistered(ChannelHandlerContext ctx) throws Exception;`
    - `void channelUnregistered(ChannelHandlerContext ctx) throws Exception;`
	- `void channelActive(ChannelHandlerContext ctx) throws Exception;`
	- `void channelInactive(ChannelHandlerContext ctx) throws Exception;`
	- `void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception;`
	- `void channelReadComplete(ChannelHandlerContext ctx) throws Exception;`
	- `void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws Exception;`
	- `void channelWritabilityChanged(ChannelHandlerContext ctx) throws Exception;`
	- `void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception;`
- 每个outboundHandler有如下方法需要实现
	- `void bind(ChannelHandlerContext ctx, SocketAddress localAddress, ChannelPromise promise) throws Exception;`
	- `void connect(ChannelHandlerContext ctx, SocketAddress remoteAddress, SocketAddress localAddress, ChannelPromise promise) throws Exception;
	- `void disconnect(ChannelHandlerContext ctx, ChannelPromise promise) throws Exception;`
	- `void close(ChannelHandlerContext ctx, ChannelPromise promise) throws Exception;`
	- `void deregister(ChannelHandlerContext ctx, ChannelPromise promise) throws Exception;`
	- `void read(ChannelHandlerContext ctx) throws Exception;`
	- `void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) throws Exception;`
	- `void flush(ChannelHandlerContext ctx) throws Exception;`
- 我为啥滔滔不绝写上面那么多？肯定不是蛋疼，对不对。我是要告诉你，你在用别人的handler的时候，千万记住这些方法肯定都实现了，你没看到的话，去超类里面找！
- 你没看到`channelRead()`方法？快去父类里面找，不然你怎么知道整个的逻辑是怎么处理的？decode()其实是包含在channelread()方法中的
- 你只需要关注某些已经实现的Codec类的部分方法？握草，不知道整个的逻辑是怎么处理的你怎么会用？
- 并且！！！！channel上产生事件以后，每个handler中的对应方法都会走到的，为什么？因为默认的adaptor里面用的处理方法是ctx.fireXXX()这样就把时间传递到下一个handler去处理了，并且肯定会调用下一个handler中的channelXXX()

# ByteBuf
- [请看这篇文章](http://calvin1978.blogcn.com/articles/netty-leak.html)

# 其他一些心得体会
- 最坑的pipeline，习惯性地`pipeline.addLast(handler)`，结果把outboundhandle加到最后面，导致你在outboundhandler之前的inboundhandler用ctx.write()的时候就是没有办法把数据写入，还以为只有channel.write()才能真正把数据写入。


