---
layout: post
title: "多线程-AQS简单总结"
subtitle: 'AQS与锁'
author: "Kang"
date: 2019-09-10 23:58:40
header-img: "img/post-head-img/car-49278_1280.jpg"
catalog: true
tags:
  - 多线程
  - 锁
---
考虑了三分钟，最终还是觉得两者放在一起好，驱动自己一次性总结完也方便后面观看。    
## 基础知识
### 公平锁与非公平锁
&emsp;&emsp;公平锁：实现机理在于每次有线程来抢占锁的时候，都会检查一遍有没有等待队列，如果有，当前线程会进入队列。    
&emsp;&emsp;非公平锁：与公平锁的区别在于新晋获取锁的进程会有多次机会(进入时/排队前)去抢占锁。如果被加入了等待队列后则跟公平锁没有区别。 

### 可重入锁
&emsp;&emsp;如果当前线程已经获得了锁， 那该线程下的所有方法都可以获得这个锁。

### CAS问题
&emsp;&emsp;循环时间长不成功时开销较大，ABA问题没法避免。通过使用自旋次数避免对CPU带来较大开销。    


## CLH锁
&emsp;&emsp;简单的一句话说说CLH锁是什么：在访问共享资源时，多请求节点在使用锁时会存在频繁地缓存同步操作(锁资源状态)，导致繁重的系统总线和内存的流量。为了解决这个问题，则以队列的形式去使用锁，并且后置节点将死循环监控前置节点的一个状态值(active,AQS中为state)，当状态为true时，则表示前置节点在等待锁或者使用锁，当为false表示前置节点释放了锁。     
&emsp;&emsp;CLH也是一种基于单向链表(隐式创建)的高性能、公平的自旋锁，申请加锁的线程只需要在其前驱节点的本地变量上自旋，从而极大地减少了不必要的处理器缓存同步的次数，降低了总线和内存的开销。

&emsp;&emsp;CLH锁主要有一个QNode类，QNode类内部维护了一个boolean类型的变量，每个线程拥有一个前驱节点（myPred）和当前自己的节点（myNode），还有一个tail节点用于存储最后一个获取锁的线程的状态。CLH的从逻辑上形成一个锁等待队列从而实现加锁，CLH锁只支持按顺序加锁和解锁，不支持重入，不支持中断。  

