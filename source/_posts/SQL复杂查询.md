---
title: SQL复杂查询
date: 2017-03-30 16:52:18
tags:
	-数据库
	-SQL
categories: database
---

PS:本文中数据库表请参考上一篇文章 http://blog.csdn.net/jdqm2014/article/details/64921344

## I.连接查询
定义
前一篇中提到的查询都是针对单个表的。若一个查询涉及到两个以上的表，则称之为连接查询。连接查询是关系型数据库中的主要查询，包括等值连接查询、自然连接查询、非等值连接查询、自身连接查询、外连接查询和复合条件连接查询等。

1、等值与非等值连接查询

一般格式
[表名1.]<列名1> <比较运算符> [表名2.]<列名2> 

其主要运算符有：=、>、<、>=、<=、!=(或<>)等；

当比较运算符为=时，称为等值连接，其他的运算符称为非等值连接。
```

-- 查询每个学生及其选修课程的情况
select student.*, sc.*
from student, sc
where student.Sno=sc.Sno;
```

<!-- more -->
![此处输入图片的描述][1]

上述例子在属性前面加上了表名，这个因为两个表都存在Sno这一列，加上前缀可以避免混淆，当参与连接的表的属性列是唯一的时候，此前缀可省略。

若把等值连接中目标列中重复的属性列去掉则为自然连接，如：
```
-- 自然连接
select student.Sno, Sname,Ssex, Sage, Sdept, Cno, Grade
from student, sc
where student.Sno=sc.Sno;
```

![][2]

2、自身连接

一个表与自己连接，则称为表的自身连接。比如在Course表中，我们只能的到直接的先修课，如果要得到先修课的先修课，则必须与自身连接。
```
-- 查询课程的先修课的先修课
select c1.Cno, c2.Cpno
from course c1, course c2
where c1.Cpno=c2.Cno;
```
![][3]

3、外连接

在通常的连接中，只有满足的条件的元组才能作为结果输出。例如下面这个例子
```
-- 查询每个学生及其选修课程的情况
select student.*, sc.*
from student, sc
where student.Sno=sc.Sno;
```
没有选课的学生信息被舍弃了。有时候想以Student表为主体列出每个学生的信息和选课信息，若某个学生没有选课，则在SC表的属性列填充null值，这时候就要使用到外连接。
```
-- 左外连接查询
select student.Sno, Sname, Sage, Ssex, Sdept, Cno, Grade
from student
left join sc on (student.Sno=sc.Sno);
```

![][4]

左外连接列出左边关系的所有元组（例如本例中的Student），由外连接列出右边关系的所有元组。

4、复合条件连接
前面所提到的连接查询，where子句只有一个条件，当where子句中有多个连接条件时，称为复合条件连接。
```
-- 查询选修了2号可并且成绩在90分以上的学生信息
select student.Sno, Sname
from student,sc
where student.Sno=sc.Sno and sc.Cno=2 and Grade>=90;
```
```
-- 查询选修了2号可并且成绩在90分以上的学生信息
select student.Sno, Sname, Grade
from student,sc
where student.Sno=sc.Sno and sc.Cno=2 and Grade>=90;
```

![][5]

```
-- 查询每个学生的学号、选修的课程名、成绩
select student.Sno, Cname, Grade
from student,course,sc
where student.Sno=sc.Sno and sc.cno=course.cno;
```

![][6]

## II.嵌套查询

定义
在SQL语言中，一个select-from-where语句称为一个查询块。将一个查询块嵌套在另一个查询块的where子句或者having短语的条件中的查询称为嵌套查询。
1、在in谓词的子查询
```
-- 使用嵌套查询查出选修了2号课程的学生的学号
select Sno
from student
where Sno in(
    select Sno
    from sc
    where Cno='2'
);
```
这类查询外层查询（父查询）和内层查询（子查询）的条件不相关，称为不相关子查询。

