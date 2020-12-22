---
title: 深入浅出模块化
date: 2020-11-02 20:18:05
toc: true
categories: 
- 前端
tags:
- node
- javascript
---

JavaScript自诞生以来很长一段时间大部分开发者都没有将其当成一门真正的编程语言，认为其只不过是一种又小又简单的网页脚本，没必要将其模块化，但是随着前端领域的快速发展，模块化成为了前端开发大型应用所需的一项基本能力，但是相较于Java、Python等其他高级语言，JavaScript直到ES6才真正意义上拥有语言级的模块语法，前端模块化从早期的最原始实现到后面模块化规范的提出，经历了很多阶段，但是这整个过程中模块化的思想值得我们所有人思考。
## 模块化的早期演进
早期的前端技术标准根本没有预料到前端行业会有今天这个规模，所以在设计上存在很多缺陷，导致我们现在去实现前端模块化时会遇到诸多问题。虽然说，如今绝大部分问题都已经被一些标准或者工具解决了，但是从整体演进过程来看前端方向落实模块化主要有如下的几个代表阶段：
### 一、文件划分阶段
最早我们会基于文件划分的方式实现模块化，也就是 Web 最原始的模块系统。具体做法是将每个功能及其相关状态数据各自单独放到不同的 JS 文件中，约定每个文件是一个独立的模块。使用某个模块将这个模块引入到页面中，一个 script 标签对应一个模块，然后直接调用模块中的成员（变量 / 函数）。
```
└─ stage-1

    ├── module-a.js

    ├── module-b.js

    └── index.html
```
```javascript
// module-a.js 
function foo () {
   console.log('moduleA#foo') 
}
```
```javascript
// module-b.js 
var data = 'something'
```
```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <title>Stage 1</title>
</head>
<body>
  <script src="module-a.js"></script>
  <script src="module-b.js"></script>
  <script>
    // 直接使用全局成员
    foo() // 可能存在命名冲突
    console.log(data)
    data = 'other' // 数据可能会被修改
  </script>
</body>
</html>
```
缺点：
- 模块直接在全局工作，大量模块成员污染全局作用域；
- 没有私有空间，所有模块内的成员都可以在模块外部被访问或者修改；
- 一旦模块增多，容易产生命名冲突；
- 无法管理模块与模块之间的依赖关系；
- 在维护的过程中也很难分辨每个成员所属的模块；

总之，这种原始“模块化”的实现方式完全依靠约定实现，一旦项目规模变大，这种约定就会暴露出种种问题，非常不可靠，所以我们需要尽可能解决这个过程中暴露出来的问题。

### 二、命名空间方式
后来，我们约定每个模块只暴露一个全局对象，所有模块成员都挂载到这个全局对象中，具体做法是在第一阶段的基础上，通过将每个模块“包裹”为一个全局对象的形式实现，这种方式就好像是为模块内的成员添加了“命名空间”，所以我们又称之为命名空间方式。
```typescript
// module-a.js
window.moduleA = {
  method1: function () {
    console.log('moduleA#method1')
  }
}
```
```typescript
// module-b.js
window.moduleB = {
  data: 'something'
  method1: function () {
    console.log('moduleB#method1')
  }
}
```
```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <title>Stage 2</title>
</head>
<body>
  <script src="module-a.js"></script>
  <script src="module-b.js"></script>
  <script>
    moduleA.method1()
    moduleB.method1()
    // 模块成员依然可以被修改
    moduleA.data = 'foo'
  </script>
</body>
</html>
```
这种命名空间的方式只是解决了命名冲突的问题，但是其它问题依旧存在。

