---
layout: post
title: "事务隔离级别、锁与日志操作"
subtitle: '只想说的通俗易懂'
author: "Kang"
date: 2020-06-08 11:45:06
header-img: "img/post-head-img/herbal-3504948_1280.jpg"
catalog: true
tags:
  - 数据库
  - 事务
---
一个重要理解前提
1. 数据库的锁是加在数据行所在的索引上的。不同的隔离级别是在数据可靠性和并发性之间的均衡取舍，隔离级别越高，对应的并发性能越差，数据越安全可靠。  
2. RR与RC所存在锁的区别：<font color="red">RC不存在GAP间隙锁，而RR中存在。同样可知RC少了Next-key lock。</font>  
3. 每一个隔离级别都是对上一个隔离级别一个问题点的处理。     

> 还是没理解的点：对于同一条数据，在RR下是怎么防止快照读下变更覆盖的??  -- 在提交阶段检测DB_TRX_ID版本号or业务自己控制版本号？？

## 事务的隔离级别
- 未提交读(Read_Uncommit)    
&emsp;&emsp;B使用查询语句可能会读到A未提交的行（脏读）

- 提交读(Read_Commit,Oracle默认级别)  
  &emsp;&emsp;**<font color="red">对于某个正在操作的事务，在整个事务期间，若其他事务前后修改并提交了两次一个变量，那么在提交之后当前这个事务发生了读取，那么可以读取并检测到这两次值的变化，也即可以一直读取到最新的提交值</font>**      
  &emsp;&emsp;(解决读未提交,存在不可重复读<font color="green">[优化：通过整个事务加锁]</font>和幻读)

- 可重复读(Repeatable_Read,Mysql默认级别)  
&emsp;&emsp;**<font color="red">一个事务只能读到另一个已经提交的事务修改过的数据，但是第一次读过某条记录后，即使其他事务修改了该记录的值并且提交，该事务之后再读该条记录时，读到的仍是第一次读到的值，而不是每次都读到不同的数据，这种隔离级别就称之为可重复读。</font>**      
&emsp;&emsp;<font color="red">在当前事务提交之前，其他任何事务均不可以修改或删除当前事务已读取的数据。只有当整个事务完成后，记录锁才会被释放，所以在同一个事务中，多次读取同一条数据肯定是相同的，但是对于其他事务插入的数据没法管理，所以对于新增数据，没有办法管理，会出现幻读。</font>      
&emsp;&emsp;(解决不可重复读[同一条数据读]，但是还是存在幻读<font color="green">[优化：通过GAP间隙锁进行减轻]</font>)  

- 串行化：  
&emsp;&emsp;事务操作加锁，构成操作队列。(解决不可重复读与幻读)

## InnoDB几种锁
#### 总览
- Record Lock：单个行记录上的锁，我们通常讲的行锁，<font color="red">它的实质是通过对索引的加锁实现</font>；只有通过索引条件检索数据，InnoDB才使用行级锁，否则，InnoDB将使用表锁。在事务隔离级别为读已提交下，仅采用Record Lock。   
- Gap Lock：间隙锁，锁定一个间隙范围，但不包含记录本身；    
- Next-Key Lock：Record Lock+Gap Lock，锁定一个范围，并且锁定记录本身。     
- 意向锁：意向锁不需要用户关心，每次事务A做行级锁等的时候，会在全局设置一个意向锁，当另一个事务B<font color="red">需要全局锁做操作时</font>，不需要事务B去做每一行数据的锁检查，而只需要看全局意向锁就行，所以意向锁和全局锁阻塞。意向锁与意向锁之间不阻塞；
- 表锁：操作对象是数据表。Mysql大多数锁策略都支持(常见mysql innodb)，是系统开销最低但并发性最低的一个锁策略。事务t对整个表加读锁，则其他事务可读不可写，若加写锁，则其他事务增删改都不行。通过添加表锁，达到部分情况下的快阻塞。  

####  临界锁 next-key
&emsp;&emsp;后面MVCC分析可知，可重复读和提交读是矛盾的。在同一个事务里，如果保证了可重复读，就会看不到其他事务的提交读取的总是快照旧数据，违背了提交读；如果保证了提交读，就会导致前后两次读到的结果不一致，违背了可重复读。    
&emsp;&emsp;为<font color="blue">左开右闭</font>，左间隙与行锁组合起来用就叫做Next-Key Lock，间隙锁主要创建在联合索引或者主键索引中，其是将查找到的主键前后节点之间的插入关闭。  

