---
title: JavaScript 中的数组
date: 2021-06-02 16:29:07
toc: true
mathjax: false
categories: 
- 深入理解 JavaScript
tags: 
- JavaScript
---

数组在任何一门编程语言中都是一个最常见的词，它不仅仅是一种编程语言中的数据类型，还是一种最基础的数据结构。其作为数据结构的官方定义如下所示：

> 数组（Array）是一种线性表数据结构。它用一组连续的内存空间，来存储一组具有相同类型的数据。

这其中有几个关键词

- 线性表：线性表就是数据排成像一条线一样的结构。每个线性表上的数据最多只有前和后两个方向。除了数组，链表、队列、栈等也是线性表结构。
- 连续的内存空间与相同的类型数据：这两个特点赋予了数组结构最为核心的特性：随机访问性。但有利就有弊，这两个限制也让数组的很多操作变得非常低效，比如要想在数组中删除、插入一个数据，为了保证连续性，就需要做大量的数据搬移工作。

<!-- more -->

那么数组结构是如何实现随机访问性的呢？

我们拿一个长度为 10 的 int 类型的数组 `int[] a = new int[10]` 来举例。在我画的这个图中，计算机给数组 `a[10]`，分配了一块连续内存空间 1000～1039，其中，内存块的首地址为 base_address = 1000。

<img src="/assets/javascript-array/01.jpeg" width="600" />

计算机给每个内存单元分配一个地址，计算机通过地址来访问内存中的数据。当计算机需要随机访问数组中的某个元素时，它会首先通过下面的寻址公式，计算出该元素存储的内存地址：

```js
a[i]_address = base_address + i * data_type_size
```

其中 data_type_size 表示数组中每个元素的大小。

因此数组结构拥有一个其他数据结构都无法比拟的核心特性：**在确定数组下标的前提性可以直接访问数组元素，即数组结构访问元素的时间复杂度为O(1)**，当然与之相对的数组结构的删除数据和插入数据都显得极为低效，我们首先来看插入操作，假设数组的长度为 n，现在，如果我们需要将一个数据插入到数组中的第 k 个位置。为了把第 k 个位置腾出来，给新来的数据，我们需要将第 k～n 这部分的元素都顺序地往后挪一位，操作的具体复杂度随 k 的不同，如果 k 为 1 需要移动 n个元素，如果 k 为 2 则需要移动 n-1 个元素，...，事实上数组插入的平均时间复杂度为O(n)。

跟插入数据类似，如果我们要删除第 k 个位置的数据，为了内存的连续性，也需要搬移数据，不然中间就会出现空洞，内存就不连续了。和插入类似，如果删除数组末尾的数据，则最好情况时间复杂度为 O(1)；如果删除开头的数据，则最坏情况时间复杂度为 O(n)；平均情况时间复杂度也为 O(n)。

那么 JavaScript 中的数组（Array）概念是否与数组结构相同呢？事实上**完全不一样！！！**，JavaScript 中的数组本质上是对象，它依然属于 key-value 的结构，因此 JavaScript 中的数组在**内存空间上不是连续（contiguous）的**，在不同的平台（比如不同浏览器端）可以使用不同的数据结构来实现，比如链表。与此同时由于 JavaScript 本身是一门弱类型语言，因此 JavaScript 中的数组**也不仅仅支持一种类型数据**，这也是 JavaScript 中数组与很多编程语言如 C、C++、C# 的数组不同的地方，那么我们应该怎么理解 JavaScript 中的数组呢？

其实 JavaScript 中的数组本质上单纯只是编程语言的数据类型，它本身与数据结构中的数组并不等价，其具体实现依然是 key-value 键值对结构，与普通对象不同的是它的 key 只能是索引属性，即数字 1，2，3...，显而易见，读取目标位置的具体元素、删除目标元素等操作真实的数组结构相较于这种不连续内存的实现都有太多的优越性，而且随着计算机硬件的不断发展计算机内存也有了极大的提升，因此现代浏览器在具体实现 Javascript 中的数组时都采用了不同的性能优化手段，我们以 Chrome 浏览器为例：

