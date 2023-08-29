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
&emsp;&emsp;ForwardingNode标识当前hash桶节点已经完全迁移成功   
##### 扩容过程分析
1、线程执行put操作，发现容量已经达到扩容阈值，需要进行扩容操作，此时transferindex=tab.length=32；     
2、扩容线程A 以CAS的方式修改transferindex=31-16(MIN_TRANSFER_STRIDE迁移步长)=16[数组上hash桶的迁移工作已经进行(分配)到哪个位置的标识] ,然后按照降序迁移table[31]至table[16]这个区间的hash桶，一个线程处理一批hash桶下的链表或者红黑树     
3、迁移hash桶时，会将桶内的链表或者红黑树，按照一定算法，拆分成2份，将其插入nextTable[i]和nextTable[i+n]（n是table数组的长度）。 迁移完毕的hash桶,会被设置成ForwardingNode节点，以此告知访问此桶的其他线程，此节点已经迁移完毕。     
4、此时线程2访问到了ForwardingNode节点，如果线程2执行的put或remove等写操作，那么就会先帮其扩容。如果线程2执行的是get等读方法，则会调用ForwardingNode的find方法，去nextTable里面查找相关元素。     
ps:在每个hash桶节点进行迁移之前，都会使用同步锁将自己锁住(使用synchronized锁住链表/红黑树的头元素，可以认为是一把迁移锁)，而put操作的时候也需要获得该锁。
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

##### size计算
- JDK1.7 和 JDK1.8 对 size 的计算是不一样的。 1.7 中是先不加锁计算三次，如果三次结果不一样在加锁。
- JDK1.8 size 是通过对 baseCount 和 CounterCell数组进行CAS计算，最终通过 baseCount 和 遍历 CounterCell 数组得出 size。

##### 为什么ConcurrentHashMap中key不允许为null
&emsp;&emsp;简单来说，就是为了避免在多线程环境下出现歧义问题，在并发的情况下null可能会带来难以容忍的二义性。
- HashMap的设计是给单线程使用的，所以如果取到 null（空） 值，我们可以通过HashMap的 containsKey(key)方法来区分这个 null（空） 值到底是插入值是 null（空），还是本就没有才返回的 null（空） 值。
- 在并发条件下，当一个线程从ConcurrentHashMap获取某个key，如果返回的结果是null的时候，这个线程无法通过containsKey(key)方确认，因为当确实不存在时，此时在判断可能有其它线程新写入值。


