---
layout: post
title: "网络通信-一个简单的负载通讯"
subtitle: '一个简单的负载通讯'
author: "Kang"
date: 2019-09-15 17:27:52
header-img: "img/post-head-img/girl-4440487_1280.jpg"
catalog: true
tags:
  - 网络通讯
---
![一次请求全局通讯过程](https://raw.githubusercontent.com/kangzhihu/images/master/%E4%B8%80%E6%AC%A1%E8%AF%B7%E6%B1%82%E5%85%A8%E5%B1%80%E8%BF%87%E7%A8%8B.png)  
外部解析到的是LVS/F5的全局服务地址，是LVS/F5再将请求转发给Nginx，Nginx转发给后端的真实服务器地址。  

#### 为什么要引入LVS/F5？
&emsp;&emsp;Nginx其实能处理的吞吐量是有限的，为了提高吞吐量，引入了LVD或F5。   
