---
title: 从vue-cli到Vue CLI（一）
date: 2021-01-28 21:03:53
toc: true
mathjax: false
categories: 
- 前端工程化
tags: 
- cli
---

到底什么才是前端工程化呢？我们知道一个前端项目的开发往往要经历如下步骤：

- 创建项目，主要工程化内容是创建项目结构、特定类型文件
- 编码，主要工程化内容是编译/构建/打包
- 预览/测试，主要工程化内容有Web Server / Mock，Live Reloading / HMR，Source Map等等
- 提交，主要工程化内容有Git Hooks / Husky， Lint-staged等等
- 部署，主要工程化内容有持续集成（CI）、持续部署（CD）

实际上在这个过程中一切以提高效率、降低成本、质量保证为目的的手段都属于前端工程化，前端工程化从早期的脚手架到现在流行的主流工具链的演变主要是因为什么呢？本篇博客希望通过对 vue-cli 和 Vue CLI 架构思想和实现分析其演变的主要原因及设计动机，也是笔者对于前端工程化的一些阶段性理解。

<!-- more -->

## vue-cli

vue-cli 是 Vue 早期官方推荐的脚手架工具，用于快速创建项目，其功能丰富，使用它能够帮助开发者快速的搭建一个完整的项目，开发者只需要在生成的项目结构的基础上进行开发即可，非常高效。并且由于基于模块机制，所以不仅仅可以用于创建 Vue 项目，甚至通过开发自定义模版可以实现定制化项目快速创建，我们接下来来分析其主要实现思路：

### 一、源码分析

#### 1、入口

