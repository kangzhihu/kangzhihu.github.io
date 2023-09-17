---
layout: post
title: "中间件-RocketMQ"
subtitle: '中间件RocketMQ总览'
author: "Kang"
date: 2021-08-17 12:26:04
header-img: "img/post-head-img/grass.jpg"
catalog: true
tags:
  - 中间件
  - MQ
  - RocketMQ
---
> 事务管理，异步解耦，削峰填谷，数据同步等应用场景

![RocketMq整体框架图](https://raw.githubusercontent.com/kangzhihu/images/master/RocketMQ%E6%A1%86%E6%9E%B6%E5%9B%BE.jpg)  
1、NameServer 之间互不通信，无法感知对方的存在。  
2、Broker 服务会与每台 NameServer 保持长连接,每30s向 NameServer 发送心跳包。  
3、生产者&消费者与 NameServer 集群中的一台服务器建立长连接，并持有整个 NameServer 集群的列表，且定期去拉取主题的路由信息。   
4、rocketMq通过自身的namesrv进行协调，当一个broker挂了，会把当前请求迁移到其他Broker上去，而不是slave升级为Master  
> 所以都不会立即感知到变更；  
> 发送的消息中已经携带了QueueId，标识当前消息会被放到哪个ConsumeQueue中。

### 消息存储文件设计
&emsp;&emsp;消息存储主要体现在三个文件中：CommitLog（真正存储消息体的地方）、ConsumeQueue（某个Topic下某个Queue的消息索引信息）、IndexFile（通过key或时间区间来查询消息的索引文件）。  
全图：  
![RocketMq存储文件](https://raw.githubusercontent.com/kangzhihu/images/master/RocketMQ文件全图.png)
简图：  
![RocketMq存储简图](https://raw.githubusercontent.com/kangzhihu/images/master/RocketMQ文件简图.png)
细图：
![RocketMq存储简图](https://raw.githubusercontent.com/kangzhihu/images/master/RocketMQ文件细图.png)

##### 文件处理
[RocketMq写文件参阅](https://blog.csdn.net/sjzsylkn/article/details/121897370?spm=1001.2014.3001.5502)

##### CommitLog 文件
&emsp;&emsp;**真正存储消息内容的地方**，一个Broker中，所有的topic消息都混在在一起的。  
&emsp;&emsp;消息随着到达 Broker 的顺序写入 CommitLog 文件，每个文件默认为1G，文件的命名和kafka一样，使用该存储在消息文件中的第一个全局偏移量来命名文件。  
&emsp;&emsp;producer发送消息后，消息先保存到commitLog，然后重投递线程会扫描是否有新消息被保存到 CommitLog，如果有则将这条消息查出来，执行重投递逻辑，构建该消息的索引(包括 ConsumeQueue 和 IndexFile）。


##### ConsumeQueue 文件
![RocketMQ文件ConsumeQueue](https://raw.githubusercontent.com/kangzhihu/images/master/RocketMQ文件ConsumeQueue.png)

可以把 ConsumeQueue 看作是索引项组成的数组，根据QueueId进行追加。  
>也可以看出，消息在全局上不是有序的。  

![RocketMq文件ConsumeQueue节点结构](https://raw.githubusercontent.com/kangzhihu/images/master/RocketMq文件ConsumeQueue节点结构.png)
通过存入节点固定大小，能使用访问类似数组下标的方式来快速定位条目。  

##### IndexFile 文件
&emsp;&emsp;IndexFile文件主要是服务于通过Key进行消息查询的功能。  

##### 消息消费
&emsp;&emsp;客户端发起消息消费请求，请求码为RequestCode.PULL_MESSAGE，对应的处理类为PullMessageProcessor。  
&emsp;&emsp;Broker 在收到客户端的请求之后，会根据topic和queueId、消息消费进度(ConsumeQueue 逻辑偏移量)，通过逻辑偏移量logicOffset* 20可以快速定位到在哪个ConsumeQueue的哪个起始位置，然后读取20个字节即得到了一个条目。  
&emsp;&emsp;Pull模式下，PullMessageService 拉取完一批消息后，将消息提交到线程池后会“马不蹄停”去拉下一批消息，如果此时消息消费线程池处理速度很慢，处理队列中的消息会越积越多。
##### 消息积压  
&emsp;&emsp;消费端的限流机制:
- 消息堆积数量控制：当消息消费处理队列中的消息条数超过1000条会触发消费端的流控，其具体做法是放弃本次拉取动作，并且延迟50ms后将放入该拉取任务放入到pullRequestQueue中，每1000次流控会打印一次消费端流控日志。  
- 消息堆积大小：处理队列中堆积的消息总内存大小超过100M，同样触发一次流控。  
通过调整发送方速率、调整分区数提高并行处理能力、监控和告警、压力测试、定期审查、容量规划、消费端异常处理等手段去规避积压。  

### 文件清理
&emsp;&emsp;broker 文件清理主要是清理commitlog ， ConsumeQueue ， indexFile。RocketMQ默认是清理72小时之前的消息， 默认是凌晨4点触发清理， 除非磁盘空间占用到75% 以上了。  
&emsp;&emsp;RocketMQ拿到所有的MappedFile（抛去现在正在使用的）比对每个MappedFile的最后一条消息的时间，如果是72小时之前的就把MappedFile对应的文件删除了，销毁对应MappedFile，这种情况的话只要你MappedFile 最后一条消息还在存活实效内的话，它就不会清理这个MappedFile。  
&emsp;&emsp;清理完成commitlog 之后，就会拿到commitlog中最小的offset ，然后去ConsumeQueue与indexFile中把小于offset 的记录删除掉。 