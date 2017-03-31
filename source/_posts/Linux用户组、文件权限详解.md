---
title: Linux用户组、文件权限详解
date: 2017-03-31 15:57:20
tags:
	- Linux
	- 文件权限
categories: Linux

---

要了解Linux下的文件/目录的权限，首先应该了解Linux下的用户、用户组的概念。
## I. 用户组
在linux中的每个用户必须属于一个组，不能独立于组外。在Linux中每个文件有所有者、所在组、其它组的概念。

- 所有者
一般为文件的创建者，谁创建了该文件，就天然的成为该文件的所有者，用"ls ‐ahl"命令可以看到文件的所有者，也可以使用"chown  用户名 文件名"来修改文件的所有者。

- 所在组
当某个用户创建了一个文件后，这个文件的所在组就是该用户所在的组，用"ls ‐ahl"命令可以看到文件的所有组，也可以使用"chgrp 组名 文件名"来修改文件所在的组。

- 其它组
除开文件的所有者和所在组的用户外，系统的其它用户都是文件的其它组。

<!-- more -->

## II. 文件权限
在某个有文件的目录执行如下命令
```
ls -l
```

我截取其中一个文件的显示详情
```
-rw-r--r-- 1 root root 24789 Mar 20 15:58 msgapi.log
```

“-rw-r--r--”这10位字符展示了不同的用户对这个文件的权限信息。

- 第一位有三种取值：文件（-）、目录（d），链接（l），这个例子是-代表是一个文件。

- 其余9位每3位一组，分别对应所有者，所在组，其他组；读（r）、写（w）、执行（x）

- 第一组rw-：文件所有者的权限是读、写

- 第二组r--：与文件所有者同一组的用户拥有读权限

- 第三组r--：不与文件所有者同组的其他用户拥有读权限

也可用数字表示为：r=4，w=2，x=1  因此rwx=4+2+1=7，其实就是3位2进制所对应的10进制的值。

除了前面10位，后面的字符的含义如下：

- 1 表示连接的文件数

- root : 所有者

- root : 用户组

- 24789  : 文件大小（字节）

- Mar 20 15:58  :  最后修改时间

- msgapi.log  : 文件名

 
## III. 修改文件的权限
```
chmod 权限标识符 目标文件
```

chmod 改变文件或目录的权限
```
chmod 777 msgapi.log
```
修改msgapi.log的权限为所有用户都可读可写可执行,在次执行 "ls -l"命令，结果如下
```
-rwxrwxrwx 1 root root 24789 Mar 20 15:58 msgapi.log

```

chmod u=rwx，g=rwx，o=rwx msgapi.log：同上,u=用户权限，g=组权限，o=不同组其他用户权限

chmod u-x，g+w msgapi.log：给msgapi.log去除用户执行的权限，增加组写的权限

chmod a+r msgapi.log：给所有用户添加读的权限


## IV. 改变文件所有者（chown）和用户组（chgrp）命令

chown xiaoming msgapi.log：改变msgapi.log的所有者为xiaoming

chgrp root msgapi.log：改变msgapi.log所属的组为root

chown root ./msgapi：改变msgapi这个目录的所有者是root

chown ‐R root ./msgapi：改变msgapi这个目录及其下面所有的文件和目录的所有者是root

 
## V. 改变用户所在组

在添加用户时，可以指定将该用户添加到哪个组中，同样用root的管理权限可以改变某个用户所在的组

- usermod ‐g 组名 用户名

可以用

- usermod ‐d 目录名 用户名，改变该用户登录的初始目录