---
layout: post
title: "基础-Synchronized"
subtitle: 'Synchronized概括与总结'
author: "Kang"
date: 2019-09-10 16:10:47
header-img: "img/post-head-img/kitten-1639027-1279x853.jpg"
catalog: true
tags:
  - Java基础
---
## 线程通知
1. 当线程执行wait()时，会把当前的锁释放，然后让出CPU，直接进入等待状态。
2. 当线程执行notify()/notifyAll()方法时，会唤醒一个处于等待状态该对象锁的线程，但是当前线程让然持有锁会继续往下执行，**直到执行完退出对象锁锁住的区域（synchronized修饰的代码块）后再释放锁**。

## 对象与锁结构图
![对象内存结构](https://raw.githubusercontent.com/kangzhihu/images/master/%E5%9F%BA%E7%A1%80-Java%E5%AF%B9%E8%B1%A1%E5%86%85%E5%AD%98%E7%BB%93%E6%9E%84.png)

## Synchronized锁膨胀过程
>为何会做不同级别的锁？   
&emsp;&emsp;通过锁在对象头中的结构可以看出，当处于偏向锁状态时，线程获取到锁后只需要修改对象头的标志位就行，但是当为重量级锁时，就需要获取锁对象的monitor监视器，这涉及到一个内核态和用户态的上下文切换过程。   

粗图，放大观看  
![锁膨胀过程](https://raw.githubusercontent.com/kangzhihu/images/master/%E5%9F%BA%E7%A1%80-synchronized%E9%94%81%E8%86%A8%E8%83%80%E8%BF%87%E7%A8%8B.png)  

### 关锁的概念

&emsp;&emsp;自旋锁(CAS)：让不满足条件的线程等待一会看能不能获得锁，通过占用处理器的时间来避免线程切换带来的开销。自旋等待的时间或次数是有一个限度的，如果自旋超过了定义的时间仍然没有获取到锁，则该线程应该被挂起。  

&emsp;&emsp;偏向锁：大多数情况下，锁总是由同一个线程多次获得(<font color="red">适用于一个线程反复进入同步代码块</font>)。当一个线程访问同步块并获取锁时，会在对象头和栈帧中的锁记录里存储锁偏向的线程ID，偏向锁是一个可重入的锁。如果锁对象头的Mark Word里存储着指向当前线程的偏向锁，无需重新进行CAS操作来加锁和解锁。当有其他线程尝试竞争偏向锁时，持有偏向锁的线程（不处于活动状态）才会释放锁。偏向锁无法使用自旋锁优化，因为一旦有其他线程申请锁，就破坏了偏向锁的假定进而升级为轻量级锁。   

&emsp;&emsp;轻量级锁：减少无实际竞争情况下，使用重量级锁产生的性能消耗(<font color="red">适用于线程交替执行</font>)。JVM会现在当前线程的栈桢中创建用于存储锁记录的空间，并将对象头中的Mark Word复制到锁记录中。然后线程会尝试使用CAS将对象头中的Mark Word替换为指向锁记录的指针，成功则当前线程获取到锁，失败则表示其他线程竞争锁当前线程则尝试使用自旋的方式获取锁。自旋获取锁失败则锁膨胀会升级为重量级锁。   

&emsp;&emsp;重量级锁：通过对象内部的监视器(monitor)实现(<font color="red">适用于高并发场景，此种业务下，推荐关闭锁的升级过程，减少升级过程的消耗</font>)，其中monitor的本质是依赖于底层操作系统的Mutex Lock实 现，操作系统实现线程之间的切换需要从用户态到内核态的切换，切换成本非常高。线程竞争不使用自旋，不会消耗CPU。但是线程会进入阻塞等待被其他线程被唤醒，响应时间缓慢。 -- 在启动JVM的时候加上-XX:-UseBiasedLocking参数来禁用偏向锁。      


[推荐阅读](https://blog.csdn.net/baidu_38083619/article/details/82527461)
