---
layout: post
title: "基础-JVM内存模型"
subtitle: 'JVM内存模型粗理解'
author: "Kang"
date: 2019-09-09 13:10:47
header-img: "img/post-head-img/jvm.png" #kitten-1639027-1279x853.jpg
catalog: true
tags:
  - Java基础
---

## 部分基础概念
Minor GC：发生于年轻代  
Major GC：发生于永久代  
Full GC ：发生于整个堆空间    

动态链接：将方法中的符号引用变更为具体的对象方法指针。     
内存溢出：要求分配的内存超出了系统能给你的。    
内存泄漏：申请内存后，一直可达无法进行内存释放。   

安全点：safepoint，方法调用前后、循环跳转、异常跳转等流程切换时间周期较长的特殊位置，应用线程检查GC标志主动停止或者GC抢占方式停止应用线程。SafePoint是活动线程与GC之间的交互。  

安全区域：是指在一段代码片段中，引用关系不会发生变化，在该区域的任何地方发生gc都是安全的。    
&emsp;&emsp;当代码执行到安全区域时，首先标示自己已经进入了安全区域，那样如果在这段时间里jvm发起gc，就不用管标示自己在安全区域的那些线程了(认为这些线程都是"干净"的)，在线程离开安全区域时，会检查系统是否正在执行gc，如果是那么就等到gc完成后再离开安全区域。   

方法区和永久代的关系：方法区和永久代/元数据区的关系很像Java中接口和类的关系，类实现了接口，而永久代就是HotSpot虚拟机对虚拟机规范中方法区的一种实现方式。    

