---
layout: post
title: "诊断-arthas"
subtitle: '命令与使用'
author: "Kang"
date: 2021-01-27 14:22:06
header-img: "img/post-head-img/arthas.jfif"
catalog: true
tags:
  - 诊断
  - arthas
---
## arthas使用
#### 一、常用命令

##### 基础命令

```bash
help ——查看命令帮助信息
cat ——打印文件内容，和linux里的cat命令类似
echo ––打印参数，和linux里的echo命令类似
grep ——匹配查找，和linux里的grep命令类似
tee ——复制标准输入到标准输出和指定的文件，和linux里的tee命令类似
pwd ——返回当前的工作目录，和linux命令类似
cls ——清空当前屏幕区域
session ——查看当前会话的信息
reset ——重置增强类，将被 Arthas 增强过的类全部还原，Arthas 服务端关闭时会重置所有增强过的类
version ——输出当前目标 Java 进程所加载的 Arthas 版本号
history ——打印命令历史
quit ——退出当前 Arthas 客户端，其他 Arthas 客户端不受影响
stop ——关闭 Arthas 服务端，所有 Arthas 客户端全部退出
keymap ——Arthas快捷键列表及自定义快捷键
```

##### JVM相关命令

```bash
dashboard ——当前系统的实时数据面板
thread ——查看当前 JVM 的线程堆栈信息
jvm ——查看当前 JVM 的信息
sysprop ——查看和修改JVM的系统属性
sysenv ——查看JVM的环境变量
vmoption ——查看和修改JVM里诊断相关的option
perfcounter ——查看当前 JVM 的Perf Counter信息
logger ——查看和修改logger
getstatic ——查看类的静态属性
ognl ——执行ognl表达式
mbean ——查看 Mbean 的信息
heapdump ——dump java heap, 类似jmap命令的heap dump功能
profiler ——采集CPU火焰图，配合start(开始)、getSamples(获取样本数)、status(状态)、stop(结束)使用
```

##### class/classloader相关

```bash
sc ——查看JVM已加载的类信息
sm ——查看已加载类的方法信息
jad ——反编译指定已加载类的源码
mc ——内存编译器，内存编译.java文件为.class文件
redefine ——加载外部的.class文件，redefine到JVM里
dump ——dump 已加载类的 byte code 到特定目录
classloader ——查看classloader的继承树，urls，类加载信息，使用classloader去getResource
```



#### 二、使用

##### 监控大盘

```bash
$ dashboard  --可用来查看是否存在内存泄漏
```

查看JVM相关信息，包括线程(按照cpu占用百分比倒排)、内存(堆空间实时情况)、GC情况等数据。

##### 查看线程

```bash
$ thread (pid  -i  -- 查看特定线程)
$ thread pid ｜ grep 'main('  --查找main class
$ thread -b --查看死锁的线程
```

##### 查找jvm已加载的类

```bash
$ sc *UserController (-d 打印详细加载信息)
```

##### 查看jvm已加载类的方法信息

```bash
$ sm com.example.demo.arthas.user.UserController (methadName 只查看特定的方法) 
```

##### 反编译jvm已加载的类

```bash
$ jad com.example.demo.arthas.user.UserController
```

##### 实时查看方法的返回值、抛出异常、入参

```bash
$ watch [class-pattern] [method-pattern] [express] [condition-express]
[express]:主要由ognl表达式,可组合:[params(参数), returnObj(返回值), target(当前对象), throwExp(异常信息)]
[condition-express]:
    4个观察事件点:-b 方法调用前，-e 方法异常后，-s 方法返回后，-f 方法结束后; 
    -x 2:指定输出结果的属性遍历深度，默认为1
    -n 2:指定执行次数
    条件表达式: 'params[0]<0' 根据参数值过滤、 '#cost>200' 根据耗时过滤
eg:
$ watch com.example.demo.arthas.user.UserController hello  "{params,returnObj}" -x 3  --监控该方法的入参、返回值,并且打印属性遍历深度2
$ watch demo.MathGame primeFactors "{params[0],throwExp}" -e -x 2 观察异常信息
$ watch com.example.demo.arthas.user.UserController hello  'target.log' -n 2  观察参数变化情况
```



