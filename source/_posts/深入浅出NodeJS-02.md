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

#### (4)数据上传与安全

**内存限制**

在我们通过 Node 解析表单、JSON 和 XML 部分时我们采取的策略往往是先保存用户提交的所有数据，然后再解析处理，最后再传递给业务逻辑。这种策略潜在的问题是它仅仅适合数据量小的提交请求，一旦数据量过大将发生内存被占光的情景。

要解决这个问题主要有两种方案：

- 限制上传内容的大小，一旦超过限制，停止接受数据，并响应 400 状态码
- 通过流式解析，将数据流导向磁盘中，Node 只保留文件路径等小数据

**CSRF**

服务器端与客户端通常通过 Cookie 来标识和认证用户，但是部分情况下会出现通过引诱用户点击恶意网站的链接来冒充用户的信息，除了通过配置 Cookie 的相关属性外我们还可以通过添加随机值的方式来解决。也就是说为每个请求的用户，在 Session 中赋予一个随机值，由于该值是一个随机值，攻击者构造出相同随机值的难度相当大，我们只需要在接收端做一次校验就能轻易地识别出该请求是否为伪造的。

### 3、路由解析

对于不同的业务我们希望有不同的处理方式，这就带来了路由的选择问题。

#### (1)文件路径型

**静态文件**

这种路由的处理方式十分简单，将请求路径对应的文件发送到客户端即可。

**动态文件**

这种方式的实现原理是 Web 服务器根据 URL 路径找到对应的文件，如 index.asp。Web 服务器根据文件名的后缀去寻找脚本的解析器，并传入 HTTP 请求的上下文。

现今大多数服务器都能很智能地根据后缀同时服务动态文件和静态文件。但这种方式在 Node 中不太常见，主要原因是文件的后缀都是 js，分不清是后端脚本还是前端脚本。而且 Node 中 Web 服务器与应用业务脚本是一体的，也无须按照这种方式实现。

#### (2)MVC

MVC 模型的主要思想是将业务逻辑按职责分离，主要分为以下几种：

- 控制器 Controller，一组行为的集合
- 模型 Model，数据相关的操作和封装
- 视图 View，视图的渲染

这是目前最经典的封层模式，其工作模式如下：

- 路由解析，根据 URL 寻找到对应的控制器和行为。
- 行为调用相关的模型，进行数据操作。
- 数据操作结束后，调用视图和相关数据进行页面渲染，输出到客户端。

这里如何根据 URL 做路由映射主要有两种方法实现，一种是通过手工关联映射，一种是自然关联映射。前者会有一个对应的路由文件来将 URL 映射到对应的控制器，后者没有这样的文件。

手工映射主要是通过一个路由文件来将 URL 映射到对应的控制器，其对 URL 的要求十分灵活。不过这种映射关系的解析需要两大基本能力：正则匹配与参数解析，用于对 URL 中的携带参数进行提取与处理。

相较于手工关联，自然关联采用了按照一种约定的方式自然而然地实现路由而无须去维护路由映射文件的手段，例如我们可以对；路径进行如下的划分处理：

`/controller/action/pararm1/param2/param3`

以 '/user/setting/12/1987' 为例，它会按照约定去找 controllers 目录下的 user 文件，将其 require 出来后，调用这个模块的 setting() 方法，而其余的值作为参数直接传递给这个方法。

总而言之，手工映射对 URL 的处理十分灵活，不过需要我们维护一份路由关系映射文件，而且依赖于正则匹配与参数解析的核心能力。而自然映射的设计十分简洁，但是如果 URL 变动，它的文件也需要发生变动，手工映射只需要改动路由映射即可。

#### (3)RESTful

REST 的全称是 Representational State Transfer，中文含义是表现层状态转化。符合 REST 规范的设计，我们称为 RESTful 的设计。它的设计哲学主要是将服务端提供的内容实体看作是一个资源，并表现在 URL 上，然后通过请求方法定义资源的操作，通过 Accept 决定资源的表现形式。

RESTful 与 MVC 的设计并不冲突，而且是更好的改进。相比 MVC，RESTful 只是将 HTTP 请求方法也加入了路由的过程，以及在 URL 的路径上体现得更资源化。

### 4、中间件

中间件的主要用于简化和隔离基础设施与业务逻辑之间的细节，让开发者能够关注在业务的开发上，以达到提升开发效率的目的。

