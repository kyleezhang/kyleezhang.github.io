---
title: TypeScript工程化实践笔记
date: 2021-01-06 23:41:06
toc: true
categories: 
- 前端
tags: 
- TypeScript
---

TypeScript 作为 JavaScript 的超集，由于其静态类型系统的引入，使得前端开发大型项目更容易实现团队内代码规范限制及接口数据约定，同时使得代码更容易阅读和理解，但是当我们在基于 TypeScript 实现大型项目的过程中同样要注意如下内容：
<!-- more -->

## 声明空间

在 TypeScript 里存在两种声明空间：类型声明空间与变量声明空间：

### 一、类型声明空间

类型声明空间包含用来当做类型注解的内容，例如下面的类型声明：

```ts
class Foo {}
interface Bar {}
type Bas = {};
```

你可以将 Foo, Bar, Bas 作为类型注解使用，示例如下：

```ts
let foo: Foo;
let bar: Bar;
let bas: Bas;
```

注意，尽管你定义了 interface Bar，却并不能够把它作为一个变量来使用，因为它没有定义在变量声明空间中。

```ts
interface Bar {}
const bar = Bar; // Error: "cannot find name 'Bar'"
```

出现错误提示提示： cannot find name 'Bar' 的原因是名称 Bar 并未定义在变量声明空间。这将带领我们进入下一个主题 -- 变量声明空间。

### 二、变量声明空间

变量声明空间包含可用作变量的内容，在上文中 Class Foo 提供了一个类型 Foo 到类型声明空间，此外它同样提供了一个变量 Foo 到变量声明空间，如下所示：

```ts
class Foo {}
const someVar = Foo;
const someOtherVar = 123;
```

因此当我们想把一个类来当做变量传递时这显得极为便捷，但是我们并不能把一些如 interface 定义的内容当作变量使用。与此相似，一些用 var 声明的变量，也只能在变量声明空间使用，不能用作类型注解。举个🌰:

```ts
const foo = 123;
let bar: foo; // ERROR: "cannot find name 'foo'"
```

提示 ERROR: "cannot find name 'foo'" 原因是，名称 foo 没有定义在类型声明空间里。

## 模块系统

### 一、全局模块

在默认情况下，如果一个目录下存在一个 tsconfig.json 文件，那么它意味着这个目录是当前 TypeScript 项目的根目录。默认情况下，你在当前项目目录下任何一处编写的 TS 代码总是对于根目录起的所有子目录可见（不论是变量声明空间亦或是类型声明空间都可见），当你开始在一个新的 TypeScript 文件中写下代码时，它处于全局命名空间中。如在 foo.ts 里的以下代码。

```ts
const foo = 123;
```

如果你在相同的项目里创建了一个新的文件 bar.ts，TypeScript 类型系统将会允许你使用变量 foo，就好像它在全局可用一样：

```ts
const bar = foo; // allowed
```

毋庸置疑，使用全局变量空间是危险的，因为它会与文件内的代码命名冲突。我们推荐使用下文中将要提到的文件模块。

### 二、文件模块

文件模块也被称为外部模块。如果在你的 TypeScript 文件的根级别位置含有 import 或者 export，那么它会在这个文件中创建一个本地的作用域。因此，我们需要把上文 foo.ts 改成如下方式（注意 export 用法）：

```ts
export const foo = 123;
```

在全局命名空间里，我们不再有 foo，这可以通过创建一个新文件 bar.ts 来证明：

```ts
const bar = foo; // ERROR: "cannot find name 'foo'"
```

如果你想在 bar.ts 里使用来自 foo.ts 的内容，你必须显式地导入它，更新后的 bar.ts 如下所示。

```ts
import { foo } from './foo';
const bar = foo; // allow
```

在 bar.ts 文件里使用 import 时，它不仅允许你使用从其他文件导入的内容，还会将此文件 bar.ts 标记为一个模块，文件内定义的声明也不会“污染”全局命名空间。
文件模块拥有强大的功能和较强的可用性，我们可以通过指定 tsconfig.json 中的 module 属性指定不同的模块系统，较为常用的是 ES Module 以及 Commonjs 模块：

#### 1、ES Module

（1）单独导出一个变量或类型

```ts
export const someVar = 123;
export type someType = {
  foo: string;
};
```