2、带比较运算符的子查询
```
-- 查询每个学生成绩超过他平均成绩的课程号
select Sno, Cno
from sc x
where Grade >(
        select avg(Grade)
        from sc y
        where y.Sno=x.Sno);
```
子查询依赖父查询的Sno，这类查询称为相关子查询。

3、带有any(some)或all谓词的子查询

子查询返回单值时可以用比较运算符，返回多值要用any(有的系统用some)或者all谓词来修饰，而使用any或all谓词修饰是必须同时使用比较运算符，其语义为：

|比较运算|语义|
|:--|:--|
|  ```> ANY```|大于子查询中的某个值|
|  ```> ALL```|大于子查询中的所有值|
|  ```< ANY```|小于子查询中的某个值|
|  ```< ALL```|小于子查询中的所有值|
|  ```>= ANY```|大于等于子查询中的某个值|
|  ```>= ALL```|大于等于子查询中的所有值|
|  ```<= ANY```|小于等于子查询中的某个值|
|  ```<= ALL```|小于等于子查询中的所有值|
|  ```= ANY```|等于子查询中的某个值|
|  ```= ALL```|等于子查询中的所有值(通常没有意义)|
|  ```!=(或<>) ANY```|不等于子查询中的某个值|
|  ```!=(或<>) ALL```|不等于子查询中的所有值|

```
-- 查询其他系中比计算机系某一个学生年龄小的学生姓名和年龄
select Sname, Sage
from student
where Sage < any(
        select Sage
        from student
        where Sdept='CS'
        )
and Sdept !='CS';
```

![][7]
```
-- 用聚集函数查询其他系中比计算机系某一个学生年龄小的学生姓名和年龄
select Sname, Sage
from student
where Sage <(
        select max(Sage)
        from student
        where Sdept='CS'
        )
and Sdept !='CS';
```

事实上，用聚集函数实现比用any或all谓词效率要高。

4、带有exists，not exists谓词的子查询

带有exists的子查询不返回人户数据，只返回true 或者false。
```
-- 查询选修了1号课的学生姓名
select Sname
from student
where exists(
        select * from sc
        where student.Sno=sc.Sno and Cno='1'
);
```
使用 not exists就是去取其非值。
```
-- 查询没有选修了1号课的学生姓名
select Sname
from student
where not exists(
        select * from sc
        where student.Sno=sc.Sno and Cno='1'
);
```
## III. 集合查询

select 语句一般是返回多个元组的集合，所以多个select语句的结果集合可以进行集合操作。集合操作包括并操作（union），交操作（intersect）和差操作(except)。但是需要注意的是参与集合运算的集合列数量必须相等，而且数据类型也要相同。
```
-- 查询计算机系及年龄不大于19岁的学生 ，不保留重复的元组
select * from student
where Sdept='CS'
union
select * from student
where Sage<=19;
```

![][8]
```
-- 查询计算机系及年龄不大于19岁的学生，保留重复的元组
select * from student
where Sdept='CS'
union all
select * from student
where Sage<=19;
```
![][9]

  [1]: https://raw.githubusercontent.com/jdqm/hello-world/master/db/Image22.png
  [2]: https://raw.githubusercontent.com/jdqm/hello-world/master/db/Image23.png
  [3]: https://raw.githubusercontent.com/jdqm/hello-world/master/db/Image24.png
  [4]: https://raw.githubusercontent.com/jdqm/hello-world/master/db/Image25.png
  [5]: https://raw.githubusercontent.com/jdqm/hello-world/master/db/Image26.png
  [6]: https://raw.githubusercontent.com/jdqm/hello-world/master/db/Image27.png
  [7]: https://raw.githubusercontent.com/jdqm/hello-world/master/db/Image28.png
  [8]: https://raw.githubusercontent.com/jdqm/hello-world/master/db/Image29.png
  [9]: https://raw.githubusercontent.com/jdqm/hello-world/master/db/Image30.png