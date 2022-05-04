---
title: Node中的网络编程
date: 2021-08-31 10:52:38
toc: true
mathjax: false
categories: 
- Node 
tags:
- Node
---

网络编程的概念是"使用套接字来达到进程间通信的目的"。通常情况下，我们要使用网络提供的功能，可以有以下几种方式：

- 使用应用软件提供的网络通信功能来获取网络服务，最著名的就是浏览器，它在应用层上使用 http 协议，在传输层基于 TCP 协议；
- 在命令行方式下使用 shell 命令获取系统提供的网络服务，如 telnet、ftp 等；
- 使用编程的方式通过系统调用获取操作系统提供给我们的网络服务。

<!-- more -->

网络编程涉及到一个套接字（socket）的概念，所谓套接字，实际上是两个不同进程间进行通信的端口（这里的端口有别于IP地址中常用的端口），它是对网络层次模型中网络层及其下面各层操作的一个封装，为了让开发者能够使用各种语言调用操作系统提供的网络服务，在不同服务端语言中都使用了套接字这个概念，开发者只要获得一个套接字（socket），就可以使用套接字（socket）中各种方法来创建不同进程之间的连接进而达到通信目的。

Socket 源于 Unix，而 Unix 的基本哲学是『一些皆文件』，都可以用『打开open ==> 读/写(read/write) ==> 关闭(close)』模式来操作，Socket 也可以采用这种方法进行理解。关于 Socket，可以总结如下几点:

- 可以实现底层通信，几乎所有的应用层都是通过 socket 进行通信的，因此『一切且socket』
- 对 TCP/IP 协议进行封装，便于应用层协议调用，属于二者之间的中间抽象层
- 各个语言都有相关实现，例如 C、C++、node
- TCP/IP 协议族中，传输层存在两种通用协议: TCP、UDP，两种协议不同，因此 Socket 的参数不同，实现过程也不一样

<img src="/assets/node-network/01.jpeg" />

## Node 中网络通信的架构实现

Node 中的模块，从两种语言实现角度来说，存在 javscript、c++ 两部分，通过 process.binding 来建立关系。具体分析如下:

- 标准的 Node 模块有 net、udp、dns、http、tls、https 等
- V8 是 Chrome 的内核，提供了 JavaScript 解释运行功能，里面包含 tcp_wrap.h、udp_wrap.h、tls_wrap.h 等
- OpenSSL 是基本的密码库，包括了 MD5、SHA1、RSA 等加密算法，构成了 Node 标准模块中的 crypto
- cares 模块用于 DNS 的解析
- libuv 实现了跨平台的异步编程
- http_parser 用于 http 的解析

<img src="/assets/node-network/02.png" />

## net 模块

net 模块是基于 TCP 协议的 socket 网路编程模块，http 模块就是建立在该模块的基础上实现的，先来看看基本使用方法:

### 1、创建 TCP 服务端

创建一个TCP服务器，可以通过使用构造函数 `new net.Server` 或者使用工厂方法 `net.createServer`，这两个方法都会返回一个 net.Server 类，可接收两个可选参数。

```js
const net = require('net')
const server = net.createServer();
server.on('connection', (socket) => {
  socket.pipe(process.stdout);
  socket.write('data from server');
});
server.listen(3000, () => {
  console.log(`server is on ${JSON.stringify(server.address())}`);
});
```

或者也可以采用另一种写法：

```js
var net=require('net');
var server=net.createServer(function(socket){
  console.log('客户端和服务端建立连接');
  server.getConnections(function(err,count){
    console.log("当前连接数为%d",count);
  });
  server.maxConnections=2;
  console.log('tcp最大连接数为%d',server.maxConnections);
});
server.on('error',function(e){
  if(e.code=='EADDRINUSE'){
    console.log('地址和端口被占用');
  }
});
server.listen(3000,'localhost',function(){
  //console.log('服务器端开始监听...');
  var address=server.address();
  console.log(address);
});
```

在终端使用tenlent 0.0.0.0 3000 即可与之通信。

示例中我们使用 `const server = net.createServer();` 创建了 server 对象，那 server 对象有哪些特点:

```js
// net.js
exports.createServer = function(options, connectionListener) {
  return new Server(options, connectionListener);
};
function Server(options, connectionListener) {
  EventEmitter.call(this);
  ...
  if (typeof connectionListener === 'function') {
    this.on('connection', connectionListener);
  }
  ...
  this._handle = null;
}
util.inherits(Server, EventEmitter);
```

