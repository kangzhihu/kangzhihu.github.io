---
layout: post
title: "事务隔离级别-MVCC"
subtitle: '只想说的通俗易懂'
author: "Kang"
date: 2020-06-10 11:45:06
header-img: "img/post-head-img/herbal-3504948_1280.jpg"
catalog: true
tags:
  - 数据库
  - 事务
  - MVCC
---
## 写在前面
Read View 、MVCC和 Next-Key Locks 配合的方式如下：  
1、创建事务时，构建Read View确认活跃事务表；  
2、MVCC会为事务操作(读/写)期间涉及到的数据都创建一个版本进行管理，后续都读这个快照；  
3、当产生一个写动作时，则要去判断这个记录的锁情况(更新、当前读等加锁了)，冲突时进行排队等待。

> &emsp;&emsp;<font color="red">**其实MVCC本来的用途是解决写时加锁不能读的问题,也就是增加读的速度,可以代替行锁。** 两者都可以解决幻读问题，但是在**快照读**时使用MVCC,在**当前读**时使用next_key锁。</font>

## MVCC
&emsp;&emsp;Multi-Version Concurrency Control 多版本并发控制。MySQL的大多数事务型存储引擎实现的都不是简单的行级锁。基于提升并发性能的考虑使用了MVCC机制。**MVCC是行级锁的一个变种，是对行锁的优化**，其实现没有标准，但大都实现了非阻塞的读操作，写操作也只锁定必要的行。   
&emsp;&emsp;MVCC在使用READ COMMITTD、REPEATABLE READ这两种隔离级别的事务在执行普通的SEELCT操作时访问记录的版本链的过程，这样子可以使不同事务的读-写、写-读操作并发执行，从而提升系统性能。粗暴的可以认为是一种读写分离锁(虽然不是)，只不过是通过其它方式实现的。
&emsp;&emsp;MVCC只在Repeatable-Read和Read-Commit两个隔离级别下工作。    

&emsp;&emsp;当我们select的时候,因为MVCC的特性使得我们根本不需要锁,因为MVCC所加的记录删除时间的列会帮我们筛选掉幻读的行,从而在不加锁的情况下避免幻读.但是此时数据仍然是可以加入表的.但是当我们需要对表进行修改的时候就不一样了,此时MVCC显然无法满足要求,我们需要保证在一个区间插入的时候其他会话不在此区间进行插入,所采取的策略就是next_key锁. 

### MVCC的两种读

#### 快照读(读锁)
&emsp;&emsp;读取的是快照版本，也就是历史版本。InnoDB默认普通的SELECT就是快照读。<font color="red">产生的快照是发生select的瞬间，而不是开启事务的时候。</font>
```sql
select * from table where ?;
```
#### 当前读(写锁)
&emsp;&emsp;直接从磁盘或 buffer 中获取当前内容的最新数据，读到什么就是什么。并在读取之后，还需要保证其他并发事务不能修改当前记录，对读取记录加锁。  
下面👇的示例中，除了第一条语句，对读取记录加S锁 (共享锁)外，其他的操作，都加的是X锁 (排它锁，注意不是写锁，排它锁其它事务也不能读)。UPDATE、DELETE、INSERT、SELECT …  LOCK IN SHARE MODE、SELECT … FOR UPDATE是当前读。<font color="red">InnoDB默认涉及到数据变更操作和手动SELECT就是快照读。</font>      
```sql
select * from table where ? lock in share mode;
select * from table where ? for update;
insert into table values (…);
update table set ? where ?;
delete from table where ?;
```
&emsp;&emsp;用户一个update语句发送给Mysql Server后涉及到两步操作：  
1. Mysql Server将更新时携带条件的where进行提取，将提取语句投递给InnoDB引擎，InnoDB将返回符合条件的第一条数据，并将当前数据加排它锁(X锁)；--当前读。  
2. MySQL Server收到这条加锁的记录之后，会再发起一个Update请求，更新这条记录。  
3. 一条记录操作完成，再读取下一条记录，直至没有满足条件的记录为止。  

