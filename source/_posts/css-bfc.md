---
layout: posts
title: BFC-块状格式化上下文
date: 2020-10-26 20:56:53
toc: true
categories: 
- 前端
tags:
- CSS
---

BFC的全称是块状格式化上下文，MDN中对于对于BFC的定义如下：

> 一个块格式化上下文（block formatting context） 是Web页面的可视化CSS渲染出的一部分。它是块级盒布局出现的区域，也是浮动层元素进行交互的区域。

BFC是一个独立的布局环境，按照块级盒子进行布局，其中的元素布局是不受外界的影响，并且在一个 BFC 中，块盒与行盒（行盒由一行中所有的内联元素所组成）都会垂直的沿着其父元素的边框排列。

<!-- more -->
## BFC的产生
一个块状格式化上下文可由以下方法创建：

 - 根元素（html元素）或其它包含它的元素
 - 浮动元素 (元素的 float 不是 none)
 - 绝对定位元素 (元素具有 position 为 absolute 或 fixed)
 - 内联块 (元素具有 display: inline-block)
 - 表格单元格 (元素具有 display: table-cell，HTML表格单元格默认属性)
 - 表格标题 (元素具有 display: table-caption, HTML表格标题默认属性)
 - 具有overflow 且值不是 visible 的块元素，
 - display: flow-root
 - column-span: all 应当总是会创建一个新的格式化上下文，即便具有 column-span: all 的元素并不被包裹在一个多列容器中。
 - 弹性盒flex boxes（元素具有display: flex或inline-flex）

使用overflow或上述某些方法创建BFC时会有两个问题。首先，这些方法本身是有自身的设计目的，所以在使用它们创建BFC时可能会产生副作用，例如，使用overflow创建BFC后在某些情况下可能会看到出现一个滚动条或者元素内容被裁切。这是由于overflow属性的设计是用来让你告诉浏览器如何定义元素的溢出状态的。浏览器执行了它最基本的定义。
即使在没有任何不想要的副作用的情况下，使用 overflow 也可能会让其他开发人员感到困惑。为什么 overflow 设置为 auto 或 scroll?最初的开发者的意图是什么?他们想要这个组件上的滚动条吗?
因此最理想的做法是创建一个BFC的同时不会产生任何副作用，因此这就有了display: flow-root的诞生，但是目前各大主流浏览器对于display: flow-root属性的支持还是有限的，各大浏览器的支持情况如下图所示：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190813095641166.png)
除此之外我们需要注意到一个块状格式化上下文包括创建它的元素内部所有内容，除了被包含于创建新的块级格式化上下文的后代元素内的元素。

## BFC的特性
你可以把BFC看成布局中的一个迷你布局，一旦一个元素创建了一个BFC，那么它就包含了所有内容。这样一个迷你布局有着如下特性：

1. BFC内部的盒会在垂直方向一个接一个排列（可以看作BFC中有一个常规流）
2. 内部盒子垂直方向的距离由margin决定。属于同一个BFC的两个相邻子元素盒模型中的垂直方向margin会发生重叠；
3. 每一个盒子的左外边距应该和包含块的左边缘相接触。即使存在浮动也是如此，除非子盒子形成了一个新的BFC。
4. BFC就是页面上的一个隔离的独立容器，容器里面的子元素不会影响到外面的元素，反之亦然；
5. 计算BFC的高度时，考虑BFC所包含的所有元素，连浮动元素也参与计算；
6. BFC的区域不会与float box重叠；

特性1中的内部的盒实际上指的是块盒（块级元素）和行盒（一行内联元素），例如html元素作为根元素会创建一个BFC，因此正常情形下页面中的div元素、p元素等块级元素都是一行一行垂直排列；

特性2中BFC内部元素的垂直方向的margin会发生折叠，但是，直接子孙元素与该BFC上下边界margin不能折叠，这是为了保证BFC外部元素不会影响内部内容。以html元素为例，因为创建了一个BFC，所以块级元素的垂直相邻外边距会合并。同时，两个上下相邻的BFC之间折不折叠要看具体情况，如display: inline-block float: left不会折叠，而overflow: hidden则会折叠。

特性3中需要注意以下几点：

- 包含块不一定指的是父元素，比如position属性为absolute的元素的包含块为第一个position属性不为static的祖先元素，如果向上一直找不到则为body元素。
- BFC中的盒子应该与其自身的包含块相接触，而非与BFC盒子本身相接触。
- BFC中的盒子是与其包含块的 left edge 相接触，而不是包含块的 left-border 相接触。left edge 正确的翻译为左边缘。左边缘可能是content box的左边缘（非绝对定位如position: relative; float: left），也可能是padding box的左边缘（如绝对定位position: absolute; position: fixed）。
 
特性4是BFC的一个核心特性，它规定了BFC子元素无论margin-top: -10000px float: left 等都不会影响到BFC外部的元素的布局。所以BFC是最好的清除浮动的方式，连浮动的文字环绕问题都能解决；

特性5 BFC计算高度时包含浮动元素的高度，因此可以利用BFC此特性解决浮动元素高度坍塌的问题；

特性6 可以利用BFC的此特性实现自适应两栏式布局，例如：

