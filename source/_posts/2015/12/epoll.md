title:  "epoll"
date: 2015-12-29
categories:
- 后端
tags:
- epoll
- network
- socket
- 异步
---
> NIO只是IO的一种，不要将其神话

## select, poll, epoll都是干嘛的
一言以蔽之，都是Linux的系统调用，都是针对网络IO的。这些还都是NIO的，注意不是AIO。
- NIO，No-Blocking IO非阻塞的IO，非阻塞不一定是异步，比如我让内核在发生事件的时候再通知到App，App再去`clientSocket = serverSocket.accept();`；此处serverSocket.accept()本身是阻塞的，但是调用这个方法的时候是内核通知App其有ACCEPT事件，那么这个其实就是非阻塞的IO了，因为App上的IO并没有阻塞啊。
- AIO，Async-IO异步IO，一般都需要回调或者Promise的支持，比如node.js的IO都是异步IO

## 要解决什么问题
高并发，长连接。期望做到一个进程能维持住1w以上的TCP长连接。在之前Apache httpd的时代，这是做不到的，因为httpd对每个HTTP请求都会创建一个线程来处理，在面对1w以上的连接时，内存会被吃光，并且CPU做上下文切换也很低效。*最主要的原因，或者说需求是，这个1w长连接不是一直都活跃的，其实没有必要为不活跃的连接维持一个线程来处理*

## select和poll怎么解决？
- 由App保存所有打开的socket连接（其实就是一个file descriptor）
- App将这1w个连接交给内核（用户空间到内核空间拷贝），让内核找到是哪些socket发生了事件
- App根据内核找到的结果来处理哪些发生了事件的socket

## epoll怎么解决
- 先让App进行epoll_create()，不再在用户空间保持socket连接了
- App调用epoll_ctl注册socket
- App调用epoll_wait搜集发生时间的socket