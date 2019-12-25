---
layout: post
title: "多线程-BlockingQueue"
subtitle: 'BlockingQueue概括与总结'
author: "Kang"
date: 2019-09-11 10:31:09
header-img: "img/post-head-img/women-2808629_1280.jpg"
catalog: true
tags:
  - 多线程
---
## 简介

BlockingQueue即阻塞队列，它算是一种将ReentrantLock用得非常精彩的一种表现，其最常用的还是用于实现生产者与消费者模式，大致如下图所示：  
![BlockingQueue生产者消费者示意图](https://raw.githubusercontent.com/kangzhihu/images/master/BlockingQueue.png)    

## 源码简解
BlockingQueue内部有一个ReentrantLock，其生成了两个Condition，在ArrayBlockingQueue的属性声明中可以看见：   
```java
/** Main lock guarding all access */
final ReentrantLock lock;
/** Condition for waiting takes */
private final Condition notEmpty;
/** Condition for waiting puts */
private final Condition notFull;
 
public ArrayBlockingQueue(int capacity, boolean fair) {
    if (capacity <= 0)
        throw new IllegalArgumentException();
    this.items = new Object[capacity];
    lock = new ReentrantLock(fair);
    notEmpty = lock.newCondition();
    notFull =  lock.newCondition();
}
```
而如果能把notEmpty、notFull、put线程、take线程拟人的话，那么我想put与take操作可能会是下面这种流程：  
1. put动作：   
![put](https://raw.githubusercontent.com/kangzhihu/images/master/BlockingQueue.put.png) 

ArrayBlockingQueue.put(E e)源码如下：
```java
public void put(E e) throws InterruptedException {
    checkNotNull(e);
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        // 如果队列已满，则等待
        while (count == items.length)
            notFull.await(); //notFull的标志条件为false（至为await）
        insert(e);
    } finally {
        lock.unlock();
    }
}
private void insert(E x) {
    items[putIndex] = x;
    putIndex = inc(putIndex);
    ++count;
    notEmpty.signal(); //有新的元素被插入，通知等待中的取走元素线程(若有等待)
}

```
2. take动作：  
![take](https://raw.githubusercontent.com/kangzhihu/images/master/BlockingQueue.take.png)    

ArrayBlockingQueue.take()源码如下：
```java
public E take() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        // 如果队列为空，则等待
        while (count == 0)
            notEmpty.await(); //notEmpty的标志条件为false（至为await）
        return extract();
    } finally {
        lock.unlock();
    }
}
 
/**
 * Extracts element at current take position, advances, and signals.
 * Call only when holding lock.
 */
private E extract() {
    final Object[] items = this.items;
    E x = this.<E>cast(items[takeIndex]);
    items[takeIndex] = null;
    takeIndex = inc(takeIndex);
    --count;
    notFull.signal(); // 有新的元素被取走，通知等待中的插入元素线程
    return x;
}
```
&emsp;&emsp;可以看见，put(E)与take()是同步的，在put操作中，当队列满了，会阻塞put操作，直到队列中有空闲的位置。而在take操作中，当队列为空时，会阻塞take操作，直到队列中有新的元素。  
&emsp;&emsp;而这里使用两个Condition，则可以避免调用signal()时，会唤醒相同的put或take操作。
