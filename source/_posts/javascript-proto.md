---
title: JavaScript 中基于原型的继承
date: 2021-04-20 12:37:42
toc: true
categories: 
- 前端
tags:
- JavaScript
---

继承是面向对象语言最为重要的概念之一，许多面向对象语言都支持两种继承方式：接口继承和实现继承，接口继承只继承方法签名，而实现继承则继承实际的方法，由于JavaScript中函数没有签名，因此JavaScript中无法实现接口继承，只支持实现继承。
在传统的基于类面向对象的语言如Java、C++中，继承的本质是扩展一个已有的类，并生成新的子类。由于这类语言严格区分类和实例，继承实际上是类型的扩展。但是，JavaScript其实现继承主要是依靠原型链来实现的，本文主要介绍JavaScript中基于原型实现继承的几种主要方式：

<!-- more -->

## 一、原型链继承

核心：父类型的实例作为子类型的原型。

示例：

```javascript
function SupType() {
  this.property = true;
}
SupType.prototype.getSuperValue = function() {
  return this.property;
}
function SubType() {
  this.subproperty = false;
}
// 继承了Suptotype
SubType.prototype = new Suptotype;
// 添加新方法
SubType.prototype.getSubValue = function() {
  return this.subproperty;
}
var instance = new subType();
console.log(instance.getSuperValue()); // false
```

原型继承的实现本质是重写原型对象，代之以一个新类型的实例，在上面示例中原来存在于Suptype的所有实例中的属性和方法，现在也存在于SubType.prototype中了。在确定了继承关系后我们给Subtype.prototype添加了一个方法，这样就在继承了SuperType属性的方法和方法的基础上又添加了一个新方法，示例中的继承关系如下图所示

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191127223528684.png)

优点：

- 非常纯粹的继承关系，实例是子类型的实例，也是父类型的实例
- 父类型新增原型方法或原型属性，子类型都能访问到
- 简单，易于实现

缺点：

- 可以在SubType构造函数中，为SubType实例增加实例属性。如果要新增原型属性和方法，则必须放在new SupType()这样的语句之后执行
- 无法实现多继承
- 来自原型对象的所有属性被所有实例共享
- 创建子类型实例时，无法向父类型构造函数传参
- 不能使用对象字面量创建原型方法，会重写原型链

## 二、借用构造函数继承

核心：子类型构造函数的内部调用父类型构造函数，等于是复制父类的实例属性给子类（JavaScript中的函数本质上是在特定环境中执行代码的对象，因此也可以通过使用apply()和call()方法也可以在（将来）新创建的对象上执行构造函数）

示例：

```javascript
function SuperType() {
  this.colors = ["red", "blue", "green"];
}
function SubType() {
  // 继承了SupType
  SupType.call(this)
}
var instance1 = new SubType();
instance1.colors.push("black");
console.log(instance1.colors); // "red, blue, green, black"

var instance2 = new SubType()
console.log(instance2.colors); // "red, blue, greeen" 
```

优点：
 - 解决了原型继承中，子类型实例共享父类型引用属性的问题
 - 创建子类型实例时，可以向父类型传递参数
 - 可以实现多继承（call多个父类对象）

缺点：
 - 实例并不是父类型的实例，只是子类型的实例
 - 只能继承父类型的实例属性和方法，不能继承原型属性/方法
 - 无法实现函数复用，每个子类型都有父类型实例函数的副本，影响性能

## 三、组合继承

核心：使用原型链实现原型属性和方法的继承，通过借用构造函数实现对实例属性的继承。

示例：

```js
function SuperType(name) {
  this.name = name;
  this.colors = ["red", "blue", "green"];
}
SupType.prototype.sayName = function() {
  console.log(this.name)
}
function SubType(name, age) {
  // 继承实例属性
  SupType.call(this, name);
  this.age = age;
}
// 继承原型方法 
SubType.prototype = new SuperType();
SubType.prototype.sayAge = function () {
  console.log(this.age)
}
var instance1 = new SubType('xiaozhang', 23);
instance1.colors.push("black");
console.log(instance1.colors) // "red, blue, green, black"
instance1.sayName() // "xiaozhang"
instance1.sayAge() // 23 
var instance2 = new SubType('xiaoli', 22);
console.log(instance2.colors); // "red, blue, green, black"
instance2.sayName() // "xiaoli"
instance2.sayAge() // 22
```

优点：
- 弥补了借用构造函数继承的缺陷，可以继承实例属性/方法，也可以继承原型属性/方法
- 既是子类型的实例，也是父类型的实例，可以使用 instanceof 和 isPrototypeOf() 来识别组合继承的对象
- 不存在引用属性共享问题
- 可传参
- 函数可复用

