---
title: TypeScript学习笔记（五）
date: 2020-11-25 10:27:57
toc: true
categories: 
- 前端
tags: 
- TypeScript
---

## 装饰器

在 ES6 中增加了对类对象的相关定义和操作（比如class和extends），这就使得我们在多个不同类之间共享或者扩展一些方法或者行为的时候，变得并不是那么优雅。这个时候，我们就需要一种更优雅的方法来帮助我们完成这些事情，这个方法就是**装饰器**。

装饰器（decorators）这一特性的提出来源于python之类的语言，如果你熟悉python的话，对它一定不会陌生。那么我们先来看一下python里的装饰器是什么样子的吧：

> A Python decorator is a function that takes another function, extending the behavior of the latter function without explicitly modifying it.

<!-- more -->

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

## 参考资料
极客时间《TypeScript开发实战》专栏
《深入理解TypeScript》
[一份不可多得的 TS 学习指南（1.8W字）](https://juejin.im/post/6872111128135073806#heading-21)
[Typescript使用手册](https://www.bookstack.cn/read/TypeScript-3.6/doc-handbook-README)
[reflect-metadata的研究](https://juejin.cn/post/6844904152812748807)