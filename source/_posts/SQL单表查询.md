---
title: SQL单表查询
date: 2017-03-24 10:58:01
updated: 2017-04-06 18:39:01
tags:
  - SQL
  - 数据库
categories: database
---

### I.导语
数据库查询是数据库操作的核心，SQL提供select语句进行查询，其一般的格式为：
```
select [all | distinct] <目标列表达式> [,<目标列表达式>] ...
from <表名或试图名> [,<表名或试图名>] ...
[where <条件表达式>]
[group by<列名1> [having <条件表达式>]]
[order by<列名2> [ASC | DESC]];
```
<!-- more -->

### 数据表
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

-- 查询全体学生的学号和姓名
```
select Sno, Sname
from Student;
```
![此处输入图片的描述][1]


-- 查询学生表的全部信息
-- 方式一
```
select *
from Student;
```
-- 方式二，这种方式可以改变结果列的顺序,下面这个例子将Sno和Sname交互了位置
```
select Sname, Sno, Sage, Ssex, Sdept
from Student;
```
![此处输入图片的描述][2]


-- 查询经过计算的值
```
select Sname, 2017-Sage
from Student;
```
![此处输入图片的描述][3]


-- 改变表头为 birthYear
```
select Sname, 2017-Sage birthYear
from Student;
```
![此处输入图片的描述][4]

-- select 等价于 select all
```
select all Sno
from SC;
```
![此处输入图片的描述][5]

上面的结果中Sno有重复行，如何消除重复,使用distinct关键字

-- 消除重复行
```
select distinct Sno
from SC;
```
![此处输入图片的描述][6]

###II.查询满足条件的元组
查询满足条件的元组可以通过where子句实现。

常用的查询条件
|条件|谓语|
|:----|:----|
|比较|=, >, <, >=, <=, !=, <>, !>, !<; NOT+上述比较运算符|
|确定范围|between and, not between and|
|确定集合|in, not in|
|字符匹配|like, not like|
|空值|is null, is not null|
|多重条件（逻辑运算）|and, or, not|

(1)比较大小
-- 查询计算机系的全体学生名单
```
select Sname 
from Student
where Sdept='CS';
```
![此处输入图片的描述][7]

上面这个查询操作，RDBMS可能的一种操作是全表扫描，取出一个元组，检查该元组的Sdept列的值是否为CS，如果相等，则取出Sname形成新的元组输出，否则跳过。假设这个表有上万条数据，而Sdept=CS的人数较少，可以在Sdept上建立索引，系统会利用索引的来查找Sdept=CS的元组，避免全表扫描，加快查询效率。

-- 查询20岁以下的学生姓名和年龄
```
select Sname, Sage
from Student
where Sage<20;
```

![此处输入图片的描述][8]

-- 查询考试成绩有不合格的学生学号
```
select distinct Sno
from SC
where Grade < 60;
```
这里采用distinct消除重复行，因为一个学号可能有几门课不及格，只需要列出一次就行。

(2)确定范围
-- 查询年龄在 20到23岁（包含20/23）之间的学生的姓名、系别、年龄
```
select Sname, Sdept, Sage
from Student
where Sage between 20 and 23;
```
![此处输入图片的描述][9]

-- 查询年龄不在 20到23岁（包含20/23）之间的学生的姓名、系别、年龄
```
select Sname, Sdept, Sage
from Student
where Sage not between 20 and 23;
```
![此处输入图片的描述][10]

(3)确定集合

-- 查询计算机系（CS）、数学系（MA）和信息系（IS）的学生姓名和性别
```
select Sname, Ssex
from Student
where Sdept in ('CS', 'MA', 'IS');
```
![描述][11]

-- 查询不在计算机系（CS）、数学系（MA）和信息系（IS）的学生姓名和性别
```
select Sname, Ssex
from Student
where Sdept not in ('CS', 'MA', 'IS');
```
(4)字符匹配
一般格式

[not] like '<匹配串>' [escape '<换码字符>']

其含义是查找指定属性列值与匹配串相匹配的元组，匹配串可以是完整的字符串，也可以是带有通配符%和_。
%：代表任意长度（可以是0）的字符串。例如a%b表示以a开头b结尾的任意长度字符串。如abc, abgggc，ab都满足该匹配。
_:代表任意单个字符。例如a_b，表示以a开头b结尾的长度为3的任意 字符串。如：abc，afc等都满足该匹配。

-- 查询序号为 200215121 的学生的详细情况
```
select *
from Student
where Sno like '200215121';
```
这个等价于
```
select *
from Student
where Sno = '200215121';
```
如果like后面的匹配串不含通配符，like可用=（等于）来代替， not like可以用 ！= 或者<>（不等于来代替）。

-- 查询所有姓刘的学生的学号、姓名、性别
```
select Sno, Sname, Ssex
from Student
where Sname like '刘%';
```
![描述][12]

-- 查询姓欧阳且全名长度为3的学生姓名,一个汉字占两个_
```
select Sname
from student
where Sname like '欧阳__';
```
-- 查询名字中第二字为阳的学生的姓名
```
select Sname
from Student
where Sname like '__阳%';
```
-- 查询不姓刘的学生的姓名
```
select Sname
from Student
where Sname not like '刘%';
```
![图片描述][13]


如果查询的字符串包含通配符 %或者_，这时就要使用escape'<换码字符>'短语，对统配父进行转义。

