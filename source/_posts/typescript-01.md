---
title: TypeScript学习笔记
date: 2020-11-15 16:30:03
toc: true
categories: 
- 前端
tags: 
- TypeScript
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

<img src="/assets/typescript-01/01.png" width="520" />

- 在程序运行时动态计算属性偏移量
- 需要额外的空间存储属性名
- 所有对象的编译量信息各存一份

而对于C++而言：

<img src="/assets/typescript-01/02.png" width="520" />

- 编译阶段确定属性偏移量
- 偏移量访问代替属性名访问
- 偏移量信息共享

由此可以看到动态类型语言无论在时间还是空间上都有比较多的性能损耗，虽然实际上V8为了提升JavaScript运行时的性能做了很多优化，但是TypeScript更重要的价值在于将静态类型的编程思维引入了javaScript，让我们可以在编译阶段以一种完全不同的视角去看待我们的代码。

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

从上面示例中我们可以发现字符串枚举没有实现反向映射。

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

除了数字枚举和字符串枚举之外，还有一种特殊的枚举——常量枚举。它是使用const关键字修饰的枚举，常量枚举会使用内联语法，不会为枚举类型编译生成任何JavaScript，举个🌰:

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
- 类型：枚举类型成员的值包括两种类型：**常量类型(const number)** 和 **计算类型(computer number)** ，常量类型包括没有初始值、引用已有枚举属性值和常量表达式三类，常量类型会在编译阶段编译出结果，以常量的形式出现在运行时环境，计算类型主要是一些非常量的表达式，这些表达式的值不会在编译阶段被计算而是保留到执行时阶段。

我们举个🌰:

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

### 九、Tuple类型

众所周知，数组一般由同种类型的值组成，但有时我们需要在单个变量中存储不同类型的值，这时候我们就可以使用元组。在JavaScript中是没有元组的，元组是TypeScript中特有的类型，其工作方式类似于数组。
元组可用于定义具有有限数量的未命名属性的类型。每个属性都有一个关联的类型。使用元组时，必须提供每个属性的值，举个🌰:

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

需要注意的是在TypeScript规范中undefined和null是所有类型的子类型，所以当我们将tsconfig.json中的strictNullChecks属性设为false时我们可以将null和undefined赋值给其他类型的值，举个🌰:

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

还有一点要注意的是，对bigint使用typeof操作符返回一个新的字符串："bigint"。因此，TypeScript能够正确地使用typeof细化类型，举个🌰:

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

然而他忘记同时修改 controlFlowAnalysisWithNever 方法中的控制流程，这时候else分支的foo类型会被收窄为boolean类型，导致无法赋值给never类型，这时就会产生一个编译错误。通过这个方式，我们可以确保controlFlowAnalysisWithNever方法总是穷尽了Foo的所有可能类型。 通过这个示例，我们可以得出一个结论：使用never避免出现新增了联合类型没有对应的实现，目的就是写出类型绝对安全的代码。

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

(1)**Object接口定义**

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

(2)**ObjectConstructor接口定义**

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

## 断言

### 一、类型断言

有时候你会遇到这样的情况，你会比 TypeScript 更了解某个值的详细信息。通常这会发生在你清楚地知道一个实体具有比它现有类型更确切的类型。
通过类型断言这种方式可以告诉编译器，“相信我，我知道自己在干什么”。类型断言好比其他语言里的类型转换，但是不进行特殊的数据检查和解构。它没有运行时的影响，只是在编译阶段起作用。
类型断言有两种形式：

#### 1、“尖括号”语法

```typescript
let someValue: any = "this is a string";
let strLength: number = (<string>someValue).length;
```

#### 2、as语法

```typescript
let someValue: any = "this is a string";
let strLength: number = (someValue as string).length;
```

### 二、非空断言

在上下文中当类型检查器无法断定类型时，一个新的后缀表达式操作符!可以用于断言操作对象是非null和非undefined类型。具体而言，x!将从x值域中排除null和undefined。
那么非空断言操作符到底有什么用呢？下面我们先来看一下非空断言操作符的一些使用场景：

#### 1、忽略undefined和null类型

```typescript
function myFunc(maybeString: string | undefined | null) {
  // Type 'string | null | undefined' is not assignable to type 'string'.
  // Type 'undefined' is not assignable to type 'string'. 
  const onlyString: string = maybeString; // Error
  const ignoreUndefinedAndNull: string = maybeString!; // Ok
}
```

#### 2、调用时忽略undefined类型

```typescript
type NumGenerator = () => number;

function myFunc(numGenerator: NumGenerator | undefined) {
  // Object is possibly 'undefined'.(2532)
  // Cannot invoke an object which is possibly 'undefined'.(2722)
  const num1 = numGenerator(); // Error
  const num2 = numGenerator!(); //OK
}
```

因为!非空断言操作符会从编译生成的JavaScript代码中移除，所以在实际使用的过程中，要特别注意。比如下面这个例子：

```typescript
const a: number | undefined = undefined;
const b: number = a!;
console.log(b); 
```

以上TS代码会编译生成以下ES5代码：

```typescript
"use strict";
const a = undefined;
const b = a;
console.log(b);
```

虽然在 TS 代码中，我们使用了非空断言，使得`const b: number = a!;`语句可以通过TypeScript类型检查器的检查。但在生成的ES5代码中，!非空断言操作符被移除了，所以在浏览器中执行以上代码，在控制台会输出 undefined。

### 三、确定赋值断言

在TypeScript 2.7版本中引入了确定赋值断言，即允许在实例属性和变量声明后面放置一个!号，从而告诉TypeScript该属性会被明确地赋值。为了更好地理解它的作用，我们来看个具体的例子：

```typescript
let x: number;
initialize();
// Variable 'x' is used before being assigned.(2454)
console.log(2 * x); // Error

function initialize() {
  x = 10;
}
```

很明显该异常信息是说变量 x 在赋值前被使用了，要解决该问题，我们可以使用确定赋值断言：

```typescript
let x!: number;
initialize();
console.log(2 * x); // Ok

function initialize() {
  x = 10;
}
```

通过`let x!: number;`确定赋值断言，TypeScript编译器就会知道该属性会被明确地赋值。

## 接口与类

### 一、接口

在面向对象的语言中“接口”是个很重要的概念，它是对行为的抽象，而具体如何实现需要通过类实现，TypeScript中的接口是一个非常灵活的概念，可以用来约束对象、函数以及类的结构和类型，是一种代码协作的契约，我们必须遵守而且不能改变。

#### 1、对象类型接口

对象类型的接口用来设置对象需要存在的普通属性、可选属性和只读属性，另外还可以通过as或[propName: string]: any来制定可以接受的其他任意额外属性，举个🌰:
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

混合类型接口就是接口既可以定义一个函数，也可以像对象一样拥有属性和方法，因此往往可以用来描述一个函数接收什么参数，输出什么结果，同时这个函数有另外什么方法或属性之类的，举个🌰:

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

跟类一样，接口通过extend关键字继承，更新新的形状，比方说继承接口并生成新的接口，这个新的接口可以设定一个新的方法检查。举个🌰:

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

从编译后的代码我们不难看出成员属性会添加到类的示例上，成员方法会添加到类的原型对象上，因此对于类的实例而言是可调用的，而静态属性与静态方法都会只添加到类自身，只能被类自身调用。除此之外我们需要注意类的成员属性和方法也有public、private和protected修饰符，举个🌰:

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

继承（Inheritance）是一种联结类与类的层次模型。指的是一个类（称为子类、子接口）继承另外的一个类（称为父类、父接口）的功能，并可以增加它自己的新功能的能力，继承是类与类或者接口与接口之间最常见的关系。在TypeScript中，我们通过extends关键字来实现继承，通过super关键字来调用父类的构造函数和方法，举个🌰:

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

使用abstract关键字声明的类，我们称之为抽象类。抽象类不能被实例化，因为它里面包含一个或多个抽象方法，所谓的抽象方法，是指不包含具体实现的方法，抽象类的好处在于可以抽离出一些事物的共性，有利于代码的复用，和扩展，举个🌰:

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

方法重载是指在同一个类中方法同名，参数不同（参数类型不同、参数个数不同或参数个数相同时参数的先后顺序不同），调用时根据实参的形式，选择与它匹配的方法执行操作的一种技术。所以类中成员方法满足重载的条件是：在同一个类中，方法名相同且参数列表不同，在以下示例中我们重载了ProductService类的getProducts成员方法：

