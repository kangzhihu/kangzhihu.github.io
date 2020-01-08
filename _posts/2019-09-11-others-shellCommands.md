---
layout: post
title: "其它-常用命令集"
subtitle: '常用命令'
author: "Kang"
date: 2019-09-11 11:19:19
header-img: "img/post-head-img/baby-900.jpg"
catalog: true
tags:
  - 其它
---
## 数据库
### binlog解析
1. 二进制binlog转换为阅读文本文件：
```shell
mysqlbinlog --base64-output=DECODE-ROWS  -vv -d 二进制文件 > XXX.txt
```



## Linux

字符串内容查找截取:

```
grep '清洗异常' online.base.traderplat.service-2020-01-08-0.log | awk '{a=index($0,"订单");b=index($0,"清洗异常");print substr($0,a+2,b-a-2)}'
```

