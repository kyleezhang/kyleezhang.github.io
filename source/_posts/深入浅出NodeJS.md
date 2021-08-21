---
title: 《深入浅出NodeJS》读书笔记
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

一般而言事件发布/订阅模式中事件与侦听器的关系是一对多，但在异步编程中也会出现事件与侦听器的关系是多对一的情况，所以往往会出现一直被诟病的函数嵌套过深的问题，例如：

```js
fs.readFile(template_path, 'utf8', function (err, template) {
    db.query(sql, function(er, data) {
        l10n.get(function (err, response) {
            // TODO
        })
    })
})
```

但是这种编程方式一方面导致回调函数嵌套过深，而且导致了可以并行调用但实际上只能并行执行的问题，我们可以通过引入**哨兵变量**来解决这个问题：

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

fs.readFile(template_path, 'utf8', function(err, template) {
    done("template", template)
})
db.query(sql, function (err, data) {
    done("data", data)
})
l10n.get(function (err, resource) {
    done("resource", resource)
})
```

除了这种方式之外我们还可以通过 [EventProxy](https://github.com/JacksonTian/eventproxy) 来实现。