```typescript
class ProductService {
  // 重载签名
  getProducts(): void;
  getProducts(id: number): void;
  // 重载实现
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

这里需要注意的是，当TypeScript编译器处理方法重载时，它会查找重载列表，尝试使用第一个重载定义。 如果匹配的话就使用这个。 因此，在定义重载的时候，一定要把最精确的定义放在最前面。另外在ProductService类中，`getProducts(id?: number){}`并不是重载列表的一部分，因此对于`getProducts`成员方法来说，我们只定义了两个重载方法。

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

接口除了可以继承接口还可以继承类，相当于接口把类的成员都抽象了出来，也就是只有类的成员结构而没有具体的实现，举个🌰:

```typescript
class Auto {
   state = 1
}
interface AutoInterface extends Auto {} // 接口内只有成员state且类型为number
class C implements AutoInterface {
    state = 1
}
```

需要注意的是接口在抽离类的成员时不仅抽离了公共成员，还抽离了私有成员和受保护成员，举个🌰:

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

## 类型别名

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

### 一、type的应用和interface的区别

#### 1、和接口一样，用来描述对象或函数的类型

```typescript
type User = {
  name: string
  age: number
};
type SetUser = (name: string, age: number)=>void;
```

在ts编译成js后，所有的接口和type 都会被擦除掉。

#### 2、扩展和实现
接口可以扩展，但type不能extends和implement,但是type可以通过交叉类型实现interface的extends行为。interface 可以extends type，同时type也可以与interface类型交叉，举个🌰:

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

#### 3、接口的声明合并

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

#### 4、映射类型

type 能使用 in 关键字生成映射类型，但interface不行。

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

## 函数

### 一、TypeScript中函数与JavaScript中的区别

| TypeScript | JavaScript |
|--|--|
| 含有类型 | 无类型 |
| 箭头函数 | 剪头函数（ES2015）|
| 函数类型 | 无函数类型 |
| 必填参数和可选参数 | 所有参数都是可选的 |
| 默认参数 | 默认参数 |
| 剩余参数 | 剩余参数 |
| 函数重载 | 无函数重载 |

### 二、函数的定义

TypeScript中函数的定义一共有四种方式：

```typescript
// 函数类型的定义及实现
function add1(x: number, y: number) {
  return x + y;
}

// 通过变量名定义函数类型，后需指定具体实现
let add2: (x: number, y: number) => number; // 无法描述函数重载

// 通过类型别名定义函数类型，后需指定具体实现
type add3 = (x: number, y: number) => number; // 无法描述函数重载
type add4 = {
  (x: number, y: number): number
}
// 通过接口定义函数类型，后需指定具体实现
interface add5 {
  (x: number, y: number): number;
}
```

### 三、可选参数与默认参数

```typescript
// 可选参数
function createUserId(name: string, id: number, age?: number): string {
  return name + id;
}

// 默认参数
function createUserId(
  name: string = "semlinker",
  id: number,
  age?: number
): string {
  return name + id;
}
```

此处需要注意的是可选参数要放在普通参数的后面，不然会导致编译错误。

### 四、剩余参数

```typescript
function push(array, ...items) {
  items.forEach(function (item) {
    array.push(item);
  });
}

let a = [];
push(a, 1, 2, 3);
```

### 四、函数的重载

在静态类型的语言中例如C++、Java都有函数重载的概念，它们本质上还是使用相同名称和不同参数数量或类型的多个函数，函数重载的好处在于我们不需要为相似或相同功能的函数选择不同的名称，这样增强了函数的可读性，TypeScript中的函数重载与C++、Java中的有所不同，举个🌰:

```typescript
// 重载签名
function add(a: number, b: number): number;
function add(a: string, b: string): string;
function add(a: string, b: number): string;
function add(a: number, b: string): string;
// 在一个最宽泛的版本中实现函数重载
function add(a: string | number, b: string | number) {
  if (typeof a === 'string' || typeof b === 'string') {
    return a.toString() + b.toString();
  }
  return a + b;
}
```

与类的方法的重载类似，当TypeScript编译器处理函数重载时，它会查找重载列表，尝试使用第一个重载定义，因此，在定义重载的时候，一定要把最精确的定义放在最前面。
需要注意的是**TypeScript中的函数重载没有任何运行时开销**，它只允许你记录希望调用函数的方式，并且编译器会检查其余代码，举个🌰，上述示例编译成JavaScript后的内容如下所示：

```javascript
"use strict";
function add(a, b) {
  if (typeof a === 'string' || typeof b === 'string') {
    return a.toString() + b.toString();
  }
  return a + b;
}
```

## 泛型

泛型（Generics）在编程语言中是一个较为普遍的概念，在像 C# 和 Java 这样的语言中，可以使用泛型来创建可重用的组件，一个组件可以支持多种类型的数据。 这样用户就可以以自己的数据类型来使用组件。这给软件工程带来了极高的灵活性，进一步提高了组件或函数的可重用性。那么泛型具体的定义是什么呢？
泛型是指不预先确定的数据类型，具体的类型在使用的时候才能确定，它允许同一个函数可以接受不同类型参数的一个模板。设计泛型的关键目的是在成员之间提供有意义的约束，这些成员可以是：类的实例成员、类的方法、函数参数和函数返回值。

### 一、泛型函数

```typescript
function log<T>(value: T): T {
  console.log(value);
  return value;
}
log<string>('hello'); // 'hello'
```

如上所示，当我们调用`log<string>('hello')`时，string类型就像参数一样，它将在出现`T`的任何位置填充该类型。图中`<T>`内部的`T`被称为类型变量，它是我们希望传递给log函数的类型占位符，同时它被分配给 value参数和函数返回值用来代替它的类型：此时`T`充当的是类型，而不是特定的string类型。
我们除了可以这样显式定义泛型函数，还可以先通过类型别名指定泛型函数类型，然后指定函数实现，举个🌰:

```typescript
type Log = <T, U>(value: T, comment: U) => T;
function log<T, U>(value: T, comment: U): T {
  console.log(comment);
  return value;
}

let myLog: Log = log
```

### 二、泛型接口

上面函数我们也可以通过泛型接口实现定义：

```typescript
// 这儿接口与类型别名完全一致
interface Log {
  <T>(value: T): T
}
function log<T>(value: T): T {
  console.log(value);
  return value;
}
let mylog: Log = log;
```

在上述示例的接口中泛型仅仅约束了一个函数，我们也可以用泛型来约束接口的其他成员，举个🌰:

```typescript
interface Obj<T> {
  value: T;
  name: string;
}
// 需要注意这种情形下我们必须注明泛型类型，不支持类型推断
const obj1: Obj<number> = {
  value: 21,
  name: 'age'
}
// Generic type 'Obj<T>' requires 1 type argument(s).
const obj2: Obj = {
  value: 21,
  name: 'age'
}
```

除此之外我们还可以为泛型接口指定一个默认类型，举个🌰:

```typescript
interface Obj<T = number> {
  value: T;
  name: string;
}

const obj: Obj = {
  value: 2021,
  name: 'year'
}
```

### 三、泛型类

与接口类似，泛型还可以约束类的成员，举个🌰:

```typescript
// 我们将泛型放在类的后面这样就可以约束类的所有成员了
class Log<T> {
  run(value: T) {
    console.log(value);
    return value
  }
}

const log1 = new Log<number>()
log1.run(1234)

// 如果不指定泛型则可以使用任意类型
const log2 = new Log()
log2.run('12')
log2.run({name: 'kylee'})
```

此处需要注意的是泛型约束不能作用于静态属性和方法，举个🌰:

```typescript
class Greeter<T> {
  // 静态属性是只读属性，必须在初始化的时候赋值，因此无法使用泛型
  static cname: string = "Greeter";

  // 静态方法添加到类自身，不能获取到类实例内部的泛型参数
  // Parameter 'value' of public static method from exported class has or is using private name 'T'.
  static getClassName(value: T) {
    return value;
  }
}
```

### 四、泛型约束

在部分情况下我们需要对泛型做一些约束，这个时候我们就需要用到泛型约束，举个🌰:

```typescript
function log<T> (value: T): T {
  // 这种情况下TypeScript编译器会提示泛型T上不存在length属性
  console.log(value, value.length);
  return value;
}

// 此处我们可以通过接口实现对泛型的约束
interface Log {
  length: number;
}
// 此时我们为泛型T引入约束，必须具备length属性
function log<T extends Log> (value: T): T {
  console.log(value, value.length);
  return value;
}

// 泛型约束能够带来很多场景的巧妙使用，比如上述示例我们在不指定泛型的情况下我们可以传入所有带有length属性的变量
log([1]);
log('12334');
leo({ length: 23 });
```

### 五、泛型工具

为了方便开发者 TypeScript 内置了一些常用的工具类型，比如 Partial、Required、Readonly、Record和ReturnType等，不过在具体介绍之前，我们得先介绍一些相关的基础知识：

#### 1、基础知识

(1)**typeof**
在TypeScript中，typeof操作符可以用来获取一个变量声明或对象的类型，举个🌰:

```typescript
interface Person {
  name: string;
  age: number;
}

