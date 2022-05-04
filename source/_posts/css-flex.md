---
title: CSS Flex 弹性盒布局
date: 2020-10-26 21:20:48
toc: true
mathjax: false
categories: 
- CSS 世界
tags:
- CSS
---

Flex布局又称弹性盒布局，是在CSS3中的一种新布局方式，可以简洁、方便、响应式地实现各种页面布局，因此自一提出受到了极大地追捧，目前也得到了各大主流浏览器的支持，因此迅速替代了之前的“display+float+position”的布局形式。
<!-- more -->

## 弹性盒布局简介

 - 弹性盒子由弹性容器(Flex container)和弹性子元素(Flex item)组成。 
 - 弹性容器通过设置 display 属性的值为flex 或 inline-flex将其定义为弹性容器，不遵循标准文档流的显示方式。
 - 弹性容器外以及弹性子元素内部仍遵循标准文档流，弹性盒布局只是定义了弹性子元素在弹性容器内部的布局方式。
 - 弹性容器添加display: flex属性显示为块级元素，添加display: inline-flex显示为行级元素。
 - 设为弹性盒布局以后弹性子元素的float、clear和vertical-align属性将失效，但position属性，依然生效。

示例：

```html
<style>
#container {
    border: 1px dashed red;
    display: flex;
    width: 350px;
    height: 150px;
}
#container div {
    background-color: yellow;
    width: 80px;
    height: 80px;
    margin-right: 5px;
}
</style>
<body>
	<div id="container">
		<div id="first">first</div>
		<div id="second">second</div>
		<div id="third">third</div>
		<div id="forth">forth</div>
	</div>
</body>
```
页面效果：
<img src="/assets/css-flex/01.png" width="423" height="200"></img>
如上图所示，弹性子元素通常在弹性盒子内一行显示，默认情况每个容器只有一行。

## 父容器的六大属性

了解两个基本方向，这个牵扯到弹性布局的使用：
>主轴： 在弹性布局中，我们会通过属性规定水平/垂直方向为主轴
>交叉轴： 与主轴垂直的另一方向，称为交叉轴

### 1、flex-direction属性

决定弹性子元素的排列方向（即定义了主轴方向），可选值：
 - row（默认值）： 主轴为水平方向，起点在左端，子元素从左至右排列；              
 - row-reverse： 主轴在水平方向，起点在右端，子元素从右至左排列；    
 - column：主轴为垂直方向，起点在上沿，子元素从上向下排列；         
 - column-reverse：主轴为垂直方向，起点在下沿，子元素从下向上排列；

### 2、flex-wrap属性

定义如果一行排列不下如何换行，可选值：
 - nowrap（默认）：不换行。当容器宽度不够时，每个项目会被挤压宽度；            
 - wrap： 换行，并且第一行在容器最上方； 
 - wrap-reverse： 换行，并且第一行在容器最下方；

示例：
```html
<style>
#container {
    border: 1px dashed red;
    display: flex;
    flex-wrap: wrap-reverse;
    width: 350px;
    height: 200px;
}
#container div {
    background-color: yellow;
    width: 80px;
    height: 80px;
    margin-right: 5px;
}
</style>
<body>
 	<div id="container">
  		<div id="first">first</div>
  		<div id="second">second</div>
  		<div id="third">third</div>
  		<div id="forth">forth</div>
  		<div id="five">five</div>
	</div>
</body>
```
页面效果：
<img src="/assets/css-flex/02.png" width="423" height="200"></img>
当把wrap-reverse换成wrap：
<img src="/assets/css-flex/03.png" width="423" height="200"></img>

### 3、flex-flow属性

flex-direction和flex-wrap的缩写形式，默认值为：row wrap，可以将flex-direction的可选值和flex-wrap的可选值进行任意组合，此处不再做过多赘述。

### 4、justify-content属性

