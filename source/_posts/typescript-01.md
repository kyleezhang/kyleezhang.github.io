---
title: TypeScript学习笔记
date: 2020-11-15 16:30:03
toc: true
categories: 
- 前端
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
### 一、Boolean类型
```typescript
let bool: boolean = true
```
### 二、Number类型
```typescript
let num: number = 123
```
### 三、String类型
```typescript
let str: string = 'abc'
```
### 四、Symbol类型
```typescript
let s1 = Symbol();
let s2: symbol = Symbol()
```
### 五、Array类型
```typescript
let list: number[] = [1, 2, 3];
let list: Array<number> = [1, 2, 3]; // Array<number>泛型语法
```
### 六、Enum类型
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
<!-- more -->
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
### 九、Tuple类型
众所周知，数组一般由同种类型的值组成，但有时我们需要在单个变量中存储不同类型的值，这时候我们就可以使用元组。在JavaScript中是没有元组的，元组是TypeScript中特有的类型，其工作方式类似于数组。
元组可用于定义具有有限数量的未命名属性的类型。每个属性都有一个关联的类型。使用元组时，必须提供每个属性的值，举个🌰：

```typescript
let tupleType: [string, boolean];
tupleType = ["semlinker", true];

console.log(tupleType[0]) // 'semlinker'
```
在上述示例中我们定义了一个名为tupleType的变量，它的类型是一个类型数组[string, boolean]，然后我们按照正确的类型依次初始化tupleType变量，并通过下标来访问其中元素，需要注意的是初始化元组时不仅仅需要保证每个属性类型的一致，同时必须提供每个属性的值，否则都会报错。
### 十、Void类型
在JavaScript中void是关键字，最关键的用途是获取undefined：
```javascript
void 0; // undefined
```
我们通过它可以有效避免undefined常量被重新赋值的情形，但在TypeScript中void类型表示没有任何返回：
```typescript
function func(): void {
  console.log("This is a function");
}
```
需要注意的是，声明一个void类型的变量没有什么作用，因为在严格模式下，它的值只能为undefined：
```typescript
let unusable: void = undefined;
```
### 十一、Null和Undefined类型
TypeScript里，undefined和null两者有各自的类型分别为undefined和null。
```typescript
let u: undefined = undefined;
let n: null = null;
```
需要注意的是在TypeScript规范中undefined和null是所有类型的自类型，所以当我们将tsconfig.json中的strictNullChecks属性设为false时我们可以将null和undefined1赋值给其他类型的值，举个🌰
```typescript
let num: number = 123;
num = null;
num = undefined;
```
### 十二、Bigint类型
BigInt里ECMAScript的一项提案，它在理论上允许我们建模任意大小的整数。TypeScript 3.2可以为BigInit进行类型检查，并支持在目标为esnext时输出BigInit字面量。为支持BigInt，TypeScript引入了一个新的原始类型bigint。可以通过调用BigInt()函数或书写BigInt字面量（在整型数字字面量末尾添加n）来获取bigint:
```typescript
let foo: bigint = BigInt(100); // the BigInt function
let bar: bigint = 100n;        // a BigInt literal
// *Slaps roof of fibonacci function*
// This bad boy returns ints that can get *so* big!
function fibonacci(n: bigint) {
    let result = 1n;
    for (let last = 0n, i = 0n; i < n; i++) {
        const current = result;
        result += last;
        last = current;
    }
    return result;
}
fibonacci(10000n)
```
需要注意的是bigint和number之间无法混用，是完全不同的东西：
```typescript
declare let foo: number;
declare let bar: bigint;
foo = bar; // error: Type 'bigint' is not assignable to type 'number'.
bar = foo; // error: Type 'number' is not assignable to type 'bigint'.
```
还有一点要注意的是，对bigint使用typeof操作符返回一个新的字符串："bigint"。因此，TypeScript能够正确地使用typeof细化类型，举个🌰
```typescript
function whatKindOfNumberIsIt(x: number | bigint) {
    if (typeof x === "bigint") {
        console.log("'x' is a bigint!");
    }
    else {
        console.log("'x' is a floating-point number");
    }
}
```
### 十三、Never类型
never类型表示的是那些永不存在的值的类型。 例如，never类型是那些总是会抛出异常或根本就不会有返回值的函数表达式或箭头函数表达式的返回值类型。
```typescript
// 返回never的函数必须存在无法达到的终点
function error(message: string): never {
  throw new Error(message);
}

function infiniteLoop(): never {
  while (true) {}
}
```
在TypeScript中，可以利用never类型的特性来实现全面性检查，具体示例如下：
```typescript
type Foo = string | number;

function controlFlowAnalysisWithNever(foo: Foo) {
  if (typeof foo === "string") {
    // 这里 foo 被收窄为 string 类型
  } else if (typeof foo === "number") {
    // 这里 foo 被收窄为 number 类型
  } else {
    // foo 在这里是 never
    const check: never = foo;
  }
}
```
注意在else分支里面，我们把收窄为never的foo赋值给一个显式声明的never变量。如果一切逻辑正确，那么这里应该能够编译通过。但是假如后来有一天你的同事修改了Foo的类型：
```typescript
type Foo = string | number | boolean;
```
然而他忘记同时修改controlFlowAnalysisWithNever方法中的控制流程，这时候else分支的foo类型会被收窄为boolean类型，导致无法赋值给never类型，这时就会产生一个编译错误。通过这个方式，我们可以确保controlFlowAnalysisWithNever方法总是穷尽了Foo的所有可能类型。 通过这个示例，我们可以得出一个结论：使用never避免出现新增了联合类型没有对应的实现，目的就是写出类型绝对安全的代码。

