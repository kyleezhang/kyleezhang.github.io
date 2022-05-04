---
title: Webpack中的CodeSplitting
date: 2021-01-03 10:42:50
toc: true
mathjax: false
categories: 
- 前端工程化
tags: 
- webpack
---

通过 Webpack 实现前端项目整体模块化的优势固然明显，但是它也会存在一些弊端：如果我们的应用非常复杂，这种 All in One 的打包方式就会导致打包的结果过大，然后在绝大多数情形下应用刚开始工作时并不是所有模块都是必须的，更为合理的方案是**把打包的结果按照一定的规则分离到多个 bundle 中，然后根据应用的运行按需加载**。这样就可以降低启动成本，提高响应速度。
<!-- more -->
为了解决打包结果过大导致的问题，Webpack 设计了一种分包功能：Code Splitting(代码分割)。

Code Splitting 通过把项目中的资源模块按照我们设计的规则打包到不同的 bundle 中，从而降低应用的启动成本，提高响应速度，其实现方式主要有两种：

- 根据业务不同配置多个打包入口，输出多个打包结果；
- 结合 ES Modules 的动态导入（Dynamic Imports）特性，按需加载模块。

## 多入口打包

多入口打包一般适用于传统的多页应用程序，最常见的划分规则就是一个页面对应一个打包入口，对于不同页面间公用的部分，再提取到公共的结果中。举个🌰:

```
// 项目目录
.
├── dist
├── src
│   ├── common
│   │   └── fetch.js
│   ├── album.html
│   ├── album.js
│   ├── index.html
│   └── index.js
├── package.json
└── webpack.config.js
```

```typescript
// ./webpack.config.js
const HtmlWebpackPlugin = require('html-webpack-plugin')
module.exports = {
  entry: {
    index: './src/index.js',
    album: './src/album.js'
  },
  output: {
    filename: '[name].bundle.js' // [name] 是入口名称
  },

  // ... 其他配置
  plugins: [
    new HtmlWebpackPlugin({
      title: 'Multi Entry',
      template: './src/index.html',
      filename: 'index.html',
      chunks: ['index'] // 指定使用 index.bundle.js
    }),

    new HtmlWebpackPlugin({
      title: 'Multi Entry',
      template: './src/album.html',
      filename: 'album.html',
      chunks: ['album'] // 指定使用 album.bundle.js
    })
  ]
}
```

一般 entry 属性中只会配置一个打包入口，如果我们需要配置多个入口，可以把 entry 定义成一个对象。注意这里 entry 是定义为对象而不是数组，如果是数组的话就是把多个文件打包到一起，还是一个入口。在这个对象中一个属性就是一个入口，属性名称就是这个入口的名称，值就是这个入口对应的文件路径。那我们这里配置的就是 index 和 album 页面所对应的 JS 文件路径。

一旦我们的入口配置为多入口形式，那输出文件名也需要修改，因为两个入口就有两个打包结果，不能都叫 bundle.js。我们可以在这里使用 `[name]` 这种占位符来输出动态的文件名，`[name]` 最终会被替换为入口的名称。

除此之外，在配置中还通过 html-webpack-plugin 分别为 index 和 album 页面生成了对应的 HTML 文件。

### 提取公共模块

多入口打包本身非常容易理解和使用，但是它也存在一个小问题，就是不同的入口中一定会存在一些公共使用的模块，如果按照目前这种多入口打包的方式，就会出现多个打包结果中有相同的模块的情况。

例如我们上述案例中，index 入口和 album 入口中就共同使用了 fetch.js 这个公共的模块。这里是因为我们的示例比较简单，所以重复的影响没有那么大，但是如果我们公共使用的是 jQuery 或者 Vue.js 这些体积较大的模块，那影响就会比较大，不利于公共模块的缓存。

所以我们还需要把这些公共的模块提取到一个单独的 bundle 中。Webpack 4 中实现公共模块提取非常简单，我们只需要在优化配置中开启 splitChunks 功能就可以了，具体配置如下：

```typescript
// ./webpack.config.js
module.exports = {
  entry: {
    index: './src/index.js',
    album: './src/album.js'
  },

  output: {
    filename: '[name].bundle.js' // [name] 是入口名称
  },

  optimization: {
    splitChunks: {
      // 自动提取所有公共模块到单独 bundle
      chunks: 'all'
    }
  }
  // ... 其他配置
}
```

配置完成以后我们打开命令行终端，再次运行 Webpack 打包，打包结果如下图：

<img src="/assets/webpack-codesplitting/01.png" width="603px" />

示例中提取公共模块的核心 SplitChunks 是由 webpack 4 内置的 SplitChunksPlugin 插件提供的能力，可直接在 optimization 选项中配置，可选参数如下：