Node 的 http 模块提供了应用层协议网络的封装，对具体业务并没有支持，因此通过中间件搭建开发框架对业务提供强力支撑是很有必要的。

中间件的编写需要注意如下两个点：

- 编写高效的中间件
- 合理利用路由，避免不必要的中间件执行

#### (1)编写高效的中间件

编写高效的中间件其实就是提升单个单元的处理速度，以尽早调用 next() 执行后续逻辑。由于中间件一旦被匹配，那么每个请求都会使该中间件执行一次，哪怕它只浪费1毫秒的执行时间，都会让我们的 QPS 显著下降。常见的优化方法主要有：

- 使用高效的方法，必要时通过 jsperf.com 测试基准性能。
- 缓存需要重复计算的结果
- 避免不必要的计算

#### (2)合理使用路由

在拥有了一堆高效的中间件后我们还需要对每个中间件的合理使用做出判断，避免中间件参与不必要的请求处理处理过程。

### 5、页面渲染

#### (1)内容响应

服务端的响应从一定程度上决定或指示了客户端该如何处理响应的内容。因此响应包头中的 Content-* 字段十分重要。

**MIME**

不同的文件类型有不同的 MIME 值，响应在在 Content-Type 字段中返回以便客户端进行对应的处理。

除了 MIME 值外，Content-Type 还可以包括一些参数，比如字符集。

**附件下载**

Content-Disposition 字段决定了客户端将相应报文数据当作即时浏览的内容，还是可下载的附件。当数据可以存为附件时，它的值为 attachemnt，如果只需即时查看时它的值为 inline。

除此之外 Content-Disposition 字段还能通过参数指定保存时应该使用的文件名。例如：

`Content-Disposition: attachment; filename="filename.ext"`

**响应JSON**

我们可以通过指定 Content-Type 字段为 'application/json' 快捷响应 JSON。

**响应跳转**

我们可以返回 302 状态码同时响应头添加 Location 字段来将用户跳转到别的 URL。

#### (2)视图渲染

Web 应用最终呈现出来的内容都是通过一系列的视图渲染呈现出来的，在动态页面技术中，最终视图是由模板和数据共同生成出来的。

模板是带有特殊标签的 HTML 字段，通过与数据的渲染，将数据填充到这些特殊的标签中，最后生成普通的带数据的 HTML 片段。

#### (3)模板

模板技术的实质上就是将模板文件和数据通过模版引擎生成最终的 HTML 代码，形成模板技术主要包括如下四个要素：

- 模板语言
- 包含模板语言的模板文件
- 拥有动态数据的数据对象
- 模板引擎

##### 模板引擎

模板引擎的实现主要分为如下几个步骤：

- 语法分解。提取出普通字符串和表达式，这个过程通常用正则表达式匹配出来。
- 处理表达式，将标签表达式转换成普通的语言表达式。
- 生成待执行的语句。
- 与数据一起执行，生成最终的字符串。

**模板编译**

为了能够最终与数据一起执行生成字符串，我们需要将原始的模板字符串转换成一个函数对象，这个过程称为模板编译。

```js
var compile = function (str) {
    var tpl = str.replace(/<%=([\s\S]+?)%>/g, function (match, code) {
        return "' + obj." + code + "+ '"
    })

    tpl = "var tpl = '" + tpl + "'\nreturn tpl;";
    return new Function('obj', tpl);
}

var compiled = compile(str);

var render = function (compiled, data) {
    return compiled(data);
}
```

##### with的应用

为了让模板引擎更加灵活，字符串能继续表达为字符串，变量能够自动寻找属于它的对象，我们需要引入关键字 with。

```js
var compile = function (str, data) {
    var tpl = str.replace(/<%=([\s\S]+?)%>/g, function (match, code) {
        return "' + " + code + " + '"
    })
    tpl = "tpl = '" + tpl + "'";
    tpl = 'var tpl = "";\nwith (obj) {' + tpl + '}\nreturn tpl';
    return new Function('obj', tpl)
}
```

**模板安全**

实际上 XSS 漏洞的产生大多数跟模板有关，如果数据传入的值为恶意字符串比如 `<script>slert("I am XSS")</script>` 那么页面就会执行这个脚本。因此为了安全性，大多数模板都提供了转义的功能。转义就是将能形成 HTML 标签的字符转换成安全的字符，转义函数如下：

