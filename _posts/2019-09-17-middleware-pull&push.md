---
layout: post
title: "中间件-消息队列之推&拉模型"
subtitle: '消息队列之推&拉模型简单总结'
author: "Kang"
date: 2019-09-17 15:30:52
header-img: "img/post-head-img/post-bg-kuaidi.jpg"
catalog: true
tags:
  - 中间件
  - MQ
---
## 消息中间件
&emsp;消息中间件的主要功能是消息的路由(Routing)和缓存(Buffering)。其中缓存可以理解为Queue消息的队列。生产端通过路由规则发送消息到不同queue，消费端根据queue名称消费消息。      

### 消费者获取消息模型
&emsp;&emsp;①Push方式：由消息中间件主动地将消息推送给消费者，可以尽可能快地将消息发送给消费者。   
&emsp;&emsp;②Pull方式：由消费者主动向消息中间件拉取消息，会增加消息的延迟，即消息到达消费者的时间有点长。       
Push方式坏处：  
&emsp;&emsp;如果消费者的处理消息的能力很弱(一条消息需要很长的时间处理)，而消息中间件不断地向消费者Push消息，消费者的缓冲区可能会溢出。       
Pull方式坏处：  
&emsp;&emsp;Pull有个缺点是，如果broker没有可供消费的消息，将导致consumer不断在循环中轮询，直到新消息到达。

####  ActiveMQ消费模型
&emsp;&emsp;ActiveMQ同时支持Push和Pull两种方式。   
##### 获取消息数据
&emsp;&emsp;对于Push：ActiveMQ使用了，prefetchSize规定了一次可以向消费者Push(推送)多少条消息。当推送消息的数量到达了prefetchSize规定的数值时，消费者还没有向消息中间件返回ACK，消息中间件将不再继续向消费者推送消息。消息数据少，但是每条消息消费时间长，则prefetch limit设置的越小。  
&emsp;&emsp;当prefetchSize设置成0，那么意味着，消息的获取<font color='blue'>不再是Push方式了，而是Pull方式了</font>   
       
##### 确认消息
&emsp;&emsp;消费者是每次消费一条消息之后就向消息中间件确认呢？还是采用“延迟确认”---即采用批量确认(optimizeAcknowledge)的方式(消费了若干条消息之后，统一再发ACK)。   
&emsp;&emsp;参数optimizeAck表示是否开启“优化ACK”，只有在为true的情况下，prefetchSize以及optimizeAcknowledgeTimeout(超时确认)参数才会有意义。如果prefetchACK为true，那么，prefetchSize必须大于0；当prefetchACK为false时，你可以指定prefetchSize为0(变为Pull方式)以及任意大小的正数。

##### 两种模式的使用
&emsp;&emsp;从是否阻塞来看，消费者有两种方式获取消息。同步方式和异步方式。    
&emsp;&emsp;同步方式(Push方式)使用的是ActiveMQMessageConsumer的receive()方法。而异步方式(Pull方式)则是采用消费者实现MessageListener接口，监听消息。    
使用同步方式receive()方法获取消息时，prefetch limit即可以设置为0，也可以设置为大于0

>
&emsp;&emsp;prefetch limit为零，意味着：“receive()方法将会首先发送一个PULL指令并阻塞，直到broker端返回消息为止，这也意味着消息只能逐个获取(类似于Request<->Response)”   
&emsp;&emsp;prefetch limit 大于零，意味着：“broker端将会批量push给client 一定数量的消息(<= prefetch)，client端会把这些消息(unconsumedMessage)放入到本地的队列中，只要此队列有消息，那么receive方法将会立即返回，当一定量的消息ACK之后，broker端会继续批量push消息给client端。”   
&emsp;&emsp;当使用MessageListener异步获取消息时，prefetch limit必须大于零了。因为，prefetch limit 等于零 意味着消息中间件不会主动给消费者Push消息，而此时消费者又用MessageListener被动获取消息(不会主动去轮询消息)。这二者是矛盾的。

####  Kafka消费模型
&emsp;&emsp;Kafka只支持消息持久化，消费端为pull 模式从 broker 中读取数据，消费状态和订阅关系由客户端负责维护，消息消费完后不会立即删除，会保留历史消息(故可能重复消费)。对于 Kafka 而言，pull 模式更合适(对消息消费实时性不高)，它可简化 broker 的设计，consumer 可自主控制消费消息的速率，同时 consumer 可以自己控制消费方式——即可批量消费也可逐条消费，同时还能选择不同的提交方式从而实现不同的传输语义。    
&emsp;&emsp;为了避免Pull的不停轮训缺点，Kafka有个参数可以让consumer阻塞直到新消息到达(当然也可以阻塞知道消息的数量达到某个特定的量这样就可以批量发送)。      
