---
title: NodeJS的底层原理
date: 2021-12-28 00:08:45
toc: true
mathjax: false
categories: 
- 前端
tags:
- Node
---

本文的内容主要分为两部分：

- NodeJS 的基础和架构
- NodeJS 核心模块的实现

## NodeJS 的基础和架构

### 一、NodeJS 的组成

Node.js 主要由 V8、Libuv 和第三方库组成。

<!-- more -->

#### 1、Libuv

Libuv 是 Node.js 底层的异步 IO 库，但它提供的功能不仅仅是 IO，还包括进程、线程、信号、定时器、进程间通信等，而且 Libuv 抹平了各个操作系统之间的差异。Libuv 提供的功能大概如下：

- Full-featured event loop backed by epoll, kqueue, IOCP, event ports.
- Asynchronous TCP and UDP sockets
- Asynchronous DNS resolution
- Asynchronous file and file system operations
- File system events
- ANSI escape code controlled TTY
- IPC with socket sharing, using Unix domain sockets or named pipes (Windows) 
- Child processes
- Thread pool 
- Signal handling
- High resolution clock 
- Threading and synchronization primitives

#### 2、V8

Node.js 是基于 V8 的 JS 运行时，它利用 V8 提供的能力，极大地拓展了 JS 的能力。这种拓展不是为 JS 增加了新的语言特性，而是拓展了功能模块，比如在前端，我们可以使用 Date 这个函数，但是我们不能使用 TCP 这个函数，因为 JS 中并没有内置这个函数。而在 Node.js 中，我们可以使用 TCP，这就是 Node.js 做的事情，让用户可以使用 JS 中本来不存在的功能，比如文件、网络。Node.js 中最核心的部分是 Libuv 和 V8，V8 不仅负责执行 JS，还支持自定义的拓展，实现了 JS 调用 C++ 和 C++ 调用 JS 的能力。比如我们可以写一个 C++ 模块，然后在 JS 调用，Node.js 正是利用了这个能力，完成了功能的拓展。JS 层调用的所有 C、C++ 模块都是通过 V8 来完成的。可以说得益于 V8 支持自定义扩展这才有了 Node.js

#### 3、第三方库

Node.js 中的第三方库主要包括异步 DNS 解析（ cares ）、HTTP 解析器（旧版使用 http_parser，新版使用 llhttp）、HTTP2 解析器（ nghttp2 ）、 解压压缩库( zlib )、加密解密库( openssl )等等。

### 二、NodeJS 代码架构

<img src="/assets/node-principle/01.png" width="600" />

上图是 Node.js 的代码架构，Node.js的代码主要分为 JS、C++、C 三种。

1. JS 是我们平时使用的那些模块(http/fs)。
2. C++ 代码分为三个部分，第一部分是封装了 Libuv 的功能，第二部分则是不依赖于 Libuv ( crypto 部分 API 使用了 Libuv 线程池)，比如 Buffer 模块，第三部分是 V8 的代码。
3. C 语言层的代码主要是封装了操作系统的功能，比如 TCP、UDP。

了解了 Node.js 的组成和代码架构后，我们看看 Node.js 启动的过程都做了什么。

### 三、NodeJS 启动过程

Node.js启动的主流程图如下所示：

<img src="/assets/node-principle/08.png" width="700" />

接下来我们详细分析每个过程：

#### 1、注册 C++ 模块

<img src="/assets/node-principle/02.png" width="600" />

首先 Node.js 会调用 RegisterBuiltinModules 函数注册 C++ 模块，这个函数会调用一系列 registerxxx 的函数，我们发现在 Node.js 源码里找不到这些函数，因为这些函数是在各个 C++ 模块中，通过宏
定义实现的，宏展开后就是上图黄色框的内容，每个 registerxxx 函数的作用就是往 C++ 模块的链表了插入一个节点，最后会形成一个链表。

这儿我们以 _register_tcp_wrap 函数为例详细分析一下在函数中具体做了什么，在这之前我们要先了解一下 Node.js 中表示 C++ 模块的数据结构：

```cpp
struct node_module {  
    int nm_version;  
    unsigned int nm_flags;  
    void* nm_dso_handle;  
    const char* nm_filename;  
    node::addon_register_func nm_register_func;  
    node::addon_context_register_func nm_context_register_func;  
    const char* nm_modname;  
    void* nm_priv;  
    struct node_module* nm_link;  
};  
```

