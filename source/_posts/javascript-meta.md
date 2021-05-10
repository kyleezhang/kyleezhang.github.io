---
title: JavaScript中的元编程
date: 2021-01-11 10:44:29
toc: true
categories: 
- 前端
tags: 
- JavaScript
---

什么是元编程？

维基百科对其的定义如下：

>元编程（英语：Metaprogramming），又译超编程，是指某类计算机程序的编写，这类计算机程序编写或者操纵其它程序（或者自身）作为它们的资料，或者在运行时完成部分本应在编译时完成的工作。多数情况下，与手工编写全部代码相比，程序员可以获得更高的工作效率，或者给与程序更大的灵活度去处理新的情形而无需重新编译。
编写元程序的语言称之为元语言。被操纵的程序的语言称之为“目标语言”。一门编程语言同时也是自身的元语言的能力称之为“反射”或者“自反”。
反射是促进元编程的一种很有价值的语言特性。把编程语言自身作为一级资料类型（如LISP、Forth或Rebol）也很有用。支持泛型编程的语言也使用元编程能力。
元编程通常通过两种方式实现。一种是通过应用程序编程接口（APIs）将运行时引擎的内部信息暴露于编程代码。另一种是动态执行包含编程命令的字符串表达式。因此，“程序能够编写程序”。虽然两种方式都能用于同一种语言，但大多数语言趋向于偏向其中一种。

<!-- more -->

从上面定义我们可以看到**一门语言拥有对自身的元编程能力就表现在反射**，反射又有三个子分支：

- 自省（Introspection）：代码能够自我检查、访问内部属性，我们可以据此获得代码的底层信息。
- 自我修改（Self-Modification）：顾名思义，代码可以修改自身。
- 调解（Intercession）：字面意思是“代他人行事”。在元编程中，调解的概念类似于包装（wrapping）、捕获（trapping）、拦截（intercepting）。

ES6为我们提供了Reflect对象（Reflect API）来实现自省，还提供了Proxy对象帮助我们实现调解，不过元编程并不是由ES6引入的，JavaScript 语言从一开始就支持元编程，ES6 只是让它变得更易于使用。

## ES6之前的元编程

JavaScript 中的 eval 语法糖是 ES6 之前 JavaScript 元编程最简单朴素的实现方式，举个🌰:

```js
const blog = {
    name: 'meta-javascript',
}
const key = "author";
const value = "kylee";
testEval = () => eval(`blog.${key} = '${value}'`);
testEval();
console.log(blog); // {name: "meta-javascript", author: "kylee"}
```

eval生成了额外的代码。示例代码执行时为blog对象增加了一个author属性。

### 自省

在ES6引入Reflect对象之前，我们也可以实现自省。举个🌰:

```js
var users = {
    'Tom': 20,
    'Bill': 21,
    'Sam': 22
}
Object.keys(users).forEach(name => {
    const age = users[name];
    console.log(`User ${name} is ${age} years old`);
})
// User Tom is 20 years old
// User Bill is 21 years old
// User Sam is 22 years old
```

### 自我修改

举个🌰:

```js
const obj = {
    value: 12,
    addValue() {
        this.value ++
    }
}
```
示例中的 obj 对象就可以调用自身的 addValue 方法修改自身属性：

```js
obj.addValue();
console.log(obj.value); // 13
```

### 调解

元编程中的调解指的是改变其他对象的语义，在 ES6 之前，可以用 Object.defineProperty 方法改变对象的语义，举个🌰:

```js
const blog = {
    name: 'meta-javascript',
}
Object.defineProperty(blog, 'name', {
    value: 'meta',
    enumerable: false,
    configurable: false,
    writable: false,
})

blog.name = "javascript";
console.log(blog.name) // meta
```

示例中我们创建了 blog 对象，但是我们在后面通过 Object.defineProperty 方法将其 name 属性设为不可写，这儿就是元编程中调解的具体体现。

## Reflect

在 ES6 中，Reflect 是一个新的全局对象（像 math 一样），它提供了一些工具函数，如下为 Reflect 对象提供的方法列表：

```js
Reflect.apply();
Reflect.construct();
Reflect.get();
Reflect.has();
Reflect.ownKeys();
Reflect.set();
Reflect.setPrototypeOf();
Reflect.getPrototypeOf();
Reflect.defineProperty();
Reflect.deleteProperty();
Reflect.getOwnPropertyDescriptor();
Reflect.isExtensible();
```

这些都是自省方法，可以用它们在运行时获取程序内部信息，其中一些方法与 Object 或 Function 对象中的同名方法功能是相同的，那么为什么我们还要引入新的 API 呢？

### 集中在一个命名空间

JavaScript 已经支持对象反射，但是这些API没有集中到一个命名空间中。从ES6开始，它们被集中到 Reflect对象中。