vue-cli 的源码在[这儿](https://github.com/vuejs/vue-cli/tree/v2)，需要注意项目的 dev 分支是Vue CLI版本，vue-cli 源码需查看 v2 版本，我们首先查看 package.json 文件：

```json
{
  "bin": {
    "vue": "bin/vue",
    "vue-init": "bin/vue-init",
    "vue-list": "bin/vue-list"
  }
}
```

bin 字段用来指定各个内部命令对应的可执行文件的位置。当 package.json 提供了 bin 字段后，即相当于做了一个命令名和本地文件名的映射。当用户安装带有 bin 字段的包时，如果是全局安装，npm 将会使用符号链接把这些文件链接到 /usr/local/node_modules/.bin/，如果是本地安装，会链接到 ./node_modules/.bin/。

因此当我们使用 `vue init <template-name> <project-name>` 时应该直接直接执行 bin/vue-init 这个文件，我们查看文件内容：

```js
#!/usr/bin/env node

const download = require('download-git-repo')
const program = require('commander')
const exists = require('fs').existsSync
const path = require('path')
const ora = require('ora')
const home = require('user-home')
const tildify = require('tildify')
const chalk = require('chalk')
const inquirer = require('inquirer')
const rm = require('rimraf').sync
const logger = require('../lib/logger')
const generate = require('../lib/generate')
const checkVersion = require('../lib/check-version')
const warnings = require('../lib/warnings')
const localPath = require('../lib/local-path')
```

如果想要直接使用 vue init 这样简便的命令，我们还需要在命令映射的文件的第一行写入命令 `#!/usr/bin/env node`，这行命令的作用是告诉系统用 node 解析，当我们输入一个命令的时候，npm 是如何识别并执行对应的文件的呢？具体的原理阮一峰大佬已经在[npm scripts 使用指南](http://www.ruanyifeng.com/blog/2016/10/npm_scripts.html)中介绍过。简单的理解，就是输入命令后，会有在一个新建的 shell 中执行指定的脚本，在执行这个脚本的时候，我们需要来指定这个脚本的解释程序是 node，`#!` 其实是 Shebang，在文件中存在 Shebang 的情况下，类 Unix 操作系统的程序加载器会分析 Shebang 后的内容，将这些内容作为解释器指令，并调用该指令，并将载有 Shebang 的文件路径作为该解释器的参数， `/usr/bin/env` 就是告诉系统可以在 PATH 目录中查找。这种写法主要是为了让你的程序在不同的系统上都能适用。 不管你的 node 是在 `/usr/bin/node` 还是 `/usr/local/bin/node`，`#!/usr/bin/env node` 会自动的在你的用户 PATH 变量中所定义的目录中寻找 node 来执行的，所以配置 `#!/usr/bin/env node`, 就是解决了不同的用户 node 路径不同的问题，可以让系统动态的去查找 node 来执行你的脚本文件。如果没有这行命令我们就需要通过 `node node_modules/.bin/vue-init` 命令执行。

上面引用到各模块主要作用如下：

- download-git-repo 一个用于下载git仓库的项目的模块
- commander 可以将文字输出到终端当中
- fs 是node的文件读写的模块
- path 模块提供了一些工具函数，用于处理文件与目录的路径
- ora 这个模块用于在终端里有显示载入动画
- user-home 获取用户主目录的路径
- tildify 将绝对路径转换为波形路径 比如/Users/sindresorhus/dev → ~/dev
- chalk 主要作用是修改控制台中字符串的样式，比如字体样式(加粗、隐藏等)、字体颜色、背景颜色等
- inquirer 是一个命令行的回答的模块，可以自己设定终端的问题，然后对这些回答给出相应的处理
- rimraf 是一个可以使用 UNIX 命令 rm -rf 的模块

剩下的本地路径的模块其实都是一些工具类，等用到的时候我们再分析其实现。我们再接着往下看：

```js
const isLocalPath = localPath.isLocalPath
const getTemplatePath = localPath.getTemplatePath

// ../lib/local-path
const path = require('path')

module.exports = {
  isLocalPath (templatePath) {
    return /^[./]|(^[a-zA-Z]:)/.test(templatePath)
  },

  getTemplatePath (templatePath) {
    return path.isAbsolute(templatePath)
      ? templatePath
      : path.normalize(path.join(process.cwd(), templatePath))
  }
}
```

紧接着代码调用了 ../lib/local-path 暴露的 isLocalPath 和 getTemplatePath 方法，作用分别是根据模版路径当中是否存在 `./` 是否为本地路径和获取模版路径的方法，如果路径参数是绝对路径则直接返回，如果是相对的，则根据当前路径拼接。我们再接着往下看：

```js
/**
 * Usage.
 */

program
  .usage('<template-name> [project-name]')
  .option('-c, --clone', 'use git clone')
  .option('--offline', 'use cached template')

/**
 * Help.
 */

program.on('--help', () => {
  console.log('  Examples:')
  console.log()
  console.log(chalk.gray('    # create a new project with an official template'))
  console.log('    $ vue init webpack my-project')
  console.log()
  console.log(chalk.gray('    # create a new project straight from a github template'))
  console.log('    $ vue init username/repo my-project')
  console.log()
})

/**
 * Help.
 */

function help () {
  program.parse(process.argv)
  if (program.args.length < 1) return program.help()
}
help()
```

这部分代码声明了vue init用法，如果在终端当中 输入 vue init --help 或者跟在 vue init 后面的参数长度小于1，也会输出下面的描述：

```sh
  Usage: vue-init <template-name> [project-name]

  Options:

    -c, --clone  use git clone
    --offline    use cached template
    -h, --help   output usage information
  Examples:

    # create a new project with an official template
    $ vue init webpack my-project

    # create a new project straight from a github template
    $ vue init username/repo my-project
```

#### 2、配置数据

我们再接着往下看：

```js
/**
 * Settings.
 */

// 模块路径
let template = program.args[0]
const hasSlash = template.indexOf('/') > -1
// 项目名称
const rawName = program.args[1]
const inPlace = !rawName || rawName === '.'
// 如果不存在项目名称或项目名称输入的'.' 则name取的是 当前文件夹的名称
const name = inPlace ? path.relative('../', process.cwd()) : rawName
// 输出路径
const to = path.resolve(rawName || '.')
// 是否需要用到 git clone
const clone = program.clone || false
// tmp为本地模版路径 如果是离线状态 那么模版路径取本地的
const tmp = path.join(home, '.vue-templates', template.replace(/[\/:]/g, '-'))
if (program.offline) {
  console.log(`> Use cached template at ${chalk.yellow(tildify(tmp))}`)
  template = tmp
}
```

可以看到此处代码主要是用于获取模块路径、项目名称和输出路径，那么vue-cli如何根据模版名称下载对应模版并生成对应项目呢？我们再接着往下看：

```js
/**
 * Padding.
 */

console.log()
process.on('exit', () => {
  console.log()
})

if (inPlace || exists(to)) {
  inquirer.prompt([{
    type: 'confirm',
    message: inPlace
      ? 'Generate project in current directory?'
      : 'Target directory exists. Continue?',
    name: 'ok'
  }]).then(answers => {
    if (answers.ok) {
      run()
    }
  }).catch(logger.fatal)
} else {
  run()
}
```

此处根据项目名称和输出路径是否存在进行判断，如果项目名称不存在或输出路径已经存在会在终端提问用户是否确定生成项目，如果用户确定则开始调用 run 方法，需要注意如果项目名称存在并且输出路径不存在则直接调用 run 方法，我们接着直接查看 run 方法：

```js
/**
 * Check, download and generate the project.
 */
function run () {
  // check if template is local
  if (isLocalPath(template)) {
    const templatePath = getTemplatePath(template)
    if (exists(templatePath)) {
      generate(name, templatePath, to, err => {
        if (err) logger.fatal(err)
        console.log()
        logger.success('Generated "%s".', name)
      })
    } else {
      logger.fatal('Local template "%s" not found.', template)
    }
  } else {
    // 检查版本 
    checkVersion(() => {
      // 判断路径中是否含有 /
      if (!hasSlash) {
        // use official templates
        // 拼接路径 'vuejs-tempaltes/'下的都是官方的模版包
        const officialTemplate = 'vuejs-templates/' + template
        // 如果模块路径中含有‘#’则直接下载
        if (template.indexOf('#') !== -1) {
          // 下载并生成模版
          downloadAndGenerate(officialTemplate)
        } else {
          // 如果不存在 -2.0 的字符串 则会输出 模版废弃的相关提示
          if (template.indexOf('-2.0') !== -1) {
            warnings.v2SuffixTemplatesDeprecated(template, inPlace ? '' : name)
            return
          }

          // warnings.v2BranchIsNowDefault(template, inPlace ? '' : name)
          downloadAndGenerate(officialTemplate)
        }
      } else {
        downloadAndGenerate(template)
      }
    })
  }
}
```

我们可以看到代码会首先判断模块路径是否为本地路径，如果为本地路径则调用之前生成的 getTemplatePath 方法生成最终的模块路径，只要判断路径存在就调用 generate 方法生成最终项目，而如果不是本地路径则检查版本，然后调用 downloadAndGenerate 方法根据模版路径下载并生成项目。我们首先查看这儿的 checkVersion 方法：

```js
// ./lib/check-version.js
const request = require('request')
const semver = require('semver')
const chalk = require('chalk')
const packageConfig = require('../package.json')

module.exports = done => {
  // Ensure minimum supported node version is used
  if (!semver.satisfies(process.version, packageConfig.engines.node)) {
    return console.log(chalk.red(
      '  You must upgrade node to >=' + packageConfig.engines.node + '.x to use vue-cli'
    ))
  }

  request({
    url: 'https://registry.npmjs.org/vue-cli',
    timeout: 1000
  }, (err, res, body) => {
    if (!err && res.statusCode === 200) {
      const latestVersion = JSON.parse(body)['dist-tags'].latest
      const localVersion = packageConfig.version
      if (semver.lt(localVersion, latestVersion)) {
        console.log(chalk.yellow('  A newer version of vue-cli is available.'))
        console.log()
        console.log('  latest:    ' + chalk.green(latestVersion))
        console.log('  installed: ' + chalk.red(localVersion))
        console.log()
      }
    }
    done()
  })
}
```

这儿的 semver 模块主要用于模块版本号的管理，这儿会首先判断 Node 的版本号是否满足 vue-cli 的 package.json 文件中的 engines 字段中对于 node 版本号的限制，然后再比较本地 vue-cli 的版本号与 vue-cli 的最新版本号，并在终端提示用户，然后接着调用传入的回调函数，即我们前面看到的 downloadAndGenerate 方法：

```js
/**
 * Download a generate from a template repo.
 *
 * @param {String} template
 */

function downloadAndGenerate (template) {
  // 终端执行加载动画
  const spinner = ora('downloading template')
  spinner.start()
  // Remove if local template exists
  if (exists(tmp)) rm(tmp)
  // 调用download-git-repo模块下载模块，template参数为目标地址 tmp为下载地址 clone参数代表是否需要clone
  download(template, tmp, { clone }, err => {
    // 结束加载动画
    spinner.stop()
    // 如果下载出错打印日志
    if (err) logger.fatal('Failed to download repo ' + template + ': ' + err.message.trim())
    generate(name, tmp, to, err => {
      if (err) logger.fatal(err)
      console.log()
      logger.success('Generated "%s".', name)
    })
  })
}
```

代码中下载模版用的 download 方法是属于 download-git-repo 模块的。最基础的用法如上述示例所示，这里的参数很好理解，第一个参数为仓库地址，第二个为输出地址，第三个是否需要 git clone，带四个为回调参数，需要注意在上面的 run 方法中有提到一个 # 的字符串实际就是这个模块下载分支模块的用法，举个🌰:

```js
download('bitbucket:flipxfx/download-git-repo-fixture#my-branch', 'test/tmp', { clone: true }, function (err) {
  console.log(err ? 'Error' : 'Success')
})
```

我们接下来通过 generate 方法查看 vue-cli 是如何通过模版生成具体的项目的：

```js
// ./lib/generate.js
const chalk = require('chalk')
const Metalsmith = require('metalsmith')
const Handlebars = require('handlebars')
const async = require('async')
const render = require('consolidate').handlebars.render
const path = require('path')
const multimatch = require('multimatch')
const getOptions = require('./options')
const ask = require('./ask')
const filter = require('./filter')
const logger = require('./logger')
```

这儿又引入了一些其他模块：

- Metalsmith是一个静态网站（博客，项目）的生成库
- handlerbars 是一个模版编译器，通过template和json，输出一个html
- async 异步处理模块，有点类似让方法变成一个线程
- consolidate 模版引擎整合库
- multimatch 一个字符串数组匹配的库

随后紧接着注册了2个渲染器，类似于vue中的 v-if v-else的条件渲染：

```js
// ./lib/generate.js
// register handlebars helper
Handlebars.registerHelper('if_eq', function (a, b, opts) {
  return a === b
    ? opts.fn(this)
    : opts.inverse(this)
})

Handlebars.registerHelper('unless_eq', function (a, b, opts) {
  return a === b
    ? opts.inverse(this)
    : opts.fn(this)
})
```

然后就是**最最最重要的 generate 方法**

#### 3、generate

```js
// ./lib/generate.js
module.exports = function generate (name, src, dest, done) {
  const opts = getOptions(name, src)
  ...
}
```

这儿的 getOptions 的具体实现如下：

```js
// ./lib/options.js
const path = require('path')
const metadata = require('read-metadata')
const exists = require('fs').existsSync
const getGitUser = require('./git-user')
const validateName = require('validate-npm-package-name')

/**
 * Read prompts metadata.
 *
 * @param {String} dir
 * @return {Object}
 */

module.exports = function options (name, dir) {
  // 获取 meta.json 或 meta.js 里面的信息，比如 prompts，helpers，filters 等等
  const opts = getMetadata(dir)

  // 为 prompts 属性中的项目名称添加默认值，映射后续生成项目 package.json 中的 name 字段
  setDefault(opts, 'name', name)

  // 利用 validate-npm-package-name 检查你输入的 app-name 是否符合 npm 包名命名规范，当然你也可以在 meta.js 中的 prompts 字段中的 name 下面增加 validate 字段来进行校验，但和 validate-npm-package-name 的规则是 && 的关系。
  setValidateName(opts)

  // 通过 git config --get user.name , git config --get user.email 获取 name 和 email，并拼接生成 author 字段
  const author = getGitUser()

  // 为 prompts 属性中的作者名称添加默认值，映射后续生成项目 package.json 中的 author 字段
  if (author) {
    setDefault(opts, 'author', author)
  }

  return opts
}
```

通过 getOptions 方法我们读取了模版项目目录下的配置文件 meta.json 或 meta.js 文件的内容， 同时将 name auther(当前git用户) 赋值到了 opts 当中，我们继续查看 generate 方法：

```js
// ./lib/generate.js
module.exports = function generate (name, src, dest, done) {
  const opts = getOptions(name, src);
  const metalsmith = Metalsmith(path.join(src, 'template'))
  ...
}
```

Metalsmith 在渲染项目文件流程中角色相当于 gulp.js，可以通过添加一些插件对构建文件进行处理，如重命名、合并等，此处是 定义 Metalsmith 工作目录 ` ~/.vue-templates`。我们接着往下看：

```js
// ./lib/generate.js
module.exports = function generate (name, src, dest, done) {
  ...
  // 定义一些全局变量，这样可以在 layout-files 中使用
  const data = Object.assign(metalsmith.metadata(), {
    destDirName: name,
    inPlace: dest === process.cwd(),
    noEscape: true
  })
  opts.helpers && Object.keys(opts.helpers).map(key => {
    Handlebars.registerHelper(key, opts.helpers[key])
  })
  ...
}
```

Handlebars.registerHelper 用于注册配置文件meta.json 或 meta.js 中的一些 helper（或者说成是一些逻辑方法），在模版中来处理一些数据，比如在之前注册的 `if_eq helper`，他的作用就是判断两个字符串是否相等。然后在 webpack 的模板中就有以下的用法：

<img src="/assets/vue-cli/01.png" width="900" />

就是根据你在构建项目时选择的 test runner （Jest，Karma and Mocha，none configure it yourself） 来生成对应的 npm script。接着往下看：

```js
// ./lib/generate.js
module.exports = function generate (name, src, dest, done) {
  ...
  const helpers = { chalk, logger }

  if (opts.metalsmith && typeof opts.metalsmith.before === 'function') {
    opts.metalsmith.before(metalsmith, opts, helpers)
  }
  ...
}
```

此处 `opts.metalsmith.before` 方法的作用主要是合并一些全局变量，怎么理解呢，我们从 webpack 模板入手。在 webpack 模板的 meta.js 中含有 metalsmith.before:

```js
module.exports = {
  metalsmith: {
    // When running tests for the template, this adds answers for the selected scenario
    before: addTestAnswers
  }
  ...
}
```

addTestAnswers 的具体内容如下：

```js
const scenarios = [
  'full', 
  'full-karma-airbnb', 
  'minimal'
]

const index = scenarios.indexOf(process.env.VUE_TEMPL_TEST)

const isTest = exports.isTest = index !== -1

const scenario = isTest && require(`./${scenarios[index]}.json`)

exports.addTestAnswers = (metalsmith, options, helpers) => {
  Object.assign(
    metalsmith.metadata(),
    { isNotTest: !isTest },
    isTest ? scenario : {}
  )
}
```

metalsmith.before 结果就是将 metalsmith metadata 数据和 isNotTest 合并，如果 isTest 为 ture，还会自动设置 name，description等字段。那么它的作用是什么呢，作用就是为模版添加自动测试脚本，它会将 isNotTest 设置为 false，而通过 inquirer 来提问又会是在 isNotTest 为 true 的情况下才会发生，因此设置了 VUE_TEMPL_TEST 的值会省略 inquirer 提问过程，并且会根据你设置的值来生成对应的模板，有以下三种值可以设置：

- minimal：这种不会设置 router，eslint 和 tests
- full： 会带有 router，eslint (standard) 和 tests (jest & e2e)
- full-airbnb-karma：带有 router eslint（airbnb） 和 tests（karma）

例如我们可以通过如下指令直接生成项目：

```sh
VUE_TEMPL_TEST=full vue init webpack demo
```

在这种情况下，会自动跳过 inquirer 的问题，直接生成项目：

<img src="/assets/vue-cli/02.png" width="500" />

项目目录结构如下图所示：

<img src="/assets/vue-cli/03.png" width="900" />

我们接着查看 generate 方法：

```js
// ./lib/generate.js
module.exports = function generate (name, src, dest, done) {
  ...
  metalsmith.use(askQuestions(opts.prompts))
    .use(filterFiles(opts.filters))
    .use(renderTemplateFiles(opts.skipInterpolation))
  ...
}
```

metalsmith.use 是 metalsmith 使用插件的写法，前面说过 metalsmith 最大的特点就是所有的逻辑都是由插件处理，此处一共有使用了三个 metalsmith 插件，分别为：askQuestions filterFiles renderTemplateFiles:

1、askQuestions 方法主要是通过 inquirer.prompt 来实现命令行交互，并将交互的值通过 metalsmith.metadata() 存到全局，然后在渲染模板的时候直接获取这些值。

2、 filterFiles 方法的主要作用是根据用户在命令行界面的选择来对模版中文件结构进行处理，举个🌰：

webpack 模版中 meta.js 中 filter 字段如下：

```js
filters: {
  '.eslintrc.js': 'lint',
  '.eslintignore': 'lint',
  'config/test.env.js': 'unit || e2e',
  'build/webpack.test.conf.js': "unit && runner === 'karma'",
  'test/unit/**/*': 'unit',
  'test/unit/index.js': "unit && runner === 'karma'",
  'test/unit/jest.conf.js': "unit && runner === 'jest'",
  'test/unit/karma.conf.js': "unit && runner === 'karma'",
  'test/unit/specs/index.js': "unit && runner === 'karma'",
  'test/unit/setup.js': "unit && runner === 'jest'",
  'test/e2e/**/*': 'e2e',
  'src/router/**/*': 'router',
},
```

以 .eslintrc.js 为例，在模板中默认是有 .eslintrc.js 文件的。利用 vue-cli 初始化一个项目的时候，会询问你 Use ESLint to lint your code? ，然后 inquirer.prompt 通过回调将你回答的值存在 metalsmith.metadata() 的 lint 字段中，在调用 filterFiles 方法的时候就会判断在 metalsmith.metadata() 下 lint 的值是否为 true，如果为 false 的就会删除 .eslintrc.js。

3、renderTemplateFiles

renderTemplateFiles 方法的源码如下图所示：

```js
function renderTemplateFiles (skipInterpolation) {
  // 在 meta.js 的 skipInterpolation 下面添加跳过插值的文件，这样在渲染的时候就不会使用 consolidate.handlebars.render 去渲染页面
  skipInterpolation = typeof skipInterpolation === 'string'
    ? [skipInterpolation]
    : skipInterpolation
  return (files, metalsmith, done) => {
    const keys = Object.keys(files)
    const metalsmithMetadata = metalsmith.metadata()
    async.each(keys, (file, next) => {
      // skipping files with skipInterpolation option
      if (skipInterpolation && multimatch([file], skipInterpolation, { dot: true }).length) {
        return next()
      }
      const str = files[file].contents.toString()
      // do not attempt to render files that do not have mustaches
      // 如果在该文件中没有遇到 {{}} 就跳过该文件
      if (!/{{([^{}]+)}}/g.test(str)) {
        return next()
      }
      render(str, metalsmithMetadata, (err, res) => {
        if (err) {
          err.message = `[${file}] ${err.message}`
          return next(err)
        }
        files[file].contents = new Buffer(res)
        next()
      })
    }, done)
  }
}
```

renderTemplateFiles 的主要功能就是利用 consolidate.handlebars.render 将 ~/.vue-templates 下面的 handlebars 模板文件渲染成正式的文件。

我们再接着查看 generate 方法：

```js
// ./lib/generate.js
module.exports = function generate (name, src, dest, done) {
  ...
  // 进一步给提供了模版提供了配置数据处理及包装的能力
  if (typeof opts.metalsmith === 'function') {
    opts.metalsmith(metalsmith, opts, helpers)
  } else if (opts.metalsmith && typeof opts.metalsmith.after === 'function') {
    opts.metalsmith.after(metalsmith, opts, helpers)
  }

  // clean方法是设置在写入之前是否删除原先目标目录 默认为true
  // source方法是设置原路径
  // destination方法就是设置输出的目录
  // build方法执行构建
  metalsmith.clean(false)
    .source('.') // start from template root instead of `./src` which is Metalsmith's default for `source`
    .destination(dest)
    .build((err, files) => {
      done(err)
      if (typeof opts.complete === 'function') {
        const helpers = { chalk, logger, files }
        opts.complete(data, helpers)
      } else {
        logMessage(opts.completeMessage, data)
      }
    })

  return data
  ...
}
```

此处的核心是 metalsmith.build 使用刚才分析的 askQuestions 、filterFiles 和 renderTemplateFiles 三个插件将项目的初始化文件生成出来并输出到目标目录，完成后输出相关的信息。

至此 vue-cli 创建项目的实现流程我们就全部分析完成了，generate 方法通过模版项目下的 meta.json 或 meta.js 配置文件的解析生成对应的项目配置文件及目录，我们可以[点这儿](https://github.com/vuejs-templates/webpack)查看模版 vuejs-templates/webpack 模块的配置文件。

<img src="/assets/emoji/02.png">

### 二、总结

通过上面的分析我们得到 vue-cli 的整体打包流程如下所示：  

<img src="/assets/vue-cli/04.svg" width="1000" />

### 三、vue-cli的不足

从上面对于 vue-cli 的分析我们也可以看出其只是一个单纯的脚手架工具，用于快速创建项目的工具，由于基于模块机制，所以不仅仅可以用于创建 Vue 项目，甚至通过开发自定义模版可以实现定制化项目快速创建，那么为什么其后续慢慢被遗弃呢？

这主要是由于脚手架本身的局限性：用完即丢。创建完项目就不管了，后续流程不参与。后续开发过程中，各项工程化需求由其它工具分别完成，因此用户大量的时间耗费在各种工具的学习、配置、调试和踩坑上 应对工具升级、迁移，维护成本极高，因此更为理想的方案应该是项目整个流程各个工程化需求都参与的工具链。这也导致了 vue-cli 的后续版本 Vue CLI 的诞生。


## 参考资料

[Vue-cli原理分析](https://juejin.cn/post/6844903646556078093#heading-3)

