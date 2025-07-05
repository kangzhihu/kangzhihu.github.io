---
layout: post
title: "中间件-RocketMQ"
subtitle: '中间件RocketMQ之Offset(一)'
author: "Kang"
date: 2020-01-15 13:26:04
header-img: "img/post-head-img/gingko-tree.jpg"
catalog: true
tags:
  - 中间件
  - MQ
  - RocketMQ
---
### 集群管理总结
&emsp;&emsp;1、主独立NameServer(元数据管理，相互独立，与每个Broker通讯)；    
&emsp;&emsp;2、RebalanceService，位于每个消费端，之间也不存在通讯，从NameServer获取Topic路由和消费者组成员元数据动态计算当前实例应消费的队列。  
&emsp;&emsp;3、考虑到RebalanceService和NameServer均独立，所以允许心跳、锁检测和Rebalance再平衡过程快速恢复达到最终一致性  
&emsp;&emsp;6、那么为了数据不一致导致的消费冲突问题，Broker端对消息队列的消费权只允许单个消费者持有锁（通过消息队列锁定机制），持有锁后进行消费。  

> - RocketMq简化了管理，除了Broker额外管理消费者的offset信息以及提供集群元数据信息的查询，消费组的管理基本上不参与；  
> - RocketMQ通过灵活的复制和事务设计，牺牲部分强一致性（AP方案），换取业务流程中的高可靠性和连续性，避免消息丢失和业务中断；
> - RocketMQ需要管理事务，流程追踪、MessageQueue index管理等增加集群的高可用性和连续性，所以需要识别数据，没办法使用零拷贝，所以相比较kafka来说性能低些；

#### <font color = "red">为什么 Redis 选择无中心而 Kafka、RocketMQ 没有？</font>
> 
> 1、偏向AP，Redis Cluster 的目标是高性能、高可用的键值存储，元数据管理相对简单，节点状态变化和槽位映射更新的管理复杂度较低，适合利用 Gossip 去中心化实现。  
> 2、偏向CP，Kafka 和 RocketMQ 除了存储消息外，还承担复杂的分区管理、消费者负载均衡、事务管理及强一致性保证，需要有一个集中协调的组件保证全局状态的准确和一致。

| &zwnj;**维度**&zwnj; | &zwnj;**Redis Cluster**&zwnj; | &zwnj;**Kafka**&zwnj; | &zwnj;**RocketMQ**&zwnj; |
| --- | --- | --- | --- |
| &zwnj;**元数据管理**&zwnj; | 无中心化，所有节点有完整集群元信息副本 | 中心化，由单一Controller节点维护（选举产生） | 中心化，NameServer和Broker Controller负责 |
| &zwnj;**元数据内容**&zwnj; | 节点列表、节点状态、槽(slot)分配、拓扑信息 | Topic元数据，Partition分配，Leader & ISR列表 | Topic路由信息，Broker状态，消息队列元数据 |
| &zwnj;**集群协调方式**&zwnj; | Gossip协议交换状态，分布式投票选举故障转移 | Controller节点负责，Zookeeper协调选举管理 | NameServer + Broker Controller协调 |
| &zwnj;**故障检测**&zwnj; | 节点相互gossip检测，主观+客观下线判定 | Zookeeper监控Broker和Controller状态 | 心跳机制+控制节点监控 |
| &zwnj;**故障恢复与选举**&zwnj; | 从节点发起选举，通过分布式投票完成故障转移 | Controller节点通过Zookeeper发起Partition Leader选举 | Broker Controller进行Broker Leader选举 |
| &zwnj;**消费者协调**&zwnj; | 无中心化，客户端独立维护元数据并通过MOVED错误重定向请求 | 中心化，有Group Coordinator管理消费者组成员和Rebalance | 有消费组协调机制，依赖NameServer/Broker Controller |
| &zwnj;**扩展性**&zwnj; | 高，无中央协调瓶颈，节点数量增大影响较小 | 中等，Controller及Zookeeper可能成为瓶颈 | 中等，中心节点会有压力 |
| &zwnj;**容错和可用性**&zwnj; | 高，去中心化避免单点故障 | 中等，依赖Controller和Zookeeper的高可用部署 | 中等，依赖中心组件高可用 |
| &zwnj;**设计侧重点**&zwnj; | 轻量、高性能、简单容错 | 强一致性、可靠的消息存储与消费，复杂协调 | 异步消息传递，高吞吐与消费协调 |
| &zwnj;**设计难度**&zwnj; | 相对简单，利用一致性哈希和Gossip | 复杂，需要实现复杂的分布式协调算法 | 复杂，结合Broker与NameServer多组件协调处理 |


