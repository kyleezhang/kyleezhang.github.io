---
title: TypeScript学习笔记（二）——断言、类型别名与函数
date: 2020-11-17 10:05:53
toc: true
mathjax: false
categories:
- TypeScript 学习笔记
tags: 
- TypeScript
---

# 断言

## 一、类型断言

有时候你会遇到这样的情况，你会比 TypeScript 更了解某个值的详细信息。通常这会发生在你清楚地知道一个实体具有比它现有类型更确切的类型。
通过类型断言这种方式可以告诉编译器，“相信我，我知道自己在干什么”。类型断言好比其他语言里的类型转换，但是不进行特殊的数据检查和解构。它没有运行时的影响，只是在编译阶段起作用。
类型断言有两种形式：

<!-- more -->

### 1、“尖括号”语法

```typescript
let someValue: any = "this is a string";
let strLength: number = (<string>someValue).length;
```

### 2、as语法

```typescript
let someValue: any = "this is a string";
let strLength: number = (someValue as string).length;
```

## 二、非空断言

在上下文中当类型检查器无法断定类型时，一个新的后缀表达式操作符 ! 可以用于断言操作对象是非 null 和非 undefined 类型。具体而言，x! 将从 x 值域中排除 null 和 undefined。
那么非空断言操作符到底有什么用呢？下面我们先来看一下非空断言操作符的一些使用场景：

### 1、忽略 undefined 和 null 类型

```typescript
function myFunc(maybeString: string | undefined | null) {
  // Type 'string | null | undefined' is not assignable to type 'string'.
  // Type 'undefined' is not assignable to type 'string'. 
  const onlyString: string = maybeString; // Error
  const ignoreUndefinedAndNull: string = maybeString!; // Ok
}
```

### 2、调用时忽略 undefined 类型

```typescript
type NumGenerator = () => number;

function myFunc(numGenerator: NumGenerator | undefined) {
  // Object is possibly 'undefined'.(2532)
  // Cannot invoke an object which is possibly 'undefined'.(2722)
  const num1 = numGenerator(); // Error
  const num2 = numGenerator!(); // OK
}
```

因为 ! 非空断言操作符会从编译生成的 JavaScript 代码中移除，所以在实际使用的过程中，要特别注意。比如下面这个例子：

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

虽然在 TS 代码中，我们使用了非空断言，使得 `const b: number = a!;` 语句可以通过 TypeScript 类型检查器的检查。但在生成的 ES5 代码中，! 非空断言操作符被移除了，所以在浏览器中执行以上代码，在控制台会输出 undefined。

## 三、确定赋值断言

在 TypeScript 2.7 版本中引入了确定赋值断言，即允许在实例属性和变量声明后面放置一个 ! 号，从而告诉 TypeScript 该属性会被明确地赋值。为了更好地理解它的作用，我们来看个具体的例子：

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

通过 `let x!: number;` 确定赋值断言，TypeScript 编译器就会知道该属性会被明确地赋值。


# 类型别名

TypeScript 引入了类型别名 type，其作用就是给类型起一个新名字，可以作用于原始值（基本类型）、联合类型、元组以及其它任何你需要手写的类型：

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

## 一、type的应用和interface的区别

### 1、和接口一样，用来描述对象或函数的类型

```typescript
type User = {
  name: string
  age: number
};
type SetUser = (name: string, age: number) => void;
```

在 ts 编译成 js 后，所有的接口和 type 都会被擦除掉。

### 2、扩展和实现

接口可以扩展，但 type 不能 extends 和 implement，但是 type 可以通过交叉类型实现 interface 的 extends 行为。interface 可以 extends type，同时 type 也可以与 interface 类型交叉，举个🌰:

```typescript
// interface 扩展 type
interface Name {
  name: string;
}
interface User extends Name {
  age: number;
}
let stu: User = { name:'wang', age: 10 }

// 上面的扩展可以用type交叉类型来实现
type Name = {
  name: string;
}
type User = Name & { age: number  };
let stu: User = { name: 'wang',age: 1 };
console.log(stu) // { name: 'wang', age: 1 }
```

### 3、接口的声明合并

接口可以定义多次，并将被视为单个接口（即所有声明属性的合并）。而 type 不可以定义多次。

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

### 4、映射类型

type 如果定义的是联合类型能使用 in 关键字生成映射类型，但 interface 不行。

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

# 函数

## 一、TypeScript中函数与JavaScript中的区别

| TypeScript | JavaScript |
|--|--|
| 含有类型 | 无类型 |
| 箭头函数 | 剪头函数（ES2015）|
| 函数类型 | 无函数类型 |
| 必填参数和可选参数 | 所有参数都是可选的 |
| 默认参数 | 默认参数 |
| 剩余参数 | 剩余参数 |
| 函数重载 | 无函数重载 |

