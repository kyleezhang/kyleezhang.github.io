---
title: 浏览器缓存
date: 2021-03-16 19:36:17
toc: true
categories: 
- 前端
tags:
- Chrome
---

Web 缓存按存储位置来区分，包括数据库缓存、服务端缓存、CDN 缓存和浏览器缓存，浏览器缓存的实现方式主要有两种：HTTP 和 ServiceWorker 。

## HTTP 缓存

使用缓存最大的问题往往不在于将资源缓存在什么位置或者如何读写资源，而在于如何保证缓存与实际资源一致的同时，提高缓存的命中率。也就是说尽可能地让浏览器从缓存中获取资源，但同时又要保证被使用的缓存与服务端最新的资源保持一致。

为了达到这个目的，需要制定合适的缓存过期策略（简称“缓存策略”），HTTP 支持的缓存策略有两种：**强制缓存**和**协商缓存**。

<!-- more -->

### 1、强制缓存

强制缓存是在浏览器加载资源的时候，先直接从缓存中查找请求结果，如果不存在该缓存结果，则直接向服务端发起请求。

**(1) Expires**

HTTP/1.0 中可以使用响应头部字段 Expires 来设置缓存时间，它对应一个未来的时间戳。客户端第一次请求时，服务端会在响应头部添加 Expires 字段。当浏览器再次发送请求时，先会对比当前时间和 Expires 对应的时间，如果当前时间早于 Expires 时间，那么直接使用缓存；反之，需要再次发送请求。

```
Expires: Tue Mar 16 2021 00:00:00 GMT
```

上述 Expires 信息告诉浏览器：在 2021.03.16 日之前，可以直接使用该请求的缓存。但是使用 Expires 响应头时容易产生一个问题，那就是服务端和浏览器的时间很可能不同，因此这个缓存过期时间容易出现偏差。同样的，客户端也可以通过修改系统时间来继续使用缓存或提前让缓存失效。

为了解决这个问题，HTTP/1.1 提出了 Cache-Control 响应头部字段。

**(2) Cache-Control**

它的常用值有下面几个：

- no-cache，表示使用协商缓存，即每次使用缓存前必须向服务端确认缓存资源是否更新；
- no-store，禁止浏览器以及所有中间缓存存储响应内容；
- public，公有缓存，表示可以被代理服务器缓存，可以被多个用户共享；
- private，私有缓存，不能被代理服务器缓存，不可以被多个用户共享；
- max-age，以秒为单位的数值，表示缓存的有效时间；
- must-revalidate，当缓存过期时，需要去服务端校验缓存的有效性。

这几个值可以组合使用，比如像下面这样：

```
cache-control: public max-age=31536000
```

告诉浏览器该缓存为公有缓存，有效期 1 年。

需要注意的是，cache-control 的 max-age 优先级高于 Expires，也就是说如果它们同时出现，浏览器会使用 max-age 的值。

### 2、协商缓存

协商缓存的更新策略是不再指定缓存的有效时间了，而是浏览器直接发送请求到服务端进行确认缓存是否更新，如果请求响应返回的 HTTP 状态为 304，则表示缓存仍然有效。控制缓存的难题就是从浏览器端转移到了服务端。

**(1) Last-Modified 和 If-Modified-Since**

服务端要判断缓存有没有过期，只能将双方的资源进行对比。若浏览器直接把资源文件发送给服务端进行比对的话，网络开销太大，而且也会失去缓存的意义，所以显然是不可取的。有一种简单的判断方法，那就是通过响应头部字段 Last-Modified 和请求头部字段 If-Modified-Since 比对双方资源的修改时间。

具体工作流程如下：

1. 浏览器第一次请求资源，服务端在返回资源的响应头中加入 Last-Modified 字段，该字段表示这个资源在服务端上的最近修改时间；

2. 当浏览器再次向服务端请求该资源时，请求头部带上之前服务端返回的修改时间，这个请求头叫 If-Modified-Since；

