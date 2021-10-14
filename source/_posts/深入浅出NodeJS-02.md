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


### 2、构建 UDP 服务

UDP 又称用户数据包协议，提供面向事务的简单不可靠信息传输服务，与 TCP 一样同属于网络传输层。其与 TCP 最大的不同在于 UDP 不是面向连接的。TCP 中连接一旦建立所有的会话都基于连接完成，客户端如果要与另一个服务端通信必须新建一个套接字来完成连接。但在 UDP 中一个套接字可以与多个 UDP 服务通信。

#### (1)创建 UDP 服务器端

```js
var dgram = require('dgram')

const server = dgram.createSocket("udp4")

server.on("message", function (msg, rinfo) {
    console.log("server got:" + msg + "from" + rinfo.address + ":" + rinfo.port)
})

server.on("lessoning", function () {
    var address = server.address
    console.log("server listening" + ":" + address.port)
})

server.bind(41234)
```

如上述示例所示，创建 UDP 服务端的核心在于创建 UDP 套接字，UDP 套接字一旦创建既可以作为客户端发送数据，也可以作为服务器端接收数据。想要让 UDP 套接字接收网络信息只需要调用 `dgram.bind(port, [address])` 方法对网卡和端口进行绑定即可。

#### (2)创建 UDP 客户端

```js
var dgram = require("dgram")
var message = new Buffer("深入浅出NodeJS")
const client = dgram.createSocket("udp4")
client.send(message, 0, message.length, 41234, "localhost", function (err, bytes) {
    client.close()
})
```

当使用套接字作为客户端时可以调用 send 方法发送消息到网络中，send 方法的参数如下：

```js
socket.send(buf, offset, length, port, address, [callback])
```

与 TCP 连接的 write 方法相比 send 方法的参数列表要复杂的多，但是它更灵活的地方在于可以随意发送数据到网络中的服务器端。

#### (3)UDP套接字事件

UDP 的套接字与 TCP 的不同，它只是 EventEmitter 实例，而非 Stream 实例，它具备如下自定义事件：

- message: 当 UDP 套接字侦听网卡端口后，接收到消息时触发该事件，触发携带的事件为消息 Buffer 对象和一个远程地址信息。
- listening: 当 UDP 套接字开始侦听时触发该事件。
- close: 调用 close 方法时触发该事件，并不再触发 message 事件，如需再次触发 message 事件重新绑定即可。
- error: 当异常发生时触发该事件，如果不侦听异常抛出，使进程退出。

### 3、构建 HTTP 服务

HTTP 的全称是超文本传输协议，HTTP 构建于 TCP 之上，属于应用层协议。从协议的角度来说现在的应用如浏览器，其实是一个 HTTP 的代理，用户的行为将会转化为 HTTP 报文发送给服务器端。服务器端在处理请求后，发送响应报文给代理，因此 HTTP 服务只做两件事：处理 HTTP 请求和发送 HTTP 响应。

#### (1)http 模块

Node 的 http 模块包含了对于 HTTP 处理的封装。在 Node 中 HTTP 服务继承自 TCP 服务器（net 模块），它能够与多个客户端保持连接，由于其采用事件驱动的形式，并不为每一个连接创建额外的线程或进程，保持很低的内存占用，所以能实现高并发。HTTP 服务与 TCP 服务有区别的地方在于，在开启了 keep-alive 之后，一个 TCP 会话可以用于多次请求和响应。TCP 以 connection 为单位进行服务，HTTP 服务以 request 为单位进行服务，http 模块是将 connection 到 request 的过程进行了封装。

除此之外，http 模块将连接所用的套接字读写抽象为 ServerRequest 和 ServerResponse 对象，它们分别对应请求和响应操作。在请求产生的过程中，http 模块拿到连接中传来的数据，调用二进制模块 http_parser 进行解析，在解析完请求报文的报头后，触发 request 事件，调用用户的业务逻辑，流程如下图所示：

<img src="/assets/深入浅出NodeJS/13.png">

**http 请求**

对于 TCP 连接的读操作，http 模块将其封装为 ServerRequest 对象。请求报文通过 http_parser 进行解析，报文头被解析为如下部分：

