---
layout: posts
title: 浮动与清除浮动
date: 2020-10-29 10:31:49
toc: true
mathjax: false
categories: 
- 前端
tags:
- CSS
---

浮动属性最早提出是在CSS1中，其最初的主要目的就是为了允许其他内容（如文本）“围绕”图像，因此浮动属性也只允许作用于图像（有些浏览器还支持表格），后来随着不断发展，浮动属性也允许作用于任何元素，但是文本环绕这一页面样式目前仍然只有利用float属性可以实现，具有唯一性，以下面代码为例：
<!-- more -->
```html
<!DOCTYPE html>
<html>
    <head>
        <title>文本环绕</title>
        <meta charset="UTF-8"/>
        <style type="text/css">
        #text1 {
            float: left;
            top: 0;
            left: 0;
            width: 200px;
            height: 200px;
            border: 2px solid red;
        }
        #text2 {
            width: 400px;
            height: 400px;
            border: 2px dashed green;
        }
        </style>
    </head>
    <body>
        <div id="text1">A</div>
        <div id="text2">B：我们发现元素A和元素B有一部分重叠，但是文本并没有被元素A所覆盖，而是环绕在
        元素A的周围，这恰是浮动特性提出最初的目的；同时我们从这个图中不难看出，首先浮动脱离了标准文档
        流，所以元素A和元素B有一部分重叠，但是浮动依然会影响标准文档流的布局，具体表现为虽然元素A和
        元素B有一部分重叠但是元素B的文本完全绕开了元素A</div>
    </body>
</html>
```

页面效果：
<img src="/assets/css-float/01.png" width="420" height="420"></img>
我们发现**浮动虽然脱离了标准文档流，但是仍然会对标准文档流中的内存空间产生影响**，这也是浮动与绝对定位之间的一个重要区别，我们下面尝试用绝对定位来实现文本环绕效果，你将会清晰看到两者之间的区别。
示例：

```html
<style type="text/css">
#text1 {
    position: absolute;
    top: 0;
    left: 0;
    width: 200px;
    height: 200px;
    border: 2px solid red;
}
#text2 {
    width: 400px;
    height: 400px;
    border: 2px dashed green;
}
</style>
```

页面结果：
<img src="/assets/css-float/02.png" width="420" height="420"></img>
从两图对比中我们可以发现虽然浮动和绝对定位都可以脱离标准文档流，但是**绝对定位是将元素彻底从标准文档流中删除，元素在标准文档流中不占据内存空间，因此更不会对其他元素产生影响，而浮动也是脱离标准文档流，元素向左或向右浮动，但是这些已经被从标准文档流中删除的元素依然会对页面布局产生影响**，下面让我们一起去探讨浮动的特性及影响:

## 特性

首先我们来看CSS中对于浮动的定义：
> 使元素脱离文档流，按照指定的方向（左或右发生移动），直到它的外边缘碰到包含框或另一个浮动框的边框为止。

更为通俗来说也就是CSS浮动的两大原则：

- 浮动的框可以向左或向右移动，直到它的外边缘碰到包含框或另一个浮动框的边框为止。
- 由于浮动框不在文档的普通流中，所以文档的普通流中的块框的排列可以认为浮动框不存在一样。

在CSS中，我们float属性实现元素框的浮动，可选值有如下三种

- left： 表示元素框向左浮动
- right：表示元素框向右浮动
- none：默认值，表示元素框不浮动，按照标准流排列

那么float属性又有什么特性呢？

### 1、文字环绕

这一功能也是float最初的设计目的，CSS在不断地发展过程中对于float属性一直“不忘初心”，甚至还更加完善，使其能够支持除图像以外的其他元素，关于文字环绕在本文前半部分已经有了详细介绍，此处不再赘述。
 
### 2、行级元素横排显示

左浮动:

```html
<body>
    <div id="first">first</div>
    <div id="second">second</div>
    <div id="third">third</div>
</body>

<style>
#first {
    float: left;
    background-color: red;
    width: 50px;
    height: 50px;
}
#second {
    float: left;
    background-color: blue;
    width: 50px;
    height: 50px;
}
#third {
    float: left;
    background-color: yellow;
    width: 50px;
    height: 50px;
}
</style>
```