```js
var excape = function (html) {
    return String(html)
        .replace(/&(?!\w+;)/g, '&amp')
        .replace(/</g, '&lt;')
        .replace(/>/g, '&gt;')
        .replace(/"/g, '&quot;')
        .replace(/'/g, '&#039;');
}
```

为了让转义与非转义表现得更方便，我们可以使用不同的标签来表示：

```js
var render = function (str, data) {
    var tpl = str
        .replace(/\n/g, '\\n')
        .replace(/<%=([\s\S]+?)%>/g, function (match, code) {
            // 转义
            return "' + escope(" + code + ") + '";
        })
        .replace(/<%-([\s\S]+?)%/g, function (match, code) {
            // 正常
            return "' + " + code + " + '"
        });
    tpl = "tpl = '" + tpl + "'";
    tpl = 'var tpl = "";\nwith(obj) {' + tpl + '}\nreturn tpl;';
    return new Function('obj', 'escape', tpl);
} 
```

##### 模板逻辑

为了让模板更强大一些我们为它添加逻辑代码

```js
var compile = function (str) {
    var tpl = str
        .replace(/\n/g, '\\n') // 将换行符替换
        .replace(/<%=([\s\S]+?)%>/g, function (match, code) {
            // 转义
            return "' + escope(" + code + ")+ '";
        })
        .replace(/<%-([\s\S]+?)%>/g, function (match, code) {
            // 正常
            return "' + " + code + " + '";
        })
        .replace(/<%([\s\S]+?)%>/g, function (match, code) {
            return "';\n" + code + "\ntpl += '";
        })
        .replace(/\'\n/g, '\'')
        .replace(/\n\'/gm, '\'');

    tpl = "tpl = '" + tpl + "';";

    // 转换空行
    tpl = tpl.replace(/''/g, '\'\\n\'');
    tpl = 'var tpl = "";\nwith( obj || {}) {\n' + tpl + '\n}\nreturn tpl;';
    return new Function('obj', 'escape', tpl);
}
```

##### 集成文件系统

结合我们之前实现的 compile() 与 render() 方法我们已经能够实现将输入的模板字符串进行编译替换的功能，但是通过模板编译生成的中间函数只与模板字符串相关，与具体的数据无关，因此我们可以采用模板预编译的方法，预编译缓存模板编译后的结果，以此实现一次编译多次执行。这里我们再集成文件系统：

```js
var cache = {}
var VIEW_FOLDER = '/path/to/wwwroot/views';

res.render = function (viewname, data) {
    if (!cache[viewname]) {
        var text;
        try {
            text = fs.readFileSync(path.join(VIEW_FOLDER, viewname), 'utf-8')
        } catch (e) {
            res.writeHead(500, { 'Content-Type': 'text/html' });
            res.end('模板文件错误');
            return;
        }
        cache[viewname] = text;
    }
    var compiled = cache[viewname];
    res.writeHead(200, { 'Content-Type': 'text/html' });
    var html = compiled(data);
    res.end(html);
}
```

与文件系统集成后再引入缓存就可以很好的解决性能问题，接口也得到大大简化。

##### 子模板

有的时候模板太大太过复杂，会增加维护上的困难，这就导致了子模板的诞生，子模板可以嵌套在别的模板中，多个模板可以嵌入同一个子模板中。我们可以采用 include 关键字实现模板的嵌套：

```js
var files = {};

var preCompile() = function (str) {
    var replaced = str.replace(/<%\s+(include.*)\s+%>/g, function (match, data) {
        var partial = code.split(/\s/)[1];
        if (!files[partial]) {
            files[partial] = fs.readFileSync(path.join(VIEW_FOLDER, partial), 'utf-8');
        }
        return files[partial]
    })

    // 多层嵌套，继续替换
    if (str.match(/<%\s+(include.*)\s+%>/)) {
        return preCompile(replaced);
    } else {
        return replaced
    }
}
```

我们在 compile() 方法对字符串解析前先调用 preCompile 方法对数据进行预编译即可实现对子模板的支持。 

#### (4)Bigpipe

Bigpipe 是一个需要前后端配合实现的优化技术，它的主要思路是将页面分割成多个部分，先向用户输出没有数据的布局（框架），将每个部分逐步输出到前端，并最终渲染填充框架，完成整个网页的渲染，这个过程中需要前端 JavaScript 的参与，它负责将后续输出的数据渲染到页面上。

它比较重要的有如下几个点：

- 页面布局框架
- 后端持续性数据输出
- 前端渲染



