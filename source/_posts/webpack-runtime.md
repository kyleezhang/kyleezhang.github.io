---
title: Webpack运行机制
date: 2020-12-16 15:20:42
toc: true
mathjax: false
categories: 
- 前端
tags: 
- 前端工程化
---

其实 Webpack 官网首屏的英雄区就已经很清楚地描述了它的工作原理，如下图所示：
<img src="/assets/webpack-runtime/01.png" width="760" />
<!--more-->
以一个普通的前端项目为例，项目中一般都会散落着各种各样的代码及资源文件，比如 JS、CSS、图片、字体等，这些文件在 Webpack 的思想中都属于当前项目中的一个模块。Webpack 可以通过打包，将它们最终聚集到一起。Webpack 在整个打包的过程中：

- 通过 Loader 处理特殊类型资源的加载，例如加载样式、图片；
- 通过 Plugin 实现各种自动化的构建任务，例如自动压缩、自动发布。

具体来看打包的过程，Webpack 启动后，会根据我们的配置，找到项目中的某个指定文件（一般这个文件都会是一个 JS 文件）作为入口。然后顺着入口文件中的代码，根据代码中出现的 import（ES Modules）或者是 require（CommonJS）之类的语句，解析推断出来这个文件所依赖的资源模块，然后再分别去解析每个资源模块的依赖，周而复始，最后形成整个项目中所有用到的文件之间的依赖关系树，具体过程如下所示：
<img src="/assets/webpack-runtime/02.gif" width="680" />

有了这个依赖关系树过后， Webpack 会遍历（递归）这个依赖树，找到每个节点对应的资源文件，然后根据配置选项中的 Loader 配置，交给对应的 Loader 去加载这个模块，最后将加载的结果放入 bundle.js（打包结果）中，从而实现整个项目的打包，具体过程如下所示：
<img src="/assets/webpack-runtime/03.gif" />

下面我们详细了解一下 Webpack 的核心工作流程：

## Loader机制

loader 用于对模块的源代码进行转换。loader 可以使你在 import 或"加载"模块时预处理文件。因此，loader 类似于其他构建工具中“任务(task)”，并提供了处理前端构建步骤的强大方法。loader 可以将文件从不同的语言（如 TypeScript转换为 JavaScript)，或将内联图像转换为 data URL。loader 甚至允许你直接在 JavaScript 模块中 import CSS文件！通俗点讲就是loader在webpack中担任翻译官的角色。
Webpack 配置文件中对 Loader 的配置主要分为：匹配规则和匹配规则后的应用。

### 一、loader的匹配规则

loader 的匹配规则包括下面几种：

- { test: ... } 指定特定后缀名文件
- { include: ... } 指定特定的文件路径
- { exclude: ... } 排除特定的路径
- { and: [...] } 必须匹配数组中所有条件
- { or: [...] } 匹配数组中任意一个条件
- { not: [...] } 排除匹配数组中所有条件

上述的所谓条件的值可以是：

- 字符串：必须以提供的字符串开始，所以是字符串的话，这里我们需要提供绝对路径
- 正则表达式：调用正则的 test 方法来判断匹配
- 函数：(path) => boolean，返回 true 表示匹配
- 数组：至少包含一个条件的数组
- 对象：匹配所有属性值的条件

举个🌰:
```js
rules: [
  {
    test: /\.jsx?/, // 正则
    include: [
      path.resolve(__dirname, 'src'), // 字符串，注意是绝对路径
    ], // 数组
    // ...
  },
  {
    test: {
      js: /\.js/,
      jsx: /\.jsx/,
    }, // 对象，不建议使用
    not: [
      (value) => { /* ... */ return true; }, // 函数，通常需要高度自定义时才会使用
    ],
  },
],
```

### 二、loader配置

通过匹配规则匹配到对应的资源后最重要的还是通过use字段指定具体loader的应用，use字段我们可以简单指定 loader 名称，也可以指定具体配置，举个🌰:

```js
rules: [
  {
    test: /\.less/,
    use: [
      'style-loader', // 直接使用字符串表示 loader
      {
        loader: 'css-loader',
        options: {
          importLoaders: 1
        },
      }, // 用对象表示 loader，可以传递 loader 配置等
      {
        loader: 'less-loader',
        options: {
          noIeCompat: true
        }, // 传递 loader 配置
      },
    ],
  },
],
```

这儿需要注意的就是数组的解析顺序是从右往左。

### 三、实现一个loader

实现一个 loader 必须遵循如下准则：

- 简单【Simple】loader只做单一任务，多个单功能loader > 一个多功能loader
- 链式【Chaining】遵循链式调用原则
- 无状态【Stateless】即函数式里的Pure Function，无副作用
- 使用工具库【Loader Utilities】充分利用 loader-utils 包

每个 Webpack 的 Loader 都需要导出一个函数，这个函数就是我们这个 Loader 对资源的处理过程，它的输入就是加载到的资源文件内容，输出就是我们加工后的结果。我们通过 source 参数接收输入，通过返回值输出。loader 分为同步 loader 和 异步 loader，因为其本质是一个函数。所以这两种 loader 的区别，可以理解为只是返回值不同（只是为了方便理解的类比）。

#### 1、同步loader

同步 loader 比较简单，无论是 return 还是调用 this.callback 都可以同步地返回转换后的 content 内容，不过 this.callback 的函数参数类型更为丰富：

```typescript
this.callback(
  err: Error | null,
  content: string | Buffer,
  sourceMap?: SourceMap,
  meta?: any
);
```
举个🌰:

