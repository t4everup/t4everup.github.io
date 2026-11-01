+++
title = '批量网站sec权重查询小工具'
date = '2024-06-04T12:53:03+08:00'
categories = ['安全', 'python']
tags = ['开发']
+++

## 前言

好久没更新文章了0-0，(水篇文章) 最近在努力探究java安全(太难了ww)，还接触了src漏洞挖掘。大多都是0权重的网站，但是有的平台不要0权重的网站，有的平台要。于是就有了个想法，自己写一个网站seo批量查询的脚本工具，脚本不算太难，练练手 hh 为以后漏洞复现打下编写脚本的能力。

[**脚本链接**]  <https://github.com/4everwl>

## 简介

使用python3编写的一个批量查询网站权重的小工具。通过批量txt文件，将最终的结果输出为txt文件，目前只包含百度，移动，谷歌相关权重。使用多线程，提高脚本的运行速度。

## 模块安装

```
pip install requests
```

## 使用方法

```
python3 domain_check.py txt文件路径
```

![image-20240604182047570](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202406041820646.png)

结果会输出到当前目录下的out文件夹内，结果文件命名根据时间命名，更好的判断出文件位置。

![image-20240604182152604](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202406041821635.png)

![image-20240604182212796](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202406041822837.png)

## 注意事项

[尽量使用python3.9以上版本运行。有关脚本问题以及src挖掘交流联系email：[email protected]](https://blog.foreverwl.top/cdn-cgi/l/email-protection#ac491c11452b234811134b3804dcd5d8c4c3c29f82954817094814264b25244a300044133c440d204f2c2e4a302549291f4428364a3000453b02450e34481709492326dfdecf4a203a4a22344816084a192d442d384b1f17c9c1cdc5c04310369d9a94eccac3dec9dac9dedbc082d8c3dc)