（2）批量导出

```ts
const someVar = 123;
type someType = {
  type: string;
};

export { someVar, someType };
```

（3）导出时重命名

```ts
const someVar = 123;
export { someVar as aDifferentName };
```

（4）默认导出

ES 模块中导出主要通过 export default 实现，主要有如下三种应用场景：

- 在一个变量之前（不需要使用 let/const/var）；
- 在一个函数之前（不需要函数名）；
- 在一个类之前。

```ts
// some var
export default (someVar = 123);

// some function
export default function someFunction() {}

// some class
export default class someClass {}
```

（5）导入一个变量或者是一个类型

```ts
import { someVar, someType } from './foo';
```

（6）通过重命名的方式导入变量或者类型

```ts
import { someVar as aDifferentName } from './foo';
```

（7）整体导入

```ts
import * as foo from './foo';
// 你可以使用 `foo.someVar` 和 `foo.someType` 以及其他任何从 `foo` 导出的变量或者类型
```

（8）只导入模块

```ts
import 'core-js'; // 一个普通的 polyfill 库
```

（9）从其他模块导入后整体导出

```ts
export * from './foo';
```

（10）从其他模块导入后，部分导出

```ts
export { someVar } from './foo';
```

（11）通过重命名，部分导出从另一个模块导入的项目

```ts
export { someVar as aDifferentName } from './foo';
```

（12）默认导入

```ts
import someLocalNameForThisFile from './foo';
```

#### 2、CommonJS

（1）部分导出

```ts
class someClass {};
exports.someClass = someClass
```

（2）部分导入

```ts
const test = require("./foo").someClass;
```

（3）整体导出

```ts
class Class1 {};
class Class2 {}
module.exports = { Class1, Class2 }
```

（4）整体导入

```ts
const test = require("./foo")
```

在 CommonJS 中，整体导出就是一个 module.exports ，exports 实际指向的也是 module.exports 指向的引用，而 require 方法实际上返回的是 module.exports 而不是 exports，因此如果对 exports 重新赋值，则断开了 exports 对 module.exports 的指向。

#### 3、CommonJS与ES Module之间的区别

- CommonJs是被加载的时候运行，ES Module是编译的时候运行
- CommonJs输出的是值的浅拷贝，ES Module输出值的引用
- CommentJs具有缓存。在第一次被加载时，会完整运行整个文件并输出一个对象，拷贝（浅拷贝）在内存中。下次加载文件时，直接从内存中取值

**CommonJS输出值拷贝**

```js
// a.js
let count = 0
exports.count = count; // 输出值的拷贝
exports.add = ()=>{
    //这里改变count值，并不会将module.exports对象的count属性值改变
    count++;
}

// b.js
const { count, add } = require('./a.js')
console.log(count) // 0
add();
console.log(count) // 0
```

**ES Module输出值引用**
```js
// a.js
export let count = 0; // 输出的是值的引用，指向同一块内存
export const add = ()=>{
    count++; // 此时引用指向的内存值发生改变
}


// b.js
import { count, add } from './a.js'
console.log(count) // 0
add();
console.log(count)// 1
```

**CommonJS输出浅拷贝验证**

```js
// a.js
const foo = {
	count: 0
}
// module.exports的foo属性为 foo 对象的浅拷贝，指向同一个内存中
exports.foo=foo;

window.setTimeout(()=>{
	foo.count += 1
	console.log('changed foo')
},1000)

// b.js
const  { foo }  = require('./a.js')
console.log('foo', foo); // 'foo',{count: 0}
window.setTimeout(()=>{
  console.log('after 2s foo', foo); //'after 2s foo ',{count: 1}
}, 2000)
```

其实上个🌰中的 `const { foo } = require('./a.js')` 或者 `const foo = require('./a.js').foo` 写法是相当危险的。因为 CommonJS 输出的值的拷贝，若后面在 a.js 中对 foo 的内存指向作出改动，则不能及时更新。举个🌰:

