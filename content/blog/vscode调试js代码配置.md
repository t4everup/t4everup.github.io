+++
title = 'vscode调试js代码配置'
date = '2023-09-03T07:37:00+08:00'
categories = ['教程', '学习', 'nodejs']
tags = ['vscode', 'js']
+++

## 前言

前几天在分析原型链污染的代码时，对js代码需要调试，当我用vscode进行调试时发现了一些问题。调试运行不起来？

## 如何用vscode对js代码进行调试？

### 第一步

这里我们使用node.js对js代码进行调试。下载安装node.js环境。官网下载即可，[Node.js (](https://nodejs.org/en)[nodejs.org](http://nodejs.org)[)](https://nodejs.org/en)。安装完毕后我们使用 node -v查看一下是否环境配置成功。

### 第二步

![image-20230903091851935](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202309030918021.png)

打开我们要调试的文件夹，这里我创建一个新的文件夹演示。

![image-20230903092126079](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202309030921154.png)

我们使用vscode打开工作文件夹。我们单击 运行->打开配置

![image-20230903092531956](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202309030925004.png)

这个时候会自动创建一个文件夹，包含了一个launch.json文件，就是我们要配置的文件。

![image-20230903092702712](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202309030927786.png)

这里面是一些配置信息，我们只需要修改最后一个program，这个${workspaceFolder}表示当前目录，后面是我们要进行调试的文件，我们这里对1.js进行调试。

### 第三步

到这里我们已经可以对代码进行调试了，使用快捷键F5启动调试。

![image-20230903093514117](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202309030935179.png)

左边可以看到一些变量，上边我们可以对代码进行下一步或者上一步的调试。

## 结语

一直敲的代码都不多，一些基本的功能也不常用。vscode确实是一个很好用的开发工具，首先占用的运行内存小(轻量化)，其次通过插件实现多种语言的编写，以及各种各样的插件，还需要自己去探究。