![读取案例](https://raw.githubusercontent.com/kangzhihu/images/master/mysql-读取案例.png)

[阅读-where 提取参考](http://hedengcheng.com/?p=577)

##### 对于当前读的锁操作    
[阅读-加锁参考](http://hedengcheng.com/?p=771)    
- 在RC级别中，若where存在主键可以直接定位，则将主键直接加X排它锁；若为组合索引，则先将组合索引中所有符合条件的索引加X锁，然后再将主键索引中全部加排他锁；若无任何索引，则将会引发所有索引添加X锁；
- 在RR级别中，若where存在主键可以直接定位，则将主键直接加X排它锁；若为组合索引，则先将组合索引中所有符合条件的索引加X锁，然后<font color="blue">将组合索引的间隙加锁(GAP锁)，最后再将对应的主键索引全部加排他锁；若无任何索引，则将会引发所有索引添加X锁并将间隙加GAP锁；</font>-- 二级索引都是加GAP锁而不是X锁       
![RR组合索引下查询加锁情况](https://raw.githubusercontent.com/kangzhihu/images/master/mysql-RR%E9%9A%94%E7%A6%BB%E7%BA%A7%E5%88%AB%E4%B8%8B%E7%BB%84%E5%90%88%E7%B4%A2%E5%BC%95%E8%8C%83%E5%9B%B4%E6%9F%A5%E6%89%BE%E9%94%81%E6%83%85%E5%86%B5%E7%A4%BA%E4%BE%8B.jpg)    
&emsp;&emsp;上图为RR级别下的示例，(id,name)为一个组合索引。在RC级别下类似，只是少了GAP锁。     

###  InnoDB的MVCC版本处理  
&emsp;&emsp;MVCC (Multiversion Concurrency Control)，即多版本并发控制技术,它使得大部分支持行锁的事务引擎，不再单纯的使用行锁来进行数据库的并发控制，取而代之的是把数据库的行锁与行的多个版本结合起来，只需要很小的开销,就可以实现<font color="red">非锁定读</font>，从而大大提高数据库系统的并发性能。    

&emsp;&emsp;每一个业务字段中存在隐藏的几个字段(字段具体顺序不关注)：    
> DELETE_BIT|DB_TRX_ID|DB_ROLL_PTR|DB_ROW_ID     
> 删除位标识(1-已删除，0-未删除)|事务版本id|回滚指针(指向undo log)|行ID(可以认为是PK)   

MVCC每次更新(update,delete也是一种特殊的update)操作都会复制一条新的记录，新记录的创建时间为当前事务id
+ DATA_TRX_ID 字段记录了数据的创建和删除时间，由对数据进行操作的事务的id表示。
+ DATA_ROLL_PTR 指向当前数据的undo log记录，回滚数据就是通过这个指针来找到该记录修改前的信息。
+ DELETE BIT位用于标识该记录是否被删除(copy且设置为1)，这里的不是真正的删除数据，而是标志出来的删除。真正意义的删除是commit后在mysql进行数据的GC，清理历史版本数据的时候。

>小贴士：每当开始一个事务时，当前事务版本号从innodb事务系统申请，并且按照申请顺序严格递增，即currentTransactionId=sysTransactionId。

**具体的DML特性：**
- INSERT：创建一条新数据，DB_TRX_ID中的创建时间为当前事务id，DB_ROLL_PT为NULL
- UPDATE：写一个这行数据的新拷贝，这个拷贝的DB_TRX_ID为当前事务id。它同时也会将当前事务id写到旧行的删除版本里。   
- UPDATE：与更新一样的操作，但是设置了DELETE BIT标志位，若没有commit则只是DELETE BIT设置为1，未真实删除。

可知，为了提高并发度，InnoDb提供了这个「非锁定读」，即不需要等待访问行上的锁释放，读取行的一个快照即可。 既然是多版本读，那么肯定读不到隔壁事务的新插入数据了，所以解决了幻读。

>小贴士：需要提前给自已一个疑问：能不能在两个事务中交叉更新(也即写操作)同一条记录呢？  
>&emsp;&emsp;**<font color="red">这是不可以滴，读后在写时Next-lock就会起作用，第一个事务更新了某条记录后(未提交)，就会给这条记录加锁(排他锁锁定该行)，另一个事务再次更新时就需要等待第一个事务提交了，把锁释放之后才可以继续更新。</font>**而添加的锁就是之前提到的在不同隔离级别下的几种锁。

![mysql-MVCC在底层行中的结构](https://raw.githubusercontent.com/kangzhihu/images/master/mysql-MVCC%E8%AE%B0%E5%BD%95%E8%A1%8C%E4%B8%AD%E7%BB%93%E6%9E%84.png)  
具体的执行过程：<font color="blue">begin事务->用排他锁锁定该行->记录redo log->记录undo log->修改当前行的值，写事务编号(DATA_TRX_ID)，回滚指针指向undo log中的修改前的行</font>。    
&emsp;&emsp;上述过程确切地说是描述了UPDATE的事务过程，其实undo log分insert和update undo log，因为insert时，原始的数据并不存在，所以回滚时把insert undo log丢弃即可，而update undo log则必须遵守上述过程。    

&emsp;&emsp;DB_ROLL_PTR指向的undo log变更链中，保存了每个事务的操作版本，并且后一个log记录指向前一次变更的历史数据记录。  

#### ReadView  
&emsp;&emsp;在实现上，innodb为每个事务构建一个数组，用来保存这个事务启动瞬间，当前正在活跃的所有事务ID（**活跃指的是开启了事务但还未提交**）列表m_ids(ReadView)      
&emsp;&emsp;访问某条记录时，只需要按照下边的步骤判断记录的某个版本是否可见：  
1. 如果被访问数据当前数据行的插入：DATA_TRX_ID属性值小于m_ids列表中最小的事务id，表明生成该版本的事务在生成ReadView前已经提交，所以该版本可以被当前事务访问。  
2. 如果被访问版本的DATA_TRX_ID属性值大于m_ids列表中最大的事务id，表明生成该版本的事务在生成ReadView后才生成，所以该版本不可以被当前事务访问。  
3. 如果被访问版本的DATA_TRX_ID属性值在m_ids列表中最大的事务id和最小事务id之间，那就需要判断一下DATA_TRX_ID属性值是不是在m_ids列表中
  - 如果在，说明创建ReadView时生成该版本的事务还是活跃的，该版本不可以被访问；  
  - 如果不在，说明创建ReadView时生成该版本的事务已经被提交，该版本可以被访问。   

如果某个版本的数据对当前事务不可见的话，那就顺着版本链找到下一个版本的数据，继续按照上边的步骤判断可见性，依此类推，直到版本链中的最后一个版本，如果最后一个版本也不可见的话，那么就意味着该条记录对该事务不可见，查询结果就不包含该记录。
>ReadView读取规则：  
&emsp;&emsp;当前事务在读取数据时，**不能读取当前事务m_ids列表中的事务或比当前事务id大的事务所修改的数据**，为了阻断其他事务的修改，所以使用For Update 添加X锁。  

##### READ COMMITTED下的ReadView
&emsp;&emsp;☞每次读取数据前都生成一个ReadView,这样当前事务持有的ReadView中一直包含最新的事务id，所以一直可以获得当前最新的数据。-- RC会从当前m_ids列表中剔除已提交的事务id，若这个事务id比当前事务id小，那么就能读取到了。  
##### REPEATABLE READ下的ReadView
&emsp;&emsp;☞在第一次读取数据时生成一个ReadView,这样当前事务持有的ReadView一直不会被更新。  


###  MVCC示例解析
[示例来源-MySQL事务隔离级别和MVCC](https://blog.csdn.net/qq_38538733/article/details/88902979)     
本示例中，事务80插入了一行数据：  
![mysql-MVCC示例初始值](https://raw.githubusercontent.com/kangzhihu/images/master/mysql-MVCC%E7%A4%BA%E4%BE%8B%E5%88%9D%E5%A7%8B%E5%80%BC.png)   
##### READ COMMITTED MVCC
&emsp;&emsp;假设现在系统里有两个id分别为100、200的事务在执行：
```sql
# Transaction 100
BEGIN;
UPDATE t SET c = '关羽' WHERE id = 1;
UPDATE t SET c = '张飞' WHERE id = 1;
```
```sql
# Transaction 200
BEGIN;
# 更新了一些别的表的记录
...
```
此刻，表t中id为1的记录得到的版本链表如下所示：  
![mysql-MVCC之RC事务链1](https://raw.githubusercontent.com/kangzhihu/images/master/mysql-MVCC%E4%B9%8BRC%E4%BA%8B%E5%8A%A1%E9%93%BE1.png)      
假设现在有一个使用READ COMMITTED隔离级别的事务300开始执行：
```sql
# 使用READ COMMITTED隔离级别的事务
BEGIN;
# 场景SELECT1：Transaction 100、200未提交
SELECT * FROM t WHERE id = 1; # 得到的列c的值为'刘备'
```
这个SELECT1的执行过程如下：  
- 在执行SELECT语句时会先生成一个ReadView，ReadView的m_ids列表的内容就是[100, 200]。
- 然后从版本链中挑选可见的记录，从图中可以看出，最新版本的列c的内容是'张飞'，该版本的trx_id值为100，在m_ids列表内，所以不符合可见性要求，根据roll_pointer跳到下一个版本。
- 下一个版本的列c的内容是'关羽'，该版本的trx_id值也为100，也在m_ids列表内，所以也不符合要求，继续跳到下一个版本。
- 下一个版本的列c的内容是'刘备'，该版本的trx_id值为80，小于m_ids列表中最小的事务id100，所以这个版本是符合要求的，最后返回给用户的版本就是这条列c为'刘备'的记录。  

之后，我们把事务id为100的事务提交一下，就像这样：  
```sql
# Transaction 100
BEGIN; 
UPDATE t SET c = '关羽' WHERE id = 1;
UPDATE t SET c = '张飞' WHERE id = 1; 
COMMIT;
```
然后再到事务id为200的事务中更新一下表t中id为1的记录：  
```sql
# Transaction 200
BEGIN;
# 更新了一些别的表的记录
...
UPDATE t SET c = '赵云' WHERE id = 1;
UPDATE t SET c = '诸葛亮' WHERE id = 1;
```
此刻，表t中id为1的记录的版本链就长这样：  
![mysql-MVCC之RC事务链2](https://raw.githubusercontent.com/kangzhihu/images/master/mysql-MVCC%E4%B9%8BRC%E4%BA%8B%E5%8A%A1%E9%93%BE2.png)   
然后再到刚才使用READ COMMITTED隔离级别的事务中继续查找这个id为1的记录，如下：  
```sql
# 使用READ COMMITTED隔离级别的事务
BEGIN;
# 场景SELECT1：Transaction 100、200均未提交
SELECT * FROM t WHERE id = 1; # 得到的列c的值为'刘备'
# 场景SELECT2：Transaction 100提交，Transaction 200未提交
SELECT * FROM t WHERE id = 1; # 得到的列c的值为'张飞'
```
这个SELECT2的执行过程如下：   
+ 在执行SELECT语句时会先生成一个ReadView，ReadView的m_ids列表的内容就是[200]（事务id为100的那个事务已经提交了，所以生成快照时就没有它了）。
+ 然后从版本链中挑选可见的记录，从图中可以看出，最新版本的列c的内容是'诸葛亮'，该版本的trx_id值为200，在m_ids列表内，所以不符合可见性要求，根据roll_pointer跳到下一个版本。
+ 下一个版本的列c的内容是'赵云'，该版本的trx_id值为200，也在m_ids列表内，所以也不符合要求，继续跳到下一个版本。
+ 下一个版本的列c的内容是'张飞'，该版本的trx_id值为100，比m_ids列表中最小的事务id200还要小，所以这个版本是符合要求的，最后返回给用户的版本就是这条列c为'张飞'的记录。

以此类推，如果之后事务id为200的记录也提交了，再此在使用READ COMMITTED隔离级别的事务中查询表t中id值为1的记录时，得到的结果就是'诸葛亮'了，具体流程我们就不分析了。总结一下就是：使用READ COMMITTED隔离级别的事务在每次查询开始时都会生成一个独立的ReadView。  

&emsp;&emsp;在RC场景下，个人理解为了安全性，其实还是要手动强制性直接加行锁：select name from student where id >= 0 for update。所以在事务中，第一句操作入口添加for update作为全局事务操作锁。  

##### REPEATABLE READ MVCC
仍然与前面的一样，假设现在系统里有两个id分别为100、200的事务在执行：
```sql
# Transaction 100
BEGIN;
UPDATE t SET c = '关羽' WHERE id = 1;
UPDATE t SET c = '张飞' WHERE id = 1;
```
```sql
# Transaction 200
BEGIN;
# 更新了一些别的表的记录
...
```
此刻，表t中id为1的记录得到的版本链表如下所示：  
![mysql-MVCC之RC事务链1](https://raw.githubusercontent.com/kangzhihu/images/master/mysql-MVCC%E4%B9%8BRR%E4%BA%8B%E5%8A%A1%E9%93%BE1.png)   
假设现在有一个使用REPEATABLE READ隔离级别的事务开始执行：   
```sql
# 使用REPEATABLE READ隔离级别的事务
BEGIN;
# 场景SELECT1：Transaction 100、200未提交
SELECT * FROM t WHERE id = 1; # 得到的列c的值为'刘备'
```
这个SELECT1的执行过程如下：  
+ 在执行SELECT语句时会先生成一个ReadView，ReadView的m_ids列表的内容就是[100, 200]。
+ 然后从版本链中挑选可见的记录，从图中可以看出，最新版本的列c的内容是'张飞'，该版本的trx_id值为100，在m_ids列表内，所以不符合可见性要求，根据roll_pointer跳到下一个版本。
+ 下一个版本的列c的内容是'关羽'，该版本的trx_id值也为100，也在m_ids列表内，所以也不符合要求，继续跳到下一个版本。
+ 下一个版本的列c的内容是'刘备'，该版本的trx_id值为80，小于m_ids列表中最小的事务id100，所以这个版本是符合要求的，最后返回给用户的版本就是这条列c为'刘备'的记录。

之后，我们把事务id为100的事务提交一下，就像这样：  

```sql
# Transaction 100
BEGIN;
UPDATE t SET c = '关羽' WHERE id = 1;
UPDATE t SET c = '张飞' WHERE id = 1;
COMMIT;
```
然后再到事务id为200的事务中更新一下表t中id为1的记录：  
```sql
# Transaction 200
BEGIN;
# 更新了一些别的表的记录
...
UPDATE t SET c = '赵云' WHERE id = 1;
UPDATE t SET c = '诸葛亮' WHERE id = 1;
```
此刻，表t中id为1的记录的版本链就长这样：  
![mysql-MVCC之RC事务链2](https://raw.githubusercontent.com/kangzhihu/images/master/mysql-MVCC%E4%B9%8BRR%E4%BA%8B%E5%8A%A1%E9%93%BE2.png)   
然后再到刚才使用REPEATABLE READ隔离级别的事务中继续查找这个id为1的记录，如下：  
```sql
# 使用REPEATABLE READ隔离级别的事务
BEGIN;
# 场景SELECT1：Transaction 100、200均未提交
SELECT * FROM t WHERE id = 1; # 得到的列c的值为'刘备'
# 场景SELECT2：Transaction 100提交，Transaction 200未提交
SELECT * FROM t WHERE id = 1; # 得到的列c的值仍为'刘备'
```
这个SELECT2的执行过程如下：  
+ 因为之前已经生成过ReadView了，所以此时直接复用之前的ReadView，之前的ReadView中的m_ids列表就是[100, 200]。
+ 然后从版本链中挑选可见的记录，从图中可以看出，最新版本的列c的内容是'诸葛亮'，该版本的trx_id值为200，在m_ids列表内，所以不符合可见性要求，根据roll_pointer跳到下一个版本。
+ 下一个版本的列c的内容是'赵云'，该版本的trx_id值为200，也在m_ids列表内，所以也不符合要求，继续跳到下一个版本。
+ 下一个版本的列c的内容是'张飞'，该版本的trx_id值为100，而m_ids列表中是包含值为100的事务id的，所以该版本也不符合要求，同理下一个列c的内容是'关羽'的版本也不符合要求。继续跳到下一个版本。
+ 下一个版本的列c的内容是'刘备'，该版本的trx_id值为80，80小于m_ids列表中最小的事务id100，所以这个版本是符合要求的，最后返回给用户的版本就是这条列c为'刘备'的记录。

也就是说两次SELECT查询得到的结果是重复的，记录的列c值都是'刘备'，这就是可重复读的含义。如果我们之后再把事务id为200的记录提交了，之后再到刚才使用REPEATABLE READ隔离级别的事务中继续查找这个id为1的记录，得到的结果还是'刘备'，具体执行过程大家可以自己分析一下。 


### MVCC的使用
&emsp;&emsp;在读多写少的场景下通过MVCC处理并发读，通过乐观锁控制并发写，以提高数据库的读性能。     
- 多版本并发控制（MVCC）是一种用来解决读-写冲突的无锁并发控制，也就是为事务分配单向增长的时间戳，为每个修改保存一个版本，版本与事务时间戳关联，读操作只读该事务开始前的数据库的快照。 这样在读操作不用阻塞写操作，写操作不用阻塞读操作的同时，避免了脏读和不可重复读。 
- 乐观并发控制（OCC）是一种用来解决写-写冲突的无锁并发控制，认为事务间争用没有那么多，所以先进行修改，在提交事务前，检查一下事务开始后，有没有新提交改变，如果没有就提交，如果有就放弃并重试。乐观并发控制类似自选锁。乐观并发控制适用于低数据争用，写冲突比较少的环境。

-----

## 操作日志

##### 操作日志过程 
&emsp;&emsp;一个事务的事务型操作过程归纳为下列步骤：  
1. 索引打标上排它锁(事务型操作)；
2. 将原数据入log buffer进行备份；
3. copy原数据到undo log文件中，方便事务进行构建回滚点；
4. 操作变更业务字段值，DB_TRX_ID，回滚指针到undo log中地址；
5. 在事务操作时，检测到存在排它锁，进入事务挂起等待阶段，等前面的事务提交后才继续执行。<font color="blue">RR下为了防止对同一条数据变更出现可重复读并发变更，需要再变更前使用select ... for update进行排它锁定 -- --> 一锁二查三变更</font>
6. 提交事务，log buffer刷入redo log文件中(或purge线程处理？)。

&emsp;&emsp;此处只通过RR隔离级别下的一个例子讲解undo日志怎么变更。   
```sql
create index test_idx on test(comment);
insert into test values(1, ‘aaa’);
insert into test values(2, ‘bbb’);

-- update primary key
update test set id = 9 where id = 1;   --- 语句1

-- update non-primary key with different value
update test set comment = ‘ccc’ where id = 9; --- 语句2

-- 初始时DB_TRX_ID事务id为1809，当前事务id为1811
```
[示例参考](http://hedengcheng.com/?p=148#_Toc322691905)

![数据主键变更undo示例](https://raw.githubusercontent.com/kangzhihu/images/master/mysql-undo%E5%88%A0%E9%99%A4%E7%A4%BA%E4%BE%8B.png)   
&emsp;&emsp;语句1处：主键的变更对InnoDB来说其实是删除原数据并插入一条新数据，所以在索引上加锁后，原数据删除标志位DELETE_BIT变更为0，并且回滚指针指向了undo log中记录的变更前的原始数据并变更事务版本号。同时，在索引上添加一条记录，该记录回滚指向为空。     
![数据变更undo示例](https://raw.githubusercontent.com/kangzhihu/images/master/mysql-undo%E5%8F%98%E6%9B%B4%E7%A4%BA%E4%BE%8B.png)    
&emsp;&emsp;语句2处：此阶段和前面没有什么不同，只是需要将二级索引进行重构，将comment='ccc'添加进去，同时，在undo long中，回滚的原始数据添加到回滚链中去。  
&emsp;&emsp;有一个点需要注意，<font color="red">undo log记录了所有事务操作数据的回滚点，同一条数据变更(即DB_ROW_ID相同)在同一个(链表)链路中</font>


preDo:   
&emsp;&emsp;将数据库事务commit提交4个阶段：  
1. 清理undo段信息：对于innodb存储引擎的更新操作来说，undo段需要purge，这里的purge主要职能是，真正删除物理记录。在执行delete或update操作时，实际旧记录没有真正删除，只是在记录上打了一个标记，而是在事务提交后，purge线程真正删除，释放物理页空间。因此，提交过程中会将undo信息加入purge列表，供purge线程处理。
2. 释放锁资源：mysql通过锁互斥机制保证不同事务不同时操作一条记录，事务执行后才会真正释放所有锁资源，并唤醒等待其锁资源的其他事务；
3. 刷redo日志：通过redo日志落盘操作，保证了即使修改的数据页即使没有更新到磁盘，只要日志是完成了，就能保证数据库的完整性和一致性；
4. 清理保存点列表：每个语句实际都会有一个savepoint(保存点)，保存点作用是为了可以回滚到事务的任何一个语句执行前的状态，由于事务都已经提交了，所以保存点列表可以被清理了。
