---
layout: post
title: "多线程-线程池"
subtitle: '线程池概括与总结'
author: "Kang"
date: 2019-09-10 23:34:47
header-img: "img/post-head-img/woman-1539416_1280.jpg"
catalog: true
tags:
  - 多线程
---
## 基础知识
### 相关创建类
&emsp;&emsp;Executors、ThreadPoolExecutor、ExecutorService


### 核心参数
&emsp;&emsp;corePoolSize、maximumPoolSize、keepAliveTime、unit、workQueue等待队列、handler拒绝策略

#### 参数的使用
- 初始阶段：刚创建是，工作线程为0，此时当任务加入时，线程数不断增加，任务被直接执行；    
- 达到核心阶段：若当前线程数为corePoolSize，此时线程队列将起作用，任务先不断的先进入到阻塞队列中，等待有空闲线程去执行；    
- 队列满：当线程队列数量达到最大值时，则触发线程池中工作线程的继续创建，直到达到maxiumSize；    
- 达到maxiumSize：若达到maxiumSize时，仍然有任务进来，则将触发采取拒绝策略。    

### 线程池类别
&emsp;&emsp;newFixedThreadPool、newCachedThreadPool、newSingleThreadExecutor、newScheduledThreadPool    
&emsp;&emsp;这四种方式均不推荐使用，而直接使用ThreadPoolExecutor创建线程池，这样创建者可以更精确的掌控线程池的参数。  
&emsp;&emsp;ForkJoinPool，使用任务窃取算法，即自己工作完成了可以去忙碌的工作线程窃取任务执行。  

### Queue说明
- SynchronousQueue：直接提交，当有新的任务到来时，必须能够提交成功否则会抛出异常，所以要求maximumPoolSize无限大，可能存在执行任务量过多。  
- LinkedBlockingQueue：链表队列，这种方式下，执行线程数永远不会超过corePoolSize，可能排队过多。  
- ArrayBlockingQueue：数组队列，有助于防止资源耗尽，但是可能较难调整和控制。队列大小和最大池大小可能需要相互折衷。若设置不合理，可能造成资源浪费或者大量任务被丢弃。  


### 四种拒绝策略
- ThreadPoolExecutor.AbortPolicy:丢弃任务并抛出RejectedExecutionException异常。 
- ThreadPoolExecutor.DiscardPolicy：也是丢弃任务，但是不抛出异常。 
- ThreadPoolExecutor.DiscardOldestPolicy：丢弃队列最前面的任务，然后重新尝试执行任务（重复此过程）
- ThreadPoolExecutor.CallerRunsPolicy：由调用线程处理该任务 

## 线程池原理
```java
public void execute(Runnable command) {
     if (command == null)
         throw new NullPointerException();
     //如果线程数小于corePoolSize，则执行addWorker方法创建新的线程执行任务
     int c = ctl.get();
     if (workerCountOf(c) < corePoolSize) {
         if (addWorker(command, true))
             return;
         c = ctl.get();
     }
     //如果线程池处于RUNNING状态，且把提交的任务成功放入阻塞队列中(offer添加失败时返回false)
     if (isRunning(c) && workQueue.offer(command)) {
         int recheck = ctl.get();
         if (!isRunning(recheck) && remove(command))
             reject(command);
         else if (workerCountOf(recheck) == 0)
             addWorker(null, false);
     }
     else if (!addWorker(command, false))
         reject(command);
 }

```
&emsp;&emsp;我们看看addWorker()方法：其为将任务包装成一个工作节点，作为线程池中的一个基本工作单位。最主要的是理解其构造方法:

```java
Worker(Runnable firstTask) {
    setState(-1); // inhibit interrupts until runWorker
    this.firstTask = firstTask;
    this.thread = getThreadFactory().newThread(this);
}

public void run() {
    runWorker(this);
}
```
&emsp;&emsp;线程工厂在创建线程thread时，将Woker实例本身this作为参数传入，当执行start方法启动线程thread时，本质是执行了Worker的runWorker方法。  
所以本质上最后启动的是Worker的runWorker方法，下面为简化后核心处理：
```java
final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();
    Runnable task = w.firstTask;
    w.firstTask = null;
    w.unlock(); // allow interrupts
    boolean completedAbruptly = true;
    try {
       // task当前创建时传入了值，或者从等待队列中能取到值
        while (task != null || (task = getTask()) != null) {
            w.lock();
            ......
            try {
              beforeExecute(wt, task);
              task.run();
              afterExecute(task, thrown);
              ...
            } finally {
                task = null;
                w.completedTasks++;
                w.unlock();
            }
        }
        completedAbruptly = false;
    } finally {
        processWorkerExit(w, completedAbruptly);
    }
}
```
&emsp;&emsp;可以看出，整个流程也是比较清晰的，在while中不停循环处理当前传入值或者从getTask()中获取任务去执行。最后看看getTask()：  
```java
private Runnable getTask() {
    // 紫旋+队列取值方式的巧妙运用
    for (;;) {
        ...
        boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;
        try {
            Runnable r = timed ?
                workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                workQueue.take();
            if (r != null)
                return r;
            timedOut = true;
        } catch (InterruptedException retry) {
            timedOut = false;
        }
    }
}
```
&emsp;&emsp;整个getTask操作是一个自旋，其主要完成：
- 1、workQueue.take：如果阻塞队列为空，当前线程会被挂起等待；当队列中有任务加入时，线程被唤醒，take方法返回任务，并执行；
- 2、workQueue.poll：如果在keepAliveTime时间内，阻塞队列还是没有任务，则返回null；
