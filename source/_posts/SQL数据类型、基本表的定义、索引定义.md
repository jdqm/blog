---
title: SQL数据类型、基本表的定义、索引定义
date: 2017-03-24 10:07:53
tags:
---

### 前言
由于最近做消息中心后台开发，需要进行数据库的操作，在编写SQL（mybatis）上略感吃力，于是复习一下大学时代所学习的数据库的相关基础知识，直接从SQL开始。
     
### I.```SQL```与```模式```
![此处输入图片的描述][1]


  [1]: https://raw.githubusercontent.com/jdqm/hello-world/master/db/Image.png
  
### II.本次学习所用到的几个表
Student
| 学号 Sno | 姓名 Sname  |  性别 Ssex | 年龄 Sage | 所在系 Sdept |
| :----:   | :----:   | :----:  | :----:  | :----:  |
| 20021521     | 李勇 |   男    |  20     |  CS     |
| 20021522     | 刘晨 |   女    |  19     |  CS     |
| 20021523     | 王敏 |   女    |  18     |  MA     |
| 20021524     | 张力 |   男    |  19     |  IS     |

Course
| 课程号 Cno | 课程名 Cname  |  先行课 Cpno | 学分 Sage |
| :----:   | :----:   | :----:  | :----:  | 
| 1     | 数据库 |   5    |  4     |
| 2     | 数学 |       |  2    |
| 3     | 信息系统 |   1    |  4     |
| 4     | 操作系统 |   6    |  3     |
| 5     | 数据结构 |   7    |  4     |
| 6     | 数据处理 |       |  2    |
| 7     | PASCAL语言 |   6    |  4     |

CS
| 学号 Sno | 课程号 Cno  |  成绩 Grade |
| :----:   | :----:   | :----:  |
| 20021521     | 1 |  92   |
| 20021521     | 2 |  85   |
| 20021521     | 3 |  88   |
| 20021522     | 2 |  90   |
| 20021522     | 3 |  80   |

### III.SQL的数据定义语句
| 操作对象 | 创建 |  删除 |  修改 |
| :----:   | :----:   | :----:  | :----:  |
| 模式     | create schema |  drop schema   |    |
| 表     | create table |  drop table   |alter table    |
| 视图     | create view |  drop view   |    |
| 索引     | create index |  drop index   |    |

### IV.创建表
一般格式
```
create table <表名> (<列名> <数据类型> [列级完整性约束]
    [ , <列名> <数据类型> [列级完整性约束]]
    ...
    [,<表级完整性约束>] );
```
例如：
```
create table Student(
    Sno char(9) primary key,  /*列级完整性约束， Sno为主码*/
    Sname char(20) unique,   /*列级完整性约束， Sname唯一*/
    Ssex char(2),
    Sage smallint,
    Sdept char(20)
);
```
```
create table Course(
    Cno char(4) Primary key,  /*列级完整性约束， Cno为主码*/
    Cname char(40),
    Cpno char(4),                  /*Cpno是先行课*/
    Ccredit smallint,
    foreign key (Cpno) references Course(Cno)  /*表级完整性约束，Cpno 是外码，被参考Course,被参照Cno*/

);
```
```
create table SC(
    Sno char(9),
    Cno char(4),
    Grade smallint,
    primary key (Sno, Cno),   /* 主码有两个属性构成，必须作为表级完整性约束进行定义*/
    foreign key (Sno) references Student(Sno),  /*表级完整性约束，Sno 是外码，被参考Student,被参照Sno*/
    foreign key (Cno) references Course(Cno)   /*表级完整性约束，Cpno 是外码，被参考Course,被参照Cno*/
);
```



### V.关于数据类型
各个数据库的数据类型不完全相同，选择数据类型的原则有两个，一个是取值范围，二是要做哪些数据运算。比如年龄：取值为100左右的正整数，取值范围很多数据类型都满足，比如char(3)、长整形、短整型；运算比如需要求平均年龄，这个只有长整形和短整型支持，考虑到存储空间的占用，故选用短整型。

|数据类型|含义|
|:----:|:----:|
|char(n)|长度为n的定长字符串|
|varchar(n)|最大长度为n的变长字符串|
|int|长整数，也可以写成integer|
|smallint|短整数|
|numeric(p,d)|定点数，由p位数字（不包含小数点、符号）组成，小数点后面有d为数字|
|real|取决于机器精度的浮点数|
|double precision|取决于机器精度的双精度浮点数|
|float(n)|浮点数，精度至少为n位数字|
|date|日期，包含年、月、日，格式为 YYYY-MM-DD|
|time|时间，包含一日的时、分、秒，格式为 HH:MM-SS|

更多数据类型请参考具体的数据库厂商说明书。

### VI.修改基本表
一般格式
```
alter table <表名> 
    [add <列名> <数据类型> [完整性约束]]
    [drop <完整性约束 名>]
    [alter column<列名> <数据类型>];
```
例如：
```
alter table Student add S_ontrance date;  /*学生表增加入学时间列*/

alter table Student alter column Sage int; /*修改学生表的Sage列的数据类型为int，这个在mysql中执行错误*/

alter table Course add unique(Cname); /*增加课程名唯一的约束*/
```
### VII.删除表
一般格式
```
drop table <表名> [restrict | cascade] 
```
restrict：删除表示有限制的，如果该表有外键、视图、触发器、存储过程、函数等约束，则删除失败，缺省为restrict

cascade: 删除没有条件，删除该表时，相关联的对象，视图等都会被删除，使用值需谨慎；不同的数据库厂商对的实现细节还是有差距的，比如mysql如果该表有外键的话即使指定cascade也是无法删除的。


比如：在mysq下
```
drop table Student restrict;
```
提示错误信息：
[SQL] drop table Student restrict ;
[Err] 1451 - Cannot delete or update a parent row: a foreign key constraint fails

```
drop table Student cascade;
```
[SQL] drop table Student cascade;
[Err] 1451 - Cannot delete or update a parent row: a foreign key constraint fails

验证一个表只有视图的情况下：restrict和cascade的差别：首先给SC表创建一个视图，然后删除该表，SC表无其他关联约束：
```
create view v_sc as select * from SC;
drop table SC restrict;
drop table SC cascade;
```
可以看到在mysql的实现下restrict，cascade 都可以删除表，但是视图没有删除，在这一点上表现是一致的。

### VIII.建立索引
建立索引是加快查询的有效手段。

一般格式
```
create [unique] [cluster] index <索引名称> on <表名>(<列名>[<次序>] [,<列名>[<次序>]]...);
```
```
/**建立索引*/
create unique index StuNoIndex on Student(Sno);
```
### IX.删除索引
一般格式
```
drop index<索引名称> on <表名>
```
```
drop index StuNoIndex on Student;
```
索引已经建立，由系统使用并维护，不许用户干预。建立索引是为了减少查询时间，如果数据频繁增删改，系统会花费许多时间来维护索引，从而减低了数据库系统的效率，这个时候可以删除一些不必要的索引。在RDBMS中一般采用B+树、HASH索引来实现，B+树具有动态平衡的优点，Hash索引则具有查找快速的优点。索引是数据库的内部实现技术，属于内模式范畴。至于某一索引在创建时是采用B+树还是Hash索引则由具体的RDBMS来决定。