3. 服务端再次收到请求，根据请求头 If-Modified-Since 的值，判断相关资源是否有变化，如果没有，则返回 304 Not Modified，并且不返回资源内容，浏览器使用资源缓存值；否则正常返回资源内容，且更新 Last-Modified 响应头内容。

这种方式虽然能判断缓存是否失效，但也存在两个问题：

1. 精度问题，Last-Modified 的时间精度为秒，如果在 1 秒内发生修改，那么缓存判断可能会失效；

2. 准度问题，考虑这样一种情况，如果一个文件被修改，然后又被还原，内容并没有发生变化，在这种情况下，浏览器的缓存还可以继续使用，但因为修改时间发生变化，也会重新返回重复的内容。

**(2) ETag 和 If-None-Match**

为了解决精度问题和准度问题，HTTP 提供了另一种不依赖于修改时间，而依赖于文件哈希值的精确判断缓存的方式，那就是响应头部字段 ETag 和请求头部字段 If-None-Match。

具体工作流程如下：

1. 浏览器第一次请求资源，服务端在返响应头中加入 Etag 字段，Etag 字段值为该资源的哈希值；

2. 当浏览器再次跟服务端请求这个资源时，在请求头上加上 If-None-Match，值为之前响应头部字段 ETag 的值；

3. 服务端再次收到请求，将请求头 If-None-Match 字段的值和响应资源的哈希值进行比对，如果两个值相同，则说明资源没有变化，返回 304 Not Modified；否则就正常返回资源内容，无论是否发生变化，都会将计算出的哈希值放入响应头部的 ETag 字段中。

这种缓存比较的方式也会存在一些问题，具体表现在以下两个方面。

1. 计算成本。生成哈希值相对于读取文件修改时间而言是一个开销比较大的操作，尤其是对于大文件而言。如果要精确计算则需读取完整的文件内容，如果从性能方面考虑，只读取文件部分内容，又容易判断出错。

2. 计算误差。HTTP 并没有规定哈希值的计算方法，所以不同服务端可能会采用不同的哈希值计算方式。这样带来的问题是，同一个资源，在两台服务端产生的 Etag 可能是不相同的，所以对于使用服务器集群来处理请求的网站来说，使用 Etag 的缓存命中率会有所降低。

需要注意的是，强制缓存的优先级高于协商缓存，在协商缓存中，Etag 优先级比 Last-Modified 高。既然协商缓存策略也存在一些缺陷，那么我们转移到浏览器端看看 ServiceWorker 能不能给我们带来惊喜。

## ServiceWorker

ServiceWorker 是浏览器在后台独立于网页运行的脚本，也可以这样理解，它是浏览器和服务端之间的代理服务器，是一种可编程网络代理，使我们能够控制页面所发送网络请求的处理方式。。ServiceWorker 非常强大，可以实现包括推送通知和后台同步等功能，更多功能还在进一步扩展，但其最主要的功能是通过拦截拦截浏览器请求并返回缓存的资源文件从而实现**离线缓存**。

### 1、使用限制

越强大的东西往往越危险，所以浏览器对 ServiceWorker 做了很多限制：

1. 在 ServiceWorker 中无法直接访问 DOM，但可以通过 postMessage 接口发送的消息来与其控制的页面进行通信，页面可在必要时对 DOM 执行操作；

2. ServiceWorker 只能在本地环境下或 HTTPS 网站中使用；

3. ServiceWorker 有作用域的限制，一个 ServiceWorker 脚本只能作用于当前路径及其子路径；