页面效果：
<img src="/assets/css-float/03.png" width="200" height="70"></img>
右浮动：

```css
#first {
    float: right;
    background-color: red;
    width: 50px;
    height: 50px;
}
#second {
    float: right;
    background-color: blue;
    width: 50px;
    height: 50px;
}
#third {
    float: right;
    background-color: yellow;
    width: 50px;
    height: 50px;
}
```

页面效果：
<img src="/assets/css-float/04.png" width="200" height="70"></img>

至此，我们展示了如何利用左浮动和右浮动将块级元素在行内横排显示，但是我们一起看一下下面几种特殊情况：
（1）如果我们只给第二个元素添加左浮动会怎么样？
示例：

```html
<body>
	<div id="first">first</div>
	<div id="second">second</div>
	<div id="third">third</div>
</body>
<style>
#first {
    background-color: red;
    width: 50px;
    height: 50px;
}
#second {
    float: left;
    background-color: blue;
    width: 50px;
    height: 50px;
}
#third {
    background-color: yellow;
    width: 100px;
    height: 50px;
}
</style>
```

页面效果：
<img src="/assets/css-float/05.png" width="120" height="120"></img>
这儿增加了third元素的宽度是希望能够更清楚得看到左浮动的second元素对third元素的覆盖，发生这种覆盖的原因是second元素脱离了标准文档流，不再占据标准文档流的空间，所以third元素向上移动，second元素覆盖了部分third元素。

