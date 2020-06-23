---
layout: post
title:  "MySQL索引学习"
date:   2020-06-17 20:43:00
categories: mysql
tags: mysql 唯一索引 索引失效
---

* content
{:toc}


最近，在项目中要对一个mysql表建立唯一索引，但是发生了一系列自己想象之外的现象，因此趁机巩固下mysql索引相关的知识。




## 1. 唯一性约束不起作用

我对我的表建立了一个唯一索引，包含几列：  
```sql
CREATE UNIQUE INDEX UK_IDX_TEST_A_B_C ON testTable (A, B, C);
```
这里A、B、C都有可能为NULL，但是不会全部为NULL。然后我往里面插入数据
```sql
Insert into testTable (A,B,C) VALUES ("test", "test"); --- OK
Insert into testTable (A,B,C) VALUES ("test", "test"); --- OK
```
<font color="red">上面的两条SQL都是执行成功了！</font> 数据库中会有多条一样的记录。可按照我之的构想，在执行后面一条SQL时应该抛出 ‘Duplicate key’ 异常的。   
后来查了一下，才发现MySQL唯一性索引是允许多个 NULL 值的存在的：
```
A UNIQUE index creates a constraint such that all values in the index must be distinct. An error occurs if you try to add a new row with a key value that matches an existing row. For all engines, a UNIQUE index allows multiple NULL values for columns that can contain NULL.
```
MySQL官网上也有一个report讨论这个问题：[unique index allows duplicates with null values](https://bugs.mysql.com/bug.php?id=8173) ，一部分人认为这是MySQL的bug， 另一部分则认为是一个feature（NULL != NULL，NULL只是表示data的空缺）。这是一个仁者见仁，智者见智的问题。  
既然MySQL认为存在多个NULL值是合理的，那么怎么解决这个问题呢？我的方案是把A、B、C的默认值设置为空串，然后还是执行上面的SQL：
```sql
Insert into testTable (A,B,C) VALUES ("test", "test"); --- OK
Insert into testTable (A,B,C) VALUES ("test", "test"); --- Duplicate key
```
插入第二条语句时报错，就解决这个问题了。


## 2. MySQL explain详解

解决完上面的唯一性约束后


## 参考文献

1. [MySQL 唯一性约束与 NULL](https://yemengying.com/2017/05/18/mysql-unique-key-null/)  
2. [MySQL EXPLAIN详解](https://www.jianshu.com/p/ea3fc71fdc45)
3. [mysql为什么有些时候会选错索引](https://www.cnblogs.com/wangchunli-blogs/p/10417454.html)






