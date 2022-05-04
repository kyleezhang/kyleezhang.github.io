---
title: Webpack Dependency Graph深度解析
date: 2021-08-12 11:31:12
toc: true
mathjax: false
categories: 
- 前端工程化
tags: 
- webpack
---

> 非原创：原文地址[有点难的 webpack 知识点：Dependency Graph 深度解析](https://mp.weixin.qq.com/s/kr73Epnn6wAx9DH7KRVUaA)

Dependency Graph 概念来自官网 Dependency Graph | webpack 一文，原文解释是这样的：

> Any time one file depends on another, webpack treats this as a dependency. This allows webpack to take non-code assets, such as images or web fonts, and also provide them as dependencies for your application.
> When webpack processes your application, it starts from a list of modules defined on the command line or in its configuration file. Starting from these entry points, webpack recursively builds a dependency graph that includes every module your application needs, then bundles all of those modules into a small number of bundles - often, just one - to be loaded by the browser.

翻译过来核心意思是：webpack 处理应用代码时，会从开发者提供的 entry 开始递归地组建起包含所有模块的 dependency graph，之后再将这些 module 打包为 bundles 。

然而事实远不止官网描述的这么简单，Dependency Graph 贯穿 webpack 整个运行周期，从 make 阶段的模块解析，到 seal 阶段的 chunk 生成，以及 tree-shaking 功能都高度依赖于Dependency Graph ，是 webpack 资源构建的一个非常核心的数据结构

本文将围绕 webpack@v5.x 的 Dependency Graph 实现，展开讨论三个方面的内容：

- Dependency Graph 在 webpack 实现中以何种数据结构呈现
- Webpack 运行过程中如何收集模块间依赖关系，进而构建出 Dependency Graph
- Dependency Graph 构建完毕后，又是如何被消费的

<!-- more -->

## 一、Dependency Graph

本节将深入 webpack 源码，解读 Dependency Graph 的内在数据结构及依赖关系收集过程。在正式展开之前，有必要回顾几个 webpack 重要的概念：

- Module：资源在 webpack 内部的映射对象，包含了资源的路径、上下文、依赖、内容等信息
- Dependency ：在模块中引用其它模块，例如 `import "a.js"` 语句，webpack 会先将引用关系表述为 Dependency 子类并关联 module 对象，等到当前 module 内容都解析完毕之后，启动下次循环开始将 Dependency 对象转换为适当的 Module 子类。
- Chunk ：用于组织输出结构的对象，webpack 分析完所有模块资源的内容，构建出完整的 Dependency Graph 之后，会根据用户配置及 Dependency Graph 内容构建出一个或多个 chunk 实例，每个 chunk 与最终输出的文件大致上是一一对应的。

### 1、数据结构

Webpack 4.x 的 Dependency Graph 实现较简单，主要由 Dependence/Module 内置的系列属性记录引用、被引用关系。

而 Webpack 5.0 之后则实现了一套相对复杂的类结构记录模块间依赖关系，将模块依赖相关的逻辑从 Dependence/Module 解耦为一套独立的类型结构，主要类型有：

- ModuleGraph ：记录 Dependency Graph 信息的容器，一方面保存了构建过程中涉及到的所有 module 、dependency 对象，以及这些对象互相之间的引用；另一方面提供了各种工具方法，方便使用者迅速读取出 module 或 dependency 附加的信息
- ModuleGraphConnection ：记录模块间引用关系的数据结构，内部通过 originModule 属性记录引用关系中的父模块，通过 module 属性记录子模块。此外还提供了一系列函数工具用于判断对应的引用关系的有效性
- ModuleGraphModule ：Module 对象在 Dependency Graph 体系下的补充信息，包含模块对象的 incomingConnections —— 指向模块本身的 ModuleGraphConnection 集合，即谁引用了模块自己；outgoingConnections —— 该模块对外的依赖，即该模块引用了其他那些模块。

之间关系大致为：

<img src="/assets/webpack-dependency/01.png">

上面类图需要额外注意：

- ModuleGraph 对象通过 _dependencyMap 属性记录 Dependency 对象与 ModuleGraphConnection 连接对象之间的映射关系，后续的处理中可以基于这层映射迅速找到 Dependency 实例对应的引用与被引用者
- ModuleGraph 对象通过 _moduleMap 在 module 基础上附加 ModuleGraphModule 信息，而 ModuleGraphModule 最大的作用就是记录了模块的引用与被引用关系，后续的处理可以基于该属性找到 module 实例的所有依赖与被依赖关系

### 2、依赖收集过程

ModuleGraph、ModuleGraphConnection、ModuleGraphModule 三者协作，在 webpack 构建过程(make 阶段)中逐步收集模块间的依赖关系，webpack 整体构建流程图如下：

<img src="/assets/webpack-dependency/02.png">

依赖关系收集过程主要发生在两个节点：

- addDependency ：webpack 从模块内容中解析出引用关系后，创建适当的 Dependency 子类并调用该方法记录到 module 实例
- handleModuleCreation ：模块解析完毕后，webpack 遍历父模块的依赖集合，调用该方法创建 Dependency 对应的子模块对象，之后调用 compilation.moduleGraph.setResolvedModule 方法将父子引用信息记录到 moduleGraph 对象上

setResolvedModule 方法的逻辑大致为：

```js
class ModuleGraph {
    constructor() {
        /** @type {Map<Dependency, ModuleGraphConnection>} */
        this._dependencyMap = new Map();
        /** @type {Map<Module, ModuleGraphModule>} */
        this._moduleMap = new Map();
    }

    /**
     * @param {Module} originModule the referencing module
     * @param {Dependency} dependency the referencing dependency
     * @param {Module} module the referenced module
     * @returns {void}
     */
    setResolvedModule(originModule, dependency, module) {
        const connection = new ModuleGraphConnection(
            originModule,
            dependency,
            module,
            undefined,
            dependency.weak,
            dependency.getCondition(this)
        );
        this._dependencyMap.set(dependency, connection);
        const connections = this._getModuleGraphModule(module).incomingConnections;
        connections.add(connection);
        const mgm = this._getModuleGraphModule(originModule);
        if (mgm.outgoingConnections === undefined) {
            mgm.outgoingConnections = new Set();
        }
        mgm.outgoingConnections.add(connection);
    }
}
```

上例代码主要更改了 _dependencyMap 及 moduleGraphModule 的出入 connections 属性，以此收集当前模块的上下游依赖关系。

### 3、实例解析

看个简单例子，对于下面的依赖关系：

<img src="/assets/webpack-dependency/03.png" width="400">

Webpack 启动后，在构建阶段递归调用 compilation.handleModuleCreation 函数，逐步补齐 Dependency Graph 结构，最终可能生成如下数据结果：

```js
ModuleGraph: {
    _dependencyMap: Map(3){
        { 
            EntryDependency{request: "./src/index.js"} => ModuleGraphConnection{
                module: NormalModule{request: "./src/index.js"}, 
                // 入口模块没有引用者，故设置为 null
                originModule: null
            } 
        },
        { 
            HarmonyImportSideEffectDependency{request: "./src/a.js"} => ModuleGraphConnection{
                module: NormalModule{request: "./src/a.js"}, 
                originModule: NormalModule{request: "./src/index.js"}
            } 
        },
        { 
            HarmonyImportSideEffectDependency{request: "./src/a.js"} => ModuleGraphConnection{
                module: NormalModule{request: "./src/b.js"}, 
                originModule: NormalModule{request: "./src/index.js"}
            } 
        }
    },
    _moduleMap: Map(3){
        NormalModule{request: "./src/index.js"} => ModuleGraphModule{
            incomingConnections: Set(1) [
                // entry 模块，对应 originModule 为null
                ModuleGraphConnection{ module: NormalModule{request: "./src/index.js"}, originModule:null }
            ],
            outgoingConnections: Set(2) [
                // 从 index 指向 a 模块
                ModuleGraphConnection{ module: NormalModule{request: "./src/a.js"}, originModule: NormalModule{request: "./src/index.js"} },
                // 从 index 指向 b 模块
                ModuleGraphConnection{ module: NormalModule{request: "./src/b.js"}, originModule: NormalModule{request: "./src/index.js"} }
            ]
        },
        NormalModule{request: "./src/a.js"} => ModuleGraphModule{
            incomingConnections: Set(1) [
                ModuleGraphConnection{ module: NormalModule{request: "./src/a.js"}, originModule: NormalModule{request: "./src/index.js"} }
            ],
            // a 模块没有其他依赖，故 outgoingConnections 属性值为 undefined
            outgoingConnections: undefined
        },
        NormalModule{request: "./src/b.js"} => ModuleGraphModule{
            incomingConnections: Set(1) [
                ModuleGraphConnection{ module: NormalModule{request: "./src/b.js"}, originModule: NormalModule{request: "./src/index.js"} }
            ],
            // b 模块没有其他依赖，故 outgoingConnections 属性值为 undefined
            outgoingConnections: undefined
        }
    }
}
```

从上面的 Dependency Graph 可以看出，本质上 ModuleGraph._moduleMap 已经形成了一个有向无环图结构，其中字典 _moduleMap 的 key 为图的节点，对应 value ModuleGraphModule 结构中的 outgoingConnections 属性为图的边，则上例中从起点 index.js 出发沿 outgoingConnections 向前可遍历出图的所有顶点。

## 二、作用

以 webpack@v5.16.0 为例，关键字 moduleGraph 出现了 1277 次，几乎覆盖了 webpack/lib 文件夹下的所有文件，其作用可见一斑。虽然出现的频率很高，但总的来说可以看出有两个主要作用：信息索引、转变为 ChunkGraph 以确定输出结构。

### 1、信息索引

ModuleGraph 类型提供了很多实现 module / dependency 信息查询的工具函数，例如：

- getModule(dep: Dependency) ：根据 dep 查找对应的 module 实例
- getOutgoingConnections(module: Module) ：查找 module 实例的所有依赖
- getIssuer(module: Module) ：查找 module 在何处被引用
- 等等。

Webpack@v5.x 内部的许多插件、Dependency 子类、Module 子类的实现都需要用到这些工具函数查找特定模块、依赖的信息，例如：

- SplitChunksPlugin 在优化 chunks 处理中，需要使用 moduleGraph.getExportsInfo 查询各个模块的 exportsInfo 信息以确定如何分离 chunk。
- 在 compilation.seal 函数中，需要遍历 entry 对应的 dep 并调用 moduleGraph.getModule 获取完整的 module 定义
- ...

那么，在您编写插件时，可以考虑适度参考 webpack/lib/ModuleGraph.js 中提供的方法，确认可以获取使用那些函数获取到您所需要的信息。

### 2、构建ChunkGraph

webpack 实现中，原始的资源模块以 Module 对象形式存在、流转、解析处理。而 Chunk 则是输出产物的基本组织单位，在生成阶段 webpack 按规则将 entry 及其它 Module 插入 Chunk 中，之后再由 SplitChunksPlugin 插件根据优化规则与 ChunkGraph 对 Chunk 做一系列的变化、拆解、合并操作，重新组织成一批性能(可能)更高的 Chunks 。运行完毕之后 webpack 继续将 chunk 一一写入物理文件中，完成编译工作。

综上，Module 主要作用在 webpack 编译过程的前半段，解决原始资源“「如何读」”的问题；而 Chunk 对象则主要作用在编译的后半段，解决编译产物“「如何写」”的问题，两者合作搭建起 webpack 搭建主流程。

Chunk 的编排规则非常复杂，涉及 entry、optimization 等诸多配置项，Webpack 主体流程中，make 构建阶段结束之后会进入 seal 阶段，开始梳理以何种方式组织输出内容。在 webpack@v4.x 时，seal 阶段主要围绕 Chunk 及 ChunkGroup 两个类型展开，而到了 5.0 之后，与 Dependency Graph 类似也引入了一套全新的基于 ChunkGraph 的图结构实现资源生成算法。

到了生成(seal) 阶段，webpack 会根据模块依赖图的内容组织分包 —— Chunk 对象，默认的分包规则有：

同一个 entry 下触达到的模块组织成一个 chunk
异步模块单独组织为一个 chunk
entry.runtime 单独组织成一个 chunk
默认规则集中在 compilation.seal 函数实现，seal 核心逻辑运行结束后会生成一系列的 Chunk、ChunkGroup、ChunkGraph 对象，后续如 SplitChunksPlugin 插件会在 Chunk 系列对象上做进一步的拆解、优化，最终反映到输出上才会表现出复杂的分包结果。

#### (1) Entry 分包处理

在 compilation.seal 函数中，首先根据默认规则 —— 每个 entry 对应组织为一个 chunk ，之后调用 webpack/lib/buildChunkGraph.js 文件定义的 buildChunkGraph 方法，遍历 make 阶段生成的 moduleGraph 对象从而将 module 依赖关系转化为 chunkGraph 对象。

```js
module.exports = {
  entry: {
    main: "./src/main",
    home: "./src/home",
  }
};
```

对于上述配置 Webpack 会遍历 entry 对象属性并创建出 `chunk[main]`、`chunk[home]` 两个对象，此时两个 chunk 分别包含 main 、home 模块：

<img src="/assets/webpack-dependency/04.webp" width="600">

初始化完毕后，Webpack 会读取 ModuleDependencyGraph 的内容，将 entry 所对应的内容塞入对应的 chunk (发生在 webpack/lib/buildChunkGrap.js 文件)。比如对于如下文件依赖：

<img src="/assets/webpack-dependency/05.png" width="700">

main.js 以同步方式直接或间接引用了 a/b/c/d 四个文件，分析 ModuleDependencyGraph 过程会逐步将 a/b/c/d 模块逐步添加到 chunk[main] 中，最终形成：

<img src="/assets/webpack-dependency/06.png" width="600">

#### (2)异步模块分包处理

Webpack 4 之后，只需要用异步语句 require.ensure("./xx.js") 或 import("./xx.js") 方式引入模块，就可以实现模块的动态加载，这种能力本质也是基于 Chunk 实现的。

Webpack 生成阶段中，遇到异步引入语句时会为该模块单独生成一个 chunk 对象，并将其子模块都加入这个 chunk 中。例如对于下面的例子：

```js
// index.js, entry 文件
import 'sync-a'
import 'sync-b'

import('async-c')
```

在 index.js 中，以同步方式引入 sync-a、sync-b；以异步方式引入 async-a 模块；同时，在 async-a 中以同步方式引入 sync-c 模块。对应的模块依赖如：

<img src="/assets/webpack-dependency/07.png" width="600">

此时，webpack 会为入口 index.js、异步模块 async-a.js 分别创建分包，形成如下数据：

<img src="/assets/webpack-dependency/08.png" width="600">

这里需要引入一个新的概念 —— Chunk 间的父子关系。由 entry 生成的 Chunk 之间相互孤立，没有必然的前后依赖关系，但异步生成的 Chunk 则不同，引用者(上例 index.js 块)需要在特定场景下使用被引用者(上例 async-a 块)，两者间存在单向依赖关系，在 webpack 中称引用者为 parent、被引用者为 child，分别存放在 `ChunkGroup._parents` 、`ChunkGroup._children` 属性中。

上述分包方案默认情况下会生成两个文件：

- 入口 index 对应的 index.js
- 异步模块 async-a 对应的 src_async-a_js.js

运行时，webpack 在 index.js 中使用 promise 及 `__webpack_require__.e` 方法异步载入并运行文件 src_async-a_js.js ，从而实现动态加载。

#### (3)Runtime 分包

除了 entry、异步模块外，webpack 5之后还支持基于 runtime 的分包规则。除业务代码外，Webpack 编译产物中还需要包含一些用于支持 webpack 模块化、异步加载等特性的支撑性代码，这类代码在 webpack 中被统称为 runtime。举个例子，产物中通常会包含如下代码：

```js
/******/ (() => {
  // webpackBootstrap
  /******/ var __webpack_modules__ = {}; // The module cache
  /************************************************************************/
  /******/ /******/ var __webpack_module_cache__ = {}; // The require function
  /******/

  /******/ /******/ function __webpack_require__(moduleId) {

    /******/ /******/ __webpack_modules__[moduleId](
      module,
      module.exports,
      __webpack_require__
    ); // Return the exports of the module
    /******/

    /******/ /******/ return module.exports;
    /******/
  } // expose the modules object (__webpack_modules__)
  /******/

  /******/ /******/ __webpack_require__.m = __webpack_modules__; /* webpack/runtime/compat get default export */
  /******/

  // ...
})();
```

编译时，Webpack 会根据业务代码决定输出那些支撑特性的运行时代码(基于 Dependency 子类)，例如：

- 需要 `__webpack_require__.f`、`__webpack_require__.r` 等功能实现最起码的模块化支持
- 如果用到动态加载特性，则需要写入 `__webpack_require__.e` 函数
- 如果用到 Module Federation 特性，则需要写入 `__webpack_require__.o` 函数
- 等等

虽然每段运行时代码可能都很小，但随着特性的增加，最终结果会越来越大，特别对于多 entry 应用，在每个入口都重复打包一份相似的运行时代码显得有点浪费，为此 webpack 5 专门提供了 entry.runtime 配置项用于声明如何打包运行时代码。用法上只需在 entry 项中增加字符串形式的 runtime 值，例如：

```js
module.exports = {
  entry: {
    index: { import: "./src/index", runtime: "solid-runtime" },
  }
};
```

Webpack 执行完 entry、异步模块分包后，开始遍历 entry 配置判断是否带有 runtime 属性，如果有则创建以 runtime 值为名的 Chunk，因此，上例配置将生成两个chunk：chunk[index.js] 、chunk[solid-runtime]，并据此最终产出两个文件：

- 入口 index 对应的 index.js 文件
- 运行时配置对应的 solid-runtime.js 文件

在多 entry 场景中，只要为每个 entry 都设定相同的 runtime 值，webpack 运行时代码最终就会集中写入到同一个 chunk，例如对于如下配置：

```js
module.exports = {
  entry: {
    index: { import: "./src/index", runtime: "solid-runtime" },
    home: { import: "./src/home", runtime: "solid-runtime" },
  }
};
```

入口 index、home 共享相同的 runtime ，最终生成三个 chunk，分别为：

<img src="/assets/webpack-dependency/09.png" width="600">

同时生成三个文件：

- 入口 index 对应的 index.js
- 入口 index 对应的 home.js
- 运行时代码对应的 solid-runtime.js

#### (4)分包规则的问题

至此，webpack 分包规则的基本逻辑就介绍完毕了，实现上，大部分功能代码都集中在：

- webpack/lib/compilation.js 文件的 seal 函数
- webpack/lib/buildChunkGraph.js 的 buildChunkGraph 函数

默认分包规则最大的问题是无法解决模块重复，如果多个 chunk 同时包含同一个 module，那么这个 module 会被不受限制地重复打包进这些 chunk。比如假设我们有两个入口 main/index 同时依赖了同一个模块：

<img src="/assets/webpack-dependency/10.png" width="600">

默认情况下，webpack 不会对此做额外处理，只是单纯地将 c 模块同时打包进 main/index 两个 chunk，最终形成：

<img src="/assets/webpack-dependency/11.png" width="600">

可以看到 chunk 间互相孤立，模块 c 被重复打包，对最终产物可能造成不必要的性能损耗！

为了解决这个问题，webpack 3 引入 CommonChunkPlugin 插件试图将 entry 之间的公共依赖提取成单独的 chunk，但 CommonChunkPlugin 本质上是基于 Chunk 之间简单的父子关系链实现的，很难推断出提取出的第三个包应该作为 entry 的父 chunk 还是子 chunk，CommonChunkPlugin 统一处理为父 chunk，某些情况下反而对性能造成了不小的负面影响。在 webpack 4 之后则引入了更负责的设计 —— ChunkGroup 专门实现关系链管理，配合 SplitChunksPlugin 能够更高效、智能地实现「启发式分包」。
