+++
title = 'session_start对数组处理进行写文件'
date = '2023-10-17T12:55:00+08:00'
categories = ['学习', '安全', 'web']
tags = ['安全', 'php']
+++

看代码：

```
<?php
$action = $_GET['action'];
$parameters = $_GET;
if (isset($parameters['action'])) {
    unset($parameters['action']);//消除数组中的action
}

call_user_func($action, $parameters);//回调函数

if(count(glob(__DIR__.'/*'))>3){//对当前目录进行判断，如果文件大于3 读取flag文件
    readfile('flag.txt');
}

?>
```

也就是往当前目录写文件，我们如果想要使用fopen()，file_put_contents()，进行写入，参数至少都要两个，而这里只能传一个参数；可以考虑session_star()，这个函数可以接收一个数组进行初始化，数组中可以设定session options

## session_start():

session_start — 启动新会话或者重用现有会话。session_start(array `$options` = []): bool

```
options
此参数是一个关联数组，如果提供，那么会用其中的项目覆盖 会话配置指示 中的配置项。此数组中的键无需包含 session. 前缀。
```

可以看到对options的解释：此数组中的键无需包含 session. 前缀。[PHP: 运行时配置 - Manual](https://www.php.net/manual/zh/session.configuration.php)

### session.save_path：

定义了传递给存储处理器的参数。如果选择了默认的 files 文件处理器，则此值是创建文件的路径。默认为 `/tmp`。

那么我们就能够自定义存储会话的存储路径，我们就能够往指定的目录下进行保存我们的回话文件。

```
?action=session_start&save_path=D:/a_tool/phpStudy/PHPTutorial/WWW/zsxq/10.5
```

然后我们添加或修改cookie，我们的会话文件就保存在指定的目录。

## 注意：

测试时注意文件的权限，使用php 7.0以上测试 ,我使用的php7.1.11。
