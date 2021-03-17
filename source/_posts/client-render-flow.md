---
title: 浏览器如何渲染页面
date: 2021-01-22 10:34:23
toc: true
categories: 
- 前端
tags:
- Chrome
---

从用户在浏览器中输入一个网址或关键字到浏览器成功渲染目标页面过程中到底发生了什么？浏览器端如何从远程服务器拉取目标页面？又如何完成完整页面的渲染？本篇文章将尽可能详细的展开描述这个过程。

<!-- more -->

## 一、用户输入

用户在地址栏输入时浏览器会判断用户输入是搜索内容还是请求的url:

 - 如果是搜索关键字，地址栏会根据浏览器默认的搜索引擎来合成新的带搜索关键字的url。
 - 如果用户输入符合请求url的格式，比如`baidu.com`，那么地址栏会根据规则添加协议合并成完整的url: `https://www.baidu.com`。

需要注意的是当用户在地址栏输入并按下回车键后首先会触发当前页面的 beforeunload 事件，如果当前页面监听了 beforeunload 事件那么可以在 beforeunload的 事件监听函数里询问用户是否要离开当前页面，因此如果当前页面没有监视 beforeunload 事件或者在监听beforeunload的事件处理函数里没有取消导航，那么浏览器便开始地址栏最后生成url地址的加载。

## 二、请求url

在这一步浏览器主线程会通过IPC(进程间通信)把url请求发送给网络进程，网络进程会真正的发起url请求流程，具体流程如下：

### 1、判断资源是否已缓存

首先网络进程会查找本地缓存是否缓存了请求资源，如果本地存在缓存资源，那么直接返回资源给浏览器主进程，如果本地缓存不存在请求资源，那么网络进程会正式开始网络请求流程。

### 2、获取IP地址以及端口号

正式网络请求流程的第一步就是通过 DNS 解析获取请求域名对应的服务器地址。此时需要注意 DNS 缓存，即如果某个域名解析过了，那么浏览器会缓存解析的结果，以供下次查询时直接使用，这样也会减少一次网络请求。拿到 IP 地址后就需要获取端口号，通常情况下如果 URL 没有特别指明端口号那么 HTTP 协议默认的就是80端口。

### 3、等待TCP队列

接下来我们需要根据 IP 地址以及端口号建立与请求服务器之间的 TCP 连接（如果请求协议是 HTTPS 那么需要先建立 TLS 连接），这儿我们需要解释一下 HTTP 协议和 TCP 协议之间的关系：浏览器使用 HTTP 协议作为应用层协议，用于封装请求的文本信息，并使用 TCP/IP 协议作为传输层协议将请求文本信息发送到网络上，因此在 HTTP 工作开始之前，浏览器需要通过 TCP 协议与服务器建立连接，但是实际上在建立 TCP 连接之前请求还需要排队等待，Chrome 有个机制，同一个域名同时最多只能建立6个TCP连接，因此如果在同一个域名下同时有10个请求发生，那么其中4个请求会进入排队等待状态，直至进行中的请求完成。

### 4、建立TCP连接

TCP 是面向连接的通信传输协议，面向连接是指在数据通信开始之前需要先做好两端之间的准备工作，因此在建立 TCP 连接之前，客户端和服务器之间会发送三次数据，以确认双方的接收和发送能力，这个过程称为“三次握手”（Three-way Handshake），具体过程如下所示。

**第一次握手**：刚开始客户端和服务端都处于 CLOSED 状态。先是服务端主动监听某个端口，处于 LISTEN 状态。然后客户端主动发起发送 SYN 报文，并指明客户端的初始化序列号 x，之后处于 SYN-SENT 状态。

**第二次握手**：当服务器收到客户端的 SYN 报文之后，会以自己的 SYN 报文作为应答，并且也指定了自己的初始化序列号 y。同时会把客户端初始化序列号 x + 1 作为 ACK 报文的值，表示自己已经收到了客户端的 SYN，之后处于 SYN-RCVD 状态。

**第三次握手**：当客户端收到 SYN 报文之后，会发送一个 ACK 报文，当然，也同样把服务器的初始序列号 y + 1 作为 ACK 的值，表示已经收到了服务端的 SYN 报文，同时还会附带服务端发回的 ACK 报文返回，然后客户端处于 ESTABLISHED 状态。服务器收到 ACK 报文之后，也会处于 ESTABLISHED 状态，此时，双方成功建立起了连接。


<img src="/assets/client-render-flow/02.png" width="600px" />

为什么TCP连接需要进行“三次握手”呢？
这是因为第一次握手成功让服务端知道了客户端具有发送能力，第二次握手成功让客户端知道了服务端具有接收和发送能力，但此时服务端并不知道客户端是否接收到了自己发送的消息，所以第三次握手就起到了这个作用。经过三次通信后，服务端和客户端都确认了双方的接收和发送能力。

### 5、发送请求信息

一旦建立了 TCP 连接，浏览器就可以和服务器通信了，而 HTTP 正是在这个通信过程中负责请求文本信息的封装并将请求文本发送到网络上，事实上 HTTP 请求报文由三部分构成：

- 请求行：包括了请求方法、请求 URI(Uniform Resource Indetifier) 和 HTTP 版本协议，展示基础请求信息。
- 请求头：即HTTP的报文头，用于把浏览器相关一些基础信息告诉服务器，比如包含了浏览器所使用的操作系统、浏览器内核等信息，以及当前请求的域名信息等等，同时浏览器端会将与该域名相关的Cookie等数据附加到请求头中。
- 请求体：即HTTP的报文体，用于承载请求参数。

具体如下图所示：

<img src="/assets/client-render-flow/01.png" />

### 6、接受响应信息

服务器在接收并处理来自浏览器的请求信息后会返回一个HTTP的响应信息，响应信息也包括三部分：

 - 响应行：包括协议版本和状态码，HTTP的响应状态码以“清晰明确”的语言告诉客户端本次请求的处理结果，主要由如下五种情况组成：

  > 1xx 消息，一般是告诉客户端，请求已经收到了，正在处理，别急...
  > 2xx 处理成功，一般表示：请求收悉、我明白你要的、请求已受理、已经处理完成等信息.
  > 3xx 重定向到其它地方。它让客户端再发起一个请求以完成整个处理。
  > 4xx 处理发生错误，责任在客户端，如客户端的请求一个不存在的资源，客户端未被授权，禁止访问等。
  > 5xx 处理发生错误，责任在服务端，如服务端抛出异常，路由出错，HTTP版本不支持等。

 - 响应头：响应头包含了服务器自身的一些信息，比如服务器生成返回数据的时间、返回的数据类型（JSON、HTML、流媒体等类型），以及服务器要在客户端保存的Cookie等信息。
 - 响应体：通常响应体包含了HTML的实际内容。

