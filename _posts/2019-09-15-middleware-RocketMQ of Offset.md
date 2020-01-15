---
layout: post
title: "中间件-RocketMQ"
subtitle: '中间件RocketMQ之Offset'
author: "Kang"
date: 2020-01-15 13:26:04
header-img: "img/post-head-img/gingko-tree.jpg"
catalog: true
tags:
  - 中间件
  - MQ
  - RocketMQ
---
### 两种消费模式
- Pull模式：由消费者客户端主动向消息中间件（MQ消息服务器代理）拉取消息；(**消费端消费慢问题**)
- Push模式：由消息中间件（MQ消息服务器代理）主动地将消息推送给消费者；(**消息延迟与忙等待问题**)

RocketMQ的消费方式：是基于拉模式拉取消息的，在这其中有一种长轮询机制（对普通轮询的一种优化），来平衡上面Push/Pull模型的各自缺点。基本设计思路是：消费者如果第一次尝试Pull消息失败（比如：Broker端没有可以消费的消息），并不立即给消费者客户端返回Response的响应，而是先hold住并且挂起请求（将请求保存至pullRequestTable本地缓存变量中），然后Broker端的后台独立线程—PullRequestHoldService会从pullRequestTable本地缓存变量中不断地去取，具体的做法是查询待拉取消息的偏移量是否小于消费队列最大偏移量，如果条件成立则说明有新消息达到Broker端（这里，在RocketMQ的Broker端会有一个后台独立线程—ReputMessageService不停地构建ConsumeQueue/IndexFile数据，同时取出hold住的请求并进行二次处理），则通过重新调用一次业务处理器—PullMessageProcessor的处理请求方法—processRequest()来重新尝试拉取消息（此处，每隔5S重试一次，默认长轮询整体的时间设置为30s）。



### Offset

#### read读取

&emsp;&emsp;我们重点看下Pull模式下的Offset。DefaultMQPullConsumer为消费端代码，可以看到其中存在大致逻辑：

```java
public class DefaultMQPullConsumerImpl implements MQConsumerInner {
    
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