首先我们看到 server 对象是一个 EventEmitter 实例，它的自定义事件有如下几种:

- listening：在调用 server.listen() 绑定端口或 Domain Socket 后触发，可以写作 server.listen(port, listeningListener)。
- connection：每个客户端 socket 连接到服务器时触发，可以写作 net.createServer(options, connectionListener)。
- close：服务器关闭时触发。server.close() 会停止接受新的 socket，但是保存已有的连接，等待所有的连接断开后触发。
- error：服务器发生异常时触发。

其次我们在 server 对象中发现了 _handle 属性，_handle 是 server 处理的句柄，属性值最终由 C++ 部分的 TCP、Pipe 类创建。

```js
function createServerHandle(address, port, addressType, fd) {
  ...
  if (typeof fd === 'number' && fd >= 0) {
    ...
    handle = createHandle(fd);
    ...
  } else if(port === -1 && addressType === -1){
    handle = new Pipe();
  } else {
    handle = new TCP();
  }
  ...
  return handle;
}
function createHandle(fd) {
  var type = TTYWrap.guessHandleType(fd);
  if (type === 'PIPE') return new Pipe();
  if (type === 'TCP') return new TCP();
  throw new TypeError('Unsupported fd type: ' + type);
}
```

_handle 由 C++ 中的 Pipe、TCP 实现，主要用于 Node 进程通信中 socket 句柄传递，具体可查看[通过源码解析 Node.js 中进程间通信中的 socket 句柄传递](https://cnodejs.org/topic/572b8a2c8783d212174bd72b)。

再来看看 connectionListener 事件的回调函数，里面包含一个 socket 对象，该对象是一个连接套接字，是个五元组 (server_host、server_ip、protocol、client_host、client_ip)，相关实现如下:

```js
function onconnection(err, clientHandle) {
  ...
  var socket = new Socket({
    ...
  });
  ...
  self.emit('connection', socket);
}
```

因为 Socket 是继承了 stream.Duplex，所以 Socket 也是一个可读可写流，可以使用流的方法进行数据的处理。

接下来就是很关键的端口监听(port)，这是 server 与 client 的主要区别，代码:

```js
Server.prototype.listen = function() {
  ...
  listen(self, ip, port, addressType, backlog, fd, exclusive);
  ...
}
function listen(self, address, port, addressType, backlog, fd, exclusive) {
  ...
  if (!cluster) cluster = require('cluster');
  if (cluster.isMaster || exclusive) {
    self._listen2(address, port, addressType, backlog, fd);
    return;
  }
  cluster._getServer(self, {
    ...
  }, cb);
  function cb(err, handle) {
    ...
    self._handle = handle;
    self._listen2(address, port, addressType, backlog, fd);
    ...
  }
}
Server.prototype._listen2 = function(address, port, addressType, backlog, fd) {
  if (this._handle) {
    ...
  } else {
    ...
    rval = createServerHandle(address, port, addressType, fd);
    ...
    this._handle = rval;
  }
  this._handle.onconnection = onconnection;
  var err = _listen(this._handle, backlog);
  ...
}

function _listen(handle, backlog) {
  return handle.listen(backlog || 511);
}
```

上述代码有几个点需要注意:

- 监听的对象可以是端口、路径、定义好的 server 句柄、文件描述符
- 当通过 cluster 创建工作进程(worker)时，exclusive 判断是否进行 socket 连接的共享
- 事件监听最终还是通过 TCP/Pipe 的 listen 来实现
- backlog 规定了 socket 连接的限制，默认最多为 511

### 2、创建 TCP 客户端

创建一个 TCP 客户端链接可以使用构造函数 `new net.Socket` 或者其工厂方法 `net.createConnection`，创建成功后都会返回一个 net.Socket 实例。

```js
var net = require('net');

var client = net.createConnection({port:3000, host:'localhost'});

client.on('connect',function(){
  console.log('client connect');
  client.write('hello world!');
  setTimeout(function(){
    client.end('bye bye');
  },10000);
});

client.on('data',function(data){
  console.log('client data', data.toString());
});

client.on('error',function(error){
  throw error;
  client.destroy();
});

client.on('close',function(){
  console.log('client close');
});
```

服务器可以与多个客户端保存连接，每个连接都是典型的可读可写的 Stream 对象。它的自定义事件有如下几种:

- data：当一端调用 write() 发送数据，另外一端触发 data 事件。
- end：当连接中的任一端发送 FIN 数据时，触发该事件。
- connect：客户端 socket 与服务器连接成功适触发。
- drain：rain 和 socket.write() 的返回值强关联，当任意一端调用 write()，当前这端会触发该事件。
- error：异常时触发。
- close：socket 关闭时触发。
- timeout：一定时间连接不再活跃时，该事件触发，通知用户当前连接已经闲置。

从上面可以看出基于TCP连接的通信具有的特点:

- 面向连接，必须建立连接后才能够互相通信
- TCP 连接是一对一的，就是说在 TCP 中，一个客户端 socket 连接一个服务端 socket，并且两者可以相互通信，通信是双向的
- TCP 连接关闭的时候是可以只关闭一方的连接而保留单向通信
- 一个特定的 IP 加端口可以连接多个 TCP 客户端，也可以通过编程指定连接上限

## dgram模块

跟 net 模块相比，基于 UDP 通信的 dgram 模块就简单了很多，因为不需要通过三次握手建立连接，所以整个通信的过程就简单了很多，对于数据准确性要求不太高的业务场景，可以使用该模块完成数据的通信。

### 1、创建 UDP 服务端

```js
var dgram=require('dgram');
var server=dgram.createSocket('udp4');
server.on("message",(msg,rinfo) => {
  console.log('已接收到客户端发送的数据为' + msg);
  console.log("客户端地址新信息为%j", rinfo);
  var buff=new Buffer("确认信息"+msg);
  server.send(buff,0,buff.length,rinfo.port,rinfo.address);
  setTimeout(() => {
    server.unref();
  },10000);
});
server.on("listening", () => {
  var address = server.address();
  console.log("服务器开始监听，地址信息为%j",address);
});
server.bind(3000,'localhost');
```

从源码层面分析上述代码的原理实现:

```js
exports.createSocket = function(type, listener) {
  return new Socket(type, listener);
};
function Socket(type, listener) {
  ...
  var handle = newHandle(type);
  this._handle = handle;
  ...
  this.on('message', listener);
  ...
}
util.inherits(Socket, EventEmitter);
const UDP = process.binding('udp_wrap').UDP;
function newHandle(type) {
  if (type == 'udp4') {
    const handle = new UDP();
    handle.lookup = lookup4;
    return handle;
  }

  if (type == 'udp6') {
    const handle = new UDP();
    handle.lookup = lookup6;
    handle.bind = handle.bind6;
    handle.send = handle.send6;
    return handle;
  }
  ...
}
Socket.prototype.bind = function(port_ /*, address, callback*/) {
  ...
  startListening(self);
  ...
}
function startListening(socket) {
  socket._handle.onmessage = onMessage;
  socket._handle.recvStart();
  ...
}
function onMessage(nread, handle, buf, rinfo) {
  ...
  self.emit('message', buf, rinfo);
  ...
}
Socket.prototype.send = function(buffer, offset, length, port, address, callback) {
  ...
  self._handle.lookup(address, function afterDns(ex, ip) {
    doSend(ex, self, ip, list, address, port, callback);
  });
}
const SendWrap = process.binding('udp_wrap').SendWrap;
function doSend(ex, self, ip, list, address, port, callback) {
  ...
  var req = new SendWrap();
  ...
  var err = self._handle.send(req, list, list.length, port, ip, !!callback);
  ...
}
```

上述代码存在几个点需要注意:

- UDP 模块没有继承 stream，仅仅继承了 EventEmit，后续的所有操作都是基于事件的方式，它的自定义事件如下：
  - message：当 UDP socket 侦听网卡端口后，接收到消息时触发该事件。
  - listening：当 UDP 开始侦听时触发该事件。
  - close：调用 close() 方法时触发该事件，并不再触发 message 事件。
  - error：发生异常时触发该事件。
- UDP 在创建的时候需要注意 ipv4 和 ipv6
- UDP 的 _handle 是由 UDP 类创建的
- 通信过程中可能需要进行 DNS 查询，解析出 ip 地址，然后再进行其他操作

### 2、创建 UDP 客户端

```js
var dgram=require('dgram');
var message= Buffer.from('hello world');
var client=dgram.createSocket('udp4');
client.send(message,0,message.length, 3000,"localhost", (err,bytes) => {
  if(err) console.log('数据发送失败');
  else console.log("已发送%d字节数据",bytes);
});
client.on("message", (msg,rinfo) => {
  console.log("已接收到服务端发送的数据%s",msg);
  console.log("服务器地址信息为%j",rinfo);
  client.close();
});
client.on("close", () => {
  console.log("socket端口被关闭");
});
```

### 3、UNIX Domain Socket IPC

socket API 原本是为网络通讯设计的，但后来在 socket 的框架上发展出一种 IPC 机制，就是 UNIX Domain Socket。虽然网络 socket 也可用于同一台主机的进程间通讯（通过 loopback 地址 127.0.0.1），但是 UNIX Domain Socket 用于 IPC 更有效率：不需要经过网络协议栈，不需要打包拆包、计算校验和、维护序号和应答等，只是将应用层数据从一个进程拷贝到另一个进程。这是因为，IPC 机制本质上是可靠的通讯，而网络协议是为不可靠的通讯设计的。UNIX Domain Socket 也提供面向流和面向数据包两种 API 接口，类似于 TCP 和 UDP，但是面向消息的 UNIX Domain Socket 也是可靠的，消息既不会丢失也不会顺序错乱。
UNIX Domain Socket 与网络 socket 编程最明显的不同在于地址格式不同，用结构体 sockaddr_un 表示，网络编程的 socket 地址是IP地址加端口号，而 UNIX Domain Socket 的地址是一个 socket 类型的文件在文件系统中的路径，这个 socket 文件由 bind() 调用创建，如果调用 bind() 时该文件已存在，则 bind() 错误返回。

**创建一个 UNIX 域套接字服务器**

```js
const net = require("net");
const server = net.createServer(c => {
  c.on("end", () => {
    console.log("client disconnected");
  });
  c.write("hello\r\n");
  c.pipe(c);
});
server.on("error", err => {
  throw err;
});
// server.listen(path[, backlog][, callback]) for IPC servers
server.listen("/tmp/echo.sock", () => {
  console.log("server bound");
});
```

**连接UNIX 域套接字服务器**

```sh
nc -U /tmp/echo.sock # -U — Use UNIX domain socket
```

> nc（netcat）可以用于涉及 TCP 或 UDP 的相关内容，比如通过它我们可以打开 TCP 连接，发送 UDP 数据包，监听任意的 TCP 和 UDP 端口，执行端口扫描和处理 IPv4 和 IPv6 等。

## dns 模块

DNS(Domain Name System) 用于域名解析，也就是找到 host 对应的 ip 地址，在计算机网络中，这个工作是由网络层的 ARP 协议实现。在 node 中存在 net 模块来完成相应功能，其中 dns 里面的函数分为两类:

- 第一类函数，使用底层操作系统工具进行域名解析，且无需进行网络通信。 这类函数只有一个：dns.lookup()。
- 第二类函数，连接到一个真实的 DNS 服务器进行域名解析，且始终使用网络进行 DNS 查询。 这类函数包含了 dns 模块中除 dns.lookup() 以外的所有函数。 这些函数使用与 dns.lookup() 不同的配置文件（例如 /etc/hosts）。 这类函数适合于那些不想使用底层操作系统工具进行域名解析、而是想使用网络进行 DNS 查询的开发者。

```js
const dns = require('dns');
const host = 'bj.meituan.com';
dns.lookup(host, (err, address, family) => {
  if (err) {
    console.log(err);
    return;
  }
  console.log('by net.lookup, address is: %s, family is: %s', address, family);
});

dns.resolve(host, (err, address) => {
  if (err) {
    console.log(err);
    return;
  }
  console.log('by net.resolve, address is: %s', address);
})
// by net.resolve, address is: 103.37.152.41
// by net.lookup, address is: 103.37.152.41, family is: 4
```

在这种情况下，二者解析的结果是一样的，但是假如我们修改本地的/etc/hosts文件呢

```
// 在/etc/host文件中，增加:
10.10.10.0 bj.meituan.com

// 然后再执行上述文件，结果是:
by net.resolve, address is: 103.37.152.41
by net.lookup, address is: 10.10.10.0, family is: 4
```

接下来分析下dns的内部实现:

```js
const cares = process.binding('cares_wrap');
const GetAddrInfoReqWrap = cares.GetAddrInfoReqWrap;
exports.lookup = function lookup(hostname, options, callback) {
  ...
  callback = makeAsync(callback);
  ...
  var req = new GetAddrInfoReqWrap();
  req.callback = callback;
  var err = cares.getaddrinfo(req, hostname, family, hints);
  ...
}

function resolver(bindingName) {
  var binding = cares[bindingName];
  return function query(name, callback) {
    ...
    callback = makeAsync(callback);
    var req = new QueryReqWrap();
    req.callback = callback;
    var err = binding(req, name);
    ...
    return req;
  }
}
var resolveMap = Object.create(null);
exports.resolve4 = resolveMap.A = resolver('queryA');
exports.resolve6 = resolveMap.AAAA = resolver('queryAaaa');
...
exports.resolve = function(hostname, type_, callback_) {
  ...
  resolver = resolveMap[type_];
  return resolver(hostname, callback);
  ...
}
```

上面的源码有几个点需要关注:

- lookup 与 resolve 存在差异，使用的时候需要注意
- 不管是 lookup 还是 resolve，均依赖于 cares 库
- 域名解析的 type 很多: resolve4、resolve6、resolveCname、resolveMx、resolveNs、resolveTxt、resolveSrv、resolvePtr、resolveNaptr、resolveSoa、reverse

## http 模块

在WEB开发中，HTTP作为最流行、最重要的应用层，是每个开发人员应该熟知的基础知识，我面试的时候必问的一块内容。同时，大多数同学接触node时，首先使用的恐怕就是http模块。先来一个简单的demo看看:

```js
const http = require('http');
const server = http.createServer();
server.on('request', (req, res) => {
  res.setHeader('foo', 'test');
  res.writeHead(200, {
    'Content-Type': 'text/html',
  });
  res.write('<!doctype>');
  res.end(`<html></html>`);
});

server.listen(3000, () => {
  console.log('server is on ', server.address());
  var req = http.request({ host: '127.0.0.1', port: 3000});
  req.on('response', (res) => {
    res.on('data', (chunk) => console.log('data from server ', chunk.toString()) );
    res.on('end', () => server.close() );
  });
  req.end();
});
// 输出结果如下:
// server is on  { address: '::', family: 'IPv6', port: 3000 }
// data from server  <!doctype>
// data from server  <html></html>
```

### 1、http.Agent

因为HTTP协议是无状态协议，每个请求均需通过三次握手建立连接进行通信，众所周知三次握手、慢启动算法、四次挥手等过程很消耗时间，因此HTTP1.1协议引入了keep-alive来避免频繁的连接。那么对于tcp连接该如何管理呢？http.Agent就是做这个工作的。先看看源码中的关键部分:

```js
function Agent(options) {
  ...
  EventEmitter.call(this);
  ...
  self.maxSockets = self.options.maxSockets || Agent.defaultMaxSockets;
  self.maxFreeSockets = self.options.maxFreeSockets || 256;
  ...
  self.requests = {}; // 请求队列
  self.sockets = {}; // 正在使用的tcp连接池
  self.freeSockets = {}; // 空闲的连接池
  self.on('free', function(socket, options) {
    ...
    // requests、sockets、freeSockets的读写操作
    self.requests[name].shift().onSocket(socket);
    freeSockets.push(socket);
    ...
  }
}
Agent.defaultMaxSockets = Infinity;
util.inherits(Agent, EventEmitter);
// 关于socket的相关增删改查操作
Agent.prototype.addRequest = function(req, options) {
  ...
  if (freeLen) {
    var socket = this.freeSockets[name].shift();
    ...
    this.sockets[name].push(socket);
    ...
  } else if (sockLen < this.maxSockets) {
    ...
  } else {
    this.requests[name].push(req);
  }
  ...
}
Agent.prototype.createSocket = function(req, options, cb) { ... }
Agent.prototype.removeSocket = function(s, options) { ... }
exports.globalAgent = new Agent();
```

上述代码有几个点需要注意:

- maxSockets默认情况下，没有tcp连接数量的上限(Infinity)
- 连接池管理的核心是对sockets、freeSockets的增删查
- globalAgent会作为http.ClientRequest的默认agent

下面可以测试下agent对请求本身的限制:

```js
// req.js
const http = require('http');

const server = http.createServer();
server.on('request', (req, res) => {
  var i=1;
  setTimeout(() => {
    res.end('ok ', i++);
  }, 1000)
});

server.listen(3000, () => {
  var max = 20;
  for(var i=0; i<max; i++) {
    var req = http.request({ host: '127.0.0.1', port: 3000});
    req.on('response', (res) => {
      res.on('data', (chunk) => console.log('data from server ', chunk.toString()) );
      res.on('end', () => server.close() );
    });
    req.end();
  }
});
// 在终端中执行time node ./req.js，结果为:
// real  0m1.123s
// user  0m0.102s
// sys 0m0.024s

// 在req.js中添加下面代码
http.globalAgent.maxSockets = 5;
// 然后同样time node ./req.js，结果为:
real  0m4.141s
user  0m0.103s
sys 0m0.024s
```

当设置maxSockets为某个值时，tcp的连接就会被限制在某个值，剩余的请求就会进入requests队列里面，等有空余的socket连接后，从request队列中出栈，发送请求。

### 2、http.ClientRequest

当执行http.request时，会生成ClientRequest对象，该对象虽然没有直接继承Stream.Writable，但是继承了http.OutgoingMessage，而http.OutgoingMessage实现了write、end方法，因为可以当跟stream.Writable一样的使用。

```js
var req = http.request({ host: '127.0.0.1', port: 3000, method: 'post'});
req.on('response', (res) => {
  res.on('data', (chunk) => console.log('data from server ', chunk.toString()) );
  res.on('end', () => server.close() );
});
// 直接使用pipe，在request请求中添加数据
fs.createReadStream('./data.json').pipe(req);
```

接下来，看看http.ClientRequest的实现, ClientRequest继承了OutgoingMessage:

```js
const OutgoingMessage = require('_http_outgoing').OutgoingMessage;
function ClientRequest(options, cb) {
  ...
  OutgoingMessage.call(self);
  ...
}
util.inherits(ClientRequest, OutgoingMessage);
```

### 3、http.Server

http.createServer其实就是创建了一个http.Server对象，关键源码如下:

```js
exports.createServer = function(requestListener) {
  return new Server(requestListener);
};
function Server(requestListener) {
  ...
  net.Server.call(this, { allowHalfOpen: true });
  if (requestListener) {
    this.addListener('request', requestListener);
  }
  ...
  this.addListener('connection', connectionListener);
  this.timeout = 2 * 60 * 1000;
  ...
}
util.inherits(Server, net.Server);
function connectionListener(socket) {
  ...
  socket.on('end', socketOnEnd);
  socket.on('data', socketOnData)
  ...
}
```

有几个需要要关注的点:

- 服务的创建依赖于net.server，通过net.server在底层实现服务的创建
- 默认情况下，服务的超时时间为2分钟
- connectionListener处理tcp连接后的行为，跟net保持一致

### 4、http.ServerResponse

看node.org官方是如何介绍server端的response对象的:

> This object is created internally by an HTTP server–not by the user. It is passed as the second parameter to the ‘request’ event.

> The response implements, but does not inherit from, the Writable Stream interface.

跟http.ClientRequest很像，继承了OutgoingMessage，没有继承Stream.Writable，但是实现了Stream的功能，可以跟Stream.Writable一样灵活使用:

```js
function ServerResponse(req) {
  ...
  OutgoingMessage.call(this);
  ...
}
util.inherits(ServerResponse, OutgoingMessage);
```

### 5、http.IncomingMessage

> An IncomingMessage object is created by http.Server or http.ClientRequest and passed as the first argument to the ‘request’ and ‘response’ event respectively. It may be used to access response status, headers and data.

http.IncomingMessage有两个地方时被内部创建，一个是作为server端的request，另外一个是作为client请求中的response，同时该类显示地继承了Stream.Readable。

```js
function IncomingMessage(socket) {
  Stream.Readable.call(this);
  this.socket = socket;
  this.connection = socket;
  ...
}
util.inherits(IncomingMessage, Stream.Readable);
```

## 参考资料

[初步研究node中的网络通信模块](http://zhenhua-lee.github.io/node/socket.html)

[node基础篇之网络编程](https://github.com/kekobin/blog/issues/32)