具体如下图所示：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200217210427948.jpeg)

在此我们需要特别注意一下重定向的情况，即当响应行返回状态码为301时浏览器需要重定向到另一个地址，而需要重定向的地址正是包含在响应头的Location字段中的地址，并使用该地址重新导航，这就是一个完整重定向的流程。

### 7、断开连接

通常情况下一旦服务器向客户端返回了请求数据，客户端就会通过“四次挥手”关闭TCP连接，四次挥手的具体过程如下所示。

**第一次挥手**：在挥手之前服务端与客户端都处于 ESTABLISHED 状态。客户端发送一个 FIN 报文，用来关闭客户端到服务器的数据传输，此时客户端处于 FIN_WAIT_1 状态。

**第二次挥手**：当服务端收到 FIN 之后，会发送 ACK 报文，并且把客户端的序列号值加 1 作为 ACK 报文的序列号值，表明已经收到客户端的报文了，此时服务端处于 CLOSE_WAIT 状态。

**第三次挥手**：如果服务端同意关闭连接，则会向客户端发送一个 FIN 报文，并且指定一个序列号，此时服务端处于 LAST_ACK 的状态。

**第四次挥手**：当客户端收到 ACK 之后，处于 FIN_WAIT_2 状态。待收到 FIN 报文时发送一个 ACK 报文作为应答，并且把服务端的序列号值 +1 作为自己 ACK 报文的序列号值，此时客户端处于 TIME_WAIT 状态。等待一段时间后会进入 CLOSED 状态，等待的时间设为 2MSL，MSL 是 Maximum Segment Lifetime，报文最大生存时间，它是任何报文在网络上存在的最长时间，超过这个时间报文将被丢弃，当服务端收到 ACK 报文之后，也会变为 CLOSED 状态，此时连接正式关闭。

<img src="/assets/client-render-flow/03.png" width="600px" />

为什么建立连接只通信了三次，而断开连接却用了四次？

这是因为当服务端收到客户端的 FIN 报文后，发送的 ACK 报文只是用来应答的，并不表示服务端也希望立即关闭连接。只有当服务端把所有的报文都发送完了，才会发送 FIN 报文，告诉客户端可以断开连接了，因此在断开连接时需要四次挥手。

不过这儿需要注意如果浏览器或者服务器在其头部信息中加入了：

> Connection: Keep-Alive

那么 TCP 连接在发送后将仍然保持打开状态，这样浏览器就可以继续通过同一个 TCP 连接发送请求。保持 TCP 连接可以省去下次请求时需要建立连接的时间，提升资源加载速度。

## 三、响应数据处理

浏览器在接收到服务器返回的响应信息后首先会查看响应头中的 Content-Type 字段来判断响应体数据的类型，然后浏览器会根据 Content-Type 的值来决定如何显示响应体的内容。如果 Content-Type 的值为`application/octet-stream`类型，这表明数据类型为字节流，那么通常情况下浏览器会按照下载类型来处理此类请求，将请求提交给浏览器的下载管理器，如果 Content-Type 的值为`text/html`，那么浏览器会继续进行导航流程，由于Chrome的页面渲染是运行在渲染进程中的，所以接下来就需要准备渲染进程了。

默认情况下每打开一个页面，浏览器就会创建一个新的渲染进程。但是如果当打开的页面属于**同一站点**（同一站点即根域名与协议相同），那么这个页面会同时运行在一个渲染进程中，事实上这跟 Chrome 默认的`process-per-site-instance` 策略有关，也就是当如果从一个页面打开了另一个新页面，而新页面和当前页面属于同一站点的话那么新页面会复用父页面的渲染进程。

但是当渲染进程准备好以后还不能立马进入文档解析阶段，因为此时的文档数据还在网络进程中，并没有提交给渲染进程，下一步就进入了提交文档阶段。

## 四、提交文档

这个阶段是指浏览器进程将网络进程接收到的HTML数据提交给渲染进程，具体流程如下：

- 首先当浏览器进程接收到网络进程的响应头数据中 Content-Type 的值为 `text/html` 之后便会向渲染进程发起“提交文档”的消息；
- 渲染进程接收到“提交文档”的消息后，会和网络进程建立传输数据的“管道”；
- 等文档数据传输完成后，渲染进程会自动返回“确认提交”的消息给浏览器进程；
- 浏览器进程在收到“确认提交”的消息后，会更新浏览器界面状态，包括安全状态、地址栏的URL、前进后退的历史状态，并更新Web页面。

## 五、字节流解码

