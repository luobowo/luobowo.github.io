---
layout: post
title:  "数据库基础知识复习"
date:   2015-08-25 10:52:19
categories: 数据库
tags: sql
---


由于很久没有用数据库相关的知识，突然用起来好多东西还是有点记忆模糊，就复习了下SQL的基础知识，看的书是日本人MICK写的《SQL基础教程》，该书非常简单，适合对数据库完全不知道的入门教程，里面甚至没有范式以及索引的介绍。下面是自己在看的过程中记录下的对于自己来说比较容易遗忘的知识点。   



## 《SQL基础教程》	 
1. 当聚合键中包含NULL时，也会将NULL作为一组特定的数据。  
2. 除了select和from字句时，只包含where和group by字句时，字句的书写顺序select=>from=>where=>group，而字句的执行书序为from=>where=>group by=>select。  
3. 与聚合函数和group by子句有关的常见错误  
	(1) 常见错误1 — 在select子句中书写了多余的列。实际上，使用聚合函数时，select子句中只能存在以下三种元素： 
	* 常数   
	* 聚合函数.  
	* Group by子句中指定的列名（也就是聚合键）（MySQL中支持非聚合键列名，但是除了MySQL别的DBMS都不支持这样的语法，因此请不要使用这样的写法） 
  
	(2) 常见错误2 — 在group by子句中写了列的别名（在MySQL和PostgreSQL中可以执行正确，只是不是通常的使用方法）。      
	(3) 常见错误3 — group by子句的结果能排序吗？ Group by子句的结果是随机排序的。    
	(4) 常见错误4—在where子句中使用聚合函数. 
4. where子句只能指定记录（行）的条件，而不能用来指定组的条件（例如，“数据行数为2行”或者“平均值为500”等）。  
5. 加上having和order by子句后，子句的书写为select=>from=>where=>group by=>having=>order by，执行顺序为from=>where=>group   by=>having=>select=>order by。  
6. having子句中能够使用的3种要素如下所示：  
* 常数
* 聚合函数
* Group by子句中指定的列名（即聚合键）
7. 相对于having子句，更适合写在where子句中的条件，有些条件既可以写在having子句当中，又可以写在where子句当中，这些条件就是聚合键所对应的条件。那么应该写在哪呢？应该写在where子句中。原因有两个  
(1) 首先根本原因是where子句和having子句的作用不同。having子句是用来指定“组”的条件的，因此，“行”对应的条件还是应该写在where子句当中。  
(2) 通常情况下，为了得到相同的结果，将条件写在where子句中要比写在having子句中的处理速度更快，返回结果所需的时间更短。原因有两个：第一，使用count函数等对表中的数据进行聚合操作时，DBMS内部就会进行排序处理，排序处理会大大增加机器的负担；通过where子句指定条件时，由于排序之前就对数据进行了过滤，所以能够减少排序的数据量，但having子句是在排序之后才对数据进行分组的；第二，where子句更具速度优势的另一个理由是，可以对where子句指定条件所对应的列创建索引，这样也可以大幅提高处理速度。  
8. 不能对NULL使用比较运算符，因此，使用含有NULL的列作为排序键时，NULL会在结果的开头或末尾汇总显示。  
9. 在Oracle、SQL Server、PostgreSQL和MySQL中，存在只能删除表中全部数据的truncate语句，该语句比delete速度要快得多。因此需要删除全部数据行时，使用truncate可以缩短执行时间。  
10. 事务是需要在同一个处理单元中执行的一系列更新处理的集合。使用事务开始语句和事务结束语句，将一系列DML语句（insert/update/delete语句）括起来，就实现了一个事务处理。但在标准SQL中并没有定义事务的开始语句，而是由哥哥DBMS自己来定义的，比较有代表性的语法如下所示：  
* SQL Server、PostgreSQL
Begin transaction
* MySQL
Start transaction
* Oracle、DB2
无
11. 视图和表的区别只有一个，那就是“是否保存了实际的数据”。  
(1) 视图是一张临时表，视图中的数据会随着原表的变化自动更新。视图归根到底就是select语句。  
(2) 应该将经常使用的select语句做成视图。  
(3) 多重视图会降低SQL的性能，应该尽量避免。  
12. 视图的限制  
(1) 定义视图时不能使用order by子句。（有些DBMS可以，例如postgreSQL）.   
(2) 对视图进行更新。视图和原表要同时更新，所以要对视图进行更新，一些比较有代表性的条件是：  
* select子句中未使用distinct
* from子句只有一张表
* 未使用group by子句
* 未使用having子句
13. 标量查询就是返回单一值的子查询，即必须而且只能返回1行1列的结果。
14. 使用子查询的SQL会从子查询开始执行。
15. 使用关联子查询时，经常犯的一个错误就是将关联条件写在子查询之外的外层查询之中。因为一般会违反关联名称的作用域规则（子查询内部设定的关联名称，只能在该子查询内部使用）。  
16. 大部分SQL函数对NULL操作的结果是NULL（例如字符串拼接函数\|\|，但该函数在SQL Server和MySQL中无法使用，SQL Server使用+，MySQL使用concat函数）。
17. coalesce函数—将NULL转换为其他值
用法：coalesce（数据1，数据2，数据3……）：返回可变参数中左侧第一个不是NULL的值。
18. 使用谓词in和not in是无法选取出NULL数据的。（谓词，简单的说，就是返回值为真值的函数）。
19. exists谓词  
(1) exists只需在右侧有一个参数，该参数通常都会是一个关联子查询。  
(2) exists只关心记录是否存在，所以返回哪些列都没有关系。  
20. case表达式是一个表达式，可以写在任意位置。
21. 集合运算的注意事项  
(1) 注意事项1—作为运算对象的记录的列数必须相同  
(2) 注意事项2—作为运算对象的记录中列的类型必须一致  
(3) 注意事项3—可以使用任何select语句，但order by子句只能在最后一次使用  
22. 在集合运算union后面加上all可以保留结果中的重复行。
23. 外联结有一点非常重要，就是要把哪张表作为主表。最终结果中会包含主表内所有的数据。指定主表的关键字是left和right。
24. 窗口函数  
	(1) 窗口函数也称OLAP函数，为了让大家快速形成直观印象，才起了这样一个容易理解的名称。OLAP是Online Analytical Processing的简称，意为对数据库数据进行实时分析处理。窗口函数就是为了实现OLAP而添加的标准SQL功能。  
	(2) 语法：<窗口函数\> over(\[partition by<列清单\>\] order by <排序用列清单\>)     
	(3) 窗口函数大体可以分为以下2种:  
		1) 能够作为窗口函数的聚合函数（sum、avg、count、max、min）  
		2) Rank、dense_rank、row_number等专用窗口函数   
	(4) partition by对表的横向进行分组，而order by决定了纵向排序的规则。  
	(5) rank():计算排序时，如果存在相同位次的记录，则会跳过之后的位次。例如1、1、1、4。  
	Dense_rank()：同样时计算排序，即使存在相同位次的记录，也不会跳过之后的位次。例如1、1、1、2。  
	Row_number()：赋予唯一的连续位次。例如有3条记录排在第一位时，1、2、3、4。  
	(6) 窗口函数只能写在select子句之中。（严格的说，也能写在order by子句或者update语句的set子句中，但是几乎没有实际的业务示例）。  
25. grouping运算符.  
(1) grouping运算符包含以下3种：rollup、cube、grouping sets
