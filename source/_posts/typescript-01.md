---
title: TypeScript知识点总结
date: 2020-11-15 16:30:03
toc: true
tags: 
- typescript
---
TypeScript是一种由微软开发的自由和开源的编程语言。它是JavaScript的一个超集，为JavaScript引入了可选的静态类型，相比于JavaScript它的特点主要有以下三点：
- 类型检查：使我们可以在编译阶段发现问题而不是运行时
- 语言扩展：TypeScript不仅仅包括了ES6及未来提案中的一些特性，还从其他语言借鉴了一些特性，比如接口和抽象类
- 工具属性：TypeScript会编译生成JavaScript运行在浏览器及不同操作系统上，无其他运行时开销

那么为什么我们需要TypeScript帮我们引入静态类型呢？
<!-- more -->
静态类型语言可以在编译阶段确定所有变量的类型，而动态类型语言只能在执行阶段确定所有变量的类型，我们以相同功能的Javascript和C++代码为例：
```javascript
// javascript
class C {
    constructor(x, y) {
        this.x = x;
        this.y = y;
    }
}
function add(a, b) {
    return a.x + a.y + b.x + b.y 
}
```
```C++
// C++
class C {
    public:
        int x;
        int y;
}
int add(C a, C b) {
    return a.x + a.y + b.x + b.y
}
```
在上述示例中a、b都是类C的实例对象，add函数返回a、b对象所有属性的和，对于JavaScript而言，属性结构图如下图所示：
<img src="/assets/typescript-01/01.png" width="520">
- 在程序运行时动态计算属性偏移量
- 需要额外的空间存储属性名
- 所有对象的编译量信息各存一份
而对于C++而言：
<img src="/assets/typescript-01/02.png" width="520">
- 编译阶段确定属性偏移量
- 偏移量访问代替属性名访问
- 偏移量信息共享

由此可以看到动态类型语言无论在时间还是空间上都有比较多的性能损耗，实际上虽然V8为了提升JavaScript运行时的性能做了很多优化，但是TypeScript更重要的价值在于将静态类型的编程思维引入了javaScript，让我们可以在编译阶段以一种完全不同的视角去看待我们的代码。

## 基本类型&语法
### 一、`Boolean`类型
```typescript
let bool: boolean = true
```
### 二、`Number`类型
```typescript
let num: number = 123
```
### 三、`String`类型
```typescript
let str: string = 'abc'
```
### 四、`Symbol`类型
```typescript
let s1 = Symbol();
let s2: symbol = Symbol()
```
### 五、`Array`类型
```typescript
let list: number[] = [1, 2, 3];
let list: Array<number> = [1, 2, 3]; // Array<number>泛型语法
```
### 六、`Enum`类型
枚举类型：一组有名字的常量集合，使用枚举可以清晰地表达意图或创建一组有区别的用例。 TypeScript支持数字的和基于字符串的枚举。
#### 1、数字枚举
```typescript
enum Direction {
  NORTH, // 0
  SOUTH, // 1
  EAST, // 2
  WEST, // 3
}
let dir: Direction = Direction.NORTH;
```
默认情况下，NORTH的初始值为0，其余的成员会从1开始自动增长。换句话说，Direction.SOUTH的值为1，Direction.EAST的值为2，Direction.WEST的值为3。
以上的枚举示例经编译后，对应的ES5代码如下：
```javascript
"use strict";
var Direction;
(function (Direction) {
  Direction[(Direction["NORTH"] = 0)] = "NORTH";
  Direction[(Direction["SOUTH"] = 1)] = "SOUTH";
  Direction[(Direction["EAST"] = 2)] = "EAST";
  Direction[(Direction["WEST"] = 3)] = "WEST";
})(Direction || (Direction = {}));
var dir = Direction.NORTH;
```
从上面代码我们可以看到TypeScript通过反向映射实现了数字枚举，但是其他枚举却又有所不同。
#### 2、字符串枚举
```typescript
enum Direction {
  NORTH = "NORTH",
  SOUTH = "SOUTH",
  EAST = "EAST",
  WEST = "WEST",
}
```
对应的ES5代码如下：
```javascript
"use strict";
var Direction;
(function (Direction) {
    Direction["NORTH"] = "NORTH";
    Direction["SOUTH"] = "SOUTH";
    Direction["EAST"] = "EAST";
    Direction["WEST"] = "WEST";
})(Direction || (Direction = {}));
```
从上面示例中我们也可以发现字符串枚举没有实现反向映射。
#### 3、异构映射
异构枚举的成员值是数字和字符串的混合：
```typescript
enum Enum {
  A,
  B,
  C = "C",
  D = "D",
  E = 8,
  F,
}
```
对应的ES5代码如下：
```javascript
"use strict";
var Enum;
(function (Enum) {
    Enum[Enum["A"] = 0] = "A";
    Enum[Enum["B"] = 1] = "B";
    Enum["C"] = "C";
    Enum["D"] = "D";
    Enum[Enum["E"] = 8] = "E";
    Enum[Enum["F"] = 9] = "F";
})(Enum || (Enum = {}));
```
从示例我们可以发现异构枚举中的数字成员实现了反向映射，而字符串成员没有，但是异构枚举容易造成混淆，不推荐使用。
#### 4、常量枚举
除了数字枚举和字符串枚举之外，还有一种特殊的枚举——常量枚举。它是使用const关键字修饰的枚举，常量枚举会使用内联语法，不会为枚举类型编译生成任何JavaScript，举个🌰
```typescript
const enum Direction {
  NORTH,
  SOUTH,
  EAST,
  WEST,
}

let dir: Direction = Direction.NORTH;
```
对应的ES5代码如下：
```javascript
"use strict";
var dir = 0 /* NORTH */;
```
#### 5、枚举成员
枚举成员的值具有如下特性：
- 只读：枚举类型初始化以后不支持属性的修改，即枚举类型成员的属性都是只读属性
- 类型：枚举类型成员的值包括两种类型：**常量类型(const number)**和**计算类型(computer number)**，常量类型包括没有初始值、引用已有枚举属性值和常量表达式三类，常量类型会在编译阶段编译出结果，已常量的形式出现在运行时环境，计算类型主要是一些非常量的表达式，这些表达式的值不会在编译阶段被计算而是保留到执行时阶段。