### 三、IIFE
使用立即执行函数表达式（IIFE，Immediately-Invoked Function Expression）为模块提供私有空间。具体做法是将每个模块成员都放在一个立即执行函数所形成的私有作用域中，对于需要暴露给外部的成员，通过挂到全局对象上的方式实现。
```javascript
// module-a.js
;(function () {
  var name = 'module-a'
  function method1 () {
    console.log(name + '#method1')
  }
  window.moduleA = {
    method1: method1
  }
})()
```
```javascript
// module-b.js
;(function () {
  var name = 'module-b'
  function method1 () {
    console.log(name + '#method1')
  }
  window.moduleB = {
    method1: method1
  }
})()
```
这种方式带来了私有成员的概念，私有成员只能在模块成员内通过闭包的形式访问，这就解决了前面所提到的全局作用域污染和命名冲突的问题。
### 四、 IIFE 依赖参数
在 IIFE 的基础之上，我们还可以利用 IIFE 参数作为依赖声明使用，这使得每一个模块之间的依赖关系变得更加明显。
```javascript
// module-a.js
;(function ($) { // 通过参数明显表明这个模块的依赖
  var name = 'module-a'
  function method1 () {
    console.log(name + '#method1')
    $('body').animate({ margin: '200px' })
  }
  window.moduleA = {
    method1: method1
  }
})(jQuery)
```
除了模块加载的问题以外，目前这几种通过约定实现模块化的方式，不同的开发者在实施的过程中会出现一些细微的差别，因此，为了统一不同开发者、不同项目之间的差异，我们就需要制定一个行业标准去规范模块化的实现方式。提到模块化规范，大家可能会首先想到 CommonJS 规范，它是 Node.js 中所遵循的模块规范，该规范约定，一个文件就是一个模块，每个模块都有单独的作用域，通过 module.exports 导出成员，再通过 require 函数载入模块。
## CommonJS模块规范
CommonJS对模块的定义十分简单，主要分为**模块引用**、**模块定义**和**模块标识**3个部分。
1、 模块引用
在CommonJS规范中，存在require()方法，这个方法接受模块标识，以此引入一个模块的API到当前上下文中。
<!-- more -->
2、 模块定义
CommonJS规范提供了exports对象用于导出当前模块的方法或者变量，并且它是上下文唯一导出的出口。在模块中，还存在一个module对象，它代表模块自身，而exports是module的属性。在Node中，一个文件就是一个模块，将方法挂载在exports对象上作为属性即可定义导出的方式：
```javascript
// math.js￼   
exports.add = function () {￼
    var sum = 0,￼         
    i = 0,￼           
    args = arguments,￼         
    l = args.length;￼       
    while (i < l) {￼          
        sum += args[i++];￼       
    }￼        
    return sum;￼      
};
```
3、 模块标识
模块标识其实就是传递给require()方法的参数，它必须是符合小驼峰命名的字符串，或者以．、.．开头的相对路径，或者绝对路径。它可以没有文件名后缀．js。
CommonJS规范中模块的定义十分简单，接口也十分简洁。它的意义在于将类聚的方法和变量等限定在私有的作用域中，同时支持引入和导出功能以顺畅地连接上下游依赖，每个文件就是一个模块，有自己的作用域。在一个文件里面定义的变量、函数、类，都是私有的，对其他文件不可见，因此用户完全不必考虑变量污染，同时用户可以在模块内引入其他模块内引入其他模块，方便了模块之间依赖关系的管理。

## Node中模块的实现
Node在实现中并非完全按照规范实现，而是对模块规范进行了一定的取舍，同时也增加了少许自身需要的特性，在Node中引入模块，需要经历如下3个步骤。
1、 路径分析
2、 文件定位
3、 编译执行
在Node中，模块分为两类：一类是Node提供的模块，称为核心模块；另一类是用户编写的模块，称为文件模块。
> 核心模块部分在Node源代码的编译过程中，编译进了二进制执行文件。在Node进程启动时，部分核心模块就被直接加载进内存中，所以这部分核心模块引入时，文件定位和编译执行这两个步骤可以省略掉，并且在路径分析中优先判断，所以它的加载速度是最快的。
> 文件模块则是在运行时动态加载，需要完整的路径分析、文件定位、编译执行过程，速度比核心模块慢。

接下来，我们展开详细的模块加载过程：

### 1、优先从缓存加载
与前端浏览器会缓存静态脚本文件以提高性能一样，Node对引入过的模块都会进行缓存，以减少二次引入时的开销。不同的地方在于，浏览器仅仅缓存文件，而Node缓存的是编译和执行之后的对象。不论是核心模块还是文件模块，require()方法对相同模块的二次加载都一律采用缓存优先的方式，这是第一优先级的。不同之处在于核心模块的缓存检查先于文件模块的缓存检查。

