title:  "深入Jedis"
date: 2016-02-18
categories:
- cache
tags:
- cache
- network
---

> Redis客户端与服务器端的通信协议是如此简单

## RESP协议
RESP(REdis Serialization Protocol)是redis server与redis client的通信协议。
- TCP Port 6379
- Request-Response模型。2个例外，1）pipeline；2）pub/sub
- 5种DataType，Simple String（+）；Errors（-）；Integers（:）；Bulk String（$）；Arrays（*）
- `\r\n`(CRLF)是结束符
- Simple String 例子：`"+OK\r\n"`
- Errors 例子：`-WRONGTYPE Operation against a key holding the wrong kind of value`
- Integer 例子：`":1000\r\n"`
- Bulk String 例子：`"$6\r\nfoobar\r\n"` 6表示后面有6个byte的长度
- Null 例子：`"$-1\r\n"`
- Arrays 例子：`"*2\r\n$3\r\nfoo\r\n$3\r\nbar\r\n"` 2表示有2个元素； `"*0\r\n"`表示空数组
- 客户端发送命令：就是Bulk String。例子：`llen mylist` -> `*2\r\n$4\r\nllen\r\n$6\r\nmylist\r\n`
- redis服务器回答RESP DataType。例子：`:48293\r\n`

## Jedis对RESP协议的抽象
![Jedis类图](/images/2016/2/深入Jedis/jedis-class-diagram.png)
- Protocol是实现上述RESP协议的主要类，其中可以看到`sendCommand(final RedisOutputStream os, final byte[] command, final byte[]... args)`是如何根据协议拼接字符串发送到redis server，`Object read(final RedisInputStream is)`是如何接收redis server的返回，并且转换为Java Object。
- `BinaryXxxCommands <- BinaryJedis, XxxCommands <- Jedis` 用来抽象所有通过二进制流来发送的Redis命令
- `XxxCommands <- Jedis`用来抽象类似ClusterCommands的命令，最终都是走的二进制流，去掉Binary一层估计是作者觉得厌烦了。不对之处还请赐教。
- `Commands, Connection <- BinaryClient <- Client`抽象了网络发送命令和接收回复，其中Client将参数encode为byte[]，然后调用BinaryClient的方法；BinaryClient调用Connection#sendCommand；sendCommand调用connect()，构造RedisInputStream和RedisOutputStream，用Protocol.sendCommand来发送命令；client.getXxxReply()首先将outputstream中的内容flush出去，然后调用Protocol.read来处理接收到的返回值。
``` java
/* 发送命令 Connection.java */
protected Connection sendCommand(final ProtocolCommand cmd, final byte[]... args) {
    try {
      connect();
      Protocol.sendCommand(outputStream, cmd, args);
      pipelinedCommands++;
      return this;
    } catch (JedisConnectionException ex) {
      // Any other exceptions related to connection?
      broken = true;
      throw ex;
    }
}

public void connect() {
    if (!isConnected()) {
      try {
        socket = new Socket();
        // ->@wjw_add
        socket.setReuseAddress(true);
        socket.setKeepAlive(true); // Will monitor the TCP connection is
        // valid
        socket.setTcpNoDelay(true); // Socket buffer Whetherclosed, to
        // ensure timely delivery of data
        socket.setSoLinger(true, 0); // Control calls close () method,
        // the underlying socket is closed
        // immediately
        // <-@wjw_add

        socket.connect(new InetSocketAddress(host, port), connectionTimeout);
        socket.setSoTimeout(soTimeout);
        outputStream = new RedisOutputStream(socket.getOutputStream());
        inputStream = new RedisInputStream(socket.getInputStream());
      } catch (IOException ex) {
        broken = true;
        throw new JedisConnectionException(ex);
      }
    }
}

/* 接收回复 */
public String getBulkReply() {
    final byte[] result = getBinaryBulkReply();
    if (null != result) {
      return SafeEncoder.encode(result);
    } else {
      return null;
    }
}

public byte[] getBinaryBulkReply() {
    flush();
    pipelinedCommands--;
    return (byte[]) readProtocolWithCheckingBroken();
}

protected Object readProtocolWithCheckingBroken() {
    try {
      return Protocol.read(inputStream);
    } catch (JedisConnectionException exc) {
      broken = true;
      throw exc;
    }
}
```
- Jedis通过Pipeline这个类来对redis的pipeline进行抽象，`jedis.pipelined()`返回一个Pipeline实例，并且这个Pipeline实例的client就是当前jedis实例的client；调用`pipeline.a_redis_command()`的时候会有一个`responseList`，用来记录每个command应该对应的response；`pipeline.syncAndReturnAll()`会调用`client.getAll()`将所有command一次`flush()`出去，然后拿回List&lt;Object&gt;，再将这些Object填充到responseList中。


## Jedis使用注意事项
- Jedis instance本身不是线程安全的！要用JedisPool
``` java
//将JedisPool定义为spring单例
JedisPool pool = new JedisPool(new JedisPoolConfig(), "localhost");

Jedis jedis = null;
try {
  jedis = pool.getResource();
  /// ... do stuff here ... for example
  jedis.set("foo", "bar");
  String foobar = jedis.get("foo");
  jedis.zadd("sose", 0, "car"); jedis.zadd("sose", 0, "bike"); 
  Set<String> sose = jedis.zrange("sose", 0, -1);
} finally {
  if (jedis != null) {
    jedis.close();
  }
}
/// ... when closing your application:
pool.destroy();
```
- JedisPool是一个包装模式，内部就是Apache Common Pool 2， Pool里面装的是Jedis。Jedis之所以不是线程安全的主要是由于Jedis类中的fields(client, pipeline, transaction)没有做同步。如果每个thread都有一份Jedis实例，其实也不存在线程安全问题，就是要注意使用完了需要`jedis.close()`。JedisPool和DBCP的Pool一样，就是用来创建Jedis实例，然后提供给线程使用，Pool技术能够复用已经标记为IDLE的Jedis，以此来提供内存利用率和减小开销。

## 小结
- Redis的通信协议简单容易实现
- Jedis在实现协议的时候用的Client将Connection和Command解耦，中规中矩，值得学习
- JedisPool用了Apache Common Pool来做到ThreadSafe


