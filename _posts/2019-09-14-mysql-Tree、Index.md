---
layout: post
title: "数据库-树、索引"
subtitle: '树与索引简单总结'
author: "Kang"
date: 2019-09-14 18:48:57
header-img: "img/post-head-img/beauty-2348890_1280.jpg"
catalog: true
tags:
  - 数据库
  - 索引
  - mysql
---
[强烈推荐阅读-MySQL-INNODB形成原理](https://mp.weixin.qq.com/s/dTwvk901VMnX_wAX8VNxcw)
[推荐阅读-浅谈AVL树,红黑树,B树,B+树原理及应用](https://blog.csdn.net/whoamiyang/article/details/51926985)  
[推荐阅读-MySQL索引背后的数据结构及算法原理](http://blog.codinglabs.org/articles/theory-of-mysql-index.html)

### 使用场景概要
&emsp;&emsp;红黑树：是有序的数据结构，可用作数据容器。红黑树多用在内部排序，即全放在内存中的。   
&emsp;&emsp;B树：多用在内存里放不下，大部分数据存储在外存上时，**子节点上**都携带有指向实际数据的指针。    
&emsp;&emsp;B+树：同B树，多用在内存里放不下，大部分数据存储在外存上时，但是**只在叶子节点上**携带有指向实际数据的指针，所以层数比较低。

### 二叉树
&emsp;&emsp;每个节点下只有两个子节点，极端情况下只有一边存在数据而形成一个链表。    

### 满二叉树
&emsp;&emsp;除叶子节点外，每个节点下有且并有两个子节点的树。    

### 完全二叉树
&emsp;&emsp;满二叉树的一个简化版，其要求同层遍历发时，不能存在跳空的节点，即整个树，只能右下角子节点存在为空。    

### AVL树
&emsp;&emsp;在满二叉树上增加平衡性，在添加子节点后，若左右高差超过1，那么会通过旋转来重新达到平衡。   

### 红黑树
具有特性：
- 每个节点非红即黑；  
- 根节点为黑节点；  
- 每个叶子节点都是黑色的(即：叶子节点均为黑色null节点)。--从黑开始，到黑结束；  
- 如果一个节点是红色的，那么其两个子节点都是红色的；  
- 对于任意节点而言，其到叶子点树NULL指针的每条路径都包含相同数目的黑节点；  
- 每条路径都包含相同的黑节点；

### B树
&emsp;&emsp;B树也即B-树，一种平衡的多叉树。<font color="red">B树子节点节点中均携带了一个指针指向具体的数据。</font>  
![B树](https://raw.githubusercontent.com/kangzhihu/images/master/B%E6%A0%91%E5%AD%98%E5%82%A8%E7%A4%BA%E4%BE%8B.png)   
&emsp;&emsp;上图只是一个简单的B树,在实际中B树节点中关键字很多的.上面的图中比如15节点,15代表一个key(索引)，同时也携带了数据记录除key外的数据data(指针).    
&emsp;&emsp;B树的插入，下面以6 10 4 14 5 11 15 3 2 12 1 7 8 8 6 3 6 21 5 15 15 6 32 23 45 65 7 8 6 5 4 为例：  
![B树构建动画](https://raw.githubusercontent.com/kangzhihu/images/master/btreebuild.gif)   
每当插入值后，当前节点数据量超过N个(此例为3)，则节点自动分裂，将分裂节点(中间节点)值移动到插入到父节点中，如此递归直到重新平衡。<font color="green">值也被携带在非叶子节点中</font>。    

#### B树特点
&emsp;&emsp;B树中的每个结点根据实际情况可以包含大量的关键字信息和分支(当然是不能超过磁盘块的大小，根据磁盘驱动(diskdrives)的不同，一般块的大小在1k~4k左右)；这样树的深度降低了，这就意味着查找一个元素只要很少结点从外存磁盘中读入内存，很快访问到要查找的数据。   
&emsp;&emsp;<font color="#8B008B">B树主要是用于存储海量数据的，一般其一个结点就占用磁盘一个块的大小，这样对数据进行磁盘块级别的处理。</font>  

### B+树
&emsp;&emsp;B+树是应<font color="red">文件系统</font>所需而产生的一种B树的变形树（文件的目录一级一级索引，只有最底层的叶子节点（文件）保存数据）非叶子节点只保存索引与下一个节点的地址，不保存实际的数据，数据都保存在叶子节点中。  

&emsp;&emsp;举个文件查找的例子：有3个文件夹a、b、c， a包含b，b包含c，一个文件yang.c，a、b、c就是索引（存储在非叶子节点）， a、b、c只是要找到的yang.c的key，而实际的数据yang.c的地址存储在叶子节点上。   
&emsp;&emsp;路径只是一个具体文件的索引指向，具体的数据内容在文件中。  

![B+树](https://raw.githubusercontent.com/kangzhihu/images/master/B%2B%E6%A0%91%E5%AD%98%E5%82%A8%E7%A4%BA%E4%BE%8B.png)     
&emsp;&emsp;上图中，非叶子节点（比如15 20）只是一个key（实际数据关键码的复制，索引），实际的数据存在叶子节点上才是指向真实数据的指针。  
&emsp;&emsp;仍然以上面的序列6 10 4 14 5 11 15 3 2 12 1 7 8 8 6 3 6 21 5 15 15 6 32 23 45 65 7 8 6 5 4 为例子:  
![B+树构建动画](https://raw.githubusercontent.com/kangzhihu/images/master/B%2Btreebuild.gif)   
同样每当N个(4)值时，当前节点将发生分裂，但是分裂的这个关键码(值)，保留在当前节点和父节点中，这样整个过程下来，最终的值将保留在叶子节点中。B+分裂保留了叶子节点指向兄弟节点的一个指针。      

#### B+数作为索引特点
&emsp;&emsp;1、磁盘效率上：B+树的内部节点<font color="green">不存具体的数据而是指针(但是还是携带索引信息)</font>。通过将<font color="red">所有的索引(key)放在同一盘块中</font>同一盘块能容纳大量的路由数据。这样由于一次性就能将该节点所有的数据加载入内存中(也即加载了更多的待查证关键字)，极大的减少了IO次数。--<font color="#E9967A">很重要的一个指标，也是一个重要的影响因数。</font>  
&emsp;&emsp;2、B+树中主键值和数据行均存在叶子节点上，那么为了得到实际的数据，必须遍历完整个路径到子节点，所有关键字查询的路径长度相同，所以每一个数据的查询效率相当。--<font color="#E9967A">B+树的树高和平均检索长度均大于B树，但是其实在数据量相等时，由1知B+树树高低一些</font>     
&emsp;&emsp;3、所有记录都集中在叶子节点一层，并且叶子节点可以构成一维线性表，便于连续访问和范围查询。--<font color="#E9967A">B+树的叶子结点本身依关键字的大小自小而大的顺序链接(在B+Tree的每个叶子结点增加一个指向相邻叶子结点的指针)，B树就不行了，其每个子节点均携带了具体的数据指针，在查找单数据时效率比较高。</font> <font color="#FF00FF">顺序链接的特点决定了B+树天然在范围查找上具有优势</font>  


### B树与B+树补充点
&emsp;&emsp;B树和B+树均可以简单认为一个节点需要做一次IO加载到内存中。  
#### 两者明显的区别
- 1、B+树中只有叶子节点会带有指向记录的指针（ROWID），而B树则所有节点都带有，在内部节点出现的索引项不会再出现在叶子节点中。   
- 2、B+树中所有叶子节点都是通过指针连接在一起，而B树不会。  

### 两者各自的优点： 
##### B+树的优点
- 1、非叶子节点不会带上ROWID，这样，一个块中可以容纳更多的索引项，一是可以降低树的高度，二是一个内部节点可以定位更多的叶子节点。    
- 2、<font color="red">叶子节点之间通过指针来连接，范围扫描将十分简单，只需要顺序查找定位叶子节点上的索引范围即可</font>。而对于B树来说，则需要在叶子节点和内部节点不停的往返移动。   
###### B树的优点
对于在内部节点的数据，可直接得到，不必根据叶子节点来定位。  

### 索引的查找过程
一个查找表结构示例：  
| id  | name | company   |    
|------|:---:|---------:|    
| 5  | Gates | Microsoft |    
| 7  | Bezos  | Amazon  |    
| 11  | Jobs   | Apple |    
| 14  | Ellison | Oracle |       
![聚簇索引与非聚簇索引](https://raw.githubusercontent.com/kangzhihu/images/master/%E8%81%9A%E7%B0%87%E7%B4%A2%E5%BC%95%E4%B8%8E%E9%9D%9E%E8%81%9A%E7%B0%87%E7%B4%A2%E5%BC%95.png)
<p align="center">图1</p>
  
&emsp;&emsp;我们来看一个联合索引的示例:
![联合索引示例](https://raw.githubusercontent.com/kangzhihu/images/master/B%2B%E6%A0%91%E7%BB%84%E5%90%88%E7%B4%A2%E5%BC%95%E7%A4%BA%E4%BE%8B.png)   
对于联合索引，可以认为第一列作为全局查找值，当值相同时，在相同节点上根据后面的列做排序处理。看一下上面的最后一个联合索引值，两个人的姓和名都相同，则按照生日来排序。

#### 聚簇索引与非聚簇索引区别   
&emsp;&emsp;聚簇索引与非聚簇索引是对B+树，也即InnoDB来说的。对于一般的索引，默认key为索引值，data为一个指向实际数据的指针，下面来详细看下：    
&emsp;&emsp;从图1可以看出两者明显的区别:       
1. 聚簇索引中数据文件本身就是索引文件,其叶子节点存储了主键(key)和具体的数据行(value)，而<font color="red">不是指向数据的一个指针</font>。-- 这就要求<font color="red">InnoDB表必须有主键（MyISAM可以没有）</font>，如果没有显式指定，则MySQL系统会自动选择一个可以唯一标识数据记录的列作为主键，如果不存在这种列，则MySQL自动为InnoDB表生成一个隐含字段作为主键，这个字段长度为6个字节，类型为长整形。   
2. 非聚簇索引索引文件和数据文件是分离的，索引树的叶子节点仅保存一个指向具体数据记录的地址.--MyISAM只能缓存索引。       

#### 聚簇索引查找
&emsp;&emsp;根据上面分析的InnoDB使用的结构可知，该结构下主键在一棵B+树中，而行数据就储存在叶子节点上，
- 主键查找：在主键查找时("where id = 14")，则按照B+树的检索算法即可查找到对应的叶子节点，之后获得行数据。  
- 辅助查找：若对Name列进行条件搜索，则需要两个步骤：
  + 第一步：在辅助索引B+树中检索Name，到达其叶子节点获取对应的主键(<font color="red">注意：叶子节点此时存储的直接是主键值</font>)。
  + 第二步：使用主键在主索引B+树种再执行一次B+树检索操作，最终到达叶子节点，通过子节点上数据指针即可将数据块载入从而获取整行数据。    
  -- <font color="red">辅助索引叶子节点存的为主键值。</font>  

### 索引覆盖、回表、下推
- 覆盖：查找返回的结果列，全部在索引树上，这样就不用去回表加载数据而直接从内存树中读取。  
  + 非主键索引即为二级索引，二级索引不能满足所有列在索引树上，则需要回表，二级索引在叶子节点才包含主键信息。 
- 回表：索引上查找返回的索引结果列，部分需要的字段不存在，需要拿着主键到聚簇索引上在查询一次。
- 下推：在执行查询语句时，MySQL优化器将WHERE条件中可下推到存储引擎层的部分下推到存储引擎进行过滤，只将满足条件的数据返回给MySQL服务器，从而减少MySQL服务器的工作量，提高查询效率。通常情况下，只有使用索引的查询才能利用索引下推功能。  
  + MySQL中存储引擎层负责实际的数据存储和检索工作，而服务器则负责处理SQL语句、解析查询语句、优化查询计划等任务。
  + 说白了就是之前根据一个索引获取到部分数据后，会将结果返回到服务层后进行过滤，而现在把其它过滤条件给下推到存储引擎层直接过滤了，不需要回表。
  > &emsp;&emsp;存在查询中命中使用了辅助索引的查询才会使用索引下推，在这个前提下，非辅助索引中的列也可被下推，目前理解有下推动过就是应为辅助索引只是按照最左原因被使用了部分。  
  > &emsp;&emsp;比如：index_abc(a,b,c)中，查询使用a like '%张' 导致b和c没使用到，那么就可能触发下推。
  > 
  > 遇到范围查询(大于、小于、between、like)就会停止后续匹配，在 执行计划中的extra 字段上面 有一个 **「Using index condition」** 这个就是 索引下推啦。
  
#### 非聚簇索引查找
&emsp;&emsp;MyISM使用的结构，<font color="red">非聚簇索引下，两个的索引树完全是独立的，且叶子节点存储的也是实际数据的地址，通过辅助键检索无需访问主键的索引树。</font> 这也导致了必须有两次IO，先查找到数据所在的叶子节点，然后在IO到具体的数据行。   

#### InnoDB和MyISAM怎么选择
1. InnoDB具有事务，支持4个事务隔离级别，回滚，崩溃修复能力和多版本并发的事务安全；当应用中需要执行大量的INSERT或UPDATE操作时，则应该使用InnoDB，这样可以提高多用户并发操作的性能。    
2. MyISAM管理非事务表。它提供高速存储和检索(因为索引不包含数据，数据单独文件存储？)，以及全文搜索能力。如果应用中需要执行大量的SELECT查询，那么MyISAM是更好的选择。
3. InnoDB支持行级锁而MyISAM只支持到表级锁定。   
4. InnoDB 读写阻塞与事务隔离级别相关,而MyISAM读写互相阻塞：不仅会在写入的时候阻塞读取，MyISAM还会在读取的时候阻塞写入，但读本身并不会阻塞另外的读。   
5. InnoDB 要缓存数据和索引，MyISAM只缓存索引块，这就导致InnoDB要求的内存比较多。  
6. select count(*)和order by这种操作Innodb其实也是会锁表的，很多人以为Innodb是行级锁，那个只是where对它主键是有效，非主键的都会锁全表的。   


### 优化建议

### mysql在何时放弃索引而使用全表扫描
&emsp;&emsp;sql查询优化器认为全表扫描效率比索引扫描高，就会使用全表扫描。
&emsp;&emsp;-- null查询会导致全表扫描。  

#### 什么字段都可以建索引吗？
&emsp;&emsp;索引选择性=基数/数据行，在实际应用中，索引选择性应尽可能的接近1。    
&emsp;&emsp;执行 show index from tableName 命令可以看到当前存在索引[基数]的预估值Cardinality（散列程度），它会估计索引中不重复记录，如果这个相对值很小，可能就要评估索引是否有意义。

#### 正向范围查找对索引的使用？
&emsp;&emsp;in能够使用索引，若索引为组合索引，则也按照最左原则使用索引   
&emsp;&emsp;<，<=，=，>，>=，BETWEEN，IN 也是正向范围查找，可以使用到索引，但是范围列后面的列无法用到索引。同时，索引最多用于一个范围列，因此如果查询条件中有两个范围列则无法全用到索引。   

#### 负向查找能使用索引么？
&emsp;&emsp;负向查询（not, not in, not like, <>, != ,!>,!< ,or ） 不会使用索引，这些负向的范围查找，都<font color="red">具有范围不确定性</font>。    

#### like查找对索引的使用？  
&emsp;&emsp;以%开头的LIKE查询不能够利用B-tree索引，但是若使用'XXX%'则可以使用到索引; 

#### order by与GROUP BY使用索引
- 覆盖索引：如果一个索引包含了查询中的所有列，并且 ORDER BY 子句中的列也包含在这个索引中，那么这个查询可以使用覆盖索引。
- 单列索引：如果查询中的 ORDER BY 子句只涉及一个列，并且存在一个单列索引与该列匹配，那么数据库通常可以使用这个索引来加速排序。
- 多列索引：如果查询中的 ORDER BY 子句涉及多个列，并且存在一个多列索引与这些列匹配，并且 ORDER BY 子句中的列按照索引的顺序出现，那么数据库通常可以使用这个多列索引来加速排序。
- 排序顺序：考虑排序的顺序，如果 ORDER BY 子句要求升序排序（ASC），而索引是升序的，或者要求降序排序（DESC），而索引是降序的，那么数据库可能会更容易利用索引来执行排序操作。
> GROUP BY基本上除了不排序，基本上一样

#### 善于使用覆盖索引 
&emsp;&emsp;最少列返回查询列，同时记住order by下的列其实也是要参与索引优化的。

#### offset与limit优化案例
常见的查询方式：
```sql
select * from member where gender=1 limit 300000,1;
```
&emsp;&emsp;上面语句在limit比较小的情况下比较快，但是当其值比较大时，比如上面偏移30000时，需要查询后丢弃的数量比较多，此时上面语句性能较差。      
思考：    
&emsp;&emsp;上面的查询，假设gender为辅助键索引。其查询过程为优先查找辅助索引（找出所有gender=1的id)，然后再通过辅助索引中获取的主键ID在主键索引树种查找并加载数据块，最后，将前30000条数据丢弃。  
&emsp;&emsp;整个过程比较明确，可以发现，整个查询前半部分都是进行数据定位操作，没有必要发生具体业务数据的加载，所以一个比较明确的优化方式是避免将前面需要偏移掉的数据块加载到内存中。所以采用以下的优化方式：
```sql
select * from member where id in(
seleect id from member where gender=1 limit 300000,1;
)
```
上面的语句，其实际在查询时将<font color="red">被处理为exists</font>处理方式：
```sql
select * from member a where exists(select * from member b where b.id=a.id );
```
&emsp;&emsp;而exists相关子查询的执行原理是: 循环取出a表的每一条记录与b表进行比较，比较的条件是a.id=b.id，看a表的每条记录的id是否在b表存在，如果存在就行返回a表的这条记录。exists查询它的执行计划只能一条条拿着a表的数据到b表查（外表到里表中），虽然在b表的id字段存在索引可以提高查询效率。
```sql
select * from member a inner join member b on a.id=b.id; 
```
&emsp;&emsp;为啥选择inner join？使用 inner join时，优化器能自动选择哪种a链接b还是b链接a，这样从整体性能上比使用left join或者right join 要高。

### InnoDB 记录行与数据页
[参考阅读-InnoDB 记录行与数据页](https://zhuanlan.zhihu.com/p/639042049)  
&emsp;&emsp;在mysql中，叶子节点的存储是逻辑连续但是物理上不连续的，当叶子节点(一个叶子节点=一页)满时，新插入节点会导致页分裂，旧节点一半的数据会挪动到新页上去；  
为了加快减少IO和加快查找，会  
1、把经常访问的数据页和索引页缓存在内存里，减少磁盘 I/O，提高数据库性能。  
2、采用预读机制，顺序查时提前多读几个页，批量放进内存，避免频繁随机 IO；  