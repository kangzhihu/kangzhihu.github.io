---
layout: post
title: "其它-关于缓存更新的思考"
subtitle: '如何更新缓存'
author: "Kang"
date: 2019-09-16 15:59:00
header-img: "img/post-head-img/lake-4472491_1280.jpg"
catalog: true
tags:
  - 其它
  - 缓存
---
## 热点数据处理
#### 归档热点数据方案
&emsp;&emsp;本地设计LRU(Least recently used,最近最少使用)策略，这样本地不用提前去采用hash这样的策略去分散可能的热点数据，而本地能自动保持一直是缓存的热点。

#### 热点数据问题处理方案
&emsp;&emsp;本地缓存+远程缓存共同解决。在本地缓存一份数据(也可防雪崩)，该缓存数据存在物理失效时间、业务失效时间和提前更新时间。    
&emsp;&emsp;更新策略：每次查询时检查当前时间与业务失效时间之间的差值，若差值已经满足提前更新时间条件，则将更新(异步)请求进行队列等处理，这样能实现一个简单的本地更新削峰。为了控制并发更新，则可以使用guava的限流工具类RateLimter：  
```java
//5s一个请求
RateLimiter rateLimiter = RateLimiter.create(0.2);
//当前没获取到令牌，则直接返回
boolean res = rateLimiter.tryAcquire(10,TimeUnit.MILLISECONDS);
if(res){
   //远程获取更新缓存
}else{
   return;
}
```
## 并发更新导致更新到旧值问题
&emsp;&emsp;在数据更新时添加版本，更新的数据版本只能高于当前版本，这样可以有效避免并发情况下例如更新消息乱序而将已经更新的新增更新回旧值。