_register_tcp_wrap 函数调了 node_module_register，并传入一个node_module数据结构，这里我们看一下 node_module_register 的实现：

```cpp
void node_module_register(void* m) {  
    struct node_module* mp = reinterpret_cast<struct node_module*>(m);  
    if (mp->nm_flags & NM_F_INTERNAL) {  
        mp->nm_link = modlist_internal;  
        modlist_internal = mp;  
    } else if (!node_is_initialized) { 
        mp->nm_flags = NM_F_LINKED;  
        mp->nm_link = modlist_linked;  
        modlist_linked = mp;  
    } else {  
        thread_local_modpending = mp;  
    }  
}  
```

C++ 内置模块的 flag 是 NM_F_INTERNAL，所以会执行第一个 if 的逻辑，modlist_internal 类似一个头指针。if 里的逻辑就是头插法建立一个单链表。

那么 Node.js 里是如何访问这些 C++ 模块的呢？在 Node.js 中，是通过 internalBinding 访问 C++ 模块的，internalBinding 的逻辑很简单，就是根据模块名从模块队列中找到对应模块。但是这个函数只能
在 Node.js 内部使用，不能在用户 JS 模块使用，用户可以通过 process.binding 访问 C++ 模块。

#### 2、创建 Environment 对象，并绑定到 Context

注册完 C++ 模块后就开始创建 Environment 对象，Environment 是 Node.js 执行时的环境对象，类似一个全局变量的作用，他记录了 Node.js 在运行时的一些公共数据。创建完 Environment 后，Node.js
会把该对象绑定到 V8 的 Context 中，为什么要这样做呢？主要是为了在 V8 的执行上下文里拿到 env 对象，因为 V8 中只有 Isolate、Context 这些对象，如果我们想在 V8 的执行环境中获取 Environment 
对象的内容，就可以通过 Context 获取 Environment 对象。

<img src="/assets/node-principle/03.png" width="400" />

<img src="/assets/node-principle/04.png" width="500" />

#### 3、初始化模块加载器

1. Node.js 首先传入 C++ 模块加载器，执行 loader.js，loader.js 主要是封装了 C++ 模块加载器和原生 JS 模块加载器，并保存到 env 对象中。
2. 接着传入 C++ 和原生 JS 模块加载器，执行 run_main_module.js。
3. 在 run_main_module.js 中传入普通 JS 和原生 JS 模块加载器，执行用户的 JS。

假设用户 JS 代码如下：

```js
require('./myModule')
require('net')
```

代码中分别加载了一个用户模块和原生 JS 模块，我们看看加载过程，执行 require 的时候。

1. Node.js 首先会判断是否是原生 JS 模块，如果不是则直接加载用户模块，否则，会使用原生模块加载器加载原生 JS 模块。
2. 加载原生 JS 模块的时候，如果用到了 C++ 模块，则使用 internalBinding 去加载。

<img src="/assets/node-principle/05.png"  />

#### 4、执行用户 JS 代码，然后进入 Libuv 事件循环

接着 Node.js 就会执行用户的 JS，通常用户的 JS 会给事件循环生产任务，然后就进入了事件循环系统，比如我们 listen 一个服务器的时候，就会在事件循环中新建一个 TCP handle。Node.js 就会在这个事件
循环中一直运行。

```js
net.createServer(() => {}).listen(80)
```

<img src="/assets/node-principle/06.png"  />

### 四、事件循环

下面我们看一下事件循环的实现。事件循环主要分为 7 个阶段，timer 阶段主要是处理定时器相关的任务，pending 阶段主要是处理 Poll IO 阶段回调里产生的回调，check、prepare、idle 阶段是自定义的阶段，这三个阶段的任务每次
事件序循环都会被执行，Poll IO 阶段主要是处理网络 IO、信号、线程池等等任务，closing 阶段主要是处理关闭的 handle，比如关闭服务器。

<img src="/assets/node-principle/07.png" width="500" />

1. timer 阶段: 用二叉堆实现，最快过期的在根节点。
2. pending 阶段：处理 Poll IO 阶段回调里产生的回调
3. check、prepare、idle 阶段：每次事件循环都会被执行。
4. Poll IO 阶段：处理文件描述符相关事件。
5. closing 阶段：执行调用 uv_close 函数时传入的回调。

下面我们详细看一下每个阶段的实现。

## 参考资料

[Node.js的底层原理](https://zhuanlan.zhihu.com/p/375276722)

[通过源码分析nodejs原理](https://github.com/theanarkh/understand-nodejs)