浏览器进程通过 HTTP 协议接收到的文档内容实际上是二进制字节数据，当渲染进程接收到浏览器传输过来的字节数据流后会通过“[编码嗅探算法](https://html.spec.whatwg.org/multipage/parsing.html#encoding-sniffing-algorithm)”来确定字符编码，然后根据字符编码将字节流数据进行解码，生成字符数据也就是我们编写的代码。

通过解码得到的字符流数据在进入解析环节之前还需要进行一些预处理操作。比如将换行符转换成统一的格式，最终生成规范化的字符流数据，这个把字符数据进行统一格式化的过程称之为“输入流预处理”。

## 六、构建DOM树

由于浏览器无法直接理解HTML字符串，因此将这一系列的字节流转换成一种有意义并且方便操作的数据结构，这种数据结构就是DOM树，DOM树本质上是一个以document为根节点的多叉树。
那么通过什么样的方式来进行解析呢？

**HTML文法的本质**：
首先，我们应该清楚把握一点：HTML的文法并不是上下文无关文法，什么是上下文无关文法呢？
在编译原理学科中对于其的定义为：

> 若一个形式文法G = (N, Σ, P, S) 的产生式规则都取如下的形式：V->w，则叫上下文无关语法。其中 V∈N ，w∈(N∪Σ)* 。

其中把 G = (N, Σ, P, S) 中各个参量的意义解释一下:

- N 是非终结符(顾名思义，就是说最后一个符号不是它, 下面同理)集合。 
- Σ 是终结符集合。
- P 是开始符，它必须属于 N，也就是非终结符。
- S 就是不同的产生式的集合。如 S -> aSb 等等。

通俗一点讲，上下文无关文法就是说这个文法中所有产生式的左边都是一个非终结符，举个例子：

```javascript
A -> B
```
这个文法中，每个产生式左边都会有一个非终结符，这就是上下文无关文法。在这种情况下，`xBy`一定是可以规约出`xAy`的。

```javascript
aA -> B
Aa -> B
```

这种情况就不是上下文无关文法，当遇到B的时候，我们不知道到底能不能规约出A，取决于左边或者右边是否有a存在，也就是说和上下文有关。

那么HTML为什么不是上下文无关文法，实际上规范的 HTML 语法，是符合上下文无关文法的，能够体现它非上下文无关的是不标准的语法。比如解析器扫描到 form 标签的时候，上下文无关文法的处理方式是直接创建对应 form 的 DOM 对象，而真实的 HTML5 场景中却不是这样，解析器会查看 form 的上下文，如果这个 form 标签的父标签也是 form, 那么直接跳过当前的 form 标签，否则才创建 DOM 对象。
常规的编程语言都是上下文无关的，而HTML却相反，也正是它非上下文无关的特性，决定了HTML Parser并不能使用常规编程语言的解析器来完成，需要另辟蹊径。[HTML5 规范](https://html.spec.whatwg.org/multipage/parsing.html)详细地介绍了解析算法。这个算法分为两个阶段:

 - 标记化。 
 - DOM树构建。

分别对应词法分析和语法分析。

### 1、标记化

标记化是词法分析过程，这个算法输入为HTML文本，输出为HTML标记，也称为标记生成器。HTML 标记包括起始标记、结束标记、属性名称和属性值。
标记生成器运用**有限自动状态机**来完成，状态机一共有以下几种状态：

- 数据状态(Data)
- DOCTYPE 状态(DOCTYPE State)
- DOCTYPE 名称之前状态(Before DOCTYPE Name State)
- DOCTYPE 名字状态(DOCTYPE Name State)
- 标签声明(Markup Declaration Open State)
- 标记打开状态(Tag open)
- 标记名称状态(Tag name)
- 标签声明打开状态(Close tag open state)

有限自动状态机在当前状态下，接收一个或多个字符，就会更新到下一个状态，通过4个状态之间的切换标记生成器识别标记，传递给树构造器，然后接受下一个字符以识别下一个标记；如此反复直到输入的结束。
我们通过如下示例演示该过程：

```html
<!DOCTYPE html>
<html lang="en">
  <body>
    hello world
  </body>
</html>
```

1. 初始化为“数据状态”
2. 匹配到字符 <，状态切换到“标签打开状态”
3. 匹配到字符 !，状态切换至“标签声明打开状态”，后续 7 个字符可以组成字符串 DOCTYPE，跳转到“DOCTYPE 状态”
4. 匹配到字符为空格，当前状态切换至“DOCTYPE 名称之前状态”
5. 匹配到字符串 html，创建一个新的 DOCTYPE 标记，标记的名字为 html ，然后当前状态切换至“DOCTYPE 名字状态”
6. 匹配到字符 >，跳转到“数据状态”并且释放当前的 DOCTYPE 标记
7. 遇到 <, 状态切换为“标记打开”
8. 接收`[a-z]`的字符，会进入“标记名称状态”
9. 这个状态一直保持，直到遇到 >，表示标记名称记录完成，这时候变为“数据状态”
10. 接下来遇到 body 标签做同样的处理，这个时候 html 和 body 的标记都记录好了。
11. 现在来到`<body>`中的 >，进入“数据状态”，之后保持这样状态接收后面的字符 hello world。
12. 接着接收 `</body>` 中的 <，回到“标记打开状态”，接收下一个`/`后，这时候会创建一个`end tag`的token。
13. 随后进入“标记名称状态”, 遇到 > 回到“数据状态”。
14. 接着以同样的样式处理 `</html>`。

需要注意如果在 HTML 解析过程中遇到 script 标签，则会发生一些变化。

如果遇到的是内联代码，也就是在 script 标签中直接写代码，那么解析过程会暂停，执行权限会转给 JavaScript 脚本引擎，待 JavaScript 脚本执行完成之后再交由渲染引擎继续解析。有一种情况例外，那就是脚本内容中调用了改变 DOM 结构的 document.write() 函数，此时渲染引擎会回到第二步，将这些代码加入字符流，重新进行解析。

如果遇到的是外链脚本，那么渲染引擎会按照我们在第 01 课时中所述的，根据标签属性来执行对应的操作。

### 2、DOM树构建

树构建的过程也就是语法分析，通过标记生成器生成的标签会发送到DOM树构建器，构建树会根据传过来的标签创建对应的DOM对象，由于DOM 树是一个以document为根节点的多叉树。因此解析器首先会创建一个document对象。标记生成器会把每个标记的信息发送给建树器。建树器接收到相应的标记时，会创建对应的 DOM 对象。

构建算法主要由两部分构成：

- DOM树
- 存放标签名的栈

还以上面的示例演示：

1. 首先，状态为初始化状态。
2. 树构建器接收到 DOCTYPE 令牌后，树构建器会创建一个 DocumentType 节点附加到 Document 节点上，DocumentType 节点的 name 属性为 DOCTYPE 令牌的名称，切换到before html模式
2. 接收到标记生成器传来的`<html>`标签后创建一个 HTMLHtmlElement 的 DOM 元素, 将其加到 document 根对象上，并进行压栈操作，切换为 before head 模式
3. 虽然没有接收到 head 令牌，但仍然会隐式地创建 head 元素并加到 DOM 树和开放元素栈中，切换到 in head 模式
4. 此时从标记生成器那边传来`<body>`标签，表示并没有 head, 这时候建树器会自动创建一个 HTMLHeadElement 并将其加入到 DOM 树中，同时状态切换为 after head
5. 接收到`<body>`标签后会创建 HTMLBodyElement 并插入到DOM树中，同时压入开放标记栈
6. 接着状态变为 in body，然后来接收后面一系列的字符: hello world
7. 接收到第一个字符的时候，会创建一个 Text 节点并把字符插入其中，然后把 Text 节点插入到 DOM 树中 body 元素的下面。随着不断接收后面的字符，这些字符会附在 Text 节点上。
8. 现在，标记生成器传过来一个body的结束标记，即`</body>`标签，进入到 after body 状态。
9. 标记生成器最后传过来一个html的结束标记, 进入到 after after body 的状态，表示解析过程到此结束。

整体流程如下图所示： 
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200229213555644.png) 

但事实上 Chrome 浏览器中 DOM 树构建算法却又与此不同，Chrome 浏览器中 DOM 树构建器主要有以下三部分组成：

 - DOM树
 - 存放标签名的栈
 - 任务队列

我们通过如下示例演示该过程：
```html
<html>
<body>
    <h1>HelloWorld</h1>
    <div>
        <div>
            <p>picture:</p>
            <img src="example.png"/>
        </div>
        <div>
            <p>A paragraph of explanatory text...</p>
        </div>
    </div>
</body>
</html>
```

1. 创建解析器的同时也会创建Document对象。
2. 第一个遇到的开标签是`<html>`标签，解析器会先创建一个html节点，然后把它加到任务队列里面，传递两个参数，第一个参数是父节点，第二个参数是当前节点。
3. 解析器会将html节点压入栈中，这个栈负责存放所有未遇到闭标签的所有开标签。
4. 执行队列里的任务，此时解析器会根据任务类型的不同调用不同的执行函数，由于本次是insert的，它会去执行一个插入的函数，在具体调用前插入函数前它会先去检查父元素是否支持子元素，如果不支持，则直接返回，就像video标签不支持子元素。
5. 在插入函数中首先会对父节点进行判断，如果没有lastChild，会将这个子节点作为firstChild，如果父节点已经有lastChild了，因此会把这个子元素的previousSibling指向老的lastChild，老的lastChild的nexSibling指向它，在我们的示例中由于`<html>`标签并没有兄弟元素，因此html节点将作为Document的firstChild插入。
6. 接下来便是`<body>`标签，解析器同样会创建一个对应的body节点，然后将其加入到任务队列，同时将body节点入栈，由于我们刚刚把html元素压了进去，则栈顶元素为html元素，所以head的父节点就为html。所以每当遇到一个开标签时，就把它压起来，下一次再遇到一个开标签时，它的父元素就是上一个开标签，因此解析器借助一个栈建立起了父子关系。
7. 重复上述步骤直到当解析器遇到一个闭标签时，会把栈里面的元素一直pop出来，直到pop到第一个和它标签名字一样的，在我们的示例中首先接收到一个`</h1>`结束标签，此时查询栈顶元素，恰好与h1属于同种类型的标签，因此解析器将栈顶元素弹出，向DOM树中加入节点h1。
8. 继续向下执行，我们需要注意如果遇到的是没有封闭标签的元素如我们示例中的`<img/>`，则直接加入DOM树中即可，无需入栈。
9. 继续执行直到遇到body闭标签后，此时需要注意解析器并不会把body给pop出来，因为如果body闭标签后面又再写了标签的话，就会自动当成body的子元素。这也是HTML语法的容错机制中的一种，后面的html节点与body相同，但是body节点与html节点依然会加入到DOM树中，至此DOM树便成功生成。

这个时候你可能会有一个问题，为什么要用一个任务队列存放将要插入的结点呢，而不是直接插入呢？一个原因是放到task里面方便统一处理，并且有些任务可能不能立即执行，要先存起来。不过在我们这个案例里面都是存完后下一步就执行了。

### 3、容错机制
HTML相较于XML最突出的就是其强大的宽容策略，也就是其突出的容错能力，事实上你从来不会在一个html页面上看到“无效语法”这样的错误，这是因为浏览器修复了无效内容并继续工作。
以下面这段HTML代码为例：

```html
<html>
	<mytag></mytag>
	<div>
		<p>
	</div>
	Really lousy HTML
	</p>
</html>
```
这段html违反了很多规则：mytag不是合法的标签，p及div错误的嵌套等等，但是浏览器仍然可以没有任何怨言的继续显示，它在解析的过程中修复了html出现的错误。
浏览器都具有错误处理的能力，但是，另人惊讶的是，这并不是html最新规范的内容，就像书签及前进后退按钮一样，它只是浏览器长期发展的结果。一些比较知名的非法html结构，在许多站点中出现过，浏览器都试着以一种和其他浏览器一致的方式去修复。
但是在[Html5规范](https://link.zhihu.com/?target=http://lib.csdn.net/base/html5)定义了这方面的需求，webkit在html解析类开始部分的注释中做了很好的总结。下面是一些webkit容错的经典例子：

#### （1）`</br>`标签会被替换为`<br>`
　一些网站使用`</br>`替代`<br>`，为了兼容IE和Firefox，webkit将其看作`<br>`，对应的源码处理为：
```cpp
if (t->isCloseTag(brTag) && m_document->inCompatMode()) {
  reportError(MalformedBRError);
  t->beginTag = true;
}
```

#### （2）表格离散
这指一个表格嵌套在另一个表格中，但不在它的某个单元格内，例如：

```html
<table>
  <table>
    <tr><td>inner table</td></tr>
  </table>
  <tr><td>outer table</td></tr>
</table>
```

此时webkit将会将嵌套的表格变为两个兄弟表格：

```html
<table>
    <tr><td>outer table</td></tr>
</table>
<table>
    <tr><td>inner table</td></tr>
</table>
```

对应的源码实现为：

```cpp
if (!m_currentFormElement) {
	m_currentFormElement = new HTMLFormElement(formTag,m_document);
}
```

#### （3）表单元素嵌套

如果代码中出现form元素的子元素还是form元素会直接忽略里面的form元素，对应的源码实现为：

```cpp
if (!m_currentFormElement) {
	m_currentFormElement = new HTMLFormElement(formTag,m_document);
}
```

#### （4）太深的标签继承

Webkit最多只允许20个相同类型的标签嵌套，多出来的将被忽略，对应的源码实现为：

```cpp
bool HTMLParser::allowNestedRedundantTag(const AtomicString& tagName){
	unsigned i = 0;
	for (HTMLStackElem* curr = m_blockStack;
		i < cMaxRedundantTagDepth && curr && curr->tagName == tagName;
		curr = curr->next, i++) { }
	return i != cMaxRedundantTagDepth;
}
```

#### （5）错误的html、body闭合标签

此处无须赘述，在我们之前DOM树构建算法时提到我们从来不闭合body与html，因为有时一些网页总是在还未真正结束时就闭合它，又或者在在body或html的闭合标签后再次添加元素标签，事实上我们依赖调用end方法去执行关闭的处理：

```cpp
if (t->tagName == htmlTag || t->tagName == bodyTag )
return;
```

## 七、样式计算

样式计算的目的是计算出 DOM 节点中每个元素的具体样式，这个阶段大体上可以分为如下三步：

### 1、CSS文本转换为styleSheets

和HTML文件一样，浏览器无法直接理解这种纯文本的CSS样式，因此当浏览器渲染引擎接收到CSS文本时会执行一个转换操作，将CSS文本装换成一种浏览器可以理解的结构——styleSheets。我们在控制台输入document.styleSheets可以查看浏览器生成的styleSheets结构：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200412101157718.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDE5NjI5OQ==,size_16,color_FFFFFF,t_70)

那么渲染引擎是如何将CSS文本转换为styleSheets结构呢？

#### （1）加载CSS

博客前文介绍了HTML文档解析构建DOM树的过程，事实上如果在DOM树的构建过程中遇到link标签，当把它插入到DOM里面后就会触发资源加载，如下所示：

```html
<link rel="stylesheet" href="demo.css">
```

上面的 rel 属性表明它是一个样式文件，href 属性表明了其资源加载的链接路径，这个资源加载是一个异步加载，并不会影响 DOM 树的构建，但是我们需要注意到在 CSS 没有处理好之前构建好的 DOM 并不会显示出来，以下面代码为例：

```html
<!DOCType html>
<html>
<head>
    <link rel="stylesheet" href="demo.css">
</head>
<body>
<div class="text">
    <p>hello, world</p>
</div>
</body>
```
dom.css文件的内容如下：

```css
.text{
    font-size: 20px;
}
.text p{
    color: #505050;
}
```

整体解析过程如下图所示：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200412162530690.png)