网上获取的一个最简单的CLH队列示意图如下：
![CLH示意图](https://raw.githubusercontent.com/kangzhihu/images/master/%E5%A4%9A%E7%BA%BF%E7%A8%8B-CLH%E7%A4%BA%E6%84%8F%E5%9B%BE.jpg)


## AQS
&emsp;&emsp;AQS中使用了类似CLH的队列，但是后继节点不会监控前驱节点的状态，而是当前驱节点在等待时，新增的后继节点本身也park进入waiting状态。当前驱节点释放锁时，unpark激活后继节点。(锁释放激活驱动)  

### AQS核心三板斧
- 自旋
- CAS
- LockSupport
```java
public class AQSLock {

    private volatile     int                            state        = 0; //并发资源
    private static final int                            expLockState = 1; //加锁值
    private              ThreadLocal<AQSNode>           threadLocal  = new ThreadLocal<>();
    private              ConcurrentLinkedQueue<AQSNode> queue        = new ConcurrentLinkedQueue();
    private              Thread                         lockHolder   = null;
    /**
     * 原子更新器，以原子的方式更新被volatile修饰的字段
     */
    private static final AtomicReferenceFieldUpdater    UPDATER      = AtomicReferenceFieldUpdater
            .newUpdater(
                    AQSLock.class, // 变量所在的类
                    AQSLock.class, // 变量的类型
                    "state");// 变量名

    public void lock() {
        AQSNode node = threadLocal.get();
        if (node == null) {
            Thread thread = Thread.currentThread();
            node = new AQSNode(thread, 0);
            threadLocal.set(node);
        }
        if (node.getState() == 0) {
            for (; ; ) {   // 三板斧1：自旋
                if (UPDATER.compareAndSet(node, state, expLockState)) { // 三板斧2：CAS
                    lockHolder = node.getThread();
                    break;
                }
                //将当前节点放在队尾
                queue.add(node);
                //阻塞当前操作的线程
                LockSupport.park();  // 三板斧3：LockSupport
                //设置当前节点的状态等其他操作
            }
        }

    }

    public void unLock() {
        AQSNode node = threadLocal.get();
        Thread thread = node.getThread();
        if (state == expLockState && thread == lockHolder) {
            for(;;){
                if(UPDATER.compareAndSet(this,expLockState,0)){
                    if(queue.size() > 0 ){
                        //移除并获取队头的节点
                        AQSNode headerNode = queue.remove();
                        //唤醒头节点线程
                        LockSupport.unpark(headerNode.getThread());
                        //更新当前节点的状态等其他操作
                    }
                    break;
                }
            }
        }
    }

    private class AQSNode {

        private          Thread thread;
        private volatile int    waitStatus;

        public AQSNode(Thread thread, int waitStatus) {
            this.thread = thread;
            this.waitStatus = waitStatus;
        }

        public Thread getThread() {
            return thread;
        }

        public int getWaitStatus() {
            return waitStatus;
        }
    }
}
```

### AQS整体简单示意图
![整体示意图](https://raw.githubusercontent.com/kangzhihu/images/master/AQS%20lock%26UnLock%E6%95%B4%E4%BD%93%E7%A4%BA%E6%84%8F%E5%9B%BE.png)

#### lock获取锁
- 1、tryAcquire尝试获取锁，若未获取到，则
- 2、addWaiter(Node.EXCLUSIVE) 创建并将当前节点接入到队列中，其中新创建节点的nextWaiter = 传入Node.EXCLUSIVE
- 3、acquireQueued 开始进入等待状态，过程如下：  
	+ 如果当前节点的前驱节点为头节点，尝试获取锁，未获取到则更新前驱节点的状态并进入阻塞状态：  
		 - shouldParkAfterFailedAcquire方法查看前驱结点pred.waitStatus，根据前驱节点的不同状态做不同动作：
			static final int CANCELLED =  1;   前驱节点已取消，这种状态下需循环遍历前驱，直到找到一个非取消节点 -- 最后返回false

			static final int SIGNAL    = -1;   前驱节点在取消或者释放的时候需要释放其后继节点，则直接返回false，当前节点将尝试parkAndCheckInterrupt -- 返回true

			static final int CONDITION = -2;   前驱节点处于等待队列中 -- 方法直接返回false
			static final int PROPAGATE = -3;   前驱节点共享模式下的释放，此时将前驱节点状态处理为Node.SIGNAL -- 返回false

		 - 若上面返回的true，则使用parkAndCheckInterrupt进行阻塞    
		     ⟹ 1. LockSupport.park  将当前线程挂起进入waiting等待状态  
		     ⟹ 2. Thread.interrupted  一旦被唤醒(等待unpark()或interrupt()唤醒自己，上面park结束),则返回中断标识(在阻塞中哪怕发生过一次中断)并清除当前线程的中断标记位。  

	+ 如果进入过阻塞，在被唤醒后acquireQueued返回interrupted = true，否则interrupted = false

	+ 如果没有进入过阻塞，则finally中使用cancelAcquire对当前节点进行出队操作，并修改当前节点的状态

- 4、如果acquireQueued也返回true(阻塞中发生过中断），则selfInterrupt中使用Thread.currentThread().interrupt()中断当前线程
   > 为啥在这个地方加上selfInterrupt()？   
     &emsp;&emsp;那是应为如果线程在等待过程中被中断过，阻塞中它是不响应的。只是获取资源后才再进行自我中断selfInterrupt()，将中断补上。

#### unlock释放锁
- 1、tryRelease()判断state&是否为锁拥有者来尝试释放锁，若ok则返回true；
- 2、若尝试释放成功，则进入unparkSuccessor
  + 先检查头节点状态，若后续节点不存在或waitStatus>0(已取消)，则进入补偿：从队列尾部向前遍历到头结点，直到找到一个未取消的节点。
  + 若头节点未取消，则直接使用LockSupport.unpark(s.thread)激活找到的后继节点。
