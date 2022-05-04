---
title: 深入解析 SourceMap
date: 2021-07-11 07:57:42
toc: true
mathjax: false
categories: 
- 前端工程化
tags: 
- webpack
---

## 什么是 SourceMap

在前端开发过程中，通常我们编写的源代码会经过多重处理（编译、封装、压缩等），最后形成产物代码。于是在浏览器中调试产物代码时，我们往往会发现代码变得面目全非，因此，我们需要一种在调试时将产物代码显示回源代码的功能，SourceMap 就是实现这一目标的工具。

SourceMap 的基本原理是，在编译处理的过程中，在生成产物代码的同时生成产物代码中被转换的部分与源代码中相应部分的映射关系表。有了这样一张完整的映射表，我们就可以通过 Chrome 控制台中的"Enable Javascript source map"来实现调试时的显示与定位源代码功能。

注：我们在控制台的网络面板中通常看不到 source map 文件的请求，其原因是出于安全考虑 Chrome 隐藏了 source map 的请求，需要通过 [net-log](chrome://net-export/) 来查询。

<!-- more -->

## Webpack 中 SourceMap 的预设

在 Webpack 中，通过设置 devtool 来选择 source map 的预设类型，文档中共有 [20 余种](https://webpack.js.org/configuration/devtool/#devtool) source map 的预设，这些预设通常包含了 "eval" "cheap" "module" "inline" "hidden" "nosource" "source-map" 等关键字的组合，这些关键字的具体逻辑如下：

```js
// webpack/lib/WebpackOptionsApply.js:232 

if (options.devtool.includes("source-map")) {
  const hidden = options.devtool.includes("hidden"); 
  const inline = options.devtool.includes("inline"); 
  const evalWrapped = options.devtool.includes("eval"); 
  const cheap = options.devtool.includes("cheap"); 
  const moduleMaps = options.devtool.includes("module"); 
  const noSources = options.devtool.includes("nosources"); 

  const Plugin = evalWrapped 
    ? require("./EvalSourceMapDevToolPlugin") 
    : require("./SourceMapDevToolPlugin"); 

  new Plugin({ 
    filename: inline ? null : options.output.sourceMapFilename, 
    moduleFilenameTemplate: options.output.devtoolModuleFilenameTemplate, 
    fallbackModuleFilenameTemplate: options.output.devtoolFallbackModuleFilenameTemplate, 
    append: hidden ? false : undefined, 
    module: moduleMaps ? true : cheap ? false : true, 
    columns: cheap ? false : true, 
    noSources: noSources, 
    namespace: options.output.devtoolNamespace 
  }).apply(compiler); 
} else if (options.devtool.includes("eval")) { 
  const EvalDevToolModulePlugin = require("./EvalDevToolModulePlugin"); 
  new EvalDevToolModulePlugin({ 
    moduleFilenameTemplate: options.output.devtoolModuleFilenameTemplate, 
    namespace: options.output.devtoolNamespace 
  }).apply(compiler); 
}
```

如上面的代码所示， devtool 的值匹配并非精确匹配，某个关键字只要包含在赋值中即可获得匹配，不同关键字的具体作用如下：

- false：即不开启 source map 功能，其他不符合上述规则的赋值也等价于 false。
- eval：是指在编译器中使用 EvalDevToolModulePlugin 作为 source map 的处理插件。
- [xxx-...]source-map：根据 devtool 对应值中是否有 eval 关键字来决定使用 EvalSourceMapDevToolPlugin 或 SourceMapDevToolPlugin 作为 source map 的处理插件，其余关键字则决定传入到插件的相关字段赋值。
- inline：决定是否传入插件的 filename 参数，作用是决定单独生成 source map 文件还是在行内显示，该参数在 eval- 参数存在时无效。
- hidden：决定传入插件 append 的赋值，作用是判断是否添加 SourceMappingURL 的注释，该参数在 eval- 参数存在时无效。
- module：为 true 时传入插件的 module 为 true ，作用是为加载器（Loaders）生成 source map，因此可以查看到转换前的代码。
- cheap：这个关键字有两处作用。首先，当 module 为 false 时，它决定插件 module 参数的最终取值，最终取值与 cheap 相反。其次，它决定插件 columns 参数的取值，作用是决定生成的 source map 中是否包含列信息，在不包含列信息的情况下，调试时只能定位到指定代码所在的行而定位不到所在的列。
- nosource：nosource 决定了插件中 noSource 变量的取值，作用是决定生成的 source map 中是否包含源代码信息，不包含源码情况下只能显示调用堆栈信息。

从上面的规则中我们还可以看到，根据不同规则，实际上 Webpack 是从三种插件中选择其一作为 source map 的处理插件。

- EvalDevToolModulePlugin：模块代码后添加 sourceURL=webpack:///+ 模块引用路径，不生成 source map 内容，模块产物代码通过 eval() 封装。
- EvalSourceMapDevToolPlugin：生成 base64 格式的 source map 并附加在模块代码之后， source map 后添加 sourceURL=webpack:///+ 模块引用路径，不单独生成文件，模块产物代码通过 eval() 封装。
- SourceMapDevToolPlugin：生成单独的 .map 文件，模块产物代码不通过 eval 封装。

通过上面的代码分析，我们了解了不同参数在 Webpack 运行时起到的作用。那么这些不同参数组合下的各种预设对我们的 source map 生成又各自会产生什么样的效果呢？下面我们通过示例来看一下。

<img src="/assets/sourcemap/01.png" />

> *注1：“/”前后分别表示产物 js 大小和对应 .map 大小。
> *注2：“/”前后分别表示初次构建时间和开启 watch 模式下 rebuild 时间。对应统计的都是 development 模式下的笔者机器环境下几次构建时间的平均值，只作为相对快慢与量级的比较。

从结果中可以看到，eval- 对应的 EvalSourceMapDevToolPlugin 整体要快于不带 eval- 的 SourceMapDevToolPlugin。尤其在质量最佳的配置下，eval-source-map 的再次构建速度要远快于其他几种，这是为什么呢？

原作者对于其的解释如下：

> devtool: "source-map" cannot cache SourceMaps for modules and need to regenerate complete SourceMap for the chunk. It's something for production.
devtool: "eval-source-map" is really as good as devtool: "source-map", but can cache SourceMaps for modules. It's much faster for rebuilds.

可以看到加 eval 和不加的功能是一样的，但加了 eval 后可利用字符串可缓存从而提效，因此再次构建速度更快。

在开发环境和生产环境下，我们对于 source map 功能的期望也有所不同：

我们在开发环境对 sourceMap 的要求是：快（eval），信息全（module），且由于此时代码未压缩，我们并不那么在意代码列信息(cheap),所以开发环境比较推荐配置是 cheap-module-eval-source-map

生产环境中一般情况下，我们并不希望任何人都可以在浏览器直接看到我们未编译的源码，所以我们不应该直接提供 sourceMap 给浏览器。但我们又需要 sourceMap 来定位我们的错误信息，一方面 webpack 会生成 sourcemap 文件以提供给错误收集工具比如 sentry，另一方面又不会为 bundle 添加引用注释，以避免浏览器使用。这时我们可以选择 hidden-source-map

除了通过 devtool 属性指定关键字外我们还可以通过自定义插件实现 source map 的生成，以 EvalSourceMapDevToolPlugin 为例，在 EvalSourceMapDevToolPlugin 的传入参数中，除了上面和预设相关的 filename、append、module、columns 外，还有影响注释内容的 moduleFilenameTemplate 和 protocol，以及影响处理范围的 test、include、exclude。这里重点看处理范围的参数，因为通常我们需要调试的是开发的业务代码部分，而非依赖的第三方模块部分。因此在生成 source map 的时候如果可以排除第三方模块的部分而只生成业务代码的 source map，无疑能进一步提升构建的速度，例如示例：

```js
// webpack.config.js
module.exports = {
    ... 
  //devtool: 'eval-source-map', 
  devtool: false, 
  plugins: [ 
    new webpack.EvalSourceMapDevToolPlugin({ 
      exclude: /node_modules/, 
      module: true, 
      columns: false 
    }) 
  ], 
  ...
}
```

在上面的示例中，我们将 devtool 设为 false，而直接使用 EvalSourceMapDevToolPlugin，通过传入 module: true 和 column:false，达到和预设 eval-cheap-module-source-map 一样的质量，同时传入 exclude 参数，排除第三方依赖包的 source map 生成。保存设定后通过运行可以看到，在文件体积减小的同时，再次构建的速度相比上面表格中的速度提升了将近一倍，达到了最快一级。

## SourceMap 实现的原理是什么

我们在前文提到 SourceMap 的主要作用就是在生成产物代码的同时生成产物代码中被转换的部分与源代码中相应部分的映射关系表，那么 SourceMap 到底怎么做到源文件和处理后文件映射的？

### 一、map 文件详解

要分析实现，还是得先从现象下手，假定源文件 script.js 内容为

```js
let a=1;
let b=2;
let c=3;
```

其输出内容为 script-min.js

```js
var a=1,b=2,c=3;
```

对应生成的 source map 文件 script-min.js.map:

```map
{"version":3,"file":"script-min.js","lineCount":1,"mappings":"AAAA,IAAIA,EAAE,CAAN,CACIC,EAAE,CADN,CAEIC,EAAE;","sources":["script.js"],"names":["a","b","c"]}
```

各个字段的具体含义如下：

| 字段 | 含义                               |
| ---- | ---------------------------------- |
| version    | Source map 的版本，目前为 3               |
| file    | 转换后的文件名 |
| sourceRoot    | 转换前的文件所在的目录。如果与转换前的文件在同一目录，该项为空            |
| sources    | 转换前的文件,该项是一个数组,表示可能存在多个文件合并   |
| names    | 转换前的所有变量名和属性名    |
| mappings    | 记录位置信息的字符串     |


可以看到，既然我们要定位，自然最关心的是具有【记录位置信息】功能的 mapping 属性，接下来详细讲解如何分析 mapping。

**mappings 属性值的含义**：

| 分析角度 | 含义                               |
| ---- | ---------------------------------- |
| 行对应    | 以分号（;）表示，每个分号对应转换后源码的一行。所以，第一个分号前的内容，就对应源码的第一行，以此类推。              |
| 位置对应  | 以逗号（,）表示，每个逗号对应转换后源码的一个位置。所以，第一个逗号前的内容，就对应该行源码的第一个位置，以此类推。 |
| 分词信息  | 以 VLQ 编码表示，代表记录该位置对应的转换前的源码位置、原来属于那个文件等信息。            |

- 【行对应】很好理解，即一个分号为一行，因为压缩后基本上都是一行了，所以这个没啥有用信息；
- 【位置对应】可以理解为分词，每个逗号对应转换后源码的一个位置；
- 【分词信息】是关键，如AAAA代表该位置转换前的源码位置，以VLQ编码表示；

举例来说，假定mappings属性的内容如下：

```
mappings:"AAAAA,BBBBB;CCCCC"
```

就表示，转换后的源码分成两行，第一行有两个位置，第二行有一个位置。

其中【分词信息】每组最多五位（如果不是变量，只会有四位），分别是：

- 第一位，表示这个位置在【转换后代码】的第几列。
- 第二位，表示这个位置属于【sources 属性】中的哪一个文件。
- 第三位，表示这个位置属于【转换前代码】的第几行。
- 第四位，表示这个位置属于【转换前代码】的第几列。
- 第五位，表示这个位置属于【names 属性】的哪一个变量。

有几点需要说明。首先，所有的值都是以0作为基数的。其次，第五位不是必需的，如果该位置没有对应names属性中的变量，可以省略第五位。再次，每一位都采用VLQ编码表示；由于VLQ编码是变长的，所以每一位可以由多个字符构成。

如果某个位置是AAAAA，由于A在VLQ编码中表示0，因此这个位置的五个位实际上都是0。它的意思是，该位置在转换后代码的第0列，对应sources属性中第0个文件，属于转换前代码的第0行第0列，对应names属性中的第0个变量。

到此，我们也算是知道 map 文件到底是怎么组成的了。

### 二、VLQ 编码

VLQ 编码最早用于MIDI文件，后来被多种格式采用。它的特点就是可以非常精简地表示很大的数值。

VLQ编码是变长的。如果（整）数值在-15到+15之间（含两个端点），用一个字符表示；超出这个范围，就需要用多个字符表示。它规定，每个字符使用6个两进制位，正好可以借用Base 64编码的字符表。

<img src="/assets/02.png" width="500" />

在这6个位中，左边的第一位（最高位）表示是否"连续"（continuation）。如果是1，代表这６个位后面的6个位也属于同一个数；如果是0，表示该数值到这6个位结束。

```
Continuation
|　　　　　Sign
|　　　　　|
V　　　　　V
１０１０１１
```

这6个位中的右边最后一位（最低位）的含义，取决于这6个位是否是某个数值的VLQ编码的第一个字符。如果是的，这个位代表"符号"（sign），0为正，1为负（Source map的符号固定为0）；如果不是，这个位没有特殊含义，被算作数值的一部分。

下面看一个例子，如何对数值16进行VLQ编码。

- 第一步，将16改写成二进制形式10000。
- 第二步，在最右边补充符号位。因为16大于0，所以符号位为0，整个数变成100000。
- 第三步，从右边的最低位开始，将整个数每隔5位，进行分段，即变成1和00000两段。如果最高位所在的段不足5位，则前面补0，因此两段变成00001和00000。
- 第四步，将两段的顺序倒过来，即00000和00001。
- 第五步，在每一段的最前面添加一个"连续位"，除了最后一段为0，其他都为1，即变成100000和000001。
- 第六步，将每一段转成Base 64编码。

查表可知，100000为g，000001为B。因此，数值16的VLQ编码为gB。上面的过程，看上去好像很复杂，做起来其实很简单，具体的实现请看官方的[base64-vlq.js](https://github.com/mozilla/source-map/blob/master/lib/source-map/base64-vlq.js)文件，里面有详细的注释。


## 参考资料

[万字长文：关于sourcemap，这篇文章就够了](https://juejin.cn/post/6969748500938489892)

[JavaScript Source Map 详解](http://www.ruanyifeng.com/blog/2013/01/javascript_source_map.html)

拉勾教育课程《前端工程化精讲》