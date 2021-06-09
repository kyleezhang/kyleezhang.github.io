---
title: TypeScript学习笔记（二）
date: 2020-11-17 10:05:53
toc: true
mathjax: false
categories: 
- 前端
tags: 
- typescript
---


## 接口与类

### 一、接口

在面向对象的语言中“接口”是个很重要的概念，它是对行为的抽象，而具体内容需要通过类实现，TypeScript 中的接口是一个非常灵活的概念，可以用来约束对象、函数以及类的结构和类型，是一种代码协作的契约，我们必须遵守而且不能改变。

<!-- more -->

#### 1、对象类型接口

对象类型的接口用来设置对象需要存在的普通属性、可选属性和只读属性，另外还可以通过类型注解语法或 [propName: string]: any 来制定可以接受的其他任意额外属性，举个🌰:

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

需要注意的是此处 `ReadonlyArray<T>` 类型，它与 `Array<T>` 相似，只是把所有可变方法去掉了，因此可以确保数组创建后再也不能被修改，`ReadonlyMap<T>` 和 `ReadonlySet<T>` 与之类似。

#### 2、函数类型接口

TypeScript 中接口还可以用来规范函数的形状，列出参数列表及返回值类型的函数定义。写法如下：

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
interface PersonInfoInterface {
  name: string
  age: number
  log?(): void
}

interface Student extends PersonInfoInterface {
  doHomework(): boolean // 新增一个方法检查
}
interface Teacher extends PersonInfoInterface {
  dispatchHomework(): void // 新增了一个方法检查
}

// interface Emmm extends Student, Teacher // 也可以继承多个接口

let Alice: Teacher = {
  name: 'Alice',
  age: 34,
  dispatchHomework() { // 必须满足继承的接口规范
    console.log('dispatched')
  }
}

let oliver: Student = {
  name: 'oliver',
  age: 12,
  log() {
    console.log(this.name, this.age)
  },
  doHomework() { // 必须满足继承的接口规范
    return true
  }
}
```

### 二、类

在面向对象语言中，类是一种面向对象计算机编程语言的构造，是创建对象的蓝图，描述了所创建的对象共同的属性和方法。

#### 1、类的属性及方法

在 JavaScript 中我们通过 class 关键字来定义一个类：

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

那么成员属性与静态属性，成员方法与静态方法有什么区别呢？我们直接查看编译后的 ES5 代码：

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

从编译后的代码我们不难看出成员属性会添加到类的实例上，成员方法会添加到类的原型对象上，因此对于类的实例而言是可调用的，而静态属性与静态方法都会只添加到类自身，只能被类自身调用。除此之外我们需要注意类的成员属性和方法还有 public、private 和 protected 可访问性修饰符，举个🌰:

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

需要注意当我们给构造函数添加 private 修饰符时表示类既不可以被继承也不可以被实例化，当我们给构造函数添加 protected 修饰符时表示类不可被实例化只能被继承，常用于声明基类。

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
- 每个私有字段名称都唯一的限定于其包含的类；
- 不能在私有字段上使用 TypeScript 可访问性修饰符（如 public 或 private）；
- 私有字段不能在包含的类之外访问，甚至不能被检测到。

#### 3、访问器

在 TypeScript 中，我们可以通过 getter 和 setter 方法来实现数据的封装和有效性校验，防止出现异常数据。

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
  console.log(employee.fullName); // "Semlinker"
}
```

#### 4、类的继承

继承（Inheritance）是一种联结类与类的层次模型。指的是一个类（称为子类、子接口）继承另外的一个类（称为父类、父接口）的功能，并可以增加它自己的新功能的能力，继承是类与类或者接口与接口之间最常见的关系。在 TypeScript 中，我们通过 extends 关键字来实现继承，通过 super 关键字来调用父类的构造函数和方法，举个🌰:

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

使用 abstract 关键字声明的类，我们称之为抽象类。抽象类不能被实例化，因为它里面包含一个或多个抽象方法，所谓的抽象方法，是指不包含具体实现的方法，抽象类的好处在于可以抽离出一些事物的共性，有利于代码的复用和扩展，举个🌰:

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

面向对象（OOP）语言的三大特性分别是：封装（Encapsulation）、继承（Inheritance）和多态（Polymorphism），多态是指由继承而产生了相关的不同的类，对同一个方法可以有不同的响应。比如下面示例中 Cat 和 Dog 都继承自 Animal，但是分别实现了自己的 eat 方法。此时针对某一个实例，我们无需了解它是 Cat 还是 Dog，就可以直接调用 eat 方法，程序会自动判断出来应该如何执行 eat：

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

方法重载是指在同一个类中方法同名，参数不同（参数类型不同、参数个数不同或参数个数相同时参数的先后顺序不同），调用时根据实参的形式，选择与它匹配的方法执行操作的一种技术。所以类中成员方法满足重载的条件是：在同一个类中，方法名相同且参数列表不同，在以下示例中我们重载了 ProductService 类的 getProducts 成员方法：

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

如果你希望在类中使用必须要被遵循的接口（类）或别人定义的对象结构，可以使用 implements 关键字来确保其兼容性：

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

## 参考资料

极客时间《TypeScript开发实战》专栏

《深入理解TypeScript》

[一份不可多得的 TS 学习指南（1.8W字）](https://juejin.im/post/6872111128135073806#heading-21)

[Typescript使用手册](https://www.bookstack.cn/read/TypeScript-3.6/doc-handbook-README)