```js
// a.js
const foo = {
	count: 0
}
exports.foo=foo; // 此时 foo 指向 {count: 0} 的内存地址
window.setTimeout(()=>{
  //改变 foo 的内存指向
	exports.foo='haha';
},1000)

// b.js
const  { foo }  = require('./a.js'); // 拷贝了 foo 属性指向 {count: 0} 内存地址的引用
console.log('foo', foo); // 'foo',{count: 0}
window.setTimeout(()=>{
  //此处并没有改变
  console.log('after 2s foo', foo); // 'after 2s foo ',{count: 0}
}, 2000)
```

改进：

```js
// b.js
const test = require('./a.js'); 
// test 拷贝了 整个输出对象{foo:{count: 0}}内存地址的引用
// 当内存中的属性值发生变化时，可以拿到最新的值，因为指向的是同一片内存

console.log('foo', test.foo); //'foo',{count: 0}
window.setTimeout(()=>{
  // 保证获取到的是最新的
  console.log('after 2s foo', test.foo); // 'after 2s foo ','haha'
}, 2000)
```

进阶：

```js
// a.js
let foo = 1
setTimeout(()=>{
  foo = 2;
  exports.foo = foo
},1000)
exports.foo = foo

// b.js
var test = require('./b.js');
console.log(test.foo); // 1
setTimeout(()=>{
  console.log(test.foo); // 2
},2000)
```

但是需要注意由于 module.exports 实际上输出的是内存地址的引用，所以如果缓存的内存地址依然存在且无变更的话依然是之前的值，举个🌰:

```js
// a.js
let foo = 1
setTimeout(()=>{
  foo=2;
  module.exports={foo}; //注意：指向新内存 {foo:2}
},1000)
module.exports={foo}; //指向内存 {foo:1}

// b.js
var test =require('./a.js'); // 浅拷贝，指向的还是{foo:1}的内存，并缓存在内存中
//从缓存的内存中取值
console.log(test.foo); // 1

setTimeout(()=>{
  // 从缓存的内存取值
  console.log(test.foo) // 1
},2000)
```

#### 4、兼容性处理

我们在 tsconfig.json 中通过配置 target 和 module 属性可以实现不同版本、不同模块化方案代码的编译生成，需要注意的是如果 target 属性设为 ES3 或 ES5 时及时 module 指定为 es2015 也依然会编译成CommonJS 模块，举个🌰:

```ts
export const a: number = 1223;

export function getA() {
    return a;
}

const b: string = 'b';
const c: string = 'c';
export { b, c }

export default function () {
    console.log('6666');
}
```

编译后（target: "es5", module: "es2015"）：

```js
"use strict";
exports.__esModule = true;
exports.c = exports.b = exports.getA = exports.a = void 0;

exports.a = 1223;

function getA() {
    return exports.a;
}
exports.getA = getA;

var b = 'b';
exports.b = b;
var c = 'c';
exports.c = c;

function default_1() {
    console.log('6666');
}
exports["default"] = default_1;
```

CommonJS 本身不支持默认导出，所以 CommonJS 中转换 ES Module 默认导出的方案是给 exports 添加 default 属性， 因此默认导入也会相应的发生变更：

```ts
import * as All from './action';
console.log(All);

import { a } from './action';
console.log(a);

import { getA as get } from './action';
console.log(get);

import defaultFuc from './action';
console.log(defaultFuc);
```

编译后：

```js
"use strict";
exports.__esModule = true;

var All = require("./action");
console.log(All); // {__esModule: true, a: 1223, getA: [Function: getA], b: 'b', c: 'c', default: [Function: default_1]}

var action_1 = require("./action");
console.log(action_1.a); // 1223

var action_2 = require("./action");
console.log(action_2.getA); // [Function: getA]

var action_3 = require("./action");
console.log(action_3["default"]); // [Function: default_1]
```

因此 ES Module 中的默认导出向 CommonJS 转换对于我们开发者来说其实是无感知的，因为它的引入的时候会自动获取导出模块的 default 属性，因此我们可以在 CommonJS 模块中通过获取导出对象的 default 属性获取默认导出，举个🌰:

```ts
// a.ts
export default function () {
    console.log('default');
}
// b.ts
const a = require('./a');
a.default(); // "default"
```

但是 CommonJS 模块中通过 default 属性调用显得有些格格不入，而且很容易产生错误，那么如何解决两个模块系统之间的不兼容性问题呢？TypeScript 为了解决兼容性问题给我们提供了一些其他语法糖，举个🌰:

```ts
// a.ts
export = function () {
  console.log('I am module A');
}
// b.ts
import a = require('./a');
// import a from './a';
a(); // "I am module A"
```

`export = ***` 语法本身会被编译成 CommonJS 模块中的 module.exports 语法糖，所以在文件 a.ts 中不能再导入其他模块，否则会编译报错。在 b.ts 文件中我们可以通过 TypeScript 提供的 `import moduleName = require('***')` 语法导入，需要注意当 tsconfig.json 中的 esModuleInterop 属性如果为 true，这儿也可以通过 ES6 提供的普通导入方法导入，如果为 false 则只能通过 `import moduleName = require('***')` 语法导入。

## 命名空间

在 JavaScript 中命名空间能有效的避免全局污染，但是在 ES6 引入模块系统之后命名空间就很少被提及了，但是 TypeScript 中依然实现了这个特性，尽管在模块化系统中我们完全不用考虑命名空间了，但是如果使用了一些全局的类库，命名空间仍然是一个比较好的解决方案。

### 一、定义

TypeScript 中命名空间可以通过关键字 namespace 声明，命名空间内可以定义任意多变量，这些变量仅在命名空间内可见，如果想要部分成员在全局模块内可见的话则需要使用 export 关键字将其导出，举个🌰:

```ts
// a.ts
namespace Shape {
  const pi = Math.PI;
  export function circle(r: number) {
    return pi * r ** 2 
  }
}
// b.ts
Shape.circle(2); // 12.566370614359172
```

随着我们程序的不断扩张这个命名空间也可能会随着不断扩大，所以需要将命名空间进行进一步的拆分，TypeScript中将命名空间进行拆分极为简单，举个🌰:

```ts
// a.ts
namespace Shape {
  const pi = Math.PI;
  export function circle(r: number) {
    return pi * r ** 2 
  }
}
// b.ts
/// <reference path="a.ts" />
// a.ts的相对路径
namespace Shape {
  export function square(x: number) {
    return x ** 2
  }
}

Shape.circle(2);
Shape.square(2);
```

在文件 a.ts 和 b.ts 中都通过 namespace 关键字命名了同名的命名空间，那么这个命名空间就分布在了这两个文件中，它们之间是共享一个命名空间的，此处需要注意由于在 b.ts 文件中调用了 a.ts 文件中声明的 circle 方法，所以需要在 b.ts 文件中通过三斜线指令 `/// <reference path="a.ts" />` 来调用 a.ts 文件，此时我们编译成 JavaScript 代码并查看：

```js
// a.js
var Shape;
(function (Shape) {
    var pi = Math.PI;
    function circle(r) {
        return pi * Math.pow(r, 2);
    }
    Shape.circle = circle;
})(Shape || (Shape = {}));

// b.js
/// <reference path="a.ts" />
var Shape;
(function (Shape) {
    function square(x) {
        return Math.pow(x, 2);
    }
    Shape.square = square;
})(Shape || (Shape = {}));
Shape.square(2);
Shape.circle(2);
```

从编译后的代码中我们可以看到 TypeScript 中的命名空间本质上依然是通过 JavaScript 的立即执行匿名函数实现的，这在 JavaScript 确保创建的变量不会泄漏至全局命名空间时是极为常见的一种做法，接下来我们一起看一下 TypeScript 命名空间成员的别名。

### 二、成员别名

在上述示例中我们每次通过 Shape.circle 或 Shape.square 方法调用难免有些繁杂累赘，TypeScript 为我们提供了更为简便的调用，举个🌰:

```ts
namespace Shape {
  export function square(x: number) {
    return x ** 2
  }
}

import square = Shape.square;
square(2);
```

需要注意这里的 import 与模块系统中的 import 没有任何关系，通过这种方法将命名空间中的部分成员导入后我们就可以直接调用了，由于我们在导入的过程中还可以为导入变量或函数重命名，因此这种命名空间中导出变量或函数的导入方法的使用更为普遍与广泛。

### 三、命名空间与函数的合并

```ts
function lib () {}

namespace lib {
    export let version = '0.1.0'
}
```

在上述示例中我们通过命名空间与同名函数的合并实现了向函数 lib 中添加属性 version，这在 JavaScript 中是极为常见的，TypeScript 为我们提供了这种更为优雅的实现，我们查看其编译后的 JavaScript 代码是怎样的？