### 2、模块标识符分析
require()方法接受一个标识符作为参数。在Node实现中，正是基于这样一个标识符进行模块查找的。模块标识符在Node中主要分为以下几类：
- 核心模块，如http、fs、path等。
- .或..开始的相对路径文件模块。
- 以/开始的绝对路径文件模块。
- 非路径形式的文件模块，如自定义的connect模块。

核心模块的优先级仅次于缓存加载，它在Node的源代码编译过程中已经编译为二进制代码，其加载过程最快，同时由于核心模块在模块标识符分析的过程中判断优先级最高，因此如果试图加载一个与核心模块标识符相同的自定义模块，那是不会成功的。如果自己编写了一个http用户模块，想要加载成功，必须选择一个不同的标识符或者换用路径的方式。
以.或..开始的相对路径标识符和/开始的绝对路径标识符在分析过程中都会被当做文件模块来处理。在分析文件模块时，require()方法会将路径转为真实路径，并以真实路径作为索引，将编译执行后的结果存放到缓存中，以使二次加载时更快。由于文件模块给Node指明了确切的文件位置，所以在查找过程中可以节约大量时间，其加载速度仅慢于核心模块。
自定义模块指的是非核心模块，也不是路径形式的标识符。它是一种特殊的文件模块，可能是一个文件或者包的形式。这类模块的查找是最费时的，也是所有方式中最慢的一种。Node在自定义模块定位具体文件时制定的查找策略称为**模块路径**，具体表现为一个路径组成的数组，我们可以通过在模块内打印module.paths属性查看，模块路径的生成规则如下所示：
❑ 当前文件目录下的node_modules目录。
❑ 父目录下的node_modules目录。
❑ 父目录的父目录下的node_modules目录。
❑ 沿路径向上逐级递归，直到根目录下的node_modules目录。
它的生成方式与JavaScript的原型链或作用域链的查找方式十分类似。在加载的过程中，Node会逐个尝试模块路径中的路径，直到找到目标文件为止。可以看出，当前文件的路径越深，模块查找耗时会越多，这是自定义模块的加载速度是最慢的原因。

### 3、文件定位
从缓存加载的优化策略使得二次引入时不需要路径分析、文件定位和编译执行的过程，大大提高了再次加载模块时的效率。
但在文件的定位过程中，还有一些细节需要注意，这主要包括文件扩展名的分析、目录和包的处理

#### 文件扩展名分析
equire()在分析标识符的过程中，会出现标识符中不包含文件扩展名的情况。CommonJS模块规范也允许在标识符中不包含文件扩展名，这种情况下，Node会按．js、.json、.node的次序补足扩展名，依次尝试，在尝试的过程中，需要调用fs模块同步阻塞式地判断文件是否存在。因为Node是单线程的，所以这里是一个会引起性能问题的地方。小诀窍是：如果是．node和．json文件，在传递给require()的标识符中带上扩展名，会加快一点速度。另一个诀窍是：同步配合缓存，可以大幅度缓解Node单线程中阻塞式调用的缺陷。

#### 目录分析和包的处理
在分析标识符的过程中，require()通过分析文件扩展名之后，可能没有查找到对应文件，但却得到一个目录，这在引入自定义模块和逐个模块路径进行查找时经常会出现，此时Node会将目录当做一个包来处理。
在这个过程中，Node对CommonJS包规范进行了一定程度的支持。首先，Node在当前目录下查找package.json（CommonJS包规范定义的包描述文件），通过JSON.parse()解析出包描述对象，从中取出main属性指定的文件名进行定位。如果文件名缺少扩展名，将会进入扩展名分析的步骤。而如果main属性指定的文件名错误，或者压根没有package.json文件，Node会将index当做默认文件名，然后依次查找index.js、index.json、index.node。如果在目录分析的过程中没有定位成功任何文件，则自定义模块进入下一个模块路径进行查找。如果模块路径数组都被遍历完毕，依然没有查找到目标文件，则会抛出查找失败的异常。