- req.method 属性: 标识请求方法，常见的请求方法有 GET、POST、DELETE等。
- req.url 属性: 请求 url 路径
- req.httpVersion属性: 请求的 http 版本，通常为 1.1
- 其余报头被解析为很规律的 `Key: Value` 格式，被解析后放置在 req.headers属性上

报文体部分则被抽象为一个只读流对象，如果业务逻辑需要读取报文体中的数据，则要在这个数据流结束后才能进行操作，如下所示：

```js
function (req, res) {
    var buffers = [];
    req.on('data', function (trunk) {
        buffers.push(trunk)
    }).on ('end', function () {
        var buf = Buffer.concat(buffers);
        res.end('Hello world')
    })
}
```

**http 响应**

HTTP 响应相对简单一些，它封装了对底层连接的写操作，可以将其看成一个可写的流对象。它影响响应报文头部信息的 API 为 res.setHeader() 和 res.writeHeader()。可以调用 setHeader 进行多次设置，但只有调用了 writeHeader 后，报头才会写入到连接中，除此之外，http 模块会自动帮你设置一些头信息，如Date、Connection 等。

报文体部分则是调用 res.write() 和 res.end() 方法实现，后者与前者的区别在于 res.end() 会先调用 write() 发送数据，然后发送信号告知服务器这次响应结束。

**http 服务的事件**

HTTP 服务器也是 EventEmitter 的实例，实现的自定义事件列表如下所示：

- connection 事件: 在开始 HTTP 请求和响应前，客户端与服务端需要建立底层的 TCP 连接，这个连接可能因为开启了 keep-alive，可以在多次请求响应之间使用；当这个连接建立时，服务器触发一次 conection 事件。
- request 事件: 建立 TCP 连接后，http 模块底层将在数据流中抽象出 HTTP 请求和 HTTP 响应，当请求数据发送到服务器端，在解析出 HTTP 请求头后，将会触发该事件；在 res.end() 后，TCP 连接可能将用于下次请求响应。
- data 事件: 与 TCP 服务器的行为一致，调用 server.close() 方法停止接受新的连接，当已有的连接都断开时触发该事件。
- checkContinue 事件: 某些客户端在发送较大的数据时，并不会将数据直接发送，而是先发送一个头部为带 `Except：100-continue` 的请求到服务器，服务器将会触发 checkContinue 事件；如果没有为服务监听这个事件，服务器将会自动响应客户端 100 Continue 状态码，表示接受数据上传；如果不接受较多的数据时，响应客户端 400 Bad Request 拒绝客户端继续发送数据即可。需要注意的是，当该事件发生时不会触发 request 事件，两个事件之间互斥。当客户端收到 100 Continue 后重新发起请求时，才会触发 request 事件。
- connect事件: 当客户端发起 CONNECT 请求时触发，而发起 CONNECT 请求通常在 HTTP 代理时出现；如果不监听该事件，发起该请求的连接将会关闭。
- upgrade 事件: 当客户端要求升级连接的协议时，需要和服务器协商，客户端会在请求头中带上 Upgrade 字段，服务器端会在接收到这样的请求时触发该事件，如果不监听该事件，发起请求的连接将会关闭。
- clientError 事件: 连接的客户端触发 error 事件时，这个错误会传递到服务器端，此时触发该事件。

#### (2)http 客户端

http 模块提供了一个底层 API：`http.request(options, connect)` 用于构造 HTTP 客户端，其中 options 参数决定了这个 HTTP 请求头中的内容，它的选项有如下这些：

- host: 服务器的域名或 IP 地址，默认 localhost。
- hostname: 服务器名称。
- port: 服务器端口，默认 80。
- localAddress: 建立网络连接的本地网卡
- socketPath: Domain 套接字路径。
- method: HTTP请求方法。默认 GET。
- path: 请求路径，默认为 /。
- headers: 请求头对象。
- auth: Basic 认证，这个值被计算成请求头中的 Authorization 部分。

