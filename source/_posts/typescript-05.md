---
title: TypeScript学习笔记（四）
date: 2020-11-23 10:27:51
toc: true
categories: 
- 前端
tags: 
- TypeScript
---

## 泛型

泛型（Generics）在编程语言中是一个较为普遍的概念，在像 C# 和 Java 这样的语言中，可以使用泛型来创建可重用的组件，一个组件可以支持多种类型的数据。 这样用户就可以以自己的数据类型来使用组件。这给软件工程带来了极高的灵活性，进一步提高了组件或函数的可重用性。那么泛型具体的定义是什么呢？
泛型是指不预先确定的数据类型，具体的类型在使用的时候才能确定，它允许同一个函数可以接受不同类型参数的一个模板。设计泛型的关键目的是在成员之间提供有意义的约束，这些成员可以是：类的实例成员、类的方法、函数参数和函数返回值。

<!-- more -->

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

## 参考资料

极客时间《TypeScript开发实战》专栏

《深入理解TypeScript》

[一份不可多得的 TS 学习指南（1.8W字）](https://juejin.im/post/6872111128135073806#heading-21)

[Typescript使用手册](https://www.bookstack.cn/read/TypeScript-3.6/doc-handbook-README)

[TS一些工具泛型的使用及其实现](https://zhuanlan.zhihu.com/p/40311981)