在CSS没有加载好之前，DOM 树已经构建好了。为什么 DOM 构建好不直接开始页面的渲染呢？这是因为没有样式的页面将无比混乱。所以CSS不能太大，页面一打开将会停留较长时间的白屏，所以把图片/字体等转成 base64 放到CSS里面是一种不太推荐的做法。

#### （2）解析CSS

**1、字符流 -> tokens**

CSS 解析和 html 解析的过程有很多相似的地方，都是先将字符串格式化为tokens。CSS token 定义了很多种类型，如下所示的CSS样式会被拆成多个 token：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200412170323745.png)

**2、tokens -> styleRule**

每个 styleRule主要包含两个部分，一个是选择器 selectors，第二个是属性集 properties。举个🌰:

```css
.text .hello{
    color: rgb(200, 200, 200);
    width: calc(100% - 20px);
}
 
#world{
    margin: 20px;
}
```

解析生成的选择器结果如下所示：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200412172112662.png)

从上图我们也可以看出选择器的解析是从右向左进行的，因此先识别出的是 .hello 选择器，其次才是 .text 选择器。同时我们也注意到了解析结果中的 matchType 和 relation 字段，那么这两个字段分别代表着什么呢？

Chrome 定义了如下 matchType，是选择器匹配元素的主要方式：