const sem: Person = { name: 'semlinker', age: 33 };
type Sem = typeof sem; // Person

function toArray(x: number): Array<number> {
  return [x];
}

type Func = typeof toArray; // (x: number) => number[]
```

(2)**keyof**
keyof操作符是在TypeScript 2.1版本引入的，该操作符可以用于获取某种类型的所有键，其返回类型是联合类型。

```typescript
interface Person {
  name: string;
  age: number;
}

type K1 = keyof Person; // "name" | "age"
type K2 = keyof Person[]; // "length" | "toString" | "pop" | "push" | "concat" | "join" 
type K3 = keyof { [x: string]: Person };  // string | number
```

在 TypeScript 中支持两种索引签名，数字索引和字符串索引：

```typescript
interface StringArray {
  // 字符串索引 keyof StringArray => string | number
  [index: string]: string; 
}

interface StringArray1 {
  // 数字索引 keyof StringArray1 => number
  [index: number]: string;
}
```

为了同时支持两种索引类型，就得要求数字索引的返回值必须是字符串索引返回值的子类。其中的原因就是当使用数值索引时，JavaScript在执行索引操作时，会先把数值索引先转换为字符串索引。所以`keyof { [x: string]: Person }`的结果会返回`string | number`。

(3)**in**
in用来遍历枚举类型

```typescript
type Keys = "a" | "b" | "c"

type Obj =  {
  [p in Keys]: any
} // -> { a: any, b: any, c: any }
```

(4)**infer**
infer表示在extends条件类型语句中待推断的类型变量，简单举个🌰:

```typescript
type ParamType<T> = T extends (param: infer P) => any ? P : T;
```

在这个条件语句`T extends (param: infer P) => any ? P : T`中，`infer P`表示待推断的函数参数。
整句表示为：如果 T 能赋值给`(param: infer P) => any`，则结果是`(param: infer P) => any`类型中的参数`P`，否则返回为`T`。

```typescript
interface User {
  name: string;
  age: number;
}

type Func = (user: User) => void;

type Param = ParamType<Func>; // Param = User
type AA = ParamType<string>; // string
```

#### 2、Partial

`Partial<T>` 的作用就是将某个类型里的属性全部变为可选项

```typescript
type Partial<T> = {
  [P in keyof T]?: T[P];
};
```

在以上代码中，首先通过 keyof T 拿到 T 的所有属性名，然后使用 in 进行遍历，将值赋给 P，最后通过 T[P] 取得相应的属性值。中间的 ? 号，用于将所有属性变为可选。举个🌰:

```typescript
interface Todo {
  title: string;
  description: string;
}

function updateTodo(todo: Todo, fieldsToUpdate: Partial<Todo>) {
  return { ...todo, ...fieldsToUpdate };
}

const todo1 = {
  title: "Learn TS",
  description: "Learn TypeScript",
};

const todo2 = updateTodo(todo1, {
  description: "Learn Ge Chui Zi",
});
```

#### 3、Required

`Required<T>`的作用是将传入的属性变为必选项

```typescript
type Required<T> = { [P in keyof T]-?: T[P] };
```

这里的`-?`就是将可选项代表的`?`去掉, 从而让这个类型变成必选项. 与之对应的还有个`+?`, 这个含义自然与`-?`之前相反, 它是用来把属性变成可选项的。举个🌰:

```typescript
interface Todo {
  title: string;
  description?: string;
}

function updateTodo(todo: Required<Todo>, fieldsToUpdate: Todo) {
  return { ...todo, ...fieldsToUpdate };
}

const todo1 = {
  title: "Learn TS",
  description: "Learn TypeScript",
};

const todo2 = updateTodo(todo1, {
  title: "Learn Ge Chui Zi"
});
```

#### 4、Readonly

`Readonly<T>`用于将所有传入的属性转变成只读项

```typescript
type Readonly<T> = { readonly [P in keyof T]: T[P] };
```

举个🌰:

```typescript
interface Todo {
  title: string;
  description: string;
}

function updateTodo(todo: Readonly<Todo>) {
  // Cannot assign to 'title' because it is a read-only property.
  todo.title = 'Learn JS'
}

const todo1 = {
  title: "Learn TS",
  description: "Learn TypeScript",
};

```

#### 5、Record

`Record<K, T>`用于将K中所有的属性的值转化为T类型

```typescript
type Record<K extends keyof any, T> = { [P in K]: T };
```

举个🌰:

```typescript
type subjects = 'ts' | 'js';
interface Todo {
  title: string,
  description: string
}

type Sub = Record<subjects, Todo>;

const sub: Sub = {
  ts: {
    title: 'learn TS',
    description: 'learn TypeScript'
  },
  js: {
    title: 'learn JS',
    description: 'learn JavaScript'
  }
}
```

#### 6、Pick

`Pick<T, K>`用于从T中取出一系列K的属性

```typescript
type Pick<T, K extends keyof T> = { [P in K]: T[P] };
```

举个🌰:

```typescript
interface Sup {
  name: string;
  value: string;
  color: string;
}

type Sub = 'name' | 'value';

type PickType = Pick<Sup, Sub>

const pick: PickType = {
  name: 'width',
  value: '100px',
}
```

#### 7、Exclude

在ts 2.8中引入了一个条件类型, 示例如下:

```typescript
T extends U ? X : Y
```

以上语句的意思就是 如果T是U的子类型的话，那么就会返回X，否则返回Y
条件类型甚至可以组合多个，举个🌰:

```typescript
type TypeName<T> =
    T extends string ? "string" :
    T extends number ? "number" :
    T extends boolean ? "boolean" :
    T extends undefined ? "undefined" :
    T extends Function ? "function" :
    "object";
```

对于联合类型来说会自动分发条件，例如`T extends U ? X : Y`，`T`可能是`A | B`的联合类型, 那实际情况就变成`(A extends U ? X : Y) | (B extends U ? X : Y)`
有了以上的了解我们再来了解工具泛型Exclude，`Exclude<T, U>` 的作用是从T中找出U中没有的元素, 换种更加贴近语义的说法其实就是从T中排除U

```typescript
type Exclude<T, U> = T extends U ? never : T;
```

举个🌰:

```typescript
type T = Exclude<1 | 2, 1 | 3> // -> 2
```

#### 8、Extract

与Exclude恰好相反，`Extract<T, U>`的作用是提取出T包含在U中的元素, 换种更加贴近语义的说法就是从T中提取出U

```typescript
type Extract<T, U> = T extends U ? T : never;
```

举个🌰:

```typescript
type T = Exclude<1 | 2, 1 | 3> // -> 1
```

#### 9、ReturnType、Parameters、InstanceType、ConstructorParameters、

在2.8版本中，TypeScript内置了一些与infer有关的映射类型，当当infer用于函数类型中，可用于参数位置`new (...args: infer P) => any;`和返回值位置 `new (...args: any[]) => infer P;`。
因此就内置如下两个映射类型：
**用于提取函数类型的返回值类型**:

```typescript
type ReturnType<T> = T extends (...args: any[]) => infer P ? P : any;
```

**用于提取函数中参数类型**

```typescript
type Parameters<T extends (...args: any[]) => any> = T extends (...args: infer P) => any ? P : never;
```

举个🌰:

```typescript
type return = ReturnType<() => string>; // string
type parametersA = Parameters<() => void>;           // []
type parametersB = Parameters<typeof Array.isArray>; // [any]
```

**用于提取构造函数中参数（实例）类型**:
一个构造函数可以使用new来实例化，因此它的类型通常表示如下：

```typescript
type Constructor = new (...args: any[]) => any;
```

当infer用于构造函数类型中，可用于参数位置`new (...args: infer P) => any;`和返回值位置 `new (...args: any[]) => infer P;`。
因此就内置如下两个映射类型：

```typescript
// 获取参数类型
type ConstructorParameters<T extends new (...args: any[]) => any> = T extends new (...args: infer P) => any ? P : never;

// 获取实例类型
type InstanceType<T extends new (...args: any[]) => any> = T extends new (...args: any[]) => infer R ? R : any;
```

举个🌰:

```typescript
class TestClass {
  constructor(public name: string, public age: number) {}
}

type Params = ConstructorParameters<typeof TestClass>; // [string, number]

