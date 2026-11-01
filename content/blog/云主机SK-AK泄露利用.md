+++
title = '云主机SK/AK泄露利用'
date = '2024-07-30T09:15:20+08:00'
categories = ['学习', '安全']
tags = ['云安全']
+++

## 前言

昨天看了一份hw报告，其中是对云主机泄露了AK/SK的利用，之前没有接触过。今天来学习记录一下。本文章主要讲理论和一些工具介绍以及如何防护。

## 介绍

#### 什么是SK/AK

云主机通过**Access Key Id / Secret Access Key加密的方法来验证某个请求的发送者身份**。其中SK就是Secret Access Key；AK就是Access Key Id 。云SK/AK是用来生成用户和云平台API通信的访问凭证。AK/SK原理使用对称加解密。这样的话，也就是如果我们拿到了AK/SK,我们就有云主机的权限了。

## SK/AK泄露

大部分云主机都支持AK/SK认证的方式。用于API的调用，如果导致泄露，那么就能够攻击服务器。

### 泄露场景

1、报错页面或者debug信息调试。

2、GITHUB关键字、FOFA等。（阿里的已经采取相应的措施）

![image-20240730113902204](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202407301139257.png)

3、网站的配置文件

4、js文件中泄露

5、源码泄露。APK、小程序反编译后全局搜索查询。

6、文件上传、下载的时候也有可能会有泄露，比如上传图片、上传文档等位置。

7、HeapDump文件。

#### HeapDump文件泄露

HeapDump文件是JVM虚拟机运行时内存的一个快照，通常用于性能分析等，但是因为其保存了对象、类等相关的信息，如果被泄露也会造成信息泄露。

在生产环境中，需要实时或定期监控服务的可用性。Spring Boot的actuator（健康监控）功能提供了很多监控所需的接口，可以对应用系统进行配置查看、相关功能统计等。

 Actuator组件为Spring Boot提供对应用系统的监控和管理的集成功能，可以查看应用配置的详细信息。如果`Actuator`使用不当或者一些不经意的疏忽，可能造成信息泄露等严重的安全隐患。

解析heapdump工具：[wyzxxz/heapdump_tool: heapdump敏感信息查询工具，例如查找 spring heapdump中的密码明文，AK,SK等 (github.com)](https://github.com/wyzxxz/heapdump_tool)

#### APP泄露

有的程序可以在APP中泄露了key等的位置，使用工具可以确定到位置，然后进行访问。

工具：(jdax-gui)https://github.com/skylot/jadx

## 漏洞利用

#### 阿里提供的OSS图形化管理工具

https://github.com/aliyun/oss-browser

![image-20240730112041312](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202407301120398.png)

通过AK/SK验证。

#### 腾讯云云主机管理平台

https://cosbrowser.cloud.tencent.com/web/bucket

![image-20240730112159387](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202407301121465.png)

#### 对主机命令执行

[Phuong39/cf: 云环境利用框架（Cloud exploitation framework）主要用来方便红队人员在获得 AK 的后续工作。 (github.com)](https://github.com/Phuong39/cf)

在获取到AK/SK后 可以对主机执行命令。

![image-20240730112708267](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202407301127315.png)

#### 针对阿里云主机RCE

https://github.com/mrknow001/aliyun-accesskey-Tools

非常好用。

## 安全建议

- **不要将\**AccessKey\**嵌入代码中**

  嵌入代码中的AK凭证容易被人忽视，经验丰富的开发者会将其写入数据库或者独立的文件中，使得其管理起来更方便。

- **定期更新\**AccessKey\****

  定期更新代码中存在的**AccessKey**，可保证一些旧的代码泄漏后不会影响当前线上业务。

- **定期吊销不需要的\**AccessKey\****

  在阿里云AccessKey控制台可查看最后一次**AccessKey**的访问时间，建议**禁用**所有不用的**AccessKey**。

- **遵循最小权限原则使用RAM账户**

  根据不同业务需要授予不同子账户的读写权限，为不同业务分配不同子账户的**AccessKey**。

- **开启操作日志审计，并将其投递至OSS和SLS保存和审计**

  将操作日志存储至OSS，异常情况时可以起到固证的作用；操作日志投递至SLS，帮助您在日志数量大的时候也能实现高效检索。

