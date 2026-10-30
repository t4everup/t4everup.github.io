+++
title = 'JPress v4.2 代码审计'
date = '2026-08-19T11:54:00+08:00'
tags = ["安全", "审计"]
+++

## 环境搭建

##### CVE-2024-50919

下载地址

```
https://gitee.com/JPressProjects/jpress/repository/archive/v4.2.0.zip
```

等待加载maven；之后在项目根目录执行`mvn clean package` 命令进行编译

![image-20241231190550096](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202412311905196.png)

根据自己配置，然后将文件名改为我上边的。

tomcat部署启动
![image-20250101161310075](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202501011613225.png)

## 审计过程

黑白盒结合

![image-20250101161807808](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202501011618873.png)

这边存在一个上传文件的位置，我们看能不能上传jsp
![image-20250101162034354](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202501011620479.png)

这边是一直失败的，我们尝试随便写一个后缀

![image-20250101162105931](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202501011621058.png)

可以看到直接就能上传上去了，那么后端很可能就是通过黑名单来进行判断的，我们尝试绕过

```
11.jsp::$DATA
```

![image-20250101162237976](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202501011622095.png)

我们看看文件

![image-20250101162310038](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202501011623077.png)

看到已经上传成功了，我们去访问；

![image-20250101162345206](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202501011623253.png)

返回500，说明文件是存在的，只是doFilter拦截了。

![image-20250101162520120](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202501011625178.png)

![image-20250101162920950](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202501011629001.png)

jspx jsp都不能访问。那我们就去尝试其他绕过

![image-20250101163021051](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202501011630164.png)

![image-20250101163356462](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202501011633509.png)

## 分析

这里上传一个php文件，看他是如何判断的

![image-20250101170734255](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202501011707331.png)

可能看到这里列了黑名单

![image-20250101170808184](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202501011708254.png)

根据前面就知道了，不能上传非法的文件。然后使用jspf绕过即可。

