---
layout: post
title: "其他-防采坑&采坑记"
subtitle: '可能踩的坑与踩过的坑'
author: "Kang"
date: 2019-09-11 11:50:12
header-img: "img/post-head-img/swimmer-1678307_1280.jpg"
catalog: true
tags:
  - 其它
---
## 1. 事务
&emsp;&emsp;在做事务时，需要考虑尽量控制一个事务的大小，特别是对外部系统的操作，除非接口或者本身也是事务操作，否则将该操作放在事务外面。这样可以避免在高并发下外部调用时间过长本地事务一直不释放而导致连接池资源耗尽问题。

## 2.异常的捕获
&emsp;&emsp;对于涉及到底层物理盘的写操作(如写文件)，目前碰见过两次底层异常但是业务方感知不到直接返回成功的情况，所以对于最底层的操作，推荐catch捕获Throwable异常，而不是Exception。