- chunks：表示从哪些chunks里面抽取代码，除了三个可选字符串值 initial、async、all 之外（all 代表所有模块，async代表异步加载的模块, initial代表初始化时就能获取的模块），还可以通过函数来根据 chunk 参数的 name 等属性过滤所需的 chunks；
- minSize：表示抽取出来的文件在压缩前的最小大小，如果一个模块符合其他的拆分规则，但是如果提取出来最后生成文件大小比 minSize 要小，那它仍然不会被提取出来。这个属性可以在每个缓存组属性中设置，也可以在splitChunks属性中设置，这样在每个缓存组都会继承这个配置，默认为 30000；
- maxSize：与 minSize 类似，不过表示抽取出来的文件在压缩前的最大大小，默认为 0，表示不限制最大大小；
- minChunks：表示被引用的最小次数，当模块被不同 entry 引用的次数大于等于这个配置值时，才会被抽离出去，默认为1，也就是任何模块都会被抽离出去；
- maxAsyncRequests：最大的按需(异步)加载次数，默认为 5；
- maxInitialRequests：最大的初始化加载次数，默认为 3；
- automaticNameDelimiter：抽取出来的文件的自动生成名字的分割符，默认为 ~；
- name：抽取出来文件的名字，默认为 true，表示自动生成文件名，默认文件名的大致格式是index～a.js这样的；
- cacheGroups: 配置的核心，缓存组，缓存组的每一个属性都是一个配置规则，属性的值是一个对象，里面放的我们对一个代码拆分规则的描述。

cacheGroups 是我们配置的关键。它可以继承/覆盖上面 splitChunks 中所有的参数值，除此之外还额外提供了三个配置，分别为：test、priority 和 reuseExistingChunk。

1. test: 表示要过滤 modules，默认为所有的 modules，可匹配模块路径或 chunk 名字，当匹配的是 chunk 名字的时候，其里面的所有 modules 都会选中；
2. priority: 表示抽取权重，数字越大表示优先级越高。因为一个 module 可能会满足多个 cacheGroups 的条件，那么抽取到哪个就由权重最高的说了算；
3. reuseExistingChunk: 表示是否使用已有的 chunk，如果为 true 则表示如果当前的 chunk 包含的模块已经被抽取出去了，那么将不会重新生成新的。

SplitChunks 的默认配置如下：

```js
module.exports = {
  //...
  optimization: {
    splitChunks: {
      chunks: 'async', 
      minSize: 30000,
      maxSize: 0,
      minChunks: 1,
      maxAsyncRequests: 5,
      maxInitialRequests: 3,
      automaticNameDelimiter: '~',
      name: true,
      cacheGroups: {
        vendors: {  // 拆分第三方库（通过npm|yarn安装的库）
          test: /[\\/]node_modules[\\/]/,
          priority: -10
        },
        default: {
          minChunks: 2, // 模块被引用2次以上的才抽离
          priority: -20,
          reuseExistingChunk: true
        }
      }
    }
  }
};
```

如果我们还想把项目中的某一些文件单独拎出来打包（比如工程本地开发的组件库），可以继续添加拆分规则。比如我的 src 下有个 locallib.js 文件要单独打包，可以这么配置：

```js
//webpack.config.js
module.exports = {
  //...
  optimization: {
    splitChunks: {
        minSize: 30,  //提取出的chunk的最小大小
        cacheGroups: {
            default: {
                name: 'common',
                chunks: 'initial',
                minChunks: 2,  //模块被引用2次以上的才抽离
                priority: -20
            },
            vendors: {  //拆分第三方库（通过npm|yarn安装的库）
            	test: /[\\/]node_modules[\\/]/,
                name: 'vendor',
                chunks: 'initial',
                priority: -10
            },
            locallib: {  //拆分指定文件
            	test: /(src\/locallib\.js)$/,
                name: 'locallib',
                chunks: 'initial',
                priority: -9
            }
        }
    }
}
};
```

示例中缓存组下又新增了一个拆分规则，通过 test 正则指定我就要单独打包 src/locallib.js 文件，并且把优先级设置为-9，这样当它被多次引用时，不会进入其他拆分规则组，因为另外两个规则的优先级都比它要低。

## 动态导入

除了多入口打包的方式，Code Splitting 更常见的实现方式还是结合 ES Modules 的动态导入特性，从而实现按需加载。按需加载是开发浏览器应用中一个非常常见的需求。一般我们常说的按需加载指的是加载数据或者加载图片，但是我们这里所说的按需加载，指的是在应用运行过程中，需要某个资源模块时，才去加载这个模块。这种方式极大地降低了应用启动时需要加载的资源体积，提高了应用的响应速度，同时也节省了带宽和流量。

Webpack 中支持使用动态导入的方式实现模块的按需加载，而且所有动态导入的模块都会被自动提取到单独的 bundle 中，从而实现分包。

相比于多入口的方式，动态导入更为灵活，因为我们可以通过代码中的逻辑去控制需不需要加载某个模块，或者什么时候加载某个模块。而且我们分包的目的中，很重要的一点就是让模块实现按需加载，从而提高应用的响应速度，举个🌰:

```js
// .src/posts/posts.js
import fetchApi from '../common/fetch'

export default () => {
  const posts = document.createElement('div')
  posts.className = 'posts'
  posts.innerHTML = '<h2>Posts</h2>'

  fetchApi('/posts').then(data => {
    data.forEach(item => {
      const article = document.createElement('article')
      article.className = 'post'
      posts.appendChild(article)
    })
  })

  return posts
}

// .src/album/album.js
import fetchApi from '../common/fetch'

export default () => {
  const album = document.createElement('div')
  album.className = 'album'

  album.innerHTML = '<h2>Albums</h2>'

  fetchApi('/photos?albumId=1').then(data => {
    data.forEach(item => {
      const section = document.createElement('section')
      section.className = 'photo'
      album.appendChild(section)
    })
  })

  return album
}

// .src/index.js
import posts from './posts/posts'
import album from './album/album'

const update = () => {
    const hash = window.location.hash || '#posts';
    const mainElement = document.querySelector('.main');
    mainElement.innerHtml = '';
    if (hash === '#posts') {
        mainElement.appendChild(posts());
    } else if (hash === '#album') {
        mainElement.appendChild(album());
    }
}

window.addEventListener('hashchange', update);
update();
```

上述示例同时导入 posts 组件和 album 组件，然后根据页面的锚点变化决定显示哪个组件，但是在这种情况下就有可能产生资源浪费。试想一下：如果用户只需要访问其中一个页面，那么加载另一个页面对应的组件就是浪费。而如果我们采用动态导入的方式，就不会产生浪费的问题了，因为所有的组件都是惰性加载，只有用到的时候才会加载，具体实现如下：

```js
// .src/index.js
// import posts from './posts/posts'
// import album from './album/album'

const update = () => {
    const hash = window.location.hash || '#posts';
    const mainElement = document.querySelector('.main');
    mainElement.innerHtml = '';
    if (hash === '#posts') {
        // mainElement.appendChild(posts());
        import('./posts/posts').then(({ default: posts })) => {
            ainElement.appendChild(posts());
        }
    } else if (hash === '#album') {
        // mainElement.appendChild(album());
        import('./album/album').then(({ default: album })) => {
            ainElement.appendChild(album());
        }
    }
}

window.addEventListener('hashchange', update);
update();
```

这里我们先移除 import 这种静态引入，然后在需要使用组件的地方通过 import 函数导入指定路径，那这个方法返回的就是一个 Promise。在这个 Promise 的 then 方法中我们能够拿到模块对象，假定我们这里的 posts 和 album 模块是以默认成员导出，所以我们需要解构模块对象中的 default，先拿到导出成员，然后再正常使用这个导出成员。

然后我们在命令行终端重新打包，此时的打包结果如下图所示：

<img src="/assets/webpack-codesplitting/02.png" width="472px" />

此时 dist 目录下就会额外多出三个JS文件，其中两个是动态导入的模块，另外一个文件是动态导入模块中的公共模块，这三个文件就是由动态导入自动分包产生的。

以上就是动态导入在 Webpack 中的使用，整个过程我们无需额外配置任何地方，只需要按照 ES Modules 动态导入的方式去导入模块就可以了，Webpack 内部会自动处理分包和按需加载。如果是在使用 Vue.js 之类的 SPA 开发框架的话，那么项目中路由映射的组件就可以通过这种动态导入的方式实现按需加载，从而实现分包。

### 魔法注释

默认通过动态导入产生的 bundle 文件，它的 name 就是一个序号，这并没有什么不好，因为大多数时候，在生产环境中我们根本不用关心资源文件的名称，但是如果你还需要给这些 bundle 命名的话，就可以使用 Webpack 所特有的魔法注释来实现，具体实现如下：

```js
// 魔法注释
import(/* webpackChunkName: 'posts' */'./posts/posts').then(({ default: posts }) => {
    mainElement.appendChild(posts())
})
```

所谓魔法注释就是在 import 函数的形式参数的位置，添加一个行内注释，这个注释有一定的格式： `webpackChunkName: '***'`，这样就可以给分包的 chunk 起名字了。修改完成之后我们再次运行打包，此时我们生成的 bundle 的 name 就会使用刚刚注释中提供的名称了，具体结果如下：

<img src="/assets/webpack-codesplitting/03.png" width="385px" />

除此之外，魔法注释还有个特殊用途：如果你的 chunkName 相同的话，那么相同的 chunkName 最终就会被打包到一起，例如我们可以把这两个 chunkName 都设置为 components，然后运行打包，那么此时这两个模块都会被打包到一个文件中，具体结果如下：

<img src="/assets/webpack-codesplitting/04.png" width="412px" />

## 参考资料

[webpack优化之玩转代码分割和公共代码提取](https://www.cnblogs.com/champyin/p/11904831.html)

[webpack 4 Code Splitting 的 splitChunks 配置探索](https://imweb.io/topic/5b66dd601402769b60847149)

[webpack 代码分离](https://webpack.docschina.org/guides/code-splitting/)