我们举个🌰说明：
```typescript
enum Char {
  // const number
  a,
  b = Char.a,
  c = 1 + 3,
  // computer number
  d = Math.random(),
  e = '1234'.length
}
```
对应的ES5的代码如下：
```javascript
var Char;
(function (Char) {
    // const number
    Char[Char["a"] = 0] = "a";
    Char[Char["b"] = 0] = "b";
    Char[Char["c"] = 4] = "c";
    // computer number
    Char[Char["d"] = Math.random()] = "d";
    Char[Char["e"] = '1234'.length] = "e";
})(Char || (Char = {}));
```
### 七、Any类型
在TypeScript中，任何类型都可以被归为any类型。这让any类型成为了类型系统的顶级类型（也被称作全局超级类型）。
```typescript
let notSure: any = 666;
notSure = "semlinker";
notSure = false;
```
any 类型本质上是类型系统的一个逃逸舱。作为开发者，这给了我们很大的自由：TypeScript 允许我们对 any 类型的值执行任何操作，而无需事先执行任何形式的检查。比如:
```typescript
let value: any;

value.foo.bar; // OK
value.trim(); // OK
value(); // OK
new value(); // OK
value[0][1]; // OK
```
在许多场景下，这太宽松了。使用any类型，可以很容易地编写类型正确但在运行时有问题的代码。如果我们使用any类型，就无法使用TypeScript提供的大量的保护机制。为了解决any带来的问题,TypeScript 3.0 引入了unknown类型。
### 八、unknown类型
就像所有类型都可以赋值给any，所有类型也都可以赋值给unknown。这使得unknown成为TypeScript类型系统的另一种顶级类型。下面我们来看一下unknown类型的使用示例：
```typescript
let value: unknown;

value = true; // OK
value = 42; // OK
value = "Hello World"; // OK
value = []; // OK
value = {}; // OK
value = Math.random; // OK
value = null; // OK
value = undefined; // OK
value = new TypeError(); // OK
value = Symbol("type"); // OK
```
对value变量的所有赋值都被认为是类型正确的。但是，当我们尝试将类型为unknown的值赋值给其他类型的变量时会发生什么？
```typescript
let value: unknown;

let value1: unknown = value; // OK
let value2: any = value; // OK
let value3: boolean = value; // Error
let value4: number = value; // Error
let value5: string = value; // Error
let value6: object = value; // Error
let value7: any[] = value; // Error
let value8: Function = value; // Error
```
unknown类型只能被赋值给any类型和unknown类型本身。直观地说，这是有道理的：只有能够保存任意类型值的容器才能保存unknown类型的值。毕竟我们不知道变量value中存储了什么类型的值。
现在让我们看看当我们尝试对类型为unknown的值执行操作时会发生什么。以下是我们在之前any章节看过的相同操作：
```typescript
let value: unknown;

value.foo.bar; // Error
value.trim(); // Error
value(); // Error
new value(); // Error
value[0][1]; // Error
```
将value变量类型设置为unknown后，这些操作都不再被认为是类型正确的。通过将any类型改变为unknown类型，我们已将允许所有更改的默认设置，更改为禁止任何更改。
### 九、`Tuple`类型
众所周知，数组一般由同种类型的值组成，但有时我们需要在单个变量中存储不同类型的值，这时候我们就可以使用元组。在JavaScript中是没有元组的，元组是TypeScript中特有的类型，其工作方式类似于数组。
元组可用于定义具有有限数量的未命名属性的类型。每个属性都有一个关联的类型。使用元组时，必须提供每个属性的值，举个🌰：

```typescript
let tupleType: [string, boolean];
tupleType = ["semlinker", true];

console.log(tupleType[0]) // 'semlinker'
```
在上述示例中我们定义了一个名为tupleType的变量，它的类型是一个类型数组[string, boolean]，然后我们按照正确的类型依次初始化tupleType变量，并通过下标来访问其中元素，需要注意的是初始化元组时不仅仅需要保证每个属性类型的一致，同时必须提供每个属性的值，否则都会报错。

(未完，待续)

参考资料：
极客时间《TypeScript开发实战》专栏
[一份不可多得的 TS 学习指南（1.8W字](https://juejin.im/post/6872111128135073806#heading-21)