## JVM内存模型
借用网上一幅图：
![JVM内存模型全部](https://raw.githubusercontent.com/kangzhihu/images/master/JVM-全图.png)

&emsp;&emsp;JVM内存由五大块组成：程序计数器，方法区、本地方法栈、堆、栈。 
![JVM内存模型](https://raw.githubusercontent.com/kangzhihu/images/master/%E5%9F%BA%E7%A1%80-JVM%E5%86%85%E5%AD%98%E6%A8%A1%E5%9E%8B.png)    
&emsp;&emsp;上图为一个简单的JVM内存模型，其中，methodOne调用了methodTwo

>小贴士：Java对象是否一定分配在堆上？   
&emsp;&emsp;JVM在Server模式下的逃逸分析可以分析出某个对象是否永远只在某个方法、线程的范围内，并没有“逃逸”出这个范围，逃逸分析的一个结果就是对于某些未逃逸对象可以直接在栈上分配，由于该对象一定是局部的，所以栈上分配不会有问题。

Java对象内存分配策略：  
![Java对象内存分配策略](https://raw.githubusercontent.com/kangzhihu/images/master/java%E5%9F%BA%E7%A1%80-Java%E5%AF%B9%E8%B1%A1%E5%86%85%E5%AD%98%E5%88%86%E9%85%8D%E7%AD%96%E7%95%A5.png)    
此处不展开讲解，了解更多请自行百度。    


### 模型变更点
&emsp;&emsp;1、在jdk8之前，永久代，也即方法区，经常发生OutOfMemory异常，调优时也不知道设置多大合理，所以在jdk8时将永久代去掉转而使用元空间来代替。    
&emsp;&emsp;2、仍然存储的是类的信息、常量池、方法数据、方法代码等可长久驻存在内存中的数据。     
&emsp;&emsp;3、元空间与永久代之间最大的区别在于：元空间并不在虚拟机中，而是使用本地内存。     

### 示例解说

```java
public static void main(String args[]) { 
        Math math = new Math();
        math.compute(); 
        Math math1 = new Math();
        math1.compute(); 
 } 
public int compute(){
        int a=3;
        int b=5;
        int c=(a+b)*10;
        return c;
    }
```
通过javap -c XXX.class 查看文件： 
``` sh
public int compute();
    Code:
       0: iconst_3
       1: istore_1
       2: iconst_5
       3: istore_2
       4: iload_1
       5: iload_2
       6: iadd
       7: bipush       10 //无8：可以认为其JVM指令行号被常量10所占
       9: imul
       10: istore_3
       11: iload_3
       12: ireturn
```
那么当一个线程X中执行到compute时，其执行过程为：  
>
1. iconst_3 将常量3压入到操作数栈中；-->操作数栈
2. istore_1
   将刚才操作数栈的int常量存入到局部变量1(位置1)中；-->局部变量表
3. iconst_5 将常量5压入到操作数栈中；-->操作数栈
4. istore_2 将刚才操作数栈的常量存入到局部变量2（表位置2）中；-->局部变量表
5. iload_1 将局部变量1压入操作数栈；
6. iload_2 将局部变量2压入操作数栈；
7. iadd 操作数栈中执行int类型的相加；
8. bipush 10 将常量10直接压入到操作数栈中；
9. imul int类型乘法操作；
10. istore_3 完成整个计算后，将结果存入局部变量表中3；
11. iload_3 将局部变量表中的局部变量3压入操作数栈；
12. ireturn 从操作数栈栈顶弹出到上层方法

&emsp;&emsp;ps:先load到操作数栈，后入局部变量表，操作时再从局部变量表压入操作数栈

程序计数器：其记载的是当前操作数栈的JVM指令行号.比如运行到iload_1时，程序计数器当前值为iload_1的行号4.    
方法出口：线程中当前方法计算完成弹出结果后，整个结果在上层方法中应该存储在哪一行(局部变量表地址？)。    
动态链接：指向了当前方法字节码在MetaSpace元数据区中具体的存储位置(符号引用)，当被调用时，将根据该符号引用将方法字节码加载进来。

[阅读1](https://www.cnblogs.com/dingyingsi/p/3760730.html)

## Heap分代模型
![Heap分代模型](https://raw.githubusercontent.com/kangzhihu/images/master/%E5%9F%BA%E7%A1%80-Heap%E5%88%86%E4%BB%A3%E6%A8%A1%E5%9E%8B.png)    
&emsp;&emsp;元数据区和堆是共享内存的。在内存一定时，调整任何一个的大小，都有可能发生挤占另一个空间的情况。  
 
#### 默认分配比例  
1. 新生代:老年代 = 1:2;      
2. Eden:Survivor = 8:1;   
3. Eden区中当发生minor GC时对象才会进入survivor区   
4. 年轻代Minor GC发生16被移动到老年代  

## 内存分配
#### 两种可用空间查找方式
&emsp;&emsp;指针碰撞：对于<font color="green">连续的</font>剩余空闲空间，当需要为对象分配时，只需要将当前指向已用和空闲的边界指针向空闲区域移动相应距离即可。   
&emsp;&emsp;空闲列表：对于<font color="green">不规则(非连续)的</font>剩余空闲空间，为了需要知道哪些零散空间可用，则需要一张列表去表明哪些内存空间可用。当需要分配空间时，就在该表中查找。   

#### 分配方式的选择
&emsp;&emsp;根据具体的回收算法，取决于堆空余空间是否完整，而是否完整进一步取决于所使用的垃圾回收算法。   
&emsp;&emsp;对于CMS来说，堆中的地址指针是多线程共享的，所以指针是并发点(cas串行化)。为了优化这种指针的共享碰撞，所以采用了TLAB做线程块化处理。    
&emsp;&emsp;Thread Local Allaction Buffer(TLAB，类似空闲列表),即考虑到eden多线程并发度较高，在eden区再次划分出一个小的线程块内存，这样每个线程操作时都在自己的内存中操作，相互直接不会共享使用同一个地址指针，避免了碰撞。


#### GC空间分配担保机制
&emsp;&emsp;新对象分配过程：
1. 先去Eden区查找，若空余空间不足且已经存在数据，则触发一次Minor GC(or FULL GC，见下面解释)。
2. 若GC后(或本来也没数据)Eden区空间仍然不足，则尝试将eden中存活的对象全部转移到survivor区中(当前正在使用的，大对象直接进入old区)，若survivor区不能满足，则对象进入old区(老年代进行分配担保，把Survivor无法容纳的对象放到老年代)。    
&emsp;&emsp;上面提到的在进行Minor GC之前会先做内存判断，检查老年代最大可用的**连续空间**是否大于新生代所有对象的总空间：  
- 如果大于，则此次Minor GC是安全的直接进行Minor GC即可；
- 如果小于，则虚拟机会查看HandlePromotionFailure设置值是否允许担保失败
    + HandlePromotionFailure=true and 老年代最大可用连续空间大于历次晋升到老年代的对象的平均大小（预估） --> 允许失败则仍然进行**有风险下的Minor GC**
    + HandlePromotionFailure=false or 小于平均值-->需要升级为Full GC进行全面空间释放（让老年代能腾出空间）。

## 垃圾回收
#### 回收算法种类
1. 标记-清除：清理后会出现碎片
2. 复制算法：需要存在一个对等大小的空白空间
3. 标记-整理-压缩算法：标记之后，它不是直接清理可回收对象，而是将存活对象都向一端移动，然后清理掉端边界以外的内存。
4. 分代收集算法：按照回收特点，将内存划分为不同的区域，不同区域采用不同算法进行回收。

&emsp;&emsp;整体上看，是采用了分代回收算法，局部来看，新生代采用了复制算法，而老年代由于回收频率较低，采用了标记-清理-压缩算法。


#### 分代回收过程
&emsp;&emsp;新增对象使用eden区时，若eden区满，则将eden做清理，存活的对象复制到from/to区，后面eden再次满时，则将(eden+from，假设此时使用from区)一起copy到to区域并清理eden+from区域，同样下次eden满后copy(eden+to)到from。每次的copy时将会做年龄的判断，若达到指定年龄后，直接移动到老年代。  


### 垃圾回收器

#### GC Roots 查找根 
Java语言里，可作为GC Roots对象的包括如下几种，可以看出都是线程运行可能在使用的地方：   
&emsp;&emsp; a.虚拟机栈(栈桢中的本地变量表)中的引用的对象(临时正在指向堆)   
&emsp;&emsp; b.方法区中的类静态属性引用的对象(常量指向堆)   
&emsp;&emsp; c.方法区中的常量引用的对象 (常量指向堆)  
&emsp;&emsp; d.本地方法栈中JNI的引用的对象(临时正在指向堆)  
&emsp;&emsp; e.元数据区(常量池)中引用的对象(常量指向堆)    

ps:不可达对象不一定会被回收，finalize()中可以做最后的挽救(重新建立连接)

#### CMS与G1回收器整体比较
1. CMS收集器在Minor GC时会暂停所有的应用线程(大部分应用程序，停顿导致的延迟都是可以忽略不计)，并以多线程的方式进行垃圾回收。在Major GC时不再暂停应用线程，而是使用若干个后台线程定期的对老年代空间进行扫描，及时回收其中不再使用的对象。    
2. G1相对于CMS的优势而言是内存碎片的产生率大大降低并方便性能调优。   
3. G1在本质上仍然属于分代收集器，新生代的垃圾收集依然采用暂停所有应用线程的方式，将存活对象拷贝到老年代或者Survivor空间。其改进点：  
- 在物理存储上打破了原有的新生代和老年代的界限，使用相同一块内存，但是将内存划分为一些基本的单元区域。      
- 以基础单元区域为操作单位，这样就不会存在散乱的内存碎片。  
- 增加Humongous区域，当一个对象的大小超过单元区域的50%时，则认为是巨型对象，将直接被放在Humongous区。   

### CMS
#### CMS垃圾收集器标记过程
- 1、<font color="red">停顿</font>标记GC Roots可<font color="red">直接</font>可到达的对象；  
- 2、并发查找通过上面被标记的对象可达对象；  
- 3、<font color="red">停顿</font>再次处理上面并发标记期间重现变动的对象；  
- 4、并发清理    
  &emsp;&emsp;CMS垃圾回收由于使用了并发清理的功能，使得整个过程的停顿时间较短，但是明显有下面两个缺点：
  - 1、CMS时CPU较高；
  - 2、CMS采用标记-清理算法进行回收，由于并行清理不对内存空间进行压缩，所以容易产生碎片。  
  ps:设置-XX:CMSFullGCsBeforeCompaction=n 多少次FULL GC后进行内存空间压缩


### G1

![G1垃圾收集器模型](https://raw.githubusercontent.com/kangzhihu/images/master/g1%E5%9E%83%E5%9C%BE%E6%94%B6%E9%9B%86%E5%99%A8.png)

#### 新生代清理<暂时这样简单理解>
&emsp;&emsp;当只需要清理新生代的时候，是否也需要通过GC Roots遍历整个空间？    
&emsp;&emsp;其实是不需要的，通过以下两种不同的方式进行单独标记：  
- 对于CMS，其存在一个单独的point-out空间，这个空间记录了所有的新生代中被老年代使用的对象的引用，所以扫描根时，仅仅需要扫描这一块区域即可标记新生代对象中可达性。
- 对于G1，每个Region中都有一个RSet空间，记录的是其他Region中的对象引用本Region对象的关系，是一种point-in的关系，即：谁引用了我的对象。因为G1分了很多Region，需要回收那个区域的时候，只需要判断要回收的区域是否有其他对象引用了该区域里的对象，即只需要找待回收区域的根对象即可(只将引用region作为根)，避免无效扫描。RSet中存储结构是一个卡表(Hash
  Table)，其k是引用区域的起始地址，v是被引用对象在当前区域index地址集合，-
  由于g1收集器中，Region种类和数量都很多，所以使用CSet(Collection
  Set)来记录GC要收集的Region的集合。   
  ![G1垃圾回收器RSet](https://raw.githubusercontent.com/kangzhihu/images/master/g1%E5%9E%83%E5%9C%BE%E5%9B%9E%E6%94%B6%E5%99%A8RSet.png)
  
  如上图所示，要回收年轻代的region A,只需要扫描C,D,F
  区域的根对象即可，而不需要扫描整个old区。








[阅读2](https://www.jianshu.com/p/26b112b73c9d)