```ts
/** webpack.config.js */  
const path = require("path");  
  
module.exports = {  
  entry: {  
    index: path.resolve(__dirname, "src/index.js"),  
  },  
  output: {  
    path: path.resolve(__dirname, "dist"),  
  },  
  module: {  
    rules: [  
      {  
        test: /\.js$/,  
        use: [  
          {  
            loader: path.resolve("lib/loader/loader1.js"),  
            options: {  
              message: "this is a message",  
            }  
          }  
        ],  
      },  
    ],  
  },  
}; 

/** lib/loader/loader1.js */  
const loaderUtils = require('loader-utils');  
  
/** 过滤console.log和换行符 */  
module.exports = function (source) {
  
  // 获取loader配置项  
  const options = loaderUtils.getOptions(this);  // 此处也可通过this.query获取options信息
  
  console.log('loader配置项:', options);  
  
  const result = source  
    .replace(/console.log\(.*\);?/g, "")  
    .replace(/\n/g, "")  
    .concat(`console.log("${options.message || '没有配置项'}");`);  
  
  return result;
  // 或this.callback(null, source);
};  
```
在上述示例中我们实现了一个简单的 loader ，功能是替换`console.log`、去除换行符、在文件结尾处增加一行自定义内容，在示例中我们通过 loader-utils 包来获取配置参数，实际上我们也可以使用 Webpack 提供的 `this.query` 参数来获取 loader 配置中的 options 参数。

#### 2、异步loader

异步loader只有一种处理方式。对于异步 loader，使用 this.async 来获取 callback 函数。这里的 callback 函数，和同步 loader 里面 this.callback 的参数是一样的，举个🌰:

```typescript
module.exports = function (source) {  
  
  let count = 1;  
  
  // 1.调用this.async() 告诉webpack这是一个异步loader，需要等待 callback 回调之后再进行下一个loader处理  
  // 2.this.async 返回异步回调，调用表示异步loader处理结束  
  const callback = this.async();  
  
  const timer = setInterval(() => {  
    console.log(`时间已经过去${count++}秒`);  
  }, 1000);  
  
  // 异步操作  
  setTimeout(() => {  
    clearInterval(timer);  
    callback(null, source);  
  }, 3200);  
  
};
```

此处需要注意的是虽然 Webpack 加载资源文件的过程类似于一个工作管道，你可以在这个过程中依次使用多个 Loader，但是最终这个管道结束过后的结果必须是一段标准的 JS 代码字符串，如果当前 loader 返回的结果不是 JS 代码字符串，我们可以再选择合适的加载器，在后面接着处理我们这里得到的结果，举个🌰:

```javascript
// ./markdown-loader.js
const marked = require('marked')

module.exports = source => {
  // 1. 将 markdown 转换为 html 字符串
  const html = marked(source)
  return html
}

// ./webpack.config.js

module.exports = {
  entry: './src/main.js',
  output: {
    filename: 'bundle.js',
  },
  module: {
    rules: [
      {
        test: /\.md$/,
        use: [
          'html-loader',
          './markdown-loader'
        ]
      }
    ]
  }
}
```

## Plugin机制

Webpack 插件机制的目的是为了增强 Webpack 在项目自动化构建方面的能力，Loader 主要负责完成项目中各种各样资源模块的加载，从而实现整体项目的模块化，而 Plugin 则是用来解决项目中除了资源模块打包以外的其他自动化工作，所以说 Plugin 的能力范围更广，用途自然也就更多。
Webpack 的插件引入极为简单，一般来说，当我们有了某个自动化的需求过后，可以先去找到一个合适的插件，然后安装这个插件，最后将它配置到 Webpack 配置对象的 plugins 数组中，这个过程唯一有可能不一样的地方就是，有的插件可能需要有一些配置参数。

### 一、实现一个plugin

通过前面的介绍，我们知道相比于 Loader，插件的能力范围更宽，因为 Loader 只是在模块的加载环节工作，而插件的作用范围几乎可以触及 Webpack 工作的每一个环节。
那么，这种插件机制是如何实现的呢？其实 Webpack 的插件机制就是我们在软件开发中最常见的钩子机制，Webpack 钩子机制的实现主要通过第三方库：Tapable，Tapable 是一个类似于 Node.js 的 EventEmitter 的库，主要是控制钩子函数的发布与订阅，Tapable 库为插件提供了很多 Hook 以便挂载：

- SyncHook：同步钩子
- SyncBailHook：同步熔断钩子
- WaterfallHook：同步流水钩子
- LoopHook：同步循环钩子
- cParalleHook：异步并发钩子
- cRarallelBailHook：异步并发熔断钩子
- cSeriesHook：异步串行钩子
- cSeriesBailHook：异步串行熔断钩子
- cSeriesWaterfallHook：异步串行流水钩子

