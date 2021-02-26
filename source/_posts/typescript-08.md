---
title: TypeScript学习笔记（七）
date: 2020-11-28 18:44:45
toc: true
categories: 
- 前端
tags: 
- TypeScript
---

## 高级类型

### 一、交叉类型

交叉类型写法类似于`T & U`，用于将多个类型合并为一个类型。交叉类型要求同时满足所有指定的类型的要求。也就是所有类型的并集（包含所有属性）。如果函数的返回值是交叉类型，必须做显式类型转换（类型断言）。如果几个类型中有同名属性，后面的属性值会覆盖前面的属性值。

<!-- more -->

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
[一份不可多得的 TS 学习指南（1.8W字）](https://juejin.im/post/6872111128135073806#heading-21)
[Typescript使用手册](https://www.bookstack.cn/read/TypeScript-3.6/doc-handbook-README)
