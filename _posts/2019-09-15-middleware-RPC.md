---
layout: post
title: "中间件-RPC"
subtitle: '中间件之RPC整理总结一'
author: "Kang"
date: 2019-09-15 17:38:22
header-img: "img/post-head-img/landscape-4426419_1280.jpg"
catalog: true
tags:
  - 中间件
  - RPC
---
![一个RPC简单调用过程示例](https://raw.githubusercontent.com/kangzhihu/images/master/rpc%E8%B0%83%E7%94%A8%E7%A4%BA%E4%BE%8B.png)  
&emsp;&emsp;Consumer端使用动态代理模式，代理远程对象，让像调用本地对象一样。  
&emsp;&emsp;Producr端使用java反射，依据传递过来的信息反射调用原对象服务。  
```java
// 请求参数封装协议：
String interfaceClassName;//接口名
String methodName;//方法名
Class<?>[] params;//参数类型
Object[] values;//参数值
```
