---
title: 从vue-cli到Vue CLI
date: 2021-01-28 21:03:53
toc: true
mathjax: false
categories: 
- 前端
tags: 
- 前端工程化
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

### 一、源码分析

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

如果想要直接使用 vue init 这样简便的命令，我们还需要在命令映射的文件的第一行写入命令 `#!/usr/bin/env node`，这行命令的作用是告诉系统用 node 解析，如果没有这行命令就需要使用 `node node_modules/.bin/vue-init` 命令执行。

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

然后就是最重要的 generate 方法：

```js
module.exports = function generate (name, src, dest, done) {
  // 读取了模版项目目录下的配置文件 meta.json 或 meta.js 文件的内容， 同时将 name auther(当前git用户) 赋值到了 opts 当中
  const opts = getOptions(name, src)
  // 拼接了目录 src/{template} 要在这个目录下生产静态文件
  const metalsmith = Metalsmith(path.join(src, 'template'))
  // 将 metalsmitch 中的 meta 与 destDirName、inPlace、noEscape三个属性合并起来 形成 data
  const data = Object.assign(metalsmith.metadata(), {
    destDirName: name,
    inPlace: dest === process.cwd(),
    noEscape: true
  })
  // 遍历 meta.js 元数据中的 helpers 对象，注册渲染模版数据
  // 分别指定了 if_or 和 template_version 内容
  opts.helpers && Object.keys(opts.helpers).map(key => {
    Handlebars.registerHelper(key, opts.helpers[key])
  })

  const helpers = { chalk, logger }

  // 将metalsmith metadata 数据 和 { isNotTest, isTest 合并 }
  if (opts.metalsmith && typeof opts.metalsmith.before === 'function') {
    opts.metalsmith.before(metalsmith, opts, helpers)
  }

  // askQuestions是会在终端里询问一些问题
  // 名称 描述 作者 是要什么构建 在meta.js 的opts.prompts当中
  // filterFiles 是用来过滤文件
  // renderTemplateFiles 是一个渲染插件
  metalsmith.use(askQuestions(opts.prompts))
    .use(filterFiles(opts.filters))
    .use(renderTemplateFiles(opts.skipInterpolation))

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
        // 当生成完毕之后执行 meta.js当中的 opts.complete方法
        const helpers = { chalk, logger, files }
        opts.complete(data, helpers)
      } else {
        logMessage(opts.completeMessage, data)
      }
    })

  return data
}
```

至此 vue-cli 创建项目的实现流程我们就全部分析完成了，generate 方法通过模版项目下的 meta.json 或 meta.js 配置文件的解析生成对应的项目配置文件及目录，我们可以[点这儿](https://github.com/vuejs-templates/webpack)查看模版 vuejs-templates/webpack 模块的配置文件。

### 二、vue-cli的不足

从上面对于 vue-cli 的分析我们也可以看出其只是一个单纯的脚手架工具，用于快速创建项目的工具，由于基于模块机制，所以不仅仅可以用于创建 Vue 项目，甚至通过开发自定义模版可以实现定制化项目快速创建，那么为什么其后续慢慢被遗弃呢？
这主要是由于脚手架本身的局限性：用完即丢。创建完项目就不管了，后续流程不参与。后续开发过程中，各项工程化需求由其它工具分别完成，因此用户大量的时间耗费在各种工具的学习、配置、调试和踩坑上 应对工具升级、迁移，维护成本极高，因此更为理想的方案应该是项目整个流程各个工程化需求都参与的工具链。这也导致了 vue-cli 的后续版本 Vue CLI 的诞生。

## Vue CLI

Vue CLI 是目前 Vue 官方推荐的 Vue 项目快速开发的完整系统，它基于 Webpack 实现，提供了终端命令行工具、零配置脚手架、插件体系、图形化管理界面等诸多功能，近乎提供了前端项目工程化的所有步骤的完整工具链，也是当前 Vue 项目构建的主流工具。

### 一、源码分析

> 主要分析项目初始化部分，源码为 4.5.9 版本

我们首先查找入口文件：

```json
{
  "bin": {
    "vue": "bin/vue.js"
  }
}
```

bin/vue.js 里的代码不少，主要作用是注册了 create / add / ui  等命令，此处只分析 create 部分：

```js
// 检查 node 版本
checkNodeVersion(requiredVersion, '@vue/cli');

// 挂载 create 命令
program.command('create <app-name>').action((name, cmd) => {
  // 获取额外参数
  const options = cleanArgs(cmd);
  // 执行 create 方法
  require('../lib/create')(name, options);
});
```



## 参考资料

[Vue-cli原理分析](https://juejin.cn/post/6844903646556078093#heading-3)

