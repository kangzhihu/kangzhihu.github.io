---
layout: post
title: "基础-谈谈SPI对双亲委托模型的破坏"
subtitle: '关于SPI对双亲委托模型的破坏'
author: "Kang"
date:       2019-09-08 15:40:47
header-img: "img/post-head-img/bathing-1382786-1280x960.jpg"
catalog: true
tags:
  - Java基础
---

### 准备知识

##### 1.XXX.class与Class.forName 加载类区别
- XXX.class仅仅是把类加载进JVM而没有做任何类的初始化，故静态代码块&静态变量也不会被加载；  
- Class.forName的形式是初始化了类的(包括初始化静态变量和静态代码快) -- 这就是为啥最开始加载驱动使用Class.forName("com.mysql.jdbc.Driver")

##### 2.双亲委托模型
&emsp;&emsp;在加载类时，将加载动作委托给自己的父类加载器，当基本加载器仍然不能加载时，则子类本身进行加载，否则抛异常。


### 以JDBC的加载为例谈谈SPI如何破坏双亲委托模型的
&emsp;&emsp;之前常见的JDBC加载过程如下：
```java
 // 1.加载数据访问驱动
Class.forName("com.mysql.jdbc.Driver");
//2.连接到数据"库"上去
Connection conn= DriverManager.getConnection("jdbc:mysql://localhost:3306/mydb?characterEncoding=GBK", "root", "");
```
Class.forName只是为了驱动类的初始化过程，使用DriverManager.registerDriver向DriverManager中注入Driver对象。   

#### 破坏双亲委派模型的情况
&emsp;&emsp;在JDBC4.0以后以spi的形式去加载驱动，在mysql的jar包中的META-INF/services/java.sql.Driver文件下配置有具体的驱动使用类。在
DriverManager中SPI加载过程如下：
```java
public class DriverManager {
    static {
        loadInitialDrivers();
        println("JDBC DriverManager initialized");
    }
    private static void loadInitialDrivers() {
     // 省略...
     AccessController.doPrivileged(new PrivilegedAction<Void>() {
            public Void run() {

                ServiceLoader<Driver> loadedDrivers = ServiceLoader.load(Driver.class);   // SPI加载
                Iterator<Driver> driversIterator = loadedDrivers.iterator();
                try{
                    while(driversIterator.hasNext()) {
                        driversIterator.next();     //  循环处理
                    }
                } catch(Throwable t) {
                // Do nothing
                }
                return null;
            }
        });
       // 省略...
    }
}
```
&emsp;&emsp;我们可以看到，ServiceLoader.load进行了加载，其具体代码如下：
```java
public static <S> ServiceLoader<S> load(Class<S> service) {
    ClassLoader cl = Thread.currentThread().getContextClassLoader();   // 封装外部加载器，也即线程上下文类加载器
    return ServiceLoader.load(service, cl);
}
public static <S> ServiceLoader<S> load(Class<S> service, ClassLoader loader){
    return new ServiceLoader<>(service, loader);
}
```
&emsp;&emsp;其将当前上下文的加载器，传入到ServiceLoader的对象构造当中，ServiceLoader对象拿到了线程上下文类加载器。在上面的处理中，
loadedDrivers.iterator()获取到的是内部类LazyIterator的静态代理类。driversIterator.next()转调LazyIterator的nextService()，最后对
具体的类进行加载：
```java
private S nextService() {
    // 省略...
    try {
        c = Class.forName(cn, false, loader);   //加载具体的类，使用外部传入的加载器loader
    } catch (ClassNotFoundException x) {
        fail(service,
             "Provider " + cn + " not found");
    }
    // 省略...
}
```
&emsp;&emsp;从上面可以看到，数据的加载具体工作在ServiceLoader中，但是由DriverManager进行管控。rt.jar的ClassLoader是启动类加载器，而具体的驱动实现是外部厂商各自实现。   
&emsp;&emsp;<font color='red'>这样就实现了对双亲委托模型的破坏：做到了父级类加载器加载了子级路径中的类，</font>虽然是通过持有子类加载器的方式。
