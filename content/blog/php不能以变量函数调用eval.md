+++
title = 'php不能以变量函数调用eval'
date = '2023-08-12T11:11:00+08:00'
categories = ['学习', '安全', 'web']
tags = ['安全', 'php', 'rce']
+++

# php不能以变量函数调用eval

## 变量函数：

将一个函数的名称赋值给一个变量，在这个变量后面加上括号就实现了变量函数调用。

先举一个例子

```php
<?php
$a="system";
$b="echo nihao";
$c='eval';
$a($b);
echo  "\n";
输出 nihao  相当于执行了 system("echo nihao");
```

再看一个例子

```php
$a="system";
$b="echo nihao";
$c='eval';
$c($b);
echo  "\n";
这里运行抛出了异常，相当于 eval("echo nihao");
```

这里是因为eval在php中并不是一个函数，而是语言结构，也就是不符合变量函数。

assert在==php7.2==之前是函数。在7.2版本之后也变成了语言结构。

## eval:

将字符串当做php代码执行。

## assert：

执行有返回值的php代码。

## 本地测试

```php
<?php
highlight_file(__FILE__);

@$a = $_GET['a'];
@$b = $_GET['b'];
@$a($b);
?>
 
```

这里我使用的php5.6版本。

先通过eval测试：
![image-20230727164724075](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202307271647101.png)

发现没有任何回显。
再通过assert测试一下：
![image-20230727164735163](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202307271647201.png)

成功返回。

# 总结

之前一直以为eval是一个函数，现在知道了是语法结构。assert在==php7.2==之前是函数。在7.2版本之后也变成了语言结构