### 十四、object、Object和{}类型
#### 1、object类型
TypeScript 2.2引入了被称为object类型的新类型，它用于表示非原始类型，在JavaScript中以下类型被视为原始类型：string、boolean、number、bigint、symbol、null和undefined。它的引入主要是因为随着TypeScript 2.2的发布，标准库的类型声明已经更新，例如Object.create()和Object.setPrototypeOf()方法都需要为它们的原型参数指定object | null类型：
```typescript
// node_modules/typescript/lib/lib.es5.d.ts
interface ObjectConstructor {
  create(o: object | null): any;
  setPrototypeOf(o: any, proto: object | null): any;
  // ...
}
```
将原始类型作为原型传递给Object.setPrototypeOf()或Object.create()将导致在运行时抛出类型错误。TypeScript现在能够捕获这些错误，并在编译时提示相应的错误：
```typescript
const proto = {};

Object.create(proto);     // OK
Object.create(null);      // OK
Object.create(undefined); // Error
Object.create(1337);      // Error
Object.create(true);      // Error
Object.create("oops");    // Error
```
object类型的另一个用例是作为ES2015的一部分引入的WeakMap数据结构。它的键必须是对象，不能是原始值。这个要求现在反映在类型定义中：
```typescript
interface WeakMap<K extends object, V> {
  delete(key: K): boolean;
  get(key: K): V | undefined;
  has(key: K): boolean;
  set(key: K, value: V): this;
}
```
#### 2、Object类型
TypeScript定义了另一个与新的object类型几乎同名的类型，那就是Object类型。该类型是所有Object类的实例的类型。它由以下两个接口来定义：
- Object接口定义了Object.prototype原型对象上的属性
- ObjectConstructor接口定义了Object类的属性

（一）Object接口定义
```typescript
// node_modules/typescript/lib/lib.es5.d.ts
interface Object {
  constructor: Function;
  toString(): string;
  toLocaleString(): string;
  valueOf(): Object;
  hasOwnProperty(v: PropertyKey): boolean;
  isPrototypeOf(v: Object): boolean;
  propertyIsEnumerable(v: PropertyKey): boolean;
}
```
（二）ObjectConstructor接口定义
```typescript
// node_modules/typescript/lib/lib.es5.d.ts
interface ObjectConstructor {
  /** Invocation via `new` */
  new(value?: any): Object;
  /** Invocation via function calls */
  (value?: any): any;

  readonly prototype: Object;

  getPrototypeOf(o: any): any;

  // ···
}

declare var Object: ObjectConstructor;
```
Object类的所有实例都继承了Object接口中的所有属性。我们可以看到，如果我们创建一个返回其参数的函数：
```typescript
function f(x: Object): { toString(): string } {
  return x; // OK
}
```
当我们传入一个Object对象的实例时，它总是会满足该函数的返回类型 —— 即要求返回对象包含一个toString()方法。

