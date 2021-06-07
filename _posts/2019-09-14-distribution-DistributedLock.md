---
layout: post
title: "分布式-分布式锁"
subtitle: '分布式锁简单总结'
author: "Kang"
date: 2019-09-14 17:45:54
header-img: "img/post-head-img/woman-water-1280.jpg"
catalog: true
tags:
  - 分布式
  - 锁
---
### 按锁的用途特点分类   
1. 允许多个客户端操作共享资源    
&emsp;&emsp;这种情况下，(相同业务操作)对共享资源的操作**一定是幂等性操作**，无论你操作多少次都不会出现不同结果。在这里使用锁，只是为了最大程度上避免对共享资源的重复性操作，但其**允许出现重复操作**。   
2. 只允许一个客户端操作共享资源    
&emsp;&emsp;这种情况下，(互斥业务操作)对共享资源的操作**一般是非幂等性操作**。在这种情况下，如果出现多个客户端操作共享资源，就可能意味着数据不一致，数据丢失，需要全局强一致性锁。   

### 为啥难
&emsp;&emsp;分布式情况下之所以问题变得复杂，主要就是需要考虑到网络的延时和服务不可靠。  

### 分布式锁基本要求
1. 排他性：同一时间只会有一个客户端能获取到锁，其它客户端无法同时获取； -- 基础要求
2. 避免死锁：这把锁在一段有限的时间之后，一定会被释放（正常释放或异常释放）； -- 基础要求
3. 高可用：获取或释放锁的机制必须高可用且性能佳(单点)。
4. 重入性、公平性、

## 常见集中分布式锁
### 1. 数据库
#### mysql性能  
- 机械硬盘：每秒1W左右    
- SSD硬盘：每秒2W左右   
注意：单库情况下多系统连接将连接资源耗尽问题；     
#### 需要解决的问题    
1. 分布式锁：通过插入语句的方式，谁插入谁获得锁。
2. 单点问题：可以使用主从备份解决
3. 阻塞问题：锁的获取没有队列可进行排队 -- 使用while死循环。
4. 重入问题：可以在表中增加字段记录当前获得锁的机器的主机信息和线程信息。  
5. 失效时间：一旦解锁操作失败，就会导致锁记录一直在数据库中，其他线程无法再获得到锁。-- 不断扫描超时锁
6. 资源占用：若排他锁长时间不提交，就会占用数据库连接。一旦类似的连接变得多了，就可能把数据库连接池撑爆。
7. 锁获取：是否走行级锁依赖于数据库对执行计划的判断，一旦数据库认为全表扫描(小表)效率更高时，则会走表锁，那么这种情况下所有锁进行竞争。    

优点：简单易懂  
缺点：各种各样的问题    

### 2. redis-基本实现方式
#### 实现方式
&emsp;&emsp;Redis命令：
```shell
SET key my_random_value  [EX seconds]|[PX milliseconds] [NX|XX]
```
语句说明：
- key为方法名，
- my_random_value为客户端随机值，
- EX/PX为不同的过期时间单位，
- NX ：只在键不存在时，才对键进行设置操作。
- XX ：只在键已经存在时，才对键进行设置操作。  
为了避免自己的锁被其他客户端释放，使用Lua脚本来做释放：
```shell
if redis.call("get",KEYS[1]) == ARGV[1] then
    return redis.call("del",KEYS[1])
else
    return 0
end
```
先比较身份标记"my_random_value"，若验证通过，则可以进行释放。     

#### 不可避免的问题：
&emsp;&emsp;两种实现方式中，不管怎么处理，都不可避免获得锁的客户端未处理完(网络延时/执行过长/fullGC)时锁被其他客户端获取。同时，宕机后的失效转移也不能避免在master宕机后，若slave来不及同步数据就被选为master，则数据丢失。
![分布式锁-redis问题示意图](https://raw.githubusercontent.com/kangzhihu/images/master/%E5%88%86%E5%B8%83%E5%BC%8F%E9%94%81-redis%E5%87%BA%E7%8E%B0%E9%97%AE%E9%A2%98%E7%A4%BA%E6%84%8F%E5%9B%BE.png)

### 3. redis-RedLock
#### 基本思路
&emsp;&emsp;客户端在事件点T时向N个(如5个)完全独立的redis服务端发送lock请求，设置请求锁超时时长为T1，设置自动释放时长为T2，则在T3时刻获取到锁。当在T1时间内获取到指定个锁(3个)时，那么表示当前应用获取到锁。在当前业务处理完成或者获取锁失败，客户端会依次删除所有的锁。  
&emsp;&emsp;为了尽可能的在网络较差时快失败，则需要将T1设置为一个较小的合理值。

#### 优化了什么   
&emsp;&emsp;RedLock能解决可用性(单点)和锁的失效转移问题。 
    
#### 仍存在的问题
&emsp;&emsp;为了保证在业务处理时锁不过期以及在释放时由于网络问题导致主动释放锁失败，需要对失效时间进行合理控制，这也是ReadLock的一个难点。(主要难点)      
&emsp;&emsp;当客户端甲在在5个节点中A、B、C三个中获取了锁，但是节点C锁数据未持久化时发生宕机，在C重启后客户端乙在C、E、F获取到了三个节点，这样出现了重锁。  

### 4. redis-Redisson
&emsp;&emsp;redisson是 redis 官方的分布式锁组件。其对ReadLock的失效时间做了一个优化：每获得一个锁时，设置超时时间，同时起一个后台线程定时(默认[超时时间/3])去刷新锁的超时时间。在释放锁的同时结束这个线程。


### 5. Zookeeper
&emsp;&emsp;Zookeeper的一致性协议可以很好的保证集群的高可用性，可以将整个集群看做一个稳定的单点服务。
##### 原理
&emsp;&emsp;上锁改为创建临时有序节点，每个上锁的节点均能创建节点成功，只是其序号不同。只有序号最小的可以拥有锁，如果这个节点序号不是最小的则 watch 序号比本身小的前一个节点 (公平锁)。

##### 优点
&emsp;&emsp;有效的解决单点问题，不可重入问题，非阻塞问题以及锁无法释放的问题。实现起来较为简单。一旦客户端和Zookeeper断开连接，临时有序节点将自动被删除(也是问题所在)。

##### 缺点
&emsp;&emsp;ZK 中创建和删除节点只能通过 Leader 服务器来执行，每次在创建锁和释放锁的过程中，都要动态创建、销毁临时节点来实现锁功能，所以性能上可能并没有缓存服务那么高(也依赖于集群模式做到容错等)。   
&emsp;&emsp;俗话说凡是都存在两面性，zookeeper的session断开可以防止锁无法快速释放问题，也同样是这个特点，导致客户端A拿到锁但是session由于网络波动等意外断开时，客户端B也能拿到锁。--对于这种由于网络波动导致的意外中断，zk有重试机制，一旦zk集群检测不到客户端的心跳，就会重试，多次重试之后还不行的话才会删除临时节点。所以在使用锁时需要有考虑好重试策略。         

![分布式锁-zookeeper问题示意图](https://raw.githubusercontent.com/kangzhihu/images/master/%E5%88%86%E5%B8%83%E5%BC%8F%E9%94%81-zookeeper%E9%97%AE%E9%A2%98%E7%A4%BA%E6%84%8F%E5%9B%BE.png)


### 总结
&emsp;&emsp;分布式锁绕不过的问题：产生并发
- 客户端fullGC导致在锁已超时后还能运行;
- 不管是redis失效时间导致还是zookeeper出现session断开,导致锁自动释放，被其他客户端获取到锁。