## 二、函数的定义

TypeScript 中函数的定义方式有如下几种：

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

// 函数构造方法
let greet5 = new Function('name', 'return "hello " + name')
```

但需要注意函数构造方法并不被推荐，当我们用这种方式编写函数实例我们发现其类型为 `Function`，具体表现为一个可调用对象，具有 `Function.prototype` 的所有原型方法。但是这里并没有体现出参数和返回值的类型，因此可以使用任何参数类型调用函数，因此并不是类型安全的。

## 三、可选参数与默认参数

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

## 四、剩余参数

```typescript
function push(array, ...items) {
  items.forEach(function (item) {
    array.push(item);
  });
}

let a = [];
push(a, 1, 2, 3);
```

一个函数最多只能有一个剩余参数，而且必须位于参数列表的最后。

## 五、注解 this 的类型

TypeScript 支持通过在函数的第一个参数中声明 this 类型的方式对 this 的使用符合预期，举个🌰:

```typescript
function fancyDate(this: Date) {
  return `${this.getDate()}/${this.getMonth()}/${this.getFullYear()}`
}
```

这儿的 this 不是常规的参数，而是保留字，是函数签名的一部分。

如果想强制显式注解函数中的 this 类型，可以在 `tsconfig.json` 中启用 noImpicitThis 设置（strict 模式包括 noImpicitThis 设置）。

## 六、迭代器

从概念上讲，<font color="#e1a092" size="4">迭代器</font>是一个对象，它允许我们遍历某些容器（列表、数组，...）。在 Javascript 中，这个概念转换为定义了有 next 方法的对象，该方法返回一个具有 value 和 done 属性的对象。

- value：迭代序列中的下一个值。如果存在 `when done === false`，那么它就是迭代器的返回值。
- done：布尔值。指示序列是否已完成。

在 TypeScript 中提供了如下迭代器接口：

```typescript
interface IteratorYieldResult<TYield> {
  done?: false
  value: TYield
}
interface IteratorReturnResult<TRutern> {
  done: true
  value: TRutern
}
type IteratorResult<T, TRutern = any> = IteratorYieldResult<T> | IteratorReturnResult<TRutern>

interface Iterator<T, TReturn = any, TNext = undefined> {
  // NOTE: 'next' is defined using a tuple to ensure we report the correct assignability errors in all places.
  next(...args: [] | TNext[]): IteratorResult<T, TReturn>
  return?(value?: TReturn): IteratorResult<T, TReturn> 
  throw?(e?: any): IteratorResult<T, TReturn> 
} 
```

迭代器接口中还有另外两个可选方法，return 和 throw. 基本上，return允许我们向迭代器发出信号，表明它应该完成（将 done 设置为 true）并返回其返回值，throw 允许您将错误传递给它可能知道如何处理的迭代器。

在这儿我们嗨需要提到另一个概念<font color="#e1a092" size="4">可迭代对象</font>。可迭代对象是实现了 `Symbol.interator` 方法的任何对象。这意味着对象（或它的原型链中的任何对象）必须有一个方法，由 `Symbol.iterator` 键索引，返回一个迭代器。TypeScript 中对于可迭代对象的类型描述如下：

```typescript
interface Interable<T> {
  [Symbol.iterator](): Iterator<T>
}
```

除了 Iterable 接口，我们还有一个名为 IterableIterator 的接口，表示既是可迭代对象，又是迭代器，这在描述生成器函数的时候非常有用。

```typescript
interface IterableIterator<T> extends Iterator<T> {
  [Symbol.iterator](): IterableIterator<T>
}
```

我们可以使用迭代器在二叉树上实现 BFS（广度优先遍历算法）。

首先，我们定义我们的二叉树：

```typescript
class BinaryTreeNode<T> {
  constructor(
    public value: T,
    public left?: BinaryTreeNode<T>,
    public right?: BinaryTreeNode<T>,
  ){}
  getChildren(): BinaryTreeNode<T>[] {
    const children: BinaryTreeNode<T>[] = []
    if (this.left) children.push(this.left)
    if (this.right) children.push(this.right)
  }
}

