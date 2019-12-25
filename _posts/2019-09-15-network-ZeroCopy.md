---
layout: post
title: "网络通信-零拷贝"
subtitle: '网络通信之零拷贝整理总结'
author: "Kang"
date: 2019-09-15 15:56:14
header-img: "img/post-head-img/night-1927265_1280.jpg"
catalog: true
tags:
  - 网络通讯
---
### 对于IO和零拷贝的理解：
&emsp;&emsp;IO决定了用户如何去感知(主动?被动?阻塞?非阻塞?)到数据的变更；而拷贝方式决定了数据如何被用户拿到。零拷贝解决的是内存到内存的拷贝，只是一个为系统内存一个为应用内存，所以将两者优化后合并。   

### 拷贝示意图
非零拷贝示意图：  
![非零拷贝](https://raw.githubusercontent.com/kangzhihu/images/master/%E9%9D%9E%E9%9B%B6%E6%8B%B7%E8%B4%9D.png)    
零拷贝示意图：
![零拷贝](https://raw.githubusercontent.com/kangzhihu/images/master/%E9%9B%B6%E6%8B%B7%E8%B4%9D.png)    


&emsp;&emsp;从上图中可以清楚的看到，Zero Copy的模式中，避免了数据在用户空间和内存空间之间的拷贝，从而提高了系统的整体性能。Linux中的sendfile()以及Java NIO中的FileChannel.transferTo()方法都实现了零拷贝的功能，而在Netty中也通过在FileRegion中包装了NIO的FileChannel.transferTo()方法实现了零拷贝。

[推荐阅读-待整理](https://my.oschina.net/plucury/blog/192577)

### 通讯零拷贝
#### 传统的发送
在发送数据的时候，传统的实现方式是：
```java
File.read(bytes)
Socket.send(bytes)
```
这种方式需要四次数据拷贝和四次上下文切换：
1. 数据从磁盘读取到内核的read buffer
2. 数据从内核缓冲区拷贝到用户缓冲区
3. 数据从用户缓冲区拷贝到内核的socket buffer
4. 数据从内核的socket buffer拷贝到网卡接口的缓冲区

&emsp;&emsp;明显上面的第二步和第三步是没有必要的，通过java的FileChannel.transferTo方法，可以避免上面两次多余的拷贝（当然这需要底层操作系统支持）
1. 调用transferTo,数据从文件由DMA引擎拷贝到内核read buffer
2. 接着DMA从内核read buffer将数据拷贝到网卡接口buffer   
这两次操作都不需要CPU参与，所以就达到了零拷贝。

#### Netty零拷贝

Netty中也用到了FileChannel.transferTo方法，所以Netty的零拷贝也包括上面将的操作系统级别的零拷贝。除此之外，在ByteBuf的实现上，Netty也提供了零拷贝的一些实现。关于ByteBuffer，Netty提供了两个接口:
1. ByteBuf
2. ByteBufHolder

其中，ByteBuf，Netty提供了多种实现：
1. Heap ByteBuf:直接在堆内存分配
2. Direct ByteBuf：直接在内存区域分配而不是堆内存
3. CompositeByteBuf：组合Buffer
