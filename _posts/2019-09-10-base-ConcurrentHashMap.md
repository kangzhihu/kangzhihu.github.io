---
layout: post
title: "基础-ConcurrentHashMap"
subtitle: 'ConcurrentHashMap简单总结'
author: "Kang"
date: 2019-09-10 18:30:47
header-img: "img/post-head-img/jellyfish-257860_960_720.jpg"
catalog: true
tags:
  - Java基础
---
## 特点
&emsp;&emsp;jdk7:数组+链式+分段锁结构，在扩容时直接一把梭,添加全局锁。最大并发个数就是Segment的个数。     
&emsp;&emsp;jdk8：数组+链式+红黑树+无锁结构，采用渐进式扩容。JDK8通过Unsafe类的CAS自旋赋值+synchronized同步+LockSupport阻塞等手段实现的高效并发，JDK8的锁粒度更细，理想情况下talbe数组元素的大小就是其支持并发的最大个数。

## 打标扩容
[参考阅读](https://blog.csdn.net/ZOKEKAI/article/details/90051567)
&emsp;&emsp;jdk8的ForwardingNode节点类：对于在扩容步长范围内的节点，在扩容时，将旧节点加锁，并将旧节点移动到新数组中，移动<font color="blue">完成后将旧节点位置存放一个ForwardingNode节点</font>，表示当前节点已经做扩容处理，其保留了新数组中nextTable的引用，若取值需要转新表结构去查找。 

#### 哪些会参与扩容
```
final V putVal(K key, V value, boolean onlyIfAbsent) {
  ...
  else if ((fh = f.hash) == MOVED)
               tab = helpTransfer(tab, f);
       else{
         ...
       }
}
```
&emsp;&emsp;从上面可以看出，当前线程获取的数组Node节点，若为扩容状态，则参与进来。也即受到影响的线程会参与进来。
#### 扩容时的读写

##### 读
&emsp;&emsp;对于未完成的节点(不为ForwardingNode)，正常读取数据。对于已经完成的节点，ForwardingNode方法重写了find方法，会转调新表访问已经迁移到了nextTable中的数据。  

##### 写/删
&emsp;&emsp;如检查到当前节点已完成扩容(Node#hash = MOVED)，则参与到扩容中来。如果扩容没有完成，当前链表的头节点会被锁住，所以写线程会被阻塞，直到扩容完成。 