-- 查询DB_design课程的课程号和学分
```
select Cno, Ccredit
from Course
where Cname like 'DB/_design'
escape '/';
```
escape '/'表示 “/”为转义字符，这样紧跟在“/” 后面的“_”不在具有统配符的含义，转义为普通的“_”字符。

-- 查询课程名以DB_开头且倒数第三个字符为i的课程详情
```
select *
from Course
where Cname like 'DB/_%i__' escape '/';
```
![图片描述][14]
(5)涉及空值的查询
-- 查询成绩为空的学号和课程号
```
select Sno, Cno
from sc
where Grade is null;
```
(6)多条件查询

逻辑运算符and 和 or可以联结多个查询条件。and的优先级高于or，但是可以通过括号来该变优先级。

-- 查询计算机系年龄在20岁以下的学生姓名
```
select Sname
from Student
where Sdept = 'CS' and Sage < 20;
```

![图片描述][15]

-- 查询计算机系（CS）、数学系（MA）和信息系（IS）的学生姓名和性别
```
select Sname, Ssex
from Student
where Sdept in ('CS', 'MA', 'IS');
```
-- 上面这个可以改造为 or 联结条件
```
select Sname, Ssex
from Student
where Sdept='CS' or Sdept='MA' or Sdept='IS';
```
###III.order by子句
用户可以通过order by 子句对查询结果按照一个或多个属性列的升序（asc）或降序（desc）排列，缺省为升序排序。

-- 查询选修了3号课程的学生的学号及其成绩并按照成绩的降序排列
```
select Sno, Grade
from SC
where Cno=3
order by Grade desc;
```
对于空值的，升序空值排在最后，降序排在最前面。

-- 查询所有学生的信息，按系的升序排列，同一个系的按照年龄降序排序
```
select *
from Student
order by Sdept, Sage desc;
```
###IV.聚集函数（aggregate functions）

SQL提供了许多聚集函数，主要包括：

|函数|含义|
|:--|:--|
|count([distinct\|all]*)|统计元组个数|
|sum([distinct\|all] <列名>)|计算一列的总和（此列必须是数值型）|
|avg([distinct\|all] <列名>)|计算一列的平均值（此列必须是数值型）|
|max([distinct\|all] <列名>)|求一列的最大值|
|min([distinct\|all] <列名>)|求一列的做小值|

-- 查询学生的总数
```
select count(*)
from Student;
```
![图片描述][16]

-- 查询选修了课程的学生人数 ，消除重复学号
```
select count(distinct Sno)
from SC;
```

![图片描述][17]

-- 计算1号课程的平均成绩
```
select avg(Grade)
from SC
where Cno=1;
```
![图片描述][18]

-- 查询1号课程的最高分
```
select max(Grade)
from SC
where Cno='1';
```
-- 查询200215122学生的总学分
```
select sum(Ccredit)
from sc, course
where sc.Sno='200215122' and sc.Cno = course.Cno;
```
![图片描述][19]

### V.group by子句

group by 子句将查询结果按照某一列或多列的值进行分组，值相同的为一组。

-- 求各个课程号和相应的选课人数
```
select Cno,count(Sno)
from SC
group by Cno;
```
![图片描述][20]

-- 查询选修了3门课以上的学生学号
```
select Sno
from sc
group by Sno
having count(*)>3;
```
where子句和having短语的的区别在于作用对象不同。where做的的是基本表或者视图，从中选择符合条件的元组；而having短语作用的是组，从中选择符合条件的组。


  [1]: https://raw.githubusercontent.com/jdqm/hello-world/master/db/Image2.png
  [2]: https://raw.githubusercontent.com/jdqm/hello-world/master/db/Image3.png
  [3]: https://raw.githubusercontent.com/jdqm/hello-world/master/db/Image4.png
  [4]: https://raw.githubusercontent.com/jdqm/hello-world/master/db/Image5.png
  [5]: https://raw.githubusercontent.com/jdqm/hello-world/master/db/Image6.png
  [6]: https://raw.githubusercontent.com/jdqm/hello-world/master/db/Image7.png
  [7]: https://raw.githubusercontent.com/jdqm/hello-world/master/db/Image8.png
  [8]: https://raw.githubusercontent.com/jdqm/hello-world/master/db/Image9.png
  [9]: https://raw.githubusercontent.com/jdqm/hello-world/master/db/Image10.png
  [10]: https://raw.githubusercontent.com/jdqm/hello-world/master/db/Image11.png
  [11]: https://raw.githubusercontent.com/jdqm/hello-world/master/db/Image12.png
  [12]: https://raw.githubusercontent.com/jdqm/hello-world/master/db/Image13.png
  [13]: https://raw.githubusercontent.com/jdqm/hello-world/master/db/Image14.png
  [14]: https://raw.githubusercontent.com/jdqm/hello-world/master/db/Image15.png
  [15]: https://raw.githubusercontent.com/jdqm/hello-world/master/db/Image16.png
  [16]: https://raw.githubusercontent.com/jdqm/hello-world/master/db/Image17.png
  [17]: https://raw.githubusercontent.com/jdqm/hello-world/master/db/Image18.png
  [18]: https://raw.githubusercontent.com/jdqm/hello-world/master/db/Image19.png
  [19]: https://raw.githubusercontent.com/jdqm/hello-world/master/db/Image20.png
  [20]: https://raw.githubusercontent.com/jdqm/hello-world/master/db/Image21.png
