---
title: CSS Grid网格布局详解
date: 2021-01-26 11:40:48
toc: true
mathjax: false
categories: 
- CSS 世界
tags:
- CSS
---

Grid 布局又称网格布局，是W3C提出的一个二维布局系统，它与 Flex 布局有一定的相似性，都可以指定容器内部多个项目的位置。但是，它们也存在重大区别。Flex 布局是轴线布局，只能指定"项目"针对轴线的位置，可以看作是一维布局。Grid 布局则是将容器划分成"行"和"列"，产生单元格，然后指定"项目所在"的单元格，可以看作是二维布局。目前为止Grid布局是CSS中最为强大的布局方案。

<!-- more -->

## 一、基础概念与浏览器支持

截止目前为止许多浏览器都提供了对 CSS Grid 的原生支持，而且无需加浏览器前缀：Chrome（包括 Android ），Firefox，Edge，Safari（包括iOS）和 Opera 。 另一方面，Internet Explorer 10和11也支持Grid布局，但是是以一个过时的语法实现。

各大主流浏览器对于Grid布局的支持情况如下图所示：
**PC端浏览器：**
![](https://img-blog.csdnimg.cn/20190821092301578.png)
**移动端浏览器：**
![](https://img-blog.csdnimg.cn/20190821092339600.png)

学习CSS Grid布局我们还需要了解以下基础概念：

### 1、网格容器(Grid Container)

采用Grid 网格布局的元素称为网格容器，即应用display: grid的元素。

### 2、网格项目(Grid Item)

网格容器内部采用Grid 网格布局的**直接子元素**，注意网格项目的子元素不再是网格项目，举个🌰:

```html
<div id="container" style="display: grid;">
	<div class="item"><span></span></div>
	<div class="item"><span></span></div>
	<div class="item"><span></span></div>
</div>
```

上面示例中container元素就是网格容器，item元素就是网格项目，注意item里的span元素并不是网格项目，不受Grid 网格布局的影响。

### 3、网格轨道(Grid Track)

关于网格布局可以直接想象成一个网格，而网格轨道就是这个网格中的“行”和“列”。![](https://img-blog.csdnimg.cn/20190821142351773.png)

### 4、网格线(Grid Line)

划分网格的线，称为"网格线"（grid line）。水平网格线(column grid lines)划分出行，垂直网格线(row grid lines)划分出列。一个m x n的网格有m+1根水平网格线，n+1根垂直网格线。

### 5、单元格(Grid cell)

网格系统的最小单元，行和列的交叉区域，一个 m x n 的网格有 m x n 个单元格。

### 6、网格区域(Grid Area)

一个 网格区域可以由任意数量的网格单元格组成，如下图所示：
![](https://img-blog.csdnimg.cn/20190821160458377.png)
橙色边框包围内容就是一个网格区域。

## 二、网格容器属性

### 1、display属性

将元素定义为网格容器，并为其内容建立新的网格格式上下文。

属性值：

 - grid: 生成块级网格
 - inline-grid: 生成内联网格
 - subgrid: 生成一个**继承其父级网格容器的行和列的大小**的网格容器，它是其父级网格容器的一个子项

注意：设为网格布局以后，容器子元素（项目）的float、display: inline-block、display: table-cell、vertical-align 和 column-* 等设置都将失效。

### 2、grid-template-columns，grid-template-rows属性

容器指定了网格布局以后，接着就要定义网格轨道，即划分行和列。grid-template-columns属性定义每一列的列宽，grid-template-rows属性定义每一行的行高。

用法：

```CSS
grid-template-columns: <track-size> ... | <line-name> <track-size> ... ;
grid-template-rows: <track-size> ... | <line-name> <track-size> ... ;
```

属性值：

- track-size: 网格轨道大小，可以使用css长度，百分比或用分数（用fr单位）。
- line-name: （可选）使用方括号来指定网格线名字，方便以后的引用，网格布局允许同一根网格线有多个名字。

#### 2.1、网格线的名称

如果没有直接指定网格线的名称网格线会自动分配正数和负数名称。示例：

```html
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="gbk"/>
	<meta name="viewport" content="width=device-width"/>
	<title>Grid 网格布局</title>
	<style type="text/css">
 		.container {
  			margin: 20px;
  			display: grid;
  			grid-template-columns: 50px 50px 50px;
  			grid-template-rows: 50px 50px 50px;
 		}
 		.item { 
  			text-align: center;
  			border: 1px solid gray;
 		}
	</style>
</head>
<body>
	<div class="container">
 		<div class="item">1</div>
 		<div class="item">2</div>
 		<div class="item">3</div>
 		<div class="item">4</div>
 		<div class="item">5</div>
 		<div class="item">6</div>
 		<div class="item">7</div>
 		<div class="item">8</div>
 		<div class="item">9</div>
	</div>
</body>
</html>
```

页面效果：

![](https://img-blog.csdnimg.cn/20190821171736580.png)

网格线名称如上图所示。

我们也可以直接给网格线定义名字，示例：

```css
.container{
    display:grid;
    grid-template-columns: [col1-start] 50px [col1-end col2-start] 50px [col2-end col3-start] 50px [col3-end];
    grid-template-rows: [row1-start] 50px [row1-end row2-start] 50px [row2-end row3-start] 50px [row3-end] ;
}
```

其中有的网格线就不止一个名字，例如[col1-end col2-start]。

#### 2.2、repeat()

有时候，重复写同样的值非常麻烦，尤其网格很多时。这时，可以使用repeat()函数，简化重复的值。

用法：

```js
repeat (m , n)
```

参数：
m: 重复的次数
n: 重复的值

上面的示例代码用repeat方法可以改写成：

```css
.container {
  display: grid;
  grid-template-columns: repeat(3, 50px);
  grid-template-rows: repeat(3, 50px);
}
```

repeat()重复某种模式也是可以的，示例：

```css
.container {
  display: grid;
  grid-template-columns: repeat(2, 100px 20px 80px);
}
```
页面效果：
![](https://img-blog.csdnimg.cn/20190821175919563.png)

#### 2.3、auto关键字
当你设置行或列大小为auto时，浏览器为自动为网格分配空间，示例：

```css
.container {	
	display: grid;
    grid-template-columns: 50px auto 50px;
    grid-template-rows: 50px auto 50px;
}	
```

页面效果：

![](https://img-blog.csdnimg.cn/20190822100941666.png)

可以清晰地看到第二列的宽度基本上等于该列单元格的最大宽度，除非单元格内容设置了min-width，且这个值大于最大宽度。第二行的高度表现出了“包裹性”，即由内部文本或子元素决定高度。

#### 2.4、auto-fill 关键字

有时，单元格的大小是固定的，但是容器的大小不确定。如果希望每一行（或每一列）容纳尽可能多的单元格，这时可以使用auto-fill关键字表示自动填充。

```css
.container {
  display: grid;
  grid-template-columns: repeat(auto-fill, 150px);
}
```

页面效果：
![](https://img-blog.csdnimg.cn/20190822151222970.png)

如上图所示，每列宽度150px，浏览器会自动填充，直到容器不能放置更多的列。

#### 2.5、fr关键字

为了方便表示比例关系，网格布局提供了fr关键字（fraction 的缩写，意为"片段"），我们可以将每个fr视为一个小单元，fr 单元允许我们用等分网格容器剩余可用空间来设置 网格轨道(Grid Track) 的大小 。

示例：
```css
.container {
  display: grid;
  grid-template-columns: 150px 1fr 2fr;
}
```
页面效果：
![](https://img-blog.csdnimg.cn/20190822162434374.png)
从上图我们可以看出第一列的宽度为150px，第二列和第三列以1：2的比例均分了剩余空间。

#### 2.6、minmax()

minmax()函数产生一个长度范围，表示长度就在这个范围之中。它接受两个参数，分别为最小值和最大值。
示例：

```css
.container {
  display: grid;
  grid-template-columns: 1fr 1fr minmax(10px, 50px);
}
```
页面效果：
![](https://img-blog.csdnimg.cn/20190822170040225.png)
如上图所示，第三列的宽度最小不小于10px，最大不超过50px。

### 3、grid-row-gap 属性，grid-column-gap 属性，grid-gap 属性

三个属性都是用来指定网格线(grid lines)的大小，grid-row-gap 属性定义水平网格线的大小，即行与行的间隔（行间距），grid-column-gap 属性定义垂直网格线的大小，即列与列的间隔（列间距），grid-gap 属性是 grid-column-gap 和 grid-row-gap 的合并简写形式，语法如下：

```css
grid-gap: <grid-row-gap> <grid-column-gap>;
```

示例：

```css
.container {
	display: grid;
	grid-template-columns: 1fr 1fr 1fr;
	grid-template-rows: 1fr 1fr 1fr;
	grid-row-gap: 20px;
	grid-column-gap: 20px;
}
```
页面效果：
![](https://img-blog.csdnimg.cn/20190822180201438.png)

上面代码等同于：

```css
.container {
 	display: grid;
 	grid-template-columns: 1fr 1fr 1fr;
 	grid-template-rows: 1fr 1fr 1fr;
 	grid-gap: 20px 20px;
}
```
注：

- 如果grid-gap省略了第二个值，浏览器认为第二个值等于第一个值。
- 根据最新标准，上面三个属性名的grid-前缀已经删除，grid-column-gap和grid-row-gap写成column-gap和row-gap，grid-gap写成gap。

### 4、grid-template-areas 属性
定义网格布局中网格区域(Grid Area)，一个网格区域由单个或多个单元格组成，重复网格区域的名称让区域内容跨越这些单元格。

取值：

- grid-area-name：由网格项的 grid-area 指定的网格区域名称
- .（点号） ：代表一个空的网格单元
- none：不定义网格区域

示例：

```css
grid-template-areas: 'a a a'
                     'b b b'
                     'c c c';
```
上面代码将9个单元格分成a、b、c三个区域。
示例：

```css
grid-template-areas: "header header header"
                     "main main sidebar"
                     "footer footer footer";
```

上面代码中，顶部是页眉区域header，底部是页脚区域footer，中间部分则为main和sidebar。

注意，区域的命名会影响到网格线。每个区域的起始网格线，会**自动命名**为区域名-start，终止网格线自动命名为区域名-end。比如，区域名为 header，则起始位置的水平网格线和垂直网格线叫做 header-start，终止位置的水平网格线和垂直网格线叫做 header-end。注意此处是自动命名，我们无法通过 grid-template-areas 属性来自定义网格线名称。

### 5、grid-auto-flow 属性

定义网格容器中网格项目（即容器内直接子元素）的自动排列方式，划分网格以后，容器的子元素会按照顺序，自动放置在每一个单元格中，grid-auto-flow 属性用来定义这种放置顺序。
取值：

- row 默认值，先行后列，即先填满第一行，再开始放入第二行
- column 先列后行，即先填满第一列，再开始放入第二列
- row dense 先行后列，尽可能紧密填满，不出现空格
- column dense 先列后行，尽可能紧密填满，不出现空格

示例：

```css
.container {
  	display: grid;
  	grid-template-columns: 100px 100px 100px;
  	grid-template-rows: 100px 100px 100px;
  	grid-auto-flow: column
}
```
页面效果：

![](https://img-blog.csdnimg.cn/20190823131122860.png)

关于row-dense和column-dense属性我们看下面示例：

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width">
  <title>Grid布局</title>
  <style>
  	.container {
  		display: grid;
  		grid-template-rows: 100px 100px 100px;
  		grid-template-columns: 100px 100px 100px;
  	}
  	.item {
  		text-align: center;
     	border: 1px solid gray;
  	}
  	.item-1 {
  		// grid-column-start 和 grid-column-end属性会在后面介绍
  		grid-column-start: 1;
  		grid-column-end: 3;  
  	}
  	.item-2 {
  		grid-column-start: 1;
  		grid-column-end: 3;
  	}
  </style>
</head>
<body>
<div class="container">
  <div class="item item-1">1</div>
  <div class="item item-2">2</div>
  <div class="item">3</div>
  <div class="item">4</div>
  <div class="item">5</div>
  <div class="item">6</div>
  <div class="item">7</div>
  <div class="item">8</div>
  <div class="item">9</div>
</div>
</body>
```

页面效果：
![](https://img-blog.csdnimg.cn/20190823140305391.png)

从上图我们可以看到1号网格项目和2号网格项目各占据了2个单元格，由于3号项目默认跟着2号项目，因此1号项目后面空白，并且7、8、9号元素由于没有设置行高度，所以高度表现出“包裹性”，因此在默认的`grid-auto-flow: row`情况下页面布局如上图所示。

现在给 container 元素添加`grid-auto-flow: row dense;`则页面效果如下：
![](https://img-blog.csdnimg.cn/20190823141711769.png)

将 grid-auto-flow 属性修改为 column dense，则页面效果如下图所示：
![](https://img-blog.csdnimg.cn/20190823142358912.png)

页面效果如上图所示，column dense 表示“先列后行”，先填满第一列，再填满第2列，所以3号项目在第一列，4号项目在第二列。8号项目和9号项目被挤到了第四列。因为没有指定第四列的宽度，所以第四列的宽度为最大宽度。

### 6、justify-items 属性

定义单元格内容的水平对齐方式（左中右），适用于网格容器里的所有网格项。

属性值：

- start: 左对齐。
- end: 右对齐。
- center: 居中对齐。
- stretch: 填满（默认）。

示例：

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width">
  <title>Grid布局</title>
  <style>
  	.container {
        display: grid;
        grid-template-rows: 100px 100px 100px;
        grid-template-columns: 100px 100px 100px;
  		justify-items: start;
   	}
   	.item {
  		width: 80px;
  		height: 80px;
    	text-align: center;
  		color: #fff;
       	border: 1px solid #fff;
  		background: lightblue;
   	}
  </style>
</head>
<body>
<div class="container">
  <div class="item">1</div>
  <div class="item">2</div>
  <div class="item">3</div>
  <div class="item">4</div>
  <div class="item">5</div>
  <div class="item">6</div>
  <div class="item">7</div>
  <div class="item">8</div>
  <div class="item">9</div>
</div>
</body>
```

页面效果：
![](https://img-blog.csdnimg.cn/20190823150241214.png)

将 justify-items 属性分别修改为 end、center可以分别看到单元格内容在水平方向右对齐、居中对齐，此处限于篇幅不再一一展示，

注意：只有当网格项目的宽度未指定时`justify-items: stretch`的页面效果才是单元格内容在水平方向铺满网格，否则页面效果跟`justify-items: start`一致。
示例：

```css
.container {
    display: grid;
    grid-template-rows: 100px 100px 100px;
    grid-template-columns: 100px 100px 100px;
    justify-items: stretch;
}
```

页面效果如图：
![](https://img-blog.csdnimg.cn/20190823151303734.png)

我们发现跟justify-items属性取start的页面效果相同，这是因为我们给网格子元素指定了宽高属性，所以并没有出现网格项铺满的效果，当我们去除网格项目的宽高属性时发现铺满了网格项，那么当我们不添加宽高属性时， justify-items 取 start 时页面效果又是怎么样呢，答案是如下图所示：
![](https://img-blog.csdnimg.cn/20190823152159249.png)

我们发现子元素在水平方向上表现出了宽度的“包裹性”，即由文本内容的宽度决定。

注意justify-items属性取stretch时（即为默认取值）的页面效果是实现单元格内容在水平方向铺满单元格，但当网元格给定宽度时页面效果表现为左对齐。

### 7、align-items 属性

定义单元格内容的垂直对齐方式（上中下），适用于网格容器里的所有网格项。
属性值：
- start：将网格项对齐到其单元格的顶部起始边缘（顶部对齐）
- end：将网格项对齐到其单元格的底部结束边缘（底部对齐）
- center：将网格项对齐到其单元格的垂直中间位置（垂直居中对齐）
- stretch：填满单元格的高度（默认值）

align-items属性基本与justify-items属性基本一致，主要区别在于align-items属性定义的是单元格内容在垂直方向的对齐，需要注意只有当网格项目的高度未指定时align-items: stretch的页面效果才是单元格内容在垂直方向铺满网格，否则页面效果跟align-items: start一致。

### 8、place-items属性

设置 align-items 和 justify-items 的简写形式。
代码形式：

```
place-items: <align-items> <justify-items>;
```
需要注意以下两点：

 - 如果省略第二个值，则浏览器认为与第一个值相等。
 - Edge浏览器不支持 place-items 简写属性（其他主流浏览器都支持）。

### 9、justify-content 属性

定义整个内容区域在容器里面的水平对齐方式（左中右）
属性值：

 - start：将网格对齐到 网格容器(grid container) 的左侧起始边缘（左侧对齐）
 - end：将网格对齐到 网格容器 的右侧结束边缘（右侧对齐）
 - center：将网格对齐到 网格容器 的水平中间位置（水平居中对齐）
 - stretch：调整 网格项(grid items) 的宽度，允许该网格填充满整个网格容器的宽度
 - space-around：在每个网格项之间放置一个均匀的空间，左右两端放置一半的空间
 - space-between：在每个网格项之间放置一个均匀的空间，左右两端没有空间
 - space-evenly：在每个网格项目之间放置一个均匀的空间，左右两端放置一个均匀的空间

示例：

```css
.container {
	width: 350px;
	height: 350px;
	border: 2px dashed gray;
	display: grid;
    grid-template-rows: 100px 100px 100px;
    grid-template-columns: 100px 100px 100px;
    justify-content: start;
}
.item {
	text-align: center;
  	color: #fff;
    border: 1px solid #fff;
  	background: lightblue;    
}
```

页面效果：
![](https://img-blog.csdnimg.cn/20190823163611291.png)

```css
justify-content: end;
```

![](https://img-blog.csdnimg.cn/20190823163658809.png)

```css
justify-content: center;
```

页面效果：
![](https://img-blog.csdnimg.cn/20190823163921251.png)

```css
justify-content: space-around;
```

页面效果：
![](https://img-blog.csdnimg.cn/20190823164255875.png)

```css
justify-content: space-between;
```

页面效果：
![](https://img-blog.csdnimg.cn/20190823164444837.png)

```css
justify-content: space-evenly;
```

页面效果：
![](https://img-blog.csdnimg.cn/20190823164648228.png)

注意：只有当项目宽度没有指定时，justify-content: stretch;才会使得整个内容区域在水平方向拉伸占据整个网格容器，如果指定了项目宽度页面效果与justify-content: start;相同。

### 10、align-content
定义整个内容区域在容器里面的垂直对齐方式（上中下）

属性值：

- start：将网格对齐到 网格容器(grid container) 的顶部起始边缘（顶部对齐）
- end：将网格对齐到 网格容器 的底部结束边缘（底部对齐）
- center：将网格对齐到 网格容器 的垂直中间位置（垂直居中对齐）
- stretch：调整 网格项(grid items) 的高度，允许该网格填充满整个 网格容器 的高度
- space-around：在每个网格项之间放置一个均匀的空间，上下两端放置一半的空间
- space-between：在每个网格项之间放置一个均匀的空间，上下两端没有空间
- space-evenly：在每个网格项目之间放置一个均匀的空间，上下两端放置一个均匀的空间

align-content 属性基本与 justify-content 属性基本一致，主要区别在于 align-content 属性定义的是整个内容区域在容器里面的垂直方向对齐方式，需要注意只有当网格项目的高度未指定时 align-content: stretch 的页面效果才是内容区域在垂直方向铺满网格容器，否则页面效果跟 align-content: start 一致。

### 11、place-content属性

定义 align-content 和 justify-content 的简写形式。

代码形式：

```
place-content: <align-content> <justify-content>
```
需要注意以下两点：

 - 如果省略第二个值，则浏览器认为与第一个值相等。
 - Edge浏览器不支持 place-content 简写属性（其他主流浏览器都支持）。

### 12、grid-auto-columns / grid-auto-rows 属性

网格布局中，当网格中的网格项多于单元格时，或者当网格项位于显式网格之外时，比如网格只有3列，但是某一个项目指定在第5列，这些情况下就会创建隐式网格，以便放置项目。
grid-auto-columns属性和grid-auto-rows属性用来设置浏览器自动创建的隐式网格的列宽和行高。它们的写法与grid-template-columns和grid-template-rows完全相同。如果不指定这两个属性，浏览器完全根据单元格内容的大小，决定新增网格的列宽和行高。

示例：

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width">
  <title>Grid 布局</title>
  <style>
  	.container{
  		display: grid;
  		grid-template-columns: 100px 100px 100px;
  		grid-template-rows: 100px 100px 100px;
  		grid-auto-rows: 50px; 
	}
	.item {
  		font-size: 2em;
  		text-align: center;
  		background: lightblue;
  		border: 1px solid #fff;
	}
	.item-8 {
		background-color: #d0e4a9;
  		grid-row-start: 4;
  		grid-column-start: 2;
  	}
  	.item-9 {
  		background-color: #4dc7ec;
  		grid-row-start: 5;
  		grid-column-start: 3;
  	}
  </style>
</head>
<body>
<div class="container">
  <div class="item">1</div>
  <div class="item">2</div>
  <div class="item">3</div>
  <div class="item">4</div>
  <div class="item">5</div>
  <div class="item">6</div>
  <div class="item">7</div>
  <div class="item item-8">8</div>
  <div class="item item-9">9</div>
</div>
</body>
</html>
```
页面效果：
![](https://img-blog.csdnimg.cn/20190824110023455.png)

由于8号项目和9号项目超出了显式网格的行数，因此我们可以通过grid-auto-rows定义其网格项的行高。那么我们再给网格容器添加 “grid-auto-columns: 50px;” 能不能使得其列宽也只有50px呢？答案是不能，这是因为8号和9号项目的列数并没有超出显式网格，如果我们需要定义隐式网格的列宽看如下示例：

```css
.container{
    display: grid;
    grid-template-columns: 100px 100px 100px;
    grid-template-rows: 100px 100px 100px;
    grid-auto-columns: 50px; 
}
.item-8 {
  	background-color: #d0e4a9;
    grid-row-start: 3;
    grid-column-start: 4;
}
.item-9 {
    background-color: #4dc7ec;
    grid-row-start: 3;
    grid-column-start: 5;
}
```

页面效果：
![](https://img-blog.csdnimg.cn/2019082411260175.png)

我们可以发现4号项目和5号项目也发生了改变，想要了解原因需要了解隐式网格的定义：
如果网格项的数量多于网格单元格，或者网格项位于显式网格外部，则网格容器会通过向网格添加网格线自动生成网格轨道，显式网格与这些额外的隐式轨道和网格线一起形成所谓的隐式网格（更多关于隐式网格可以查看[显式网格和隐式网格之间的区别](https://www.html.cn/archives/10327)）。
我们再来看上面两个示例，两个网格项放置在显式网格之外，导致创建隐式网格线条和网格轨道，grid-auto-columns 属性和 grid-auto-rows 属性设置了隐式网格的列宽和行高，所有网格项目在默认情况下在隐式网格中按照“先行后列”的顺序进行排列，因此呈现出了上面两个示例的页面效果。

补充：grid-auto-columns，grid-auto-rows，和 grid-auto-flow 属性都属于隐形网格属性，grid-auto-flow 属性也可以定义隐形网格中网格项的自动布局方式。

### 13、grid-template 属性

grid-template 属性是 grid-template-columns、grid-template-rows 和 grid-template-areas 这三个属性的合并简写形式。

示例：

```css
.container {  
	grid-template:
		[row1-start] "header header header" 25px [row1-end]
		[row2-start] "footer footer footer" 25px [row2-end]
		/ auto 50px auto;
}
```

等价于：

```css
.container {  
	grid-template-rows: [row1-start] 25px [row1-end row2-start] 25px [row2-end];  
	grid-template-columns: auto 50px auto;  
	grid-template-areas: 
		"header header header"     
		"footer footer footer";
}
```

使用grid-template属性需要注意以下两点：

- grid-template 属性的取值还可以直接为none，是将所有三个属性设置为其初始值
- grid-template 不会重置隐式网格属性（grid-auto-columns，grid-auto-rows，和 grid-auto-flow)

### 14、grid属性

在一个声明中设置所有以下属性的简写： grid-template-rows, grid-template-columns, grid-template-areas, grid-auto-rows, grid-auto-columns, 和 grid-auto-flow 。（注意：只能在单个网格声明中指定显式或隐式网格属性）。
属性值：

 - none：将所有子属性设置为其初始值。
 - &lt;grid-template&gt;：与grid-template 简写的工作方式相同。
 - &lt;grid-template-rows&gt; / [ auto-flow && dense? ] &lt;grid-auto-columns&gt;? ：将grid-template-rows 设置为指定的值。 如果 auto-flow 关键字位于斜杠的右侧，则会将 grid-auto-flow 设置为 column。 如果另外指定了 dense 关键字，则自动放置算法使用 “dense” 算法。 如果省略 grid-auto-columns ，则将其设置为 auto。
 - [ auto-flow && dense? ] &lt;grid-auto-rows&gt;? / &lt;grid-template-columns>：将 grid-template-columns 设置为指定值。 如果 auto-flow 关键字位于斜杠的左侧，则会将grid-auto-flow 设置为 row 。 如果另外指定了 dense 关键字，则自动放置算法使用 “dense” 打包算法。 如果省略 grid-auto-rows ，则将其设置为 auto。

示例1：

```css
.container {  
	grid: 100px 300px / 3fr 1fr;
}
```

等价于：

```css
.container {  
	grid-template-rows: 100px 300px;  
	grid-template-columns: 3fr 1fr;
}
```

示例2：

```css
.container {  
	grid: auto-flow / 200px 1fr;
}
```

等价于：

```css
.container {  
	grid-auto-flow: row;  
	grid-template-columns: 200px 1fr;
}
```

示例3：

```css
.container {  
	grid: auto-flow dense 100px / 1fr 2fr;
}
```

等价于：

```css
.container {  
	grid-auto-flow: row dense;  
	grid-auto-rows: 100px;  
	grid-template-columns: 1fr 2fr;
}
```

示例4：

```css
.container {  
	grid: 100px 300px / auto-flow 200px;
}
```

等价于：

```css
.container {  
	grid-template-rows: 100px 300px;  
	grid-auto-flow: column;  
	grid-auto-columns: 200px;
}
```

示例5：

```css
.container {  
	grid: 
		[row1-start] "header header header" 1fr [row1-end]        
		[row2-start] "footer footer footer" 25px [row2-end]       
		/ auto 50px auto;
}
```

等价于：

```css
.container {  
	grid-template-areas:     
		"header header header"    
		"footer footer footer";  
	grid-template-rows: [row1-start] 1fr [row1-end row2-start] 25px [row2-end];  
	grid-template-columns: auto 50px auto;    
}
```

## 三、网格项目属性

### 1、grid-column-start / grid-column-end / grid-row-start / grid-row-end属性

通过引用特定网格线来确定网格项在网格内的位置。 grid-column-start / grid-row-start 是网格项开始的网格线，grid-column-end / grid-row-end 是网格项结束的网格线。

属性值：

- &lt;line> ：用网格线名称来引用对应网格线。
- span &lt;number> ：该网格项将跨越所提供的网格轨道数量。
- span &lt;name> ：该网格项将跨越到它与提供的名称位置
- auto：表示自动放置，自动跨度，默认会扩展一个网格轨道的宽度或者高度

示例：

```css
.container {
    display: grid;
    grid-template-rows: [r1]100px [r2] 100px [r3] 100px [r4];
    grid-template-columns: [c1]100px [c2] 100px [c3] 100px [c4];
    grid-auto-rows: 100px;
}
.item-8 {
    grid-column-start: c2;
    grid-column-end: c4;
}
```
页面效果：
![](https://img-blog.csdnimg.cn/20190824135547968.png)

在上面示例中我们用网格线的名字来引用一个命名的网格线，在介绍 grid-template-columns，grid-template-rows 属性中的2.1网格线名称中说到如果没有直接指定网格线的名称网格线会自动分配正数和负数名称，但实际上即使我们指定了网格线名字我们依然可以通过正数或负数名称来引用网格线，示例：

```css
.container {
    display: grid;
    grid-template-rows: [r1]100px [r2] 100px [r3] 100px [r4];
    grid-template-columns: [c1]100px [c2] 100px [c3] 100px [c4];
    grid-auto-rows: 100px;
}
.item-8 {
    grid-column-start: 2;
    grid-column-end: 4;
    // 等同于grid-column-start: -3; grid-column-end: -1;
}
```

页面效果：
![](https://img-blog.csdnimg.cn/20190824140859556.png)

因此无论我们是否在grid-template-columns，grid-template-rows属性中指定网格线名称来引用网格线，因此大家最好使用正负数来引用对应网格线。看到这儿大家可能会好奇假如我们没有指定网格线名称，但是依然使用网格线名称来引用网格线会产生什么页面结果？示例：

```css
.container {
    display: grid;
    grid-template-rows: 100px 100px 100px;
    grid-template-columns: 100px 100px 100px;
  	grid-auto-rows: 100px;
  }
.item-8 {
  	grid-column-start: c2;
  	grid-column-end: c4;
 }
```

页面效果：
![](https://img-blog.csdnimg.cn/20190824141446725.png)

从上图我们可以发现8号项目结束的网格线实际上就是网格容器的右边界的位置，因此如果网格容器大于网格内容，此时8号项目就会位于显式网格的外部，因此形成了隐式网格，所以页面布局如上图所示，这儿需要大家注意。

示例：

```css
.container {
    display: grid;
    grid-template-rows: 100px 100px 100px;
    grid-template-columns: 100px 100px 100px;
  	grid-auto-rows: 100px;
}
.item-8 {
  	grid-column-start: -4;
  	grid-column-end: span 2;
 }
```

上面代码的含义为网格项开始为名字为-4对应的网格线，然后跨越2个网格轨道，因此页面效果如下图所示：
![](https://img-blog.csdnimg.cn/20190824142846189.png)

属性还可以取span &lt;name>表示跨越到名字为&lt;name>的网格线位置，示例：

```
.container {
    display: grid;
    grid-template-rows: [r1] 100px [r2] 100px [r3] 100px [r4];
    grid-template-columns: [c1] 100px [c2] 100px [c3] 100px [c4];
   	grid-auto-rows: 100px;
}
.item-8 {
   	grid-column-start: c1;
   	grid-column-end: span c3;
}
```
页面效果：
![](https://img-blog.csdnimg.cn/20190824143449405.png)

grid-row-start、grid-row-end属性和grid-column-start、grid-column-end属性类型，仅仅是分别定义水平位置和垂直位置的区别。

### 2、grid-column / grid-row 属性

grid-column 属性是 grid-column-start 和grid-column-end 的合并简写形式，grid-row 属性是 grid-row-start 属性和 grid-row-end 的合并简写形式。

代码形式：

```css
grid-column: <grid-column-start> / <grid-column-end>;
grid-row: <grid-row-start> / <grid-row-end>;
```
注意如果没有声明分隔线结束位置，则该网格项默认占据 1 个网格轨道。

### 3、grid-area属性
指定项目放在哪一个区域，示例：

```css
.container{
  	display: grid;
  	grid-template-columns: 100px 100px 100px;
  	grid-template-rows: 100px 100px 100px;
  	grid-template-areas: 'a b c'
                      	 'd e f'
                         'g h i';
}
.item-8 {
	grid-area: e;
}
```

页面效果：
![](https://img-blog.csdnimg.cn/20190824145444704.png)

grid-area属性还可用作 grid-row-start、grid-column-start、grid-row-end、grid-column-end 的合并简写形式，直接指定项目的位置。

代码形式：

```css
grid-area: <row-start> / <column-start> / <row-end> / <column-end>;
```

因此下面示例的页面效果跟上面一致。

```css
.item-8 {
  grid-area: 1 / 1 / 3 / 3;
}
```
### 4、justify-self/align-self属性

justify-self 属性设置单元格内容的水平位置（左中右），跟 justify-items 属性的用法完全一致，但只作用于单个项目。
align-self 属性设置单元格内容的垂直位置（上中下），跟 align-items 属性的用法完全一致，也是只作用于单个项目。

属性值：

- start：对齐单元格的起始边缘。
- end：对齐单元格的结束边缘。
- center：单元格内部居中。
- stretch：拉伸，占满单元格的整个宽度（默认值）。

注意stretch属性只有在单元格内容没有自定义大小的时候才会生效，否则默认效果为对齐单元格的起始边缘。

示例：

```css
.container {
    display: grid;
    grid-template-rows: 100px 100px 100px;
    grid-template-columns: 100px 100px 100px;
}
.item-8 {
  	width: 80px;
  	height: 80px;
  	justify-self: center;
  	align-self: center;
}
```
页面效果：
![](https://img-blog.csdnimg.cn/20190824150638296.png)

限于篇幅本文不再一一展示其他属性的页面效果，大家可以自己一一尝试一下。

### 5、place-self 属性

place-self 属性是 align-self 属性和 justify-self 属性的合并简写形式。

```css
place-self: <align-self> <justify-self>;
```

注意如果省略第二个值，place-self属性会认为这两个值相等。

## 四、Grid布局中的动画

根据 CSS Grid 布局模块 Level 1 规范，有 5 个可应用动画的网格属性：

- grid-gap， grid-row-gap，grid-column-gap 作为长度，百分比或 calc。
- grid-template-columns，grid-template-rows 作为长度，百分比或 calc 的简单列表，只要列表中长度、百分比或calc组件的值不同即可。

浏览器支持可设置动画的网格属性：
![](https://img-blog.csdnimg.cn/20190824151201370.png)
示例：

```html
<!DOCTYPE html>
<html>
<head>
	<meta charset="utf-8">
  	<meta name="viewport" content="width=device-width">
  	<title>Grid布局</title>
  	<style>
 		.container {
            display: grid;
            grid-template-rows: 100px 100px 100px;
            grid-template-columns: 100px 100px 100px;
            grid-gap: 5px;
            transition: grid-gap 2s;
   	 		-moz-transition: grid-gap 2s; /* Firefox 4 */
            -webkit-transition: grid-gap 2s; /* Safari 和 Chrome */
            -o-transition: grid-gap 2s; /* Opera */
    	}
    	.grid-full {
    		grid-gap: 10px;
  		}
        .item {
            text-align: center;
            color: #fff;
            border: 1px solid #fff;
            background: lightblue;
        } 
        .button {
            background-color: green;
            color: #ffffff;
            margin: 10px auto;
            border: none;
            border-radius: 5px;
  		}
	</style>
</head>
<body>
	<button class="button">开始</button>
	<div class="container">
		<div class="item">1</div>
  		<div class="item">2</div>
  		<div class="item">3</div>
  		<div class="item">4</div>
  		<div class="item">5</div>
  		<div class="item">6</div>
  		<div class="item">7</div>
  		<div class="item">8</div>
  		<div class="item">9</div>
	</div>
</body>
<script>
 	document.querySelector('.button').addEventListener('click', function() {
  		document.querySelector('.container').classList.toggle('grid-full')
 	})
</script>
</html>
```
