---
layout: post
title: "中间件-RocketMQ"
subtitle: '中间件RocketMQ之Producer'
author: "Kang"
date: 2020-01-17 12:26:04
header-img: "img/post-head-img/grass.jpg"
catalog: true
tags:
  - 中间件
  - MQ
  - RocketMQ
---
&emsp;&emsp;路由管理是RocketMQ的核心功能之一，涵盖了订阅管理，连接管理，负载均衡等一系列功能，代码分布在NameServer、Broker、Producer和Consumer等各组件实现中。RocketMQ有一个特色功能：支持灵活的集群工作方式。Broker、Producer和Consumer都能够在运行时动态扩容或缩容，这都要依赖于强大的路由管理模块。

### Producer如何投递消息

&emsp;&emsp;我们查看投递代码的入口(DefaultMQProducer或者其事务子类TransactionMQProducer)，我们以sendMessageInTransaction发送事务型消息为例，可以看到最终调用了DefaultMQProducerImpl类的sendDefaultImpl方法：

```java
private SendResult sendDefaultImpl(
    Message msg,
    final CommunicationMode communicationMode,
    final SendCallback sendCallback,
    final long timeout
) throws MQClientException, RemotingException, MQBrokerException, InterruptedException {
	//...省略代码...
    //先从本地存储的路由信息表获取topic的路由信息
    TopicPublishInfo topicPublishInfo = this.tryToFindTopicPublishInfo(msg.getTopic());
    //...省略代码...
    String lastBrokerName = null == mq ? null : mq.getBrokerName();
    //获取具体的路由信息
    MessageQueue mqSelected = this.selectOneMessageQueue(topicPublishInfo, lastBrokerName);
   //...省略代码...
   //
   sendResult = this.sendKernelImpl(msg, mq, communicationMode, sendCallback, topicPublishInfo, timeout - costTime);

    //...省略代码...

}

//获取本地路由表
private TopicPublishInfo tryToFindTopicPublishInfo(final String topic) {
    TopicPublishInfo topicPublishInfo = this.topicPublishInfoTable.get(topic);
    if (null == topicPublishInfo || !topicPublishInfo.ok()) {
        //若当前topic的本地路由表信息不存在，则构建对象，标识当前topic存在路由信息
        this.topicPublishInfoTable.putIfAbsent(topic, new TopicPublishInfo());
        //从NameServer中远程获取相关信息，并做本地路由表维护
        this.mQClientFactory.updateTopicRouteInfoFromNameServer(topic);
        topicPublishInfo = this.topicPublishInfoTable.get(topic);
    }
    if (topicPublishInfo.isHaveTopicRouterInfo() || topicPublishInfo.ok()) {
        return topicPublishInfo;
    } else {
        this.mQClientFactory.updateTopicRouteInfoFromNameServer(topic, true, this.defaultMQProducer);
        topicPublishInfo = this.topicPublishInfoTable.get(topic);
        return topicPublishInfo;
    }
}
```

由于对路由表的管理比较复杂，所以后面单独讲解，我们此处只需要知道其对topicPublishInfoTable等路由信息进行了维护更新。我们接着看selectOneMessageQueue，通过tryToFindTopicPublishInfo获取到当前topic的路由的发布信息对象TopicPublishInfo后，其最终会调用到TopicPublishInfo#selectOneMessageQueue：

```java
public class TopicPublishInfo {
    
    public MessageQueue selectOneMessageQueue() {
        int index = this.sendWhichQueue.getAndIncrement();
        int pos = Math.abs(index) % this.messageQueueList.size();
        if (pos < 0)
            pos = 0;
        return this.messageQueueList.get(pos);
    } 
}
```

可以看出，最终返回的MessageQueue对象即为具体的路由信息：

```java
public class MessageQueue implements Comparable<MessageQueue>, Serializable {
    private static final long serialVersionUID = 6191200464116433425L;
    private String topic;
    private String brokerName;
    private int queueId;
}
```

最后通过MessageQueue中的信息获取brokerAddr后，最终调用MQClientAPIImpl#sendMessage将消息发送出去。



### 路由表管理

&emsp;&emsp;接上面的路由表讲解。