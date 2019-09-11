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