报文体的内容由请求对象的 write() 方法和 end() 方法实现：通过 write() 方法向连接中写入数据，通过 end() 方法告知报文结束。它与浏览器中的 Ajax 调用几近相同，Ajax 的实质就是一个异步的网络 HTTP 请求。

**http 响应**

HTTP 客户端的响应对象与服务器端较为类似，在 ClientRequest 对象中，它的事件叫做 response。ClientRequest 在解析响应报文时，一解析完响应头就触发 response 事件，同时传递一个响应对象以供操作 ClientResponse。后续响应报文体以只读流的方式提供。

**http 代理**

如同服务器端的实现一般，http 提供的 ClientRequest 对象也是基于 TCP 层实现的，在 keep-alive 的情况下，一个底层会话连接可以多次用于请求。为了重用 TCP 连接，http 模块包含一个默认的客户端代理对象 http.globalAgent。它对每个服务器端（host + port）创建的连接进行了管理，默认情况下，通过 ClientRequest 对象对同一个服务器端发起的 HTTP 请求最多可以创建 5 个连接，其实质上就是一个连接池。

我们也可以通过自行构造代理对象来对限制数量进行修改：

```js
var agent = new http.agent({
    maxSockets: 10
})

var options = {
    hostname: 'localhost',
    port: 1234,
    path: '/',
    method: 'GET',
    agent: agent
}
```

除此之外还可以设置 agent 选项为 false 来脱离连接池的管理，使请求不受并发的限制。

Agent 对象的 sockets 和 requests 属性分别表示当前连接池中使用中的连接数和处于等待状态的请求数，在业务中监视这两个值有助于发现业务状态的繁忙程度。

**http 客户端事件**

- response: 与服务器端的 request 事件对应的客户端在请求发出后得到服务端响应时，会触发该事件。
- socket: 当底层连接池中建立的连接分配给当前请求对象时，触发该事件。
- connect: 当客户端向服务器端发起 Upgrate 请求时，如果服务器端响应了 101 Switching Protocal 状态，客户端将会触发该事件。
- continue: 客户端向服务器端发起 `Expect: 100-continue` 头信息，以试图发送较大数据量，如果服务器端响应 100 Continue 状态，客户端将触发该事件。

### 4、构建 WebSocket 服务

WebSocket 协议与 Node 之间的配合堪称完美，主要有两点：

- WebSocket 客户端基于事件的编程模型与 Node 中自定义事件相差无几。
- WebSocket 实现了客户端与服务器端之间的长连接，而 Node 事件驱动的方式十分擅长与大量客户端保持高并发连接。

除此之外 WebSocket 相较于 HTTP 还有如下好处：

- 客户端与服务器端只建立一个 TCP 连接，可以使用更少的连接。
- WebSocket 服务器端可以推送数据到客户端，这远比 HTTP 请求响应模式更灵活、更高效。
- 有更轻量级的协议头，减少数据传送量。

相比于 HTTP，WebSocket 更接近于传输层协议，它并没有在 HTTP 的基础上模拟服务器端的推送，而是在 TCP 上定义独立的协议，HTTP 协议只是完成了其握手部分。

#### (1)WebSocket 握手

客户端建立连接时，通过 HTTP 发起请求报文，如下所示：

```
GET /chat HTTP/1.1
Host: server.example.com
Upgrate: websocket
Connection: Upgrate
Set-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebScoket-Protocal: chat, superchat
Sec-WebSocket-Version: 13 
```

与普通的 HTTP 请求协议略有区别的部分在于如下协议头：

```
Upgrate: websocket
Connection: Upgrate
```

上述字段表示请求服务器端升级协议为 WebSocket。

Set-WebSocket-Key 的值是随机生成的 Base64 编码的字符串，主要用于安全校验，服务器端接收到之后将其与字符串 "258EAFA5-E914-47DA-95CA-C5AB0DC85B11" 相连形成字符串 "GhlIHNhbXBsZSBub25jZQ==258EAFA5-E914-47DA-95CA-C5AB0DC85B11"，然后通过 sha1 安全散列算法计算出结果后，再进行 Base64 编码，最后返回客户端。