有趣的是，类型Object包括原始值：
```typescript
function func1(x: Object) { }
func1('semlinker'); // OK
```
实际上这是因为Object.prototype的属性也可以通过原始值访问：
```typescript
'semlinker'.hasOwnProperty === Object.prototype.hasOwnProperty // true
```
相反，object类型不包括原始值：
```typescript
function func2(x: object) { }

// Argument of type '"semlinker"' 
// is not assignable to parameter of type 'object'.(2345)
func2('semlinker'); // Error
```
需要注意的是，当对Object类型的变量进行赋值时，如果值对象属性名与Object接口中的属性冲突，则TypeScript编译器会提示相应的错误：
```typescript
// Type '() => number' is not assignable to type 
// '() => string'.
// Type 'number' is not assignable to type 'string'.
const obj1: Object = { 
   toString() { return 123 } // Error
};
```
而对于object类型来说，TypeScript编译器不会提示任何错误：
```typescript
const obj2: object = { 
  toString() { return 123 } 
};
```
另外在处理object类型和字符串索引对象类型的赋值操作时，也要特别注意。比如：
```typescript
let strictTypeHeaders: { [key: string]: string } = {};
let header: object = {};
header = strictTypeHeaders; // OK
// Type 'object' is not assignable to type '{ [key: string]: string; }'.
strictTypeHeaders = header; // Error
```
在上述例子中，最后一行会出现编译错误，这是因为`{ [key: string]: string }`类型相比object类型更加精确。而`header = strictTypeHeaders;`这一行却没有提示任何错误，是因为这两种类型都是非基本类型，object类型比`{ [key: string]: string }`类型更加通用。
#### 3、{}类型
还有另一种类型与之非常相似，即空类型：{}。它描述了一个没有成员的对象。当你试图访问这样一个对象的任意属性时，TypeScript 会产生一个编译时错误：
```typescript
// Type {}
const obj = {};

// Error: Property 'prop' does not exist on type '{}'.
obj.prop = "semlinker";
```
但是，你仍然可以使用在Object类型上定义的所有属性和方法，这些属性和方法可通过JavaScript的原型链隐式地使用：
```typescript
// Type {}
const obj = {};

// "[object Object]"
obj.toString();
```
在JavaScript中创建一个表示二维坐标点的对象很简单：
```javascript
const pt = {}; 
pt.x = 3; 
pt.y = 4;
```
然而以上代码在TypeScript中，每个赋值语句都会产生错误：
```typescript
const pt = {};
// Property 'x' does not exist on type '{}'
pt.x = 3; // Error
// Property 'y' does not exist on type '{}'
pt.y = 4; // Error
```
这是因为第1行中的pt类型是根据它的值{}推断出来的，你只可以对已知的属性赋值。这个问题怎么解决呢？我们可能会先想到接口，比如这样子：
```typescript
interface Point {
  x: number;
  y: number;
}

// Type '{}' is missing the following 
// properties from type 'Point': x, y(2739)
const pt: Point = {}; // Error
pt.x = 3;
pt.y = 4;
```
很可惜对于以上的方案，TypeScript编译器仍会提示错误。那么这个问题该如何解决呢？其实我们可以直接通过对象字面量进行赋值：
```typescript
const pt = { 
  x: 3,
  y: 4, 
}; // OK
```
而如果你需要一步一步地创建对象，你可以使用类型断言（as）来消除 TypeScript 的类型检查：
```typescript
const pt = {} as Point; 
pt.x = 3;
pt.y = 4; // OK
```
但是更好的方法是声明变量的类型并一次性构建对象：
```typescript
const pt: Point = { 
  x: 3,
  y: 4, 
};
```
另外在使用Object.assign方法合并多个对象的时候，你可能也会遇到以下问题：
```typescript
const pt = { x: 666, y: 888 };
const id = { name: "semlinker" };
const namedPoint = {};
Object.assign(namedPoint, pt, id);

// Property 'name' does not exist on type '{}'.(2339)
namedPoint.name; // Error
```
这时候你可以使用对象展开运算符 ... 来解决上述问题：
```typescript
const pt = { x: 666, y: 888 };
const id = { name: "semlinker" };
const namedPoint = {...pt, ...id}

//(property) name: string
namedPoint.name // "semlinker"
```
## 接口与类
### 一、接口
在面向对象的语言中“接口”是个很重要的概念，它是对行为的抽象，而具体如何实现需要通过类实现，TypeScript中的接口是一个非常灵活的概念，可以用来约束对象、函数以及类的结构和类型，是一种代码协作的契约，我们必须遵守而且不能改变。
#### 1、对象类型接口
对象类型的接口用来设置对象需要存在的普通属性、可选属性和只读属性，另外还可以通过as或[propName: string]: any来制定可以接受的其他任意额外属性，举个🌰
<!-- more -->
```typescript
interface Person {
    name: string
    bool?: boolean
    readonly timestamp: number
    readonly arr: ReadonlyArray<number> // 此外还有 ReadonlyMap/ReadonlySet
}

let p1: Person = {
    name: 'oliver',
    bool: true, // 可选属性并非必要,可写可不写
    timestamp: + new Date(), // 设置只读属性
    arr: [1, 2, 3] // 设置只读数组
}

let p2: Person = {
    // Type '{ age: string; }' is not assignable to type 'Person'
    age: 'oliver', 
    // Type 'number' is not assignable to type 'string'
    name: 123 
}

// Cannot assign to 'timestamp' because it is a constant or a read-only property.
p1.timestamp = 123
// Property 'pop' does not exist on type 'ReadonlyArray<number>'.
p1.arr.pop()
```
需要注意的是`ReadonlyArray<T>`类型，它与`Array<T>`相似，只是把所有可变方法去掉了，因此可以确保数组创建后再也不能被修改，`ReadonlyMap<T>`和`ReadonlySet<T>`与之类似。
#### 2、函数类型接口
TypeScript中接口还可以用来规范函数的形状，列出参数列表及返回值类型的函数定义。写法如下：
```typescript
let add: (x: number, y: number) => number // 常规函数类型写法
// 函数类型接口写法，与常规类型完全一致
interface Add {
  (x: number, y: number): number
}

let add: Add = (a, b) => a + b 
```
#### 3、可索引类型接口
当我们不确定一个接口中有多少个属性时就可以使用可索引类型接口，接口可以通过字符串类型或数字类型索引:
```typescript
// 数字索引
interface StringArr {
    readonly [index: number]: string // index只能为number类型并且为只读属性
    length: number // 可以指定其他属性
}

let arr1: StringArr = ['hello', 'world']
// Index signature in type 'StringArr' only permits reading.
arr1[1] = ''
// Type 'number' is not assignable to type 'string'.
let arr2: StringArr = [23,12,3,21]
// 字符串索引
interface Names {
  [index: string]: string // 索引签名为string类型
}

let names: Names = {
  '1': 'xiaozhang'
}
// 两种类型混用
interface Circle {
  [x: string]: string
  [y: number]: string // 需要注意数字索引签名的返回值必须为字符串索引签名返回值的子类型，这是因为javascript会对对象的数字属性转换成字符串，所以需要保持类型的兼容性
}
```
#### 4、混合类型接口
混合类型接口就是接口既可以定义一个函数，也可以像对象一样拥有属性和方法，因此往往可以用来描述一个函数接收什么参数，输出什么结果，同时这个函数有另外什么方法或属性之类的，举个🌰
```typescript
interface Counter {
    (start: number): void // 返回类型为函数
    version: string // 增加version属性
    add(): void // 增加add方法
}

function getCounter(): Counter { // 它返回的函数必须符合接口的三点
    let count = 0
    function counter (start: number) { count = start } // counter 方法函数
    counter.version = '0.0.1'
    counter.add = function() { count++ } // add 方法增加 count
    return counter
}

const c = getCounter()
c(10) // count 默认为 10
c.version // '0.0.1'
c.add()
```
#### 5、接口的继承
跟类一样，接口通过extend关键字继承，更新新的形状，比方说继承接口并生成新的接口，这个新的接口可以设定一个新的方法检查。举个🌰
```typescript
interface PersonInfoInterface { // 1️⃣ 这里是第一个接口
    name: string
    age: number
    log?(): void
}

interface Student extends PersonInfoInterface { // 2️⃣ 这里继承了一个接口
    doHomework(): boolean // ✔️ 新增一个方法检查
}
interface Teacher extends PersonInfoInterface { // 3️⃣ 这里又继承了一个接口
    dispatchHomework(): void // ✔️ 新增了一个方法检查
}

// interface Emmm extends Student, Teacher // 也可以继承多个接口

let Alice: Teacher = {
    name: 'Alice',
    age: 34,
    dispatchHomework() { // ✔️ 必须满足继承的接口规范
        console.log('dispatched')
    }
}

let oliver: Student = {
    name: 'oliver',
    age: 12,
    log() {
        console.log(this.name, this.age)
    },
    doHomework() { // ✔️ 必须满足继承的接口规范
        return true
    }
}
```
### 二、类
在面向对象语言中，类是一种面向对象计算机编程语言的构造，是创建对象的蓝图，描述了所创建的对象共同的属性和方法。
#### 1、类的属性及方法
在TypeScript中我们通过Class关键字来定义一个类：
```typescript
class Greeter {
  // 静态属性
  static cname: string = "Greeter";

  // 成员属性
  greeting: string;

  // 构造函数 - 执行初始化操作
  constructor(message: string) {
    this.greeting = message;
  }

  // 静态方法
  static getClassName() {
    return "Class name is Greeter";
  }

  // 成员方法
  greet() {
    return "Hello, " + this.greeting;
  }
}

let greeter = new Greeter("world");
```
那么成员属性与静态属性，成员方法与静态方法有什么区别呢？我们直接查看编译后的ES5代码：
```javascript
var Greeter = /** @class */ (function () {
    // 构造函数 - 执行初始化操作
    function Greeter(message) {
        this.greeting = message;
    }
    // 静态方法
    Greeter.getClassName = function () {
        return "Class name is Greeter";
    };
    // 成员方法
    Greeter.prototype.greet = function () {
        return "Hello, " + this.greeting;
    };
    // 静态属性
    Greeter.cname = "Greeter";
    return Greeter;
}());
var greeter = new Greeter("world");
```
从编译后的代码我们不难看出成员属性会添加到类的示例上，成员方法会添加到类的原型对象上，因此对于类的实例而言是可调用的，而静态属性与静态方法都会只添加到类自身，只能被类自身调用。除此之外我们需要注意类的成员属性和方法也有public、private和protected修饰符，举个🌰
```typescript
class Dog {
  name: string // 成员属性默认为添加了public修饰符，可被类自身、子类和实例对象访问
  private age: number // 私有属性，只可被类自身访问，对于实例对象和子类皆不可见
  protected sex: string // 保护属性，对于实例对象不可见，可被类自身和子类访问
  constructor (name: string, age: number) {
    this.name = name;
    this.age = age;
    this.sex = 'male'
  }
  private pri() { console.log('private') };  // 私有方法，不可被子类或实例对象调用
  protected pro() { console.log('protected') } // 保护方法，不可被实例对象调用，可被子类调用
}

let dog = new Dog('wangcai', 2);
// Property 'age' is private and only accessible within class 'Dog'.
console.log(dog.age);
// Property 'sex' is protected and only accessible within class 'Dog' and its subclasses.
console.log(dog.sex);
// Property 'pri' is private and only accessible within class 'Dog'.
dog.pri();
// Property 'pro' is protected and only accessible within class 'Dog' and its subclasses.
dog.pro();

class Husky extends Dog {
  color: string
  constructor(name: string, age: number) {
    super(name, age);
    this.color = 'yellow';
    this.pro(); // 保护方法，可被子类调用
    // Property 'pri' is private and only accessible within class 'Dog'.
    this.pri(); // 似有方法，不可被子类调用
  }
}

// Property 'pri' does not exist on type 'typeof Husky'.
Husky.pri()
Husky.pro()
```
需要注意当我们给构造函数添加private修饰符时表示类既不可以被继承也不可以被实例化，当我们给构造函数添加protected修饰符时表示类不可被实例化只能被继承，常用于声明基类。
#### 2、ECMA私有字段
在 TypeScript 3.8 版本就开始支持ECMAScript 私有字段，使用方式如下：
```typescript
class Person {
  #name: string;

  constructor(name: string) {
    this.#name = name;
  }

  greet() {
    console.log(`Hello, my name is ${this.#name}!`);
  }
}