与其他全局对象不同，Reflect 不是一个构造函数，不能使用 new 操作符来调用它，也不能将它当做函数来调用。Reflect对象中的方法和 math 对象中的方法一样是**静态**的。

### 易于使用

Object对象中的自省方法在操作失败的时候会抛出异常，这给开发者增加了处理异常的负担，借助 Reflect 对象我们就可以将操作结果当作布尔值来处理，举个🌰:

```js
// Object.defineProperty
try {
    Object.defineProperty(obj, name, desc);
    // 执行成功
} catch (e) {
    // 执行失败
}

// Reflect.defineProperty
if (Reflect.defineProperty(obj, name, desc)) {
    // 执行成功
} else {
    // 执行失败
}
```

### 一等函数的魅力

我们可以通过(prop in obj) 操作来判断对象中是否存在某个属性。如果多次用到这个操作，我们需要把它封装成函数。在 ES6 的Reflect API中已经包含了这些方法，例如，Reflect.has(obj, prop) 和 (prop in obj) 功能是一样的。

```js
const obj = { bar: true, baz: 'baz' }
// 自定义函数
function deleteProperty(object, key) {
    delete object[key]
}
deleteProperty(obj, 'baz');

// Reflect.deleteProperty
deleteProperty(obj, 'bar');
```

### 以更可靠的方法来使用apply方法

在 ES5 中，我们可以使用apply()方法来调用一个函数，并指定this上下文、传入一个参数数组。

```js
Function.prototype.apply.call(func, obj, arr);
// or
func.apply(obj, arr);
```

这种方式比较不可靠，因为func可能是一个具有自定义 apply 方法的对象。ES6 提供了一个更加可靠、优雅的方式来解决这个问题：

```js
Reflect.apply(func, obj, arr);
```

这样，如果 func 不是可调用对象，会抛出 TypeError，此外Reflect.apply()也更简洁、易于理解。

## Proxy对象

ES6 的 Proxy 对象可以用于调解（intercession）。Proxy 对象允许我们自定义一些基本操作的行为（例如属性查找、赋值、枚举、函数调用等），它主要包含如下内容：

- target：代理为其提供自定义行为的对象。
- handler：包含“捕获器”方法的对象。
- trap：“捕获器”方法，提供了访问目标对象属性的途径，这是通过 Reflect API 中的方法实现的。每个“捕获器”方法都对应着 Reflect API 中的一个方法。

它们的关系如图所示：

<img src="/assets/javascript-meta/01.jpg" width="536px" />

先定义一个包含“捕获器”函数的 handler 对象，再使用这个 handler 和目标对象来创建一个代理对象，这个代理对象会应用 handler 中的自定义行为，handler 方法列表如下所示：

```js
handler.apply()
handler.construct()
handler.get()
handler.has()
handler.ownKeys()
handler.set()
handler.setPrototypeOf()
handler.getPrototypeOf()
handler.defineProperty()
handler.deleteProperty()
handler.getOwnPropertyDescriptor()
handler.preventExtensions()
handler.isExtensible()
```

注意每个捕获器都对应着 Reflect 对象的方法，也就是说可以在许多场景下同时使用 Reflect 和 Proxy。

### 自定义对象行为

我们使用 Proxy 可以为对象添加一些自定义行为，举个🌰:

```js
const employee = {
    name: "kyle",
    age: 24,
}
```

**步骤一： 创建一个handler**

我们使用名为 get 的捕获器，可以通过它来获取对象的属性值。handler 代码如下：

```js
let handler = {
    get: function(target, key) {
        if (key === 'info') {
            return `name: ${target.name}, age: ${target.age}`
        }
        return key in target ? target[key] : null;
    }
}
```

以上 handler 代码创建了 info 属性的值，还将属性不存在的情况改为返回 null。

**步骤二： 创建 Proxy 对象**

目标对象 employee 和 handler 都准备好了，可以这样来创建 Proxy 对象：

```js
let proxy = new Proxy(employee, handler);
```

**步骤三：访问 Proxy 对象属性**

proxy 对象创建后我们就可以通过 proxy 对象访问 employee 的属性，举个🌰:

```js
console.log(proxy.name) // 'kyle'
console.log(proxy.age) // 24
console.log(proxy.info) // 'name: kyle, age: 24'
console.log(proxy.full) // null
```

### 与Reflect相结合

我们可以在上述案例中再与Reflect API相结合：

```js
let handler = {
    get: function(target, key) {
        if (key === 'info') {
            return `name: ${target.name}, age: ${target.age}`
        }
        return Reflect.get(target, key);
    }
}
```

## 参考资料
[JavaScript元编程](https://mp.weixin.qq.com/s/1E8d5jYb0sFGPRk3pMLaHA)

[谈元编程与表达能力](https://draveness.me/metaprogramming/)