而 Sec-WebScoket-Protocal 和 Sec-WebSocket-Version 字段主要用于指定子协议和版本号。

服务端在处理完请求后响应报文如下所示：

```
HTTP/1.1 101 Switching Protocols
Upgrate: websocket
Connection: Upgrate
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
Sec-WebSocket-Protocal: chat
```

上面的报文告知客户端正在更换协议，更新应用层协议为 WebSocket 协议，并在当前的套接字连接上应用新协议。剩余的字段分别表示服务器端基于 Sec-WebSocket-Key 生成的字符串和选中的子协议。客户端将会校验 Sec-WebSocket-Accept 的值，如果成功将开始接下来的传输。

我们以如下代码进行演示：

浏览器端：

```js
var WebSocket = function (url) {
    // 伪代码，解析 ws://127.0.0.1/updates，用于请求
    this.options = parseUrl(url)
    this.connect()
};

WebSocket.prototype.onopen = function () {
    // TODO
}
WebSocket.prototype.setSocket = function (socket) {
    this.socket = socket
}
WebSocket.prototype.connect = function () {
    var that = this;
    var key = new Buffer(this.options.protocalVersion + '-' + Date.now()).toString('base64')
    var shasum = crypto.createHash('sha1');
    var expected = shasum.update(key + '258EAFA5-E914-47DA-95CA-C5AB0DC85B11').digest('base64')

    var options = {
        port: this.options.port, // 12010
        host: this.options.hostname, // 127.0.0.1
        headers: {
            'Connection': 'Upgrade',
            'Upgrade': 'websocket',
            'Sec-WebSocket-Version': this.options.protocalVersion,
            'Sec-Websocket-Key': key
        }
    } 

    var req = http.request(options);
    req.end();

    req.on('upgrade', function (res, socket, upgradeHead) {
        // 连接成功
        that.setSocket(socket)
        // 触发 open 事件
        this.onopen()
    })
}
```

服务端：

```js
var server = http.createServer(function (req, res) {
    res.writeHead(200, { 'Content-Type': 'tet/plain' });
    res.end('Hello World\n')
})
server.listen(12010)

// 在收到 upgrade 请求后，告知客户端允许切换协议
server.on('upgrade', function (req, socket, upgradeHead) {
    var head = new Buffer(upgradeHead.length)
    upgradeHead.copy(head)
    var key = req.headers['sec-websocket-key']
    var shasum = crypto.createHash('sha1')
    key = shasum.update(key + '258EAFA5-E914-47DA-95CA-C5AB0DC85B11').digest('base64')
    var headers = [
        'HTTP/1.1 101 Switching Protocols',
        'Upgrade: websocket',
        'Connection: Upgrade',
        'Sec-WebSocket-Accept: ' + key,
        'Sec-WebSocket-Protocol: ' + protocol
    ]
    // 让数据立即发送
    socket.setNoDelay(true)
    socket.write(headers.concat("", "").join('\r\n'));
    // 建立服务器端 WebSocket 连接
    var websocket = new WebSocket()
    websocket.setSocket(socket)
})
```

#### (2)数据传输

在 WebSocket 协议中,数据传输阶段使用 frame（数据帧）进行通信，frame 分不同的类型，主要有：文本数据，二进制数据。出于安全考虑和避免网络截获，客户端发送的数据帧必须进行掩码处理后才能发送到服务器，不论是否是在 TLS 安全协议上都要进行掩码处理。服务器如果没有收到掩码处理的数据帧时应该关闭连接，发送一个 1002 的状态码。服务器不能将发送到客户端的数据进行掩码处理，如果客户端收到掩码处理的数据帧必须关闭连接。

那我们服务器端接收到的数据帧是怎样的呢？

<img src="/assets/深入浅出NodeJS/14.png">

**数据帧**

WebSocket 的数据传输是要遵循特定的数据格式-数据帧（frame）

<img src="/assets/深入浅出NodeJS/15.png">

每一列代表一个字节，一个字节8位，每一位又代表一个二进制数。