### 4、模块编译
在Node中，每个文件模块都是一个对象，它的定义如下：
```javascript
function Module(id, parent) {
    this.id = id;
    this.exports = {};
    this.parent = parent;

    if (parent && parent.children) {
        parent.children.push(this);
    }

    this.filename = null;
    this.loaded = false;
    this.children = [];
}
```
编译和执行是引入文件模块的最后一个阶段。定位到具体的文件后，Node会新建一个模块对象，然后根据路径载入并编译。对于不同的文件扩展名，其载入方法也有所不同，具体如下所示:
> .js文件: 通过fs模块同步读取文件后编译执行
> .node文件: 这是用C/C++编写的扩展文件，通过dlopen()方法加载最后编译生成的文件
> .json文件: 通过fs模块同步读取文件后，用JSON.parse()解析并返回结果
> 其余扩展文件: 都被当作.js文件载入

每一个编译成功的模块都会将其文件路径作为索引缓存在Module._cache对象上，以提高二次引入的性能。
根据不同的文件扩展名，Node会调用不同的读取方式，如．json文件的调用如下
```javascript
// Native extension for .json￼       
Module._extensions['.json'] = function(module, filename) {￼         
    var content = NativeModule.require('fs').readFileSync(filename, 'utf8');￼
    try {￼       
        module.exports = JSON.parse(stripBOM(content));￼
    } catch (err) {￼
        err.message = filename + ': ' + err.message;￼
        throw err;￼
    }￼
};
```
其中，Module._extensions会被赋值给require()的extensions属性，所以通过在代码中访问require.extensions可以知道系统中的已有的扩展加载方式，如果想对自定义的扩展名进行特殊的加载，可以通过类似require.extensions['.ext']的方式实现。早期的CoffeeScript文件就是通过添加require.extensions['.coffee']扩展的方式来实现加载的。

在确定文件的扩展名之后，Node将调用具体的编译方式来将文件执行后返回给调用者。

（一）**JavaScript模块的编译**
回到CommonJS模块规范，我们知道每个模块文件中存在着require、exports、module这3个变量，但是它们在模块文件中并没有定义，那么从何而来呢？甚至在Node的API文档中，我们知道每个模块中还有__filename、__dirname这两个变量的存在，它们又是从何而来的呢？如果我们把直接定义模块的过程放诸在浏览器端，会存在污染全局变量的情况。
事实上，在编译的过程中，Node对获取的JavaScript文件内容进行了头尾包装，一个正常的JavaScript文件会被包装成如下的样子：
```javascript
(function (exports, require, module, __filename, __dirname) {
    var math = require('math');
    exports.area = function (radius) {
        return Math.PI * radius * radius;
    };
})
```
这样每个模块文件之间都进行了作用域隔离。包装之后的代码会通过vm原生模块的runInThisContext()方法执行，返回一个具体的function对象。最后，将当前模块对象的exports属性、require()方法、module（模块对象自身），以及在文件定位中得到的完整文件路径和文件目录作为参数传递给这个function()执行，这就是这些变量并没有定义在每个模块文件中却存在的原因。在执行之后，模块的exports属性被返回给了调用方。exports属性上的任何方法和属性都可以被外部调用到，但是模块中的其余变量或属性则不可直接被调用。

（二）**C/C++模块的编译**
Node调用process.dlopen()方法进行加载和执行。在Node的架构下，dlopen()方法在Windows、*nix等平台下分别有不同的实现，通过libuv兼容层进行了封装。实际上，.node的模块文件并不需要编译，因为它是编写C/C++模块之后编译生成的，所以这里只有加载和执行的过程。在执行的过程中，模块的exports对象与．node模块产生联系，然后返回给调用者。C/C++模块给Node使用者带来的优势主要是执行效率，劣势则是C/C++模块的编写门槛比JavaScript高。

（三）**JSON文件的编译**
JSON文件的编译是3种编译方式中最简单的。Node利用fs模块同步读取JSON文件的内容之后，调用JSON.parse()方法得到对象，然后将它赋给模块对象的exports，以供外部调用。
JSON文件在用作项目的配置文件时比较有用。如果你定义了一个JSON文件作为配置，那就不必调用fs模块去异步读取和解析，直接调用require()引入即可。此外，你还可以享受到模块缓存的便利，并且二次引入时也没有性能影响。