let semlinker = new Person("Semlinker");

// Property '#name' is not accessible outside class 'Person' because it has a private identifier.
semlinker.#name;
```
与常规属性（甚至使用private修饰符声明的属性）不同，私有字段具有以下规则：
- 私有字段以 # 字符开头，有时我们称之为私有名称；
- 每个私有字段名称都唯一地限定于其包含的类；
- 不能在私有字段上使用TypeScript可访问性修饰符（如public或private）；
- 私有字段不能在包含的类之外访问，甚至不能被检测到。

#### 3、访问器
在 TypeScript 中，我们可以通过getter和setter方法来实现数据的封装和有效性校验，防止出现异常数据。
```typescript
let passcode = "Hello TypeScript";

class Employee {
  private _fullName: string = '';

  get fullName(): string {
    return this._fullName;
  }

  set fullName(newName: string) {
    if (passcode && passcode == "Hello TypeScript") {
      this._fullName = newName;
    } else {
      console.log("Error: Unauthorized update of employee!");
    }
  }
}

let employee = new Employee();
employee.fullName = "Semlinker";
if (employee.fullName) {
  console.log(employee.fullName);
}
```
#### 4、类的继承
继承（Inheritance）是一种联结类与类的层次模型。指的是一个类（称为子类、子接口）继承另外的一个类（称为父类、父接口）的功能，并可以增加它自己的新功能的能力，继承是类与类或者接口与接口之间最常见的关系。在TypeScript中，我们通过extends关键字来实现继承，通过super关键字来调用父类的构造函数和方法。：
```typescript
class Animal {
  name: string;
  
