---
layout: post
title: "分布式-2pc示例简介"
subtitle: '2p示例简介'
author: "Kang"
date: 2019-09-19 10:31:29
header-img: "img/post-head-img/thumb-1920-418965.png"
catalog: true
tags:
  - 分布式
  - 事务
---
前面[分布式-2pc、3pc、TCC](https://kangzhihu.github.io/2019/09/12/%E5%88%86%E5%B8%83%E5%BC%8F-2pc-3pc-TCC/)已经介绍分布式相关的事务。本文接上文，提供一个可简单学习的2PC示例。  

### 1. 基于XAResource底层接口实现
```xml
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>8.0.17</version>
</dependency>

<dependency>
    <groupId>javax.transaction</groupId>
    <artifactId>javax.transaction-api</artifactId>
    <version>1.3</version>
</dependency>
```
  
```java
public class XATest {

    public static void main(String[] args) throws SQLException {
        Connection conn1 = DriverManager.getConnection("jdbc:mysql://localhost:3306/db_user", "root", "root");
        Connection conn2 = DriverManager.getConnection("jdbc:mysql://localhost:3306/db_order", "root", "root");

        //获取资源管理器接口
        XAConnection xaConnection1 = new MysqlXAConnection((JdbcConnection) conn1,true);
        XAResource xaResource1 = xaConnection1.getXAResource();
        XAConnection xaConnection2 = new MysqlXAConnection((JdbcConnection) conn2,true);
        XAResource xaResource2 = xaConnection2.getXAResource();
        //全局事务id，此处模拟直接写死一个
        byte[] gtrid = "t1234".getBytes();
        int formatId = 1;
        try {
            //事务分支上的分支id
            byte[] bqual1 = "cid1234".getBytes();
            Xid xid1 = new MysqlXid(gtrid,bqual1,formatId);
            //执行事务分支
            xaResource1.start(xid1,XAResource.TMNOFLAGS); //不选择任何标志值。
            PreparedStatement ps1 = conn1.prepareStatement("insert into user(name,sex) values('zhangsan','man')");
            ps1.execute();
            xaResource1.end(xid1,XAResource.TMSUCCESS); //取消调用者与事务分支的关联。

            byte[] bqual2 = "cid12345".getBytes();
            Xid xid2 = new MysqlXid(gtrid,bqual1,formatId);
            //执行事务分支
            xaResource2.start(xid2,XAResource.TMNOFLAGS);
            PreparedStatement ps2 = conn2.prepareStatement("insert into order(userid,objName) values(1,'吸尘器')");
            ps2.execute();
            xaResource2.end(xid2,XAResource.TMSUCCESS);

            //###################两阶段提交#####################
            //第一阶段：询问所有的事务分支，准备提交事务-->开启事务并执行
            int prepare1 = xaResource1.prepare(xid1);
            int prepare2 = xaResource2.prepare(xid2);

            //第二阶段：提交所有的事务分支
            boolean onePhase = false; //是否优化为一阶段提交-->有两个事务分支，所以不能优化
            if(prepare1 == XAResource.XA_OK && prepare2 == XAResource.XA_OK){
                xaResource1.commit(xid1,onePhase);
                xaResource2.commit(xid2,onePhase);
            }else{ //事务没有提交，则回滚
                xaResource1.rollback(xid1);
                xaResource2.rollback(xid2);
            }

        }catch (XAException e) {
            //出现任何异常，也需要回滚
        }


    }

}
```

### 2. 基于atomikos框架实现
该实现可参考一些网上的实现，此处暂不介绍。 其全局使用了一把大锁，锁内部进行两阶段提交。  
