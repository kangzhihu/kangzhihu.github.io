---
layout: post
title: "网络通信-Netty"
subtitle: '网络通信之Netty'
author: "Kang"
date: 2022-04-15 15:56:14
header-img: "img/post-head-img/night-1927265_1280.jpg"
catalog: true
tags:
  - 网络通讯
---
### Netty的理解：
&emsp;&emsp;netty是一个异步的事件驱动（不同的阶段，对应不同的回调方法），netty是异步非阻塞，但是其底层实现其实是同步的。     

### 网络示意图

![IO模型](https://raw.githubusercontent.com/kangzhihu/images/master/Netty-IO网络.jpg)    
Netty 抽象出两组线程池：
![Netty底层模型](https://raw.githubusercontent.com/kangzhihu/images/master/Netty-底层模型.jpg)
- BossGroup 专门负责接收客户端连接
- WorkerGroup 专门负责网络读写操作。
  NioEventLoop 表示一个不断循环执行处理 任务的线程， 每个 NioEventLoop 都有一个 selector， 用于监听绑定在其 上的 socket 网络通道。 NioEventLoop 内部采用串行化设计， 从消息的读取->解码->处理->编码->发送， 始终由 IO 线 程 NioEventLoop 负责。  

### Netty核心组件
#### ChannelHandler 及其实现类

ChannelHandler 接口定义了许多事件处理的方法， 我们可以通过重写这些方法去实现具 体的业务逻辑 我们经常需要自定义一个 Handler 类去继承 ChannelInboundHandlerAdapter， 然后通过 重写相应方法实现业务逻 辑， 我们接下来看看一般都需要重写哪些方法：

```java
public void channelActive(ChannelHandlerContext ctx)//通道就绪事件
public void channelRead(ChannelHandlerContext ctx, Object msg)//通道读取数据事件
public void channelReadComplete(ChannelHandlerContext ctx) //数据读取完毕事件
public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause)//通道发生异常事件
```

#### ChannelPipeline
ChannelPipeline 是一个 Handler 的集合， 它负责处理和拦截 inbound 或者 outbound 的事 件和操作， 相当于一个贯穿 Netty 的链。
![Netty底层模型-ChannelPipeline](https://raw.githubusercontent.com/kangzhihu/images/master/Netty-ChannelPipeline.jpg)
```java
// 把一个业务处理类（handler） 添加到链中的第一个位置
ChannelPipeline addFirst(ChannelHandler... handlers)
//把一个业务处理类（handler） 添加到链中的最后一个位置
ChannelPipeline addLast(ChannelHandler... handlers)
```

#### ChannelHandlerContext
这是事件处理器上下文对象 ， Pipeline链中的实际处理节点 。每个处理节点ChannelHandlerContext中包含一个具体的事件处理器ChannelHandler ，
同时 ChannelHandlerContext 中也绑定了对应的 pipeline和 Channel 的信息，方便对ChannelHandler 进行调用。 常用方法如下所示：
```java
ChannelFuture close()//关闭通道
ChannelOutboundInvoker flush()// 刷新
// 将数据写到 ChannelPipeline 中 当 前ChannelHandler 的下一个 ChannelHandler开始处理（出站）
ChannelFuture writeAndFlush(Object msg) 
```
#### ChannelFuture
表示 Channel 中异步 I/O 操作的结果， 在 Netty 中所有的 I/O 操作都是异步的， I/O 的调 用会直接返回， 调用者并不能立刻获得结果， 但是可以通过 ChannelFuture 来获取 I/O 操作 的处理状态。 常用方法如下所示：
```java
Channel channel()//返回当前正在进行 IO 操作的通道
ChannelFuture sync()// 等待异步操作执行完毕
```
#### EventLoopGroup 和其实现类 NioEventLoopGroup
EventLoopGroup 是一组 EventLoop 的抽象， Netty 为了更好的利用多核 CPU 资源， 一般 会有多个 EventLoop 同时工作， 每个 EventLoop 维护着一个 Selector 实例。 EventLoopGroup 提供 next 接口， 可以从组里面按照一定规则获取其中一个 EventLoop 来处理任务。 在 Netty 服务器端编程中， 我们一般都需要提供两个 EventLoopGroup， 例 如： BossEventLoopGroup 和 WorkerEventLoopGroup。
```java
public NioEventLoopGroup()//构造方法
public Future<?> shutdownGracefully()// 断开连接， 关闭线程
```

#### ServerBootstrap 和 Bootstrap
ServerBootstrap 是 Netty 中的服务器端启动助手，通过它可以完成服务器端的各种配置； Bootstrap 是 Netty 中的客户端启动助手， 通过它可以完成客户端的各种配置。 常用方法如下 所示
```java
- public ServerBootstrap group(EventLoopGroup parentGroup, EventLoopGroup childGroup)//该方法用于服务器端， 用来设置两个 EventLoop
- public B group(EventLoopGroup group) // 该方法用于客户端， 用来设置一个 EventLoop
- public B channel(Class<? extends C> channelClass)// 该方法用来设置一个服务器端的通道实现
- public <T> B option(ChannelOption<T> option, T value)// 用来给 ServerChannel 添加配置
- public <T> ServerBootstrap childOption(ChannelOption<T> childOption, T value)// 用来给接收到的通道添加配置
- public ServerBootstrap childHandler(ChannelHandler childHandler)// 该方法用来设置业务处理类（自定义的 handler）
- public ChannelFuture bind(int inetPort) // 该方法用于服务器端， 用来设置占用的端口号
- public ChannelFuture connect(String inetHost, int inetPort) //该方法用于客户端， 用来连接服务器端
```