- fin: 标识这一帧数据是否是该分块的最后一帧。
  - 1 为最后一帧
  - 0 不是最后一帧。需要分为多个帧传输
- rsv1-3: 默认为0，接收协商扩展定义为非0设定。
- opcode： 操作码，也就是定义了该数据是什么，如果不为定义内的值则连接中断。占四个位，可以表示0~15的十进制，或者一个十六进制。
  - %x0 表示一个继续帧
  - %x1 表示一个文本帧
  - %x2 表示一个二进制帧
  - %x3-7 为以后的非控制帧保留
  - %x8 表示一个连接关闭
  - %x9 表示一个ping
  - %x10 表示一个pong
  - %x11-15 为以后的控制帧保留
- masked： 占第二个字节的一位，定义了masking-key是否存在。并且使用masking-key掩码解析Payload data。
  - 1 客户端发送数据到服务端
  - 0 服务端发送数据到客户端
- payload length： 表示Payload data的总长度。占7位，或者7+2个字节、或者7+8个字节。
  - 0-125，则是payload的真实长度
  - 126，则后面2个字节形成的16位无符号整型数的值是payload的真实长度，125<数据长度<65535
  - 127，则后面8个字节形成的64位无符号整型数的值是payload的真实长度，数据长度>65535
- masking key： 0或4字节，当masked为1的时候才存在，为4个字节，否则为0，用于对我们需要的数据进行解密
- payload data： 我们需要的数据，如果masked为1，该数据会被加密，要通过masking key进行异或运算解密才能获取到真实数据。

**关于数据帧**

因为 WebSocket 服务端接收到的数据有可能是连续的数据帧，一个 message 可能分为多个帧发送。但如果使用 fin 来做消息边界是有问题的。

我发送了一个 27378 个字节的字符串，服务器端共接收到 2 帧，两帧的 fin 都为 1，而且根据规范计算出来的两帧的 payload data 的长度为 27372 少了 6 个字节。这缺少的 6 个字节其实刚好等于 2 个固有字节加上 maskingKey 的4个字节，也就是说第二帧就是一个纯粹的数据帧。这又是怎么回事呢？？

从结果推测实现，我们接收到的第 2 帧的数据格式不是帧格式，说明数据没有先分帧（分片）后再发送的。而是将一帧分包后发送的。

> 分片
分片的主要目的是允许当消息开始但不必缓冲该消息时发送一个未知大小的消息。如果消息不能被分片，那么端点将不得不缓冲整个消息以便在首字节发生之前统计出它的长度。对于分片，服务器或中间件可以选择一个合适大小的缓冲，当缓冲满时，写一个片段到网络。

我们 27378 个字节的消息明显是知道 message 长度，那么就算这个 message 很大，根据规范 1 帧的数据长度理论上是 0 < 数据长度 < 65535 的，这种情况下应该 1 帧搞定，他也只是当做一帧来发送，但是由于传输限制，所以这一个帧（我们收到的像是好几帧一样）会被拆分成几块发送，除了第一块是带有 fin、opcode、masked 等标识符，之后收到的块都是纯粹的数据（也就是第一块的 payload data 的后续部分），这个就是 socket 的将 WebSocket 分好的一帧数据进行了分包发送。那么这种一帧被 socket 分包发送，导致像是分帧（分片）发送的情况（服务器端本应该只就收一帧），在服务器端我暂时还没有想到怎样获取状态来处理。

总结，客户端发送数据，在实现时还是需要手动进行分帧（分片）,不然就按照一帧发送，小数据量无所谓；如果是大数据量，就会被 socket 自动分包发送。这个与 WebSocket 协议规范所标榜的自动分帧（分片），存在的差异应该是各个浏览器在对 WebSocket 协议规范的实现上偷工减料所造成的。所以我们看见 socket.io 等插件会有一个客户端接口，应该就是为了重新是实现 WebSocket 协议规范。从原理出发，我们接下来还是以小数据量（单帧）数据传输为例了。

**解析数据帧**

