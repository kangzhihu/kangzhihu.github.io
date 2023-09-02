---
layout: post
title: "中间件-RocketMQ"
subtitle: '中间件RocketMQ之Rebalance（二）'
author: "Kang"
date: 2020-01-16 10:26:04
header-img: "img/post-head-img/ravine-stream.jpg"
catalog: true
tags:
  - 中间件
  - MQ
  - RocketMQ
---
### Rebalence过程

&emsp;&emsp;RocketMQ的consumer负载均衡依赖于RebalanceImpl类。我们仍然以Pull模式进行讲解。在DefaultMQPullConsumerImpl中存在RebalanceImpl属性，关键方法：

```java
public abstract class RebalanceImpl {
    
    //Rebalance
    protected AllocateMessageQueueStrategy allocateMessageQueueStrategy;
    
    public void doRebalance(final boolean isOrder) {
        Map<String, SubscriptionData> subTable = this.getSubscriptionInner();
        if (subTable != null) {
            for (final Map.Entry<String, SubscriptionData> entry : subTable.entrySet()) {
                final String topic = entry.getKey();
                try {
                    this.rebalanceByTopic(topic, isOrder);
                } catch (Throwable e) {
                    if (!topic.startsWith(MixAll.RETRY_GROUP_TOPIC_PREFIX)) {
                        log.warn("rebalanceByTopic Exception", e);
                    }
                }
            }
        }
        this.truncateMessageQueueNotMyTopic();
    }
   
    private void rebalanceByTopic(final String topic, final boolean isOrder) {
        switch (messageModel) {
                //广播消费模式，消息会被每一个消费者消费，即使同一个ConsumerGroup下
            case BROADCASTING: {
                //获取当前topic的所有MessageQueue，也即消费分区(Partition)
                Set<MessageQueue> mqSet = this.topicSubscribeInfoTable.get(topic);
                if (mqSet != null) {
                    boolean changed = this.updateProcessQueueTableInRebalance(topic, mqSet, isOrder);
                    if (changed) {
                        this.messageQueueChanged(topic, mqSet, mqSet);
                        log.info("messageQueueChanged {} {} {} {}",
                            consumerGroup,
                            topic,
                            mqSet,
                            mqSet);
                    }
                } else {
                    log.warn("doRebalance, {}, but the topic[{}] not exist.", consumerGroup, topic);
                }
                break;
            }
            // 集群消费模式,同一个ConsumerGroup下只会被一个消费者消费
            case CLUSTERING: {
                //获取当前topic的所有MessageQueue，也即消费分区(Partition)
                Set<MessageQueue> mqSet = this.topicSubscribeInfoTable.get(topic);
                //获取当前topic-消费组 所有的消费者client id
                List<String> cidAll = this.mQClientFactory.findConsumerIdList(topic, consumerGroup);
                if (null == mqSet) {
                    if (!topic.startsWith(MixAll.RETRY_GROUP_TOPIC_PREFIX)) {
                        log.warn("doRebalance, {}, but the topic[{}] not exist.", consumerGroup, topic);
                    }
                }

                if (null == cidAll) {
                    log.warn("doRebalance, {} {}, get consumer id list failed", consumerGroup, topic);
                }

                if (mqSet != null && cidAll != null) {
                    List<MessageQueue> mqAll = new ArrayList<MessageQueue>();
                    mqAll.addAll(mqSet);
					//对partition和client id做排序
                    Collections.sort(mqAll);
                    Collections.sort(cidAll);

                    AllocateMessageQueueStrategy strategy = this.allocateMessageQueueStrategy;

                    List<MessageQueue> allocateResult = null;
                    try {
                        //然后调用strategy获取当前客户端需要消费的MessageQueue(Partition)列表
                        allocateResult = strategy.allocate(
                            this.consumerGroup,
                            this.mQClientFactory.getClientId(),
                            mqAll,
                            cidAll);
                    } catch (Throwable e) {
                        log.error("AllocateMessageQueueStrategy.allocate Exception. allocateMessageQueueStrategyName={}", strategy.getName(),
                            e);
                        return;
                    }

                    Set<MessageQueue> allocateResultSet = new HashSet<MessageQueue>();
                    if (allocateResult != null) {
                        allocateResultSet.addAll(allocateResult);
                    }
					//更新订阅关系
                    boolean changed = this.updateProcessQueueTableInRebalance(topic, allocateResultSet, isOrder);
                    if (changed) {
                        log.info(
                            "rebalanced result changed. allocateMessageQueueStrategyName={}, group={}, topic={}, clientId={}, mqAllSize={}, cidAllSize={}, rebalanceResultSize={}, rebalanceResultSet={}",
                            strategy.getName(), consumerGroup, topic, this.mQClientFactory.getClientId(), mqSet.size(), cidAll.size(),
                            allocateResultSet.size(), allocateResultSet);
                        //若发生变更，则回调变更监听器
                        this.messageQueueChanged(topic, mqSet, allocateResultSet);
                    }
                }
                break;
            }
            default:
                break;
        }
    }
}
```