```js
"use strict";
function lib() { }
(function (lib) {
    lib.version = '0.1.0';
})(lib || (lib = {}));
```

与之类似命名空间还可以与类合并为其添加静态属性，与枚举类型合并为其添加方法，举个🌰:

```ts
class C {}
namespace C {
    export let version = '0.1.0'
}
console.log(C.version) // '0.1.0'

enum Color {
  Red,
  Yellow
}
namespace Color {
  export function mix() {}
}
console.log(Color.mix) // function mix() { } 
```

此处需要注意命名空间与类或与函数合并时，函数或类的定义必须在命名空间前面，否则会报错，而枚举类型与命名空间的定义之间的位置先后是没有要求的，

## 声明合并

TypeScript 中的声明合并实际上就是指编译器会把程序中的多个地方具有相同名称的多个声明合并为一个声明，这样做的好处在于可以把程序中散落各处的声明合并在一起，进而避免对程序声明的遗漏，举个🌰:

```ts
interface A {
  x: number;
}
interface A {
  y: number;
}

const a: A = {
  x: 12,
  y: 14
}
```

TypeScript 要求声明合并中非函数成员必须具有唯一性，即最好没有相同的属性名，如果两处声明具有相同的属性则必须类型一致，举个🌰:

```ts
interface A {
  x: number;
  y: string | number;
}
interface A {
  // Property 'y' must be of type 'string | number', but here has type 'number'.
  y: number;
}
```

那么函数成员会怎么处理呢？实际上 TypeScript 会将每个函数看作函数重载，举个🌰:

```ts
interface A {
  x: number;
  func(x: string): string;
}
interface A {
  y: number;
  func(x: number): number;
  func(x: number[]): number[];
}

const a: A = {
  x: 12,
  y: 14,
  func(x: any) {
      return x
  }
}
```

那么声明合并的时候函数重载列表的顺序是如何确定的呢？实际上这里主要会遵循两个原则：

1. 接口内部按照书写先后顺序确定
2. 不同接口间后面的会排在前面

因此上述示例中 func 属性的排列顺序如下所示：

```ts
interface A {
  x: number;
  func(x: string): string; // 5
  func(x: 'b'): string; // 2
}
interface A {
  y: number;
  func(x: number): number; // 3
  func(x: number[]): number[]; // 4
  func(x: 'a'): string; // 1
}
```

不过上面的原则也有一个例外，如果函数的参数为字符串的字面量的话那么这个声明就会被提升到整个函数声明的最顶端，举个🌰:

```ts
interface A {
  x: number;
  func(x: string): string; // 3
}
interface A {
  y: number;
  func(x: number): number; // 1
  func(x: number[]): number[]; // 2
}
```

## 声明文件

### 一、声明语法

#### 1、全局变量

使用 declare var 声明变量。 如果变量是只读的，那么可以使用 declare const。 你还可以使用 declare let 变量拥有块级作用域。

代码:

```js
console.log("Half the number of widgets is " + (foo / 2))
```

声明:

```js
declare var foo: number
```

一般来说，全局变量都是禁止修改的常量，所以大部分情况都应该使用 const 而不是 var 或 let。

#### 2、全局函数

使用 declare function 声明函数

代码:

```js
greet("hello, world")
```

声明:

```ts
declare function greet(greeting: string): void
```

在声明语句中，支持函数重载

```ts
declare function greet(greeting: string): void
declare function greet(greeting: number): void
```

#### 3、带属性的对象

使用 declare namespace 描述用点表示法访问的类型或值

代码:

```js
let result = myLib.makeGreeting("hello, world")
console.log("The computed greeting is:" + result)

let count = myLib.numberOfGreetings
```

声明:

```ts
declare namespace myLib {
    function makeGreeting(s: string): string
    let numberOfGreetings: number
}
```

#### 4、可重用类型接口或类型

除了全局变量之外，可能有一些类型我们也希望能暴露出来。在类型声明文件中，我们可以直接使用 interface 或 type 来声明一个全局的接口或类型。

代码:

```js
greet({
  greeting: "hello world",
  duration: 4000
})
```
声明:

```ts
interface GreetingSettings {
  greeting: string
  duration?: number
  color?: string
}

declare function greet(setting: GreetingSettings): void
```

type 与 interface 类似.

#### 5、组织类型

代码：

```js
const g = new Greeter("Hello")
g.log({ verbose: true })
g.alert({ modal: false, title: "Current Greeting" })
```

使用命名空间组织类型

```ts
declare namespace Greeter {
  interface LogOptions {
    verbose?: boolean
  }
  interface AlertOptions {
    modal: boolean
    title?: string
    color?: string
  }
}
```

也可以在一个声明中创建嵌套的命名空间

```ts
declare namespace Greeter.Options {
  // Refer to via Greeter.Options.Log
  interface Log {
    verbose?: boolean
  }
  interface Alert {
    modal: boolean
    title?: string
    color?: string
  }
}
```

#### 6、全局类

使用 declare class 描述一个类或像类一样的对象。 类可以有属性和方法，就和构造函数一样，但不能用来定义具体的实现。

代码:

```js
const myGreeter = new Greeter("hello, world")
myGreeter.greeting = "howdy"
myGreeter.showGreeting()

class SpecialGreeter extends Greeter {
  constructor() {
    super("Very special greetings")
  }
}
```

声明:

```ts
declare class Greeter {
  constructor(greeting: string)
  greeting: string
  showGreeting(): void
}
```

### 二、声明文件

在开发过程中不可避免的要引用其他第三方 JavaScript 库，但是在 Typescript 项目中直接引用 JavaScript 库无法通过 TypeScript 的严格类型检查机制，需要我们为其类型声明，即引入声明文件，举个🌰:

```ts
import $ from 'jquery'

// Error: Try `npm install @types/jquery` if it exists or add a new declaration (.d.ts) file containing `declare module 'jquery';`
```

这儿我们通过模块化方式引用 jquery 库，但是由于其是一个 JavaScript 库，所以 TypeScript 的类型检查机制提示类型声明文件不存在，jquery 库的声明文件是单独提供的，我们这儿安装一下就好：

```shell
npm install @types/jquery -D
```

安装完之后，就可以正常在 ts 文件中使用 jquery 了。但是部分类库没有提供声明文件的话就需要我们自己编写声明文件，那么声明文件该如何编写呢？


#### 1、编写声明文件

我们平时引用的第三方库主要由三种类型构成：

- 全局类库
- 模块类库
- UMD类库

全局类库是指能在全局命名空间下访问，不需要 import、require等指令导入，主要在 HTML 中通过 `<script>` 标签引入；而模块类库只能工作在模块加载器的环境下，遵循不同模块化方案的模块类库有着不同的引入和导出方式，而UMD模块既可以作为模块引入又可以作为全局使用。


**（1）全局类库**

编写 js 文件，如下所示：

```js
function globalLib (options) {
  console.log(options)
}

globalLib.version = '1.0.0'

globalLib.doSomething = function () {
  console.log('global lib do something')
}
```

上述代码中，定义了一个函数，为函数添加了两个元素。接下来我们用 script 标签引入该文件，让该函数作用在全局。

我们在 ts 中调用该函数，如下所示：

```ts
globalLib({ a: 1 }) // Error: Cannot find name 'globalLib'.
```

提示未找到该函数。解决办法为它添加一个声明文件，在同级目录下创建一个同名 d.ts 文件，如下所示：

```ts
declare function globalLib (options: globalLib.Options): void

declare namespace globalLib {
  const version: string
  function doSomething (): void
  interface Options {
    [key: string]: any
  }
}
```

定义了一个同名函数和命名空间，在上一篇声明合并中有介绍过函数与命名空间合并，相当于为函数添加了一些默认属性。函数参数定义了一个接口，参数指定为可索引类型，接受任意属性。declare 关键字可以为外部变量提供声明。

**（2）模块类库**

以下为 CommonJS 模块编写的文件：

```js
const version = '1.0.0'

function doSomething () {
  console.log('moduleLib do something')
}

function moduleLib (options) {
  console.log(options)
}

moduleLib.version = version
moduleLib.doSomething = doSomething

module.exports = moduleLib
```

同样我们将它引入 ts 文件中使用。

```ts
import module from './module-lib/index.js'
// Error: Could not find a declaration file for module './module-lib/index.js'. 
```