```c
enum MatchType {
    Unknown,
    Tag,               // Example: div
    Id,                // Example: #id
    Class,             // example: .class
    PseudoClass,       // Example:  :nth-child(2)
    PseudoElement,     // Example: ::first-line
    PagePseudoClass,   // ??
    AttributeExact,    // Example: E[foo="bar"]
    AttributeSet,      // Example: E[foo]
    AttributeHyphen,   // Example: E[foo|="bar"]
    AttributeList,     // Example: E[foo~="bar"]
    AttributeContain,  // css3: E[foo*="bar"]
    AttributeBegin,    // css3: E[foo^="bar"]
    AttributeEnd,      // css3: E[foo$="bar"]
    FirstAttributeSelectorMatch = AttributeExact,
};
```

Chrome 定义的 relationType如下所示，表明选择器类型：

```c
enum RelationType {
    SubSelector,       // No combinator
    Descendant,        // "Space" combinator
    Child,             // > combinator
    DirectAdjacent,    // + combinator
    IndirectAdjacent,  // ~ combinator
    // Special cases for shadow DOM related selectors.
    ShadowPiercingDescendant,  // >>> combinator
    ShadowDeep,                // /deep/ combinator
    ShadowPseudo,              // ::shadow pseudo element
    ShadowSlot                 // ::slotted() pseudo element
  };
```

`.text .hello`中的`.hello`选择器的类型就是 Descendant，即后代选择器。记录选择器类型的作用是协助判断当前元素是否 match 这个选择器。例如，由于`.hello`是一个后代选择器，所以它从右往左的下一个选择器就是它的父选择器，于是接下来便会判断当前元素的父元素是否匹配`.text`这个选择器。

解析生成的属性集结果如下图所示：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200412173721228.png)

如上图所示，解析生成的所有属性都是以id来标志，上面的id分别对应：

```c
enum CSSPropertyID {
    CSSPropertyColor = 15,
    CSSPropertyWidth = 316,
    CSSPropertyMarginLeft = 145,
    CSSPropertyMarginRight = 146,
    CSSPropertyMarginTop = 147,
    CSSPropertyMarkerEnd = 148,
}
```

我们发现`margin: 20px;`解析过程会转化成四个属性。从这里可以看出 CSS 提倡属性合并，但是最后还是会被拆成各个小属性。所以属性合并最大的作用应该在于减少 CSS 的代码量。

**一个选择器和一个属性集就构成一条rule，同一个css表的所有rule放到同一个stylesheet对象里面**，Chrome 会把用户的样式存放到一个 m_authorStyleSheets 的向量里面，如下图示意：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200412174539228.png)

除了 autherStyleSheet，还有浏览器默认的样式 DefaultStyleSheet，这里面有几张，最常见的是 UAStyleSheet，其它的还有svg和全屏的默认样式表。

**3、生成哈希map**

最后会把生成的rule集的引用存放到四个类型哈希map：

```c
CompactRuleMap m_idRules;
CompactRuleMap m_classRules;
CompactRuleMap m_tagRules;
CompactRuleMap m_shadowPseudoElementRules;
```
map的类型是根据最右边的selector的类型：id、class、标签、伪类选择器区分的，这样做的目的是为了在比较的时候能够很快地取出匹配第一个选择器的所有rule，然后每条rule再检查它的下一个selector是否匹配当前元素。

#### （3）标准化属性值
现在我们已经把现有的CSS文本转化为浏览器可以理解的结构，那么接下来我们就需要对其进行属性值的标准化操作，我们以如下CSS文本示例：

```css
body { font-size: 2em }
p { color: blue }
span { display: none }
div { font-weight: bold }
div p { color: green }
div { color: red }
```
示例中的许多属性值诸如2em，blue，bold这些数值不容易被渲染引擎理解，因此我们需要把所有值转换为渲染引擎容易理解的、标准化的计算值，上述示例经过标准化后的结果如下图：

```css
body { font-size: 32px }
p { color: rgb(0, 0, 255) }
span { display: none }
div { font-weight: 700}
div p { color: rgb(0, 128, 0)}
div { color: rgb(255, 0, 0) }
```

