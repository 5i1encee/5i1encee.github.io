---
title: SQL 学习笔记
categories:
- Note&Summary
tag:
- CTF
- web
date: 2024-09-07 23:18:01
excerpt: 学习中......
index_img: /img/SQL.png
banner_img: /img/SQL.png
---

# SQL 学习笔记

# SQL 学习笔记

本篇笔记内容与 B 站课程 [SQL 注入由简入精](https://www.bilibili.com/video/BV1c34y1h7So?spm_id_from=333.1245.0.0)基本一致，为知识点和个人心得总结。

# sqli-labs 靶场搭建

1. 下载 phpstudy
2. github 上下载 sqli-labs 环境复制到 phpstudy 根目录下

# SQL 基础常用语法语句

## 增删改

### 数据库

`show databases;`

（根目录下输入）查看数据库

`create database employees charset utf8;`

创建数据库 employees 并选择字符集

`drop database employees;`

删除数据库 employees

`use employees;`

选择进入数据库 employees

### 数据表

```
create table employee
(
id int,
name varchar(40),
sex char(4),
birthday data,
job varchar(100)
);
```

创建数据表 employee，必须设定列

`drop table employee;`

删除数据表

`rename table employee to user;`

修改数据表名称为 user

`alter table user character set utf8;`

修改字符集

### 数据列和数据行

```
insert into user
(
    id,name,sex,birthday,job)
values
(
    1,'ctfstu','male','1999-05-06','IT')
;
```

写入内容

`alter table user add salary decimal(8,2);`

增加一列内容 salary，最大 8 位，小数点后保留 2 位

`update user set salary=5000;`

修改所有工资为 5000

`update user set name='benben' where id=1;`

修改 id=1 的行 name 为 benben

`update user set name='benben2',salary=3000 where id=1;`

修改 id=1 的行 name 为 benben2，工资为 3000

`alter table user drop salary;`

删除指定列

`delete from user where job='IT';`

删除指定行

`delete from user;`

删除表中所有数据（不删除表的结构）

## 查

### 概述

基础查询词：

`select`，`from`，`where`

查询参数指令：

`union`，`group by`，`order by`，`limit`，`and`，`or`

常用函数：

`group_concat()`，`database()`，`version()`

### 基本语句

`select * from user where id=1;`

select+ 列名（*代表所有）+from+ 表名（user）+where+ 条件语句（id=1）+;

`select * from user where id in ('1');`

在 user 表中查询所有包含 id 为 1 的数据

`select * from user where id=(select id from user where username=('admin'));`

子查询，优先执行()内语句

### 查询参数指令

#### union

`select id from user union select email_id from emails;`

查询并合并数据显示

`select * from user where id=6 union select * from emails where id=6;`（错误示例）

返回 `ERROR: have a different number of columns` 联合注入 union 左右表格列数必须相等，此例 user 有 3 列，emails 有 2 列

`select * from user where id=6 union select *,3 from emails where id=6;`（正确示例）

由于 union 右比左少一列，因此用 3 作为填充列，使左右列数相等，查询结果 emails 部分第三列数据即为 3

#### group by

本身实现的是分组功能但也可用于判断列数。由于存在一些问题不推荐常用，例如：user 表中有两条相同姓名不同 id 的数据，只对一列分组可以，对多列分组会报错。一些防火墙过滤可能不严格，可以方便利用。

`select username from user group by username;`

在 user 表中查询 username 列，按 username 分组输出

`select username from user group by 1;`

同上输出，username 仅一列，本句查询按第一列分组

`select username from user group by 2;`

报错，username 仅一列，本句查询按第二列分组，不存在第二列

`select * from user group by 2;`

正常输出，user 表三列数据，按 user 表内第二列分组。

想要判断列数则可以根据以上原理从 2 开始 3、4、5······往后尝试，当出现报错时的数-1 即为列数

实际 SQL 注入时使用二分法，先取一个足够大会返回报错的数，而后不断取半（中间数），直到找到边界。

例如 `select * from user where id=1 group by 10;` 10->5->3->4

#### order by

本身实现的是排序，同 group by 可用于判断列数

`select * from user order by id;`

在 user 表中按 id 升序排序

`select * from user where username='Xiao Ming' order by id;`

可在 order by 前加入 where 条件

`select * from user where username='Xiao Ming' order by id desc;`

order by 的排序默认升序，在后面加 desc 改为降序

`select * from user order by 1;`

同 group by 一样可按第几列排序，判断列数方法同样相似

#### limit

限制输出内容数量，一般用于报错注入，限数显示报错反馈信息

`select * from user limit 0,3;`

限制为从第 1 行开始显示 3 行，在这里实际的第一行就是命令里的第 0 行

当回显内容有限想查看同一级其他数据时可以在语句后加上 `limit 0,1`，`limit1,1`，`limit2,1` 等等

#### and & or

即“与”和“或”。额算是比较重要但也没什么好说的。

### 常用函数

#### group_concat()

将多行合并拼接至一行显示（并用指定分隔符分隔），在 union 注入及报错注入时 group_concat 尤为重要

`select group_concat(id,username,password) from user;`

将 user 表内的 id，username，password 合并在一行输出

当遇到限制回显数量只有一条时，就可以用 `group_concat()` 较方便的一次性获得所有内容，例如 `group_concat(table_name)`，比 `limit` 一条一条查更方便。但是如果源码当中显示位数设置较少的话这种方法就可能导致显示不全，用 `limit` 则更加稳妥

#### concat()

效果与 group_concat()类似，是将任意长度的多个字符串拼接成一个字符串，在报错注入时也是尤为重要

#### select database()

查看当前数据库名称

#### select version()

查看当前数据库版本，防火墙绕过时用得到，方便找到对应版本的漏洞

# SQL 注入基础

## 基础概念

SQL 注入就是通过构造一条精巧的 SQL 命令语句来查询得到想要的信息。

注入点即可以实现输入的地方，通常是一个访问数据库的连接。

## 分类

按查询字段分：

数字型，输入参数为整型

字符型，输入参数为字符串（被单引号闭合）

按注入方法分：
union 注入、报错注入、布尔注入、时间注入

## 判断方法

### 数字型 or 字符型？

数字型提交内容一般为数字，但<u>数字不一定是数字型</u>，<u>数字型不需要闭合符来闭合</u>

字符型提交内容则需要<u>闭合符闭合</u>

1.使用 `and 1=1` 和 `and 1=2` 来判断

如果为数字型，前者照常显示返回信息，后者不会显示，因为 1 与 2 不等

如果为字符型，两者都能正常显示信息，相当于输入内容均被单引号包裹，and 不作为命令执行

2.使用(?id)`=2-1` 来判断

如果为数字型则返回的是 `?id=1` 的内容

如果为字符型则无法计算，返回为 `?id=2` 的内容

不推荐用 `+`，因为 URL 编码通常用 `+` 和 `%20` 代替空格，解码时相应会被解码成空格

### 若字符型则闭合方式？

常见闭合方式：

`'`、`"`、`')`、`")`、其他

1.输入 `?id=1'"`，报错为 `......near 1'"'......`，多了一个 `'` 则闭合符为 `'`

2.输入 `?id=1'"`，报错为 `......near 1'""......`，多了一个 `"` 则闭合符为 `"`

3.输入 `?id=1'"`，报错为 `......near 1'"')......`，多了一个 `')` 则闭合符为 `')`

4.输入 `?id=1'"`，报错为 `......near 1'"")......`，多了一个 `")` 则闭合符为 `")`

即尝试输入内容中的闭合符与输入内容前的闭合符闭合，输入内容后原本的闭合符则会变得多余，从而报错，我们根据报错得到命令当中所用的闭合符

闭合的作用：

结束前一段查询语句，在后面可加入其他语句，查询需要的参数

后面不需要的语句可以用注释符 `--+`（此处跟的 `+` 实际是代替空格，避免直接输入空格被忽略，也可以通过在空格后随便加上一些内容避免这个问题，例如 `-- abc`）、`#`、`%23` 注释掉

## union 联合注入

用二分法判断默认页面数据列数量，参考前面的 union、group by、order by 内容，再使用联合注入获得目标结果

但是页面只能显示一个内容，union 后的语句查询的内容是不显示的，因此为了显示想要的内容，可以将前一句查询的内容改为数据库中原本不存在的数据，如 `id=-1`，那么前面一句查询不到结果就只会显示后面一句查询的结果

此外并非每一列的内容都会回显，要先判断会回显的是哪些列。例如 sqli-labs 的 less-1，注入 `?id=-1' union select 1,2,3--+`，会回显的是 2、3，因此我们可以二三两列位置换成其他想要的内容，例如 `?id=-1' union select 1,version(),database()--+`

注入步骤：

1.查找注入点

2.判断字符型还是数字型

3.若是字符型则找到其闭合方式

4.判断查询列数 group by/order by

5.查询回显位置

而后可参考这篇小结查询所需信息

## 报错注入

有时尝试 union 联合注入发现语句正确但无对应回显，而输入错误语句时仍会有报错回显，那么就可以尝试报错注入

报错注入的基础是后台对于输入输出的合理性未做检测

报错注入的实现就是构造语句让错误语句中夹杂可以显示数据库内容的查询语句，使得返回的报错提示中包含数据库中的内容

### extractvalue()报错注入

extractvalue()函数包含两个参数：XML 文档对象名称、路径

根据以下例子了解该函数用法

1、先在 ctfstu 数据库内创建表 xml

```sql
create database ctfstu charset utf8;
create table xml(doc varchar(150));
```

2、在表内插入两段数据

```sql
insert into xml values('
<book>
<title>A bad boy how to get a
girlfriend</title>
<author>
<initial>Love</initial>
<surname>benben</surname>
</author>
</book>
');
```

```sql
insert into xml values('
<book>
<title>how to become a bad boy</title>
<author>
<initial>hualong</initial>
<surname>Melton</surname>
</author>
</book>
');
```

3、使用 extractValue()查询 xml 内的内容

```sql
select extractvalue(doc,'/book/author/surname')from xml;    #查询作者

返回：
benben
Melton
```

```sql
select extractvalue(doc,'/book/title')from xml;    #查询书名

返回：
A bad boy how to get a girlfried
how to become a bad boy
```

4、写错查询参数中的路径

例如 `select extractvalue(doc,'/book/titlelll')from xml;`

结果查询不到内容但是也不会报错

5、写错查询参数的格式符号

例如 `select extractvalue(doc,'~book/title')from xml;`

显示报错信息 `#1105 - XPATH syntax error: '~book/title'`

至此我们发现当写错查询参数的格式符号时会有报错返回，显示我们写错的路径。因此我们或许可以尝试在报错之前查询所需信息通过报错返回查看查询结果。

例如 `select extractvalue(doc,concat(0x7e,(select database()))) from xml;`，括号内的语句 `select database()` 优先执行，然后查询结果通过 `concat()` 函数与 `~` 符号拼接在一起作为 `extractvalue()` 函数的路径参数，最后因为格式不符规范，后台返回报错信息，而报错信息中包含“错误路径”也就是我们故意构造的查询语句的查询结果。至此，一次报错注入完成。想要查询其他内容只需要将相应的查询语句替换进 `select database()` 所在位置。

实际在运用这种报错注入时通过 `union select ......` 或者 `and 1=......` 注入均可，只要最终报错即可

例如 `?id=100' and 1=extractvalue(1,concat(0x7e,(select group_concat(username,'~',password) from users))) --+`

然而报错注入还受到一个限制，`extractvalue()` 默认只能返回 32 个字符串，有时就会显示不全。对此的解决方法是运用 `substring(str,num1,num2)` 函数，其中 str 是操作的目标字符串，num1 是起始位置，num2 是截取个数，实现的效果就是显示字符串 str 从第 num1 位起 num2 个字符长度的字符串

例如 `?id=100' and 1=extractvalue(1,concat(0x7e,(select substring(group_concat(username,'~',password),25,30) from users))) --+`

### updatexml()报错注入

updatexml()函数包含三个参数：XML 文档对象名称、路径、要替换的新数据

该函数的功能就是更新替换 XML 文档中的指定内容

该函数的报错原理与 extractvalue()一样，输入带错误符号的第二个参数。该函数也同样存在 32 位的限制，同样参考 extractvalue()用 substring()解决

### floor()报错注入