```js
//dataHandler.js
// 收集本次message的所有数据
getData(data, callback) {
    this.getState(data);
    // 如果状态码为8说明要关闭连接
    if(this.state.opcode == 8) {
        this.OPEN = false;
        this.closeSocket();
        return;
    }
    // 如果是心跳pong,回一个ping
    if(this.state.opcode == 10) {
        this.OPEN = true;
        this.pingTimes = 0;// 回了pong就将次数清零
        return;
    }
    // 收集本次数据流数据
    this.dataList.push(this.state.payloadData);

    // 长度为0，说明当前帧位最后一帧。
    if(this.state.remains == 0){
        let buf = Buffer.concat(this.dataList, this.state.payloadLength);
        //使用掩码maskingKey解析所有数据
        let result = this.parseData(buf);
        // 数据接收完成后回调回业务函数
        callback(this.socket, result);
        //重置状态，表示当前message已经解析完成了
        this.resetState();
    }else{
        this.state.index++;
    }
}
  
// 收集本次message的所有数据
getData(data, callback) {
    this.getState(data);

    // 收集本次数据流数据
    this.dataList.push(this.state.payloadData);

    // 长度为0，说明当前帧位最后一帧。
    if(this.state.remains == 0){
        let buf = Buffer.concat(this.dataList, this.state.payloadLength);
    //使用掩码maskingKey解析所有数据
        let result = this.parseData(buf);
    // 数据接收完成后回调回业务函数
        callback(this.socket, result);
    //重置状态，表示当前message已经解析完成了
        this.resetState();
    }else{
        this.state.index++;
    }
}

// 解析本次message所有数据
parseData(allData, callback){
    let len = allData.length,
        i = 0;
    for(; i < len; i++){
        allData[i] = allData[i] ^ this.state.maskingKey[ i % 4 ];// 异或运算，使用maskingKey四个字节轮流进行计算
    }
    // 判断数据类型，如果为文本类型
    if(this.state.opcode == 1) allData = allData.toString();

    return allData;
}
```

**组装需要发送的数据帧**

```js
// 组装数据帧，发送是不需要掩码加密
createData(data){
    let dataType = Buffer.isBuffer(data);// 数据类型
    let dataBuf, // 需要发送的二进制数据
        dataLength,// 数据真实长度
        dataIndex = 2; // 数据的起始长度
    let frame; // 数据帧

    if(dataType) dataBuf = data;
    else dataBuf = Buffer.from(data); // 也可以不做类型判断，直接Buffer.form(data)
    dataLength = dataBuf.byteLength; 
    
    // 计算payload data在frame中的起始位置
    dataIndex = dataIndex + (dataLength > 65535 ? 8 : (dataLength > 125 ? 2 : 0));

    frame = new Buffer.alloc(dataIndex + dataLength);

    //第一个字节,fin = 1,opcode = 1
    frame[0] = parseInt(10000001, 2);

    //长度超过65535的则由8个字节表示，因为4个字节能表达的长度为4294967295，已经完全够用，因此直接将前面4个字节置0
    if(dataLength > 65535){
        frame[1] = 127; //第二个字节
        frame.writeUInt32BE(0, 2); 
        frame.writeUInt32BE(dataLength, 6);
    }else if(dataLength > 125){
        frame[1] = 126;
        frame.writeUInt16BE(dataLength, 2);
    }else{
        frame[1] = dataLength;
    }

    // 服务端发送到客户端的数据
    frame.write(dataBuf.toString(), dataIndex);

    return frame;
}
```

**心跳检测**

```js
// 心跳检查
sendCheckPing(){
    let _this = this;
    let timer = setTimeout(() => {
        clearTimeout(timer);
        if (_this.pingTimes >= 3) {
            _this.closeSocket();
            return;
        }
        //记录心跳次数
        _this.pingTimes++;
        if(_this.pingTimes == 100000) _this.pingTimes = 0;
        _this.sendCheckPing();
    }, 5000);
}
// 发送心跳ping
sendPing() {
    let ping = Buffer.alloc(2);
    ping[0] = parseInt(10001001, 2);
    ping[1] = 0;
    this.writeData(ping);
}
```

**关闭连接**

客户端直接调用close方法，服务器端可以使用socket.end方法。


