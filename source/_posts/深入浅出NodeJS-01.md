---
title: 《深入浅出NodeJS》读书笔记（一）
date: 2021-07-17 16:53:05
toc: true
mathjax: false
categories: 
- 读书笔记
tags:
- Node
- 读书笔记
---

之前在微信读书零零散散看了一些章节，这次特意买了实体书认真全部看一遍，希望能用这篇博客去做一个记录和思考。

## 一、Node 简介

### 1、Node 的特点

- 异步IO: Node 在底层构建了很多异步 I/O 的 API，从文件的读取到网络请求等，这样的意义在于在 Node 中我们可以从语言层面很自然地进行并行 I/O 操作而无须等待之前的调用结束，在编程模型上可以极大的提升效率。
- 事件与回调函数: 事件的编程方式具有轻量级、松耦合、只关注事务点等优势，Node 将前端浏览器中应用广泛且成熟的事件引入后端，配合异步 I/O，将事件点暴露给业务逻辑，极大程度上方便了业务层的编码处理。但是在多个异步任务的场景下，事件与事件之间各自独立，如何协作是个主要的问题，Node 为此提供了回调函数，而且纵观下来回调函数也是最好的接受异步调用返回数据的方式。
- 单线程: Node 保持了 JavaScript 在浏览器中单线程的特点，而且在 Node 中 JavaScript 与其他线程是无法共享任何状态的，单线程最大的好处是不用在线线程间的状态同步问题，没有频繁切换线程上下文所带来的性能损耗，缺点在于无法利用多核 CPU、错误会引起整个应用退出、CPU 阻塞会导致无法继续调用异步 I/O。
- 跨平台: Node 借助 libuv 成功在操作系统与上层 Node 模块之间构建了一层平台层架构，因此借助 libuv 良好的架构设计 Node 实现了跨平台开发。

<!-- more -->
### 2、Node 的应用场景

- Node 本身面向网络的设计且擅长并行 I/O，因此能够有效地组织起更多的硬件资源，并且其基于事件循环强大的处理能力其非常适合 I/O 密集型的应用场景。
- 虽然由于 JavaScript 是单线程的原因并不能完美释放多核 CPU 的计算能力，但是由于 V8 实现的深度优化，同时借助 C/C++ 扩展模块或子进程的方式同样适用于处理 CPU 密集型场景
- 与遗留系统和平相处
- 分布式应用对于可伸缩性的要求非常高，借用 Node 并行 I/O 的能力可加速数据的获取流程。

## 二、模块机制

### 1、Node 的模块实现

在 Node 中引入模块需要经历如下三步：

- 路径分析
- 文件定位
- 编译执行

在 Node 中模块分为两类：一类是 Node 提供的模块，称为核心模块；另一类是用户编写的模块，称为文件模块。核心模块在 Node 源码编译的过程中编译进了二进制执行文件。在 Node 进程启动时，部分核心模块就被直接加载进内存中，因此核心模块引入时会忽略掉文件定位与编译执行的两步，而且路径分析的优先级也是最高的，因此其加载速度要比文件模块快很多。文件模块则是在运行时动态加载，需要完整的路径分析、文件定位、编译执行过程。


详细的模块加载过程如下：

#### (1)**优先从缓存加载**

需要注意以下两点：

- 浏览器仅仅缓存文件而 Node 缓存的是编译和执行后的对象；
- 核心模块的缓存检查会优于文件模块。

#### (2)**路径分析**

- 核心模块：优先级仅次于缓存加载，加载过程最快
- 路径形式的文件模块：require 方法会将路径转换为真实路径，并以真实路径为索引将编译结果存放到缓存中，由于指明了确切的文件位置，因此加载速度仅次于核心模块
- 自定义模块：指的是非核心模块也不是路径形式的标识符，是一种特殊的文件模块，由于其在加载过程中 Node 会逐个尝试**模块路径**中的路径，直到找到目标文件为止，因此其加载速度也是最慢的

> 注：模块路径是 Node 在定位文件模块的具体文件时指定的查找策略，具体表现为一个路径组成的数组，在代码中我们可以通过 module.paths 属性查看。

#### (3)**文件定位**

- 文件扩展名分析：CommonJS 模块规范允许标识符中不包含文件扩展名的情况，这时 Node 会按照 .js、.json、.node 的次序补足扩展名并调用 fs 模块同步阻塞式地判断文件是否存在。
- 目录分析和包：Node 会首先在当前目录下查找 package.json 文件并通过 JSON.parse 方法解析，获取到包描述对象后会根据 main 属性指定的文件名进行定位。

#### (4)**模块编译**

在定位到文件后首先是文件的读取，对于不同的文件扩展名其载入方式也有很大的不同：

- .js 文件：通过 fs 模块同步读取文件后编译执行
- .node 文件：C/C++ 编写的扩展文件，通过 dlopen 方法加载最后编译生成的文件
- .json 文件：通过 fs 模块同步读取文件后，用 JSON.parse 解析返回的结果
- 其余扩展名文件：都被当作 .js 文件载入

> 注：通过 require.extensions 属性可知道系统中已有的扩展加载方式，我们也可通过类似 `require.extensions['.ext'] = function(module, filename) { ... }` 的方式来实现加载，但是官方更支持先将其他语言编译成 js 文件后再加载，这样可避免将繁琐的编译加载等过程引入 Node 的执行过程中。

在读取到文件内容后紧接着就是根据不同的文件扩展名进行对应模块的编译：

**JavaScript 文件**：

Node 会首先对获取到的 JavaScript 文件内容进行头尾包装，一个正常的 JavaScript 文件会被包装成如下的样子：

```js
(function (exports, require, module, __filename, __dirname) {
    ...
});
```

这样就对每个模块文件实现了作用域隔离，包装之后的代码会通过 vm 原生模块的 runInThisContext 方法执行，返回一个具体的 function 对象，然后将当前模块对象的 exports 属性、require 方法、module（模块对象自身）以及在文件定位中得到的完整文件路径和文件目录作为参数传递给这个 function 执行，执行后模块的 exports 属性被返回给调用方，这样 exports 属性上的任何属性及方法都可以被外部调用到，但是模块中的其余变量则不可直接被调用。

> 注：在 CommonJS 中每个文件都是一个模块对象，其具体定义如下：
```js
function Module(id, parent) {
    this.id = id;
    this.exports = {};
    this.parent = parent;
    if (parent && parent.children) {
        parent.children.push(this)
    }
    this.filename = null;
    this.loaded = false;
    this.children = [];
}
```

由于函数入参中传入的 exports 属性本质上是模块对象 module 的 exports 属性的引用，因此如果文件中对 exports 直接赋值会改变 exports 属性的引用，从而导致模块暴露异常。

**Node 文件**：

.node 文件是在编写了 C/C++ 模块之后编译产生的，所以这里 Node 主要是调用 process.dlopen 方法来加载和执行，在 Node 的架构下 dlopen 方法在 Windows 和 *nix 平台下分别有不同的实现，通过 libuv 层进行了封装。

**JSON 文件**：

JSON 文件的编译在 Node 中是最简单的，Node 利用 fs 模块同步读取 JSON 文件的内容之后再调用 JSON.parse 方法得到对象，然后将它赋给模块对象的 exports 以供外部调用。

### 2、核心模块

Node 的核心模块可分为 C/C++ 编写的和 JavaScript 编写的两部分：

#### (1)JavaScript核心模块的编译过程