  constructor(theName: string) {
    this.name = theName;
  }
  
  move(distanceInMeters: number = 0) {
    console.log(`${this.name} moved ${distanceInMeters}m.`);
  }
}

class Snake extends Animal {
  constructor(name: string) {
    super(name); // 调用父类的构造函数
  }
  
  move(distanceInMeters = 5) {
    console.log("Slithering...");
    super.move(distanceInMeters);
  }
}

let sam = new Snake("Sammy the Python");
sam.move();
```
#### 5、抽象类
使用abstract关键字声明的类，我们称之为抽象类。抽象类不能被实例化，因为它里面包含一个或多个抽象方法，所谓的抽象方法，是指不包含具体实现的方法，抽象类的好处在于可以抽离出一些事物的共性，有利于代码的复用，和扩展，举个🌰
```typescript
abstract class Person {
  constructor(public name: string){}

  abstract say(words: string) :void;
}

// Cannot create an instance of an abstract class.(2511)
const lolo = new Person(); // Error
```
抽象类不能被直接实例化，我们只能实例化实现了所有抽象方法的子类。具体如下所示：
```typescript
abstract class Person {
  constructor(public name: string){}

  // 抽象方法
  abstract say(words: string) :void;
}

class Developer extends Person {
  constructor(name: string) {
    super(name);
  }
  