## 八、构建 Web应用

### 1、基础功能

#### (1)请求方法

HTTP_Parser 在解析请求报文时会将报文头中的请求方法抽取出来设置为 req.method。

#### (2)路径解析

HTTP_Parser 将其解析为 req.url。一般而言，完整的 URL 地址是如下这样的：

```
http://user:pass@host.com:8080/p/a/t/h?query=string#hash
```

但是客户端代理（浏览器）会将这个地址解析成报文，将路径和查询报文放在报文第一行，如下所示：

```
GET /p/a/t/h?query=string HTTP/1.1
```

因此最终得到的 req.url 的值为 '/p/a/t/h?query=string'。

#### (3)查询字符串

查询字符串位于路径之后，形成请求报文首行的第二部分，Node 提供了 querystring 模块用于处理这部分数据。

#### (4)Cookie

HTTP 是一个无状态的协议，但现实中的业务却是需要一定的状态的，这就催生了 Cookie 的诞生，客户端发送的 Cookie 在请求报文的 Cookie 字段中，HtTP_Parser 会将所有的报文字段解析到 req.headers 上，Cookie 是 req.headers.cookie。具体内容格式如下所示：

```
'csrftoken=HpZuyoYbH_hOQWHUv-0bbb9H; Hm_lvt_f1621cb3fb0792bb294fce1b938d5eef=1632186900; Hm_lpvt_f1621cb3fb0792bb294fce1b938d5eef=1633872461'
```

除此之外我们还可以在服务端通过 Set-Cookie 字段设置客户端 Cookie，除了 name=value 是必须包含的部分还有如下几个主要参数配置项：

- path：表示这个 Cookie 影响到的路径。
- Expires 和 Max-Age 是用来告诉浏览器这个 Cookie 何时过期的，如果不设置这个该选项在关闭浏览器时会丢失掉这个 Cookie，如果设置了浏览器会把 Cookie 内容写入磁盘中并保存。
- HttpOnly：告知浏览器不允许通过脚本 document.cookie 来更改 cookie
- Secure：当 Secure 值为 true 时，在 HTTP 中是无效的，在 HTTPS 中才有效。

#### (5)Session

Session 的数据只保留在服务器端，客户端无法修改，这样数据的安全性得到一定的保障，数据也无须在协议中每次被传递，但是还需要将每个客户和服务器中的数据一一对应起来，这里主要有常见的两种实现方式：

- 基于 Cookie 来实现用户和数据映射。
- 通过查询字符串来实现浏览器端和服务器端数据的对应。

两种方法实现的思路主要是通过 Cookie 携带 Session 的口令或通过请求中的查询字符串来携带。

除此之外我们还需要注意如下亮点：

- Session 与内存：如果将 Session 数据存放在内存中一方面 Node 有内存大小限制，并且 Node 的进程与进程之间也无法共享内存，因此常用的方案是将 Session 集中化，转移到集中的数据存储中，常用的工具是 Redis、Memcached。
- Session 与安全：Session 的口令依然保存在客户端，因此会存在口令被盗用的情况，有一种做法是将这个口令通过私钥加密进行签名，这样即使攻击者知道口令也无法伪造签名信息，但是如果攻击者通过某种方式获取了一个真实的口令和签名他就能实现身份的伪装。因此另一种方案是将客户端的某些独有信息与口令作为原值进行签名，这样攻击者一旦不在原始的客户端访问就会导致签名失败。

#### (6)缓存