定义了子元素在在主轴上的对齐方式（此属性与flex-direction息息相关，当flex-direction的取值为row时，起点在主轴左边，取值为row-reverse时起点在主轴右边，column起点在主轴上边，column-reverse-起点在主轴下边）
- flex-start（默认值）： 项目位于主轴起点。           
- flex-end：项目位于主轴终点。          
- center： 居中          
- space-between：两端对齐，项目之间的间隔都相等。(开头和最后的项目，与父容器边缘没有间隔)         
- space-around：每个项目两侧的间隔相等。所以，项目之间的间隔比项目与边框的间隔大一倍。(开头和最后的项目，与父容器边缘有一定的间隔，间隔为项目间间隔的一半)

示例：

```html
<style>
#container {
    border: 1px dashed red;
    display: flex;
    justify-content: flex-start;
    width: 350px;
    height: 200px;
}
#container div {
    background-color: yellow;
    width: 80px;
    height: 80px;
}
</style>
<body>
	<div id="container">
        <div id="first">first</div>
        <div id="second">second</div>
        <div id="third">third</div>
        <div id="forth">forth</div>
  	</div>
</body>
```
<img src="/assets/css-flex/04.png" width="423" height="200"></img>
justify-contet属性为flex-end时：
<img src="/assets/css-flex/05.png" width="423" height="200"></img>
justify-content属性为center：
<img src="/assets/css-flex/06.png" width="423" height="200"></img>
justify-content属性为space-between：
<img src="/assets/css-flex/07.png" width="423" height="200"></img>
justify-content属性为space-around：
<img src="/assets/css-flex/08.png" width="423" height="200"></img>

### 5、align-items属性
定义子元素在交叉轴上的对齐方式（flex-direction属性定义主轴方向，与之相垂直的就是交叉轴方向）

- flex-start：子元素按照交叉轴的起点对齐。            
- flex-end：子元素按照交叉轴的终点对齐。           
- center：子元素按照交叉轴的中点对齐。            
- baseline：子元素按照第一行文字的基线对齐。(文字的行高、字体大小会影响每行的基线)         
- stretch（默认值）：如果项目未设置高度或设为auto，将占满整个容器的高度。