### 消息的两种消费模式
- Pull模式：由消费者客户端主动向消息中间件（MQ消息服务器代理）拉取消息；(**消费端消费慢问题**)
- Push模式：由消息中间件（MQ消息服务器代理）主动地将消息推送给消费者；(**消息延迟与忙等待问题**)

RocketMQ的消费方式：是基于拉模式拉取消息的，在这其中有一种长轮询机制（对普通轮询的一种优化），来平衡上面Push/Pull模型的各自缺点。基本设计思路是：消费者如果第一次尝试Pull消息失败（比如：Broker端没有可以消费的消息），并不立即给消费者客户端返回Response的响应，而是先hold住并且挂起请求（将请求保存至pullRequestTable本地缓存变量中），然后Broker端的后台独立线程—PullRequestHoldService会从pullRequestTable本地缓存变量中不断地去取，具体的做法是查询待拉取消息的偏移量是否小于消费队列最大偏移量，如果条件成立则说明有新消息达到Broker端（这里，在RocketMQ的Broker端会有一个后台独立线程—ReputMessageService不停地构建ConsumeQueue/IndexFile数据，同时取出hold住的请求并进行二次处理），则通过重新调用一次业务处理器—PullMessageProcessor的处理请求方法—processRequest()来重新尝试拉取消息（此处，每隔5S重试一次，默认长轮询整体的时间设置为30s）。



### Offset

#### read读取

&emsp;&emsp;我们重点看下Pull模式下的Offset。DefaultMQPullConsumer为消费端代码，可以看到其中存在大致逻辑：

```java
public class DefaultMQPullConsumerImpl implements MQConsumerInner {
    private MQClientInstance mQClientFactory;
    private OffsetStore offsetStore;
    
    public synchronized void start() throws MQClientException {
        switch (this.serviceState) {
            case CREATE_JUST:
               // ....忽略代码...
                if (this.defaultMQPullConsumer.getOffsetStore() != null) {
                    this.offsetStore = this.defaultMQPullConsumer.getOffsetStore();
                } else {
                    switch (this.defaultMQPullConsumer.getMessageModel()) {
                        case BROADCASTING:
                            this.offsetStore = new LocalFileOffsetStore(this.mQClientFactory, this.defaultMQPullConsumer.getConsumerGroup());
                            break;
                        case CLUSTERING:
                            this.offsetStore = new RemoteBrokerOffsetStore(this.mQClientFactory, this.defaultMQPullConsumer.getConsumerGroup());
                            break;
                        default:
                            break;
                    }
                    this.defaultMQPullConsumer.setOffsetStore(this.offsetStore);
                }
				// ....忽略代码...
        }

    }
}
```

&emsp;&emsp;在cluster模式下的实现类是RemoteBrokerOffsetStore。我们看下RemoteBrokerOffsetStore：

```java
public class RemoteBrokerOffsetStore implements OffsetStore {
    
    private ConcurrentMap<MessageQueue, AtomicLong> offsetTable =
        new ConcurrentHashMap<MessageQueue, AtomicLong>();
    
    @Override
    public long readOffset(final MessageQueue mq, final ReadOffsetType type) {
        if (mq != null) {
            switch (type) {
                case MEMORY_FIRST_THEN_STORE:
                case READ_FROM_MEMORY: {
                    AtomicLong offset = this.offsetTable.get(mq);
                    if (offset != null) {
                        return offset.get();
                    } else if (ReadOffsetType.READ_FROM_MEMORY == type) {
                        return -1;
                    }
                }
                case READ_FROM_STORE: {
                    try {
                        long brokerOffset = this.fetchConsumeOffsetFromBroker(mq);
                        AtomicLong offset = new AtomicLong(brokerOffset);
                        this.updateOffset(mq, offset.get(), false);
                        return brokerOffset;
                    }
                    // No offset in broker
                    catch (MQBrokerException e) {
                        return -1;
                    }
                    //Other exceptions
                    catch (Exception e) {
                        log.warn("fetchConsumeOffsetFromBroker exception, " + mq, e);
                        return -2;
                    }
                }
                default:
                    break;
            }
        }

        return -1;
    }
}
```