在编译所有的 C/C++ 文件之前，编译程序需要将所有的 JavaScript 模块文件编译为 C/C++ 代码，因此 JavaScript 的完整编译过程如下：

**转存为 C/C++ 代码**

Node 采用了 V8 附带的 js2c.py 工具，将所有内置的 JavaScript 代码转换为 C++ 里的数组并生成 node_natives.h 头文件，这个过程中 JavaScript 以**字符串**的形式存储在 node 的命名空间中，是不可以直接执行的。在启动 Node 进程时，JavaScript 代码直接加载进内存中。在加载的过程中，JavaScript 核心模块经过标识符分析后直接定位到内存中，这比普通的文件模块从磁盘中读取要快得多。

**编译 JavaScript 核心模块**

在引入 JavaScript 核心模块的过程中也同样会经历头尾包装过程，然后再执行并导出 exports 对象，与文件模块有区别的地方主要在于：**获取源代码的方式**和**缓存执行结果的位置**：

- JavaScript 核心模块的源文件主要通过 process.bingding('natives') 从内存的 node 命名空间中取出，而 JavaScript 文件模块主要是通过 fs 模块在磁盘上读取。
- 编译成功的 JavaScript 核心模块会缓存到 NativeModule._cache 对象上，文件模块则会缓存到 Module._cache 对象上。

```js
function NativeModule(id) {
    this.filename = id + '.js';
    this.id = id;
    this.exports = {};
    this.loaded = false;
}
NativeModule._source = process.binding('natives');
NativeModule._cache = {}
```

#### (2)C/C++核心模块的编译过程

在核心模块中有些模块全部由 C/C++ 编写，有些模块则由 C/C++ 完成核心部分，其他部分则由 JavaScript 实现包装或向外导出，以满足性能的要求。Node 中这种复合模式使得其在开发速度和性能之间找到平衡点。

通常将那些由 C/C++ 编写的部分统称为**内建模块**，它们通常不被用户直接调用，Node 中的 buffer、cypto、fs 等模块都是部分由 C/C++ 编写的复合模块。

**内建模块的组织形式**

在 Node 中内建模块的内部结构定义如下：

```cpp
struct node_module_struct {
    int version;
    void *dso_handle;
    const char *filename;
    void (*register_func) (v8::Handle<v8::Object> target);
    const char *modname;
}
```

每一个内建模块在定义之后都通过 NODE_MODULE 宏将模块定义到 node 命名空间中，模块的具体初始化方法挂在为结构的 register_func 成员：

```cpp
#define NODE_MODULE(modname, regfunc) {
    extern "C" {
        NODE_MODULE_EXPORT node::node_module_struct modname ## _module = {
            NODE_STANDARD_MODULE_STUFF,
            regfunc,
            NODE_STRINGIFY(modname)
        }
    }
}
```

node_extensions.h 文件将这些散列的内建模块统一放进了一个叫 node_module_list 的数组中，这些模块有 node_buffer、node_crypto、node_evals...，这些内建模块的取出也非常简单，Node 提供了 get_builin_module() 方法从 node_module_list 数组中取出这些模块。

内建模块的优势在于：
- 它们本身由 C/C++ 编写，性能上优于脚本语言
- 在进行文件编译时，它们被编译进二进制文件，一旦 Node 开始执行，它们被直接加载进内存中，无需再次做标识符定位、文件定位、编译等过程，直接就可执行。

**内建模块的导出**

在 Node 的所有模块类型中存在着依赖层级关系：文件模可能会依赖核心模块、核心模块可能会依赖内建模块。通常不推荐文件模块直接调用内建模块而是直接调用核心模块即可，那么内建模块是如何将内部变量或方法导出以供外部 JavaScript 核心模块调用呢？

Node 在启动时会生成一个全局变量 process，并提供 Binding() 方法来协助加载内建模块，方法内会首先判断 binding_cache 缓存对象中是否存在，如果存在直接读取，如果不存在则先创建一个 exports 空对象，然后调用 get_builtin_module() 方法取出内建模块对象，通过执行 register_func() 填充 exports 对象，最后将 exports 对象按模块名缓存，并返回给调用方完成导出。

前文提到的 JavaScript 核心文件被转换为 C/C++ 数组存储后便也是通过 process.binding('natives') 取出放置在 NativeModule._source 中的，此方法将通过 js2c.py 工具转换出的字符串数组取出然后重新转换为普通字符串，以对 JavaScript 核心模块进行编译和执行。

#### (3)核心模块的引入流程

虽然对于用户而言 require 方法十分简洁但是核心模块的引入往往要经历 C/C++ 层面的内建模块的定义、(JavaScript)核心模块的定义和引入及(JavaScript)文件模块层面的引入，我们以 os 原生模块的引入为例：

<img src="/assets/深入浅出NodeJS/01.png" width="400">

#### (4)编写核心模块

核心模块中 JavaScript 部分的开发与文件模块几乎相同，遵循 CommonJS 规范，编写核心模块的主要难点在于内建模块：

编写内建模块的主要分两步：编写头文件和 C/C++ 文件。

```cpp
// node_hello.h
#ifndef NODE_HELLO_H_
#define NODE_HELLO_H_
#include <v8.h>

namespace node {
    // 预定义方法
    v8::Handle<v8::value> SayHello(const v8::Arguments& args)
}
#endif
```

```cpp
// node_hello.cc
#include <node.h>
#include <node_hello.h>
#include <v8.h>

namespace node {
    using namespace v8;
    // 实现预定义的方法
    Handle<Value> SayHello(const Arguments& args) {
        HandleScope scope;
        return scope.Close(String::New("Hello world!"));
    }

    // 给传入的目标对象添加sayHello方法
    void Init_Hello(Handle<Object> target) {
        target->Set(String::NewSymbol("sayHello"), FunctionTemplate::New(SayHello)->GetFunction());
    }
}
// 调用NODE_MODULE()将注册方法定义到内存中
NODE_MODULE(node_hello, node::Init_Hello)
```

编写完成以后还需要让 Node 认为它是内建模块，因此还需要更改 src/node_extensions.h，在 NODE_EXT_LIST_END 前添加 NODE_EXT_LIST_ITEM（如 node_hello）,以将 node_hello 模块添加进 node_module_list 数组中。其次还需要让编写的代码文件编译进执行文件，同时需要更改 Node 项目生成文件 node.gyp，并在 'target_name': 'node' 节点的 sources 中添加上新编写的两个文件，然后编译整个 Node 项目。

### 3、C/C++ 扩展模块

C/C++ 扩展模块属于文件模块中的一类，在编译时会首先被预编译为 .node 文件，然后调用 process.dlopen() 方法加载执行。不过在详细展开讲解之前我们先需要了解到 Node 的原生模块一定程度上是可以跨平台的，其前提条件是源代码可以支持在 *nix 和 Windows 上编译，其中 *.nix 下通过 g++/gcc 等编译器编译为动态链接共享对象文件(.so)，在 Windows 下则需要通过 Visual C++ 的编译器编译为动态链接库文件(.dll)，因此 .node 文件在 Windows 下实际上是一个 .dll 文件，在 *.nix 下是一个 .so 文件，dlopen 方法在内部实现时再根据平台进行区分，分别采用加载 .so 和 .dll 的方式。具体过程如下图所示：

<img src="/assets/深入浅出NodeJS/02.png" width="700">

#### (1)C/C++扩展模块的编写

