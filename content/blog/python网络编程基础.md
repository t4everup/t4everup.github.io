+++
title = 'python网络编程基础'
date = '2023-11-12T03:28:05+08:00'
categories = ['学习', 'python']
tags = ['python']
+++

## TCP/IP协议

计算机之前进行网络通信，首先需要双方确认通信所使用的协议，之后才能按照这种协议进行数据的传输。

### tcp/ip协议的作用：

- `TCP/IP`是用于因特网(`Internet`)的通信协议。它是对计算机必须遵守的规则的描述，只有遵守这些规则，计算机之间才能进行通信。
  ![image-20231110173517740](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202311102028010.png)

- **IP - 网际协议**
- **TCP - 传输控制协议**
- **UDP - 用户数据报协议**

| 协议         | 功能用处           | 端口号 | 对应模块模块                     |
| :----------- | :----------------- | :----- | :------------------------------- |
| **`HTTP`**   | 网页访问           | `80`   | `httplib`、`urllib`、`xmlrpclib` |
| **`NNTP`**   | 阅读和张贴新闻文章 | `119`  | `nntplib`                        |
| **`FTP`**    | 文件传输           | `20`   | `ftplib`、`urllib`               |
| **`SMTP`**   | 发送邮件           | `25`   | `smtplib`                        |
| **`POP3`**   | 接收邮件           | `110`  | `poplib`                         |
| **`IMAP4`**  | 获取邮件           | `143`  | `imaplib`                        |
| **`Telnet`** | 命令行             | `23`   | `telnetlib`                      |
| **`Gopher`** | 信息查找           | `70`   | `gopherlib`、`urllib`            |

### TCP和UDP:

- **UDP 在传送数据之前不需要先建立连接**，远程主机在收到 UDP 报文后，不需要给出任何确认。
- 适用于一些及时性的服务

- TCP 提供**面向连接**的服务。在传送数据之前必须先建立连接，数据传送结束后要释放连接。
- TCP **一般用于文件传输、发送和接收邮件、远程登录等场景**。

### 三次握手:

![image-20231110174327973](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202311102028052.png)

复习一下字符的含义:

- `SYN`：连接请求/接收 报文段
- `seq`：发送的第一个字节的序号
- `ACK`：确认报文段
- `ack`：确认号。希望收到的下一个数据的第一个字节的序号

1. 客户端向服务器发送一个SYN报文请求。
2. 服务器收到客户端的SYN报文后，返回一个SYN报文作为回应，并同时发出ack确认。
3. 针对服务器返回的确认，客户端也发送ACK进行应答。

为什么要进行第三次握手呢？
通信就是数据的发送与接收，第三次主要目的就是双方确认都能够正常发送和接收。



### 四次挥手(四次握手):

建立一个TCP连接需要三次握手，而释放一个TCP连接需要四次挥手。
TCP的半关闭特征:提供了连接的一端在结束它的发送后还能接收来自另一端数据的能力
![image-20231110180709726](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202311102028044.png)

复习一下符号:

- `FIN` ：连接终止位
- `seq`：发送的第一个字节的序号
- `ACK`：确认报文段
- `ack`：确认号。希望收到的下一个数据的第一个字节的序号

假设客户端先关闭连接：

1. 客户端发送一个FIN报文，主动关闭TCP连接。客户端处于FIN_WAIT1状态，等待服务器的确认。
2. 服务器收到FIN报文后，返回一个ACK确认报文。此时TCP处于半关闭状态，客户端到服务器的连接释放。
3. 如果服务器想断开了，像客户端一样，发送一个FIN报文，等待客户端的回应。
4. 客户端收到FIN之后，发送一个ACK报文应答。

## 套接字(Socket)

socker是计算机进程间通信的一种实现。不同主机甚至同台主机不同进程之间的通信。其包含进行网络通信必需的五种信息：连接使用的协议，本地主机的 IP 地址，本地进程的协议端口，对方主机的 IP 地址，对方进程的协议端口。
套接字:ip+端口。

