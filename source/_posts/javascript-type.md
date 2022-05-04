---
title: JavaScript 中的类型
date: 2021-04-18 09:51:21
toc: true
mathjax: false
categories: 
- 深入理解 JavaScript
tags: 
- JavaScript
---

## 一、JavaScript 中的类型

JavaScript中的类型可以分为基本数据类型和引用类型两种：

- 基本类型值：指的是保存在栈内存中的简单数据段；
- 引用类型值：指的是那些保存在堆内存中的对象，意思是，栈内存中保存的实际上只是一个指针，这个指针指向内存堆中实际的值；

JavaScript 中的的每一个值都属于某一种数据类型。

<!-- more -->

基本数据类型定义是按值访问，可操作保存在变量中的实际的值。基本类型值指的是简单的数据段。JavaScript 语言规定了 6 种基本数据类型。数据类型广泛用于变量、函数参数、表达式、函数返回值等场合。根据最新的语言标准，这 6 种数据类型是：Undefined、Null、Boolean、String、Number、Symbol。

引用类型：当复制保存着对象的某个变量时，操作的是对象的引用，但在为对象添加属性时，操作的是实际的对象。引用类型值指那些可能为多个值构成的对象。引用类型有这几种：Object、Array、RegExp、Date、Function、特殊的基本包装类型(String、Number、Boolean)以及单体内置对象(Global、Math)。其中Object 是 JavaScript 中最复杂的类型，也是 JavaScript 的核心机制之一。Object 表示对象的意思，它是一切有形和无形物体的总称。

## 二、基本类型值和引用类型值的区别

1、引用类型值可添加属性和方法，而基本类型值则不可以。

```js
// 为引用类型值添加属性
var p = new Object();
p.age = 11;
console.log(p.age); // 11

// 为基本类型值添加属性
var name = 'a';
name.age = 11;
console.log(name.age); // undefined
```

2、在复制变量值时，基本类型会在变量对象上创建一个新值，再复制给新变量。此后，两个变量的任何操作都不会影响到对方。而引用类型在创建一个对象类型时，计算机会在内存中开辟一个空间来存放值，我们要找到这个空间，需要知道这个空间的地址，变量存放的就是这个地址，复制变量时其实就是将地址复制了一份给新变量，两个变量的值都指向存储在堆中的一个对象，也就是说，其实他们引用了同一个对象，改变其中一个变量就会影响到另一个变量。

```js
//基本类型值
var a = 'a';
var b = a;
a = 'b';
console.log(b); //a

//引用类型值,以数组为例

//1.对其中一个变量直接赋值不会影响到另一个变量（并未操作引用的对象）
var a = [1,2,3];
var b = a;
a = [1,2,3,4];
console.log(a); //1,2,3,4
console.log(b); //1,2,3

//2.使用push(操作了引用的对象)
var a = [1,2,3];
var b = a;
a.push(4);
console.log(a); //1,2,3,4
console.log(b); //1,2,3,4
```

## 三、类型的检测

### 1、typeof

typeof 是一个一元运算，放在一个运算数之前，运算数可以是任意类型。其返回值是一个字符串，该字符串说明运算数的类型。

```js
var num = 1;
var a = 'a';
var b;
var flag = true;
var o = null;
var fn = function(){};
var rg = /hello/;
console.log(typeof num); // number
console.log(typeof a); // string
console.log(typeof b); // undefined
console.log(typeof flag); // boolean
console.log(typeof o); // object
console.log(typeof fn); // function
console.log(typeof rg); // object(sarari5、chrome7前返回function)
```

事实上，“类型”在 JavaScript 中是一个有争议的概念。一方面，标准中规定了运行时数据类型； 另一方面，JS 语言中提供了 typeof 这样的运算，用来返回操作数的类型，但 typeof 的运算结果，与运行时类型的规定有很多不一致的地方。我们可以看下表来对照一下。


| 示例表达式 | typeof结果 | 运行时类型行为 |
| --- | --- | --- |
| null | object | Object |
| (function () {}) | function | Object |
| 3 | number | Number |
| "ok" | string | String |
| true | boolean | Boolean |
| void 0 | undefined | Undefined |
| Symbol("a") | symbol | Symbol |

总结：

- typeof对于基本类型来说，除了null都可以显示正确类型
- typeof对于引用类型来说，除了函数都会显示object（判断数组可以使用Array.isArray()方法）

注：使用未声明的变量只有一种情况 不报错，就是 

```javascript
typeof(a); // undefined   
```

这是因为 typeof 返回的是一个字符串

```javascript
typeof(typeof(a)); // string 
```

其他示例：

```javascript
typeof(undefined); // undefined
typeof(NaN); // number
typeof(null); // object
var a = "123abc";
typeof(+a); // number
typeof(!!a); // boolean
typeof(a + ""); // string
```
### 2、instanceof

