---
title: TypeScript学习笔记（六）
date: 2020-11-26 10:41:45
toc: true
mathjax: false
categories: 
- 前端
tags: 
- TypeScript
---

## 类型检查机制

类型检查机制是指TypeScript在做类型检查时所秉承的一些原则，以及表现出的一些行为，其作用主要是辅助开发，提升开发效率。TypeScript的类型检查机制主要包括如下三个部分：

- 类型推断
- 类型兼容性
- 类型保护

<!-- more -->

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

## 参考资料

极客时间《TypeScript开发实战》专栏

《深入理解TypeScript》