示例：
```html
<style>
#container {
    border: 1px dashed red;
    display: flex;
    flex-direction: row;
    align-items: flex-start;
    width: 350px;
    height: 200px;
}
#container div {
    background-color: yellow;
    width: 80px;
    height: 80px;
}
</style>
<body>
	<div id="container">
		<div id="first">first</div>
		<div id="second">second</div>
		<div id="third">third</div>
		<div id="four">four</div>
	</div>
</body>
```
页面效果：
![](https://img-blog.csdnimg.cn/20190804190226451.png)
align-items属性取flex-end:
![](https://img-blog.csdnimg.cn/20190805133634588.png)
align-items属性取center
![](https://img-blog.csdnimg.cn/20190805134254591.png)
增加第一个子元素的高度和字体大小，align-items属性取baseline
![](https://img-blog.csdnimg.cn/20190805134755617.png)

我们尝试将flex-direction属性修改为column，我们一起来看看会对align-items属性的页面效果产生什么影响：
示例：
```html
<style>
#container {
    border: 1px dashed red;
    display: flex;
    flex-direction: column;
    align-items: flex-start;
    width: 350px;
    height: 200px;
}
#container div {
    background-color: yellow;
    width: 80px;
    height: 80px;
}
</style>
<body>
	<div id="container">
  		<div id="first">first</div>
  		<div id="second">second</div>
  		<div id="third">third</div>
  		<div id="four">four</div>
 	</div>
</body>
```
页面效果：
![](https://img-blog.csdnimg.cn/20190805135302650.png)
当我们通过flex-direction属性修改主轴方向时交叉轴的方向也随之改变，因此弹性子元素的交叉轴排列方式也随之改变。大家可以自己尝试不同flex-direction取值下align-items属性对于页面布局的影响，此处限于篇幅原因不再一一尝试。

6、align-content属性
修改 flex-wrap 属性的行为，类似 align-items, 但不是设置子元素对齐，而是设置行对齐，即align-content属性定义了多根轴线的对齐方式。如果项目只有一根轴线，该属性不起作用。
（可以将其视为多行情形下的align-items属性）
- flex-start：子元素按照交叉轴的起点对齐。           
- flex-end：子元素按照交叉轴的终点对齐。          
- center：子元素按照交叉轴的中点对齐。           
- space-between：子元素按照交叉轴两端对齐，轴线之间的间隔平均分布。      
- space-around：每根轴线两侧的间隔都相等。所以，轴线之间的间隔比轴线与边框的间隔大一倍。        
- stretch（默认值）：轴线占满整个交叉轴。

示例：
```html
<style>
#container {
    border: 1px dashed red;
    display: flex;
    flex-wrap: wrap;
    align-content: flex-start;
    width: 350px;
    height: 200px;
}
#container div {
    background-color: yellow;
    width: 80px;
    height: 80px;
}
</style>
<body>
	<div id="container">
        <div id="first">first</div>
        <div id="second">second</div>
        <div id="third">third</div>
        <div id="four">four</div>
        <div id="five">five</div>
        <div id="sex">sex</div>
        <div id="seven">seven</div>
        <div id="eight">eight</div>
	</div>
</body>
```
页面效果：
![](https://img-blog.csdnimg.cn/20190805141207455.png)
align-content属性为flex-end：
![](https://img-blog.csdnimg.cn/20190805141442520.png)
align-content属性为center：
![](https://img-blog.csdnimg.cn/20190805141546242.png)
align-content属性为space-between：
![](https://img-blog.csdnimg.cn/20190805141735432.png)
align-content属性为space-around：
![](https://img-blog.csdnimg.cn/20190805141936809.png)

我们来尝试一下flex-wrap属性取wrap-reverse时会对align-content属性的页面效果产生什么影响
示例：
```html
<style>
#container {
    border: 1px dashed red;
    display: flex;
    flex-wrap: wrap-reverse;
    align-content: flex-start;
    width: 350px;
    height: 200px;
}
#container div {
    background-color: yellow;
    width: 80px;
    height: 80px;
}
</style>
<body>
 	<div id="container">
        <div id="first">first</div>
        <div id="second">second</div>
        <div id="third">third</div>
        <div id="four">four</div>
        <div id="five">five</div>
        <div id="sex">sex</div>
        <div id="seven">seven</div>
        <div id="eight">eight</div>
 	</div>
</body>
```
页面效果：
![](https://img-blog.csdnimg.cn/20190805142502436.png)

我们可以看到由于修改了flex-wrap属性，子元素对齐的交叉轴起点也发生了改变，因此此时flex-start从底部开始对齐，大家可以自己尝试一下不同flex-wrap属性下align-content属性页面效果的不同，此处由于篇幅原因不再一一展示。

## 子容器的六大属性

### 1、order属性
定义弹性子元素排列顺序
用整数值来定义排列顺序，数值小的排在前面。可以为负值，默认为0
示例：
```html
<style>
#container {
    border: 1px dashed red;
    display: flex;
    width: 350px;
    height: 200px;
}
#container div {
    background-color: yellow;
    width: 80px;
    height: 80px;
}
</style>
<body>
  	<div id="container">
        <div id="first" style="order: 1;">first</div>
        <div id="second">second</div>
        <div id="third">third</div>
        <div id="four">four</div>
	</div>
</body>
```
页面效果：
![](https://img-blog.csdnimg.cn/20190805150148493.png)
由于第一个子元素的order属性为1，其他都为0，所以first子元素排在最后面，其他的排在前面。

### 2、flex-grow属性
定义子元素的放大比例。用整数值规定项目将相对于其他灵活的项目进行扩展放大的量。默认值是 0，即如果存在剩余空间子元素也不扩展。
示例：
```html
<style>
#container {
    border: 1px dashed red;
    display: flex;
    width: 480px;
    height: 200px;
}
#container div {
    background-color: yellow;
    border: 1px solid green;
    width: 80px;
    height: 80px;
}
</style>
<body>
   	<div id="container">
        <div id="first" style="flex-grow: 1;">first</div>
        <div id="second">second</div>
        <div id="third" style="flex-grow: 2;">third</div>
        <div id="four">four</div>
 	</div>
</body>
```
页面效果：
![](https://img-blog.csdnimg.cn/20190805152403444.png)
如上图所示，实际上first元素的宽度为原本元素宽度 + 剩余空间的宽度 x (1/3); third元素的宽度为原本元素的宽度 + 剩余空间的宽度 x (2/3)。

### 3、flex-shrink属性
定义了子元素的缩小比例。整数值规定项目将相对于其他灵活的项目进行收缩的量。默认值是 1，即如果空间不足，该子元素将缩小。如果值为0，则该子元素不挤压宽度，如果所有子元素的flex-shirnk属性全为0，当宽度不够时部分子元素将挤出父容器。如果值越大，则该子元素收缩的量越大。
示例：
```html
<style>
#container {
    border: 1px dashed red;
    display: flex;
    width: 300px;
    height: 200px;
}
#container div {
    background-color: yellow;
    border: 1px solid green;
    width: 80px;
    height: 80px;
}
</style>
<body>
    <div id="container">
        <div id="first" style="flex-shrink: 1;">first</div>
        <div id="second" style="flex-shrink: 0;">second</div>
        <div id="third" style="flex-shrink: 2;">third</div>
        <div id="four" style="flex-shrink: 0;">four</div>
  </div>
</body>
```
页面效果：
![](https://img-blog.csdnimg.cn/20190805155214788.png)
如上图所示：
second元素和forth元素的宽度都没有收缩，first元素的实际宽度 = first元素的正常宽度 - 不足的空间宽度 x (1/3)；third元素的实际宽度 = third元素的正常宽度 - 不足的空间宽度 x (2/3)

### 4、flex-basis属性
定义子元素占据的主轴空间，默认值为auto（如果主轴为水平，则设置这个属性类似于子元素宽度，子元素原width属性将失效；如果主轴为垂直，则设置这个属性类似于子元素的高度，子元素原height属性将失效）
可选值：
 - 整数值或百分比，规定子元素占据主轴空间的初始值；
 - auto：默认值。长度等于元素的长度。如果该项目未指定长度，则长度将根据内容决定。

示例：
```html
<style>
#container {
    border: 1px dashed red;
    display: flex;
    width: 360px;
    height: 200px;
}
#container div {
    background-color: yellow;
    border: 1px solid green;
    width: 80px;
    height: 80px;
}
</style>
<body>
    <div id="container">
        <div id="first" style="flex-basis: 100px">first</div>
        <div id="second">second</div>
        <div id="third">third</div>
        <div id="four">four</div>
    </div>
</body>
```

页面效果：
![](https://img-blog.csdnimg.cn/20190805172356649.png)
从上图我们可以看出，first元素的实际宽度为100px，覆盖之前定义的width：80px

### 5、flex属性
flex-grow, flex-shrink 和 flex-basis的简写，默认值为0 1 auto。除了直接分别给flex-grow、flex-shrink 和 flex-basis赋值还可以选取以下值：
- auto：与 1 1 auto 相同。
- none：与 0 0 auto 相同。
- initial：设置该属性为它的默认值，即为 0 1 auto。
- inherit：从父元素继承该属性。

### 6、align-self属性
定义单个项目自身在交叉轴上的排列方式，可以覆盖掉容器上的align-items属性。
属性值：与align-items相同，默认值为auto，表示继承父容器的align-items属性值。
示例：
```html
<style>
#container {
    border: 1px dashed red;
    display: flex;
    align-items: flex-start;
    width: 360px;
    height: 200px;
}
#container div {
    background-color: yellow;
    border: 1px solid green;
    width: 80px;
    height: 80px;
}
</style>
<body>
	<div id="container">
		<div id="first">first</div>
		<div id="second">second</div>
		<div id="third" style="slign-self: flex-end;">third</div>
		<div id="forth">forth</div>
	</div>
</body>
```
页面效果：
![](https://img-blog.csdnimg.cn/2019080517431075.png)
