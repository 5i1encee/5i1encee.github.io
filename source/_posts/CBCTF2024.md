---
title: CBCTF2024 WP
categories:
- CTF
tag:
- CTF
- web
- WriteUp
date: 2025-01-24 13:14:53
excerpt: CBCTF的简要wp......
index_img: /img/CBCTF2024/banner.png
banner_img: /img/CBCTF2024/banner.png
---

# CBCTF2024 WP

# Web

## 1.SignIn

![](/img/CBCTF2024/1.png)

主要是 get 传参 a、b，参数本身的值不等，md5 的结果强比较相等，结合提示的置顶帖子即可。

```
?a=TEXTCOLLBYfGiJUETHQ4hAcKSMd5zYpgqf1YRDhkmxHkhPWptrkoyz28wnI9V0aHeAuaKnak
&b=TEXTCOLLBYfGiJUETHQ4hEcKSMd5zYpgqf1YRDhkmxHkhPWptrkoyz28wnI9V0aHeAuaKnak
```

得到答案。

## 2.Notes 1

一开始以为是路径穿越的题。

URL 尝试 `/notes_read.php?file=note3.txt` 时发现返回 `Error: File not found : /var/www/html/note3.txt` 也就得知访问的根目录在 `/var/www/html/`，通过 `/notes_read.php?file=...`

传参读取该目录下的文件，因此访问 URL `/notes_read.php?file=notes_read.php` 就可以看到 notes_read.php 的源码。

![](/img/CBCTF2024/2.png)

读完代码发现对于参数 `$file` 存在正则匹配，且实际读取文件的路径 `$filePath` 由 `$notes_directory` 和 `$file` 拼接而成，无法直接获取 `/flag`。而后发现 `extract()` 存在变量覆盖漏洞，所以传参 `/notes_read.php?file=flag&notes_directory=/` 将变量 `$notes_directory` 的原内容覆盖，再拼接后即为/flag 路径，读取到 flag。

## 3.Notes 2

未完成

已有思路是反序列化后 `_0rays` 类下的 `__wakeup()` 先自动调用，而后 `__toString()`，`__set($a, $b)`，`__get($c)`，`__invoke()`，`__construct($filepath)`，最终 `readfile($filepath);` 获取 flag

但是实践起来还不熟练，没有做完。

![](/img/CBCTF2024/3.png)

# 后记

## 阳光长跑

补做了阳光长跑一题，主要考点是前后端的通信交互，用BP抓包发现开始跑步后浏览器向后端发送位置（经纬度）信息，推测后端根据位置信息的变化距离判断跑步长度，跑步时长由后端计算后返回，不受前端影响。

所以通过BP改变位置信息使距离达到要求然后再等时长到达合适的时间即可。

## Notes 2

该题为基础 PHP 反序列化，思路同上所述，具体参考我的另一篇博文[PHP反序列化小结](https://5i1encee.top/2025/02/01/PHP反序列化小结)

[官方WP](https://0rays-club.feishu.cn/wiki/SoYjwHDSGixa12kk1RYcKQHKnpd)