其实Socket 就是在应用层和传输层之间的一个抽象层。
![image-20231110193300747](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202311102028058.png)

### socket分类:

流格式套接字(SOCK_STREAM)

- TCP通信协议，流式传输
- 面向连接的可靠的传输
- 顺序传输
- 建立虚链路
- 数据的发送和接收不同步

数据报格式套接字（SOCK_DGRAM）

- UDP通信协议，数据报传输
- 无连接的不可靠的传输
- 不使用传输层协议，直接和底层进行数据传输，如IP
- 限制每次传输的数据大小
- 数据的发送和接收是同步的

### socker通信过程：

对于网站，通信模型是服务器与客户端之间的通信。两边都创建一个socket对象，然后通过socket对象对数据进行传输。通常服务端处在一个循环的状态，等待客户端的连接。

![image-20231110202503262](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202311102028821.png)

在python中，网络编程主要用的就是socket模块。

```python
#socket_family: 主要包含AF_INET、AF_INET6、AF_UNIX
#socket_type: 主要包含SOCK_STREAM、SOCK_DGRAM、SOCK_RAW
#protocol: 用于指定创建套接字的协议，通常默认省略值为0

socket(socket_family, socket_type, protocol=0)
# 创建TCP套接字
s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
```

#### 服务端过程：

```python
#server
import socket
# 创建 socket 对象
s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
# 绑定端口
s.bind(('localhost', 9999))
# 设置最大连接数，超过后排队
s.listen(5)
while True:
    # 建立客户端连接
    c, addr = s.accept()
    print('收到连接:', addr)
    # 发送数据
    c.send('谢谢你的连接'.encode())
    print(c.recv(1024).decode())
    # 关闭连接
    c.close()
```

调用accept()时，socket会进入waiting状态。客户端请求连接时，建立连接并返回服务器。`accept()` 返回一个含有两个元素的元组 (conn, addr)。第一个元素 conn 是新的 Socket 对象，服s务器必须通过它与客户端通信；第二个元素 addr 是客户端的 IP 地址及端口。

#### 客户端过程：

```python
#client
import socket
# 创建 socket 对象
s = socket.socket()
# 连接到服务器
s.connect(('localhost', 9999))
# 接收数据
s.send('我已经连接上你啦'.encode())
print (s.recv(1024).decode())
s.close()
```

客户端过程比较简单，创建一个socket对象，然后连接服务器，发送数据，读取数据。

![image-20231111143640082](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202311111436140.png)

![image-20231111143645831](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202311111436857.png)

#### **Socket 模块的方法**

- **服务器端套接字**

| 套接字方法              | 解释说明                                                     |
| :---------------------- | :----------------------------------------------------------- |
| **`s.bind(address)`**   | 将地址`(host, port)`绑定到套接字；在`AF_INET`下以元组的形式表示地址 |
| **`s.listen(backlog)`** | 开始监听`TCP`传入的连接；`backlog`指定在拒绝连接之前，操作系统可以挂起的最大连接数量；该值至少为`1`，大部分应用程序设为`5`就可以了 |
| **`s.accept()`**        | 被动接受`TCP`客户端的连接并返回一个元组`(conn, address)`；其中`conn`是新的套接字对象，可以用来接收和发送数据；`address`是连接客户端的地址 |

- **客户端套接字**

| 套接字方法                  | 解释说明                                                     |
| :-------------------------- | :----------------------------------------------------------- |
| **`s.connect(address)`**    | 连接到`address`服务器的套接字；一般`address`的格式为元组`(host, port)`；如果连接出错，则返回`socket.error`错误 |
| **`s.connect_ex(address)`** | 功能和`connect()`函数一样；但成功的时候返回数字码`0`，而出错时返回出错码，而不是抛出异常 |

- **公共用途的套接字函数**