关于 Tapable 详细信息请查看[这篇文章](https://zhuanlan.zhihu.com/p/79221553)。

Webpack 中 Compiler 和 Compilation 对象(Compiler 对象是 Webpack 工作过程中最核心的对象，里面包含了构建过程的所有配置信息，Compilation 对象可以理解为此次打包的上下文)都继承于 Tapable，为了便于插件的扩展，Webpack 几乎在每一个环节都埋下了一个钩子。这样我们在开发插件的时候，通过往这些不同节点上挂载不同的任务，就可以轻松扩展 Webpack 的能力。
具体有哪些预先定义好的钩子，我们可以参考官方文档的 API：

- [Compiler Hooks](https://webpack.js.org/api/compiler-hooks/)
- [Compilation Hooks](https://webpack.js.org/api/compilation-hooks/)
- [JavascriptParser Hooks](https://webpack.js.org/api/parser/)

plugin 的实现要点如下：
- 一个命名JS函数或者JS类
- 在 prototype 上定义一个 apply 方法（供webpack调用，并且在调用时注入 compiler 对象）
- 在 apply 函数中需要有通过 compiler 对象挂载的 webpack 事件钩子（钩子函数中能拿到当前编译的 compilation 对象）
- 处理 webpack 内部实例的特定数据
- 功能完成后调用 webpack 提供的回调

举个🌰:

```javascript
// 插件功能：移除打包文件中的注释信息
// ./remove-comments-plugin.js
class RemoveCommentsPlugin {
  apply (compiler) {
    compiler.hooks.emit.tap('RemoveCommentsPlugin', compilation => {
      for (const name in compilation.assets) {
        if (name.endsWith('.js')) {
          const contents = compilation.assets[name].source()
          const noComments = contents.replace(/\/\*{2,}\/\s?/g, '')
          compilation.assets[name] = {
            source: () => noComments,
            size: () => noComments.length
          }
        }
      }
    })
  }
}
```
在上述示例中我们选择了一个叫作 emit 的钩子，这个钩子会在 Webpack 即将向输出目录输出文件时执行，然后通过 compiler 对象的 hooks 属性访问到 emit 钩子，再通过 tap 方法注册一个钩子函数，这个方法接收两个参数：

- 第一个是插件的名称，我们这里的插件名称是 RemoveCommentsPlugin；
- 第二个是要挂载到这个钩子上的函数；

然后我们在这个函数中接收一个 compilation 对象参数并使用对象中的 assets 属性获取即将写入输出目录的资源文件信息，它是一个对象，我们这里通过 for in 去遍历这个对象，其中键就是每个文件的名称，然后我们判断文件名是不是以`.js`结尾，如果是 JS 文件，我们将文件内容得到，再通过正则替换的方式移除掉代码中的注释，最后覆盖掉 compilation.assets 中对应的对象，在覆盖的对象中，我们同样暴露一个 source 方法用来返回新的内容。另外还需要再暴露一个 size 方法，用来返回内容大小，这是 Webpack 内部要求的格式，
tap 是 Tapable 提供的同步钩子的注册方法，如果是异步钩子则必须执行异步回调，举个🌰:

```javascript
// 1、Plugin名称  
const MY_PLUGIN_NAME = "MyBasicPlugin";  
  
class MyBasicPlugin {  
  // 2、在构造函数中获取插件配置项  
  constructor(option) {  
    this.option = option;  
  }  
  
  // 3、在原型对象上定义一个apply函数供webpack调用  
  apply(compiler) {  
    // 4、注册webpack事件监听函数  
    compiler.hooks.emit.tapAsync(  
      MY_PLUGIN_NAME,  
      (compilation, asyncCallback) => { 

        // 5、操作Or改变compilation内部数据  
        console.log(compilation);        
  
        console.log("当前阶段 ======> 编译完成，即将输出到output目录");  
  
        // 6、如果是异步钩子，结束后需要执行异步回调  
        asyncCallback();  
      }  
    );  
  }  
}  
  
// 7、模块导出  
module.exports = MyBasicPlugin; 
```

## Webpack打包流程

（注：文章中源码部分分别来自于 v3.3.11 版本的 webpack-cli 和 v4.43.0 版本的 Webpack）

### 一、初始化启动

当通过命令行启动 Webpack 后，npm会让命令行工具进入node_modules.bin目录，然后执行目录下的 webpack.js文件，文件内的核心内容如下：

```typescript
// node_modules/webpack/bin/webpack.js
// 正常执行返回
process.exitCode = 0;
/**
 * 通过 child_process 快速的运行某个命令
 * @param {string} command process to run
 * @param {string[]} args commandline arguments
 * @returns {Promise<void>} promise
 */
const runCommand = (command, args) => {...};
/**
 * 通过require.resolve判断某一个包是否存在
 * @param {string} packageName name of the package
 * @returns {boolean} is the package installed?
 */
const isInstalled = packageName => {...};
// wecpack可用的CLI：webpaclk-cli和webpack-command
const CLIs = [
  {
    name: "webpack-cli",
    package: "webpack-cli",
    binName: "webpack-cli",
    alias: "cli",
    installed: isInstalled("webpack-cli"),
    recommended: true,
    url: "https://github.com/webpack/webpack-cli",
    description: "The original webpack full-featured CLI.",
  },
  {
    name: "webpack-command",
    package: "webpack-command",
    binName: "webpack-command",
    alias: "command",
    installed: isInstalled("webpack-command"),
    recommended: false,
    url: "https://github.com/webpack-contrib/webpack-command",
    description: "A lightweight, opinionated webpack CLI.",
  }
]
// 判断是否两个CLI是否安装了
const installedClis = CLIs.filter(cli=>cli.installed);
// 根据安装数量进行处理
if (installedClis.length === 0) {
  // 终端询问用户是否安装 webpack-cli
  ...
} else if (installedClis.length === 1) {
  // 直接通过require方法调用已安装 CLI
  ...
} else {
  // 终端提示安装了两个CLI，只能使用一个，请移除另一个
  ...
}
```
这儿的 node_modules/.bin/webpack.js 实际上就是 Webpack 打包流程的入口文件，启动后 Webpack 会判断是否安装了可用的CLI: webpack-cli 或 webpack-command（webpack-cli功能更加丰富），并根据安装数量进行相应的处理，最终 webpack 会找到 webpack-cli(webpack-command) 这个 npm 包并且执行 CLI。

### 二、webpack-cli

从 Webpack 4 开始 Webpack 的 CLI 部分就被单独抽到了 webpack-cli 模块中，目的是为了增强 Webpack 本身的灵活性，Webpack CLI 的作用就是将 CLI 参数和 Webpack 配置文件中的配置整合，得到一个完整的配置对象。那么 webpack-cli 中又做了哪些操作呢？

首先，webpack-cli 会处理不需要经过编译的命令：

```javascript
// node_modules/webpack-cli/bin/cli.js
const NON_COMPILATION_ARGS = [
  "init",
  "migrate",
  "add",
  "remove",
  /*
  "update",
  "make",
  */
  "serve",
  "generate-loader",
  "generate-plugin",
  "info"
];

const NON_COMPILATION_CMD = process.argv.find(arg => {
  if (arg === "serve") {
    global.process.argv = global.process.argv.filter(a => a !== "serve");
    process.argv = global.process.argv;
  }
  return NON_COMPILATION_ARGS.find(a => a === arg);
});

if (NON_COMPILATION_CMD) {
  return require("./prompt-command")(NON_COMPILATION_CMD, ...process.argv);
}
```

然后 Webpack CLI 会通过 yargs 模块解析 CLI 参数，所谓 CLI 参数指的就是我们在运行 webpack 命令时通过命令行传入的参数，例如 --mode=production，具体实现代码如下：

```typescript
// yargs will terminate the process early when the user uses help or version.
// This causes large help outputs to be cut short (https://github.com/nodejs/node/wiki/API-changes-between-v0.10-and-v4#process).
// To prevent this we use the yargs.parse API and exit the process normally
yargs.parse(process.argv.slice(2), (err, argv, output) => {
  ...
  let options;
  try {
      // 转换命令行参数
      options = require("./convert-argv")(argv);
  } catch (err) {
      ...
  }
  ...
})
```

我们发现在 yargs.parse 的回调函数中还调用了 bin/utils/convert-argv.js 模块，将得到的命令行参数转换为 Webpack 的配置选项对象，在 convert-argv.js 工作过程中，首先为传递过来的命令行参数设置了默认值:

<img src="/assets/webpack-runtime/04.png" />

然后判断了命令行参数中是否指定了一个具体的配置文件路径，如果指定了就加载指定配置文件，反之则需要根据默认配置文件加载规则找到配置文件，具体代码如下：

<img src="/assets/webpack-runtime/05.png" />

找到配置文件过后，webpack-cli 将配置文件中的配置和 CLI 参数中的配置合并，如果出现重复的情况，会优先使用 CLI 参数，最终得到一个完整的配置选项。
有了配置选项过后，开始载入 Webpack 核心模块，传入配置选项，创建 Compiler 对象，这个 Compiler 对象就是整个 Webpack 工作过程中最核心的对象了，负责完成整个项目的构建工作。

<img src="/assets/webpack-runtime/06.png" />

### 三、创建 Compiler 对象

随着 Webpack CLI 载入 Webpack 核心模块，整个执行过程就到了 Webpack 模块中，所以这一部分的代码需要回到 Webpack 模块中，同样，这里我们需要找到这个模块的入口文件，也就是 lib/webpack.js 文件。这个文件导出的是一个用于创建 Compiler 的函数，具体如下：

```js
const webpack = (options, callback) => {
  ...
  let compiler;
  // 校验外部传入options
  if (Array.isArray(options)) {
    // 如果options为数组开启多路打包
    compiler = new MultiCompiler(options.map(options => webpack(options)));
  } else if (typeof options === "object") {
    // 如果options为对象开启单路打包
    options = new WebpackOptionsDefaulter().process(options); // 1.处理options
    compiler = new Compiler(options.context); // 2.创建compiler
    compiler.options = options;
    new NodeEnvironmentPlugin().apply(compiler);
    if (options.plugins && Array.isArray(options.plugins)) {
      // 3.注册已配置的插件
      for (const plugin of options.plugins) {
        if (typeof plugin === "function") {
          plugin.call(compiler, compiler);
        } else {
          plugin.apply(compiler);
        }
      }
    }
    // 4.触发特定的Hook
    compiler.hooks.environment.call();
    compiler.hooks.afterEnvironment.call();
    // 5.处理options
    compiler.options = new WebpackOptionsApply().process(options, compiler);
  } else {
    throw new Error("Invalid argument: options");
  }
  if (callback) {
    if (typeof callback !== 'function') {
      throw new Error("Invalid argument: callback");
    }
    if (options.watch === true || (Array.isArray(options)) && options.some(o => o.watch)) {
      const watchOptions = Array.isArray(options) ? options.map(o => o.watchOptions || {}): options.watchOptions || {};
			return compiler.watch(watchOptions, callback);
    }
    compiler.run(callback);
  }
  return compiler;
};
```

在这个函数中，首先校验了外部传递过来的 options 参数是否符合要求，紧接着判断了 options 的类型。如果传入的是一个数组，那么 Webpack 内部创建的就是一个 MultiCompiler，也就是同时开启多路打包，配置数组中的每一个成员就是一个独立的配置选项。而如果我们传入的是普通的对象，流程主要分为五步：

1. 处理options
2. 创建compiler
3. 绑定自定义插件
4. 触发特定的Hook
5. 处理options.

我们可以看到第1和第5步都是处理options。那到底有啥不同呢？

- `new WebpackOptionsDefaulter().process(options)`：WebpackOptionsDefaulter 顾名思义，是设置 webpack 的默认参数的地方，比如说默认入口路径，默认 rule, 默认 optimize 策略。这行的作用就是设置默认参数，并将用户自定义参数覆盖上去。
- `new WebpackOptionsApply().process(options, compiler)`：WebpackOptionsApply 的主要功能是根据 options 中的配置，注册各种内部插件如 SingleEntryPlugin，以及负责解析的各类钩子，以及负责优化的 SplitChunksPlugin 等等。

可以注意到在创建了 Compiler 对象过后，Webpack 就开始注册我们配置中的每一个插件了，这是因为再往后 Webpack 工作过程的生命周期就要开始了，所以必须先注册，这样才能确保插件中的每一个钩子都能被命中。

完成 Compiler 对象的创建过后，紧接着这里的代码开始判断配置选项中是否启用了监视模式，如果是监视模式就调用 Compiler 对象的 watch 方法，以监视模式启动构建，如果不是监视模式就调用 Compiler 对象的 run 方法，开始构建整个应用。

### 四、开始创建

这个 run 方法定义在 Compiler 类型中，具体文件在 webpack 模块下的 lib/Compiler.js 中，方法内部就是先触发了beforeRun 和 run 两个钩子，然后最关键的是调用了当前对象的 compile 方法，真正开始编译整个项目，具体实现如下：

<img src="/assets/webpack-runtime/08.png" />

compile 方法内部主要就是创建了一个 Compilation 对象，Compilation 字面意思是“合集”，实际上可以理解为一次构建过程中的上下文对象，里面包含了这次构建中全部的资源和信息。

<img src="/assets/webpack-runtime/09.png" />

创建完 Compilation 对象过后，紧接着触发了一个叫作 make 的钩子，进入整个构建过程最核心的 make 阶段。
至此 Compiler 作为 Webpack 事件派发模式中的委派者，其整体流程的分析已经结束，其流程图以及各个阶段触发的hooks如下图所示：

<img src="/assets/webpack-runtime/10.png" />

### 五、make（构建）阶段

make 阶段主体的目标就是：根据 entry 配置找到入口模块，开始依次递归出所有依赖，形成依赖关系树，然后将递归到的每个模块交给不同的 Loader 处理。

<img src="/assets/webpack-runtime/11.png" />

这个阶段的调用过程并不像之前一样，直接调用某个对象的某个方法，而是采用事件触发机制，让外部监听这个 make 事件的地方开始执行，首先执行的是入口插件，Webpack 中入口插件实际有三个：

- SingleEntryPlugin: 单入口打包，当 options.entry 的数据类型为 String 或 Object 时使用
- MultiEntryPlugin: 多入口打包，当 options.entry 的数据类型为 Array 时使用
- DynamicEntryPlugin: 动态打包，当 options.entry 的数据类型为 Function 时使用

因为我们默认使用的就是单一入口打包的方式，所以这里接下来会执行其中的 SingleEntryPlugin，

<img src="/assets/webpack-runtime/12.png" />

这个插件中调用了 Compilation 对象的 addEntry 方法，开始解析我们源代码中的入口文件:

```js
addEntry(context, entry, name, callback) {
  // 外部监听 addEntry 事件的地方开始执行
  this.hooks.addEntry.call(entry, name);

  // 将入口模块添加到模块依赖列表中
  const slot = {
    name: name,
    // TODO webpack 5 remove `request`
    request: null,
    module: null
  };

  if (entry instanceof ModuleDependency) {
    slot.request = entry.request;
  }

  // TODO webpack 5: merge modules instead when multiple entry modules are supported
  const idx = this._preparedEntrypoints.findIndex(slot => slot.name === name);
  if (idx >= 0) {
    // Overwrite existing entrypoint
    this._preparedEntrypoints[idx] = slot;
  } else {
    this._preparedEntrypoints.push(slot);
  }
  this._addModuleChain(
    context,
    entry,
    module => {
      this.entries.push(module);
    },
    (err, module) => {
      ...
    }
  )
  ...
}
```
我们发现 addEntry 方法的主要逻辑就是将当前 entry 添加到缓存并调用了 _addModuleChain 方法，我们来查看 _addModuleChain 方法的源码：

```js
_addModuleChain(context, dependency, onModule, callback) {
  const start = this.profile && Date.now();
  const currentProfile = this.profile && {};

  const errorAndCallback = this.bail
    ? err => {
        callback(err);
      }
    : err => {
        err.dependencies = [dependency];
        this.errors.push(err);
        callback();
      };

  if (
    typeof dependency !== "object" ||
    dependency === null ||
    !dependency.constructor
  ) {
    throw new Error("Parameter 'dependency' must be a Dependency");
  }
  const Dep = /** @type {DepConstructor} */ (dependency.constructor);
  const moduleFactory = this.dependencyFactories.get(Dep);
  if (!moduleFactory) {
    throw new Error(
      `No dependency factory available for this dependency type: ${dependency.constructor.name}`
    );
  }

  this.semaphore.acquire(() => {
    moduleFactory.create(
      {
        contextInfo: {
          issuer: "",
          compiler: this.compiler.name
        },
        context: context,
        dependencies: [dependency]
      },
      (err, module) => {
        if (err) {
          this.semaphore.release();
          return errorAndCallback(new EntryModuleNotFoundError(err));
        }

        let afterFactory;

        if (currentProfile) {
          afterFactory = Date.now();
          currentProfile.factory = afterFactory - start;
        }

        const addModuleResult = this.addModule(module);
        module = addModuleResult.module;

        onModule(module);

        dependency.module = module;
        module.addReason(null, dependency);

        const afterBuild = () => {
          if (addModuleResult.dependencies) {
            this.processModuleDependencies(module, err => {
              if (err) return callback(err);
              callback(null, module);
            });
          } else {
            return callback(null, module);
          }
        };

        if (addModuleResult.issuer) {
          if (currentProfile) {
            module.profile = currentProfile;
          }
        }

        if (addModuleResult.build) {
          this.buildModule(module, false, null, null, err => {
            if (err) {
              this.semaphore.release();
              return errorAndCallback(err);
            }

            if (currentProfile) {
              const afterBuilding = Date.now();
              currentProfile.building = afterBuilding - afterFactory;
            }

            this.semaphore.release();
            afterBuild();
          });
        } else {
          this.semaphore.release();
          this.waitForBuildingFinished(module, afterBuild);
        }
      }
    )
  })
}
```
_addModuleChain 方法入参中 context 是当前项目的绝对路径， dependency 是 entry 对象，onModule 和 callback 都是 addEntry 方法传入的回调，都接收module对象。我们知道webpack的核心功能就是通过入口找到所有依赖的模块，最后编译成一个或多个bundle。那么 _addModuleChain 就是这样一个遍历依赖，构建 module 依赖图的过程。semaphone 是 Nodejs 中最受欢迎的信号量模块，解决多线程的问题。在compilation 的构造函数中可以看到这样一行代码

```js
this.semaphore = new Semaphore(options.parallelism || 100);
```
由此我们可以知道compilation允许并发解析多入口，如果 opitons 中没有设置，默认为并发100。
除此之外我们还看到代码中有一个 moduleFactory。那它是干嘛的呢？实际上 SingleEntryPlugin 的第二个重要功能就是把entry依赖对应的模块工厂类型存到了 compilation.dependencyFactories 这个 map 中。 moduleFactory 就是当前依赖的工厂构造函数。那么剩余的逻辑从字面上就很好理解了。

1. 获取当前 entry 对应的模块工厂构造器
2. 调用工厂函数的 create 方法创建 module
3. 构建解析 this.buildModule(module)
4. afterBuild -> processModuleDependencies, 处理当前 module 依赖

在 buildModule 前打上断点，查看当前module的值:

<img src="/assets/webpack-runtime/13.jpg" />

我们可以看到里面很多属性都是 undefined/null，尤其是 _source。这说明，在 create 之后，当前创建出的 module 是还没有加载相应资源的。那我们可以推测，buildModule 就是加载并解析资源的过程。继续看代码，可以发现 buildModule 中直接调用了 module.build:

```js
buildModule(module, optional, origin, dependencies, thisCallback) {
  ...
  this.hooks.buildModule.call(module);
  module.build(
    this.options,
    this,
    this.resolverFactory.get("normal", module.resolveOptions),
    this.inputFileSystem,
    error => {
      ...
    }
  )
}
```
那么剩下的资源加载，解析过程都是交给了 NormalModule.js 完成的。

```js
// NormalModule.js
build(options, compilation, resolver, fs, callback) {
  ...
  return this.doBuild(options, compilation, resolver, fs, err => {...})
}

doBuild(options, compilation, resolver, fs, callback) {
  const loaderContext = this.createLoaderContext(
    resolver,
    options,
    compilation,
    fs
  );

  runLoaders(
    {
      resource: this.resource,
      loaders: this.loaders,
      context: loaderContext,
      readResource: fs.readFile.bind(fs)
    },
    (err, result) => {
      ...
      this._source = this.createSource(
        this.binary ? asBuffer(source) : asString(source),
        resourceBuffer,
        sourceMap
      );
      this._ast =
        typeof extraInfo === "object" &&
        extraInfo !== null &&
        extraInfo.webpackAST !== undefined
          ? extraInfo.webpackAST
          : null;
      return callback();
    }
  );
}
```
doBuild 方法中又调用了 runLoaders 方法，并在参数中通过将 fs.readFile.bind(fs) 将 loader 资源文件传入。在runLoaders的回调中，可以看到用 createSource 给this._source 赋值。证明我们之前的推测是正确的，buildModule 方法中执行具体的 Loader，处理特殊资源加载。

当 buildModule 方法执行具体的 Loader 之后我们再看 build 方法中给 doBuild 方法传入的回调函数：

```js
build(options, compilation, resolver, fs, callback) {
  ...
  return this.doBuild(options, compilation, resolver, fs, err => {
    ...
    try {
      const result = this.parser.parse(
        this._ast || this._source.source(),
        {
          current: this,
          module: this,
          compilation: compilation,
          options: options
        },
        (err, result) => {
          if (err) {
            handleParseError(err);
          } else {
            handleParseResult(result);
          }
        }
      );
      if (result !== undefined) {
        // parse is sync
        handleParseResult(result);
      }
    } catch (e) {
      handleParseError(e);
    }
  })
}
```
我们发现这儿调用了 parser 的 parse 方法，我们接下来查看其具体实现：

```js
// Parser.js
static parse(code, options) {
  ...
  const parserOptions = Object.assign(
    Object.create(null),
    defaultParserOptions,
    options
  );
  ...
  try {
    ast = acorn.parse(code, parserOptions);
    threw = false;
  } catch (e) {
    threw = true;
  }
  if (threw) {
    throw error;
  }

	return ast;
}
```
我们发现在 parse 方法中调用 acorn 库解析经 loader 处理后的源文件生成抽象语法树 AST 语法树。
当当前依赖 build 完成后 _addModuleChain 方法会接着执行 afterBuild 方法：

```js
// Compilation.js
_addModuleChain(context, dependency, onModule, callback) {
  ...
  const afterBuild = () => {
    ...
    if (addModuleResult.dependencies) {
      this.processModuleDependencies(module, err => {
        if (err) return callback(err);
        callback(null, module);
      });
    } else {
      return callback(null, module);
    }
  };
  ...
  if (addModuleResult.build) {
    this.buildModule(module, false, null, null, err => {
      if (err) {
        this.semaphore.release();
        return errorAndCallback(err);
      }
      ...
      this.semaphore.release();
      afterBuild();
    });
  } else {
    this.semaphore.release();
    this.waitForBuildingFinished(module, afterBuild);
  }
}

processModuleDependencies(module, callback) {
  const addDependency = dep => {
    const resourceIdent = dep.getResourceIdentifier();
    if (resourceIdent) {
      const factory = this.dependencyFactories.get(dep.constructor);
      if (factory === undefined) {
        throw new Error(
          `No module factory available for dependency type: ${
            dep.constructor.name
          }`
        );
      }
      let innerMap = dependencies.get(factory);
      if (innerMap === undefined) {
        dependencies.set(factory, (innerMap = new Map()));
      }
      let list = innerMap.get(resourceIdent);
      if (list === undefined) innerMap.set(resourceIdent, (list = []));
      list.push(dep);
    }
  };
  const addDependenciesBlock = block => {
    if (block.dependencies) {
      iterationOfArrayCallback(block.dependencies, addDependency);
    }
    if (block.blocks) {
      iterationOfArrayCallback(block.blocks, addDependenciesBlock);
    }
    if (block.variables) {
      iterationBlockVariable(block.variables, addDependency);
    }
  };

  try {
    addDependenciesBlock(module);
  } catch (e) {
    callback(e);
  }
}

const iterationOfArrayCallback = (arr, fn) => {
	for (let index = 0; index < arr.length; index++) {
		fn(arr[index]);
	}
};
```
对于当前模块，或许存在着多个依赖模块。当前模块会开辟一个依赖模块的数组，在遍历 AST 时，将 require() 中的模块通过 addDependency() 添加到数组中。当前模块构建完成后，webpack 调用 processModuleDependencies 开始递归处理依赖的 module，接着就会重复之前的构建步骤。

**补充**
Module 是 webpack 构建的核心实体，也是所有 module 的 父类，它有几种不同子类：NormalModule , MultiModule , ContextModule , DelegatedModule 等。但这些核心实体都是在构建中都会去调用对应的构建方法，也就是 build() 。它还包括了从构建到输出的一系列的有关 module 生命周期的函数，我们通过 module 父类类图其子类类图(这里以 NormalModule 为例)来观察其真实形态：

<img src="/assets/webpack-runtime/15.png" />

可以看到无论是构建流程，处理依赖流程，包括后面的封装流程都是与 module 密切相关的。

### 六、打包输出

在所有模块及其依赖模块 build 完成后，webpack 会监听 seal 事件调用各插件对构建后的结果进行封装，要逐次对每个 module 和 chunk 进行整理，生成编译后的源码，合并，拆分，生成 hash 。 同时这是我们在开发时进行代码优化和功能添加的关键环节。

```js
seal(callback) {
  this.hooks.seal.call();
  for (const preparedEntrypoint of this._preparedEntrypoints) {
    // 整理每个Module和chunk，每个chunk对应一个输出文件。
    const module = preparedEntrypoint.module;
    const name = preparedEntrypoint.name;
    const chunk = this.addChunk(name);
    const entrypoint = new Entrypoint(name);
    entrypoint.setRuntimeChunk(chunk);
    entrypoint.addOrigin(null, name, preparedEntrypoint.request);
    this.namedChunkGroups.set(name, entrypoint);
    this.entrypoints.set(name, entrypoint);
    this.chunkGroups.push(entrypoint);

    GraphHelpers.connectChunkGroupAndChunk(entrypoint, chunk);
    GraphHelpers.connectChunkAndModule(chunk, module);

    chunk.entryModule = module;
    chunk.name = name;

    this.assignDepth(module);
  }
  ...
  this.hooks.beforeModuleAssets.call();
  this.createModuleAssets();
  if (this.hooks.shouldGenerateChunkAssets.call() !== false) {
    this.hooks.beforeChunkAssets.call();
    // 生成最终的assets
    this.createChunkAssets();
  }
}
```
在封装过程中，webpack 会调用 Compilation 中的 createChunkAssets 方法进行打包后代码的生成。 createChunkAssets 流程如下：

<img src="/assets/webpack-runtime/14.png" />

从上图可以看出通过判断是入口 js 还是需要异步加载的 js 来选择不同的模板对象进行封装，入口 js 会采用 webpack 事件流的 render 事件来触发 Template类 中的renderChunkModules() (异步加载的 js 会调用 chunkTemplate 中的 render 方法)。

```js
const template = chunk.hasRuntime() ? this.mainTemplate : this.chunkTemplate;
const manifest = template.getRenderManifest({
  chunk,
  hash: this.hash,
  fullHash: this.fullHash,
  outputOptions,
  moduleTemplates: this.moduleTemplates,
  dependencyTemplates: this.dependencyTemplates
}); 
for (const fileManifest of manifest) {
  const cacheName = fileManifest.identifier;
  const usedHash = fileManifest.hash;
  filenameTemplate = fileManifest.filenameTemplate;
  file = this.getPath(filenameTemplate, fileManifest.pathOptions);
  ...
  if (this.cache && this.cache[cacheName] && this.cache[cacheName].hash === usedHash) {
    source = this.cache[cacheName].source;
  } else {
    //  调用 Template 类的 render 方法生成代码
    source = fileManifest.render();
    // Ensure that source is a cached source to avoid additional cost because of repeated access
    if (!(source instanceof CachedSource)) {
      const cacheEntry = cachedSourceMap.get(source);
      if (cacheEntry) {
        source = cacheEntry;
      } else {
        const cachedSource = new CachedSource(source);
        cachedSourceMap.set(source, cachedSource);
        source = cachedSource;
      }
    }
    if (this.cache) {
      this.cache[cacheName] = {
        hash: usedHash,
        source
      };
    }
  }
  // assets文件生成
  this.assets[file] = source;
  chunk.files.push(file);
  this.hooks.chunkAsset.call(chunk, file);
  alreadyWrittenFiles.set(file, {
    hash: usedHash,
    source,
    chunk
  });
}
```

最后一步，webpack 调用 Compiler 中的 emitAssets() ，按照 output 中的配置项将文件输出到了对应的 path 中，从而 webpack 整个打包过程结束。要注意的是，若想对结果进行处理，则需要在 emit 触发后对自定义插件进行扩展。

### 七、总结

webpack 的整体打包流程主要还是依赖于 compilation 和 module 这两个对象，compiliation 对象负责协调整个构建过程， module 是 webpack 构建的核心实体，由这两者合作完成了整体项目的打包，由 tapable 控制各插件在 webpack 事件流上运行，最终成就了 Webpack 这目前使用最广泛的打包工具。

整个流程主要完成了**内容转换 + 资源合并**两种功能，实现上包含三个阶段：

#### 1、初始化阶段

- 初始化参数：从配置文件、 配置对象、Shell 参数中读取，与默认配置结合得出最终的参数
- 创建编译器对象：用上一步得到的参数创建 Compiler 对象
- 初始化编译环境：包括注入内置插件、注册各种模块工厂、初始化 RuleSet 集合、加载配置的插件等
- 开始编译：执行 compiler 对象的 run 方法
- 确定入口：根据配置中的 entry 找出所有的入口文件，调用 compilition.addEntry 将入口文件转换为 dependence 对象

<img src="/assets/webpack-runtime/16.jpg">

#### 2、构建阶段

- 编译模块(make)：根据 entry 对应的 dependence 创建 module 对象，调用 loader 将模块转译为标准 JS 内容，调用 JS 解释器将内容转换为 AST 对象，从中找出该模块依赖的模块，再递归本步骤直到所有入口依赖的文件都经过了本步骤的处理
- 完成模块编译：上一步递归处理所有能触达到的模块后，得到了每个模块被翻译后的内容以及它们之间的依赖关系图

<img src="/assets/webpack-runtime/17.jpg">

1. 调用 handleModuleCreate ，根据文件类型构建 module 子类
2. 调用 loader-runner 仓库的 runLoaders 转译 module 内容，通常是从各类资源类型转译为 JavaScript 文本
3. 调用 acorn 将 JS 文本解析为AST
4. 遍历 AST，触发各种钩子：在 HarmonyExportDependencyParserPlugin 插件监听 exportImportSpecifier 钩子，解读 JS 文本对应的资源依赖；调用 module 对象的 addDependency 将依赖对象加入到 module 依赖列表中
5. AST 遍历完毕后，调用 module.handleParseResult 处理模块依赖
6. 对于 module 新增的依赖，调用 handleModuleCreate ，控制流回到第一步
7. 所有依赖都解析完毕后，构建阶段结束

这个过程中数据流 module => ast => dependences => module ，先转 AST 再从 AST 找依赖。这就要求 loaders 处理完的最后结果必须是可以被 acorn 处理的标准 JavaScript 语法，比如说对于图片，需要从图像二进制转换成类似于 `export default "data:image/png;base64,xxx"` 这类 base64 格式或者 `export default "http://xxx"` 这类 url 格式。

compilation 按这个流程递归处理，逐步解析出每个模块的内容以及 module 依赖关系，后续就可以根据这些内容打包输出。

#### 3、生成阶段

- 输出资源(seal)：根据入口和模块之间的依赖关系，组装成一个个包含多个模块的 Chunk，再把每个 Chunk 转换成一个单独的文件加入到输出列表，这步是可以修改输出内容的最后机会
- 写入文件系统(emitAssets)：在确定好输出内容后，根据配置确定输出的路径和文件名，把文件内容写入到文件系统

<img src="/assets/webpack-runtime/18.jpeg">

1. 构建本次编译的 ChunkGraph 对象；
2. 遍历 compilation.modules 集合，将 module 按 entry/动态引入 的规则分配给不同的 Chunk 对象；
3. compilation.modules 集合遍历完毕后，得到完整的 chunks 集合对象，调用 createXxxAssets 方法
4. createXxxAssets 遍历 module/chunk ，调用 compilation.emitAssets 方法将 assets 信息记录到 compilation.assets 对象中
5. 触发 seal 回调，控制流回到 compiler 对象

这一步的关键逻辑是将 module 按规则组织成 chunks ，webpack 内置的 chunk 封装规则比较简单：

- entry 及 entry 触达到的模块，组合成一个 chunk
- 使用动态引入语句引入的模块，各自组合成一个 chunk

## 参考资料

[编写一个webpack的loader（1）](https://juejin.cn/post/6844903861451227150)

[webpack-loader简简单单配置入门](https://juejin.cn/post/6909459619324788750)

[Webpack 核心知识有哪些？](https://mp.weixin.qq.com/s/xT12rUsYOkypXS8YFYQEzQ)

[细说 webpack 之流程篇](https://developer.aliyun.com/article/61047)

[[万字总结] 一文吃透 Webpack 核心原理](https://zhuanlan.zhihu.com/p/363928061)