### 2、计算DOM树中每个节点的具体样式

CSS表解析好之后，会触发layout tree，即开始构建布局树，进行布局的时候，会把每个可视的 Node 节点相应地创建一个 Layout 节点，而创建 Layout 节点的时候需要计算一下得到它的 style。为什么需要计算 style，因为可能会有多个选择器的样式命中了它，所以需要把几个选择器的样式属性综合在一起，以及继承父元素的属性以及 UserAgent 样式（浏览器内置的默认样式）。这个过程包括两步：找到命中的选择器和设置样式。

#### （1）选择器命中判断

我们以用下代码示例：

```html
<style>
.text{
    font-size: 22em;
}
.text p{
    color: #505050;
}
</style>
<div class="text">
    <p>hello, world</p>
</div>
```

上面会生成两个 rule，第一个 rule 会放到上面提到的四个哈希 map 其中的 classRules 里面，而第二个 rule 会放到 tagRules 里面。

当这个样式表解析好时，触发 layout，这个 layout 从 document 节点开始递归遍历所有DOM节点，更新所有的DOM元素的布局。对于每个node节点，代码会按照id、class、伪元素、标签的顺序取出所有选择器，进行比较判断，最后是通配符，具体实现源码如下：

```cpp
//如果节点有id属性
if (element.hasID()) 
  collectMatchingRulesForList(
      matchRequest.ruleSet->idRules(element.idForStyleResolution()),
      cascadeOrder, matchRequest);

//如果节点有class属性
if (element.isStyledElement() && element.hasClass()) { 
  for (size_t i = 0; i < element.classNames().size(); ++i)
    collectMatchingRulesForList(
        matchRequest.ruleSet->classRules(element.classNames()[i]),
        cascadeOrder, matchRequest);
}

//伪类的处理
...

//标签选择器处理
collectMatchingRulesForList(
    matchRequest.ruleSet->tagRules(element.localNameForSelectorMatching()),
    cascadeOrder, matchRequest);

//最后是通配符
...
```

在遇到`div.text`这个元素的时候，会去执行上面代码的取出 classRules 的那行。

上面 demo 的rule只有两个，一个是classRule，一个是tagRule。所以会对取出来的这个classRule进行检验：

```cpp
if (!checkOne(context, subResult))
  return SelectorFailsLocally;
if (context.selector->isLastInTagHistory()) { 
    return SelectorMatches;
}
```

第一行先对当前选择器(.text)进行检验，如果不通过，则直接返回不匹配，如果通过了，第三行判断当前选择器是不是最左边的选择器，如果是的话，则返回匹配成功。如果左边还有限定的话，那么再递归检查左边的选择器是否匹配。
我们先来看一下第一行的checkOne是怎么检验的：

```cpp
switch (selector.match()) { 
  case CSSSelector::Tag:
    return matchesTagName(element, selector.tagQName());
  case CSSSelector::Class:
    return element.hasClass() &&
           element.classNames().contains(selector.value());
  case CSSSelector::Id:
    return element.hasID() &&
           element.idForStyleResolution() == selector.value();
}
```

很明显，`.text`将会在上面第6行匹配成功，并且它左边没有限定了，所以返回匹配成功。

到了检验p标签的时候，会取出`.text p`的rule，它的第一个选择器是p，将会在上面代码的第3行判断成立。但由于它前面还有限定，于是它还得继续检验前面的限定成不成立。

前一个选择器的检验关键是靠当前选择器和它的关系，即上面提到的 relationType，这里的 p 的 relationType 是 Descendant 即后代。上面在调了checkOne成功之后，继续往下走：

```cpp
switch (relation) { 
  case CSSSelector::Descendant:
    for (nextContext.element = parentElement(context); nextContext.element;
         nextContext.element = parentElement(nextContext)) { 
      MatchStatus match = matchSelector(nextContext, result);
      if (match == SelectorMatches || match == SelectorFailsCompletely)
        return match;
      if (nextSelectorExceedsScope(nextContext))
        return SelectorFailsCompletely;
    } 
    return SelectorFailsCompletely;
      case CSSSelector::Child:
    //...
}
```

由于这里是一个后代选择器，所以它会循环当前元素所有父节点，用这个父节点和第二个选择器`.text`再执行 checkOne 的逻辑，checkOne 将返回成功，并且它已经是最后一个选择器了，所以判断结束，返回成功匹配。

后代选择器会去查找它的父节点 ，而其它的 relationType 会相应地去查找关联的元素。所以不提倡把选择器写得太长，特别是用 sass/less 写的时候，新手很容易写嵌套很多层，这样会增加查找匹配的负担。例如上面，它需要对下一个父代选器启动一个新的递归的过程，而递归是一种比较耗时的操作。一般是不要超过三层。

上面已经较完整地介绍了匹配的过程，接下来分析匹配之后又是如何设置 style 的。

#### （2）设置style

设置style的顺序是先继承父节点，然后使用 UserAgent 的样式，最后再使用用户的 style（这个过程正好解释了CSS的继承规则和层叠规则）：

```cpp
style->inheritFrom(*state.parentStyle())
matchUARules(collector);
matchAuthorRules(*state.element(), collector);
```

每一步如果有styleRule匹配成功的话会把它放到当前元素的 m_matchedRules 的向量里面，并会去计算它的优先级，记录到 m_specificity 变量。这个优先级是怎么算的呢？

```cpp
for (const CSSSelector* selector = this; selector;
     selector = selector->tagHistory()) { 
  temp = total + selector->specificityForOneSelector();
}
return total;
```

如上代码所示，它会从右到左取每个selector的优先级之和。不同类型的selector的优级级定义如下：

```cpp
switch (m_match) {
    case Id: 
      return 0x010000;
    case PseudoClass:
      return 0x000100;
    case Class:
    case PseudoElement:
    case AttributeExact:
    case AttributeSet:
    case AttributeList:
    case AttributeHyphen:
    case AttributeContain:
    case AttributeBegin:
    case AttributeEnd:
      return 0x000100;
    case Tag:
      return 0x000001;
    case Unknown:
      return 0;
  }
  return 0;
}
```

其中ID选择器的优先级为 0x10000 = 65536，类、属性、伪类的优先级为 0x100 = 256，标签选择器的优先级为1。如下面计算所示：

```css
/*优先级为257 = 265 + 1*/
.text h1{
    font-size: 8em;
}
 
/*优先级为65537 = 65536 + 1*/
#my-text h1{
    font-size: 16em;
}
```