| 套接字方法                                    | 解释说明                                                     |
| :-------------------------------------------- | :----------------------------------------------------------- |
| **`s.recv(bufsize[, flag])`**                 | 接收`TCP`套接字的数据，数据以字符串形式返回；`bufsize`指定要接收的最大数据量；`flag`提供有关消息的其他信息，通常可以忽略 |
| **`s.send(string[, flag])`**                  | 发送`TCP`套接字的数据，将`string`中的数据发送到连接的套接字；返回值是要发送的字节数量，该数量可能小于`string`的字节大小 |
| **`s.sendall(string[, flag])`**               | 完整发送`TCP`套接字的数据，将`string`中的数据发送到连接的套接字；但在返回之前会尝试发送所有数据，成功返回`None`，失败则抛出异常 |
| **`s.recvfrom(bufsize[, flag])`**             | 接收`UDP`套接字的数据，与`recv()`类似，但返回值是元组`(data, address)`；其中`data`是包含接收数据的字符串；`address`是发送数据的套接字地址 |
| **`s.sendto(string[, flag], address)`**       | 发送`UDP`套接字的数据，将数据发送到指定的`address`套接字；`address`是形式为`(ipaddr，port)`的元组，指定远程地址，返回值是发送的字节数 |
| **`s.close()`**                               | 关闭套接字                                                   |
| **`s.getpeername()`**                         | 返回套接字的远程地址，返回值通常是一个包含`(ipaddr,port)`的元组 |
| **`s.getsockname()`**                         | 返回套接字自己的地址，返回值通常是一个包含`(ipaddr,port)`的元组 |
| **`s.setsockopt(level, optname, value)`**     | 设置给定套接字选项的值                                       |
| **`s.getsockopt(level, optname [, buflen])`** | 返回套接字选项的值                                           |
| **`s.settimeout(timeout)`**                   | 设置套接字操作的超时时间；`timeout`是一个浮点数，单位是秒，值为`None`表示永不过期；一般超时期应该在刚创建套接字时设置，因为它们可能用于连接的操作，如`s.connect()` |
| **`s.gettimeout()`**                          | 返回当前超时期的值，单位是秒，如果没有设置超时期，则返回`None` |
| **`s.fileno()`**                              | 返回套接字的文件描述符                                       |
| **`s.setblocking(flag)`**                     | 如果`flag`为`0`，则将套接字设为非阻塞模式，否则将套接字设为阻塞模式(默认值)；非阻塞模式下，如果调用`recv()`没有发现任何数据，或`send()`调用无法立即发送数据，那么将引起`socket.error`异常 |
| **`s.makefile()`**                            | 创建一个与该套接字相关连的文件                               |

### TCP编程:

上面分析客户端和服务端通信的过程就是TCP的一个例子。

- `TCP`协议是面向连接的，会事先建立好连接，所以不需要指定地址

- `TCP`协议是面向连接的，会事先建立好连接，所以不需要指定地址

![image-20231111165539434](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202311111655568.png)

### UDP编程:

- `UDP`是面向无连接的，所以每次发送都需要指定发送给谁

```python
#server
import socket
# 创建和绑定套接字，不需要监听连接
s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
s.bind(('127.0.0.1', 9999))
while True:
    # 不需要接收连接，直接接收数据
    data, addr = s.recvfrom(1024)
    if data.decode() == 'exit':
       exit()
    print(f'连接的主机:{addr}.')
    print(f'收到的消息:{data}.')
    # 服务端给客户端发送数据
    s.sendto(bytes(f'{data}', 'utf-8'), addr)
# 关闭服务器套接字
s.close()
```

```python
#client
import socket
HOST = '127.0.0.1'
PORT = 9999
# 不需要使用connect方法连接到服务器
s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
while True:
    msg = input('输入要发送给服务器的数据: ')
    s.sendto(bytes(msg, 'utf-8'), (HOST, PORT))#发送数据给服务器
    data = s.recvfrom(1024)#接收服务器发来的数据
    print(f'服务器发来的数据: {data}')
```

![image-20231111193116278](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202311111931434.png)

![image-20231111193131914](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202311111931949.png)