我们主要看集群模式，也是RocketMQ的默认消费方式。很明显做了下面几步：

1. 获取topic下的MessageQueue，一个MessageQueue实际上就是一个partition；
2. 获取当前topic和group的client id，即当前group中消费此topic的客户端；
3. 随后对partition和client id做排序；
4. 然后调用strategy获取当前客户端需要消费的partition；
5. 更新订阅
6. 回调变更监听器(MessageQueueListener#messageQueueChanged)

&emsp;&emsp;负载均衡算法主要是看注入的属性allocateMessageQueueStrategy，默认使用的AllocateMessageQueueAveragely算法。



### 何时Rebalence

&emsp;&emsp;MQConsumerInner的子类在创建Topic时会直接创建一个mQClientFactory属性，这个属性就为MQClientInstance，同时子类中(DefaultMQPullConsumerImpl为例)start启动时也会进行初始化：

```java
public class DefaultMQPullConsumerImpl implements MQConsumerInner {
    private MQClientInstance mQClientFactory;
    
    public synchronized void start() throws MQClientException {
        switch (this.serviceState) {
            case CREATE_JUST:
                //...省略代码...
                //MQClientInstance实例化
                this.mQClientFactory = MQClientManager.getInstance().getAndCreateMQClientInstance(this.defaultMQPullConsumer, this.rpcHook);
				//...省略代码...
        }

    }
}
```

&emsp;&emsp;MQClientInstance实例化时存在实例化属性：

```java
this.rebalanceService = new RebalanceService(this);
```

继续看下去，RebalanceService实现了Runnable

```java
public class RebalanceService extends ServiceThread {
    private static long waitInterval =
        Long.parseLong(System.getProperty(
            "rocketmq.client.rebalance.waitInterval", "20000"));
    
    @Override
    public void run() {
        log.info(this.getServiceName() + " service started");

        while (!this.isStopped()) {
            //若当前线程开启，则每20秒进行一次Rebalence
            this.waitForRunning(waitInterval);
            this.mqClientFactory.doRebalance();
        }

        log.info(this.getServiceName() + " service end");
    }  
}
```

&emsp;&emsp;可以看到每实例化一个MQClientInstance实例，会启动一个对应的线程进行Rebalance。因此，当一个consumer出现宕机后，默认最多20s，其它机器将由于Rebalance会重新消费已宕机的机器消费的partition，同样当有新的consumer连接上后，20s内也会由于进行Rebalance使得新的consumer有机会消费partition。

### RocketMQ与Kafka Rebalance区别
&emsp;&emsp;Broker是通知每个消费者各自Rebalance，即每个消费者自己给自己重新分配队列，而不是Broker将分配好的结果告知Consumer。RocketMQ与Kafka Rebalance机制类似，二者Rebalance分配都是在客户端进行，不同的是：    
- Kafka：会在消费者组的多个消费者实例中，选出一个作为Group Leader，由这个Group Leader来进行分区分配，分配结果通过Cordinator(特殊角色的broker)同步给其他消费者。相当于Kafka的分区分配只有一个大脑，就是Group Leader。
- RocketMQ：每个消费者，自己负责给自己分配队列，相当于每个消费者都是一个大脑。