普通的扩展模块与内建模块的区别在于无需将源代码编译进 Nod，而是通过 dlopen 方法动态加载，所以在编写普通扩展模块时无需将源代码写进 node 命名空间，也不需要提供头文件，其写法与内建模块一样，都是将方法挂载在 target 对象上，然后通过 NODE_MODULE 声明即可。

#### (2)C/C++扩展模块的编译

从 0.8 版本以后 Node 官方采用 GYP 工具生成各个平台下的项目文件，极大程度的减小了开发者的工作量，Node 官方也基于 GYP 工具专门推出了专有的扩展构建工具 node-gyp，因此我们编译 C/C++ 扩展模块只需要编写 .gyp 项目文件 binding.gyp 并在终端分别执行命令 `node-gyp configure` 和 `node-gyp build` 即可。编译过程会根据平台不同分别编译处理并生成对应的 .node 文件。

#### (3)C/C++扩展模块的加载

在编译得到 .node 文件后我们在代码中只需要通过 require() 方法调用即可，不过这个具体的调用过程却远比这个方法复杂，具体如下图所示：

<img src="/assets/深入浅出NodeJS/03.png" width="500">

如上图所示加载 .node 文件实际上经历了两个步骤，第一个步骤是调用 uv_dlopen() 方法去打开动态链接库，第二个步骤是调用 uv_dlsym() 方法找到动态链接库中通过 NODE_MODULE 宏定义的方法地址，这两个过程都是通过 libuv 库进行封装的：在 *nix 平台下实际上调用的是 dlfcn.h 头文件中定义的 dlopen() 和 dlsym() 两个方法，在 Windows 平台下则是通过 LoadLibraryExW()和 GetProcAddress() 这两个方法实现的，他们分别加载 .so 和 .dll 文件。

由于编写模块时通过 NODE_MODULE 将模块定义为 node_module_struct 结构，所以在获取函数地址之后，将它映射为 node_module_struct 结构几乎是无缝对接的，接下来就是将传入的 exports 对象作为入参传入，将 C++ 中定义的方法挂载在 exports 对象上，然后调用者就可以轻松调用了。

## 三、异步 I/O

### 1、异步 I/O 与非阻塞 I/O

从实际效果而言异步和非阻塞都达到了我们并行 I/O 的目的，但是从计算机内核 I/O 而言，异步/同步和阻塞/非阻塞实际上是两回事。操作系统内核对于 I/O 只有两种方式：阻塞与非阻塞。在调用阻塞 I/O 时，应用程序需要等待 I/O 完成才返回结果，而非阻塞 I/O 调用之后会立即返回。

> 注：操作系统对计算机进行了抽象，将所有输入输出设备抽象为文件。内核在进行文件 I/O 操作时，通过文件描述符进行管理，而文件描述符类似于应用程序与系统内核之间的凭证。应用程序如果需要进行 I/O 调用，需要先打开文件描述符，然后根据文件描述符去实现文件的数据读写。此处阻塞 I/O 与非阻塞 I/O 的区别在于阻塞 I/O 完成了整个获取数据的过程，而非阻塞 I/O 则不带数据直接返回，要获取数据，还需要通过文件描述符再次获取。

非阻塞 I/O 返回之后 CPU 的时间片可以用来处理其他事务，对于性能的提升是很明显的，但是非阻塞 I/O 依然存在一个问题：由于完整的 I/O 没有完成，立即返回的并不是业务层期望的数据，而仅是当前调用的状态，我们期望的异步 I/O 应该是应用程序发起阻塞调用或非阻塞调用，无需通过遍历或时间唤醒等方式轮询，可以直接处理下一个任务，只需要在 I/O 完成以后通过信号或回调将数据传递给应用程序即可，为了达到这样的目标 Node 主要借助了多线程：

让部分线程通过非阻塞 I/O 和轮询技术来完成数据读取，让另一个线程进行计算处理，通过进程之间的通信将 I/O 得到的数据进行传递，这样就轻松实现了异步 I/O。

具体异步 I/O 模型 Node 在 libuv 针对不同平台进行了不同的封装，Node 最初在 *nix 平台下采用了 libeio 配合 libev 实现了异步 I/O，在 Node v0.9.3 中自行实现了线程池来实现异步 I/O。而在 Windows 平台下直接采用了 IOCP 异步 I/O 模型实现，具体如图所示：

<img src="/assets/深入浅出NodeJS/04.png" width="500">

### 2、Node 的异步 I/O

Node 是如何实现异步 I/O 的？核心就是 Node 的自身执行模型——事件循环，事件循环是一个典型的**生产者/消费者模型**，异步 I/O、网络请求等则是事件的生产者，源源不断的为 Node 提供不同类型的事件，这些事件被传递到对应的观察者那里，事件循环则是从观察者那里不断取出事件并处理。

在 windows 下这个循环基于 IOCP 创建，而在 *nix 下则基于多线程创建。

#### (1)请求对象

对于一般的（非异步）回调函数，函数由我们自行调用，对于 Node 中的异步 I/O 调用而言，回调函数却不由开发者来调用。那么从我们发出调用后到回调函数被执行，中间发生了什么呢？

事实上，从 Javascript 发起调用到内核执行完 I/O 操作的过渡过程中，存在一种中间产物，它叫做**请求对象**。请求对象是异步 I/O 过程中的重要中间产物，所有的状态都保存在这个对象中，包括送入线程池等待执行以及 I/O 操作完毕后的回调处理。

我们以 fs.open 方法为例：

<img src="/assets/深入浅出NodeJS/05.png" width="600">

从上图我们可以看到虽然 libuv 有两个平台的实现，但是实质上还是调用了 uv_fs_open() 方法。在 uv_fs_open() 的调用过程中，我们创建了一个 FSReqWrap 请求对象，从 JavaScript 层传入的参数和当前方法都被封装在这个请求对象中，其中我们最为关注的回调函数则被设置在这个对象的 oncomplete_sym 属性上：

req_wrap -> object -> Set(oncomplete_sym, callback)

对象包装完毕后，在 Windows 下则调用 QueueUserWorkItem() 方法将这个 FSReqWrap 对象推入线程池中等待执行。

至此，JavaScript 调用立即返回，由 JavaScript 层面发起的异步调用的第一阶段就此结束。JavaScript 线程可以继续执行当前任务的后续操作。当前的 I/O 操作在线程池中等待执行，不管是否阻塞 I/O，都不会影响到 JavaScript 线程的后续执行，如此就达到了异步的目的。

#### (2)执行回调

组装好请求对象、送入 I/O 线程池等待执行，实际上只完成了异步 I/O 的第一部分，回调通知是第二部分。线程池中 I/O 操作调用完毕之后，会将获取的结果储存在 req -> result 属性上，然后调用 PostQueuedCompletionStatus() 通知 IOCP，告知对象操作已经完成。

PostQueuedCompletionStatus() 方法的作用是向 IOCP 提交执行状态，并将线程回归线程池，通过 PostQueuedCompletionStatus() 方法提交的状态可通过 GetQueuedCompletionStatus() 提取。

在这个过程中其实还动用了事件循环的 I/O 观察者，在每次 Tick 的执行中，它会调用 IOCP 相关的 GetQueuedCompletionStatus() 方法检查线程池中是否有执行完的请求，如果存在，会将请求对象加入到 I/O 观察者的队列中，然后将其当作事件处理。

I/O 观察者回调函数的行为就是取出请求对象的 result 属性做为参数，取出 oncomplete_sym 属性做为方法，然后调用执行，以此达到调用 JavaScript 中传入的回调函数的目的。

