---
layout: post
title: "分布式-一致性paxos算法简单整理"
subtitle: '一致性paxos算法简单整理'
author: "Kang"
date: 2019-09-12 18:01:10
header-img: "img/post-head-img/network-3396348_1280.jpg"
catalog: true
tags:
  - 分布式
---
## 推荐参阅列表
1. [推荐阅读-paxos生活化案例讲解](http://hedengcheng.com/?p=970)
2. [推荐阅读-架构师需要了解的Paxos原理、历程及实战](https://mp.weixin.qq.com/s?__biz=MzAwMDU1MTE1OQ==&mid=403582309&idx=1&sn=80c006f4e84a8af35dc8e9654f018ace&3rd=MzA3MDU4NTYzMw==&scene=6#rd)
3. [推荐阅读-图解分布式一致性协议Paxos](https://www.cnblogs.com/hugb/p/8955505.html)
4. [推荐阅读-Paxos协议超级详细解释+简单实例](https://blog.csdn.net/cnh294141800/article/details/53768464)
## 解决了什么
&emsp;&emsp;解决分布式环境下一致性的问题。

## 涉及到角色
1. Proposer：N个，提议发起者，将提案<proposal number, n>对外公布，发送自己的提议；  
2. Acceptor：N个，提议接受者(一般也是投票者Proposer)，Proposer提出的提议必须获得超过半数(N/2+1)的 Acceptor批准后才能通过。
3. Learner：1个，提议学习者，当一个提议通过后，Learner将通过的确定性取值同步给其他未确定的Acceptor。  

## 协议过程简单总结
&emsp;&emsp;一句话总结：proposer将发起提案（value）给所有accpetor，超过半数accpetor获得批准后，proposer将提案写入accpetor内，最终所有accpetor获得一致性的确定性取值，且后续不允许再修改。   
&emsp;&emsp;若不能收集到半数投票，这一轮参与者将一直等待；   
&emsp;&emsp;在节点角色中存在一个本地定时器，一个倒计时即为一个周期。对于选主Leader场景，如果周期内Leader未发送心跳消息，则Follower自动开启新一轮选主操作。

### 协议两大阶段

#### 第一阶段 Prepare
P1-a)：Proposer 发送 Prepare    
&emsp;&emsp;Proposer 生成全局且递增的提案 ID（Proposalid，以高位时间戳 + 低位机器 IP 可以保证性和递增性），向 Paxos 集群的所有机器发送 <font color='green'>PrepareRequest</font>，这里无需携带提案内容，只携带 Proposalid 即可。  

P1-b)：Acceptor 应答 Prepare    
&emsp;&emsp;Acceptor 收到每一个<font color='green'>PrepareRequest</font> 后，根据“两个承诺，一个应答”。    

##### 两个承诺   
&emsp;&emsp;承诺一，Acceptor不再应答Proposer发出的 Proposalid 小于等于（注意：这里是 <= ）当前请求的 <font color='green'>PrepareRequest</font>； -- --> 第一阶段的承诺,响应<font color='blue'>PrepareResponse</font>,貌似响应中也应为目前无接受值而只是包含Proposalid   
&emsp;&emsp;承诺二，Acceptor不再应答Proposer发出的 Proposalid 小于（注意：这里是 < ）当前请求的 <font color='green'>AcceptRequest</font>; -- --> 第二阶段的承诺,响应<font color='blue'>AcceptResponse </font>  

##### 一个应答   
&emsp;&emsp;若传入的提案的编号大于它已经回复的所有 prepare 消息，则Acceptor返回自己已经 Accept 过的提案中 ProposalID 较大的那个提案的内容(已批准过最大值)，如果没有则返回空值;   
&emsp;&emsp;应答当前请求前，也要按照“两个承诺”检查，且Acceptor应答前要在本地持久化当前响应的Propsalid。      

<font color="#6A5ACD">在Prepare阶段，Acceptor可以不停的根据上面的两个承诺进行应答。</font>

#### 第二阶段 Accept
<font color="#009ACD">当一个 Proposer 收到了多数 Acceptors 对 prepare 的回复后，就进入批准阶段。</font>   
P2-a)：Proposer 发送 Accept   
&emsp;&emsp;“提案生成规则”：Proposer 收集到第一阶段多数派应答的 PrepareResponse 后，从中选择proposalid最大的提案内容，作为要发起 Accept 的提案，如果这个提案为空值，则可以自己随意决定提案内容，否则携带在prepare决定的<Proposalid,value>，然后携带上当前 Proposalid，向 Paxos 集群的所有机器发送<font color='green'>AccpetRequest</font>。-- 此时要不就是初始阶段携带了自己值，要不就是携带了接受的提案编号以及对应的值   

P2-b)：Acceptor 应答 Accept  
&emsp;&emsp;Accpetor 收到 AccpetRequest 后，检查不违背自己之前作出的“两个承诺”情况下，持久化当前 Proposalid 和提案内容，若此时在半数中有新的协议版本>当前版本，则将返回新的<font color='blue'>AcceptResponse</font>。最后 Proposer 收集到多数派应答的 AcceptResponse 后，形成决议。