4. 由于 ServiceWorker 属于实验性功能，所以兼容性方面会存在一些问题，可以在 Jake Archibald 的 [is Serviceworker ready](https://jakearchibald.github.io/isserviceworkerready/#moar) 网站上查看浏览器的具体支持情况。

5. Service Worker 在不用时会被中止，并在下次有需要时重启，因此不能依赖 Service Worker onfetch 和 onmessage 处理程序中的全局状态。 如果存在需要持续保存并在重启后加以重用的信息可通过访问 IndexedDB API 实现。

### 2、使用方法

通常遵循以下基本步骤来使用 service workers：

1. service worker URL 通过 serviceWorkerContainer.register() 来获取和注册。

2. 如果注册成功，service worker 就在 ServiceWorkerGlobalScope 环境中运行； 这是一个特殊类型的 worker 上下文运行环境，与主运行线程（执行脚本）相独立，同时也没有访问 DOM 的能力。

3. service worker 现在可以处理事件了。

4. 受 service worker 控制的页面打开后会尝试去安装 service worker。最先发送给 service worker 的事件是安装事件(在这个事件里可以开始进行填充 IndexDB 和缓存站点资源)。这个流程同原生 APP 或者 Firefox OS APP 是一样的 — 让所有资源可离线访问。

5. 当 oninstall 事件的处理程序执行完毕后，可以认为 service worker 安装完成了。

6. 下一步是激活。当 service worker 安装完成后，会接收到一个激活事件(activate event)。 onactivate 主要用途是清理先前版本的 service worker 脚本中使用的资源。

7. Service Worker 现在可以控制页面了，但仅是在 register()  成功后的打开的页面。也就是说，页面起始于有没有 service worker ，且在页面的接下来生命周期内维持这个状态。所以，页面不得不重新加载以让 service worker 获得完全的控制。

下图展示了 service worker 所有支持的事件：

<img src="/assets/client-cache/01.png" />

下面我们通过示例来具体演示，在使用 ServiceWorker 脚本之前先要通过“注册”的方式加载它。常见的注册代码如下所示：

```js
if ('serviceWorker' in window.navigator) {

  window.navigator.serviceWorker

    .register('./sw.js')

    .then(console.log)

    .catch(console.error)

} else {

  console.warn('浏览器不支持 ServiceWorker!')
}
```

首先考虑到浏览器的兼容性，判断 window.navigator 中是否存在 serviceWorker 属性，然后通过调用这个属性的 register 函数来告诉浏览器 ServiceWorker 脚本的路径。

浏览器获取到 ServiceWorker 脚本之后会进行解析，解析完成会进行安装。可以通过监听 “install” 事件来监听安装，但这个事件只会在第一次加载脚本的时候触发。要让脚本能够监听浏览器的网络请求，还需要激活脚本。

在脚本被激活之后，我们就可以通过监听 fetch 事件来拦截请求并加载缓存的资源了。

下面是一个利用 ServiceWorker 内部的 caches 对象来缓存文件的示例代码。

```js
const CACHE_NAME = 'ws'
let preloadUrls = ['/index.css']

self.addEventListener('install', function (event) {
  event.waitUntil(
    caches.open(CACHE_NAME)
    .then(function (cache) {
      return cache.addAll(preloadUrls);
    })
  );
});

self.addEventListener('fetch', function (event) {
  event.respondWith(
    caches.match(event.request)
    .then(function (response) {
      if (response) {
        return response;
      }
      return caches.open(CACHE_NAME).then(function (cache) {
          const path = event.request.url.replace(self.location.origin, '')
          return cache.add(path)
        })
        .catch(e => console.error(e))
    })
  );
})
```

这段代码首先监听 install 事件，在回调函数中调用了 event.waitUntil() 函数并传入了一个 Promise 对象。event.waitUntil 用来监听多个异步操作，包括缓存打开和添加缓存路径。如果其中一个操作失败，则整个 ServiceWorker 启动失败。

然后监听了 fetch 事件，在回调函数内部调用了函数 event.respondWith() 并传入了一个 Promise 对象，当捕获到 fetch 请求时，会直接返回 event.respondWith 函数中 Promise 对象的结果。

在这个 Promise 对象中，我们通过 caches.match 来和当前请求对象进行匹配，如果匹配上则直接返回匹配的缓存结果，否则返回该请求结果并缓存。

## 参考资料

[ServiceWorker简介](https://developers.google.com/web/fundamentals/primers/service-workers?hl=zh-cn)

[使用 Service Workers](https://developer.mozilla.org/zh-CN/docs/Web/API/Service_Worker_API/Using_Service_Workers)