  say(words: string): void {
    console.log(`${this.name} says ${words}`);
  }
}

const lolo = new Developer("lolo");
lolo.say("I love ts!"); // lolo says I love ts!
```
#### 6、基于抽象类实现多态
面向对象（OOP）语言的三大特性分别是：封装（Encapsulation）、继承（Inheritance）和多态（Polymorphism），多态是指由继承而产生了相关的不同的类，对同一个方法可以有不同的响应。比如Cat和Dog都继承自Animal，但是分别实现了自己的eat方法。此时针对某一个实例，我们无需了解它是Cat还是Dog，就可以直接调用eat方法，程序会自动判断出来应该如何执行eat：
```typescript
abstract class Animal {
  abstract sleep(): void
}
class Dog {
  sleep() {
    console.log("dog sleep")
  }
}
let dog = new Dog();
class Cat {
  sleep() {
    console.log("cat sleep")
  }
}

let cat = new Cat()
let animals: Animal[] = [dog, cat]
animals.forEach(i => {
  i.sleep()
})
// dog sleep
// cat sleep
```
#### 7、类的方法的重载
对于类的方法来说，它也支持重载。比如，在以下示例中我们重载了ProductService类的getProducts成员方法：
```typescript
class ProductService {
    getProducts(): void;
    getProducts(id: number): void;
    getProducts(id?: number) {
      if(typeof id === 'number') {
        console.log(`获取id为 ${id} 的产品信息`);
      } else {
        console.log(`获取所有的产品信息`);
      }  
    }
}

const productService = new ProductService();
productService.getProducts(666); // 获取id为 666 的产品信息
productService.getProducts(); // 获取所有的产品信息 
```
### 三、类与接口的关系
#### 1、类可以实现接口
如果你希望在类中使用必须要被遵循的接口（类）或别人定义的对象结构，可以使用implements关键字来确保其兼容性：
```typescript
interface Human {
  name: string;
  eat(): void;
}

