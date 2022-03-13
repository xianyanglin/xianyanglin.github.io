---
title: netty如何支持三种Reactor
date: 2022-03-13 09:25:00
tags:
  - Netty
summary: 什么是Reactor?以及如何在Netty中使用Reactor模式?
categories: Netty
---
### 什么是Reactor及三种版本
* Reactor单线程
  客户端请求到达服务端，服务端创建一个线程处理读取、解码、业务处理、编码、返回响应一整个流程

* Reactor多线程模式
  将解码、业务处理、编码的这三个操作都由工作线程池来处理

* 主从Reactor多线程模式

主Reactor负责接收请求，从Reactor负责读请求和写响应，工作线程负责解码、业务处理、编码。

Reactor是一种开发模式，模式的核心流程：
注册感兴趣的事件 ->扫描是否有感兴趣的事件发生 ->事件发生后做出相应处理

| client/server  | SocketChannel/ServerSocketChannel  | OP_ACCEPT  | OP_CONNECT  | OP_WRITE  | OP_READ  |
|:----------|:----------|:----------|:----------|:----------|:----------|
| client    | SocketChannel    |    | Y    | Y    | Y    |
| server    | ServerSocketChannel    | Y    |      |      |     |
| server    | SocketChannel    |     |      | Y    | Y    |



### 如何在Netty中使用Reactor模式

* Reactor单线程模式：
```
EventLoopGroup eventGroup = new NioEventLoopGroup(1);
ServerBootstrap serverBootstrap = new ServerBootstrap();
serverBootstrap.group(eventGroup);
```

* 非主从Reactor多线程模式
```
EventLoopGroup eventGroup = new NioEventLoopGroup();
ServerBootstrap serverBootstrap = new ServerBootstrap();
serverBootstrap.group(eventGroup);
```

* 主从Reactor多线程模式
```
EventLoopGroup bossGroup = new NioEventLoopGroup();
EventLoopGroup workerGroup = new NioEventLoopGroup();
ServerBootstrap serverBootstrap = new ServerBootstrap();
serverBootstrap.group(bossGroup, workerGroup)
```


### 解析Netty对Reactor模式支持的常见疑问

Netty如何支持主从Reactor模式？

通过传递过来的channel创建子Channel，两种SocketChannel绑定到两个Gruop里面去，这样就完成了主从Reactor模式的支持。

为什么说Netty的main reactor大多数并不能用到一个线程组，只能线程组里的一个？
因为一个服务器一般来说只用绑定一个地址，一个端口


Netty给Channel分配NIO event loop的规则是什么？

1、增值、取模、取正值

2、executors总数是2的幂次方然后&运算


通常模式的NIO实现多路复用器是怎么跨平台的？
通过JDK 读取平台信息 ，创建适合不同平台的实现