```html
<!DOCTYPE html>
<html lang="en">
    <head>
        <meta charset="gbk">
        <meta name="viewport" content="width=device-width" />
        <title>自适应两栏式布局</title>
        <style type="text/css">
 		section {
  			height: 250px;
  			border: 1px solid;
 		}
 		aside {
  			float: left;
  			width: 100px;
  			height: 100%;
  			background: lightgreen;
 		}
 		main {
  			overflow: auto;
  			height: 100%;
  			background: lightblue;
 		}
 		div {
  			height: 20px;
  			margin-bottom: 10px;
  			background: blue;
 		}
        </style>
    </head>
    <body>
 	<section>
  		<aside></aside>
  		<main>
   			<div></div>
   			<div></div>
  		</main>
 	</section>
    </body>
</html>
```
页面效果：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190813114305623.png)
 其中main元素的宽度是自适应的。

## BFC的应用

### 1、防止margin折叠

我们首先来了解一下外边距合并，以下面代码为例：

```html
<style>
#container {
    width: 400px;
    background: #ccc;
}
p {
    padding: 0px;
    margin: 20px 0 20px 0;
    background: green;
    color: #fff;
}
</style>
<body>
	<div id="container">
		<p>悟已往之不谏</p>
		<p>知来者尤可追</p>
	</div>
</body>
```
页面效果：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190813143048100.png)
因为 p 元素的 margin 和外部 div 上的 margin 之间没有任何东西，所以两个会折叠，因此 p 最终与 div 的顶部和底部齐平。 我们在 p 的上方和下方看不到任何灰色。
在CSS当中，相邻的两个盒子（可能是兄弟关系也可能是祖先关系）的外边距可以结合成一个单独的外边距。这种合并外边距的方式被称为折叠，并且因而所结合成的外边距称为折叠外边距。折叠的结果按照如下规则计算：

1. 两个相邻的外边距都是正数时，折叠结果是它们两者之间较大的值。
2. 两个相邻的外边距都是负数时，折叠结果是两者绝对值的较大值。
3. 两个外边距一正一负时，折叠结果是两者的相加的和。

**注**：发生margin折叠的必要条件是margin必须是邻接的。
下面我们给container元素添加属性overflow: hidden;，这样container元素创建了一个BFC：
```css
#container {
 	width: 400px;
  	background: #ccc;
  	overflow: hidden;
}
```
页面效果：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190813144500720.png)
这次我们在p元素的上方和下方出现了灰色背景，这是因为BFC的特性2，BFC内部元素的垂直方向的margin会发生折叠，但是，直接子孙元素与该BFC上下边界margin不能折叠，因此利用BFC的此特性可以解决margin折叠问题。

### 2、清除浮动
利用浮动属性我们可以实现文本环绕，块级元素水平排列等等，但是我们在使用浮动属性的时候发现浮动元素也会不可避免的带来一些其他影响，例如高度坍塌，浮动塌陷等等，那么接下来我们一起看如何利用BFC清除浮动元素带给我们的影响。

#### (1)解决文本环绕
示例：

```html
<style>
#container {
    width: 400px;
    height: 300px;
    border: 1px dotted red;
}
#box {
    width: 150px;
    height: 150px;
    float: left;
    background: lightblue;
}
</style>
<body>
	<div id="container">
		<div id="box"></div>
		<p>浮动属性最早提出的目的就是为了实现文本环绕图片，后来不仅仅支持img元素，还开始支持
		其他元素。浮动属性最早提出的目的就是为了实现文本环绕图片，后来不仅仅支持img元素，还开
		始支持其他元素。浮动属性最早提出的目的就是为了实现文本环绕图片，后来不仅仅支持img元素
		还开始支持其他元素。</p>
	</div>
</body>
```
页面效果：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190813151331625.png)

给p元素增加`overflow: auto`

```css
p {
	overflow: auto;
}
```
页面效果：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190813151810811.png)
能够使用BFC消除文本环绕主要是因为BFC的特性4：BFC就是页面上的一个隔离的独立容器，容器里面的子元素不会影响到外面的元素，反之亦然；

### (2)解决高度坍塌
示例：

```html
<style>
#container {
    width: 400px;
    border: 2px dotted red;
}
.box {
    float: left;
    width: 150px;
    height: 150px;
    background: yellow;
}
</style>
<body>
	<div id="container">
		<div class="box"></div>
		<div class="box"></div>
	</div>
</body>
```
页面效果：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190813152752766.png)
由于浮动子元素脱离了标准文档流，不再占据标准文档流的内存空间，因此无法撑起container元素的高度，我们把这种情况称为高度坍塌，我们给container元素增加overflow: hidden;

```css
container {
	width: 400px;
	border: 2px dotted red;
	overflow: hidden;
}
```

页面效果：
![在这里插入图片描述](https://img-blog.csdnimg.cn/2019081315544828.png)
能够使用BFC解决高度坍塌是因为BFC的特性5：计算BFC的高度时，考虑BFC所包含的所有元素，连浮动元素也参与计算;

### (3)清除浮动塌陷
示例：

```html
<style>
#container {
    width: 400px;
    border: 2px dotted red;
}
.box {
    float: left;
    width: 150px;
    height: 150px;
    background: yellow;
    margin-right: 10px;
}
#anotherBox {
    width: 400px;
    height: 200px;
    background: gray;
}
</style>
<body>
	<div id="container">
		<div class="box"></div>
		<div class="box"></div>
	</div>
	<div id="anotherBox"></div>
</body>
```
页面效果：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190813162338400.png)
由于container元素没有能够撑起高度，这导致了anotherBox元素也没有出现在标准流中的应该位置，常常把这种情况称为浮动塌陷，我们给container元素添加overflow: hidden;后页面效果如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190813163606330.png)