type Instance = InstanceType<typeof TestClass>; // TestClass
```

#### 10、NonNullable

`NonNullable<T>`主要用于从T中剔除null和undefined

```typescript
type NonNullable<T> = T extends null | undefined ? never : T;
```

举个🌰:

```typescript
type T0 = NonNullable<string | number | undefined>; // string | number
type T1 = NonNullable<string[] | null | undefined>; // string[]
```

## 装饰器

在 ES6 中增加了对类对象的相关定义和操作（比如class和extends），这就使得我们在多个不同类之间共享或者扩展一些方法或者行为的时候，变得并不是那么优雅。这个时候，我们就需要一种更优雅的方法来帮助我们完成这些事情，这个方法就是**装饰器**。
装饰器（decorators）这一特性的提出来源于python之类的语言，如果你熟悉python的话，对它一定不会陌生。那么我们先来看一下python里的装饰器是什么样子的吧：
> A Python decorator is a function that takes another function, extending the behavior of the latter function without explicitly modifying it.

它的主要作用是给一个已有的方法或类扩展一些新的行为，而不是去直接修改它本身，JavaScript提案中对于装饰器的定义与之类似，但是装饰器被提出已经有一年多的时间了，同时期的很多其他新的特性已经随着 ES6 的推进而被大家广泛使用，装饰器现在却还停留在 stage 2 的阶段，不过幸运的是其语法在TypeScript中已经得到了实现，需要注意的是，若要启用实验性的装饰器特性，你必须在命令行或tsconfig.json里启用 experimentalDecorators编译器选项：

```json
{
  "compilerOptions": {
     "target": "ES5",
     "experimentalDecorators": true
   }
}
```

下面我们具体介绍TypeScript中的装饰器：

### 一、装饰器是什么

- 它是一个表达式
- 该表达式被执行后，返回一个函数
- 函数的入参分别为target、name和descriptor
- 执行该函数后，可能返回descriptor对象，用于配置target对象

### 二、装饰器的分类

- 类装饰器（Class decorators）
- 属性装饰器（Property decorators）
- 方法装饰器（Method decorators）
- 参数装饰器（Parameter decorators）

### 三、类装饰器

类装饰器声明：

```typescript
declare type ClassDecorator = <TFunction extends Function>(target: TFunction) => TFunction | void;
```

类装饰器顾名思义，就是用来装饰类的。它接收一个参数：

- target: TFunction - 被装饰的类

举个🌰:

```typescript
function Greeter(greeting: string) {
  return function (target: Function) {
    target.prototype.greet = function (): void {
      console.log(greeting);
    };
  };
}

@Greeter("Hello TS!")
class Greeting {
  constructor() {
    // 内部实现
  }
}

let myGreeting = new Greeting();
(myGreeting as any).greet(); // console output: 'Hello TS!';
```

### 四、属性装饰器

属性装饰器声明：

```typescript
declare type PropertyDecorator = (target:Object, propertyKey: string | symbol ) => void;
```

属性装饰器顾名思义，用来装饰类的属性。它接收两个参数：

- target: Object - 被装饰的类
- propertyKey: string | symbol - 被装饰类的属性名

```typescript
function logProperty(target: any, key: string) {
  delete target[key];

  const backingField = "_" + key;

  Object.defineProperty(target, backingField, {
    writable: true,
    enumerable: true,
    configurable: true
  });

  // property getter
  const getter = function (this: any) {
    const currVal = this[backingField];
    console.log(`Get: ${key} => ${currVal}`);
    return currVal;
  };

  // property setter
  const setter = function (this: any, newVal: any) {
    console.log(`Set: ${key} => ${newVal}`);
    this[backingField] = newVal;
  };

  // Create new property with getter and setter
  Object.defineProperty(target, key, {
    get: getter,
    set: setter,
    enumerable: true,
    configurable: true
  });
}

class Person { 
  @logProperty
  public name: string;

  constructor(name : string) { 
    this.name = name;
  }
}

const p1 = new Person("kylee");
p1.name = "xiaozhang";
```

以上代码我们定义了一个 logProperty 函数，来跟踪用户对属性的操作，当代码成功运行后，在控制台会输出以下结果：

```
"Set: name => kylee" 
"Set: name => xiaozhang" 
```

### 五、方法装饰器

方法装饰器声明：

```typescript
declare type MethodDecorator = <T>(target:Object, propertyKey: string | symbol, descriptor: TypePropertyDescriptor<T>) => TypedPropertyDescriptor<T> | void;
```

方法装饰器顾名思义，用来装饰类的方法。它接收三个参数：

- target: Object - 被装饰的类
- propertyKey: string | symbol - 方法名
- descriptor: TypePropertyDescriptor - 属性描述符

举个🌰:

```typescript
function log(target: Object, propertyKey: string, descriptor: PropertyDescriptor) {
  let originalMethod = descriptor.value;
  descriptor.value = function (...args: any[]) {
    console.log("wrapped function: before invoking " + propertyKey);
    let result = originalMethod.apply(this, args);
    console.log("wrapped function: after invoking " + propertyKey);
    return result;
  };
}

class Task {
  @log
  runTask(arg: any): any {
    console.log("runTask invoked, args: " + arg);
    return "finished";
  }
}

let task = new Task();
let result = task.runTask("learn ts");
console.log("result: " + result);
```

以上代码成功运行后，控制台会输出以下结果：

```
"wrapped function: before invoking runTask" 
"runTask invoked, args: learn ts" 
"wrapped function: after invoking runTask" 
"result: finished"
```

### 六、参数装饰器

参数装饰器声明：

```typescript
declare type ParameterDecorator = (target: Object, propertyKey: string | symbol, parameterIndex: number ) => void
```

参数装饰器顾名思义，是用来装饰函数参数，它接收三个参数：

- target: Object - 被装饰的类
- propertyKey: string | symbol - 方法名
- parameterIndex: number - 方法中参数的索引值

举个🌰:

```typescript
function Log(target: Function, key: string, parameterIndex: number) {
  let functionLogged = key || target.prototype.constructor.name;
  console.log(`The parameter in position ${parameterIndex} at ${functionLogged} has been decorated`);
}

class Greeter {
  greeting: string;
  constructor(@Log phrase: string) {
    this.greeting = phrase; 
  }
}
```

以上代码成功运行后，控制台会输出以下结果：

```
"The parameter in position 0 at Greeter has been decorated" 
```

### 七、Reflect Metadata

Reflect Metadata是ES7的一个提案，它主要用来在声明的时候添加和读取元数据，它的提出主要是基于如下目的：

1. 有很多设计模式, 比如组合, 依赖注入, 运行时类型断言, 反射/镜像, 测试等希望可以在保持原有class的一致性的前提下为class添加元数据
2. 一致性是很多工具和库使用元数据的原因
3. 元数据产生的装饰器可以通过改变装饰器来进行组合
4. 元数据不仅仅只能在对象上使用, 也应该被代理Proxy通过相应的traps所使用
5. 定义一个新的元数据装饰器不应该过于复杂
6. 元数据应该保持与其他语言以及ECMAScript的运行特性的一致性

TypeScript在1.5+的版本已经支持它，在使用之前我们需要先引入：

- `npm i reflect-metadata --save`
- 在`tsconfig.json`里配置emitDecoratorMetadata选项

然后我们来看这个库的index.d.ts, 这是一个依赖中更新最即时的API文档:

```typescript
export {}
declare global {
    namespace Reflect{}
}
```

表示该库没有任何的导出, 再往下看是对global和命名空间的声明, 于是我们在项目中直接使用`import 'reflect-metadata'`引入后便可在项目中访问到。
下面我们介绍它支持的方法：

#### 1、decorate

decorate方法的主要作用是装饰，有好几个重载，下面分别介绍：
(1)**对类的装饰**

```typescript
/**
  * 应用于一个装饰器的集合给目标对象
  * @param decorators  装饰器的数组
  * @param target 目标对象
  * @returns 返回应用提供的装饰器后的值
  * 注意:装饰器应用是与array的位置方向相反, 为从右往左
  */
