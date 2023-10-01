---
layout: post
title: "网络通信-Netty"
subtitle: '网络通信-Netty简单总结'
author: "Kang"
date: 2019-09-15 16:56:14
header-img: "img/post-head-img/photographer-920128_1280.jpg"
catalog: true
tags:
  - 网络通讯
  - IO
  - Netty
---
&emsp;&emsp;Netty 是一个 异步 事件驱动 的网络应用框架，用于快速开发高性能、可扩展协议的服务器和客户端  

### 1. 反应器设计模式
反应器设计模式(Reactor pattern)是一种为处理服务请求并发 提交到一个或者多个服务处理程序的事件设计模式。当请求抵达后，服务处理程序使用解多路分配策略，然后同步地派发这些请求至相关的请求处理程序。   
> Reactor 模式使用的是异步非阻塞 IO，NIO线程用于监听套接字（Socket）的读写事件，负责与客户端建立连接、接收请求、发送响应等底层的网络操作。整体来说主 Reactor 线程将读写事件分发给用户线程池中的用户线程来执行业务逻辑。  
#### 1.1 单线程模型
Reactor 单线程模型，指的是所有的 IO 操作都在同一个 NIO 线程上面完成，NIO 线程的职责如下：
1. 作为 NIO 服务端，接收客户端的 TCP 连接；
2. 作为 NIO 客户端，向服务端发起 TCP 连接；
3. 读取通信对端的请求或者应答消息；
4. 向通信对端发送消息请求或者应答消息。  
![IO-Reactor-单线程模式](https://raw.githubusercontent.com/kangzhihu/images/master/IO-Reactor-单线程模式.png)
&emsp;&emsp;由于 Reactor 模式使用的是异步非阻塞 IO，所有的 IO 操作都不会导致阻塞，理论上一个线程可以独立处理所有 IO 相关的操作。从架构层面看，一个 NIO 线程确实可以完成其承担的职责。例如，通过 Acceptor 类接收客户端的 TCP 连接请求消息，链路建立成功之后，通过 Dispatch 将对应的 ByteBuffer 派发到指定的 Handler 上进行消息解码。用户线程可以通过消息编码通过 NIO 线程将消息发送给客户端。  
&emsp;&emsp;对于一些小容量应用场景，可以使用单线程模型。但是**对于高负载、大并发的应用场景却不合适**。

#### 1.2 多线程模型
Rector 多线程模型与单线程模型最大的区别就是有一组 NIO 线程处理 IO 操作，它的原理图如下：

![IO-Rector-多线程模式](https://raw.githubusercontent.com/kangzhihu/images/master/IO-Rector-多线程模式.png)
Reactor 多线程模型的特点： 
1. 有专门一个 NIO 线程 Acceptor 线程用于监听服务端，接收客户端的 TCP 连接请求；
2. 网络 IO 操作 - 读、写等由一个 NIO 线程池负责，线程池可以采用标准的 JDK 线程池实现，它包含一个任务队列和 N 个可用的线程，由这些 NIO 线程负责消息的读取、解码、编码和发送；
3. 1 个 NIO 线程可以同时处理 N 条链路，但是 1 个链路只对应 1 个 NIO 线程，防止发生并发操作问题。 
> 整体来说在单线程基础上，将网络的通讯功能和数据的读写、编码、解码和发送给业务线程等进行了剥离，由单独的线程池进行处理，主线程功能更加聚焦。  

#### 1.3 主从多线程模型
&emsp;&emsp;主从 Reactor 线程模型的特点是：服务端用于接收客户端连接的不再是个 1 个单独的 NIO 线程，而是一个独立的 NIO 线程池。 Acceptor 接收到客户端 TCP 连接请求处理完成后（可能包含接入认证等），将新创建的 SocketChannel 注册到 IO 线程池（sub reactor 线程池）的某个 IO 线程上，由它负责 SocketChannel 的读写和编解码工作。 Acceptor 线程池仅仅只用于客户端的登陆、握手和安全认证，一旦链路建立成功，就将链路注册到后端 subReactor 线程池的 IO 线程上，由 IO 线程负责后续的 IO 操作

![IO-Rector-主从多线程模式](https://raw.githubusercontent.com/kangzhihu/images/master/IO-Rector-主从多线程模式.png)
它的工作流程总结如下：  
1. 从主线程池中随机选择一个 Reactor 线程作为 Acceptor 线程，用于绑定监听端口，接收客户端连接；
2. Acceptor 线程接收客户端连接请求之后创建新的 SocketChannel ，将其注册到主线程池的其它 Reactor 线程上，由其负责接入认证、IP 黑白名单过滤、握手等操作；
3. 步骤 2 完成之后，业务层的链路正式建立，将 SocketChannel 从主线程池的 Reactor 线程的多路复用器上摘除，重新注册到 Sub 线程池的线程上，用于处理 I/O 的读写操作。
> 为了更加快速的响应网络连接请求，在高网络连接下，单一的Reactor主线程不能满足要求，因此当一个新的客户端连接请求到达时，操作系统会将这个事件通知给主线程池(其实大部分情况下为单线程)。主线程池中的主 Reactor 线程会被唤醒然后随机选择一个从 Reactor 线程作为 Acceptor 线程去处理这个连接。这个从 Reactor 线程负责接受并处理连接的建立。主线程池的监听能力和从 Reactor 线程池的处理能力。  
> 一旦连接建立成功，从 Reactor 线程会将这个连接注册到合适的事件处理器（Handler）上，以后续处理连接上的数据交互。  
> 主线程池主要负责监听和分发连接请求，而从 Reactor 线程池则负责具体的连接处理。

### Netty 

#### Netty 为什么性能好？
1. 纯异步：Reactor 线程模型
2. IO 多路复用
3. GC 优化：更少的分配内存、池化（Pooling）、复用、选择性的使用 sun.misc.Unsafe
4. 更多的硬件相关优化（mechanical sympathy）
5. 内存泄漏检测
6. “Zero Copy”  

Netty 启动以及链接建立过程
![IO-Netty启动过程](https://raw.githubusercontent.com/kangzhihu/images/master/IO-Netty启动过程.png)


####  Selector
&emsp;&emsp;Selector选择器是一个通道服务，应用程序事先告诉选择器：“我对某些通道的事件感兴趣，如可读、可写等“，选择器在接受了一个或多个对通道的委托后，开始选择工作，它的选择工作就完全交给操作系统，linux下即为poll或epoll。<font color="red">AIO中的高效句柄管理，就是通过epoll来实现的</font>

#### 空轮询
&emsp;&emsp;设置了Selector.select(timeout)超时时间，一般4种情况可以跳出阻塞：
1. 有事件发生；
2. wakeup；
3. 超时；
4. 空轮询bug

&emsp;&emsp;前两种返回值不为0，可以跳出循环，但是后两种当超过一定次数(512次)，则处于一直轮询没处理任何事件，此时可能底层的操作系统的Selector实现遇到了一些问题，无法正常地检测到通道（Channel）的事件状态变化，导致Selector一直在等待新事件而不执行任何实际的工作。这种情况可能出现在高负载、资源泄漏、不正常的通道注册/取消注册等情况下。
所以通过新建一个Selector也即相当于重置Selector 的内部状态来帮助解决可能出现的空轮询等问题。
```java
long currentTimeNanos = System.nanoTime();
for (;;) {
    // 1.定时任务截止事时间快到了，中断本次轮询
    //...
    // 2.轮询过程中发现有任务加入，中断本次轮询
    //...
    // 3.阻塞式select操作
    selector.select(timeoutMillis);
    // 4.解决jdk的nio bug
    long time = System.nanoTime();
    if (time - TimeUnit.MILLISECONDS.toNanos(timeoutMillis) >= currentTimeNanos) {
        selectCnt = 1;
    } else if (SELECTOR_AUTO_REBUILD_THRESHOLD > 0 &&
            selectCnt >= SELECTOR_AUTO_REBUILD_THRESHOLD) {

        rebuildSelector();
        selector = this.selector;
        selector.selectNow();
        selectCnt = 1;
        break;
    }
    currentTimeNanos = time;
    //...
 }
```
