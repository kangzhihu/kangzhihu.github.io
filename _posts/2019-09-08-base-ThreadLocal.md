---
layout: post
title: "基础-ThreadLocal"
subtitle: '关于ThreadLocal的思考'
author: "Kang"
date:       2019-09-08 12:40:47
header-img: "img/post-head-img/falloxbow-1058032-1278x533.jpg"
catalog: true
tags:
  - Java基础
---
## 常见ThreadLocal使用方式
````java
static  class ThreadId{
    private static final AtomicInteger nextId = new AtomicInteger(0);
    //线程本地变量，为每个线程关联一个唯一的序号
    private static final ThreadLocal<Integer> threadLocal =
            new ThreadLocal<Integer>() {
                @Override
                protected Integer initialValue() {
                    return nextId.getAndIncrement();
                }
            };

    //提供外部静态调用
    public static int get() {
        return threadLocal.get();
    }
}

````
可以看到，对于使用方来说，调用的是ThreadId.get()，在底层上，查看ThreadLocal源码存在：
```java
//ThreadLocal
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this); // 关键点
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    return setInitialValue();
}
```
&emsp;&emsp;需要注意一个关键点：所有的线程使用方来说，在实例化自身的ThreadLocalMap.Entry的时候，都将类ThreadId的**属性变量threadLocal**作为自身的key。

![ThreadLocal](https://raw.githubusercontent.com/kangzhihu/images/master/ThreadLocal.png)

## 为何要使用ThreadLocal
&emsp;&emsp;弱引用：当一个对象***仅仅***被weak reference指向, 而<font color="red">没有任何其他strong reference指向的时候(或者编译器认为后面不会被使用)</font>, 如果GC运行, 那么这个对象就会被回收。    
&emsp;&emsp;ThreadLocal一个很大的作用是简化对象的回收，若有多处被使用，当强引用不存在时，对象关联的多处弱引用可以被自动回收。

```java
 byte[] cacheData = new byte[100 * 1024 * 1024];   //强引用
 WeakReference<byte[]> cacheRef = new WeakReference<>(cacheData);   // 弱引用，持有强引用
 System.gc();
 System.out.println("第一次GC前" + cacheRef.get());   //打印结果时，对象仍然存在
 cacheData = null;                                   // 将弱引用持有的强引用对象关闭
 System.gc();                                       //再次gc
 System.out.println("第一次GC前" + cacheRef.get());   // get返回为null
```

## 为何会产生泄露
&emsp;&emsp;从上图可以看到，当前线程通过ThreadLocal获取了当前线程本身的ThreadLocalMap引用。ThreadLocalMap中Entity的key是外部使用的ThreadLocal的一个弱引用，当发生GC后，若此时**属性变量threadLocal被释放回收**，Entity的key弱引用被回收，但是ThreadLocalMap仍然被当前线程持有，所以出现Entity仍然被持有但其属性key为空情况。    
&emsp;&emsp;在使用ThreadLocal的get()、set()、remove()时候，子方法getEntry才去进行空key的Entity擦除。当ThreadLocalMap后面不被访问时，一直要等到Thread被回收，这部分Entity才能被一起回收。   
&emsp;&emsp;**所以泄漏的根本原因是在不合理的情况下，ThreadLocal变量被回收后，由于ThreadLocalMap的生命周期跟Thread一样长，此时如果没有自动触发或者手动删除对应key的value就会导致内存泄漏，而不是因为弱引用。**

## 为何要选择若引用
> + key 使用强引用：引用的ThreadLocal的对象被回收了，但是ThreadLocalMap还持有ThreadLocal的强引用，如果没有手动删除，ThreadLocal不会被回收，导致Entry内存泄漏。   
> + key 使用弱引用：引用的ThreadLocal的对象被回收了，由于ThreadLocalMap持有ThreadLocal的弱引用，即使没有手动删除，ThreadLocal也会被回收。value在下一次ThreadLocalMap调用set,get，remove的时候会被清除。    

&emsp;&emsp;可以看出，由于ThreadLocalMap的生命周期跟Thread一样长，两种情况下如果都没有手动删除对应key，都会导致内存泄漏，在ThreadLocal被回收后，使用弱引用可以多一层保障。


## ThreadLocal中会不会发生WeakReference key提前被回收
&emsp;&emsp;其实不会的，前面提到，WeakReference被回收的前提是对象没有任何其他strong reference指向的时候。我们在业务代码中，一般ThreadLocal这个强引用在业务还被需要的情况下不会主动释放的，也就是其实还是有强引用存在的，所以key没有达到弱引用被回收的条件。
