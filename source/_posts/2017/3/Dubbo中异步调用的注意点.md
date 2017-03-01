title:  "Dubbo中异步调用的注意点"
date: 2017-03-01
categories:
- rpc
tags:
- rpc
---

> Dubbo中异步调用的注意点

# GenericService发起异步调用步骤
```java
// 当前应用配置
ApplicationConfig application = new ApplicationConfig();
application.setName("message-center-client");
// 连接注册中心配置
RegistryConfig registry = new RegistryConfig();
registry.setProtocol("zookeeper");
registry.setAddress(XConfig.getValue("base-zk.inner.connStr"));
// 注意：ReferenceConfig为重对象，内部封装了与注册中心的连接，以及与服务提供方的连接
// 引用远程服务
ReferenceConfig<GenericService> reference = new ReferenceConfig<GenericService>(); // 此实例很重，封装了与注册中心的连接以及与提供者的连接，请自行缓存，否则可能造成内存和连接泄漏
reference.setApplication(application);
reference.setRegistry(registry); // 多个注册中心可以用setRegistries()
reference.setInterface(MetadataReadService.class);
reference.setVersion("1.0.0");
reference.setGeneric(true);
reference.setAsync(true);
GenericService gs = reference.get();
Object obj = gs.$invoke("method", new String[]{"paramType"}, new String[]{"value"});
Future<Xxx> future = RpcContext.getContext().getFuture();
```
- 如果是通过setUrl来直连，注意如果不是Dubbo协议要加上协议头
- 异步方式设置为true的时候，`$invoke`方法在dubbo协议的情况下，是肯定返回的null的。拿result是要通过future的
- 异步方式设置为true，但是不是dubbo协议，并且不支持异步方式，那么obj会有返回，并且上次的future不会被清空
- 此时，如果是同一个线程，去getFuture，会拿到上次的future！！

# 原因
- RpcContext是一个ThreadLocal的，所以同一个线程里面的future一样
- 在Dubbo协议的情况下，异步发起调用，setFuture的类型是ResponseFuture，默认实现是DefaultFuture。这个DefaultFuture是每个request.id对应一个Channel，并且每个request.id对应一个DefaultFuture，并且是线程安全的