当 match 完当前元素的所有CSS规则，全部放到了 collector 的 m_matchedRules 里面，再把这个向量根据优先级从小到大排序，如果大小相同则比较它们的位置，出现位置靠后的选择器排在后面。

把 css 表的样式处理完了之后，Chrome 再去取 style 的内联样式（这个在已经在构建DOM的时候存放好了），把内联样式 push 到上面排好序的容器里，由于它是由小到大排序的，所以放最后面的优先级肯定是最大的，因此内联样式的优先级要高于所有的css文本样式。

样式里面的important的优先级又是怎么处理的？

实际上浏览器会先设置正常的规则，最后再设置`!important`的规则。所以越往后的设置的规则就会覆盖前面设置的规则。

最后生成的 Style 是怎么样的？

按优先级计算出来的 Style 会被放在一个 ComputedStyle 的对象里面，这个 style 里面的规则分成了几类，通过检查 style 对象可以一窥：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200413105511207.png)

把它画成一张图表：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200413105528396.png)

主要有几类，box 是长宽，surround 是 margin/padding，还有不可继承的 nonInheritedData 和可继承的 styleInheritedData 一些属性。Chrome 还把很多比较少用的属性放到 rareData 的结构里面，为避免实例化这些不常用的属性占了太多的空间。

具体来说，上面设置的font-size为：22em * 16px = 352px：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200413105611166.png)

而所有的色值会变成16进制的整数，如 Chrome 定义的两种颜色的色值：

```cpp
static const RGBA32 lightenedBlack = 0xFF545454;
static const RGBA32 darkenedWhite = 0xFFABABAB;
```

同时 Chrome 对 rgba 色值的转化算法：

```cpp
RGBA32 makeRGBA32FromFloats(float r, float g, float b, float a) {
  return colorFloatToRGBAByte(a) << 24 | colorFloatToRGBAByte(r) << 16 |
         colorFloatToRGBAByte(g) << 8 | colorFloatToRGBAByte(b);
}
```

从这里可以看到，有些CSS优化建议说要按照下面的顺序书写CSS规则：

> 1.位置属性(position, top, right, z-index, display, float等)
> 2.大小(width, height, padding, margin)
> 3.文字系列(font, line-height, letter-spacing, color- text-align等)
> 4.背景(background, border等)
> 5.其他(animation, transition等)

这些顺序对浏览器来说其实是一样的，因为最后都会放到computedStyle里面，而这个style里面的数据是不区分先后顺序的。所以这种建议与其说是优化，倒不如说是规范，大家都按照这个规范写的话，看CSS就可以一目了然，可以很快地看到想要了解的关键信息。

#### （3）调整style
最后把生成的style做一个调整：

```cpp
adjustComputedStyle(state, element); //style在state对象里面
```

调整的内容包括：

第一个：把 absolute/fixed 定位、float 的元素设置成 block：

```cpp
// Absolute/fixed positioned elements, floating elements and the document
// element need block-like outside display.
if (style.hasOutOfFlowPosition() || style.isFloating() ||
    (element && element->document().documentElement() == element))
  style.setDisplay(equivalentBlockDisplay(style.display()));
```

第二个，如果有 :first-letter 选择器时，会把元素 display 和 position 做调整：

```cpp
static void adjustStyleForFirstLetter(ComputedStyle& style) {
  // Force inline display (except for floating first-letters).
  style.setDisplay(style.isFloating() ? EDisplay::Block : EDisplay::Inline);
  // CSS2 says first-letter can't be positioned.
  style.setPosition(StaticPosition);
}
```

还会对表格元素做一些调整。

至此，浏览器就完成了样式属性的解析并根据继承与层叠两个基本规则完成了 DOM 树每个节点最终样式结果的生成，接下来就是布局阶段。

## 八、布局阶段

现在，我们有 DOM 树和 DOM 树中元素的样式，但这还不足以显示页面，因为我们还不知道 DOM 元素的几何位置信息。那么接下来就需要计算出 DOM 树中可见元素的几何位置，我们把这个计算过程叫做布局。这个阶段 Chrome 主要完成两件工作：创建布局树和布局计算。

### 1、创建布局树

事实上构建完成的 DOM 树还含有很多不可见的元素，比如 head 标签，比如使用了 display:none 属性的元素等等。所以在显示之前，浏览器还要额外地构建一棵只包含可见元素布局树。

<img src="/assets/client-render-flow/04.png" width="600px" />

如上图所示，为了构建布局树，浏览器大体上完成了下面这些工作：

- 遍历 DOM 树中的所有可见节点，并把这些节点加到布局树中；
- 而不可见的节点会被布局树忽略掉，如 head 标签下面的全部内容，再比如 body.p.span 这个元素，因为它的属性包含 dispaly:none，所以这个元素也没有被包进布局树。

### 2、布局计算

现在我们有了一棵完整的布局树。那么接下来，就要计算布局树节点的坐标位置，在执行布局操作的时候，会把布局运算的结果重新写回布局树中，所以布局树既是输入内容也是输出内容，这也是布局阶段一个不合理的地方，因为在布局阶段并没有清晰地将输入内容和输出内容区分开来。针对这个问题，Chrome 团队正在重构布局代码，下一代布局系统叫 LayoutNG，试图更清晰地分离输入和输出，从而让新设计的布局算法更加简单。

## 九、分层

在实际页面中往往有很多复杂的效果，如一些复杂的 3D 变换、页面滚动，或者使用 z-indexing 做 z 轴排序等等，为了更加方便地实现这些效果，渲染引擎还需要为特定的节点生成专用的图层，并生成一棵对应的图层树（LayerTree），这些图层按照一定顺序叠加在一起，就形成了最终的页面。

也就是说在分层的这个阶段需要对布局树解析生成对应的图层树，通常情况下，并不是布局树的每个节点都包含一个图层，如果一个节点没有对应的层，那么这个节点就从属于父节点的图层。那么需要满足什么条件，渲染引擎才会为特定的节点创建新的图层呢？通常满足下面两点中任意一点的元素就可以被提升为单独的一个图层。

### 1、拥有层叠上下文属性的元素

页面是个二维平面，但是层叠上下文能够让 HTML 元素具有三维概念，这些 HTML 元素按照自身属性的优先级分布在垂直于这个二维平面的 z 轴上。文档中的层叠上下文由满足以下任意一个条件的元素形成：