在 Chrome 的 V8 引擎中它有两种存储方式，快数组与慢数组，初始化空数组时，使用快数组，快数组使用连续的内存空间，当数组长度达到最大时，JSArray 会进行动态的扩容，以存储更多的元素，相对慢数组，性能要好得多。当数组中 hole 太多时，会转变成慢数组，即以哈希表的方式（ key-value 的形式）存储数据，以节省内存空间。（关于 v8 引擎中对象的具体存储实现可以查看我的博客[V8是怎么提升对象属性访问速度的](https://kyleezhang.github.io/2020/10/25/client-v8-01/)）。

为了对比介绍 V8 实现的这些优化我需要介绍 ES6 引入的类型化数组（Typed Arrays），它给我们提供了访问原始二进制数据的能力，了解更多可以查看 [MDN 文档](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Typed_arrays)。JavaScript 类型数组（Typed Arrays）将实现拆分为缓冲和视图两部分。一个缓冲（由 ArrayBuffer 对象实现）描述的是一个数据块。缓冲没有格式可言，并且不提供机制访问其内容。为了访问在缓冲对象中包含的内存，你需要使用视图。视图提供了上下文 — 即数据类型、起始偏移量和元素数 — 将数据转换为实际有类型的数组。举个🌰

```js
var buffer = new ArrayBuffer(8);
var view   = new Int32Array(buffer);
view[0] = 100;
```

好，接下来我们来验证 V8 引擎对于 JavaScript 中数组操作能力的具体提升，首先是数据插入：

```js
var LIMIT = 10000000;
var arr = new Array(LIMIT);
console.time("Array insertion time");
for (var i = 0; i < LIMIT; i++) {
arr[i] = i;
}
console.timeEnd("Array insertion time"); // Array insertion time: 21.74169921875 ms
```

```js
var LIMIT = 10000000;
var buffer = new ArrayBuffer(LIMIT * 4);
var arr = new Int32Array(buffer);
console.time("ArrayBuffer insertion time");
for (var i = 0; i < LIMIT; i++) {
arr[i] = i;
}
console.timeEnd("ArrayBuffer insertion time"); // ArrayBuffer insertion time: 23.27978515625 ms
```

示例中旧式数组和 ArrayBuffer 的性能不相上下，这是因为 Chrome 已经自动得将将元素类型相同的传统数组在内部转换成内存连续的数组。第一个例子正是如此。尽管使用了 new Array(LIMIT)，但实际实现上依然采用了数组的数据结构，被分配了连续的内存空间，我们接下来修改其实现：

```js
var LIMIT = 10000000;
var arr = new Array(LIMIT);
arr.push({a: 22});
console.time("Array insertion time");
for (var i = 0; i < LIMIT; i++) {
arr[i] = i;
}
console.timeEnd("Array insertion time"); // Array insertion time: 697.494140625 ms
```

这儿我们在第三行改变了数组元素，使元素不再是单一数据类型，我们观察到其执行时间发生了巨大变化，这是因为 arr 数组不再具有数组结构的特征，其底层使用哈希映射分配内存空间，是由对象链表来实现的。我们可以通过查看其内存快照来验证：

<img src="/assets/javascript-array/02.png" width="600" />

我们接着来验证数据的读取：

```js
var LIMIT = 10000000;
var arr = new Array(LIMIT);
arr.push({a: 22});
for (var i = 0; i < LIMIT; i++) {
arr[i] = i;
}
var p;
console.time("Array read time");
for (var i = 0; i < LIMIT; i++) {
//arr[i] = i;
p = arr[i];
}
console.timeEnd("Array read time"); // Array read time: 18.0458984375 ms
```

```js
var LIMIT = 10000000;
var buffer = new ArrayBuffer(LIMIT * 4);
var arr = new Int32Array(buffer);
console.time("ArrayBuffer insertion time");
for (var i = 0; i < LIMIT; i++) {
arr[i] = i;
}
console.time("ArrayBuffer read time");
for (var i = 0; i < LIMIT; i++) {
var p = arr[i];
}
console.timeEnd("ArrayBuffer read time"); // ArrayBuffer read time: 27.92626953125 ms
```

总结：我们平时在创建和使用 Javascript 中的数组时应尽可能使用同种数据类型，以方便浏览器对其性能进行优化，另外不由得感叹一句谷歌大佬们真的牛逼，，，，

参考资料：

极客时间《数据结构与算法之美》专栏

[深入 JavaScript 数组：进化与性能](https://www.wemlion.com/post/javascript-array-evolution-performance/)