## 九、玩转进程

### 1、服务模型的变迁

<img src="/assets/深入浅出NodeJS/16.jpg" />

### 2、多进程架构

Node 的多进程采用了著名的 Master-Worker 模式，即主从模式，主从模式是典型的分布式架构中用于并行处理业务的模式具备较好的可伸缩性和稳定性。主进程不负责具体的业务逻辑，而是负责调度或管理工作进程，工作进程负责具体的业务处理。

#### (1) 创建子进程

child_process 模块给予了 Node 创建子进程的能力，它提供了 4 个方法用于创建子进程：

- spawn()：启动一个子进程来执行命令
- exec()：启动一个子进程来执行命令，与 spawn() 不同的是其接口不同，它有一个回调函数获知子进程的状况
- execFile()：启动一个子进程来执行可执行文件
- fork()：与 spawn() 类似，不同点在于它创建 Node 的子进程只需要指定要执行的 JavaScript 文件模块即可

注：

1. spawn() 与 exec()、esecFile() 不同的是后两者创建时可以指定 timeout 属性设置超时时间，一旦创建的进程超过设定的时间将会被杀死。
2. exec() 与 execFile() 不同的是，exec() 适合执行已有的命令，execFile() 适合执行文件。

举个🌰：

```js
var cp = require('child_process')
cp.spawn('node', ['worker.js'])
cp.exec('node worker.js', function (err, stdout, stderr) {
    // some code
})
cp.execFile('worker.js', function (err, stdout, stderr) {
    // some code
})
cp.fork('./worker.js')
```

<img src="/assets/深入浅出NodeJS/17.jpeg" />

#### (2) 进程间通信

在主从模式中要实现主进程管理和调度工作进程的功能，需要主进程和工作进程之间的通信，进程间通信简称 IPC（Inter-Process Communication），其主要目的是为了让不同的进程能够相互的访问资源并进行协调工作。实现进程间通信的技术有很多，如命名管道、匿名管道、socket、信号量、共享内存、消息队列、Domain Socket等。Node 中实现 IPC 通道的方式如下所示：

<img src="/assets/深入浅出NodeJS/18.jpeg" width="500" />

Node 中 IPC 通道的具体细节实现由 libuv 提供，在 Windows 下由命名管道实现 (named pipe) 实现，*nix 系统则采用 Unix Domain Socket 实现。表现在应用层上的进程间通信只有简单的 message 事件和 send 方法，接口十分简洁和消息化。举个🌰：

```js
// parent.js
var cp = require('child_precess')
var n = cp.fork(__dirname + '/sub.js')

n.on('message', function (m) {
    console.log('PARENT got message:', m)
})
n.send({ hello: 'world!' })

// sub.js
process.on('message', function (m) {
    console.log('CHILD got message:', m)
})

process.send({ foo: 'bar' })
```

那么 Node 创建 IPC 通道的具体实现是怎么样的呢？

父进程在实际创建子进程之前会创建 IPC 通道（在 Node 中 IPC 通道被抽象为 Stream 对象）并**监听**它，然后才真正创建出子进程，并通过环境变量 (Node_CHANNEL_FD) 告诉子进程这个 IPC 通道的文件描述符。子进程在启动的过程中，**根据文件描述符去连接**这个已存在的 IPC 通道，从而完成父子进程之间的连接。

注：只有启动的子进程是 Node 进程时，子进程才会根据环境变量去连接 IPC 通道，对于其他类型的子进程则无法实现进程间通信，除非其他进程也按约定去连接这个创建好的 IPC 通道。

#### (3) 句柄传递

为了解决多个工作进程一个端口的问题通常的做法是**代理模式**，即主进程监听主端口、对外接收所有的网络请求，再将这些请求分别代理到不同的端口的进程上。其优点在于避免了端口不能重复监听的问题，而且我们可以在代理进程上做适当的负载均衡，但是由于**进程每接收到一个连接将会用掉一个文件描述符**，因此这种代理模式需要浪费一倍数量的文件描述符，这极大的影响了系统的扩展能力。

为了解决这个问题 Node 在版本 v0.5.9 引入了进程间发送句柄的功能。首先什么是句柄？

**句柄是一种可以用来标识资源的引用，它的内部包含了指向对象的文件描述符**，因此句柄可以用来标识一个服务器端 socket 对象、一个客户端 socket 对象、一个套接字等。

