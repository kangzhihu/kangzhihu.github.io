---
layout: post
title: "中间件-RocketMQ与kafka比较"
subtitle: '中间件RocketMQ与kafka比较'
author: "Kang"
date: 2022-01-02 12:26:04
header-img: "img/post-head-img/grass.jpg"
catalog: true
tags:
  - 中间件
  - MQ
  - RocketMQ
  - kafka
---

### RocketMQ与kafka的不同

##### 数据可靠性
RocketMQ：支持异步实时刷盘、同步刷盘、同步复制、异步复制。
kafka：使用异步刷盘方式，异步复制/同步复制。  
总结：  
1、RocketMQ支持kafka所不具备的“同步刷盘”功能，在单机可靠性上比kafka更高，不会因为操作系统Crash而导致数据丢失。    
2、kafka的同步replication理论上性能低于RocketMQ的replication，这是因为kafka的数据以partition为单位，这样一个kafka实例上可能多上百个partition。而一个RocketMQ实例上**只有一个partition**，RocketMQ可以充分利用IO组的commit机制，批量传输数据。同步replication与异步replication相比，同步replication性能上损耗约20%-30%。  

一句话概括：RocketMQ新增了同步刷盘机制，保证了可靠性；一个RocketMQ实例只有一个partition, 在replication时性能更好。  


##### 性能对比
1、kafka单机写入TPS约在百万条/秒(多文件并发写入)，消息大小为10个字节。  
2、RocketMQ单机写入TPS单实例约7万条/秒(单文件写入)，若单机部署3个broker，可以跑到最高12万条/秒，消息大小为10个字节。  
总结：
 - kafka的单机TPS能跑到每秒上百万，是因为Producer端将多个小消息合并，批量发向broker，且在写入的时候，kafka多文件并发写。 
 - RocketMQ写入性能上不如kafka, 主要因为kafka主要应用于日志场景，而RocketMQ应用于业务场景，为了保证消息必达牺牲了性能，且基于线上真实场景没有在RocketMQ层做消息合并，推荐在业务层自己做。  
> Rocketmq需要业务方对发送的每条消息做确认，保障投递成功，所以不能再发送端进行消息合并。

##### 单机支持的队列数
1、kafka单机若超过了64个partition/队列，CPU load会发生明显飙高，partition越多，CPU load越高，发消息的响应时间变长。   
2、RocketMQ单机支持最高5万个队列，CPU load不会发生明显变化。  
 - 还没理解到。。
队列多有什么好处呢？  
1、单机可以创建更多个topic, 因为每个topic都是有一组队列组成。    
2、消费者的集群规模和队列数成正比，队列越多，消费类集群可以越大。  

一句话概括：RocketMQ支持的队列数远高于kafka支持的partition数，这样RocketMQ可以支持更多的consumer集群。  

##### 消费失败重试
1、kafka不支持消费失败重试。  
2、RocketMQ消费失败支持定时重试，每次重试间隔时间顺延。  

##### 严格保证消息有序
1、kafka可保证同一个partition上的消息有序，但一旦broker宕机，就会产生消息乱序。  
2、Rocket支持严格的消息顺序，一台broker宕机，发送消息会失败，但不会乱序。举例：MySQL的二进制日志分发需要保证严格的顺序。  

一句话概括：kafka不保证消息有序，RocketMQ可保证严格的消息顺序，在broker端，前面的消息未消费，后面的消息也不会被消费掉，即使单台Broker宕机，仅会造成消息发送失败，但不会消息乱序。  

#### Rebalance异同
&emsp;&emsp;RocketMQ与Kafka Rebalance机制类似，二者Rebalance分配都是在客户端进行，不同的是
- Kafka：会在消费者组的多个消费者实例中，选出一个作为Group Leader，由这个Group Leader来进行分区分配，分配结果**通过Cordinator(特殊角色的broker)同步给其他消费者**。相当于Kafka的分区分配只有一个大脑，就是Group Leader。
- RocketMQ：每个消费者，自己负责给自己分配队列，相当于每个消费者都是一个大脑。


##### 定时消息
1、kafka不支持定时消息  
2、开源版本的RocketMQ仅支持定时级别，定时级别用户可定制  

##### 分布式事务消息
1、kafka不支持分布式事务消息。  
2、RocketMQ支持分布式事务消息。  

##### 集群处理
1、kafka通过zookeeper来进行协调,具备选举能力 (趋向于高可用性,易维护 )  
```shell
1. 某个partition的master挂了，该partition对应的某个slave会升级为主对外提供服务
2. 缺陷: 异步Replication下自动选主数据可能丢失
```
2、rocketMq通过自身的namesrv进行协调, 不具备选主能力 (去向于数据一致性, 数据不丢失)   
```shell
1. Mq只能保证当一个broker挂了，把原本写到这个broker的请求迁移到其他broker上面，而并不是这个broker对应的slave升级为主
2. 维护上没有那么方便
```

##### 消息回溯
1、kafka可按照消息的offset来回溯消息。  
2、RocketMQ支持按照时间来回溯消息，精度到毫秒，例如从一天的几点几分几秒几毫秒来重新消费消息。  

总结：RocketMQ按时间做回溯消息的典型应用场景为，consumer做订单分析，但是由于程序逻辑或依赖的系统发生故障等原因，导致今天处理的消息全部无效，需要从昨天的零点重新处理。  
