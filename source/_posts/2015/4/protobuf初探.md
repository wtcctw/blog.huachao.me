title:  "Google protobuf 初探"
date: 2015-04-26
updated: 2015-12-26
categories:
- network
tags:
- network
- rpc
- serialization
---

> protobuf 序列化、反序列化、RPC

## 序列化，反序列化
用RPC来做SOA(Service Oriented Architecture)服务化，衡量RPC好坏的一个重要指标就是序列化的长度，因为序列化的大小直接决定了网络传输的大小。众所周知，在网络IO目前和磁盘IO也还是一个量级的，所以在网络上进行IO讲究的是越小越好，越少越好。

protobuf中用来进行序列化和反序列最重要的是`.proto`文件，和protoc命令行。后者是[Google protobuf](https://github.com/google/protobuf) make install出来的一个二进制文件，建议在Linux环境下进行make，在Mac上遇到不少坑。用[protobuf官网](https://developers.google.com/protocol-buffers/docs/javatutorial?hl=zh-cn)上的例子。主要看一下生成的`.java`文件里面的结构，官网上也有说，但是还是直接看图比较直观。

- XxxOrBuilder 是一个interface，里面定义的是get方法
- Xxx 是Message，但是一般要用里面的一个Builder来构造，Xxx.newBuilder().setXxx().build() == Xxx
- Xxx 有writeTo()和parseFrom()方法用来序列化和反序列化

![generated file arch](/images/2015/4/protobuf初探/protobuf_dto_arch.png)
![rw](/images/2015/4/protobuf初探/protobuf_dto_rw.png)

其他没啥可说的，作为protobuf最主要的功能，在2.4.1和2.6.1上都测试过。

## RPC

RPC是protobuf最主要的一个功能点，但是我感觉做得一般。

- `protoc`生成`.java`文件时，默认不打开rpc选项。需要用`option java_generic_services = true;`来打开
- protobuf默认没有提供rpc的实现，只是定义了接口
- 给出的demo是基于protobuf-rpc-socket的，但是这个项目在2.4.1之后还没有发现提供支持，并且在mvn中也没有找到其artifact坐标
- protobuf-rpc-pro能提供2.6.1的支持，但是感觉其实现比较冗繁
- gRPC目前还在alpha中，等什么时候研究一下吧

先放出`protoc`生成的相应RPC接口
![rpc](/images/2015/4/protobuf初探/protobuf_rpc.png)

- 有Interface/Stub 和 BlockingInterface/BlockingStub 两组，对应在Nio客户端和Bio客户端请求时的调用接口
- Stub中的逻辑就是，构造好methodName和request param，然后通过Channel发送出去

``` java
public static final class Stub extends me.huachao.protobuf.service.HelloServiceProto.HelloService implements Interface {
      private Stub(com.google.protobuf.RpcChannel channel) {
        this.channel = channel;
      }
      
      private final com.google.protobuf.RpcChannel channel;
      
      public com.google.protobuf.RpcChannel getChannel() {
        return channel;
      }
      
      public  void sayHello(
          com.google.protobuf.RpcController controller,
          me.huachao.protobuf.service.HelloServiceProto.HelloRequest request,
          com.google.protobuf.RpcCallback<me.huachao.protobuf.service.HelloServiceProto.HelloResponse> done) {
        channel.callMethod(
          getDescriptor().getMethods().get(0),
          controller,
          request,
          me.huachao.protobuf.service.HelloServiceProto.HelloResponse.getDefaultInstance(),
          com.google.protobuf.RpcUtil.generalizeCallback(
            done,
            me.huachao.protobuf.service.HelloServiceProto.HelloResponse.class,
            me.huachao.protobuf.service.HelloServiceProto.HelloResponse.getDefaultInstance()));
      }
    }
```

## protobuf-rpc-socket
具体参看code

## protobuf-rpc-pro
[一个protobuf-rpc-pro的例子](http://blog.csdn.net/zhu_tianwei/article/details/44065097)

## gRPC
各种坑有木有！！！
[grpc](https://github.com/grpc/grpc-java)基于netty4.1
能不能build成功就看造化了。反正最后能在本地maven仓库里面install io.gpc.*
``` bash
➜  grpc  ls
grpc-all                 grpc-integration-testing grpc-stub
grpc-auth                grpc-netty               grpc-testing
grpc-benchmarks          grpc-okhttp              protoc-gen-grpc-java
grpc-core                grpc-protobuf
grpc-examples            grpc-protobuf-nano
```
注意看有个grpc-examples，里面有一些例子。比较悲剧的是目前protoc出来的`.java`文件，和grpc里面定义的Channel等不兼容。所以需要用grpc的protoc来生成，等grpc release以后再搞。

## Code
[MySampleCode@Github](https://github.com/wtcctw/protobuf-basic)