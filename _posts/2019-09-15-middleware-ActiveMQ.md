---
layout: post
title: "中间件-ActiveMQ"
subtitle: '中间件之ActiveMQ整理总结'
author: "Kang"
date: 2019-09-15 19:26:04
header-img: "img/post-head-img/tit-4449764_1280.jpg"
catalog: true
tags:
  - 中间件
  - MQ
---
### MQ作用
解耦、消峰填谷

### MQ选择
- 消息事务性(Active MQ、Rocket MQ)  
- 消息延时性(Active MQ、Rocket MQ)  
- 消息顺序性(Rocket MQ)  
- 消息持久化  
- 消息失败重试   
- 监控性  
&emsp;&emsp;其中，延时性是用在延时执行某个动作场景下，相比将执行动作落库后定时器处理，不需要单独定时任务扫描，提高了执行时效性，也不会出现对数据库造成压力。  

### MQ面试常见问题

#### 持久化与非持久化
&emsp;&emsp;对于持久化消息，若发生MQ宕机后，在其重启后能恢复；而对于非持久化消息，虽然当在内存中达到一定量时会转化为临时文件进行存储，但是重启服务器后临时文件会被删除。  
&emsp;&emsp;非持久化临时文件大小做限制，当数据量达到阈值时整个系统都是可连接但是无法服务状态。  

#### 保障消息不丢失
&emsp;&emsp;消息丢失原因：    
- 生产者：生产者发送消息后再网络中丢失或者MQ接收到消息后，还没入盘发生宕机；
	+ 解决方案：生产者开启confirm机制，MQ入盘后（高速盘）和同步子节点(异步？)后回复确认消息，或者在必要的情况下开启事务消息；  	
- 消费者：在业务处理完后手动做ACK应答；  

#### 消费消息不均衡
&emsp;&emsp;MQ存在prefetch机制，即在消费者消费消息时，可能批量的处理消息(默认1000条)，这些消息被消费者批量拿走后为"已分配未消费"状态。所以解决方案是将prefetch设置为较低的值或者1；  

### 集群搭建
&emsp;&emsp;写在前面：目前使用的最佳方案是两种大的模式混合使用，在客户端根据规则先连接到具体的Cluster中，Cluster内部使用LevelDB模式进行管理。  

##### 共享文件模式&共享数据库模式
&emsp;&emsp;所有Broker共享数据文件(FILE||DB)，slave间歇性的尝试获取排它锁，获取到的Broker节点接管成为Master节点。  
##### Replicated LevelDB Store(首选)
&emsp;&emsp;这种模式下为Zookeeper管理+数据副本存储来构建。    
&emsp;&emsp;Master产生：Zookeeper管理了所有Broker，通过选举算法产生Master(优先选择数据最新的节点)，一旦Master异常，Zookeeper通过内部选举算法产生新的阶段；    
&emsp;&emsp;对外服务及数据同步：Master才能提供对客户端的连接，当Master接收到数据时，需要将数据<font color="red">发送给一半Slave节点</font>后，才能算成功并向Producer发送ACK确认消息。因为有实时同步数据的slave的存在，master不用担心数据丢失，所以leveldb会优先采用内存存储消息，异步同步到磁盘。所以该方式的activeMQ读写性能都最好，特别是写性能能够媲美非持久化消息。    

缺点：  
&emsp;&emsp;1)、占用的节点数过多，1个zookeeper集群至少3个节点，1个activemq集群也至少得3个节点，但其实正常运行时，只有一个master节点在对外响应，换句话说，花6个节点的成本只为了保证1个master节点的高可用，太浪费资源了；   
&emsp;&emsp;2)、一个master无法支撑大的数据量，所以master需要水平扩展，做成负责均衡；   

 
#### 基于NetworkConnector的Broker Clusters模式
&emsp;&emsp;Broker-Cluster的部署方式就可以解决负载均衡的问题。Broker-Cluster部署方式中，各个(Master)broker通过网络互相连接同步，也即共享了queue(中的消息)，保证消息同步。当某个Broker挂掉后，客户端会自动连接别的broker。   
[推荐阅读](https://blog.csdn.net/jinjin603/article/details/78657387)
### 集群服务

#### failover失效转移
&emsp;&emsp;在客户端配置failover，当一个集群Master节点失效时，客户端可以直接转移连接另一个节点。注意的是，客户端的failover需要将两个集群的Broker节点都配置上去；
#### 集群之间数据共享与同步
&emsp;&emsp;两个Cluster集群之间通过networkConnector进行数据同步。由于集群Master节点共享queue队列，消费端连接到任意一个集群时，若当前消息不存在，当前服务集群会去另一个集群中取消息。  

#### 集群负载均衡
&emsp;&emsp;这个地方所说的负载均衡貌似只是消费端来说的，其可以到任何一个集群去消费，对于Producer来说，还是发送到指定集群。。。   
  