##### trace 方法内部调用路径，并输出方法路径上的每个节点上耗时

```bash
$ trace [class-pattern] [method-pattern] [condition-express]  --与watch相似，[condition-express]: -n 次数、 #cost    耗时
$ [arthas@10107]$ trace com.example.demo.arthas.user.UserController hello -n 1 --监控查看该方法内部调用耗时情况
```

##### stack 输出当前方法被调用的调用路径

```bash
$ trace [class-pattern] [method-pattern] [condition-express]  --与watch相似，[condition-express]: -n 次数
```

##### refine 热更新代码

```bash
1.修改源码：
$ jad --source-only com.example.demo.arthas.user.UserController > /tmp/UserController.java  --导出源码到目标文件夹
2.使用memory compile，结合classLoader重新编译源码
$ sc -d *UserController | grep classLoaderHash  --获取该类对应的classLoader hash
$ mc -c 5674cd4d /tmp/UserController.java  -d /tmp  --mc重新编译到目标文件夹
3.使用refine重新加载class文件
$ mc -c 5674cd4d /tmp/UserController.java  -d /tmp
```

##### ognl使用

```bash
动态更新应用Logger Level:
$ sc -d com.example.demo.arthas.user.UserController | grep classLoaderHash。--获取classLoader hash
$ ognl -c 1be6f5c3 '@com.example.demo.arthas.user.UserController@logger'  --根据hash获取UserController.logger属性
$ ognl -c 5674cd4d '@com.example.demo.arthas.user.UserController@logger.setLevel(@ch.qos.lognull.classic.Level@DEBUG)'。--根据hash重新设置logger的值


$ ognl -c 1be6f5c3 '@org.slf4j.LoggerFactory@getLogger("root").setLevel(@ch.qos.logback.classic.Level@DEBUG)'  --通过获取root logger，修改全局的logger level
```

##### tt(TimeTunnel:监控记录指定方法每次调用的入参和返回信息,并能对这些不同的时间下调用进行观测)

```bash
$ tt -t org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter invokeHandlerMethod --使用tt命令获取到spring context
$ tt -i 1000 -w 'target.getApplicationContext()' --使用tt命令从调用记录里获取到spring context
$ tt -i 1000 -w 'target.getApplicationContext().getBean("helloWorldService").getHelloMessage()' --获取spring bean，并调用函数
```

tt 命令由于保存了当时调用的所有现场信息，所以我们可以自己主动对一个 INDEX 编号的时间片自主发起一次调用，主要是用-p参数，此时调用的路径发生了变化，又原来的程序发起变成了 Arthas 自己的内部线程发起的调用了。

##### heapdump生成堆文件

```bash
$ heapdump --live /tmp/jvm.hprof
```



#### 使用

##### 动态修改日志级别

###### (1)、查找要修改日志级别的类，并找出classLoaderHash

```bash
sc -d *XXXImpl | grep classLoaderHash
```

###### (2)、查看当前日志级别

```bash
logger -c 2b53eef4 (2b53eef4–表示上面返回的classLoaderHash类的哈希码)
```

######  (3)、查看root权限下当前logger的日志级别

```bash
 logger -c 18b4aac2
```

ps:返回结果中name为根配置(访问权限，也就是ROOT)，除了级别，也可以在返回结果中看到当前日志包信息、日志输出目录等信息

###### (4)、修改日志级别

```bash
logger -c 2b53eef4 --name ROOT --level debug 
```

ps:注意name后面的值要与前面返回的一样，包括大小写。修改后貌整个项目的日志级别都被修改了？