class Tree<T = unknown> implements {
  constructor(
    private root: BinaryTreeNode<T>
  ){}
  [Symbol.iterator](): Iteratro<BinaryTreeNode<T>> {
    return new BfsIterator(this.root)
  } 
}
```

这里我们实现了一个最简单的二叉树，紧接着我们来实现迭代器：

```typescript
class BfsIterator<T> implements Iterator<BinaryTreeNode<T>, number | undefined> {
  private currentRow: BinaryTreeNode<T>[];
  private currentNodeIndex: number;
  private numberOfNodes: number;
  private done: boolean;
  constructor(root: BinaryTreeNode<T>) {
    this.currentRow = [root]
    this.currentNodeIndex = 0
    this.done = false
    this.numberOfNode = 0
  }
  next(): IteratorResult<BinaryTreeNode<T>, number | undefined> {
    if (this.done) {
      return {
        done: true,
        value: undefined
      }
    }
    if (this.currentNodeIndex === this.currentRow.length) {
      this.currentRow = this.getNewRow()
      this.currentNodeIndex = 0
      if (this.currentRow.length === 0) {
       this.done = true
       return {
        done: true,
        value: this.numberOfNode
       } 
      }
    }
    const result: IteratorYieldResult<BinaryTreeNode<T>> = {
      done: false,
      value: this.currentRow[this.currentNodeIndex]
    }
    this.currentNodeIndex += 1;
    this.numberOfNodes += 1;
    return result
  }
  private getNewRow(): BinaryTreeNode<T>[] {
    return this.currentRow.map((node) => node.getChildren()).flat()
  }
}
```

## 七、生成器函数

迭代器允许我们完全控制迭代某个结构，我们可以决定是否以及何时获得迭代序列的下一个元素，同时向迭代器的消费者隐藏我们如何获取这些元素的实现细节。然而，一切都是有代价的，迭代器实现起来可能相当棘手，因为我们必须跟踪控制执行流程的状态，以便我们可以将迭代器标记为完成。生成器就是为了简化迭代器的创建流程提出的，我们首先来写一个简单的斐波那契数列生成器函数：

```typescript
function* createFibonacciGenerator() {
  let a = 1
  let b = 1
  while (true) {
    yield a;
    [a, b] = [b, a + b]
  }
}
```

生成器的用法如上所示，生成器使用 yield 关键字产出值。使用方让生成器提供下一个值时（例如，调用 next），yield 把结果发给使用方，然后停止执行，直到使用方要求提供下一个值为止。我们查看这里 createFibonacciGenerator 函数的类型为 `function createFibonacciGenerator(): Generator<number, void, unknown>`，那么 Generator 类型到底是什么呢？

```typescript
interface Generator<T = unknown, TReturn = any, TNext = unknown> extends Iterator<T, TReturn, TNext> {
  // NOTE: 'next' is defined using a tuple to ensure we report the correct assignability errors in all places.
  next(...args: [] | [TNext]): IteratorResult<T, TReturn>;
  return(value: TReturn): IteratorResult<T, TReturn>;
  throw(e: any): IteratorResult<T, TReturn>;
  [Symbol.iterator](): Generator<T, TReturn, TNext>;
}
```

大家可以发现这里的 Generator 接口与 IterableIterator 非常相像，那么它们之间有什么不同呢？[这里](https://stackoverflow.com/questions/58568399/why-are-typescripts-iterableiterator-and-generator-generics-slightly-differ)有非常全面的讨论。

要命令生成器执行我们的代码，我们只需要调用next：

```typescript
let fibonacciGenerator = createFibonacciGenerator() // Generator<number, void, unknown>
fibonacciGenerator.next() // { value: 1, done: false }
fibonacciGenerator.next() // { value: 1, done: false }
fibonacciGenerator.next() // { value: 2, done: false }
fibonacciGenerator.next() // { value: 3, done: false }
```

## 八、函数的重载

在静态类型的语言中例如 C++、Java 都有函数重载的概念，它们本质上还是使用相同名称和不同参数数量或类型的多个函数，函数重载的好处在于我们不需要为相似或相同功能的函数选择不同的名称，这样增强了函数的可读性， TypeScript 中的函数重载与 C++、Java 中的有所不同，举个🌰:

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

与类的方法的重载类似，当 TypeScript 编译器处理函数重载时，它会查找重载列表，尝试使用第一个重载定义，因此，在定义重载的时候，一定要把最精确的定义放在最前面。
需要注意的是 **TypeScript 中的函数重载没有任何运行时开销**，它只允许你记录希望调用函数的方式，并且编译器会检查其余代码，举个🌰，上述示例编译成 JavaScript 后的内容如下所示：

```javascript
"use strict";
function add(a, b) {
  if (typeof a === 'string' || typeof b === 'string') {
    return a.toString() + b.toString();
  }
  return a + b;
}
```

# 参考资料

极客时间《TypeScript开发实战》专栏

《深入理解TypeScript》

[Iterators in TypeScript](https://dev.to/gsarciotto/iterators-in-typescript-1d78)

[Generators in Typescript](https://dev.to/gsarciotto/generators-in-typescript-4b37ß)

[一份不可多得的 TS 学习指南（1.8W字）](https://juejin.im/post/6872111128135073806#heading-21)

[Typescript使用手册](https://www.bookstack.cn/read/TypeScript-3.6/doc-handbook-README)