至此整个异步 I/O 流程完全结束，流程如图所示：

<img src="/assets/深入浅出NodeJS/06.png" width="700">

### 3、非 I/O 的异步API

#### (1)定时器

setTimeout() 和 setInterval() 分别用于单次和多次定时执行任务，它们的实现原理与异步 I/O 比较类似，只是不需要异步 I/O 线程池的参与。调用 setTimeout() 或者 setInterval() 创建的定时器会被插入到定时器观察者内部的一个红黑树中。每次 Tick 执行时，会从该红黑树中迭代取出定时器对象，检查是否超过定时时间，如果超过就会形成一个事件，它的回调函数将立即执行。

setTimeout() 的具体执行流程如下图所示，stInterval() 与之相同，区别是后者是重复性的检测与执行。

<img src="/assets/深入浅出NodeJS/07.png" width="700">

#### (2)process.nextTick()

在了解 process.nextTick() 之前我们为了立即异步执行一个任务往往要通过如下代码实现：

```js
setTimeout(function() {
    // TODO
}, 0)
```

但是由于时间循环自身的原因定时器的精度不够，而且采用定时器需要动用红黑树，创建定时器和迭代等操作，较为浪费性能，而 process.nextTick() 相对要较为轻量，每次调用 process.nextTick() 只会将回调函数放入队列中，在下一轮 Tick 时取出执行。定时器中采用红黑树的操作时间复杂度为 O(lg(n))，nextTick() 的时间复杂度为 O(1)。

#### (3)setImmediate

setImmediate() 与 process.nextTick() 类似，都是将回调函数延迟执行，但是 process.nextTick() 中的回调函数的优先级要高于 setmmediate()，这是因为 process.nextTick() 属于 idle 观察者，setImmediate() 属于 check 观察者。在每一轮循环检查中，idle 检查者先于 I/O 观察者，I/O 观察者先于 check 观察者。

而且在具体的实现中，process.nextTick() 的回调函数保存在一个数组中，而 setImmediate() 则是保存在链表中，在行为上，process.nextTick() 在每轮循环中会将数组中的回调函数全部执行完，而 setImmediate() 在每轮循环中执行链表中的一个回调函数。具体如下示例所示：

```js
process.nextTick(function () {
    console.log('nextTick延迟执行1')
})
process.nextTick(function () {
    console.log('nextTick延迟执行2')
})

setImmediate(function () {
    console.log('setImmediate延迟执行1')
    process.nextTick(function () {
        console.log('强势插入')
    })
})
setImmediate(function () {
    console.log('setImmediate延迟执行2')
})
console.log('正常执行')
```

其执行结果如下：

```
正常执行
nextTick延迟执行1
nextTick延迟执行2
setImmediate延迟执行1
强势插入
setImmediate延迟执行2
```

这样设计可以保证每轮循环可以较快的执行结束，防止 CPU 占用过多而阻塞后续 I/O 调用的情况。

## 四、异步编程

### 1、异步编程解决方案

利用事件循环的方式 Node 实现了异步模型，使得异步编程首次大规模出现在业务层面，目前异步编程的主要解决方案有如下三种：

- 事件发布/订阅模式
- Promise/Defered 模式
- 流程控制库

#### (1)事件发布/订阅模式

事件监听器模式是一种广泛用于异步编程的模式，是回调函数的事件化，又称发布/订阅模式。

Node 自身提供的 events 模块是发布/订阅的一个简单实现，发布/订阅模式可以实现一个事件与多个回调函数的关联，这些回调函数又被称为事件侦听器。通过 emit() 发布事件后消息会立即传递给当前事件的所有侦听器执行。侦听器可以很灵活的添加删除，使得事件和具体的处理逻辑之间可以很轻松地关联和解耦。

值得注意的是 Node 对事件发布/订阅的机制做了一些额外的处理，这大多是基于健壮性考虑的：

- 如果对一个事件添加超过 10 个侦听器将会得到一个警告，这是因为设计者任务侦听器太多容易造成内存泄漏，通过调用 emitter.setMaxListeners(0) 可以取消这个限制。另一方面，由于事件发布会引起一系列侦听器执行，如果事件相关的侦听器过多，可能存在过多占用 CPU 的场景。
- 为了处理异常，EventEmitter 对象对 error 事件进行了特殊对待。如果运行期间错误触发了 error 事件，EventEmitter 会检查是否有对 error 事件添加过侦听器。如果添加了，这个错误会交由侦听器处理，否则这个错误会做为异常抛出。如果外部没有捕获这个异常将会引起线程退出。

Node 中对于事件发布/订阅模式的使用主要有如下几种：

**1.继承events模块**

实现一个继承 EventEmitter 的类是十分简单的，Node 在 util 模块封装了继承的方法可以很便利的使用：

```js
var events = require('events')
const util = require('util')

function Stream() {
    events.EventEmitter.call(this)
}
util.inherits(Stream, events.EventEmitter)

const stream = new Stream()
stream.on('hi', () => {
    console.log('hi')
})
stream.emit('hi')
```

**2.利用事件队列解决雪崩问题**

在事件发布/订阅模式中，通常也有一个 once() 方法，通过它添加的侦听器只能执行一次，在执行之后就会将它与事件的关联移除，这个特性可以帮助我们过滤一些重复性的事件响应，比如雪崩问题：

```js
var proxy = new events.EventEmitter();
var status = "ready"
var select = function (callback) {
    proxy.once("selected", callback);
    if (status === "ready") {
        status = "pending";
        db.select("SQL", function (results) {
            proxy.emit("selected", results);
            status = "ready"
        })
    }
}
```

**3.多异步之间的协作方案**

一般而言事件发布/订阅模式中事件与侦听器的关系是一对多，但在异步编程中也会出现事件与侦听器的关系是多对一的情况，由于多个异步场景中回调函数的执行并不能保证顺序，且回调函数之间互相没有任何交集，所以需要借助一个第三方函数和第三方变量来处理异步协作的结果，通常我们把这个用于检测次数的变量叫做**哨兵变量**。

```js
var count = 0;
var results = {};
var done = function (key, value) {
    results[key] = value;
    count++;
    if (count === 3) {
        render(results);
    }
}

fs.readFile(template_path, 'utf-8', function(err, template) {
    done("template", template)
})
db.query(sql, function (err, data) {
    done("data", data)
})
l10n.get(function (err, resource) {
    done("resource", resource)
})
```