function decorate(decorators: ClassDecorator[], target: Function): Function;
```

那么这个ClassDecorator[]究竟是个什么数组? 

```typescript
declare type ClassDecorator = <TFunction extends Function>(target: TFunction) => TFunction | void;
```

该定义为: 这是一个Function, 只有一个参数Target, 也就是类的构造函数constructor, 返回值的类型与target的函数类型一致或为空, 也就是说, 如果是一个类的话, 需返回这个类或者空. 注意, 该类型的定义不在reflect-metadata中, 而是在`lib.es5.d.ts`中, 也就表明该为es5的原生实现。
举个🌰:

```typescript
const classDecorator: ClassDecorator = target => {
  target.prototype.sayName = () => console.log('override');
}
export class TestClassDecorator {
  name: string;
  constructor(name: string) {
    this.name = name;
  }
  sayName() {
    console.log(this.name)
  }
}
Reflect.decorate([classDecorator], TestClassDecorator) // 对其进行装饰
const t = new TestClassDecorator('nihao')
t.sayName() // 'override'
```

**注意**: 在classDecorator中传入的target, 只能修改其prototype的方法, 不能修改其属性, 因为其属性是`read-only`。
(2)**对属性或方法装饰**

```typescript
/**
  * 应用一个集合去装饰目标对象的方法
  * @param 装饰器的集合
  * @param 目标对象
  * @param 要装饰的属性名称 key
  * @param 该属性的描述 descriptor
  */
  function decorate(decorators: (PropertyDecorator | MethodDecorator)[], target: Object, propertyKey: string | symbol, attributes?: PropertyDescriptor : PropertyDescriptor;
```

举个🌰:

```typescript
// 属性装饰器
const propertyDecorator: PropertyDecorator = (target, propertyKey) => {
  const origin = target[propertyKey];
  target[propertyKey] = () => {
    origin.call(target)
    console.log('added override')
  }
}

class PropertyAndMethodExample {
    static staticProperty() {
        console.log('im static property')
    }
    method() {
        console.log('im one instance method')
    }
}

Reflect.decorate([propertyDecorator], PropertyAndMethodExample, 'staticProperty')
// test property decorator
PropertyAndMethodExample.staticProperty() // im static property \n added override

// 方法装饰器
const methodDecorator: MethodDecorator = (target, propertyKey, descriptor) => {
  // 将其描述改为不可编辑
  descriptor.configurable = false
  descriptor.writable = false
  return descriptor
}
// 获取原descriptor
let descriptor = Object.getOwnPropertyDescriptor(PropertyAndMethodExample.prototype, 'method')
// 获取修改后的descriptor
descriptor = Reflect.decorate([methodDecorator], PropertyAndMethodExample, 'method', descriptor)
// 将修改后的descriptor添加到对应的方法上
Object.defineProperty(PropertyAndMethodExample.prototype, 'method', descriptor)
// test method decorator
const example = new PropertyAndMethodExample
example.method = () => console.log('override') // 报错 Cannot assign to read only property 'method' of object '#<PropertyAndMethodExample>'
```

#### 2、metadata

默认的元数据装饰器可以被用于类, 类成员以及参数：

```typescript
/**
* @param metadataKey 元数据入口的key
* @param metadataValue 元数据入口的value
* @returns 装饰器函数
*/
function metadata(metadataKey: any, metadataValue: any): {
    (target: Function): void;
    (target: Object, propertyKey: string | symbol): void;
};
```

举个🌰:

```typescript
const nameSymbol = Symbol('lorry')
// 类元数据
@Reflect.metadata('class', 'class')
class MetaDataClass {
  // 实例属性元数据
  @Reflect.metadata(nameSymbol, 'kylee')
  public name = 'origin'
  // 实例方法元数据
  @Reflect.metadata('getName', 'getName')
  public getName () {
    console.log('get name');
  }
  // 静态方法元数据
  @Reflect.metadata('static', 'static')
  static staticMethod () {
    console.log('static method');
  }
}
const value = Reflect.getMetadata('class', MetaDataClass); // "class"
const metadataInstance = new MetaDataClass
const name = Reflect.getMetadata(nameSymbol, metadataInstance, 'name'); // "kylee"
const methodVal = Reflect.getMetadata('getName', metadataInstance, 'getName'); // "getName"
const staticVal = Reflect.getMetadata('static', MetaDataClass, 'staticMethod'); // "static"
```

**注意**: 如果metadataKey已经被定义在target或target key, 那么metadataValue将会被覆盖

#### 3、defineMetadata

该方法是metadata的定义版本, 也就是非@版本, 会多传一个参数target, 表示待装饰的对象

```typescript
/**
  * @param metadataKey 设置或获取时的key
  * @param metadataValue 元数据内容
  * @param target 待装饰的target
  * @param targetKey target的property
*/
function defineMetadata(metadataKey: any, metadataValue: any, target: Object, targetKey: string | symbol): void;
```

举个🌰:

```typescript
class DefineMetadata {
  static staticMethod() {
    console.log('static method');
  }
  static staticProperty = 'static'
  getName() {
    console.log('get name');
  }
}
const type = 'type'
Reflect.defineMetadata(type, 'class', DefineMetadata)
Reflect.defineMetadata(type, 'staticMethod', DefineMetadata.staticMethod)
Reflect.defineMetadata(type, 'method', DefineMetadata.prototype.getName)
Reflect.defineMetadata(type, 'staticProperty', DefineMetadata, 'staticProperty')
const t1 = Reflect.getMetadata(type, DefineMetadata); // class
const t2 = Reflect.getMetadata(type, DefineMetadata.staticMethod); // staticMethod
const t3 = Reflect.getMetadata(type, DefineMetadata.prototype.getName); // method
const t4 = Reflect.getMetadata(type, DefineMetadata, 'staticProperty'); // staticProperty 
```

注意t4定义和获取不一样的地方, 比如t2到t3都有两种写法, 一种就是将target传为对应的对象且必须为对象.以t2为例, 也可以写为

```typescript
Reflect.defineMetadata(type, 'staticMethod', DefineMetadata, 'staticMethod');
const t2 = Reflect.getMetadata(type, DefineMetadata, 'staticMethod');
```

需要注意的是这两者并不能混合使用, 比如:

```typescript
Reflect.defineMetadata(type, 'staticMethod', DefineMetadata, 'staticMethod')
const t2 = Reflect.getMetadata(type, DefineMetadata.staticMethod)
```

是无法获取到对应的metadataValue的, 原因是如果未传入property, 会当property当作undefined, 此时该target的`metadata entries`(本质是一个Map)中就有一个`{undefined: Map()}`而传入了property, 就是在该target下的property属性下的entries set一个`{property: Map()}`
具体拿t2来说
方法1

```typescript
Reflect.defineMetadata(type, 'staticMethod', DefineMetadata.staticMethod)
```

这条语句是在DefineMetadata.staticMethod下的元数据为:

<img src="/assets/typescript-01/03.png" width="600" />

方法2

```typescript
Reflect.defineMetadata(type, 'staticMethod', DefineMetadata, 'staticMethod')
```

元数据为:

<img src="/assets/typescript-01/04.png" width="600" />

明显这两者不等价, 所以无法互换。除此之外，@Reflect.metadata的行为跟设置property一致, 也就是为方法2的实现, 使用方法1获取@Reflect.metadata的数据会为undefined。

#### 4、hasMetadata

该方法返回布尔值, 表明该target或其原型链上有没有对应的元数据

```typescript
/**
  * @param metadataKey 元数据的key.
  * @param target 定义的对象
  * @param targetKey 重载参数, 可选
  * @returns 在target或其原型链上返回true.
*/
function hasMetadata(metadataKey: any, target: Object, targetKey?: symbol | string): boolean;
```

举个🌰:

```typescript
const type = 'type'
class HasMetadataClass {
  @Reflect.metadata(type, 'staticProperty')
  static staticProperty = ''
}
Reflect.defineMetadata(type, 'class', HasMetadataClass);
const t1 = Reflect.hasMetadata(type, HasMetadataClass); // true
const t2 = Reflect.hasMetadata(type, HasMetadataClass, 'staticProperty'); // true
```

其余的像实例属性/方法, 静态方法都以此类推。

#### 5、hasOwnMetadata

跟`Object.prototype.hasOwnProperty`类似, 是只查找对象上的元数据, 而不会继续向上查找原型链上的, 其余的跟hasMetadata一致，举个🌰:

```typescript
const type = 'type'
class Parent {
  @Reflect.metadata(type, 'getName')
  getName() {}
}
@Reflect.metadata(type, 'class')
class HasOwnMetadataClass extends Parent{
  @Reflect.metadata(type, 'static')
  static staticProperty() {}
  @Reflect.metadata(type, 'method')
  method() {}
}

const t1 = Reflect.hasOwnMetadata(type, HasOwnMetadataClass); // true
const t2 = Reflect.hasOwnMetadata(type, HasOwnMetadataClass, 'staticProperty'); // true
const t3 = Reflect.hasOwnMetadata(type, HasOwnMetadataClass.prototype, 'method'); // true
const t4 = Reflect.hasOwnMetadata(type, HasOwnMetadataClass.prototype, 'getName'); // false
const t5 = Reflect.hasMetadata(type, HasOwnMetadataClass.prototype, 'getName'); // true
```

#### 6、getMetadata

这个属性在之前验证各个属性的时候就已经使用过了, 就是用于获取target的元数据值, 会往原型链上找

```typescript
/**
* @param metadataKey 元数据key
* @param target 元数据定义的target
* @param targetKey 可选项, 是否选择target的某个key
* @returns 如果找到了元数据则返回元数据值, 否则返回undefined
*
*/
function getMetadata(metadataKey: any, target: Object, targetKey?: string | symbol ): any;
```

#### 7、getOwnMetadata

与hasOwnMetadata和hasMetadata的区别一样, 其与getMetadata之间的主要区别就在于是否往原型链上找。

#### 8、getMetadataKeys

类似Object.keys, 返回该target以及原型链上target的所有元数据的keys

```typescript
const type = 'type'
@Reflect.metadata('parent', 'parent')
class Parent {
  getName() {}
}
@Reflect.metadata(type, 'class')
class HasOwnMetadataClass extends Parent{
  @Reflect.metadata(type, 'static')
  static staticProperty() {}
  @Reflect.metadata('bbb', 'method')
  @Reflect.metadata('aaa', 'method')
  method() {}
}

const t1 = Reflect.getMetadataKeys(HasOwnMetadataClass); // ["type", "parent"]
const t2 = Reflect.getMetadataKeys(HasOwnMetadataClass.prototype, 'method'); // ["aaa", "bbb"]
```

t1很好理解, 因为会向上找原型链的Parent;
t2好像多了一些东西, `design:`开头的, 这个在后面会介绍, 看看'aaa'和'bbb'的顺序是和我们添加的顺序是相反的, 还记得前面说过装饰器的顺序是从右到左的, 所以先应用的bbb, aaa再应用的`design:xxx`。

#### 9、getOwnMetadataKeys

跟getMetadataKeys 一样, 只是不向原型链中查找。

#### 10、deleteMetadata

用于删除元数据

```typescript
/**
* @param metadataKey 元数据key
* @param target 元数据定义的对象
* @param targetKey 对象对应的key
* @returns 如果对象上有该元数据, 返回true, 否则返回false
*/
function deleteMetadata(metadataKey: any, target: Object, targetKey?:symbol|string): boolean;
```

举个🌰:

```typescript
const type = 'type'
@Reflect.metadata(type, 'class')
class DeleteMetadata {
  @Reflect.metadata(type, 'static')
  static staticMethod() {}
}

const res1 = Reflect.deleteMetadata(type, DeleteMetadata); // true
const res2 = Reflect.deleteMetadata(type, DeleteMetadata, 'staticMethod'); // true
const res3 = Reflect.deleteMetadata(type, DeleteMetadata); // false
```

## 类型检查机制

类型检查机制是指TypeScript在做类型检查时所秉承的一些原则，以及表现出的一些行为，其作用主要是辅助开发，提升开发效率。TypeScript的类型检查机制主要包括如下三个部分：

- 类型推断
- 类型兼容性
- 类型保护

### 一、类型推断

类型推断指的是有时候我们不需要指定变量的类型（函数的返回值类型），TypeScript可以根据某些规则自动地为其推断出某些类型，类型推断主要分为如下三部分内容：

- 基础类型推断
- 最佳通用类型推断
- 上下文类型推断

#### 1、基础类型推断

基础类型推断也是TypeScript出现最普遍的类型推断，通常发生在以下场景：初始化变量、设置函数默认参数、确定函数返回值，举个🌰:

```typescript
let a; // 自动推断为any类型
let b = 1; // 自动推断为number类型

let fuc1 = (c: 1) => {} // c被自动推断为number类型

let func2 = (x: 1) => x + 1 // func2被推断为(x?: number) => number类型
```

#### 2、最佳通用类型推断

当需要从几个表达式中推断类型的时候，会使用这些表达式的类型来推断出一个最合适的通用类型。默认情况下，会查看所有的表达式，检查它们是否有共同的类型，比如是否都有同一个父类。如果有，就会被推断为该类型
如果没有共同的类型，比如全部是基础类型，默认会使用联合类型，形如例如`string | number`。举个🌰:

```typescript
let arr = [1, '1'] // arr被自动推断为[number | string]类型
```

#### 3、上下文类型推断

上述两种类型推断都是从右向左的类型推断，即根据右侧表达式的值推测左侧表达式的类型，而上下文类型推断的方向与其是恰好相反的，它通常发生在事件处理中，举个🌰:

```typescript
// (parameter) event: KeyboardEvent
window.document.onkeydown = (event) => {}
```

在上述的示例中就发生了上下文类型推断，TypeScript会根据左侧的事件绑定来推断出右侧的事件类型，在这儿event就被推断成了KeyboardEvent。

### 二、类型兼容性

当一个类型Y可以被赋值给另一个类型X时我们就可以说类型X兼容类型Y，即
X兼容Y：X(目标类型) = Y(源类型)
举个🌰，当我们关闭tsconfig.json中的strictNullChecks选项时我们可以将null赋值给其他基本类型：

```typescript
let s: string = 'a';
s = null;
```

此时我们也可以说字符类型是兼容null类型的，或null类型是字符类型的子类型，之所以我们在此讨论类型兼容性是因为TypeScript是允许不同类型的变量间的相互赋值，虽然在某种程度上讲这会造成一定程度不可靠的行为，但是极大程度上提高了代码的灵活性。类型兼容性的例子广泛存在于接口、函数和类中，下面我们分别进行介绍：

#### 1、接口兼容性

```typescript
interface X {
  a: any;
  b: any
}
interface Y {
  a: any;
  b: any;
  c: any
}

let x: X = { a: 1, b: 2 }
let y: Y = { a: 1, b: 2, c: 3 }

x = y // ok
y = x // Property 'c' is missing in type 'X' but required in type 'Y'.
```

在上述示例中我们可以看到如果接口Y具备X接口的所有属性，也就是属性a和b，及时拥有额外的属性c也依然可以被认为是X类型，也就是说X类型兼容Y类型，这里再次体现了Typescript的类型检查原则，即鸭式辩形法。

#### 2、函数兼容性

函数的兼容性与接口不同，函数之间要实现兼容必须要具备如下三个条件：
(1)**参数个数**
如果一个函数x的参数列表中，每一个参数的位置和类型，都能在另一个函数y的参数列表中一一对应（y中多余的参数不做限制），那么x就是对y参数类型兼容的。简单来说就是，参数需求少的函数，允许在提供更多的参数的函数类型上使用。典型的应用场景就是，Array的原型方法，map, forEach等等的回调目标函数中，它们被定义为接收三个参数，当前遍历的元素，元素索引，整个数组，但通常提供只接收前几个的参数的回调源函数。举个🌰:

```typescript
type Handler = (a: number, b: number) => void
function func(handler: Handler) {
  return handler
}
// 当我们给func函数传入参数时TypeScript引擎就会判断传入参数与Handler类型是否兼容，因此在本例中传入参数为源类型，Handler类型为目标类型，因此此处传入源函数的参数个数必须少于两个
let handler1 = (a: number) => {}
func(handler1)
let handler2 = (a: number, b: number, c: number) => {}
func(handler2) // 类型"(a: number, b: number, c: number) => void"的参数不能赋给类型"Handler"的参数
```

上述的示例展示的是固定参数的情形，当函数中拥有可选参数或剩余参数时兼容原则却又有所不同，举个🌰:

```typescript
let a = (p1: number, p2: number) => {}
let b = (p1?: number, p2?: number) => {}
let c = (...args: number[]) => {}

// 固定参数可以兼容剩余参数与可选参数
a = b;
a = c;

// 可选参数不兼容固定参数与剩余参数
b = a; // Type 'number | undefined' is not assignable to type 'number'.
b = c; // Type 'number | undefined' is not assignable to type 'number'.
// 注：此时我们可以将tsconfig.json中的strictFunctionTypes属性设为false来使其兼容

// 剩余参数可以兼容固定参数与可选参数
c = a;
c = b;
```

(2)**参数类型**
如果参数为基本类型，源函数类型的参数必须与目标函数类型相同或为其子类型，举个🌰:

```typescript
let handler3 = (a: string) => {}
func(handler3) //  Type 'number' is not assignable to type 'string'.
```

当参数类型为对象时情况有所不同，举个🌰:

```typescript
interface Point3D {
  x: number;
  y: number;
  z: number;
}

interface Point2D {
  x: number;
  y: number;
}

let p3d = (point: Point3D) => {}
let p2d = (point: Point2D) => {}

p3d = p2d; // ok
p2d = p3d; // Property 'z' is missing in type 'Point2D' but required in type 'Point3D'.
// 注：此时我们依然可以将tsconfig.json中的strictFunctionTypes属性设为false来使其兼容
```

我们可以观察到当函数参数类型为对象且一个对象的每一个属性都可以在另一个对象中找到时属性个数多的兼容属性个数少的，这正好与我们之前接口之间的兼容恰好相反。
(3)**返回值类型**
一个函数x的返回值类型，如果是另一个函数的返回值类型的子类型，即x的返回值对象的每一个属性都能在y返回值中找到（y中多余的属性不做限制），那么x是对y返回值类型兼容的。简单来说就是，提供返回值更少的兼容更多的，举个🌰:

```typescript
let x = () => ({ name: 'Double' });
let y = () => ({ name: 'Float', location: 'Home' });

// x的返回值的属性，在y中全存在
x = y;
```

在了解了函数兼容性的前提下我们再来看函数重载：

```typescript
function overload(a: number, b: number): number;
function overload(a: string, b: string): string;
function overload(a: any, b: any): any {};
```

在文章的前面我们已经介绍过，函数的重载分为两个部分，分别是函数重载的列表和函数的具体实现，这里重载列表中的函数就是目标函数，而具体的实现就是源函数，函数在运行的时候编译器会查找重载列表，然后使用第一个匹配的定义来执行下面的函数，所以在函数重载中实现函数的参数个数必须少于或等于重载列表中的函数参数的个数，而且返回值类型也要符合相应的要求。

#### 3、枚举类型兼容性

(1)**数字枚举类型与数字类型完全兼容**

```typescript
enum Fruit { Apple, Banana }
let fruit: Fruit.Apple = 3;
let num: number = Fruit.Banana;
```

(2)**字符类型兼容字符串枚举类型**

```typescript
enum Select { NoA = 'AAA', NoB = 'BBB' }

let str: string = Select.NoA;
let a: Select.NoA = 'AAA'; // Type '"AAA"' is not assignable to type 'Select.NoA'.
```

(3)**不同枚举类型之间互不兼容**

```typescript
enum Fruit { Apple, Banana }
enum Color { Red, Green, Blue }

let val: Fruit.Apple = Color.Blue; // Type 'Color.Blue' is not assignable to type 'Fruit.Apple'.
```

#### 4、类兼容性

类的兼容性与接口类似，它们也是只比较结构，不过需要注意的是在比较两个类是否兼容的时候静态成员与构造函数是不参与比较的，如果两个类具有相同的实例成员，那它们的实例就可以完全相互兼容，举个🌰:

```typescript
class A {
  constructor(p: number, q: number) {}
  id: number = 1
}
class B {
  static s = 1
  constructor(p: number) {};
  id: number = 3
}

let aa = new A(1, 2);
let bb = new B(1);

aa = bb // ok
bb = aa // ok
```

不过需要注意如果类中存在私有成员，无论这两个成员属性是否相等，两者都不再兼容，这个时候只有父类与子类之间相互兼容。举个🌰:

```typescript
class A {
  constructor(p: number, q: number) {}
  id: number = 1
  private name: string = ''
}
class B {
  static s = 1
  constructor(p: number) {};
  id: number = 3
  private name: string = ''
}

let aa = new A(1, 2);
let bb = new B(1);

aa = bb // Types have separate declarations of a private property 'name'.
bb = aa // Types have separate declarations of a private property 'name'.

class C extends A {}
let cc = new C(1, 2);
aa = cc // ok
cc = aa // ok
```

#### 5、泛型兼容性

在使用泛型时，我们通常是传入一个类型，返回一个泛型包装的结果类型。也就是说，真正影响到后续程序和运算的，是结果类型。因此，泛型的类型兼容性判断上，仅考虑结果类型，即便泛型名称，泛型变量不同，只要结果类型一致，那么它们就是类型兼容的。举个🌰:

```typescript
interface GenericInterface<T> {
    data: T
}

let x: GenericInterface<number> = { data: 1 };
let y: GenericInterface<string> = { data: '1' };
x = y; // Type 'GenericInterface<string>' is not assignable to type 'GenericInterface<number>'.

interface EmptyInterface<T> { }

let a: EmptyInterface<number> = 1
let b: EmptyInterface<string> = '1'

a = b;
```

下面我们来看泛型函数：

```typescript
let log1 = <T>(x: T): T => {
  console.log(x);
  return x;
}
let log2 = <U>(y: U): U => {
  console.log(y);
  return y;
}

log1 = log2; // ok
log2 = log1; // ok
```

从上面示例我们可以看出如果两个泛型函数的定义相同，但是没有指定类型参数，那么它们之间也是可以相互兼容的。

### 三、类型保护

所谓的类型保护就是TypeScript能够在特定的区块中保证变量属于某种确定的类型，可以在此区块中放心地引用此类型的属性，或者调用此类型的方法。下面我们介绍四种创建这种特殊区块的方法：

#### 1、instanceof关键字

instanceof关键字常用于判断实例对象属于某个类，举个🌰:

```typescript
interface Padder {
  getPaddingString(): string;
}

class SpaceRepeatingPadder implements Padder {
  constructor(private numSpaces: number) {}
  getPaddingString() {
    return Array(this.numSpaces + 1).join(" ");
  }
}

class StringPadder implements Padder {
  constructor(private value: string) {}
  getPaddingString() {
    return this.value;
  }
}

let padder: Padder = new SpaceRepeatingPadder(6);

if (padder instanceof SpaceRepeatingPadder) {
  // padder的类型收窄为 'SpaceRepeatingPadder'
}
```

#### 2、in关键字

in关键字用于判断某个属性是否属于某个对象，举个🌰:

```typescript
interface Admin {
  name: string;
  privileges: string[];
}

interface Employee {
  name: string;
  startDate: Date;
}

type UnknownEmployee = Employee | Admin;

function printEmployeeInformation(emp: UnknownEmployee) {
  console.log("Name: " + emp.name);
  if ("privileges" in emp) {
    console.log("Privileges: " + emp.privileges);
  }
  if ("startDate" in emp) {
    console.log("Start Date: " + emp.startDate);
  }
}
```

#### 3、typeof关键字

typeof关键字可以帮我们判断基本类型，举个🌰:

```typescript
function padLeft(value: string, padding: string | number) {
  if (typeof padding === "number") {
      return Array(padding + 1).join(" ") + value;
  }
  if (typeof padding === "string") {
      return padding + value;
  }
  throw new Error(`Expected string or number, got '${padding}'.`);
}
```

typeof 类型保护只支持两种形式：`typeof v === "typename"` 和 `typeof v !== typename`，"typename"必须是"number"，"string"，"boolean"或"symbol"。 但是TypeScript并不会阻止你与其它字符串比较，语言不会把那些表达式识别为类型保护。

#### 4、自定义类型保护的类型谓词

```typescript
function isNumber(x: any): x is number {
  return typeof x === "number";
}

function isString(x: any): x is string {
  return typeof x === "string";
}
```

## 高级类型

### 一、交叉类型

交叉类型写法类似于`T & U`，用于将多个类型合并为一个类型。交叉类型要求同时满足所有指定的类型的要求。也就是所有类型的并集（包含所有属性）。如果函数的返回值是交叉类型，必须做显式类型转换（类型断言）。如果几个类型中有同名属性，后面的属性值会覆盖前面的属性值。

```typescript
function extend<First, Second>(first: First, second: Second): First & Second {
    const result: First & Second = { ...first, ...second }
    return result;
}

class Cat {
    name: string;
    catchMouse: boolean;
    constructor(n: string, c: boolean) {
        this.name = n;
        this.catchMouse = c;
    }
}

class Dog {
    name: string;
    walkDog: boolean;
    constructor(n: string, w: boolean) {
        this.name = n;
        this.walkDog = w;
    }
}

const result = extend({ name: 'a cat', catchMouse: false }, { name: 'a dog', walkDog: true });

console.log(result) // { name: "a dog", catchMouse: false, walkDog: true } 
```

### 二、联合类型

联合类型(Union Types)，其取值可以为多种类型中的一种，前提是取值的类型之前定义过，举个🌰:

```typescript
let stringAndNumber: string | number;
stringAndNumber = 'str';
stringAndNumber = 123;
```

编译后

```javascript
"use strict";
let stringAndNumber;
stringAndNumber = 'str';
stringAndNumber = 123;
```

#### 1、可辨识联合类型

TypeScript 可辨识联合（Discriminated Unions）类型，也称为代数数据类型或标签联合类型。它包含 3 个要点：

- 可辨识
- 联合类型
- 类型守卫

这种类型的本质是结合联合类型和字面量类型的一种类型保护方法。如果一个类型是多个类型的联合类型，且多个类型含有一个公共属性，那么就可以利用这个公共属性，来创建不同的类型保护区块。

(1)**可辨识**

可辨识要求联合类型中的每个元素都含有一个单例类型属性，比如：

```typescript
interface Square {
  kind: 'square';
  size: number;
}

interface Rectangle {
  kind: 'rectangle';
  width: number;
  height: number;
}
```

在上述代码中，我们分别定义了 Square 和 Rectangle 两个接口，在这些接口中都包含一个 kind 属性，该属性被称为可辨识的属性，而其它的属性只跟特性的接口相关。

(2)**联合类型**
基于前面定义了两个接口，我们可以创建一个 Shape 联合类型：

```typescript
type Shape = Square | Rectangle;
```

Square 和 Rectangle有共同成员 kind，因此 kind 存在于 Shape 中。

(3)**类型守卫**

如果你使用类型保护风格的检查（==、===、!=、!==）或者使用具有判断性的属性（在这里是 kind），TypeScript 将会认为你会使用的对象类型一定是拥有特殊字面量的，并且它会为你自动把类型范围变小：

```typescript
function area(s: Shape) {
  if (s.kind === 'square') {
    // 现在 TypeScript 知道 s 的类型是 Square
    // 所以你现在能安全使用它
    return s.size * s.size;
  } else {
    // 不是一个 square ？因此 TypeScript 将会推算出 s 一定是 Rectangle
    return s.width * s.height;
  }
}
```

此处更好的编程实践是我们可以通过一个简单的向下思想，来确保块中的类型被推断为与 never 类型兼容的类型：

```typescript
function area(s: Shape) {
  if (s.kind === 'square') {
    return s.size * s.size;
  } else if (s.kind === 'rectangle') {
    return s.width * s.height;
  } else {
    const _exhaustiveCheck: never = s;
  }
}
```

我们可以通过 switch 来实现以上例子:

```typescript
function area(s: Shape) {
  switch (s.kind) {
    case 'square':
      return s.size * s.size;
    case 'rectangle':
      return s.width * s.height;
    default:
      const _exhaustiveCheck: never = s;
  }
}
```

此处需要注意如果你使用 strictNullChecks 选项来做详细的检查，你应该返回 _exhaustiveCheck 变量（类型是 never），否则 TypeScript 可能会推断返回值为 undefined：

```typescript
function area(s: Shape) {
  switch (s.kind) {
    case 'square':
      return s.size * s.size;
    case 'rectangle':
      return s.width * s.height;
    case 'circle':
      return Math.PI * s.radius ** 2;
    default:
      const _exhaustiveCheck: never = s;
      return _exhaustiveCheck;
  }
}
```
### 三、索引类型

在实际开发中，我们经常能遇到这样的场景，在对象中获取一些属性的值，然后建立对应的集合。

```js
let person = {
    name: 'musion',
    age: 35
}

function getValues(person: any, keys: string[]) {
    return keys.map(key => person[key])
}

console.log(getValues(person, ['name', age])) // ['musion', 35]
console.log(getValues(person, ['gender'])) // [undefined]
```

在上述例子中，可以看到getValues(persion, ['gender'])打印出来的是[undefined]，但是ts编译器并没有给出报错信息，那么如何使用ts对这种模式进行类型约束呢？这里就要用到了索引类型,改造一下getValues函数，通过**索引类型查询**和**索引访问**操作符：

```typescript
// T是一个任意类型，K类型是T类型中，任意一个属性的类型，形参names是K类型变量组成的数组
// 返回值 T[K][]: T类型的K属性数组（第一个方括号表示取属性，第二个表示数组类型）
function pluck<T, K extends keyof T>(o: T, names: K[]): T[K][] {
    return names.map(n => o[n]);
}

interface Person {
    name: string;
    age: number;
}

const person: Person = {
    name: 'kylee',
    age: 17
}

const stars: string[] = pluck(person, ['name']);
console.log(`Person Name: ${stars}`); // "Person Name: kylee" 
```

上述示例其实主要分为三部分：

#### 1、索引类型查询操作符（keyof T）

`keyof T`的含义表示类型T所有公共属性的字面量的联合类型，在上述例子中，就是Person的属性名，即['name', age]

#### 2、索引访问操作符（T[K]）

`T[K]`表示对象T的属性K所表示的类型，在上述例子中，`T[K][]`表示变量T取属性K的值的数组

#### 3、泛型约束（K extends T）

泛型变量可以通过继承某些类型获取某些属性

这三个特性组合保证了代码的动态性和准确性，也让代码提示变得更加丰富了，此时我们再传入不属于对象person的属性TypeScript编译器就会报错：

```typescript
// Argument of Type '"gender"[]' is not assignable to parameter of type '("name" | "age")[]'.
// Type "gender" is not assignable to type "name" | "age".
getValues(person, ['gender'])
```

### 四、映射类型

TypeScript提供了从旧类型中创建新类型的一种方式 — 映射类型。 在映射类型里，新类型以相同的形式去转换旧类型里每个属性。TS内置了一些映射类型（实际上就是TypeScript泛型中内置的一些泛型工具，例如Readonly、Partial、Record、Pick和Required）来便于我们进行类型映射，我们根据映射过程中是否引入新属性又将其分为同态映射类型与非同态映射类型。

#### 1、同态映射

```typescript
interface Person {
    name: string;
    age: number;
}

// 因为所有的映射都是发生在类型T之上的，没有别的变量和属性参与，因此属于同态映射

const pPartial: Partial<Person> = { name: 'only name' }
const pReadonly: Readonly<Person> = { name: 'const name', age: 32 }
const pPick: Pick<Person, 'age'> = { age: 12 }
const pRequired: Required<Person> = { name: 'required name', age: 22 }
```

#### 2、非同态映射

```typescript
interface Obj {
    a: number
    b: string
    c: boolean
}
// 映射出的新类型所具有的属性由Record的第一个属性指定，而这些属性类型为第二个参数指定的已知类型，这种类型属性就是一种非同态的类型
type RecordObj = Record<'x' | 'y', Obj>
```

### 五、条件类型

Typescript在2.8版本新增了条件类型，条件类型可以让我们描述不一致的映射类型，也就是说，它是一种取决于条件的类型转换。条件类型描述一种类型关系的测试并且选择两种可能的类型中的一种，它取决于条件测试的结果。它一般有如下的形式：`T extends U ? X : Y`。条件类型使用与`...?...:...`类似的语法，这种语法跟 Javascript 里面的三元运算符相似，这里的 T、U、X、Y 表示任意的类型。`T extends U`部分描述类型关系的测试，如果这个条件满足，类型 X 会被选中，否则类型 Y 会被选中，举个🌰:

```typescript
type TypeName<T> = 
  T extends string ? string :
  T extends number ? number :
  T extends boolean ? boolean :
  undefined

type T1 = TypeName<string> // string
type T2 = TypeName<number> // number
type T3 = TypeName<string[]> // undefined

const val1: T1 = 'as'
const val2: T2 = 12
const val3: T3 = undefined
```

#### 1、分布的条件类型

如果这个 T 是联合类型的情形下结果类型也是多个类型的联合类型，即`(A | B) extends U ? X : Y`可以被拆解为`A extends U ? X : Y | B extends U ? X : Y`，这种情形也被成为分布的条件类型，举个🌰:

```typescript
type T4 = TypeName<string | number> // string | number
```

这个特性引入的重要意义在于结合Never类型可以帮助我们实现一些类型的过滤，举个🌰:

```typescript
type Diff<T, U> = T extends U ? never : T

type T5 = Diff<"a" | "b" | "c", "a" | "d"> // "b" | "c"
```

#### 2、内置的条件类型

为了便于使用者进行条件判断TypeScript内置了一些条件类型（实际上还是TypeScript泛型中内置的一些泛型工具，比如NonNullable、Extract、Exclude、ReturnType、Parameters、ConstructorParameters、InstanceType），举个🌰:

```typescript
type T6 = Exclude<"a" | "b" | "c", "a" | "d"> // "b" | "c"
type T7 = NonNullable<string | number | null | undefined> // string | number
type T8 = Extract<"a" | "b" | "c", "a" | "d"> // "a"
type T9 = ReturnType<(x: string) => number> // number
type T10 = Parameters<(x: string, y: number) => number> // [x: string, y: number]
type T11 = ConstructorParameters<ErrorConstructor> // [message?: string | undefined]
type T12 = InstanceType<ErrorConstructor>;    // Error
```

## 参考资料
极客时间《TypeScript开发实战》专栏
《深入理解TypeScript》
[typeScript 中的type关键字](https://juejin.cn/post/6876359681464336397)
[一文读懂 TS 中 Object, object, {} 类型之间的区别](http://www.semlinker.com/ts-object-type/)
[一份不可多得的 TS 学习指南（1.8W字）](https://juejin.im/post/6872111128135073806#heading-21)
[Typescript使用手册](https://www.bookstack.cn/read/TypeScript-3.6/doc-handbook-README)
[TS一些工具泛型的使用及其实现](https://zhuanlan.zhihu.com/p/40311981)
[reflect-metadata的研究](https://juejin.cn/post/6844904152812748807)
