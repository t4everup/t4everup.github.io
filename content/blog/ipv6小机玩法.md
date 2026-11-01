+++
title = 'ipv6小机玩法'
date = '2024-04-16T11:11:43+08:00'
categories = ['那些事', '教程']
+++

## 前言

前几天免费嫖到了3台只有ipv6的小机，那就玩玩吧- -

第一台使用xui搭建了梯子（网速可以 非常稳定）

第二台准备搞一个内网穿透服务，之前没接触过这个，掉了好多坑，最后还好搞定了，最近准备在宿舍搞台windows服务器（垃圾佬折腾），然后到时候内网穿透访问这台windows（windows要有ipv6）

## ipv6配置域名解析

首先就是域名解析ipv6，刚开始一直解析不了，后来经过一天的折腾，终于配好了0~0

我这里使用的cloudflare进行解析，开启cdn这样ipv4和ipv6都能访问（还是非常有意思的）。解析记得选择AAAA

![image-20240320235851157](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202403202358248.png)

这样就能正常访问了

## ipv6使用frp进行内网穿透

### 环境

ipv6服务器     frps

windows         frpc

这里主要的问题就是端口的规划和配置文件的编写。

这里使用nginx

```shell
sudo apt install nginx
```

安装frp

```php
FRP服务端一键安装脚本：

wget https://raw.githubusercontent.com/MvsCode/frps-onekey/master/install-frps.sh -O ./install-frps.sh
chmod 700 ./install-frps.sh
./install-frps.sh install

卸载命令：
./install-frps.sh uninstall

更新FRP命令：
./install-frps.sh update

FRP常用命令：
开启FRP
frps start

停止FRP：
Frps stop

重启FRP
frps restart
```

这里在安装frp向导时候，直接将http端口设置为80 https为443，后边好用，其他都默认即可。

访问后台6443地址记得填写ipv6的地址，正常没有ipv4的机子安装他会给你分配一个ipv4但是我们通过这个地址访问不到6443，我们通过 ipv6:6443 去访问后台

## nginx配置

将监听端口改为 9999，因为80，443 我们frp已经占用。

```php
 /etc/nginx/sites-enabled/default
```

![image-20240321000819419](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202403210008454.png)



这个文件夹下写我们的.conf，

```php
/etc/nginx/sites-available/
```

![image-20240321000955091](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202403210009126.png)

### proxy3.conf

```php
server {
    listen 8000 ssl;
   # listen 8080 ssl;
    server_name blog3.xxxx.top; #您的域名或服务器IP
    location / {
        proxy_redirect off;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_pass http://[2602:xxxx:0:xxxx::d9b2]; #您要代理的v6地址+端口
    }
}
```

这样我们在访问域名的时候，默认解析到ipv6的80端口，也就是我们设置的frp的http
需要多少个服务 写多少个配置文件就行啦。注意二级域名

### nginx.conf

![image-20240321002323408](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202403210023447.png)

这里记得包含我们的.conf文件

### 重新加载nginx

```php
systemctl reload nginx
```



## https配置

windows配置

这里使用frp配置https

```php
[common]

server_addr = 2602:xxxx:0:xxxx::d9b2 #ipv6

server_port = 5443 //frp设置的默认服务器端口

token = xxxxxx //

[web1_xxxxx] //随便写
type = https//https

local_ip = 10.207.234.43//windows的ip （内网）

local_port = 8000 //本地开启服务的端口

custom_domains = v6test.xxxx.top //绑定的域名

plugin = https2http  #将 https请求转换成http请求后再发送给本地服务
plugin_local_addr = 10.207.234.43:8000   #转换http后的端口 
#证书相关配置
plugin_crt_path = ./https.crt #证书路径 windows为crt后缀
plugin_key_path = ./https.key #证书路径
plugin_host_header_rewrite = 127.0.0.1
plugin_header_X-From-Where = frp
```

到此就能使用https + 域名 访问到windows内网的服务了

```js
frpc.exe -c ./frpc.toml
```

![image-20240321001610420](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202403210016511.png)

后续

因为昨天测试是在本机有ipv6的环境下测试的，然而我们学校校园网并没给ipv6，这个时候我在windows启动frpc的时候肯定是连不上我们的ipv6的服务端frps的，这个时候我们使用cloudflare warp，这样我们本机就能开启frpc啦。

关于cloudflare warp使用 [教程](https://www.youtube.com/watch?v=4P_UByoJE1I)

## 垃圾佬组装电脑

cpu：我5年级家里给组装的电脑换下的 奔腾G2010
电源：换下的电源。能用就行
主板：pdd买的 H61 随机发货 到货七彩虹的
内存：4*2 8G
硬盘：机械500g 速度还好。。

![img](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202404161659140.jpg)

本来准备搞一个鞋盒机箱，但想到学校可能会查危险隐患，所以pdd买了个开放式机箱。
![img](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202404161701502.jpg)

![img](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202404161701704.jpg)到处 这个机器就组装完毕了，按照之前frp穿透，也是没有任何问题的。

## 搭建MC服务器

这里客户端使用tcp协议

![image-20240416170419119](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202404161704146.png)

然后将本机的25565映射出去，到服务器的9998端口，然后在dns加入解析。最后通过mc.xxxx.fun:9998就能加入游戏啦。

![img](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202404161705989.png)