class Asian implements Human {
  name: string;
  constructor (name: string) {
    this.name = name;
  }
  eat() {}
}
```
类实现接口需要注意的有以下几点：
- 类实现接口的时候必须实现接口定义的所有属性，但是类可以定义接口之外自己的属性
- 接口只能约束类的公有成员
- 接口也不能约束类的构造函数

#### 2、接口可以继承类
接口除了可以继承接口还可以继承类，相当于接口把类的成员都抽象了出来，也就是只有类的成员结构而没有具体的实现，举个🌰
```typescript
class Auto {
   state = 1
}
interface AutoInterface extends Auto {} // 接口内只有成员state且类型为number
class C implements AutoInterface {
    state = 1
}
```
需要注意的是接口在抽离类的成员时不仅抽离了公共成员，还抽离了私有成员和受保护成员，举个🌰
```typescript
class Auto {
   state = 1
   private num = 12
}
interface AutoInterface extends Auto {}

// Property 'num' is missing in type 'C' but required in type 'AutoInterface'.
class C implements AutoInterface {
    state = 1;
}

// 需要注意的是由于num是Auto的私有属性，而类Class并不是其子类，因此自然也不能包含它的非公有成员，所以即使我们在C中声明了num还是会报错
// Property 'num' is private in type 'AutoInterface' but not in type 'C'.
class C implements AutoInterface {
  state = 1;
  num = 14
}
```
## 联合类型与类型别名
### 一、联合类型
### 二、类型别名
TypeScript引入了类型别名type，其作用就是给类型起一个新名字，可以作用于原始值（基本类型）、联合类型、元组以及其它任何你需要手写的类型：
```typescript
type Second = number; // 基本类型
let timeInSecond: number = 10;
let time: Second = 10;  // time的类型其实就是number类型
type userOjb = {name:string} // 对象
type getName = ()=>string  // 函数
type data = [number,string] // 元组
type numOrFun = Second | getName  // 联合类型
```
需要注意的是起别名不会新建一个类型 - 它创建了一个新名字来引用那个类型。给基本类型起别名通常没什么用。类型别名常用于联合类型。

#### 1、type的应用和interface的区别
(1)**和接口一样，用来描述对象或函数的类型**
```typescript
type User = {
  name: string
  age: number
};
type SetUser = (name: string, age: number)=>void;
```
在ts编译成js后，所有的接口和type 都会被擦除掉。
(2)**扩展和实现**
接口可以扩展，但type不能extends和implement,但是type可以通过交叉类型实现interface的extends行为。interface 可以extends type，同时type也可以与interface类型交叉，举个🌰：
```typescript
// interface 扩展 type
interface Name {
  name: string;
}
interface User extends Name {
  age: number;
}
let stu:User={name:'wang',age:10}

// 上面的扩展可以用type交叉类型来实现
type Name = {
  name: string;
}
type User = Name & { age: number  };
let stu: User = { name: 'wang',age: 1 };
console.log(stu) // { name: 'wang', age: 1 }
```
(3)**接口的声明合并**
接口可以定义多次，并将被视为单个接口（即所有声明属性的合并）。而type不可以定义多次。
```typescript
interface User {
  name: string
  age: number
}

interface User {
  sex: string
}
let user: User = { name: 'wang',age: 1,sex: 'man' }
```
(4)**映射类型**
type 能使用 in 关键字生成映射类型，但 interface 不行。
```typescript
type Keys = "name" | "sex"

type DulKey = {
  [key in Keys]: string    // 类似for...in
}

let stu: DulKey = {
  name: "wang",
  sex: "man"
}
```

参考资料：
极客时间《TypeScript开发实战》专栏
《深入理解TypeScript》
[typeScript 中的type关键字](https://juejin.cn/post/6876359681464336397)
[一文读懂 TS 中 Object, object, {} 类型之间的区别](http://www.semlinker.com/ts-object-type/)
[一份不可多得的 TS 学习指南（1.8W字）](https://juejin.im/post/6872111128135073806#heading-21)
[Typescript使用手册](https://www.bookstack.cn/read/TypeScript-3.6/doc-handbook-README.md)