用来测试一个对象在其原型链中是否存在一个构造函数的 prototype 属性。

- 语法：object instanceof constructor
- 参数：object（要检测的对象）constructor（某个构造函数）
- 描述：instanceof 运算符用来检测 constructor.prototype 是否存在于参数 object 的原型链上。

```javascript
var a = new Array();
console.log(a instanceof Array); // true
console.log(a instanceof Object); // true, 这是因为 Array 是 Object 的子类。

function test(){};
var a = new test();
console.log(a instanceof test) // true
```

instanceof 的主要作用是在继承关系中用来判断一个实例是否属于它的父类型，其优点在于即使在多层继承关系中，instanceof 运算符同样适用，示例：

```js
function Foo(){} 
Foo.prototype = new Aoo(); // JavaScript 原型继承 
var foo = new Foo(); 
console.log(foo instanceof Foo); // true 
console.log(foo instanceof Aoo); // true
```

综上所述，无论是 typeof 还是 instanceof 都不能准确判断出正确的类型，因此在需要判断类型时我们还是需要结合场景选择使用。

## 四、为什么给对象添加的方法可以用在基本类型上？

在 JavaScript 的学习过程中我们往往会产生一个疑问，为什么给对象添加的方法能用在基本类型上？

实际上这是因为在 JavaScript 中，对象的定义是“属性的集合”。属性分为数据属性和访问器属性，二者都是 key-value 结构，key 可以是字符串或者 Symbol 类型。

提到对象，我们必须要提到一个概念：类。因为 C++ 和 Java 的成功，在这两门语言中，每个类都是一个类型，二者几乎等同，以至于很多人常常会把 JavaScript 的“类”与类型混淆。事实上，**JavaScript 中的“类”仅仅是运行时对象的一个私有属性，而 JavaScript 中是无法自定义类型的**。JavaScript 中的几个基本类型，都在对象类型中有一个“亲戚”。它们是：Number、String、Boolean、Symbol。所以，我们必须认识到 3 与 new Number(3) 是完全不同的值，它们一个是 Number 类型， 一个是对象类型。

Number、String 和 Boolean，三个构造器是两用的，当跟 new 搭配时，它们产生对象，当直接调用时，它们表示强制类型转换。Symbol 函数比较特殊，直接用 new 调用它会抛出错误，但它仍然是 Symbol 对象的构造器。

JavaScript 语言设计上试图模糊对象和基本类型之间的关系，我们日常代码可以把对象的方法在基本类型上使用，比如：

```javascript
console.log("abc".charAt(0)); // a
```

甚至我们在原型上添加方法，都可以应用于基本类型，比如以下代码，在 Symbol 原型上添加了 hello 方法，在任何 Symbol 类型变量都可以调用。

```javascript
Symbol.prototype.hello = () => console.log("hello");
var a = Symbol("a");
console.log(typeof a); // symbol，a 并非对象
a.hello(); // hello，有效
```

所以这个问题的答案就是. 运算符提供了装箱操作，它会根据基础类型构造一个临时对象，使得我们能在基础类型上调用对应对象的方法。以下面例子为例：

```javascript
var up = "he is a super man";
var output = up.charAt(6);
console.log(output); // a
```

当执行

```javascript
var output = up.charAt(6);
```

这个步骤的时候后台会这样

```javascript
var up = new String("he is a super man");
```

找到对应的包装对象,包装成一个和up值相等的对象返回

```javascript
var output = up.charAt(6); // 调用方法返回给output
up = null; // 然后销毁
```

同理，数字、布尔值在读取属性的时候也可以通过自己的构造函数来创建自己的一个临时对象，并像对象一样（就是一个对象）引用各自的属性，所以，字符串、数字、布尔值都可以看成是对象，注意，这里是看成对象，他们并不是真正的对象，也就是严格来说，它们并不是对象，因为对象是可变的，可以修改属性，而原始值是不可变的是不可修改的(看下面)

```javascript
var b = "abcdefg";
console.log(b.toUpperCase()); // ABCDEFG
console.log(b); // abcdefg
```

它只是返回一个变成大写的副本没有改变原始的变量，而且不能在原始数据类型上添加属性和方法。因为**创建的只是一个临时对象，写的属性和方法只存在于临时对象上，引用完后随即销毁**。
这儿需要注意 null 和 undefined，首先是 null  它是一个关键字，表示为 “空",

```javascript
console.log(typeof null); // object
```

由此可见它是一个对象，但是它只是指向一个空对象的引用。

然后是 undefined，undefined 是另一个表示“空值”特殊值，它表示未定义，当我们对变量只声明没有初始化时(赋值)，输出为 undefined，当我们引用一个不存在的属性时，输出也为 undefined，但是需要注意注意它并不是一个关键字，它是一个变量，而且是一个全局变量，我们可以验证一下：