接下来我们通过发送句柄来实现多进程监听同端口

```js
// parent.js
var cp = require('child_process')
var child1 = cp.fork('child.js')
var child2 = cp.fork('child.js')

var server = require('net').createServer();
server.listen(1337, function () {
    child1.send('server', server)
    child2.send('server', server)
    server.close()
})

// child.js
var http = require('http')
var server = http.createServer(function (req, res) {
    res.writeHead(200, { 'Content-Type': 'text/plain' })
    res.end('handed by child, pid is ' + process.pid + '\n')
})
process.on('message', function (m, tcp) {
    if (m === 'server') {
        tcp.on('connection', function (socket) {
            server.emit('connection', socket)
        })
    }
})
```

那么我们为什么可以通过发送句柄来实现多进程监听同一端口呢？句柄发送的具体过程是什么样的呢？

目前子进程对象 send() 方法可以发送的句柄类型包括如下几种：

- net.Socket: TCP 套接字
- net.Server: TCP 服务器
- net.Native: C++ 层面的 TCP 套接字或 IPC 管道
- dgram.Socket: UDP 套接字
- dgram.Native: C++ 层面的 UDP 套接字

send() 方法在将消息发送到 IPC 管道前，将消息组装成两个对象，一个参数是 handle，另一个是 message。message 的参数如下所示：

```js
{
    cmd: 'NODE_HANDLE',
    type: 'net.Server',
    msg: messsage
}
```

这儿的文件描述符 handle 实际上是一个整数值，而这个 message 对象在写入到 IPC 通道时也会通过 JSON.stringify() 进行序列化。所以最终发送到 IPC 通道中的信息都是字符串，**send() 方法能发送消息和句柄并不意味着它能发送任意对象**。

**连接了 IPC 通道的子进程可以读取到父进程发送的消息，将字符串通过 JSON.parse() 解析还原为对象后，才触发 message 事件将消息传递给应用层使用。在这个过程中消息对象还要被进行过滤处理，message.cmd 的值如果以 NODE_ 为前缀，它将响应一个内部事件 internalMessage。如果 message.cmd 的值为 NODE_HANDLE，它将取出 message.type 值和得到的文件描述符一起还原出一个对应的对象**，以 TCP 服务器句柄为例：

```js
function (message, handle, emit) {
    var self = this

    var server = new net.Server()
    server.listen(handle, function() {
        emit(server)
    })
}
```

上面的示例中，子进程根据 message.type 创建对应 TCP 服务器对象，然后监听到文件描述符上。

我们独立启动的进程中 TCP 服务器端 socket 套接字的文件描述符并不相同，这也是导致监听相同端口时抛出异常的主要原因，但是对于 send() 发送的句柄还原出来的服务而言，它们的文件描述符是相同的，所以监听相同端口不会引起异常。

### 3、集群稳定之路

#### (1) 进程事件

<img src="/assets/深入浅出NodeJS/19.jpg">

#### (2) 自动重启

在有了父子进程之间的相关事件之后，我们就可以在这些关系之间创建出需要的机制了，比如我们可以当监听到子进程退出后重新启动一个工作进程来继续服务。

```js
// master.js
var fork = require('child_process').fork
var cpus = require('os').cpus()

var server = require('net').createServer()
server.listen(1337)

var workers = {}
var createWorker = function () {
    var worker = fork(__dirname + '/worker.js')
    worker.on('exit', function () {
        console.log('Worker ' + worker.pid + ' exited.')
        delete workers[worker.pid]
        createWorker()
    })
    // 句柄转发
    worker.send('server', server)
    workers[worker.pid] = worker
    console.log('Create worker. pid: ' + worker.pid)
}

for (var i = 0; i < cpus.length; i++) {
    createWorker()
}

process.on('exit', function () {
    for (var pid in workers) {
        workers[pid].kill()
    }
})

// worker.js
var http = require('http')

var server = http.createServer((req, res) => {
    res.writeHead(200, { 'Content-Type': 'text/plain' })
    res.end('handled by child, pid is ' + process.pid + '\n')
})

var worker;

process.on('message', (m, tcp) => {
    if (m === 'server') {
        worker = tcp
        tcp.on('connection', (socket) => {
            server.emit('connection', socket)
        })
    }
})
process.on('uncaughtException', () => {
    // 停止接收新的连接
    worker.close(function () {
        // 所有连接断开后退出进程
        process.exit(1)
    })
})
```

