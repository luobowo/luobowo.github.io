---
layout: post
title:  "MySQL索引学习"
date:   2020-06-17 20:43:00
categories: mysql
tags: mysql 唯一索引 索引失效
---

* content
{:toc}


最近，在项目中要对一个mysql表建立唯一索引，但是发生了一系列自己意料之外的现象，因此趁机巩固下mysql索引相关的知识。




## 1. 唯一性约束不起作用

首先，本文中所用的mysql版本是5.7.14：

```sql
select version();
```
得到的结果：

```
5.7.14.6.3-20171229-log
```

我对我的表建立了一个唯一索引，包含几列： 

```sql
CREATE UNIQUE INDEX UK_IDX_TEST_A_B_C ON testTable (A, B, C);
```
这里A、B、C都有可能为NULL，但是不会全部为NULL（由业务语义保证）。然后我往里面插入数据

```sql
Insert into testTable (A,B) VALUES ("test", "test"); --- OK
Insert into testTable (A,B) VALUES ("test", "test"); --- OK
```
<font color="red">上面的两条SQL都是执行成功了！</font> 数据库中会有多条一样的记录。可按照我之的构想，在执行后面一条SQL时应该抛出 ‘Duplicate key’ 异常的。   
后来查了一下，才发现MySQL唯一性索引是允许多个 NULL 值的存在的：