####  GAP间隙锁
&emsp;&emsp;范围查询或者等值查询时，若记录不存在则退化为GAP锁；

![mysql-Next-Key Lock示例1](https://raw.githubusercontent.com/kangzhihu/images/master/mysql-%E9%97%B4%E9%9A%99%E9%94%81%E7%A4%BA%E6%84%8F%E5%9B%BE.png)

````sql
-- session 1:
start  transaction;
select * from news where number>4 for update;

-- session 2:
start  transaction;
update news set id=2 where number=4 ;#(执行成功)
update news set id=4 where number=4 ;#(阻塞)
update news set id=5 where number=5 ;#(阻塞)
insert into news value(2,3);#(执行成功)
insert into news value(null,13);#(阻塞)
````

```sql
-- session 1:
start  transaction;
select * from news where id>16 for update;#(间隙锁为13->无穷大)

-- session 2:
start  transaction;
insert into news value(14,3);#(阻塞)
```

&emsp;&emsp;检索条件number>4,向左取得最靠近的值4作为左区间，向右取无穷大，因此，session
1的间隙锁的范围（4，无穷大）。   
&emsp;&emsp;如果检索条件number>=4，那么，第一个id=4&number=4也会被阻塞。
     
&emsp;&emsp;next-key间隙锁其实包含了记录锁和间隙锁，即锁定一个范围，并且锁定记录本身防止其他事务的修改，从而防止幻读出现，InnoDB默认加锁方式是next-key锁。<font color="red">在RR级别下，通过next-key锁消除了不可重复读问题，并解决了部分情况下幻读的问题</font>

##### GAP锁在delet和insert中的使用
&emsp;&emsp;假如存在表ta：

| id | a    | b    | c    |
| -- | -- | -- | -- |
|  1 |    1 |   10 |  100 |
|  2 |    3 |   20 |   99 |
|  3 |    5 |   50 |   80 |

&emsp;&emsp;并发事务：

|T1 | T2 |
|--|--|--|
|begin；| begin；|
|delete from ta where a = 4;//ok, 0 rows affected| |
| | delete from ta where a = 4; //ok, 0 rows affected |
|insert into ta(a,b,c) values(4, 11, 3),(4, 2, 5);//wating,被阻塞 |  |
| | insert into ta(a,b,c) values(4, 11, 3),(4, 2, 5); //ERROR 1213 (40001): Deadlock found when trying to get lock; |
|T1执行完成， 2 rows affected(报错后T2释放锁？) | 	|

加锁分析
1. delete时，delete的where子句没有满足条件的记录，而对于不存在的记录 并且在RR级别下，delete加锁类型为gap lock，**<font color="red">gap lock之间是兼容的</font>**，所以两个事务都能成功执行delete；关于gap lock可以参考文章[加锁分析](https://www.cnblogs.com/tutar/p/5878651.html)。这里的gap范围是索引a列(3,5)的范围。
2. insert时，其加锁过程为先在插入间隙上获取插入意向锁，插入数据后再获取插入行上的排它锁。又插入意向锁与gap lock和 Next-key lock冲突，即一个事务想要获取插入意向锁，如果有其他事务已经加了gap lock或 Next-key lock，则会阻塞。
3. 场景中两个事务都持有gap lock，然后又申请插入意向锁，此时都被阻塞，循环等待造成死锁。

#### 意向锁
&emsp;&emsp;意向锁包括共享意向锁(读锁)&排它意向锁(写锁)    
&emsp;&emsp;排它意向锁是一种Gap锁，不是意向锁，在insert操作时产生。      
&emsp;&emsp;insert在做插入时，其加锁过程为先在插入间隙上获取插入意向锁，插入数据后再获取插入行上的排它锁。插入意向锁与gap lock和 Next-key lock冲突的，需要等待这两种锁释放。   
   
#### 锁的兼容性表    
![锁兼容矩阵](https://raw.githubusercontent.com/kangzhihu/images/master/mysql-%E9%94%81%E5%85%BC%E5%AE%B9%E8%A1%A8.jpg)  
&emsp;&emsp;可以看出插入意向锁(GAP)和gap lock和 Next-key lock是冲突的，需要阻塞等待。注意：RC比RR少了GAP锁和Next-key lock。  

[一个死锁参考](https://my.oschina.net/hebaodan/blog/1835966)