HTTP 支持的缓存策略主要分为 “强制缓存” 和 “协商缓存”。“强制缓存“ 主要通过 Expires 和 Cache-Control 字段，而 “协商缓存” 主要通过 Last-Modified 和 If-Modified-Since、ETag 和 If-None-Match，更多详情可查看我的博客[浏览器缓存](https://kyleezhang.github.io/2021/03/16/client-cache/)。

除了设置缓存之外我们还需要服务端意外更新后通过客户端及时更新的能力，这使得我们在使用缓存时也要为其设定版本号，所幸浏览器是根据 URL 进行缓存，那么一旦内容有更新时，我们就让浏览器发起新的 URL 请求，使得新内容能够被客户端更新。一般的更新机制有如下两种：

- 每次发布路径中跟随 Web 应用的版本号。
- 每次发布路径中跟随文件内容的 hash 值。

#### (7)Basic 认证

Basic 认证是当客户端与服务端进行请求时，允许通过用户名和密码实现的一种身份认证方式。如果一个页面需要 Basic 认证，它会检查请求报文头中的 Authorization 字段的内容，该字段的值由认证方式和加密值构成，如下所示：

```
Authorization: Basic dXNlcjpwYXNz
```

在 Basic 认证中，它会将用户和密码部分组合：username + ":" + password 。然后进行 base64 编码。

Basic 认证虽然经过 Base64 加密后在网络传送，但是这近乎明文十分危险，一般只有在 HTTPS 的情况下才会使用，不过 Basic 认证的支持范围十分广泛，几乎所有的浏览器都支持它。

### 2、数据上传

Node 的 http 模块只对 HTTP 报文的头部进行了解析，然后触发了 request 事件。如果请求中还带有内容部分，内容部分需要用户自行接收和解析。

我们可以通过报头中的 Transfer-Encoding 或 Content-Length 字段判断请求中是否带有内容。

在 HTTP_Parser 解析报头结束后，报文内容部分会通过 data 事件触发，我们只需要以流的方式处理即可，如下所示：

```js
const hasBody = (req) => 'transfer-encoding' in req.headers || 'content-length' in req.headers

function (req, res) {
    if (hasBody(req)) {
        var buffers = []
        req.on('data', function (chunk) {
            buffers.push(chunk)
        })
        req.on('end', function () {
            req.rawBody = Buffer.concat(buffers).toString()
            handle(req, res)
        })
    } else {
        handle(req, res)
    }
}
```

#### (1)表单数据

通过 form 标签默认的表单提交请求头中的 Content-Type 字段值为 application/x-www-form-urlencoded，而它的报文体内容与查询字符串相同：`foo=bar&baz=val`，因此它的解析非常容易：

```js
const handle = (req, res) => {
    if (req.header['content-type'] === 'application/x-www-form-urlencoded') {
        req.body = querystring.parse(req.rawBody)
    }
}
```

#### (2)其他格式

除了表单数据外常见的提交还有 JSON 和 XML 文件等，判断它们的方式也是通过 Content-Type 字段，值分别为 application/json 和 application/xml。

json 文件的解析是非常简单的，我们可以直接通过 JSON.parse 方法进行解析，XML 文件的解析稍微复杂一些但我们也可以采用 XML 文件到 JSON 对象转换的库，例如 xml2js 模块。

#### (3)附件上传

通常的表单，其内容可以通过 urlencoded 的方式编码内容形成报文体，再发送给服务器端，但是业务场景往往需要用户直接提交文件。在前端 HTML 代码中，特殊表单与普通表单的差异在于该表单中可以含有 file 类型的控件，以及需要指定表单属性 enctype 为 multipart/form-data，如下所示：

```html
<form action="/upload" method="post" enctype="multipart/form-data">
  <label for="username">Username:</label><input type="text" name="username" id="username"/>
  <label for="file">Filename:</label><input type="file" name="file" id="file">
  <br />
  <input type="submit" name="submit" value="Submit">
</form>
```

浏览器在遇到 multipart/form-data 表单提交时，构造的请求报文与普通报文完全不同。首先它的报头中最为特殊的如下所示：

```
Content-Type: multipart/form-data; boundary=AaB03x
Content-Length: 18231
```

它代表本次提交的内容是由多部分构成的，其中 boundary=AaB03x 是随机生成的一段字符串，制定每部分内容的分界符。报文的内容将通过在它前面添加 '--' 进行分割，报文结束在它前后都加 '--' 表示结束。另外 Content-Length 的值必须确保是报文体的长度。

在知晓了报问题是如何构成之后解析就变得非常容易，但是对于未知大小的数据量进行处理时依然需要小心，我们也可以采用一些第三方库比如 formidable 来协助我们处理。

