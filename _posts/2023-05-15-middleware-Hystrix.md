---
layout: post
title: "中间件-Hystrix"
subtitle: '中间件Hystrix简单总结'
author: "Kang"
date: 2023-05-15 13:26:04
header-img: "img/post-head-img/gingko-tree.jpg"
catalog: true
tags:
  - 中间件
  - 降级
  - Hystrix
---
### 做什么的
- Hystrix 通过将依赖服务进行资源隔离，进而阻止某个依赖服务出现故障时在整个系统所有的依赖服务调用中进行蔓延；
- 作用于服务之间，加入一些调用延迟或者依赖故障的容错机制，提升分布式系统的可用性和稳定性。

### Hystrix 的设计原则
- 对依赖服务调用时出现的调用延迟和调用失败进行控制和容错保护。
- 在复杂的分布式系统中，阻止某一个依赖服务的故障在整个系统中蔓延。比如某一个服务故障了，导致其它服务也跟着故障。
- 提供 fail-fast（快速失败）和快速恢复的支持。
- 提供 fallback 优雅降级的支持。
- 支持近实时的监控、报警以及运维操作。
- 阻止任何一个依赖服务耗尽所有的资源，比如 tomcat 中的所有线程资源。

&emsp;&emsp;Hystrix 里面核心的一项功能，其实就是所谓的资源隔离，要解决的最最核心的问题，就是将多个依赖服务的调用分别隔离到各自的资源池内。避免说对某一个依赖服务的调用，因为依赖服务的接口调用的延迟或者失败，导致服务所有的线程资源全部耗费在这个服务的接口调用上。一旦说某个服务的线程资源全部耗尽的话，就可能导致服务崩溃，甚至说这种故障会不断蔓延。  

### 隔离技术
![线程池与信号量示例](https://raw.githubusercontent.com/kangzhihu/images/master/hystrix-semphore-thread-pool.png)

##### 线程池隔离技术
&emsp;&emsp;线程池策略里接收请求的线程（Tomcat 线程）和实际处理业务的线程（另起一个新的线程）是分开的。当业务线程池不够时，会触发Tomcat的fallback，迅速将结果返回给客户端，这样Tomcat本身的线程也会被快速释放，而不会一直被占用。
>默认的策略就是线程池。最大的好处就是对于网络访问请求，如果有超时的话，可以避免调用线程阻塞住。线程池的大小，默认是 10,一般也够了

##### 信号量技术
&emsp;&emsp;信号量隔离技术，是直接让 tomcat 线程去调用依赖服务,信号量有多少，就允许多少个 tomcat 线程通过它，然后去执行。服务使用主线程(tomcat线程)进行同步调用，即没有业务线程池。
>一般用信号量常见于那种基于纯内存的一些业务逻辑服务，通常是针对超大并发量的场景下，而不涉及到任何网络访问请求。默认是 10，尽量设置的小一些，因为一旦设置的太大，而且有延时发生，可能瞬间导致 tomcat 本身的线程资源被占满。

### 流程
![使用示例](https://raw.githubusercontent.com/kangzhihu/images/master/hystrix-process.png)
##### 断路器
&emsp;&emsp;Hystrix 在运行过程中会向 commandkey 对应的熔断器报告成功、失败、超时和拒绝的状态，熔断器维护并统计这些数据，并根据这些统计数据来决定熔断开关是否打开。  
&emsp;&emsp;一次请求中会检查这个 command 对应的依赖服务是否开启了断路器。如果断路器被打开了，那么 Hystrix 就不会执行这个 command，而是直接去执行 fallback 降级机制，返回降级结果。  
&emsp;&emsp;如果断路器统计到的异常调用的占比超过了一定的阈值，比如说在 10s 内，经过断路器的流量达到了 30 个，同时其中异常访问的数量也达到了一定的比例，比如 60%（默认值50%) 的请求都是异常（报错 / 超时 / reject），就会开启断路。  
&emsp;&emsp;当一段时间后(默认5s)，断路器变为 half-open 状态，会尝试让部分请求通过，若请求成功，则断路器关闭。  

##### request cache
&emsp;&emsp;Hystrix会在每次请求中施加一个Request Context上下文，当一个请求上下文中，在一次请求上下文中，如果有多个 command，参数都是一样的，调用的接口也是一样的，而结果可以认为也是一样的，这个请求上下文后续的其它对这个依赖的调用全部从内存中取出缓存结果就可以了。
>HystrixRequestContextFilter打开拦截相关请求，并在Command 中重写 getCacheKey() 方法，即可定制化缓存，需要注意的是，在若一个上下文中发生更新操作，要删除/刷新缓存