除了这种方式之外我们还可以通过 [EventProxy](https://github.com/JacksonTian/eventproxy) 中的 all() 方法来实现：

```js
var proxy = new EventProxy();

proxy.all("template", "data", "resources", function (template, data, resource) {
    // TODO
});
fs.readFile(template_path, "utf-8", function (err, template) {
    proxy.emit("template", template)
});
db.query(sql, function (err, data) {
    proxy.emit("data", data)
})
l10n.get(function (err, resources) {
    proxy.emit("resources", resources)
})
```


#### (2)Promise/Deferred模式

**1.Promise/A**

Promise/Deferred 模式其实包含两部分，即 Promise 和 Defered，Deferred 主要是用于内部，用于维护异步模型的状态，Promise 则用于外部，通过 then() 方法暴露给外部以添加自定义逻辑，整体关系如图所示：

<img src="/assets/深入浅出NodeJS/08.png" width="700">

我们根据规范可以实现一个简易版本：

```js
// Promise
const { EventEmitter } = require("events")
const util = require("util")

var Promise = function () {
    EventEmitter.call(this);
}
util.inherits(Promise, EventEmitter)

Promise.prototype.then = function (fulfilledHandler, errorHandler, progresssHandler) {
    if (typeof fulfilledHandler === 'function') {
        this.once('success', fulfilledHandler)
    }
    if (typeof errorHandler === 'function') {
        this.once('error', errorHandler)
    }
    if (typeof progresssHandler === 'function') {
        this.once('progress', progresssHandler)
    }
    return this
}

// Deferred
var Deferred = function () {
    this.state = "unfilled";
    this.promise = new Promise()
}
Deferred.prototype.resolve = function (obj) {
    this.state = 'fulfilled';
    this.promise.emit('success', obj);
}
Deferred.prototype.reject = function (obj) {
    this.state = 'failed';
    this.promise.emit('error', obj);
}
Deferred.prototype.progress = function (obj) {
    this.state = 'progress';
    this.promise.emit('progress', obj);
}
```

通过 Promise/Deferred 模式我们可以很轻松的完成对响应对象的封装：

```js
var promisify = function (res) {
    var deferred = new Defferred();
    var result = '';
    res.on('data', function (chunk) {
        result += chunk;
        deferred.progress(chunk);
    })
    res.on('end', function () {
        deferred.resolve(result)
    })
    res.on('error', function (err) {
        deferred.reject(err)
    })
    return deferred.promise;
}
```

需要注意这里返回 deferred.promise 的目的是为了不让外部程序调用 resolve() 和 reject() 方法，更改状态的行为交由定义者控制。这样我们就可以通过下述方法实现对响应对象的调用：

```js
promisify(res).then(function (result) {
    // TODO
}, function (error) {
    // TODO
}, function (chunk) {
    // TODO
})
```

**2.Promise中的多异步协作**

在 promise 的介绍中说过主要解决的是单个异步中存在的问题，那么当我们需要处理多个异步调用时又该如何处理呢？

```js
Deferred.prototype.all = function (promises) {
    var count = promises.length;
    var that = this;
    var results = [];
    promises.forEach(function (promise, i) {
        promise.then(function (data) {
            count--;
            results[i] = data;
            if (count === 0) {
                that.resolve(result);
            }
        }, function (err) {
            that.reject(err)
        })
    })
    return this.promise
}
```

这里通过 all() 方法抽象多个异步操作，只有所有的异步操作成功这个异步操作才算成功。

#### (3)流程控制库

**1.尾触发与Next**

除了事件和 Promise 外还有一类方法是需要手工调用才能继续执行后续调用的，我们将此类方法叫做尾触发，常见的关键词是next。尾触发目前应用得最多的地方是中间件，尾触发十分适合处理网络请求的场景，将复杂的逻辑拆解成简洁、单一的处理单元，逐层次的处理请求对象和响应对象。

但是需要注意的是虽然中间件这种尾触发模式并不要求每个中间件方法都是异步的，但是如果每个步骤都采用异步完成，实际上只是串行化的处理，没办法通过并行的异步调用来提升业务的处理效率。流式处理可以将一些串行的逻辑扁平化，但是并行逻辑处理还是需要搭配事件或者 Promise 完成的。

**2.async**

[async](https://www.npmjs.com/package/async)是最知名的流程控制模块，下面是它的几种典型用法：

- 异步的串行执行：`async.series([func1, func2, ...], callback)`
- 异步的并行执行：`async.parallel([func1, func2, ...], callback)`
- 异步调用的依赖处理：`async.waterfall([func1, func2, ...], callback)`
- 自动依赖处理：`async.auto(depsConfig)`

除此之外还有第三方库 [step](https://www.npmjs.com/package/step) 和 [wind](https://www.npmjs.com/package/wind)，此处不再详细介绍，可以查看官方文档资料。

### 2、异步并发控制

在 Node 中我们可以很轻易的利用异步发起并行调用，但是如果并发量过大我们的下层服务器会吃不消，如果是对文件系统进行大量并发调用，操作系统的文件描述符将会被瞬间用光。因此我们需要对异步并发做一定的控制，下面介绍两种解决方案：

#### (1)bagpipe的解决方案

- 通过一个队列来控制并发量
- 如果当前活跃（指调用发起但未执行回调）的异步调用量小于限定值，从队列中取出执行
- 如果活跃度调用达到限定值，调用暂时存放在队列中。
- 每个异步调用结束时，从队列中取出新的异步调用执行

#### (2)async的解决方案

async 提供了 parallelLimit() 方法来处理异步调用的限制，举个例子：

```js
async.parallelLimit([
    function (callback) {
        fs.readFile('file1.text', 'utf-8', callback);
    },
    function (callback) {
        fs.readFile('file2.text', 'utf-8', callback);
    },
], 1, function (err, results) {
    // TODO
})
```

parallelLimit() 方法提供了一个用于限制并发数量的参数，使得任务只能同时并发一定数量，而不是无限制并发。

## 五、内存控制

### 1、V8的垃圾回收机制与内存限制

#### (1)V8的内存限制

在一般的后端开发语言中在基本的内存使用上没有什么限制，然而在 Node 中通过 JavaScript 使用内存时就会发现只能使用部分内存（64位系统下约为1.4GB，32位系统下约为0.7GB），在这样的限制下 Node 无法直接操作大内存对象，而这个限制的根本原因在于 V8 的内存管理机制。

在 V8 中所有的 JavaScript 对象都是通过堆来进行分配的，当我们的代码中声明变量并赋值所使用的对象时，所使用对象的内存就分配在堆中。如果已申请的堆空闲内存不够分配新的对象，将继续申请堆内存，直到堆的大小超过 V8 的限制为止，V8 限制堆的大小的主要原因是 V8 的垃圾回收机制的限制，按官方的说法，以 1.5GB 的垃圾回收堆内存为例，V8 做一次小的垃圾回收需要 50 毫秒以上，做一次非增量的垃圾回收甚至要一秒以上，这样的时间花销是无法接受的。

V8 也提供了选项让我们使用更多内存，Node 在启动时可以传递 --max-old-space-size 或 --max-new-space-size 来调整内存限制的大小，示例如下：

```shell
node --max-old-space-size=1700 test.js // 单位是MB
node --max-new-space-size=1024 test.js // 单位是KB
```

#### (2)V8的垃圾回收机制

V8 的垃圾回收策略主要是基于分代式垃圾回收机制。

**V8的内存分代**

在 V8 中主要讲内存分为新生代与老生代两代，新生代中的对象为存活时间较短的对象，老生代中的对象为存活时间较长或常驻内存的对象，具体如下图所示：

<img src="/assets/深入浅出NodeJS/09.png" width="600">

v8 堆的整体大小就是新生代所占用的内存空间加上老生代的内存空间，前面提到的 --max-old-space-size 和 --max-new-space-size 也分别是用来设置老生代和新生代内存空间的大小，不过比较遗憾的是这两个值需要在启动时指定。这意味着 V8 使用的内存没有办法根据使用情况自动扩容，当内存内存分配过程中超过极限值时就会引起进程出错。

**Scavenge算法**

在分代的基础上，新生代中的对象主要通过 Scavenge 算法进行垃圾回收，在 Scavenage 的具体实现中主要采用了 Cheney 算法，Cheney 是一种采用复制的方式实现的垃圾回收算法，它将堆内存一分为二，每一部分空间都称为 semispace。在这两个 semispace 空间中只有一个处于使用中，另一个处于闲置状态。处于使用状态的 semispace 空间称为 From 空间，处于闲置状态的 semispace 被称为 To 空间。当我们分配对象时，先是在 From 空间中进行分配。当开始进行垃圾回收时会检查 From 空间中的存活对象并将其复制到 To 空间中，然后将 From 空间中非存活对象占用的空间进行释放，完成复制后，From 空间和 To 空间的角色发生对换。简而言之，垃圾回收的过程，就是通过将存活对象在两个 semispace 空间之间来回复制的过程。

Scavenge 的缺点是只能使用堆内存的一半，但是由于只复制存活的对象，并且对于生命周期短的场景存活对象只占少部分，所以它在时间效率上有优异的表现，因此 Scavenge 算法是典型的牺牲空间换取时间的算法。其不适合大规模的应用到所有的垃圾回收中，只适用于新生代对象周期较短的场景。

在单纯的 Scavenge 过程中，From 空间中的存活对象会被复制到 To 空间中去，然后对 From 空间和 To 空间进行角色互换（又称翻转）。但在分代式垃圾回收的前提下 From 空间中的存活对象在复制到 To 空间之前需要进行检查。在一定条件下需要将存活周期长的对象移动到老生代中，也就是完成对象晋升。

对象晋升的条件有两个，一个是对象是否经历过 Scavenge 回收，一个是 To 空间的内存占比超过限制。

<img src="/assets/深入浅出NodeJS/10.png" width="600">

设置 25% 这个限制值的原因是当这次 Scavenge 回收完成后这个 To 空间将变成 From 空间，接下来的内存分配将在这个空间中进行。如果占比过高，会影响后续的内存分配。对象晋升后将在老生代空间中作为存活周期较长的对象来对待，接受新的回收算法的处理。

**Mark-Sweep & Mark-Compact**

对于老生代中的对象，由于存活对象占较大比重，再采用 Scavenge 的方式会存在两个问题：一个是存活对象较多，复制存活对象的效率将会很低，另一个问题依然是浪费一半空间，因此老生代中主要采用了 Mark-Sweep 和 Mark-Compact 相结合的方式来进行垃圾回收。

Mark-Sweep 是标记清除的意思，它分为标记和清除两个阶段。在标记阶段 Mark-Sweep 会遍历堆中的所有对象，并标记活着的对象，而在随后的清除阶段只清除没有被标记的对象。Mark-Sweep 的最大问题在于进行了一次标记清除回收后内存空间会出现不连续的状态。这种内存碎片会对后续的内存分配造成问题，容易出现要存储大对象但是所有碎片空间都大小不足的情景，因此又引入了 Mark-Compact 算法。

Mark-Compact 是标记整理的意思，是在 Mark-Sweep 的基础上演变而来的。它们的差别在于对象被标记为死亡后，在整理的过程中，将活的对象往一侧移动，移动完成后直接清理掉边界外的内存。

3 种算法的简单对比如下：

| 回收算法               |  Mark-Sweep   |  Mark-Compact | Scavenge |
| --------------------- | :------: | :-----------: | :-----------: |
| 速度  |  中等  |  最慢   |   最快   |
| 空间开销  |  少（有碎片）  |  少（无碎片）   |   双倍空间（无碎片）   |
| 是否移动对象  |  否  |  是   |   是   |

从图中可以看出在 Mark-Sweep 和 Mark-Compact 之间由于 Mark-Compact 需要移动对象，所以它的执行速度不可能很快，所以在取舍上，V8 主要使用 Mark-Sweep，在空间不足以对从新生代晋升过来的对象进行分配时才使用 Mark-Compact。





### 2、内存指标

#### (1)查看内存使用情况

**查看进程的内存占用**

调用 process.memoryUsage() 可以查看内存使用情况：

```sh
$ node
> process.memoryUsage()
{
    rss: 13852672,
    heapTotal: 6131200,
    heapUsed: 2757120
}
```

其中 rss 是 resident set size 的缩写，即进程的常驻内存部分。进程中的内存总共有几部分，一部分是 rss，其余部分在交换区（swap）或者文件系统（filesystem）中。

除了 rss 外，heapTotal 和 heapUsed 对应的是 V8 的堆内存信息。heapTotal 是堆中总共申请的内存量，heapUsed 是目前堆中使用中的内存量，这三个值的单位都是字节。

**查看系统的内存占用**

os 模块提供了 totalmem() 和 freemem() 这两个方法用于查看操作系统的内存使用情况，它们分别返回系统的总内存和闲置内存，以字节为单位：

```sh
$ node
> os.totalmem()
8589934592
> os.freemem()
4527833088
```

#### (2)堆外内存

通过 process.memoryUsage() 的结果可以看出堆内的内存用量总是小于进程内的常驻内存用量，这意味着 Node 中的内存使用并非都是通过 V8 进行分配的，我们将那些不是通过 V8 分配的内存称为**堆外内存**。

Node 并不同于浏览器的应用场景。在浏览器中 JavaScript 直接处理字符串即可满足绝大多数的业务需求，而 Node 则需要处理网络流和文件 I/O 流，因此 Node 中的 Buffer 对象不同于其他对象，它不经过 V8 的内存分配机制，所以也不会有堆内存的大小限制。

### 3、内存泄漏

通常，造成内存泄漏的原因有如下三个：

- 缓存
- 队列消费不及时
- 作用域未释放

#### (1)慎将内存当作缓存

JavaScript 开发者通常喜欢用对象的键值对来缓存东西，但这与严格意义上的缓存又有着区别，严格意义上的有着完善的过期策略，而普通的键值对没有。使用键值对缓存对象的好处在于极其容易创建但是开发者往往容易忽略掉及时清除缓存对象。因此我们需要使用一些方法限定缓存对象的大小，防止内存无限制增长。举个例子：

```js
var limitMap = function (limit) {
    this.limit = limit || 10;
    this.map = {};
    this,keys = [];
}

var hasOwnProperty = Object.prototype.hasOwnProperty;

LimitMap.prototype.set = function (key, value) {
    var map = this.map;
    var keys = this.keys;
    if (!hasOwnProperty.call(map, key)) {
        if (keys.length === this.limit) {
            var firstKey = keys.shift();
            delete map[firstKey];
        }
        keys.push(key)
    }
    map[key] = value;
}

LimitMap.prototype.get = function (key) {
    return this.map[key];
}

module.exports = LimitMap
```

当然这种超过数量就以先进先出的方式进行淘汰的策略并不是十分高效，我们还可以采用 LRU 算法来进行处理。

除了这种直接使用键值对来当作缓存对象以外还有一种案例在于模块机制。为了加速模块的引入，所有模块都会通过编译执行，然后被缓存下来。由于通过 exports 导出的函数可以访问文件模块中的私有变量，这样每个文件模块在编译执行后形成的作用域因为模块缓存的原因不会被释放。因此在设计模块时要十分小心内存泄漏的出现。

直接将内存作为缓存的方案要十分慎重。除了限制缓存的大小外另外要考虑的事情是，进程之间无法共享内存。如果在进程内使用缓存，这些缓存不可避免地有重复，对物理内存的使用是一种浪费。如果使用大量缓存，目前比较好的解决方案是采用进程外的缓存，进程自身不存储状态。外部的缓存软件有着两孩的缓存过期淘汰策略以及自有的内存管理，不影响 Node 进程的性能。目前市场上比较好的缓存有 Redis 和 Memcached。

#### (2)关注队列状态

另一个不经意产生的内存泄漏则是队列。在 JavaScript 中可以通过队列（数组对象）来完成许多特殊的需求，队列经常在消费者-生产者模型中经常充当中间产物。这是一个容易忽略的情况，因为在大多数应用场景下，消费的速度远远大于生产的速度，内存泄漏不易发生。但是一旦消费的速度低于生产速度将会形成堆积。

这种情境下表层的解决方案是换用消费速度更高的技术，但是更深层的解决方案应该是监控队列的长度，一旦堆积，应当通过监控系统产生报警并通知相关人员。另一个解决方案是任意异步调用都应该包含超时机制，一旦在限定的时间内未完成响应，通过回调函数传递超时异常，使得任意异步调用的回调都具备可控的响应时间，给消费速度一个下限值。





#### (3)内存泄漏排查

在 Node 中由于 V8 的堆内存大小的限制，它对内存泄漏非常敏感。当在线服务的请求量变大时，哪怕是一个字节的泄漏都会导致内存占用过高，常用的内存泄漏排查工具有：

- v8-profiler
- node-headdump
- node-memwatch

### 4、大内存应用

由于 Node 的内存限制，操作大文件也要小心，好在 Node 提供了 stream 模块用于处理大文件。stream 继承自 EventEmitter，具备基本的自定义事件功能，同时抽象出标准的事件和方法，它分为可读和可写两种。Node 中大多数模块都有 stream 的应用，比如 fs 的 createReadStream() 和 createWriteStream() 方法可以分别用于创建文件的可读流和可写流，可读流提供了管道方法 pipe() ，封装了 data 事件和写入操作，举个例子：

```js
var reader = fs.createReadStream()
var writer = fs.createWriteStream()
reader.pipe(writer)
```

上述示例通过流的方式不会收到 V8 内存的限制，有效提高了程序的健壮性。

除此之外如果不需要进行字符串层面的操作，则不需要借助 V8 来处理，可以尝试进行纯粹的 Buffer 操作，这不会收到 V8 堆内存的限制，但是仍然需要小心物理内存的限制。

## 六、理解Buffer

### 1、Buffer结构

Buffer 是典型的 JavaScript 和 C++ 结合的模块。它将性能相关部分用 C++ 实现，将非性能相关的部分用 JavaScript 实现，如下图所示：

<img src="/assets/深入浅出NodeJS/11.png" width="400">

由于 Buffer 太过常见，Node 在进程启动时就已经加载了它，并将其放在全局对象上，所以在使用 Buffer 时，无须通过 require() 即可直接使用。

#### (1)Buffer对象

Buffer 对象类似于数组，元素为 16 进制的两位数，即 0～255 之间的数值。不同编码的字符串占用的元素各不相同，例如中文字符串在 UTF-8 编码下占据 3 个元素，字母和半角标点符号占用一个元素。

与数组类似，我们也可以通过 length 属性获取 Buffer 的长度，通过下标获取指定元素，举个例子：

```js
const buffer = new Buffer(100);
console.log(buffer.length) // 100
console.log(buffer[20]) // 0
```

同样我们也可以直接通过下标为 Buffer 进行赋值，不过需要注意的是如果给元素赋值的值超过 255 就会逐次减去 256，直到得到 0～255 区间的数值。如果小于 0，则会将数值逐次加 256,直到得到 0～255 区间的数值。如果是小数舍弃掉小数部分只保留整数部分。

#### (2)Buffer内存分配

Buffer 对象的内存分配不在 V8 的堆内存中，而是在 Node 的 C++ 层面实现内存的申请的，然后在 JavaScript 中分配内存，为了高效使用申请的内存 Node 采用了 slab 分配机制，简单而言 slab 就是一块申请好的固定大小的内存区域，slab 具有如下 3 种状态：

- full: 完全分配状态
- partial: 部分分配状态
- empty: 没有被分配的状态

Node 以 8KB 为界限来区分 Buffer 是大对象还是小对象：

```js
Buffer.poolSize = 8 * 1024
```

这个 8KB 也就是每个 slab 的大小值，在 JavaScript 层面，以它作为单位单元进行内存的分配。

**分配小 Buffer 对象**

如果指定 Buffer 的大小小于 8KB，Node 会按照小对象的方式进行分配。Buffer 的分配过程主要是使用一个局部变量 pool 作为中间处理对象，处于分配状态的 slab 单元都指向它，如果是分配一个全新的 slab 单元它将会申请一个新的 SlowBuffer 指向它：

```js
var pool;

function allocPool() {
    pool = new SlowBuffer(Buffer.poolSize);
    pool.used = 0;
}
```

举个例子：

```js
new Buffer(1024)
```

示例中我们新建了一个小 Buffer 对象，这次构造将会去检查 pool 对象，如果 pool 对象没有被创建，将会创建一个全新的 slab 单元指向它，这时 slab 单元处于 empty 状态：

```js
if (!pool || pool.length - pool.used < this.length) allocPool()
```

同时该 Buffer 对象的 parent 属性指向该 slab，并记录下是从 slab 的哪个位置（offset）开始使用的，slab 对象自身也记录被使用了多少字节，代码如下：

```js
this.parent = pool;
this.offset = pool.used;
pool.used += this.length;
if (pool.used & 7) pool.used = ()
```

这个时候 slab 单元处于 partial 状态。

当再次创建一个 Buffer 对象的时候构造过程就会判断这个 slab 的剩余空间是否足够，如果足够使用剩余空间，并更新 slab 的分配状态，如果 slab 的剩余空间不够，将会构造新的 slab，原 slab 中剩余的空间会造成浪费。

**分配大 Buffer 对象**

如果需要新建一个超过 8KB 的 Buffer 对象，将会直接分配一个 SlowBuffer 对象作为 slab 单元，这个 slab 单元将会被这个大 Buffer 对象独占。

```js
this.parent = new SlowBuffer(this.length)
this.offet = 0
```

需要注意 Buffer 对象都是 JavaScript 层面的，能够被 V8 的垃圾回收标记回收，但是其内部的 parent 属性指向的 slowBuffer 却是来自于 Node 自身 C++ 的定义，是 C++ 层面上的 Buffer 对象，所以内存不在 V8 的堆中，因此不推荐用户直接操作它。

**总结**

简单而言，真正的内存是在 Node 的 C++ 层面提供的，JavaScript 层面只是使用它。当进行小而频繁的 Buffer 操作时，采用 slab 的机制进行预先申请和事后分配，使得 JavaScript 到操作系统之间不必有过多的内存申请方面的系统调用。对于大块的 Buffer 而言，则直接使用 C++ 层面提供的内存，而无需细腻的分配操作。

### 2、Buffer的转换

Buffer 对象可以与字符串之间相互转换。

#### (1)字符串转 Buffer

字符串转 Buffer 对象主要是通过构造函数完成的：

```js
new Buffer(str, [encoding]);
```

通过构造函数转换的 Buffer 对象，存储的只能是一种编码类型。encoding 参数不传递时，默认按 UTF-8 编码进行转码和存储。

一个 Buffer 对象可以存储不同编码类型的字符串转码的值，调用 write() 方法可以实现这个目的，代码如下：

```js
buf.write(string, [offset], [length], [encoding])
```

由于可以不断写入内容到 Buffer 对象中，并且每次写入可以指定编码，所以 Buffer 对象中可以存在多种编码转化后的内容。因此 Buffer 反转字符串的时候需要格外小心。

#### (2)Buffer 转字符串

实现 Buffer 向字符串的转换也十分简单，Buffer 对象的 toString() 可以将 Buffer 对象转换为字符串，代码如下：

```js
buf.toString([encoding], [start], [end])
```

通过设置 encoding（默认UTF-8）、start、end这三个参数实现整体或局部的转换。如果 Buffer 对象由多种编码写入，就需要在局部指定不同的编码，才能转换回正常的编码。

### 3、Buffer 的拼接

为了解释 Buffer 最容易出现的乱码情况我们以如下代码为例：

```js
var fs = require('fs');

var rs = fs.createReadStream('test.md', { highWaterMark: 11 });
var data = '';
rs.on("data", function (chunk) {
    data += chunk;
})
rs.on("end", function () {
    console.log(data);
})
```

我们在测试数据中写入李白的《静夜思》，控制台最终输出结果为：

```
床前明��光，
疑是地上��。
举头望明月��
低头思故乡。
```

那么这里的乱码是如何产生的呢？

这是因为文件可读流在读取时会逐个读取 Buffer，这首诗的原始 Buffer 为 `<Buffer e5 ba 8a e5 89 8d e6 98 8e e6 9c 88 e5 85 89 ef bc 8c 0a e7 96 91 e6 98 af e5 9c b0 e4 b8 8a e9 9c 9c e3 80 82 0a e4 b8 be e5 a4 b4 e6 9c 9b e6 98 8e ... 25 more bytes>`，由于我们限定了 Buffer 对象的长度为 11，因此只读流需要读取 7 次才能完成完整的读取，结果是以下几个 Buffer 对象依次输出：

```
<Buffer e5 ba 8a e5 89 8d e6 98 8e e6 9c>
<Buffer 88 e5 85 89 ef bc 8c 0a e7 96 91>
<Buffer e6 98 af e5 9c b0 e4 b8 8a e9 9c>
<Buffer 9c e3 80 82 0a e4 b8 be e5 a4 b4>
<Buffer e6 9c 9b e6 98 8e e6 9c 88 ef bc>
<Buffer 8c 0a e4 bd 8e e5 a4 b4 e6 80 9d>
<Buffer e6 95 85 e4 b9 a1 e3 80 82>
```

buf.toString() 方法默认采用 UTF-8 编码，中文在 UTF-8 下占 3 个字节。所以第一个 Buffer 对象在输出时，只能显示 3 个字符，Buffer 中剩下的两个字节将会以乱码的方式显示。

在示例中我们构造了 11 这个限制，但是**对于任意长度的 Buffer 而言宽字节字符串都有可能存在被截断的情况**，只不过 Buffer 的长度越大出现的概率越低而已。

#### (1)setEncoding() 与 string_decoder()

可读流对象可以通过 setEncoding() 方法设置编码方式，在方法内部会为可读流对象设置一个 decorator 对象，每次 data 事件都通过该 decorator 对象进行 Buffer 到字符串的解码，然后传递给调用者。decorator 对象来自 string_decorator 模块 StringDecorator 的实例对象，它在对 Buffer 进行解码时会保留未成功解析的字符，等到下次字符流到来合并到一起进行解析，举个例子：

```js
var StringDecorator = require('string_decorator').StringDecorator;
var decorator = new StringDecorator('utf-8');

var buf1 = new Buffer([oxE5, oxBA, 0x8A, oxE5, ox89, ox8D, 0xE6, ox98, ox8E, oxE6, ox9C]);
console.log(decorator.write(buf1)); // 床前明

var buf2 = new Buffer([ox88, oxE5, 0x85, ox89, 0xEF, oxBC, ox8C, oxE7, ox96, ox91, oxE6]);
console.log(decorator.write(buf2)); // 月光，疑
```

decorator 在第一次 write() 时只输出前 9 个字节转码形成的字符，“月”字的前两个字节被保留在 StringDecorator 实例内部，第二次 write() 时将这剩余的两个字符与 11 个字符组合到一起再进行转码。

因此我们可以使用 setEncoding() 方法来解决《静夜思》的乱码问题：

```js
var rs = fs.createReadStream('test.md', { highWaterMark: 11 });
rs.setEncoding('utf-8');
...
```

虽然 string_decorator 模块很奇妙，但是它目前只能处理 UTF-8、Base64 和 UCS-2/UTF-16LE 这三种编码，所以 setEncoding 并不能从根本上解决这个问题。

#### (2)正确拼接 Buffer

除了 setEncoding 之外更好的解决方案是将多个小 Buffer 对象拼接成一个 Buffer 对象后再通过 iconv-lite（采用 JavaScript 实现的 Node 模块，支持更多的编码类型转换）一类的模块来转码，具体实现如下所示：

```js
var chunks = [];
var size = 0;
res.on("data", function (chunk) {
    chunks.push(chunk);
    size += chunk.length;
})
res.on("end", function () {
    var buf = Buffer.concat(chunks, size);
    var str = iconv.decode(buf, 'utf-8');
    console.log(str)
})
```

其中 Buffer.concat() 方法的具体实现如下：

```js
Buffer.concat = function (list, length) {
    if (!Array.isArray(list)) {
        throw new Error('Usage: Buffer.concat(list, [length])');
    }
    if (list.length === 0) {
        return new Buffer(0);
    } else if (list.length === 1) {
        return list[0];
    }

    if (typeof length !== 'number') {
        length = 0;
        for (var i = 0; i < list.length; i++) {
            var buf = list[i];
            length += buf.length;
        }
    }

    var buffer = new Buffer(length);
    var pos = 0;
    for (var i = 0; i < list.length; i++) {
        var buf = list[i];
        buf.copy(buffer, pos);
        pos += buf.length;
    }
    return buffer;
}
```

### 4、Buffer 与性能

在服务器传输数据时，通过预先转换静态内容为 Buffer 对象可以有效地减少 CPU 的重复使用，节省服务器的资源。在 Node 构建的 Web 应用中，可以选择将页面中动态内容和静态内容分离，静态内容部分可以通过预先转换为 Buffer 的方式使性能得到提升。由于文件自身是二进制数据，所以在不需要改变内容的场景下，尽量只读取 Buffer，然后直接传输，不做额外的转换，避免损耗。

Buffer 的使用除了与字符串的转换有性能损耗外在文件读取时，有一个 highWaterMark 设置对性能的影响也至关重要。

fs.createReadStream() 的工作方式是在内存中准备一段 Buffer，然后在 fs.read() 读取时逐步从磁盘中将字节复制到 Buffer 中。完成一次读取时，则从这个 Buffer 中通过 slice() 方法取出部分数据作为一个小 Buffer 对象，再通过 data 事件传递给调用方。如果 Buffer 用完则重新分配一个，如果还有一个则继续使用。这个过程与小 Buffer 对象的内存分配比较类似，highWaterMark 的大小对性能有两个影响的点：

- highWaterMark 设置对 Buffer 内存的分配和使用有一定的影响。
- heighWaterMark 设置过小，可能导致系统调用次数过多。

文件流读取基于 Buffer 分配，Buffer 则基于 SlowBuffer 分配，这可以理解为两个维度的分配策略。如果文件较小（小于 8KB），有可能造成 slab 未能完全使用。由于 fs.createReadStream() 内部采用 fs.read() 方法实现，将会引起对磁盘的系统调用，对于大文件而言，highWaterMark 的大小决定会触发系统调用和 data 事件的次数。


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