<font color="#6A5ACD">从整个过程上来看，Paxos协议最终由提议ID的大小决定。</font>

![分布式-paxos整体流程示意图](https://raw.githubusercontent.com/kangzhihu/images/master/%E5%88%86%E5%B8%83%E5%BC%8F-paxos-flow.png)

### 简单示例
引用[paxos原理分析简单图解](https://blog.csdn.net/zifanyou/article/details/84779615)示例
![分布式-paxos整体流程示意图](https://raw.githubusercontent.com/kangzhihu/images/master/%E5%88%86%E5%B8%83%E5%BC%8F-paxos%E7%A4%BA%E4%BE%8B%E8%AE%B2%E8%A7%A3.png)

#### 示例讲解
假设2个Proposor， 3个Acceptor。 初始编号 Proposor1=1， Proposor2=2 。

以下按时间顺序进行：
>
Prepare阶段  
1. Proposor1 把自己的编号（1）发送给所有Acceptor申请批准进行提案。
2. Proposor2 把自己的编号（2）发送给所有Acceptor申请批准进行提案。
3. Acceptor1，Acceptor2 先收到Proposor1的请求，由于之前没有接受过申请，所以都同意Proposor1。
4. Proposor1获得超过半数同意，转为提交阶段，准备提交。（此时还没向Acceptor3发送，或者已经发送还没有应答）。
5. Acceptor3接收到Proposor2的请求，由于之前没有接受过申请，所以同意Proposor2。
6. **Acceptor2接收到Proposor2的请求，判断编号2大于之前的1，所以同意Proposor2。**
7. Proposor2获得超过半数同意，准备提交。（此时还没向Acceptor1发送，或者已经发送还没有应答）。  
>
Accept阶段  
8. Proposor1向Acceptor1，Acceptor2提交(value=a）。
9. Acceptor1同意Proposor1，Acceptor2发现当前编号已经为2，所以拒绝Proposor1(无应答回复或者回应拒绝)。   
  - <font color='green'>若响应，因为此时Acceptor中本地还未存储通过的value和Proposorid，故返回内容为空</font>
10. Proposor2向Acceptor2，Acceptor3提交(value=b）。
11. Acceptor3同意Proposor2，Acceptor2也会同意Proposor2。
12. Proposor2提交成功。
13. Proposor1发现自己的提议未通过一半，增大编号，将原值通过编号（3）发送给所有Acceptor重新申请批准进行提案。
14. Acceptor2同意Proposor1，并且告知已经接受编号1（value=a）的提案。
11. Acceptor3同意Proposor1，并且告知已经接受编号2 (value=b) 的提案。
12. Proposor1选择上一步中编号大的值提交。所以，提交编号3,value=b;
13. 达成一致，Acceptor中都在本地维护了相同的值(value=b)。


### Question
1. 如果靠提议ID的大小决定，那岂不是谁的大谁就会被接受,可不可能不应该被接受的使用最大id。 
2. 想获得过半响应，则需要等待多久？
