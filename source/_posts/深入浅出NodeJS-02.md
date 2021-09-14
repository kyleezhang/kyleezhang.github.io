---
title: 《深入浅出NodeJS》读书笔记（二）
date: 2021-09-10 08:13:07
toc: true
mathjax: false
categories: 
- 读书笔记
tags:
- Node
- 读书笔记
---

## 七、网络编程

### 1、构建 TCP 服务

TCP 全名传输控制协议，在 OSI 模型中属于传输层协议。具体如下图所示：

<img src="/assets/深入浅出NodeJS/12.png">

TCP 是面向连接的协议，其显著的特征是在传输之前需要 3 次握手形成会话，只有会话形成后，服务端和客户端之间才能互相发送数据。在创建会话的过程中，服务器端和客户端分别提供一个套接字，这两个套接字共同形成一个连接。服务器端与客户端则通过套接字实现两者之间的连接操作。

#### (1)创建 TCP 服务器端

我们可以通过 net.createServer(listener) 即可创建一个 TCP 服务器，listener 是连接事件 connection 的侦听器，举个例子：

```js
var net = require("net")

var server = net.createServer(function (socket) {
    socket.on('data', function (data) {
        socket.write("你好")
    })

    socket.on('end', function () {
        console.log("连接断开")
    })

    socket.write("欢迎光临《深入浅出NodeJS》")
})

server.listen(8124, function () {
    console.log('server bound')
})
```

我们可以利用 Telnet 工具作为客户端与服务器进行会话交流:

```sh
telent 127.0.0.1 8124
```

除了端口外我们也可以对 Domain Socket 进行监听：

```js
server.listen('tmp/echo.sock')
```

Domain Socket 我们可以通过 nc 工具进行会话测试：

```sh
nc -U /tem/echo.sock
```

除了使用上述工具外我们还可以自己创建客户端：

```js
var net = require("net")

var client = net.connect({ port: 8124 }, function () {
    console.log('client connected');
    client.write('world')
})

client.on('data', function (data) {
    console.log(data.toString())
    client.end()
})

client.on('end', function () {
    console.log('client disconnected')
})
```

执行上述客户端代码文件与使用 Telnet 和 nc 的会话结果别无差别，如果是 Domain Socket 在填写选项时，填写 path 即可：

```js
var client = net.connect({ path: '/tmp/echo.sock' })
```

#### (2)TCP 服务的事件

**服务器事件**

对于通过 net.createServer() 创建的服务器而言，它是一个 EventEmitter 实例，它的自定义事件有如下几种。

- listening：在调用 server.listen() 绑定端口或者 Domain Socket 后触发，简洁的写法为 server.listen(port, listeningListener)，通过 listen 方法的第二个参数传入。
- connection：每个客户端套接字连接到服务器端时触发，简洁的写法为通过 net.createServer()，最后一个参数传递。
- close：当服务器关闭时触发，在调用 server.close() 后，服务器将停止接受新的套接字连接，但保持当前存在的连接，等待所有连接都断开后会触发该事件。
- error：当服务器发生异常时，将会触发该事件。

**连接事件**

服务器可以同时与多个客户端保持连接，每个连接都是典型的可写可读的 Stream 对象，它具有如下自定义事件：

- data：当一端调用 write() 发送数据时，另一端会触发 data 事件，事件传递的数据即是 write() 发送的数据。
- end：当连接中的任意一端发送了 FIN 数据时将会触发该事件。
- connect：该事件用于客户端，当套接字与服务器端连接成功时会触发该事件。
- drain：当任意一段调用 write() 发送数据时，当前这端会触发该事件。
- error：当异常发生时，触发该事件。
- close： 当套接字完全关闭时，触发该事件。
- timeout：当一定时间后连接不再活跃时，该事件将会被触发，通知该用户连接已经被闲置了。