## 核心模块
Node中的核心模块分为C/C++编写的和JavaScript编写的两部分，其中C/C++文件存放在Node项目的src目录下，JavaScript文件存放在lib目录下，虽然它们在编译成可执行文件的过程中都被编译成为了二进制文件，但实际上两者的编译过程之间有很大的不同：
### 一、JavaScript核心模块的编译
在编译所有核心模块之前编译程序首先要将所有的JavaScript模块文件编译为C/C++代码。
#### 1、转存为C/C++代码
Node采用了V8附带的js2c.py工具，将所有内置的JavaScript代码（src/node.js和lib/*.js）转换成C++里的数组，生成node_natives.h头文件。在这个过程中，Javascript代码以字符串的形式存储在node命名空间中，是不可直接执行的，在启动Node进程时，JavaScript代码直接加载进内存中，然后当模块被加载时Javascript核心模块经历标识符分析后直接定位到内存中，比普通文件模块从磁盘中一处一处查找要快得多。
#### 2、编译JS核心模块
lib目录下的所有模块文件也没有定义require、module、exports这些变量。在引入JavaScript核心模块的过程中，也经历了头尾包装的过程，然后才执行和导出了exports对象。与文件模块有区别的地方在于：获取源代码的方式（核心模块是从内存中加载的）以及缓存执行结果的位置。
JavaScript核心模块的定义如下面的代码所示，源文件通过process.binding('natives')取出并放置到NativeModule._source中，编译成功的模块缓存到NativeModule._cache对象上，文件模块则缓存到Module._cache对象上：
```c++
function NativeModule(id) {
    this.filename = id + '.js';
    this.id = id;
    this.exports = {};
    this.loaded = false;
}
NativeModule._source = process.binding('natives');
NativeModule._cache = {};
```

### 二、C/C++核心模块的编译
在核心模块中，有些模块全部由C/C++编写，有些模块则由C/C++完成核心部分，其他部分则由JavaScript实现包装或向外导出，以满足性能需求。后面这种C++模块主内完成核心，JavaScript主外实现封装的模式是Node能够提高性能的常见方式。通常，脚本语言的开发速度优于静态语言，但是其性能则弱于静态语言。而Node的这种复合模式可以在开发速度和性能之间找到平衡点。我们将那些由C/C++编写的部分统一称为**内建模块**，因为它们通常不被用户直接调用，Node的buffer、crypto、evals、fs、os等模块都是部分通过C/C++编写的。
#### 1、内建模块的组织形式
在Node中内建模块的内部结构定义如下：
```C++
struct node_module_struct {
    int version;
    void *dso_handle;
    const char *filename;
    void (*register_func)(v8::Handle<v8::Object> target);
    const char *modname;
}
```
每一个内建模块在定义之后，都会通过NODE_MODULE宏将模块定义到node命名空间中，模块的具体初始化方法挂载为结构的register_func成员：
```C++
#define NODE_MODULE(modname, regfunc)
extern "C" {
    NODE_MODULE_EXPORT node::node_module_struct modname ## _module = {
        NODE_STANDARD_MODULE_STUFF,
        regfunc,
        NODE_STRINGIFY(modname)
    }; 
}
```
node_extensions.h文件将这些散列的内建模块统一放进了一个叫node_module_list的数组中，这些模块有：
❑ node_buffer
❑ node_crypto
❑ node_evals
❑ node_fs
❑ node_http_parser
❑ node_os
❑ node_zlib
❑ node_timer_wrap
❑ node_tcp_wrap
❑ node_udp_wrap
❑ node_pipe_wrap
❑ node_cares_wrap
❑ node_tty_wrap
❑ node_process_wrap
❑ node_fs_event_wrap
❑ node_signal_watcher
些内建模块的取出也十分简单。Node提供了get_builtin_module()方法从node_module_list数组中取出这些模块。
内建模块的优势在于：首先，它们本身由C/C++编写，性能上优于脚本语言；其次，在进行文件编译时，它们被编译进二进制文件。一旦Node开始执行，它们被直接加载进内存中，无须再次做标识符定位、文件定位、编译等过程，直接就可执行。
#### 2、内建模块的导出
Node在启动时，会生成一个全局变量process，并提供Binding方法来协助加载内建模块，Binding方法首先会创建一个exports空对象，然后调用get_builtin_module()方法取出内建模块对象，通过执行register_func()填充exports对象，最后将exports对象按模块名缓存，并返回给调用方完成导出。
这个方法不仅可以导出内建方法，还能导出一些别的内容。前面提到的JavaScript核心文件被转换为C/C++数组存储后，便是通过process.binding('natives')取出放置在NativeModule._source中的，该方法将通过js2c.py工具转换出的字符串数组取出，然后重新转换为普通字符串，以对JavaScript核心模块进行编译和执行。

### 三、核心模块的引入流程
<img src="/assets/javascript-module/01.png" width="430">

如图所示的os原生模块的引入流程可以看到，为了符合CommonJS模块规范，从JavaScript到C/C++的过程是相当复杂的，它要经历C/C++层面的内建模块定义、（JavaScript）核心模块的定义和引入以及（JavaScript）文件模块层面的引入。但是对于用户而言，require()十分简洁、友好。

### 四、核心模块编写
核心模块中的JavaScript部分几乎与文件模块的开发相同，遵循CommonJS模块规范，上下文中除了拥有require、module、exports外，还可以调用Node中的一些全局变量，而内建模块的编写通常分两步完成：编写头文件和编写C/C++文件。
1、将以下代码保存为node_hello.h，存放到Node的src目录下：
```C++
 #ifndef NODE_HELLO_H_￼
 #define NODE_HELLO_H_￼
 #include <v8.h>￼

 namespace node {￼
    // 预定义方法￼
    v8::Handle<v8::Value> SayHello(const v8::Arguments& args);￼
}￼
#endif
```
2、编写node_hello.cc并存储到src目录下：
```C++
#include <node.h>￼
#include <node_hello.h>￼
#include <v8.h>￼
namespace node {￼
    using namespace v8;￼
    // 实现预定义的方法￼
    Handle<Value> SayHello(const Arguments& args) {￼
        HandleScope scope;￼
        return scope.Close(String::New("Hello world! "));￼
    }￼
    // 给传入的目标对象添加sayHello方法￼
    void Init_Hello(Handle<Object> target) {￼
        target->Set(String::NewSymbol("sayHello"), FunctionTemplate::New(SayHello)->GetFunction());￼
    }￼
}￼
// 调用NODE_MODULE()将注册方法定义到内存中￼
NODE_MODULE(node_hello, node::Init_Hello)
```
以上两步完成了内建模块的编写，但是真正要让Node认为它是内建模块，还需要更改src/node_extensions.h，在NODE_EXT_LIST_END前添加NODE_EXT_LIST_ITEM(node_hello)，以将node_hello模块添加进node_module_list数组中。
其次，还需要让编写的两份代码编译进执行文件，同时需要更改Node的项目生成文件node.gyp，并在’target_name': 'node’节点的sources中添加上新编写的两个文件。然后编译整个Node项目。

## C/C++扩展模块
JavaScript的一个典型弱点就是位运算。JavaScript的位运算参照Java的位运算实现，但是Java位运算是在int型数字的基础上进行的，而JavaScript中只有double型的数据类型，在进行位运算的过程中，需要将double型转换为int型，然后再进行。所以，在JavaScript层面上做位运算的效率不高。
在应用中，会频繁出现位运算的需求，包括转码、编码等过程，如果通过JavaScript来实现，CPU资源将会耗费很多，这时编写C/C++扩展模块来提升性能的机会来了。
C/C++扩展模块属于文件模块中的一类，其通过预先编译为.node文件，然后调用process.dlopen()方法加载执行，虽然这里引用加载时是．node文件。其实．node的扩展名只是为了看起来更自然一点，不会因为平台差异产生不同的感觉。实际上，在Windows下它是一个．dll文件，在*nix下则是一个．so文件。为了实现跨平台，dlopen()方法在内部实现时区分了平台，分别用的是加载．so和．dll的方式，详细过程如下图所示：
<img src="/assets/javascript-module/02.png" width="552">

因此需要注意一个平台下的．node文件在另一个平台下是无法加载执行的，必须重新用各自平台下的编译器编译为正确的．node文件。

### 一、C/C++扩展模块的编写
普通的扩展模块与内建模块的区别在于无须将源代码编译进Node，而是通过dlopen()方法动态加载。所以在编写普通的扩展模块时，无须将源代码写进node命名空间，也不需要提供头文件，我们编写hello.cc并存储到src目录下，相关代码如下：
```C++
#include <node.h>￼
#include <v8.h>￼
using namespace v8;￼
// 实现预定义的方法￼
Handle<Value> SayHello(const Arguments& args) {￼
    HandleScope scope;￼
    return scope.Close(String::New("Hello world! "));￼
}￼
// 给传入的目标对象添加sayHello()方法￼
void Init_Hello(Handle<Object> target) {￼
    target->Set(String::NewSymbol("sayHello"), FunctionTemplate::New(SayHello)->GetFunction());￼
}￼
// 调用NODE_MODULE()方法将注册方法定义到内存中￼
NODE_MODULE(hello, Init_Hello)
```
C/C++扩展模块与内建模块的套路一样，将方法挂载在target对象上，然后通过NODE_MODULE声明即可。
由于不像编写内建模块那样将对象声明到node_module_list链表中，所以无法被认作是一个原生模块，只能通过dlopen()来动态加载，然后导出给JavaScript调用。

### 二、C/C++扩展模块的编译
C/C++扩展模块的编译主要通过GYP(Generate Your Project)工具的帮助，GYP的好处主要在于可以帮助你生成各个平台下的项目文件，比如Windows下的Visual Studio解决方案文件（.sln）、Mac下的XCode项目配置文件以及Scons工具。在这个基础上，再动用各自平台下的编译器编译项目。这大大减少了跨平台模块在项目组织上的精力投入。为此，Nathan Rajlich基于GYP为Node提供了一个专有的扩展构建工具node-gyp，这个工具通过`npm install -g node-gyp`这个命令即可安装，node-gyp的编译过程会根据平台不同，分别通过make或vcbuild进行编译。编译完成后，hello.node文件会生成在build/Release目录下。

### 三、C/C++扩展模块的加载
得到hello.node结果文件后，如何调用扩展模块其实在前面已经提及。require()方法通过解析标识符、路径分析、文件定位，然后加载执行即可。对于以．node为扩展名的文件，Node将会调用process.dlopen()方法去加载文件：
```javascript
// Native extension for .node
Module._extensions['.node'] = process.dlopen;
```
但是实际上.node文件的引入过程并不是简简单单的调用process.dlopen方法而已，而是经历了四个层次上的调用：
<img src="/assets/javascript-module/03.png" width="442" />
加载．node文件实际上经历了两个步骤，第一个步骤是调用uv_dlopen()方法去打开动态链接库，第二个步骤是调用uv_dlsym()方法找到动态链接库中通过NODE_MODULE宏定义的方法地址。这两个过程都是通过libuv库进行封装的：在*nix平台下实际上调用的是dlfcn.h头文件中定义的dlopen()和dlsym()两个方法；在Windows平台则是通过LoadLibraryExW()和GetProcAddress()这两个方法实现的，它们分别加载．so和．dll文件（实际为．node文件）。
这里对libuv函数的调用充分表现Node利用libuv实现跨平台的方式，这样的情景在很多地方还会出现。
由于编写模块时通过NODE_MODULE将模块定义为node_module_struct结构，所以在获取函数地址之后，将它映射为node_module_struct结构几乎是无缝对接的。接下来的过程就是将传入的exports对象作为实参运行，将C++中定义的方法挂载在exports对象上，然后调用者就可以轻松调用了。
C/C++扩展模块与JavaScript模块的区别在于加载之后不需要编译，直接执行之后就可以被外部调用了，其加载速度比JavaScript模块略快。
使用C/C++扩展模块的一个好处在于可以更加灵活和动态地加载它们，保持Node模块自身简单性的同时，给予Node无限的可扩展性。

参考资料：
《深入浅出Node.js》
《Node学习指南》（第2版）