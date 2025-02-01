---
title: Hgame2024 WP
categories:
- CTF
tag:
- CTF
- web
- misc
- reverse
- WriteUp
date: 2024-07-03 20:24:13
excerpt: 拖了好久没整理上来的wp......
index_img: /img/Hgame2024/HGAME.jpg
banner_img: /img/Hgame2024/HGAMEs.jpg
---

# Hgame2024 WP

# Week1

## web

### 1.Bypass it

一开始想偏了，以为要绕过别的什么，但其实就是绕过对注册的阻止就行，查看html，可以看到注册相关的页面地址。

![](/img/Hgame2024/Week1/1.png)

![register_page.php](/img/Hgame2024/Week1/2.png)

然后直接向register.php传参就好了`username=1&password=1&register=注册`。然后登录。

### 2.ezHTTP

先是“请从vidar.club访问这个页面”，`Referer=vidar.club`即可

再是“请通过Mozilla/5.0 (Vidar; VidarOS x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/121.0.0.0 Safari/537.36 Edg/121.0.0.0访问此页面”，`User Agent=Mozilla/5.0 (Vidar; VidarOS x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/121.0.0.0 Safari/537.36 Edg/121.0.0.0`即可

然后“请从本地访问这个页面”，我第一个想到的是`X-Forwarded-For`，但是没反应，发现响应头里有提示“Hint：Not XFF”，所以需要更换其他等效的字段，最终`X-Real-IP: 127.0.0.1`可以发挥作用。这里附上所有起类似作用的语句（在博客园一篇博客上找到的，忘记具体出处了）

	Client-IP:127.0.0.1
	Forwarded-For-Ip: 127.0.0.1
	Forwarded-For: 127.0.0.1
	Forwarded-For: localhost
	Forwarded:127.0.0.1
	Forwarded: localhost
	True-Client-IP:127.0.0.1
	X-Client-IP: 127.0.0.1
	X-Custom-IP-Authorization : 127.0.0.1
	X-Forward-For: 127.0.0.1
	X-Forward: 127.0.0.1
	X-Forward: localhost
	X-Forwarded-By:127.0.0.1
	X-Forwarded-By: localhost
	X-Forwarded-For-Original: 127.0.0.1
	X-Forwarded-For-original: localhost
	X-Forwarded-For: 127.0.0.1
	X-Forwarded-For: localhost
	X-Forwarded-Server: 127.0.0.1
	X-Forwarded-Server: localhost
	X-Forwarded: 127.0.0.1
	X-Forwarded: localhost
	X-Forwared-Host: 127.0.0.1
	X-Forwared-Host: localhost
	X-Host: 127.0.0.1
	X-Host: localhost
	X-HTTP-Host-Override : 127.0.0.1
	X-Originating-IP: 127.0.0.1
	X-Real-IP: 127.0.0.1
	X-Remote-Addr: 127.0.0.1
	X-Remote-Addr : localhost
	X-Remote-IP: 127.0.0.1

于是“Ok, the flag has been given to you ^-^”，去看响应头发现有“Authorization”，将JWT放到<https://jwt.io/>解密即可。

### 3.Select Courses

这题我当时没做出来，就放一个官方题解记录一下：

***

题⽬主要考察的是选⼿编写脚本的能⼒。

帮助阿菇选到所有课程，即可获取FLAG。后端逻辑是每间隔 30s-180s 放出⼀⻔课，若 5s 内没有选到课程，则课程⼜会满员。已经被选上的课程不会再放出。当所有课程都选上之后，点击“选完了”按钮，后端判定所有课程都已经被选择，就会返回给前端FLAG。

选⼿可以⼿动选课，但⼯作量会⽐较⼤；也可以通过编写脚本来⾃动抢课，⽐如基于python的selenium编写抢课脚本：

```
from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
from time import sleep

driver = webdriver.Chrome()
driver.get("http://127.0.0.1:8000")
sleep(3)

courses_list = []

for i in range(1, 6):
	course = {
		'panel': f'//*[@id="selector-container"]/section[{i}]/div[1]',
		'status': f'//*[@id="selector container"]/section[{i}]/div[2]/table/tbody/tr/td[5]',
		'submit': f'//*[@id="selector container"]/section[{i}]/div[2]/table/tbody/tr/td[6]/button'
	}
	courses_list.append(course)

print(courses_list)

while courses_list:
	driver.refresh()
	sleep(2)
	for course in courses_list:
		panel = driver.find_element(By.XPATH, course['panel'])
		panel.click()
		status_element = driver.find_element(By.XPATH, course['status'])
		status_text = status_element.text
		print(status_text)
		if status_text != "已满":
			submit_button = driver.find_element(By.XPATH, course['submit'])
			submit_button.click()
			WebDriverWait(driver, 5).until(EC.alert_is_present())
			alert = driver.switch_to.alert
			
			alert.accept()
			courses_list.remove(course)
			break

sleep(10)
driver.quit()
```

同时，也可以编写脚本或使⽤Burpsuite等⼯具持续发包，检测到返回值为 `{'full': 0,'message': '选课成功！'} `即表示抢到某门课。

***

## misc

### 1.来自星尘的问候

![](/img/Hgame2024/Week1/secret.jpg)

题目描述里提到了六位弱加密，应该就是用的steghide用六位密码搞的隐写，所以放进脚本里分离一下，爆破密码之前试了123456结果直接就成功了，获得了隐藏的压缩包，里面有一张奇怪的图片。

![](/img/Hgame2024/Week1/3.png)

![](/img/Hgame2024/Week1/4.png)

再根据题目里说的官网上找相关文字，就可以去一个个对应翻译。但更方便的是直接搜网上对这个文字的分析。然后又发现网上有人指出可以去官网扒woff2文件(<https://g.nga.cn/read.php?tid=39109851&rand=99>)，跟大小写字母和数字逐一对应就好了。

![](/img/Hgame2024/Week1/5.png)

### 2.simple_attack

压缩包解密题，里面一张图片加另一个压缩包，包中包里有一个加密过的跟外面一样的图片（放Bandizip里看crc一致且命名一致）和加密过的txt，压缩算法都是ZipCrypto，那基本就是明文攻击的题型了。

![最外层压缩包内容](/img/Hgame2024/Week1/6.png)

![内层压缩包attachment.zip内容](/img/Hgame2024/Week1/7.png)

所以把外面的那张未加密图片按压缩算法ZipCrypto压缩，其他项也要与压缩包内加密的图片一致，压缩级别逐个试过来就是正常压缩，然后放到ARCHPR里明文攻击，得到解密后的attachment压缩包。

![](/img/Hgame2024/Week1/8.png)

打开photo.txt里面是Data URI scheme的格式，放到浏览器地址栏里就可以查看图片，得到flag。

![](/img/Hgame2024/Week1/9.png)

### 3.希儿希儿希尔

拿到手是一个不能正常显示的图片，题目说需要修复，所以在010Editor检查了一下，图片格式没有问题，但在末尾有secret.txt和PK，说明图里藏有压缩包，直接改了拓展名，拿到压缩包里的txt文件。但没法直接解密，暂时还不知道有什么用。

![](/img/Hgame2024/Week1/10.png)

![](/img/Hgame2024/Week1/11.png)

然后想把它放进Stegsolve里看看，结果打不开，看来应该是宽高被修改，需要crc校验。于是到python脚本里跑出正常宽高并在010Editor修改，得到希儿的正常图片。

![](/img/Hgame2024/Week1/12.png)

接下来就可以在Stegsolve里正常打开，顺便检查了一下属性没什么问题，但发现了LSB隐写藏着可疑的数据。

![](/img/Hgame2024/Week1/13.png)

因为一开始没有好好看题，结果始终不知道这个到底是怎么用的，甚至后来以为这里只是我多想了转而去尝试其他隐写，直到我又看了一遍题：题目名字“希儿希儿希尔”最后一个是“希尔”且“不过他似乎忘了这个加密的名字不是希儿了”，也就是说题目已经给出提示，去网上一搜“希尔加密”还真有，然后在[Bugku](https://ctf.bugku.com/tool/hill)解决。

## reverse

这部分因为第一周的简单就尝试了一下。

### 1.ezASM

一开始去临时学习了一下汇编知识，后来感觉没必要，像C语言理解应该就可以，于是把上面c里的数和0x22异或一下，ASCII转文字就是flag了。

![](/img/Hgame2024/Week1/14.png)

### 2.ezUPX

如题，是个UPX，所以先用upx去壳即可。

![](/img/Hgame2024/Week1/15.png)

![](/img/Hgame2024/Week1/16.png)

输入的flag要与0x32异或后等于byte_1400022A0的内容，所以找到并异或回去就得到flag。（新学的快捷键shift+e导出这些文本）

![](/img/Hgame2024/Week1/17.png)

![](/img/Hgame2024/Week1/18.png)

# Week2

## misc

### 1.ek1ng_want_girlfriend

流量分析题，是对Wireshark使用（和ek1ng）的初步介绍，流量文件中发现一张ek1ng.jpg的图片，将其导出，图片是ek1ng的照片和flag。

![](/img/Hgame2024/Week2/1.png)

![](/img/Hgame2024/Week2/2.png)

![](/img/Hgame2024/Week2/3.png)

### 2.ezWord

下载一个attachment.zip，里面“这是一个word文件.docx”，打开是“你好，这个文件的内部有你想要的”和一张图片。大概是文档加密，把文档放进010Editor，发现PK字样，说明内部藏有压缩包。改为zip后缀并解压，发现两张看起来一样的图片“100191209_p0.jpg”“image1.png”和secret.zip（打开是加密的secret.txt和直接可读的提示“你好，很高兴你看到了这个压缩包。请注意：这个压缩包的密码有11位数而且包含大写字母小写字母和数字。还有一个要注意的是，里面的这一堆英文decode之后看上去是一堆中文乱码实际上这是正常现象，如果看到它们那么你就离成功只差一步了。”）根据题目描述“破译图片的水印”可以知道考点大概率是图片盲水印，两张照片一张是原图一张是打水印后的图，而压缩包的密码应该就是水印内容。因此用github上的“bwmforpy3.py”处理。

![](/img/Hgame2024/Week2/4.png)

在处理后得到的“fan_bwm.png”中得到压缩包密码。打开secret.txt，里面是有些莫名其妙的疑似邮件内容，我一开始还以为是把信息隐藏在文本中，后来直接放到搜索引擎搜发现有一种加密方法加密后的结果刚好相似<https://www.spammimic.com/decode.cgi>。

![](/img/Hgame2024/Week2/5.png)

得到解密结果，还差最后一层加密。根据提示Unicode（感觉不提示真想不到），查看这些中文乱码的Unicode编码。

![](/img/Hgame2024/Week2/6.png)

再看看hgame的编码。

![](/img/Hgame2024/Week2/7.png)

前几个字符一一对应，发现都刚好相差31753，说明这段中文是flag经过Unicode编码的偏移的结果。所以用python把他们都处理一下得到flag。

![](/img/Hgame2024/Week2/8.png)

![](/img/Hgame2024/Week2/9.png)

## web

### 1.myflask（当时没完成，事后发现就差一点点点点......呜呜呜）

一进入就把后端的app.py发了过来。
```
import pickle
import base64
from flask import Flask, session, request, send_file
from datetime import datetime
from pytz import timezone

currentDateAndTime = datetime.now(timezone('Asia/Shanghai'))
currentTime = currentDateAndTime.strftime("%H%M%S")

app = Flask(__name__)
# Tips: Try to crack this first ↓
app.config['SECRET_KEY'] = currentTime
print(currentTime)

@app.route('/')
def index():
    session['username'] = 'guest'
    return send_file('app.py')

@app.route('/flag', methods=['GET', 'POST'])
def flag():
    if not session:
        return 'There is no session available in your client :('
    if request.method == 'GET':
        return 'You are {} now'.format(session['username'])
    
    # For POST requests from admin
    if session['username'] == 'admin':
        pickle_data=base64.b64decode(request.form.get('pickle_data'))
        # Tips: Here try to trigger RCE
        userdata=pickle.loads(pickle_data)
        return userdata
    else:
        return 'Access Denied'
 
if __name__=='__main__':
    app.run(debug=True, host="0.0.0.0")

```

研究一下代码，在进入‘/’时将客户端的session内username内容设为guest并返回app.py。在进入‘/flag’时有session的前提下若为get方法则显示当前的username，当username等于admin时读取post方法传递的pickle_data并base64解码，然后用pickle.loads()函数反序列化存储至userdata并返回。所以大致的思路就是伪造session使自己的username=admin，然后以pickle反序列化触发RCE。而session伪造的前提是对原session解码修改并得到SECRET_KEY。解码session用网上找的脚本跑一下就好，正是`{'username': 'admin'}`。SECRET_KEY如何获得则看代码中app.config['SECRET_KEY']=currentTime，而currentTime等于某个按%H%M%S格式的时间，因此我们可以尝试写一个字典爆破SECRET_KEY。

![](/img/Hgame2024/Week2/10.png)

![](/img/Hgame2024/Week2/11.png)

用flask-unsign爆破。

![](/img/Hgame2024/Week2/12.png)

得到SECRET_KEY然后修改guest为admin并伪造session即可。然后放入cookie发送，成功。

![](/img/Hgame2024/Week2/13.png)

![](/img/Hgame2024/Week2/14.png)

然后就是尝试通过pickle反序列化触发RCE了，先是傻傻去查看app.py所在目录然后突然想起来在错误传参使之报错时已经显示了文件目录，又去看了看app.py所在目录下的文件发现只有这个，然后想看看上级目录下的文件，不知道为啥（可能用错函数了）结果返回为空。最后因为出门在外没法做题导致时间不足来不及截止前做完T^T

后来：

重新尝试，构造payload的程序如下
```
import pickle
import urllib
from base64 import b64encode

class payload(object):
def __reduce__(self):
	return (eval, ("open('/flag').read()",))
###原本这里也写的/flag.txt的，看完wp后删了txt试了试发现可以！啊啊啊啊啊啊啊啊啊啊啊啊！！！！！痛苦！！！！！！！！！！！###

a = pickle.dumps(payload(),protocol=0)
print(b64encode(a).decode())
```
这段语句本质上就是构造`__reduce__`魔术方法，然后将要执行的命令pickle序列化，再base64加密处理，最后把输出的结果贴到`pickle_data=`后面post传参即可。（其实就是把前面分析出来的app.py的处理过程反过来）

![最后成功了。](/img/Hgame2024/Week2/15.png)

对于其他小细节的研究：

	eyJ1c2VybmFtZSI6Imd1ZXN0In0.ZcteaQ.c9lMyjsOph-sEkwoxMqB9TzqwwA
	eyJ1c2VybmFtZSI6Imd1ZXN0In0.Zcthhg.ijiHa85G3dwoCC08Wlk6koLtEiI

尝试对比后中间段时间戳刷新就变，与服务器最新数据的时间有关。

	'SECRET_KEY'=203048

尝试刷新后用新session爆破key，还是不变的。但启用新靶机就不一样了，说明app.py里所获取并赋值给SECRET_KEY的时间是靶机开启时间。

# Week3

## misc

### 1.与ai聊天

一道简单AI题，题目描述让我们从AI嘴里“翘出”flag，如图：

![](/img/Hgame2024/Week3/1.png)

当问AI flag的时候“But first,could you please tell me your name?”猜测AI会根据身份的判断选择给不给flag，因此说admin作为尝试，但AI表示他不能提供flag。

![](/img/Hgame2024/Week3/2.png)

于是猜测admin应该不是正确身份，但我还是谴责了AI，没想到他一边道歉一边就说漏嘴了，他只能提供flag给Dr.Chen，换个身份flag到手。

### 2.Blind SQL Injection

对SQL盲注的流量分析，先用http作为过滤器筛选，按时间顺序排序，这样前一个是发送到靶机的请求，后一个跟的就是相应的服务器的响应。

![](/img/Hgame2024/Week3/3.png)

先大致了解这里用到的语法，ascii(x)函数就是将字符x转ASCII码，substr(a,b,c)函数就是截取a字符串从b处起长度为c的部分，reverse()函数则是将字符串倒转，group_concat()函数将组中的字符串连接成为具有各种选项的单个字符串。

再看注入的内容。图中substr(...,33,1)就相当于提取第33位字符用于操作。“%3E”按十六进制ASCII码即为“>”，“%3E”前面的部分ascii()函数将“F1naI1y”中SQL注入者想获得的内容第33位转为ASCII码，推测得“%3E”后的数则是SQL注入者所猜测的字符的ASCII码。这里用的是布尔盲注，SQL注入者要结合回显判断猜测是否正确。当所求内容的ASCII码>猜测的ASCII码即id=1-1=0时，回显“ERROR!!!”；当所求内容的ASCII码<=猜测的ASCII码即id=1-0=1时，回显“NO! Not this! Click others~~~”，也就是说找到回显为“NO! Not this! Click others~~~”的最小ASCII码即为该位的内容。要获得完整内容就把每一位（指substr(...,n,1)）拼接起来。下图是本题中的两种响应：

![](/img/Hgame2024/Week3/4.png)

![](/img/Hgame2024/Week3/5.png)

而整个流量文件中SQL的注入分为四个部分：

第一部分是获取数据库名称（table_schema），按上述方法分析得数据库名称geek。

![](/img/Hgame2024/Week3/6.png)

第二部分是获取geek数据库中的表名（table_name），分析得表名F1naI1y。

![](/img/Hgame2024/Week3/7.png)

第三部分获取F1naI1y表中的列名（column_name），分析得可用列名password。

![](/img/Hgame2024/Week3/8.png)

第四部分获取password列中数据，这里面大概就是我们要找的flag了。同理分析可得flag{cbabafe7-1725-4e98-bac6-d38c5928af2f}（因为reverse()函数，按时间顺序得到的是倒过来的内容，倒回来就是flag）。

![](/img/Hgame2024/Week3/9.png)

最后总分位列第23名，1540分，再接再励！