```
A UNIQUE index creates a constraint such that all values in the index must be distinct. An error occurs if you try to add a new row with a key value that matches an existing row. For all engines, a UNIQUE index allows multiple NULL values for columns that can contain NULL.
```
MySQL官网上也有一个report讨论这个问题：[unique index allows duplicates with null values](https://bugs.mysql.com/bug.php?id=8173) ，一部分人认为这是MySQL的bug， 另一部分则认为是一个feature（NULL != NULL，NULL只是表示data的空缺）。这是一个仁者见仁，智者见智的问题。  
既然MySQL认为存在多个NULL值是合理的，那么怎么解决这个问题呢？我的方案是把A、B、C的默认值设置为空串，然后还是执行上面的SQL：

```sql
Insert into testTable (A,B) VALUES ("test", "test"); --- OK
Insert into testTable (A,B) VALUES ("test", "test"); --- Duplicate key
```
插入第二条语句时报错，就解决这个问题了。


## 2. MySQL explain详解

解决完上面的唯一性约束后，我想看看我的查询有没有使用到索引，因为我是对A B C 三列建了一个唯一索引，因此，我猜想下面这样的一条语句会使用到我的唯一索引：

```sql
explain select * from testTable where A="test"
```

但是却得到一条类似下面这样的结果：

|  id  |select_type|  table  |  partitions  |  type  |  possible_keys  |  key  |  key_len  |  ref  |  rows  |  filtered  |  Extra  |
|:----:|:----:|:----:|:----:|:----:|:----:|:----:|:----:|:----:|:----:|:----:|:----:|
|  1   |  SIMPLE | testTable| NULL  | ALL | UK_IDX_TEST_A_B_C| NULL |  NULL  | NULL | 1003  | 100.0 |Using where|

下面先对explain出来的字段进行解释。注意**explain的执行计划，只是作为语句执行过程的一个参考，实际执行的过程不一定和计划完全一致**，但是执行计划中透露出的讯息却可以帮助选择更好的索引和写出更优化的查询语句。

### 2.1 id
执行编号，标识select所属的行。如果在语句中没子查询或关联查询，只有唯一的select，每行都将显示1。否则，内层的select语句一般会顺序编号，对应于其在原始语句中的位置；SQL会从id大到小的执行, 如果id相同，则执行顺序由上至下。例如执行下面这样一条语句：

```sql
explain select * from testTable
where id in 
(
    select id from testTable where A="testA"
    UNION 
    select id from testTable where A="testB"
);
```

得到如下的结果：

|  id  |select_type|  table  |  partitions  |  type  |  possible_keys  |  key  |  key_len  |  ref  |  rows  |  filtered  |  Extra  |
|:----:|:----:|:----:|:----:|:----:|:----:|:----:|:----:|:----:|:----:|:----:|:----:|
|  1   |  PRIMARY | testTable| NULL  | ALL | NULL| NULL |  NULL  | NULL | 1003  | 100.0 |Using where|
|  2   |  DEPENDENT SUBQUERY | testTable| NULL  | eq_ref | PRIMARY| PRIMARY |  8  | func | 1  | 10.0 |Using where|
|  3   |  DEPENDENT UNION | testTable| NULL  | eq_ref | PRIMARY| PRIMARY |  8  | func | 1  | 10.0 |Using where|
|NULL  |  UNION RESULT | <union2,3>| NULL  | ALL | NULL| NULL |  NULL  | NULL | NULL  | NULL |Using temporary|


### 2.2 select_type

显示示查询中每个select子句的类型。   
(1) SIMPLE(简单SELECT,不使用UNION或子查询等)   
(2) PRIMARY(查询中若包含任何复杂的子部分,最外层的select被标记为PRIMARY)  
(3) UNION(UNION中的第二个或后面的SELECT语句)  
(4) DEPENDENT UNION(UNION中的第二个或后面的SELECT语句，取决于外面的查询)  
(5) UNION RESULT(UNION的结果)  
(6) SUBQUERY(子查询中的第一个SELECT)  
(7) DEPENDENT SUBQUERY(子查询中的第一个SELECT，取决于外面的查询)  
(8) DERIVED(派生表的SELECT, FROM子句的子查询)  
(9) UNCACHEABLE SUBQUERY(一个子查询的结果不能被缓存，必须重新评估外链接的第一行)  

### 2.3 table
显示这一行的数据是关于哪张表的；当from中有子查询的时候，表名是derivedN的形式，N指向子查询，也就是explain结果中的下一列；当有union result的时候，表名是union 1,2等的形式，1,2表示参与union的query id。

### 2.4 paritions
该列显示的为分区表命中的分区情况。非分区表该字段为空（null）。

### 2.5 type
表示MySQL在表中找到所需行的方式，又称“访问类型”。  
常用的类型有： ALL, index,  range, ref, eq_ref, const, system, NULL（从左到右，性能从差到好）  
**ALL**：Full Table Scan， MySQL将遍历全表以找到匹配的行  
**index**: Full Index Scan，和全表扫描一样。只是扫描表的时候按照索引次序进行而不是行。主要优点就是避免了排序, 但是开销仍然非常大。  
**range**:范围扫描，一个有限制的索引扫描。key 列显示使用了哪个索引。当使用=、 <>、>、>=、<、<=、IS NULL、<=>、BETWEEN 或者 IN 操作符,用常量比较关键字列时,可以使用 range      
**ref**: 一种索引访问，它返回所有匹配某个单个值的行。此类索引访问只有当使用非唯一性索引或唯一性索引非唯一性前缀时才会发生。这个类型跟eq_ref不同的是，它用在关联操作只使用了索引的最左前缀，或者索引不是UNIQUE和PRIMARY KEY。ref可以用于使用=或<=>操作符的带索引的列  
**eq_ref**: 类似ref，区别就在使用的索引是唯一索引，对于每个索引键值，表中只有一条记录匹配，简单来说，就是多表连接中使用primary key或者 unique key作为关联条件  
**const、system**: 当确定最多只会有一行匹配的时候，MySQL优化器会在查询前读取它而且只读取一次，因此非常快。当主键放入where子句时，mysql把这个查询转为一个常量（高效）；system是const类型的特例，当查询的表只有一行的情况下，使用system  
**NULL**: 意味说mysql能在优化阶段分解查询语句，在执行阶段甚至用不到访问表或索引（高效）  

### 2.6 possible_keys
指出MySQL能使用哪个索引在表中找到记录，查询涉及到的字段上若存在索引，则该索引将被列出，**但不一定被查询使用**.  
该列完全独立于EXPLAIN输出所示的表的次序。这意味着在possible_keys中的某些键实际上不能按生成的表次序使用。   
如果该列是NULL，则没有相关的索引。在这种情况下，可以通过检查WHERE子句看是否它引用某些列或适合索引的列来提高你的查询性能。如果是这样，创造一个适当的索引并且再次用EXPLAIN检查查询。

### 2.7 Key
key列显示MySQL实际决定使用的键（索引）  
如果没有选择索引，键是NULL。要想强制MySQL使用或忽视possible_keys列中的索引，在查询中使用FORCE INDEX、USE INDEX或者IGNORE INDEX。  

### 2.8 key_len
表示索引中使用的字节数，可通过该列计算查询中使用的索引的长度（key_len显示的值为索引字段的最大可能长度，并非实际使用长度，即key_len是根据表定义计算而得，不是通过表内检索出的）  
不损失精确性的情况下，长度越短越好。  

### 2.9 ref
如果是使用的常数等值查询，这里会显示const，如果是连接查询，被驱动表的执行计划这里会显示驱动表的关联字段，如果是条件使用了表达式或者函数，或者条件列发生了内部隐式转换，这里可能显示为func。

### 2.10 rows
 表示MySQL根据表统计信息及索引选用情况，估算的找到所需的记录所需要读取的行数

### 2.11 filtered
这个字段表示存储引擎返回的数据在server层过滤后，剩下多少满足查询的记录数量的比例，注意是百分比，不是具体记录数。

### 2.12 Extra
Extra是EXPLAIN输出中另外一个很重要的列，该列显示MySQL在查询过程中的一些详细信息，MySQL查询优化器执行查询的过程中对查询计划的重要补充信息，**如果你想要优化你的查询，那就要注意extra辅助信息中的using filesort和using temporary，这两项非常消耗性能，需要注意。**      
**Using where**: 使用了WHERE从句来限制哪些行将与下一张表匹配或者是返回给用户。注意：Extra列出现Using where表示MySQL服务器将存储引擎返回服务层以后再应用WHERE条件过滤。   
**Using temporary**：用临时表保存中间结果，常用于GROUP BY 和 ORDER BY操作中，一般看到它说明查询需要优化了，就算避免不了临时表的使用也要尽量避免硬盘临时表的使用。    
**Using filesort**：MySQL有两种方式可以生成有序的结果，通过排序操作或者使用索引，当Extra中出现了Using filesort 说明MySQL使用了排序操作，但注意虽然叫filesort但并不是说明就是用了文件来进行排序，只要可能排序都是在内存里完成的。大部分情况下利用索引排序更快，所以一般这时也要考虑优化查询了。使用文件完成排序操作，这是可能是ordery by，group by语句的结果，这可能是一个CPU密集型的过程，可以通过选择合适的索引来改进性能，用索引来为查询结果排序。  
**Using join buffer**：该值强调了在获取连接条件时没有使用索引，并且需要连接缓冲区来存储中间结果。如果出现了这个值，那应该注意，根据查询的具体情况可能需要添加索引来改进能。  
**Impossible where**：这个值强调了where语句会导致没有符合条件的行。  
**Select tables optimized away**：这个值意味着仅通过使用索引，优化器可能仅从聚合函数结果中返回一行  


## 3. 为什么没有使用索引
回到第二节一开始提到的问题，以下的sql：

```sql
select * from testTable where A="test"
```
使用的执行计划是这样的：

|  id  |select_type|  table  |  partitions  |  type  |  possible_keys  |  key  |  key_len  |  ref  |  rows  |  filtered  |  Extra  |
|:----:|:----:|:----:|:----:|:----:|:----:|:----:|:----:|:----:|:----:|:----:|:----:|
|  1   |  SIMPLE | testTable| NULL  | ALL | UK_IDX_TEST_A_B_C| NULL |  NULL  | NULL | 1003  | 100.0 |Using where|

key那列是NULL，也就是没有使用到索引，为什么呢？

这是因为选择索引是优化器的工作。**而优化器选择索引的目的，是找到一个最优的执行方案，并用最小的代价去执行语句。在数据库里面，扫描行数是影响执行代价的因素之一。扫描的行数越少，意味着访问磁盘数据的次数越少，消耗的CPU资源越少。**

**优化器是怎么判断扫描的行数呢? 答案是使用一个统计信息就是索引的“区分度”。显然，一个索引上不同的值越多，这个索引的区分度就越好。而一个索引上不同的值的个数，我们称之为“基数”（cardinality）。也就是说，这个基数越大，索引的区分度越好。** 我们可以使用`show index from table`方法，看到一个索引的基数。  

MySQL是使用采样统计得到索引的基数的。采样统计的时候，InnoDB默认会选择N个数据页，统计这些页面上的不同值，得到一个平均值，然后乘以这个索引的页面数，就得到了这个索引的基数。而数据表是会持续更新的，索引统计信息也不会固定不变。所以，当变更的数据行数超过1/M的时候，会自动触发重新做一次索引统计。  

如果统计信息不对，那就修正。使用`analyze table t` 命令，可以用来重新统计索引信息。  

<font color=red>当然，扫描行数并不是唯一的判断标准，优化器还会结合是否使用临时表、是否排序等因素进行综合判断。</font>**使用普通索引的时候还需要把回表的代价算进去**

在上面的例子中，因为使用索引也要扫描所有的行，而且使用了索引还需要回表，所以mysql就没有使用索引了。

tips： 什么是回表？  
```
MySQL B+树索引结构中，主键索引的叶子节点存的是整行数据。非主键索引的叶子节点内容是主键的值。
基于主键索引和普通索引的查询有什么区别？如果语句是 select * from T where id = 6，即主键查询方式，则只需要搜索 ID 这棵 B+ 树；如果语句是 select * from T where name = '张四'，即普通索引查询方式，则需要先搜索 k 索引树，得到 ID 的值为 500，再到 ID 索引树搜索一次。这个过程称为回表。
```



## 参考文献

1. [MySQL 唯一性约束与 NULL](https://yemengying.com/2017/05/18/mysql-unique-key-null/)  
2. [MySQL EXPLAIN详解](https://www.jianshu.com/p/ea3fc71fdc45)
3. [MySQL Explain详解](https://www.cnblogs.com/xuanzhi201111/p/4175635.html)  
4. [mysql为什么有些时候会选错索引](https://www.cnblogs.com/wangchunli-blogs/p/10417454.html)
5. [Mysql优化之explain详解](https://www.jianshu.com/p/73f2c8448722)  