- 文档根元素（<html>）；
- position 值为 absolute（绝对定位）或  relative（相对定位）且 z-index 值不为 auto 的元素；
- position 值为 fixed（固定定位）或 sticky（粘滞定位）的元素（沾滞定位适配所有移动设备上的浏览器，但老的桌面浏览器不支持）；
- flex (flexbox (en-US)) 容器的子元素，且 z-index 值不为 auto；
- grid (grid) 容器的子元素，且 z-index 值不为 auto；
- opacity 属性值小于 1 的元素（参见 the specification for opacity）；
- mix-blend-mode 属性值不为 normal 的元素；
- 以下任意属性值不为 none 的元素：
  - transform
  - filter
  - perspective
  - clip-path
  - mask / mask-image / mask-border
- isolation 属性值为 isolate 的元素；
- -webkit-overflow-scrolling 属性值为 touch 的元素；
- will-change 值设定了任一属性而该属性在 non-initial 值时会创建层叠上下文的元素；
- contain 属性值为 layout、paint 或包含它们其中之一的合成值（比如 contain: strict、contain: content）的元素。

在层叠上下文中，子元素同样也按照上面解释的规则进行层叠。 重要的是，其子级层叠上下文的 z-index 值只在父级中才有意义。子级层叠上下文被自动视为父级层叠上下文的一个独立单元。

总结:

1. 层叠上下文可以包含在其他层叠上下文中，并且一起创建一个层叠上下文的层级。
2. 每个层叠上下文都完全独立于它的兄弟元素：当处理层叠时只考虑子元素。
3. 每个层叠上下文都是自包含的：当一个元素的内容发生层叠后，该元素将被作为整体在父级层叠上下文中按顺序进行层叠。

### 2、需要剪裁（clip）的地方也会被创建为图层。

不过首先你需要了解什么是剪裁，结合下面的 HTML 代码：

```html
<style>
  div {
    width: 200;
    height: 200;
    overflow: auto;
    background: gray;
  } 
</style>
<body>
  <div >
    <p>所以元素有了层叠上下文的属性或者需要被剪裁，那么就会被提升成为单独一层，你可以参看下图：</p>
    <p>从上图我们可以看到，document层上有A和B层，而B层之上又有两个图层。这些图层组织在一起也是一颗树状结构。</p>
    <p>图层树是基于布局树来创建的，为了找出哪些元素需要在哪些层中，渲染引擎会遍历布局树来创建层树（Update LayerTree）。</p> 
  </div>
</body>
```

在这里我们把 div 的大小限定为 200 * 200 像素，而 div 里面的文字内容比较多，文字所显示的区域肯定会超出 200 * 200 的面积，这时候就产生了剪裁，渲染引擎会把裁剪文字内容的一部分用于显示在 div 区域，下图是运行时的执行结果：

<img src="/assets/client-render-flow/05.png" />

出现这种裁剪情况的时候，渲染引擎会为文字部分单独创建一个层，如果出现滚动条，滚动条也会被提升为单独的层。你可以参考下图：

<img src="/assets/client-render-flow/06.png" />

所以说，元素有了层叠上下文的属性或者需要被剪裁，满足其中任意一点，就会被提升成为单独一层。

## 十、图层绘制

在完成图层树的构建之后，渲染引擎需要对图层树中的每个图层进行绘制，引擎会把一个图层的绘制拆分成很多小的绘制指令，然后再把这些指令按照顺序组成一个待绘制列表，绘制列表中的指令其实非常简单，就是让其执行一个简单的绘制操作，比如绘制粉色矩形或者黑色的线等。而绘制一个元素通常需要好几条绘制指令，因为每个元素的背景、前景、边框都需要单独的指令去绘制。所以在图层绘制阶段，输出的内容就是这些待绘制列表。

我们可以打开“开发者工具”的“Layers”标签，选择“document”层，来实际体验下绘制列表，如下图所示：

<img src="/assets/client-render-flow/07.png" />

在该图中，区域 1 就是 document 的绘制列表，拖动区域 2 中的进度条可以重现列表的绘制过程。

## 十一、栅格化操作

绘制列表只是用来记录绘制顺序和绘制指令的列表，而实际上绘制操作是由渲染引擎中的合成线程来完成的, 因此当图层的绘制列表准备好之后，主线程会把该绘制列表提交（commit）给合成线程，再介绍具体绘制流程之前我们需要先了解一个概念：视口（ViewPort），即屏幕上页面的可见区域，通常一个页面可能很大，但是用户只能看到其中的一部分，我们把用户可以看到的这个部分叫做视口。在有些情况下，有的图层可以很大，比如有的页面你使用滚动条要滚动好久才能滚动到底部，但是通过视口，用户只能看到页面的很小一部分，所以在这种情况下，要绘制出所有图层内容的话，就会产生太大的开销，而且也没有必要。基于这个原因，合成线程会将图层划分为图块（tile），这些图块的大小通常是 256x256 或者 512x512，如下图所示：

<img src="/assets/client-render-flow/08.png" />

然后合成线程会按照视口附近的图块来优先生成位图，实际生成位图的操作是由栅格化来执行的。所谓栅格化，是指将图块转换为位图。而图块是栅格化执行的最小单位。渲染进程维护了一个栅格化的线程池，所有的图块栅格化都是在线程池内执行的，运行方式如下图所示：

<img src="/assets/client-render-flow/09.png" />

通常，栅格化过程都会使用 GPU 来加速生成，使用 GPU 生成位图的过程叫快速栅格化，或者 GPU 栅格化，生成的位图被保存在 GPU 内存中。GPU 操作是运行在 GPU 进程中，如果栅格化操作使用了 GPU，那么最终生成位图的操作是在 GPU 中完成的，这就涉及到了跨进程操作。具体形式你可以参考下图:

<img src="/assets/client-render-flow/10.png" />

从图中可以看出，渲染进程把生成图块的指令发送给 GPU，然后在 GPU 中执行生成图块的位图，并保存在 GPU 的内存中。

## 十二、合成与显示

一旦所有图块都被光栅化，合成线程就会生成一个绘制图块的命令——“DrawQuad”，然后将该命令提交给浏览器进程。

浏览器进程里面有一个叫 viz 的组件，用来接收合成线程发过来的 DrawQuad 命令，然后根据 DrawQuad 命令，将其页面内容绘制到内存中，最后再将内存显示在屏幕上。到这里，经过这一系列的阶段，编写好的 HTML、CSS、JavaScript 等文件，经过浏览器就会显示出漂亮的页面了。

## 参考资料

极客时间《浏览器工作原理与实践》专栏
拉勾教育《前端高手进阶》专栏
[从Chrome源码看浏览器如何计算CSS](https://zhuanlan.zhihu.com/p/25380611)