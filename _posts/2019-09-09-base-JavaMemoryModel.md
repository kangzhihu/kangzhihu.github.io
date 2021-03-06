---
layout: post
title: "基础-java内存模型"
subtitle: 'java内存模型粗理解'
author: "Kang"
date: 2019-09-09 11:10:47
header-img: "img/post-head-img/white-flowers-1639230-1280x720.jpg"
catalog: true
tags:
  - Java基础
---
### java内存模型
&emsp;&emsp;由于CUP的速度远远高于内存的速度，所以为了加快处理，CUP自己携带一块高速缓存，在处理时，先将数据从主存中copy一份到高速缓存中，后面再刷入主存。   
&emsp;&emsp;这种模式就导致一个问题：在多处理器下，工作缓存中和主存中数据不一致。    

![Java模型](https://raw.githubusercontent.com/kangzhihu/images/master/%E5%9F%BA%E7%A1%80-JAVA%E5%86%85%E5%AD%98%E6%A8%A1%E5%9E%8B.png)

&emsp;&emsp;上图为一个简单地示例，存在变量flag,初始flag=false，线程2修改为true；当总线嗅探到后，将工作内存中的数据置为失效。  
    
 其中，涉及到的命令如下：
- read(读取)：从主内存读取数据；
- load(载入)：将主内存读取到的数据写入工作内存；(读取局部变量表？)
- use(使用)：从工作内存读取数据来计算；(从局部变量表读取到操作数栈？)
- assign(赋值)：将计算好的值重新赋值到工作内存；
- stroe(存储)：将工作内存写入主内存；(简单理解为进入数据总线?)
- write(写入)：将store过去的变量赋值给主存中的变量；(简单理解为从数据总线写入主存？)
- lock：将主存变量加锁，标识为线程独占状态；
- unlock：释放主存独占锁；   
&emsp;&emsp;这些命令之间不构成原子性。

### Volatile缓存
&emsp;&emsp;添加Volatile关键字的变量，在发生变化后，将在底层硬件级别将当前处理器缓存行的值<font color="red"><b>立刻</b></font>写会到系统，并触发总线嗅探机制。 

#### 可见性
1. 如何处理其它线程并发修改或者stroe命令触发嗅探机制后，在当前线程write完成之前，其它线程再次的发生load加载到旧值；  
   &emsp;&emsp;解决方式：在store操作之前，当前线程会对主内存中的缓存行做互斥lock，直到write写完成，进行unlock操作。store一旦经过总线，则会触发其它CPU嗅探，失效自己处理器值，此时由于主内存中存在互斥锁，其它线程将不能进行读取与修改；
   
#### 原子性
2. 为何volatile不能保证原子性  
   &emsp;&emsp;原因：多处理器并发修改相同volatile变量时，若存在更新处理器缓存值后同时执行store操作，则会并发竞争主存lock锁，若线程A竞争到锁，则在A完成write写入后，会触发嗅探机制导致线程B理器缓存值失效。    
   &emsp;&emsp;两个线程若存在并发写(触发store阶段)，则虽然另一个线程的工作缓存值失效，但是存在覆盖情况，这就是为啥经典的并发加时值偏小的原因。
