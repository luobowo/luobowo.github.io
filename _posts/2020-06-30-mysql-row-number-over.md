---
layout: post
title:  "row_number() over函数学习"
date:   2020-06-30 11:26:00
categories: mysql
tags: mysql
---

* content
{:toc}



最近在项目中遇到一个这样的需求：有一个流量图，需要统计从A节点流出的流量，以及每一个下游节点收到的流量，用于统计比例，每一条流量的边作为sql的一行进行存储。如下面的sql所示：

```sql
create Table `test_table` (
 `id` int NOT NULL AUTO_INCREMENT COMMENT 'id，系统自增',
 pre_node_name varchar not null comment '上游节点名称',
 pre_node_out int not null comment '上游节点流出流量',
 next_node_name varchar not null comment '下游节点名称',
 next_node_in int not null comment '下游节点流进流量',
 rate double not null comment '下游节点流量占上游节点比例',
 primary key (id)
) COMMENT='测试'
```

然后插入下面的数据：

```sql
insert into test_table
(pre_node_name, pre_node_out, next_node_name, next_node_in, rate)
values
("A", 100, "B", 40, 0.4),
("A", 100, "C", 60, 0.6),
("D", 200, "E", 100, 0.5),
("D", 200, "F", 100, 0.5);
```

数据显示为：

| id | pre_node_name | pre_node_out | next_node_name | next_node_in | rate |
|:---:|:---:|:---:|:---:|:---:|:---:|
|1	  |	A	| 100 |B	|40	  |0.4  |
|2    |	A	| 100 |C    |60	  |0.6  |
|3	  |	D	| 200 |E	|100  |0.5  |
|4	  |	D	| 200 |F    |100  |0.5  |


**在统计汇总数据的时候，怎么统计流出的流量一共多少呢**？`select sum(pre_node_out) from test_table`肯定是错误的，会重复统计。答案是使用`row_number() over`函数。

**row_number() over函数的用法**  

首先看看row_number() over函数的用法：

```sql
ROW_NUMBER() OVER (
    [PARTITION BY partition_expression, ... ]
    ORDER BY sort_expression [ASC | DESC], ...
)
```
`partition by`语句将结果按照partition by的列进行分组，`order by`语句将每个分组内的数据按照那列进行排序，然后`row_number()`	对每个分组从1开始进行编号。例如以下语句：

```sql
select *, row_number() over(partition by pre_node_out order by next_node_name asc) rank from test_table order by pre_node_name asc
```
得到结果：

| id | pre_node_name | pre_node_out | next_node_name | next_node_in | rate | rank |
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
|1	  |	A	| 100 |B	|40	  |0.4  | 1 |
|2    |	A	| 100 |C    |60	  |0.6  | 2 |
|3	  |	D	| 200 |E	|100  |0.5  | 1 |
|4	  |	D	| 200 |F    |100  |0.5  | 2 |

<font color=red>回到最开始的那个问题，怎么统计流出的流量一共多少呢？</font>我们只要按照上游节点名进行分组，然后再统计每一组第一个个流出的流量就可以了：

```sql
select sum(case when rank=1 then pre_node_out else 0 end) from 
(select *, row_number() over(partition by pre_node_name) rank from test_table)
```
得到正确结果：300.


## 参考文献

1. [ROW_NUMBER() OVER函数的基本用法用法](https://www.cnblogs.com/icebutterfly/archive/2009/08/05/1539657.html)  
2. [SQL Server ROW_NUMBER Function](https://www.sqlservertutorial.net/sql-server-window-functions/sql-server-row_number-function/)  