```javascript
console.log(undefined in window); // true
```
而且

```javascript
console.log(typeof undefined); // undefined
```

这严格表明 undefined 是这个类型的唯一成员，除了 undefined，JavaScript 里面其他一切的都可以看作是对象。

## 五、装箱转换与拆箱转换

### 1、装箱转换

每一种基本类型 Number、String、Boolean、Symbol 在对象中都有对应的类，所谓装箱转换，正是把基本类型转换为对应的对象，它是类型转换中一种相当重要的种类。

我们都知道全局的 Symbol 函数无法使用 new 来调用，但我们仍可以利用装箱机制来得到一个 Symbol 对象，我们可以利用一个函数的 call 方法来强迫产生装箱。我们定义一个函数，函数里面只有 `return this`，然后我们调用函数的 call 方法到一个 Symbol 类型的值上，这样就会产生一个 `symbol Object`。我们可以用 `console.log` 看一下这个东西的 type of，它的值是 object，我们使用 `symbolObject instanceof` 可以看到，它是 Symbol 这个类的实例，我们找它的 constructor 也是等于 Symbol 的，所以我们无论从哪个角度看，它都是 Symbol 装箱过的对象：

```javascript
var symbolObject = (function(){ return this; }).call(Symbol("a"));

console.log(typeof symbolObject);  // object
console.log(symbolObject instanceof Symbol);  // true
console.log(symbolObject.constructor == Symbol); // true
```

装箱机制会频繁产生临时对象，在一些对性能要求较高的场景下，我们应该尽量避免对基本类型做装箱转换。使用内置的 Object 函数，我们可以在 JavaScript 代码中显式调用装箱能力。

```javascript
var symbolObject = Object(Symbol("a"));

console.log(typeof symbolObject);  // object
console.log(symbolObject instanceof Symbol);  // true
console.log(symbolObject.constructor == Symbol);  // true
```

每一类装箱对象皆有私有的 Class 属性，这些属性可以用 Object.prototype.toString 获取：

```javascript
var symbolObject = Object(Symbol("a"));
console.log(Object.prototype.toString.call(symbolObject)); // [object Symbol]
```

在 JavaScript 中，没有任何方法可以更改私有的 Class 属性，因此 Object.prototype.toString 是可以准确识别对象对应的基本类型的方法，它比 instanceof 更加准确。但需要注意的是，call 本身会产生装箱操作，所以需要配合 typeof 来区分基本类型还是对象类型。

### 2、拆箱转换

对象到 String 和 Number 的转换都遵循“先拆箱再转换”的规则。通过拆箱转换，把对象变成基本类型，再从基本类型转换为对应的 String 或者 Number。

**拆箱转换会尝试调用 valueOf 和 toString 来获得拆箱后的基本类型。如果 valueOf 和 toString 都不存在，或者没有返回基本类型，则会产生类型错误 TypeError**。

```javascript
var o = {
    valueOf : () => {console.log("valueOf"); return {}},
    toString : () => {console.log("toString"); return {}}
}

o * 2
// valueOf
// toString
// TypeError
```

我们定义了一个对象 o，o 有 valueOf 和 toString 两个方法，这两个方法都返回一个对象，然后我们进行 o*2 这个运算的时候，你会看见先执行了 valueOf，接下来是 toString，最后抛出了一个 TypeError，这就说明了这个拆箱转换失败了。到 String 的拆箱转换会优先调用 toString。我们把刚才的运算换成 String(o)，那么你会看到调用顺序就变了。

```javascript
var o = {
    valueOf : () => {console.log("valueOf"); return {}},
    toString : () => {console.log("toString"); return {}}
}

String(o)
// toString
// valueOf
// TypeError
```

可以看到，toString 和 valueOf 的执行顺序并不固定，而是根据某个条件来决定的，那么是根据什么呢？

实际上每个对象都有一个 `Symbol.toPrimitive` 属性，其指向一个方法。当对象需要被转为基本类型的值时，会调用这个方法，返回该对象对应的基本类型值。`Symbol.toPrimitive` 被调用时，会接受一个字符串参数，表示当前运算的模式，一共有三种模式。

- Number：该场合需要转成数值
- String：该场合需要转成字符串
- Default：该场合可以转成数值，也可以转成字符串

示例：

```javascript
let obj = {
  [Symbol.toPrimitive](hint) {
    switch (hint) {
      case 'number':
        return 123;
      case 'string':
        return 'str';
      case 'default':
        return 'default';
      default:
        throw new Error();
     }
   }
};

2 * obj // 246
3 + obj // '3default'
obj == 'default' // true
String(obj) // 'str'
```

因此实际上拆箱转换的具体规则如下：