缺点：
- 调用了两次父类构造函数，生成了两份实例（后文会进行详细解释）

## 四、原型式继承

核心：对传入对象进行浅复制，作为原型对象
示例：

```js
function object(o) {
  // 创建临时性构造函数
  function F(){}
  F.prototype = o;
  // 返回临时类型的新实例
  return new F();
}
var Person = {
  name: 'xiaozhang',
  colors: ["red", "blue", "green"]
};
var anotherPerson = object(Person);
anotherPerson.name = 'xiaoli';
anotherPerson.colors.push('black');
var yetAnotherPerson = object(person);
yetAnotherPerson.colors.push('yellow');
console.log(person.colors) // "red, blue, green, black, yellow"
```

ES5中新增了Object.create()方法规范了原型式继承，这个方法接受两个参数：一个用作新对象原型的对象和（可选的）一个为新对象定义额外属性的对象，第二个参数与 Object.defineProperties() 方法的第二个参数格式相同：每个属性都是通过自己的描述符定义的。
因此上面的示例可重写为：

```js
var Person = {
  name: 'xiaozhang',
  colors: ["red", "blue", "green"]
};
var anotherPerson = object.create(Person);
anotherPerson.name = 'xiaoli';
anotherPerson.colors.push('black');
var yetAnotherPerson = object.create(person);
yetAnotherPerson.colors.push('yellow');
console.log(person.colors) // "red, blue, green, black, yellow"
```

优点：
- 支持多继承
- 简单方便

缺点：
- 包含引用类型的属性始终都会共享相应的值

## 五、寄生式继承

核心：创建一个仅用于封装继承过程的函数。
示例：

```javascript
function createAnother(original) {
  // 通过调用Object.create()创建一个新对象
  var clone = Object.create(original);
  // 以某种方式增强这个对象
  clone.sayHi = function() {
    alert("hi");
  }
  // 返回这个对象
  return clone;
}
var person = {
  name: "xiaozhang",
  age: 23
}
var anotherPerson = createAnother(person);
another.sayHi(); // "hi"
```

优点：
- 示例中使用的Object.create()方法并不是必须的，任何能够返回新对象的函数都适用于此模式。
- 在主要考虑对象而不是自定义对象或者构造函数的情况下，寄生式继承显得更为灵活与方便。

缺点：
- 使用寄生式继承来为对象添加函数时会由于不能做到函数复用而降低效率，这一点与构造函数模式类似。

## 六、寄生组合式继承

核心：通过寄生方式，砍掉父类型的实例属性，这样，在调用两次父类型的构造函数的时候，就不会初始化两次实例方法/属性，避免的组合继承的缺点。
组合继承是JavaScript最常用的继承模式，但是它最大的不足就是无论在什么情况下都会调用两次父类型构造函数，一次是在创建子类型原型的时候，另一次是在子类型构造函数内部，因此这导致子类型的原型上创建了不必要的实例属性，只不过当我们在子类型的实例中进行访问时被子类型构造函数中的实例属性所覆盖。以我们之前组合继承模式的示例为例：

```javascript
function SuperType(name) {
  this.name = name;
  this.colors = ["red", "blue", "green"];
}
SupType.prototype.sayName = function() {
  console.log(this.name)
}
function SubType(name, age) {
  // 第二次调用父类型方法
  SupType.call(this, name);
  this.age = age;
}
// 第一次调用父类型方法
SubType.prototype = new SuperType();
SubType.prototype.sayAge = function () {
  console.log(this.age)
}
```

组合式继承的主要思路是通过借用构造函数来继承属性，通过原型链的混成形式来继承方法，而寄生式组合继承的优化主要是不必为了制定子类型的原型而调用父类型的构造函数，我们所需的无非就是父类型原型的副本而已，因此我们可以通过寄生式继承来继承父类型原型，然后再将结果制定给子类型的原型。
示例：

```javascript
function SupType(name) {
  this.name = name,
  this.colors = ["red", "blue", "green"]
}
function SubType(name, age) {
  SupType.call(this, name);
  this.age = age;
}
// 实现寄生式继承的另一种思路
(function() {
  // 创建一个实例方法的类
  var clone = function(){};
  clone.prototype = SupType.prototype;
  // 将实例作为子类型的原型
  SubType.prototype = new SupType;
})();
// 向子类型的原型添加专属方法
SubType.prototype.sayAge = function() {
  alert(this.age)
}
var instance = new SubType("xiaozhang", 23)
console.log(instance.age) // 23
console.log(instance.name) // "xiaozhang"
instance.getAge() // 23
```

优点：
- 只调用了一次父类型构造函数，因此避免了在子类型的原型上创建不必要的、多余的属性。
- 原型链保持不变，能够正常使用instanceof和isPrototypeOf()。