读取Offset的值有三种ReadOffsetType的模式：优先从本地再从Broker机、从本地、从Broker机。考虑到一个Group中只有一个消费者消费同一个**Queue**(分区)下的消息，所以本机维护一份Offset是可行的，在每次拉取到结果后对Offset进行本地和远程更新：

```java
// DefaultMQPushConsumerImpl存在代码：
PullCallback pullCallback = new PullCallback() {
    @Override
    public void onSuccess(PullResult pullResult) {
		if (pullResult != null) {
            // ....忽略代码...
            //长轮训有结果返回，拉取消息结果处理
            pullResult = DefaultMQPushConsumerImpl.this.pullAPIWrapper.processPullResult(pullRequest.getMessageQueue(), pullResult,subscriptionData);

            switch (pullResult.getPullStatus()) {
               case FOUND:
                    long prevRequestOffset = pullRequest.getNextOffset();
                    pullRequest.setNextOffset(pullResult.getNextBeginOffset());
                    // ....忽略代码...
                    // 异步线程请求消费消息
                    boolean dispatchToConsume = processQueue.putMessage(pullResult.getMsgFoundList());
DefaultMQPushConsumerImpl.this.consumeMessageService.submitConsumeRequest(
                                    pullResult.getMsgFoundList(),
                                    processQueue,
                                    pullRequest.getMessageQueue(),
                                    dispatchToConsume);
            }
            // ....忽略代码...
        }
    }
    
}
    
```

继续看DefaultMQPushConsumerImpl.this.consumeMessageService.submitConsumeRequest的消费过程：

```java
public class ConsumeMessageConcurrentlyService implements ConsumeMessageService {
    @Override
    public void submitConsumeRequest(
        final List<MessageExt> msgs,
        final ProcessQueue processQueue,
        final MessageQueue messageQueue,
        final boolean dispatchToConsume) {
            // ....忽略代码...
        	ConsumeRequest consumeRequest = new ConsumeRequest(msgThis, processQueue, messageQueue);
        	try {
                    this.consumeExecutor.submit(consumeRequest);
                }
        	// ....忽略代码...
            
        }
    
    // 消费业务流程收尾
    public void processConsumeResult(
        final ConsumeConcurrentlyStatus status,
        final ConsumeConcurrentlyContext context,
        final ConsumeRequest consumeRequest
    ) {
        long offset = consumeRequest.getProcessQueue().removeMessage(consumeRequest.getMsgs());
        if (offset >= 0 && !consumeRequest.getProcessQueue().isDropped()) {
            this.defaultMQPushConsumerImpl.getOffsetStore().updateOffset(consumeRequest.getMessageQueue(), offset, true);
        }
}
}
```

ConsumeRequest实现Runnable,存在代码：

```java
try{
    //消息回调业务方，Listener为业务方注入
    status = listener.consumeMessage(Collections.unmodifiableList(msgs), context);
}catch(Throwable e){
}
// ....忽略代码...
if (!processQueue.isDropped()) {
    ConsumeMessageConcurrentlyService.this.processConsumeResult(status, context, this);
}
```

可以看出，最后还是调用调用了ConsumeMessageConcurrentlyService的processConsumeResult方法进行流程收尾，将当前的offset值更新到内存中：

```java
@Override
public void updateOffset(MessageQueue mq, long offset, boolean increaseOnly) {
    if (mq != null) {
        AtomicLong offsetOld = this.offsetTable.get(mq);
        if (null == offsetOld) {
            offsetOld = this.offsetTable.putIfAbsent(mq, new AtomicLong(offset));
        }

        if (null != offsetOld) {
            if (increaseOnly) {
                MixAll.compareAndIncreaseOnly(offsetOld, offset);
            } else {
                offsetOld.set(offset);
            }
        }
    }
}
```



#### persist持久化待远程

- 单个MessageQueue的Offset持久化：当发生Rebalance或者非法消息时；
- persistAll持久化列表：服务停机时和MQClientInstance启动后10s调用一次



总结：

&emsp;&emsp;通过上面分析，其实也可以看出，只靠Offset是无法保证消息只被严格的消费一次的，若Offset内存值存在丢失情况，则消息会被多次消费。