(2）如果父组件宽度不足会怎么样？
示例：

```html
<body>
	<div id="container">
		<div id="first">first</div>
		<div id="second">second</div>
		<div id="third">third</div>
	</div>
</body>
<style>
#container {
    width: 120px;
    height: 150px;
    border: 2px dashed gray;
}
#first {
    float: left;
    background-color: red;
    width: 50px;
    height: 50px;
}
#second {
    float: left;
    background-color: blue;
    width: 50px;
    height: 50px;
}
#third {
    float: left;
    background-color: yellow;
    width: 50px;
    height: 50px;
}
</style>
```

页面效果：
<img src="/assets/css-float/06.png" width="150" height="160"></img>

（3）接下来是见证奇迹的时刻
示例：

```html
<body>
 	<div id="container">
  		<div id="first">first</div>
  		<div id="second">second</div>
  		<div id="third">third</div>
  		<div id="forth">forth</div>
 	</div>
</body>
<style>
#container {
    width: 180px;
    height: 150px;
    border: 2px dashed gray;
}
#first {
    float: left;
    background-color: red;
    width: 50px;
    height: 40px;
}
#second {
    float: left;
    background-color: blue;
    width: 50px;
    height: 80px;
}
#third {
    float: left;
    background-color: yellow;
    width: 50px;
    height: 30px;
}
#forth {
    float: left;
    background-color: pink;
    width: 50px;
    height: 50px;
}
</style>
```

页面效果：
<img src="/assets/css-float/07.png" width="200" height="160"></img>

从上图我们神奇的发现：原本应该换行的forth元素竟然被卡住了，后来我查了一下相关资料发现，以换行的那个元素为基准，如果有浮动元素的高度大于换行的那个元素，那么当换行元素换行时会被高的那个元素“卡住”。这一点容易被大家忽略。

### 3、行内元素可以设置宽度和高度

示例：

```html
<body>
	<div id="container">
		<h1 id="first">first</h1>
		<h3 id="second">second</h3>
	</div>
</body>

<style type="text/css">
#container {
    width: 300px;
    height: 200px;
    border: 2px solid gray
}
#first {
    float: left;
    width: 180px;
    background-color: blue;
}
#second {
    float: left;
    width: 100px;
    background-color: yellow;
}
</style>
```

页面效果：
<img src="/assets/css-float/08.png" width="330" height="220"></img>

### 4、支持margin

浮动脱离了标准文档流，但是浮动元素依然支持margin属性
示例：

```html
<body>
	<div id="first">first</div>
	<div id="second">second</div>
	<div id="third">third</div>
</body>

<style type="text/css">
#first {
    float: left;
    background-color: red;
    width: 50px;
    height: 40px;
    margin-right: 10px;
    }
#second {
    float: left;
    background-color: blue;
    width: 100px;
    height: 80px;
    margin-right: 5px;
    }
#third {
    float: left;
    background-color: yellow;
    width: 50px;
    height: 30px;
    margin-right: 15px;
}
</style>
```

页面效果：

<img src="/assets/css-float/09.png" width="330" height="220"></img>

但是**浮动元素却不支持margin: auto属性**
示例：

```html
<body>
	<div id="container">
		<div id="first">first</div>
	</div>
</body>

<style type="text/css">
	#container {
		width: 200px;
		height: 200px;
		border: 2px dashed blue
	}
	#first {
		float: left;
		width: 100px;
		height: 50px;
		background-color: yellow;
		margin: auto;
	}
<style>
```

页面效果：
<img src="/assets/css-float/10.png" width="227" height="227"></img>

至此，我们对浮动特性进行一个总结：

- 脱离标准文档流
- 文本环绕
- 块级元素横排显示
- 内联元素设置宽高
- 浮动元素支持margin，但是不支持margin: auto
- 元素没有设置宽度时，宽度为内容撑开宽

关于浮动的一点思考：

1. float属性使得元素脱离了标准文档流，不再继续在标准文档流中占据内存，但是却又不是完全删除，具体表现为一个元素浮动时，其他内容会“环绕”该元素。这一点与绝对定位有着根本性的区别，绝对性定位是将元素从标准文档流中彻底删除，元素原先在正常文档流中所占的空间会关闭，就好像该元素原来不存在一样，该元素再也不会影响其他元素的布局了，对于浮动元素的这一特性主要是为了避免一个元素的内容被浮动的元素覆盖掉；
2. 浮动元素本质上更像是将该元素的display属性设为inline-block，这使得块级浮动元素可以在行内横排，内联元素可以设置宽高，元素没有设置宽度时宽度为内容的撑开宽。但本质上两者还是不同，主要区别就是浮动元素的方向性，display:inline-block仅仅一个水平排列方向，就是从左往右，而float可以从右往左排列;

## 清除浮动

从之前的例子我们也可以看出，我们对一个元素添加浮动属性将不可避免的对其他元素产生影响，我们以下面代码为例：

```html
<body>
	<div id="container">
		<div class="box">first</div>
		<div class="box">second</div>
		<div class="box">third</div>
	</div>
	<div id="anotherBox">
	</div>
</body>

<style type="text/css">
#container {
    border: 2px solid red;
    width: 400px;
}
#container .box {
    float: left;
    width: 100px;
    height: 100px;
    margin: 2px;
    background-color: green;
}
#anotherBox {
    width: 400px;
    height: 200px;
    background-color: yellow;
}
</style>
```

页面效果：
<img src="/assets/css-float/11.png" width="427" height="267"></img>

我们从上图中可以很清晰的看到如下两点：

- 第一个div元素由于浮动子元素脱离了标准文档流，不再占据文档流中位置，这导致父元素的高度也无法撑开，我们把这种情况称为高度坍塌。
- 由于第一个div元素没有高度，无法包裹自己的子元素，而第二个div在页面布局中去贴第一个div的边了，所以出来的页面效果仿佛子元素是在第二个div元素中，我们一般把第二个div去贴坍塌后的第一个div元素的边这种情况称为浮动塌陷。

那我们如何解决高度坍塌和浮动塌陷，清除元素浮动带给我们的影响呢？

### 1、给定高度

发生高度坍塌的本质原因还是子元素由于浮动无法支撑起父元素的高度，所以我们可以通过给浮动元素的祖先元素指定高度，只要浮动元素在一个有高度的盒子里，那么这个浮动就不会影响后面的元素了，也就是清除了浮动带来的影响；
我们对上面例子进行修改：

```css
#container {
	border: 2px solid red;
	width: 400px;
	height: 150px;
}
```

页面效果：
<img src="/assets/css-float/12.png" width="427" height="419"></img>
我们可以看到高度坍塌问题解决了，并且第二个div元素也不会再受到浮动元素的影响。

### 2、利用clear属性清除浮动

实际上在我们日常的开发中很少给元素指定高度，往往都是内容把元素的高度撑高，所以第一种方法我们在日常开发中使用其实并不广泛，下面我们来介绍利用clear: both来清除浮动，首先clear就是清除的意思，both，代表左浮动和右浮动都清除掉。通俗一点来讲，就是说清除别人对我的影响。

我们尝试对上面例子进行如下修改:

```css
#anotherBox {
    clear: both;
    width: 400px;
    height: 200px;
    background-color: yellow;
}
```

页面效果:
<img src="/assets/css-float/13.png" width="412" height="322"></img>

我们可以看到第二个div元素不再受浮动元素的影响，解决了我们前文提到的浮动塌陷的影响，看到这儿可能有人会问：那我们能不能通过给第一个div元素添加clear属性解决高度坍塌的问题？
我们看示例:

```css
#container {
	clear: both;
	width: 400px;
	border: 2px solid red;
}
#anotherBox {
	clear: both;
	width: 400px;
	height: 200px;
	background-color: yellow;
}
```

页面效果:
<img src="/assets/css-float/13.png" width="412" height="322"></img>

我们发现答案竟然是“完全没有变化”，这是因为即使给第一个div添加clear属性清除所有浮动元素带给它的影响，但是由于浮动子元素已经脱离了标准文档流，文档流中第一个div中没有任何子元素占据文档流空间，依然无法撑起第一个div的高度，同时这种方法还有一个巨大的缺陷：那就是两个div之间margin属性失效了，我们以下面代码为例：

```css
#anotherBox {
	margin-top: 10px;
	clear: both;
	width: 400px;
	height: 200px;
	background-color: yellow;
}
```

页面效果：
<img src="/assets/css-float/13.png" width="412" height="322"></img>
后面有人又开始脑洞大开：在中间一个空盒子，然后给那个空盒子clear：both；这样加了一堵墙之后，第二个div就能掉下来并且不干扰了上面的元素。而且第二个div还是能通过magin-top调节两个div（“墙体”div不要算进去）之间的间距，把这种方法也叫做"隔墙法"。
示例：

```html
<body>
	<div id="container">
		<div class="box">first</div>
		<div class="box">second</div>
  		<div class="box">third</div>
	</div>
	<div id="wall"></div>
	<div id="anotherBox"></div>
</body>
<style type="text/css">
#container {
    border: 2px solid red;
    width: 400px;
    }
#container .box {
    float: left;
    width: 100px;
    height: 100px;
    margin: 2px;
    background-color: green;
}
    #anotherBox {
    margin-top: 10px;
    width: 400px;
    height: 200px;
    background-color: yellow;
}
#wall {
    clear: both;
}
</style>
```

页面效果：
<img src="/assets/css-float/13.png" width="412" height="322"></img>
至此隔墙法较为完美的解决了浮动塌陷的问题，但是高度坍塌问题并没有得到解决，后来有程序员基于隔墙法的思路提出了“内墙法”，即将充作墙体的div添加进第一个div元素的底部，我们看代码示例：

```html
<div id="container">
	<div class="box">first</div>
	<div class="box">second</div>
	<div class="box">third</div>
	<div style="clear: both;"></div>
</div>
<div id="anotherBox"></div>
```
页面示例：
<img src="/assets/css-float/14.png" width="417" height="330"></img>
至此，我们惊喜地发现内墙法完美的解决了高度坍塌和浮动塌陷两个问题，之所以能够解决高度坍塌是因为第一个div中底部的div元素由于clear属性清除了浮动元素对于自己的影响，在标准文档流中占据原本应该占据的内存空间，撑起了第一个div元素的高度，同时第二个div元素依然可以通过margin调节两个div元素之间的距离。因此内墙法在许多项目工程中得到了较为普遍的使用。

总结：
clear属性如何清除浮动：**clear属性不允许被清除浮动的元素的左边/右边挨着浮动元素，底层原理是在被清除浮动的元素上边或者下边添加足够的清除空间。不要在浮动元素上清除浮动**，以之前的情况示例：

```html
<div id="container"> 
	<div class="box">first</div>
	<div class="box">second</div>
	<div class="box" style="clear: both;">third</div>
</div>
<style type="text/css">
#container: {
    width: 400px;
    border: 2px solid red;
}
#container .box {
    float: left;
    width: 100px;
    height: 100px;
    margin: 2px;
    background-color: green;
}
```
页面效果:
<img src="/assets/css-float/14.png" width="417" height="222"></img>
给第三个元素加上clear:both之后，第三个元素的左右都没有挨着浮动元素，但是为什么高度还是坍塌了呢？这是由于第三个元素是浮动元素，脱离了文档流，就算给第三个元素上下加了清除空间，也是没有任何意义的。

### 3、BFC清除浮动
BFC全称是块状格式化上下文（Block Formatting Context），那为什么我们可以通过BFC来清除浮动呢，我们首先来看BFC的特征：

 - BFC容器是一个隔离的容器，和其他元素互不干扰；所以我们可以用触发两个元素的BFC来解决垂直边距折叠问题。
 - BFC可以包含浮动；通常用来解决浮动父元素高度坍塌的问题。

BFC能够清除浮动主要就是因为其“包含浮动“这条特性。那我们怎么才能触发BFC呢？
我们可以通过添加如下属性来触发BFC：

 - float设为left | right
 - overflow设为hidden | auto | scroll
 - display设为table-cell | table-caption | inline-block | flex | inline-flex
 - position设为absolute |  fixed

因此我们可以通过给父元素添加overflow: auto来实现BFC清除浮动
示例：
```css
#container {
	overflow: auto;
	width: 400px;
	border: 2px solid red;
}
```
<img src="/assets/css-float/14.png" width="417" height="322"></img>
BFC不仅清除了浮动元素带来的影响，同时也支持两个div之间通过margin设置距离，但需要注意为了兼容IE浏览器设置overflow属性最好取hidden，但是这样元素阴影或下拉菜单会被截断，比较局限。
 IE6/7不支持BFC，也不支持:after，所以IE6/7清除浮动要靠触发hasLayout，这一点在适配IE浏览器端的时候需要注意。

### 4、伪元素标签法清除浮动（最佳实践）
伪元素标签法清除浮动的方法简单便捷，给父元素添加clearfix类名：

```css
// 单伪元素标签法
// 现代浏览器clearfix方案，不支持IE6/7
.clearfix:after {
    display: table;
    content: " ";
    clear: both;
}
// 全浏览器通用的clearfix方案
// 引入了zoom以支持IE6/7
.clearfix:after {
    display: table;
    content: " ";
    clear: both;
}
.clearfix{
    *zoom: 1;
}

// 双伪元素标签法
// 全浏览器通用的clearfix方案【推荐】
// 引入了zoom以支持IE6/7
// 同时加入:before以解决现代浏览器上边距折叠的问题
.clearfix:before,
.clearfix:after {
    display: table;
    content: " ";
}
.clearfix:after {
    clear: both;
}
.clearfix{
    *zoom: 1;
}
```

以之前的情况为例我们进行修改
示例：
```html
<body>
	<div id=“container" class="clearfix">
		<div class="box">firdt</div>
		<div class="box">second</div>
		<div class="box">third</div>
	</div>
	<div id="anotherBox"></div>
</body>
<style type="text/css">
#container {
    border: 2px solid red;
    width: 400px;
}
#container .box {
    float: left;
    width: 100px;
    height: 100px;
    margin: 2px;
    background-color: green;
}
#anotherBox {
    margin-top: 10px;
    clear: both;
    width: 400px;
    height: 200px;
    background-color: yellow;
}
.clearfix: before, .clearfix: after {
    display: table;
    content: " ";
}
.clearfix: after {
    clear: both;
}
.clearfix {
    *zoom: 1;
}
</style>
```
页面效果:
<img src="/assets/css-float/14.png" width="417" height="322"></img>