1. 检查对象中是否有用户显式定义的 `[Symbol.toPrimitive]` 方法，如果有，直接调用；
2. 如果没有，则执行原内部函数 ToPrimitive，然后判断传入的 hint 值，如果其值为 string，顺序调用对象的 toString 和 valueOf 方法（其中 toString 方法一定会执行，如果其返回一个基本类型值，则返回、终止运算，否则继续调用 valueOf 方法）；
3. 如果判断传入的 hint 值不为 string，则就可能为 number 或者 default 了，均会顺序调用对象的 valueOf 和 toString 方法（其中 valueOf 方法一定会执行，如果其返回一个基本类型值，则返回、终止运算，否则继续调用 toString 方法）；

因此对象可以通过显式指定 toPrimitive Symbol 来覆盖原有的行为，示例：

```javascript
var o = {
  valueOf : () => {console.log("valueOf"); return {}},
  toString : () => {console.log("toString"); return {}}
}

o[Symbol.toPrimitive] = () => {
  console.log("toPrimitive"); 
  return "hello"
}

console.log(o + "")
// toPrimitive
// hello
```


## 六、类型转换方法

JavaScript 中类型转换的基本原理就是上述的装箱转换和拆箱转换，下面我们一起来看一下我们平时常用的内省转换

### 1、显式类型转换

**Number(mix)**  ：把 mix 转化成数字类型  可以转为数字的就转化为相应的数字，不能转化的就转为NaN ，其中：
```javascript
Number(true) // 1
Number(false) // 0
Number(null) // 0
Number(undefined) // NaN
```

**parseInt(mix,radix)** ：把 mix 转化成整数 除了数字和能转化为数字的字符串，其他都转化为 NaN。

```javascript
parseInt(true) // NaN； 
parseInt(false) // NaN；
```

当 mix 为字符串时，则从第一位一直到非数字截止，即该方法可以截断，但 `Number()` 不能 ：

```javascript
parseInt("123qqq") // 123；
Number("123qqq") // NaN
```

radix 是目标值的进制，将 mix 看成 radix 进制来进行转化，若有小数部分则是直接去掉。例如将二进制 10100 转化为 16 进制的过程是：先 parseInt() 转化为10进制，然后在 toString() 转化为16进制：

```javascript
var num = 10100;
var test = parseInt(num, 2);
num.toString(16);
```

**parseFloat(number)** ：转化成浮点类型，从一位开始看，到除了第一个点以外的非数字位截止。

**Boolean(mix)** ： 转化为 boolean 类型。

**String(mix)** ：转化为字符串类型。

注：`mix.toString(radix)` 与 `String(mix)` 用法不同，且 undefined 和 null 不能使用

```javascript
null.toString() // 报错
String(null) // "null"
```



### 2、隐式类型转换

**isNaN()**：内部隐式调用 Number()进行类型转化，再判断 Number()返回的值是否是NaN。如：

```javascript
isNaN（null）// false   
isNaN（underfined) // true
```

**++、--、+、-**：内部隐式调用 `Number()` 转化后再进行相应计算。

**+**: 当加号两边有一个是字符串的话，就会调用 `String()`，然后进行字符串的拼接。

**-    *    /    %**：内部隐式调用 `Number()` 进行类型后再计算。

**< > <=  >=**：字符串和数字比会调用 `Number()` 转化为数字。

**== 、!=**：转换规则如下表所示：
|类型1| 类型2| 转换结果|
|--|--|--|
|null  | undefined | 不发生类型转换，直接返回 true |
| null 或 undefined |其它任何非 null 或 undefined 的类型 | 不发生类型转换，直接返回 false |
| NaN | 基本类型或引用类型 | false，NaN 不等于任何东西，包括自身
| string、number 或 boolean 等基本类型 | Date 等对象 | Date 等对象发生拆箱转换
| string、number 或 boolen 等基本类型 | string、number 或 boolea 等基本类型 | 基本类型转换为数字进行比较 |

**&&、||、 ！**：

&&：先看第一个表达式转化成布尔值的值，如果为真，那么看第二个表达式转化为布尔值的值，。。。。依次进行，直到碰到假；如果只有两个表达式，则会在第一个表达式转化为布尔值为真时，直接返回第二个表达式的值；否则返回第一个表达式的值进行赋值。

```javascript
var a = 1 && 2+2; //4
var b = 0 && 2+2; //0
```

||：与&&类似，但先看第一个表达式转化为布尔值后的值，如果为真，直接返回第一个表达式的值，如果为假，则接着往下进行判断。

**判断真假只是决定是否接着“往下走”，但返回的仍是其本身的值，而不是转化的布尔值**

!：将后面的表达式转换为布尔值。

## 参考资料

极客时间《重学前端》专栏
《ES6标准入门》第三版

