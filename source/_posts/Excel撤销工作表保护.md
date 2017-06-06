---
title: Excel撤销工作表保护
date: 2017-06-06 23:10:14
updated: 2017-06-06 23:10:14
tags: -解密
categories: 加密／解密
---

昨天有一个做工程造价的同学让我帮她“破解”下工作表保护的密码，不破解的话这个Excel表格是编辑不了的，当尝试去修改文档中的内容时，就会弹出一个对话框提示“尝试修改的单元格在受保护的工作表中”，这个时候你肯定会去想，能不能把这个保护关了。<!-- more -->于是你找到顶部菜单中的“审阅”，然后工具栏就会出现“撤销工作表保护”的按钮，很激动地点了一下，这个时候就变成了下面的场景：

![场景展示][1]
 
这个时候可能会出现如下3中情况：
1、有密码，哈哈哈；
2、大脑短路把密码忘记了；
3、这个文档是别人的，不知道密码。

当你不知道密码，又想去撤销这个保护怎么办？

1、通过某些手段去知道这个密码，比如通过穷举；
2、绕过这个密码保护，直接把它变成没有保护。

第一种方式通常是通过“宏”去暴力破解，困难指数与密码的复杂度相关。这里介绍第二种直接“干掉”保护开关的方式。那么问题来了，我怎么知道如何去关掉这个保护？

这就要回到Excel文件本身，我们知道，Excel文件实际上就是一个压缩文件,Excel文档拷贝复制到别的设备，这个保护仍然有效，这就告诉我们，密码和保护开关都保存在这个Excel文档中，关键是我们要去找到是在哪判断这个文档是被保护的，只要把这个参数改成不受保护即可。下面就是具体的步骤：

1、将Excel文档的后缀改成 .zip

![修改文件后缀][2]

（如果你的电脑不显示文件的扩展名，win10用户请按照如下操作勾上“文件扩展名”，如果是win7用户请自行google或百度如何显示文件扩展名）

![显示文件后缀][3]

2、使用 7-zip压缩工具打开压缩包（别的压缩工具也可以，我比较习惯使用7-zip）

![压缩包根目录][4]

3、打开下面这个目录
```
xl/worksheets/
```
![目标目录][5]

这个目录下的sheet1.xml就是我们要找的文件，使用Notepad++打开这个文件，并搜索"sheetProtection"

![sheet1.xml文件内容][6]

4、只需要删掉图中红色方框框起来的内容即可（即sheetProtection节点），删完内容后替换掉压缩包中的sheet1.xml即可。

5、替换原来的压缩包中的sheet1.xml文件后，将文件的扩展名改回原来的 .xlsx，这个时候用Excel打开文档，工作表就变成没有保护的了。

最后研究一下被删除的这段代码都是什么东西
```
<sheetProtection algorithmName="SHA-512" 
hashValue="Qp2ANj6SKH6pd2Jv3WYjrWMvOzFRlwIRcf3EbPueSqnJ8ihPU5pYrs4wL3ahCkAUaN6gG8+EX12MpxywvX6Mvg==" saltValue="fvMYsSPur7ijWXkAa0m7tg==" 
spinCount="100000" 
sheet="1" 
objects="1" s
cenarios="1"/>
```

|属性|含义|
| :-- | :-- |
|algorithmName| 加密算法|
|hashValue|密码加密后的值|
|saltValue|加密算法的加盐|
|spinCount|旋转的次数？这个加密算法不太了解|
|sheet|表格的数量，应该是指该文件的表格数量|
|objects|对象数量，不知道是不是指加密表格的数量|
|scenarios|脚本数量|

看完这种加密方式，你还会选择去暴力破解吗？！

声明：本文仅供娱乐参考使用，切勿损害他人利益。

 [1]: https://raw.githubusercontent.com/jdqm/hello-world/master/Excel/excel_1.png
 [2]: https://raw.githubusercontent.com/jdqm/hello-world/master/Excel/excel_2.png
 [3]: https://raw.githubusercontent.com/jdqm/hello-world/master/Excel/excel_3.png
 [4]: https://raw.githubusercontent.com/jdqm/hello-world/master/Excel/excel_4.png
 [5]: https://raw.githubusercontent.com/jdqm/hello-world/master/Excel/excel_5.png
 [6]: https://raw.githubusercontent.com/jdqm/hello-world/master/Excel/excel_6.png
