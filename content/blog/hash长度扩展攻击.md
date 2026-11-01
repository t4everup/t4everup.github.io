+++
title = 'hash长度扩展攻击'
date = '2023-08-13T15:13:00+08:00'
categories = ['学习', '安全']
tags = ['hash', '安全', '攻击']
+++

# hash长度扩展攻击

## hash算法

hash算法又叫散列算法，把任意长度的字符串加密为固定长度的字符串。

### hash算法流程

这里以md5算法进行解释。

#### 1.将数据分组

对我们输入的数据进行分组，每64字节为一组，也就是对数据进行取余，余数进行数据填充。

#### 2.填充数据

数据不够64字节，会进行数据填充，填充的第一个字节为hex(80)，这里填充原则为，对二进制，第一位取1，其他后面全部填充0，填充到56字节，这里的hex(80)也就是bin(10000000),填充到56字节。

#### 3.补充长度信息

后面8个字节为对长度进行描述，比如我们输入的数据为10字节，那么对长度的描述第57位填充为dec(10*8bit)-> hex(50)。

#### 4.加密运算

这里进行一些列的运算，用每一组的64字节数据对初始向量进行运算，并将运算结果作为下一次运算的初始值，如果没有下一次运算，该结果就是运算的散列值。这里具体运算过程还没进行学习。

计算时会有一个初始向量，

```
A 01 23 45 67 0x67452301
B 89 AB CD EF 0xEFCDAB89
C FE DC BA 98 0x98BADCFE
D 76 54 32 10 0x10325476
```

这里0x后面的数据是初始代码，然后与64字节数据进行第一次运算，初始向量被覆盖掉，然后再进行相同的运算，直到最后一步。最后一步的向量经过高低位互换后就是最后的hash值。

```
假如最后一轮的运算后的向量值为：
    A=0xabcdef12
    B=0xabcdef12
    C=0xabcdef12
    D=0xabcdef12
进行高低位互换之后得到的数值为：
12 ef cd ab
12 ef cd ab
12 ef cd ab
12 ef cd ab
```



### 举个例子

我们对nihao进行md5加密，分析一下流程
首先对nihao进行64取余，不够整除，余数为5，这里对数据进行填充。
![image-20230731152439453](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202307311524478.png)

这里先填充hex(80),然后向后填充0,到56位，然后 后8为对数据长度进行描述。
5*8bit -->hex(28)，在第57为填充28
![image-20230731152501317](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202307311525340.png)

下面就是用这64为数据与初始向量进行运算。

### 重点：

假如我们在数据后面再填充数据呢?
![image-20230731152953277](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202307311529308.png)

这时数据超过了64字节，也就是进行了之前的操作，进行数据填充，长度描述，运算，但是这里我们第一组运算的结果是一样的，也就是说我们在不知道前面数据的前提下，我们知道了md5运算的值，这样就能够在后面添加任意数据，已经知道了第一轮的向量结果，那么我们就能够推导出添加数据后的md5值，(网上有很多脚本)。

### 用法：

```
在 $hash = md5($secret.$key) 中已知 $hash 和 $key 以及 $secret 的长度时  
可以找到另一个 $_hash 和 $_key 使得 $_hash = md5($secrect.$_key) 成立

```

一道题目：

```php
 <?php
highlight_file(__FILE__);
include "./flag.php";
//$salt = XXXXXXXXXXXXXX // the salt include 14 characters
//md5($salt."adminroot")=e6ccbf12de9d33ec27a5bcfb6a3293df
@$username = urldecode($_POST["username"]);
@$password = urldecode($_POST["password"]);
if (!empty($_COOKIE["digest"])) {
    if ($username === "admin" && $password != "root") {
         if ($_COOKIE["digest"] === md5($salt.$username.$password)) {
            die ("The flag is ". $flag);
        }
        else {
            die ("Your cookies don't match up! STOP HACKING THIS SITE.");
        }
    }
    else {
        die ("no no no");
    }
}
```

这里知道了==md5($salt."adminroot")=e6ccbf12de9d33ec27a5bcfb6a3293df==，并且知道了salt长度14，可以使用长度扩展攻击，这里使用hashpump工具，
![image-20230731162704033](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202307311627081.png)

也就是生成的fa7ce51222866901a35d654e42cb87c6哈希值等于md5($salt."adminroot%80%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%b8%00%00%00%00%00%00%00aaaa")的哈希值，通过hash长度扩展攻击我们可以去伪造一个我们填充数据后的hash值，这里主要还是运算时的一些原理，这里我对hash算法不是很深入理解，就不再阐述。