提示未找到该模块，同样我们需要为它编写文件声明。

```ts
declare function moduleLib (options: Options): void

interface Options {
  [key: string]: any
}

declare namespace moduleLib {
  const version: string
  function doSomething(): void
}

export = moduleLib
```

上述 ts 与刚刚编写的全局类库声明文件大致相同，唯一的区别这里需要 export 输出。

**（3）UMD类库**

```js
(function (root, factory) {
  if (typeof define === 'function' && define.amd) {
    define(factory)
  } else if (typeof module === 'object' && module.exports) {
    module.exports = factory()
  } else {
    root.umdLib = factory()
  }
}(this, function () {
  return {
    version: '1.0.0',
    doSomething () {
      console.log('umd lib do something')
    }
  }
}))
```

同样我们将它引入 ts 文件中使用，如果没有声明文件也会提示错误，我们直接看 ts 声明文件

```ts
declare namespace umdLib {
  const version: string
  function doSomething (): void
}

export as namespace umdLib

export = umdLib
```

我们声明了一个命名空间，命名空间内有两个成员 version 和 doSomething，分别对应 umd 中的两个成员。这里与其他类库不同的是，多添加了一条语句 export as namespace umdLib，如果为 umd 库声明，这条语句必不可少。

umd 同样可以使用全局方式引用。

#### 2、声明文件放哪儿

在介绍声明文件如何编写之前，列举一下一般声明文件存放的方式。

1. 目录 src/@types/，在 src 目录新建 @types 目录，在其中编写 .d.ts 声明文件，声明文件会自动被识别，可以在此为一些没有声明文件的模块编写自己的声明文件，实际上在 tsconfig.json 中 include 字段包含的范围内编写 .d.ts，都将被自动识别；
2. 与被声明的 js 文件同级目录内，创建相同名称的 .d.ts 文件，这样也会被自动识别，前面示例都采用了此方案；
3. 设置 package.json 中的 typings 属性值，如 ./index.d.ts. 这样系统会识别该地址的声明文件。同样当我们把自己的js库发布到 npm 上时，按照该方法绑定声明文件。
4. 通过 npm 模块安装，如 @type/react ，它将被存放在 node_modules/@types/ 路径下。

#### 3、如何发布声明文件

如果我们自己实现了一个js库，如何来写声明文件呢？目前有两种方式用来发布声明文件到 npm 上：

1. 与你的 npm 包同时捆绑在一起；
2. 发布到 npm 上的 [@types organization](https://www.npmjs.com/~types)

**（1）npm包含声明文件**

在 package.json 中，你的需要指定 npm 包的主 js 文件，那么你还需要指定主声明文件。如下：

```js
{
  "name": "owl-redux",
  "version": "0.0.1",
  "description": "A simple version of redux",
  "main": "dist/owl-redux.js",
  "typings": "index.d.ts"
}
```

有的 npm 包设置的 types 属性，它和 typings 具有相同意义。

**（2）发布到@types**

@types 下面的包是从 [DefinitelyTyped](https://github.com/DefinitelyTyped/DefinitelyTyped) 里自动发布的，通过 types-publisher 工具。 如果想让你的包发布为@types包，只需要提交一个pull request到 https://github.com/DefinitelyTyped/DefinitelyTyped。

#### 4、插件

有时候，我们想给一个第三方类库添加一些自定义的方法。以下介绍如何在模块插件或全局插件中添加自定义方法。

**（1）模块插件**

我们使用 moment 插件，为它添加一个自定义方法。 关键字 declare module。

```ts
import m from 'moment'
declare module 'moment' {
  export function myFunction (): void
}
m.myFunction = () => { console.log('I am a Fn') }
```

**（2）全局插件**

在上面我们有介绍全局类库，我们为它添加一个自定义方法。关键字 declare global。

```ts
declare global {
  namespace globalLib {
    function doAnyting (): void
  }
}
globalLib.doAnyting = () => {}
```




## 参考资料
极客时间《TypeScript开发实战》专栏
《深入理解TypeScript》
[CommonJs 和 ESModule 的 区别整理](https://juejin.cn/post/6844903598480965646)
[TypeScript开发教程](https://www.dengwb.com/typescript/)
