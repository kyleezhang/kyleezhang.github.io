---
title: TypeScript学习笔记(四)
date: 2020-11-30 10:33:06
toc: true
tags:
- typescript
---
书接上文[TypeScript学习笔记（三）](https://kyleezhang.github.io/2020/11/15/typescript-03/#more)
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

##### 1、type的应用和interface的区别
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
[一份不可多得的 TS 学习指南（1.8W字）](https://juejin.im/post/6872111128135073806#heading-21)
[Typescript使用手册](https://www.bookstack.cn/read/TypeScript-3.6/doc-handbook-README.md)