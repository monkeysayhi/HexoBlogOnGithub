---
title: MySQL索引  
tags: [MySQL,索引,数据库]  
reward: true  
---

数据库编程几乎是每个程序员必须掌握的一项技能。  
仅管我们不需要像DBA一样“360度花样玩转数据库”，但了解甚至熟知数据库的存储和索引原理仍然对关系型数据库的最常见的优化手段。本文以MySQL为例，以索引为中心，涉及了MySQL基本的存储原理；讨论主要基于InnoDB和MyISAM存储引擎。

<!--more-->

## 索引
### 索引的原理
#### 基本原理

>参考：  
[MySQL索引原理及慢查询优化](http://tech.meituan.com/mysql-index.html)  

#### 为什么是B-Tree，不是红黑树

>参考：  
[为什么要用B+树结构——MySQL索引结构的实现](http://database.51cto.com/art/201504/473322_all.htm)  

### 利用索引进行优化

>参考：  
[MySQL索引原理及慢查询优化](http://tech.meituan.com/mysql-index.html)  

### InnoDB 与 MyISAM 结构上的区别

>参考：  
[为什么要用B+树结构——MySQL索引结构的实现](http://database.51cto.com/art/201504/473322_all.htm)  
[2014阿里实习生面试题——mysql如何实现索引的](http://m.2cto.com/database/201404/295109.html)  
