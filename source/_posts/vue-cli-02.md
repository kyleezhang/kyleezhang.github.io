---
title: 从vue-cli到Vue CLI（二）
date: 2021-05-15 12:00:35
toc: true
mathjax: false
categories: 
- 前端
tags: 
- 前端工程化
---

## Vue CLI

Vue CLI 是目前 Vue 官方推荐的 Vue 项目快速开发的完整系统，它基于 Webpack 实现，提供了终端命令行工具、零配置脚手架、插件体系、图形化管理界面等诸多功能，近乎提供了前端项目工程化的所有步骤的完整工具链，也是当前 Vue 项目构建的主流工具。

### 一、整体架构

vue-cli 为了尽可能覆盖项目工程化需求所以模版项目往往引入了大量第三方库，但实际开发过程中开发者可能并不需要这些功能模块，虽然可以通过在模版项目的 meta.js 或 meta.json 文件配置 prompts 在命令行交互然后在 filterFiles 函数中对生成项目的文件目录结构进行筛选，但是依然可配置性不强，会存在较多冗余依赖或功能，因此为了给开发者提供更灵活的配置能力 Vue CLI 实现了一种极为巧妙的架构设计：

<!-- more -->

<img src="/assets/vue-cli/05.png" width="400" />

如上图所示，Vue CLI 的最新架构主要由如下模块构成：

1、@vue/cli: 首先我们会在全局安装一个@vue/cli，其为我们提供了一系列 vue 命令，例如我们最常用 vue create、vue add 指令等等。@vue/cli 本质上依然是脚手架工具，主要通过 CLI 交互自动化创建项目基础结构，具体步骤为：

- 准备命令行交互的问题
- 根据用户的回答决定是否使用某个插件（内置）
- 调用 npm/yarn 自动在项目本地安装这些插件
- 调用每个插件内部的 generator 方法
- generator 内部创建所有文件

2、@vue/cli-service: 如果说 @vue/cli 的主要作用是项目的生成，那么 @vue/cli-service 才是开发者日常开发的真正核心，为我们提供了 Vue.js 开发环境的 CLI 服务（本地测试、生产环境构建等等），因此它提供的主要功能点如下所示：

- 提供了一个适用于大多数 Vue.js 项目的 Webpack 配置，并且内部只做最通用的配置。
- 把 Webpack 与 Webpack Dev Server 封装进内部，提供 server & build 命令。

3、@vue/cli-plugin-xxx: 个人觉得新版 Vue CLI 设计最出彩的地方就在于插件机制，项目本身创建最开始的时候只是根据用户在命令行的交互问题生成 package.json，然后根据所拥有的模块动态完成 Webpack 的配置并生成项目结构，因此 Vue CLI 的插件模块主要完成如下功能：

- 提供了个性化的 Webpack 配置，让构建过程支持不同的技术选型
- 提供了 Generator 用于生成项目中所需的文件
- 提供了额外的 Command 用于实现扩展功能，例如 cli-plugin-eslint 插件中提供了 lint 子命令

下面我们开始详细分析每一个模块来查看其功能的具体实现：

<img src="/assets/emoji/03.png" width="260" />

### 二、@vue/cli

@vue/cli 主要为我们提供了一系列 vue 命令，具体如下图所示：

<img src="/assets/vue-cli/06.png" width="800" />

这些指令都是在 `@vue/cli/bin/vue.js` 文件中通过 command 库指定的，后续分析我们会分别进行展示，此处比较有意思的是 @vue/cli 提供了 suggestCommands 函数对用户在命令行的错误输入分析并给出猜测指令，具体实现如下：

```js
// @vue/cli/bin/vue.js
...
// output help information on unknown commands
// 如果不是前文已挂载的命令开始执行如下函数
program.on('command:*', ([cmd]) => {
  program.outputHelp()
  console.log(`  ` + chalk.red(`Unknown command ${chalk.yellow(cmd)}.`))
  console.log()
  suggestCommands(cmd)
  process.exitCode = 1
})
...
// 猜测用户意图
function suggestCommands (unknownCommand) {
  const availableCommands = program.commands.map(cmd => cmd._name)

  let suggestion

  availableCommands.forEach(cmd => {
    const isBestMatch = leven(cmd, unknownCommand) < leven(suggestion || '', unknownCommand)
    if (leven(cmd, unknownCommand) < 3 && isBestMatch) {
      suggestion = cmd
    }
  })

  if (suggestion) {
    console.log(`  ` + chalk.red(`Did you mean ${chalk.yellow(suggestion)}?`))
  }
}
```

代码中使用了 leven 了这个包，这是用于计算字符串编辑距离算法的 JS 实现，Vue CLI 这里使用了这个包，来分别计算输入的命令和当前已挂载的所有命令的编辑举例，从而猜测用户实际想输入的命令是哪个。一个简单功能的实现，但极大程度地提升了用户体验👍。


首先我们开始分析 create 命令：

#### 1、create 命令

我们首先查看 create 命令的入口：

```js
// packages/@vue/cli/bin/vue.js
program
  .command('create <app-name>')
  .description('create a new project powered by vue-cli-service')
  .option('-p, --preset <presetName>', 'Skip prompts and use saved or remote preset')
  .option('-d, --default', 'Skip prompts and use default preset')
  .option('-i, --inlinePreset <json>', 'Skip prompts and use inline JSON string as preset')
  .option('-m, --packageManager <command>', 'Use specified npm client when installing dependencies')
  .option('-r, --registry <url>', 'Use specified npm registry when installing dependencies (only for npm)')
  .option('-g, --git [message]', 'Force git initialization with initial commit message')
  .option('-n, --no-git', 'Skip git initialization')
  .option('-f, --force', 'Overwrite target directory if it exists')
  .option('-c, --clone', 'Use git clone when fetching remote preset')
  .option('-x, --proxy', 'Use specified proxy when creating project')
  .option('-b, --bare', 'Scaffold project without beginner instructions')
  .action((name, options) => {
    if (minimist(process.argv.slice(3))._.length > 1) {
      console.log(chalk.yellow('\n Info: You provided more than one argument. The first one will be used as the app\'s name, the rest are ignored.'))
    }
    // --git makes commander to default git to true
    if (process.argv.includes('-g') || process.argv.includes('--git')) {
      options.forceGit = true
    }
    require('../lib/create')(name, options)
  })
```

上述参数的具体作用如下所示：

```sh
-p, --preset <presetName>       忽略提示符并使用已保存的或远程的预设选项
-d, --default                   忽略提示符并使用默认预设选项
-i, --inlinePreset <json>       忽略提示符并使用内联的 JSON 字符串预设选项
-m, --packageManager <command>  在安装依赖时使用指定的 npm 客户端
-r, --registry <url>            在安装依赖时使用指定的 npm registry
-g, --git [message]             强制 / 跳过 git 初始化，并可选的指定初始化提交信息
-n, --no-git                    跳过 git 初始化
-f, --force                     覆写目标目录可能存在的配置
-c, --clone                     使用 git clone 获取远程预设选项
-x, --proxy                     使用指定的代理创建项目
-b, --bare                      创建项目时省略默认组件中的新手指导信息
-h, --help                      输出使用帮助信息
```

此处代码主要是调用了 `../lib/create.js` 文件暴露出的 create 方法：

```js
async function create (projectName, options) {
   if (options.proxy) {
    process.env.HTTP_PROXY = options.proxy // 根据用户是否配置 proxy 项创建环境变量 HTTP_PROXY
  }

  const cwd = options.cwd || process.cwd() // 当前目录
  const inCurrent = projectName === '.' // 是否在当前目录
  const name = inCurrent ? path.relative('../', cwd) : projectName // 项目名称
  const targetDir = path.resolve(cwd, projectName || '.') // 生成项目的目录

  const result = validateProjectName(name) // 调用 validate-npm-package-name 库校验项目名称
  if (!result.validForNewPackages) {
    console.error(chalk.red(`Invalid project name: "${name}"`))
    result.errors && result.errors.forEach(err => {
      console.error(chalk.red.dim('Error: ' + err))
    })
    result.warnings && result.warnings.forEach(warn => {
      console.error(chalk.red.dim('Warning: ' + warn))
    })
  }
  // 此处代码主要用于项目生成路径的判断，主要是当存在相同项目目录的时候调用 inquirer.prompt 来询问是否要 Overwrite || Merge || Cancel，当带有 -f || --force 的时候会跳过这些交互，即 options force = true
  if (fs.existsSync(targetDir) && !options.merge) {
    if (options.force) {
      await fs.remove(targetDir)
    } else {
      await clearConsole()
      if (inCurrent) {
        const { ok } = await inquirer.prompt([
          {
            name: 'ok',
            type: 'confirm',
            message: `Generate project in current directory?`
          }
        ])
        if (!ok) {
          return
        }
      } else {
        const { action } = await inquirer.prompt([
          {
            name: 'action',
            type: 'list',
            message: `Target directory ${chalk.cyan(targetDir)} already exists. Pick an action:`,
            choices: [
              { name: 'Overwrite', value: 'overwrite' },
              { name: 'Merge', value: 'merge' },
              { name: 'Cancel', value: false }
            ]
          }
        ])
        if (!action) {
          return
        } else if (action === 'overwrite') {
          console.log(`\nRemoving ${chalk.cyan(targetDir)}...`)
          await fs.remove(targetDir)
        }
      }
    }
  }

  const creator = new Creator(name, targetDir, getPromptModules())
  await creator.create(options)
}

module.exports = (...args) => {
  return create(...args).catch(err => { // 捕获内部所有抛出的错误，将当前的 spinner  状态停止，退出进程。
    stopSpinner(false) // do not persist
    error(err)
    if (!process.env.VUE_CLI_TEST) {
      process.exit(1)
    }
  })
}
```

经过认真的代码分析我们成功发现真正实现的核心都在 `../lib/Creator` 的 Creator 类中，，，但是这儿 getPromptModules 方法的作用是什么呢？

```js
exports.getPromptModules = () => {
  return [
    'vueVersion',
    'babel',
    'typescript',
    'pwa',
    'router',
    'vuex',
    'cssPreprocessors',
    'linter',
    'unit',
    'e2e'
  ].map(file => require(`../promptModules/${file}`))
}
```

getPromptModules 函数主要用于 `../promptModules/` 文件夹下不同配置文件内容的读取，我们以 `../promptModules/unit.js` 为例：

```js
//../promptModules/unit.js
module.exports = cli => {
  cli.injectFeature({
    name: 'Unit Testing',
    value: 'unit',
    short: 'Unit',
    description: 'Add a Unit Testing solution like Jest or Mocha',
    link: 'https://cli.vuejs.org/config/#unit-testing',
    plugins: ['unit-jest', 'unit-mocha']
  })

  cli.injectPrompt({
    name: 'unit',
    when: answers => answers.features.includes('unit'),
    type: 'list',
    message: 'Pick a unit testing solution:',
    choices: [
      {
        name: 'Jest',
        value: 'jest',
        short: 'Jest'
      },
      {
        name: 'Mocha + Chai (requires webpack 4)',
        value: 'mocha',
        short: 'Mocha'
      }
    ]
  })

  cli.onPromptComplete((answers, options) => {
    if (answers.unit === 'mocha') {
      options.plugins['@vue/cli-plugin-unit-mocha'] = {}
    } else if (answers.unit === 'jest') {
      options.plugins['@vue/cli-plugin-unit-jest'] = {}
    }
  })
}
```

`cli.injectFeature` 是注入 featurePrompt，即初始化项目时选择的勾选项如 babel，typescript，pwa 等等，如下图：

<img src="/assets/vue-cli/07.png" width="500" />

`cli.injectPrompt` 是根据选择的 featurePrompt 然后注入对应的 prompt，当选择了 unit，接下来会有以下的 prompt，选择 Mocha + Chai 还是 Jest：

<img src="/assets/vue-cli/08.png" width="500" />

`cli.onPromptComplete` 就是一个回调，会根据选择来添加对应的插件， 当选择了 mocha ，那么就会添加 @vue/cli-plugin-unit-mocha 插件。

接下来我们开始正式查看 Creator 类，打开一看，哦豁，554 行代码，12 个引入模块，告辞！

考虑到大家时间比较紧张（wo bi jiao lan）这儿梳理一下主要流程：

```js
module.exports = class Creator extends EventEmitter {
  constructor (name, context, promptModules) {
    super()

    this.name = name
    this.context = process.env.VUE_CLI_CONTEXT = context
    // 获取了 preset 和 feature 的交互选择列表，在 vue create 命令初始化项目的时候提供选择
    const { presetPrompt, featurePrompt } = this.resolveIntroPrompts()

    this.presetPrompt = presetPrompt // preset 列表
    this.featurePrompt = featurePrompt // pwa、babel、e2e等等
    // 存放项目配置的文件（package.json || congfig.js） 以及是否将 presetPrompts 存放起来
    this.outroPrompts = this.resolveOutroPrompts()
    this.outroPrompts = [] // 对应 feature 的 Prompts
    // 不同阶段回调函数列表
    this.promptCompleteCbs = []
    this.afterInvokeCbs = []
    this.afterAnyInvokeCbs = []

    this.run = this.run.bind(this)

    const promptAPI = new PromptModuleAPI(this)
    promptModules.forEach(m => m(promptAPI))
  }
  ...
}
```

这儿我们首先注意到构造函数中分别为实例对象添加了 presetPrompt、featurePrompt、outroPrompts、outroPrompts 等各种各样的 “prompt” 属性，那么它们之间有什么区别，又分别代表了什么呢？

- presetPrompt： 预设选项 prompt，当上次以 Manually 模式进行了预设选项，并且保存到了 ~/.vuerc 中，那么在初始化项目时就会列出已经保存的 preset，并提供选择。
- featurePrompt：项目的一些 feature，就是选择 babel，typescript，pwa，router，vuex，cssPreprocessors，linter，unit，e2e。
- injectedPrompts：当选择了 feature 后，就会为对应的 feature 注入 prompts，比如你选择了 unit，那么就会让你选择模式： Mocha + Chai 还是 Jest
- outroPrompts： 其他的 prompt，包含：
  - 将 Babel, PostCSS, ESLint 等等的配置文件存放在 package.json 中还是存放在 config 文件中；
  - 是否需要将这次设置的 preset 保存到本地，如果需要则会进一步让你输入名称进行保存；
  - 安装依赖是选择 npm 还是 yarn。

再搞清了 “prompt” 之间的区别后我们接着往下看 PromptModuleAPI，其源码如下：

```js
module.exports = class PromptModuleAPI {
  constructor (creator) {
    this.creator = creator
  }

  injectFeature (feature) {
    this.creator.featurePrompt.choices.push(feature)
  }

  injectPrompt (prompt) {
    this.creator.injectedPrompts.push(prompt)
  }

  injectOptionForPrompt (name, option) {
    this.creator.injectedPrompts.find(f => {
      return f.name === name
    }).choices.push(option)
  }

  onPromptComplete (cb) {
    this.creator.promptCompleteCbs.push(cb)
  }
}
```

PromptModuleAPI 实例会调用它的实例方法，然后将 injectFeature， injectPrompt， injectOptionForPrompt， onPromptComplete保存到 Creator实例对应的变量中。最后遍历前面 getPromptModules 获取的 promptModules，传入实例 promptAPI，初始化 Creator 实例中 featurePrompt, injectedPrompts, promptCompleteCbs 变量。

在 `../lib/create.js` 文件中的 create 方法中我们生成 Creator 实例后调用了其 create 方法：

```js
await creator.create(options)
```

因此我们接着查看 create 方法的具体实现：

```js
async create (cliOptions = {}, preset = null) {
  const isTestOrDebug = process.env.VUE_CLI_TEST || process.env.VUE_CLI_DEBUG
  const { run, name, context, afterInvokeCbs, afterAnyInvokeCbs } = this

  if (!preset) {
    if (cliOptions.preset) {
      // vue create foo --preset bar
      preset = await this.resolvePreset(cliOptions.preset, cliOptions.clone)
    } else if (cliOptions.default) {
      // vue create foo --default
      preset = defaults.presets.default
    } else if (cliOptions.inlinePreset) {
      // vue create foo --inlinePreset {...}
      try {
        preset = JSON.parse(cliOptions.inlinePreset)
      } catch (e) {
        error(`CLI inline preset is not valid JSON: ${cliOptions.inlinePreset}`)
        exit(1)
      }
    } else {
      preset = await this.promptAndResolvePreset()
    }
  }

  // clone before mutating
  preset = cloneDeep(preset)
  // inject core service
  preset.plugins['@vue/cli-service'] = Object.assign({ // 注入核心 @vue/cli-service
    projectName: name
  }, preset, {
    bare: cliOptions.bare
  })
  ...
}
```

先判断 vue create 命令是否带有 -p 选项，如果有的话会调用 resolvePreset 去解析 preset。resolvePreset 函数会先获取 `～/.vuerc` 中保存的 preset， 然后进行遍历，如果里面包含了 -p 中的 `<presetName>`，则返回 `～/.vuerc` 中的 preset。如果没有则判断是否是采用内联的 JSON 字符串预设选项，如果是就会解析 `.json` 文件，并返回 preset，还有一种情况就是从远程获取 preset（利用 download-git-repo 下载远程的 preset.json）并返回。

如果 vue create 命令没有带有 -p 选项就会调用 promptAndResolvePreset 函数利用 inquirer.prompt 以命令后交互的形式来获取 preset，下面看下 promptAndResolvePreset 函数的源码：

```js
async promptAndResolvePreset (answers = null) {
  // prompt
  if (!answers) {
    await clearConsole(true)
    answers = await inquirer.prompt(this.resolveFinalPrompts())
  }
  debug('vue-cli:answers')(answers)

  if (answers.packageManager) {
    saveOptions({
      packageManager: answers.packageManager
    })
  }

  let preset
  if (answers.preset && answers.preset !== '__manual__') {
    preset = await this.resolvePreset(answers.preset)
  } else {
    // manual
    preset = {
      useConfigFiles: answers.useConfigFiles === 'files',
      plugins: {}
    }
    answers.features = answers.features || []
    // run cb registered by prompt modules to finalize the preset
    this.promptCompleteCbs.forEach(cb => cb(answers, preset))
  }

  // validate
  validatePreset(preset)

  // save preset
  if (answers.save && answers.saveName && savePreset(answers.saveName, preset)) {
    log()
    log(`🎉  Preset ${chalk.yellow(answers.saveName)} saved in ${chalk.yellow(rcPath)}`)
  }

  debug('vue-cli:preset')(preset)
  return preset
}
```

代码首先利用 this.resolveFinalPrompts() 获取了最后的 prompts 并通过 inquirer.prompt 在命令行与用户交互，这儿的 resolveFinalPrompts 的实现如下所示：

```js
resolveFinalPrompts () {
  // patch generator-injected prompts to only show in manual mode
  // 将所有的 Prompt 合并，包含 preset，feature，injected，outro，只有当选择了手动模式的时候才会显示 injectedPrompts
  this.injectedPrompts.forEach(prompt => {
    const originalWhen = prompt.when || (() => true)
    prompt.when = answers => {
      return isManualMode(answers) && originalWhen(answers)
    }
  })

  const prompts = [
    this.presetPrompt,
    this.featurePrompt,
    ...this.injectedPrompts,
    ...this.outroPrompts
  ]
  debug('vue-cli:prompts')(prompts)
  return prompts
}
```

inquirer.prompt 执行完成后会返回 answers，如果选择了本地保存的 preset 或者 default，则调用 resolvePreset 进行解析 preset，否则遍历 promptCompleteCbs 执行 injectFeature 和 injectPrompt 的回调，将对应的插件赋值到 options.plugins 中，以 unit 为例：

```js
cli.onPromptComplete((answers, options) => {
  if (answers.unit === 'mocha') {
    options.plugins['@vue/cli-plugin-unit-mocha'] = {}
  } else if (answers.unit === 'jest') {
    options.plugins['@vue/cli-plugin-unit-jest'] = {}
  }
})
```

如果 feature 选择了 unit，并且 unit 模式选择的是 Mocha + Chai，则添加 @vue/cli-plugin-unit-mocha 插件，如果选择的是 Jest 则添加 @vue/cli-plugin-unit-jest 插件。

在获取到 preset 之后，还会向 preset 的插件里面注入核心插件 `@vue/cli-service`， 它是调用 `vue-cli-service <command> [...args]` 时创建的类。 负责管理内部的 webpack 配置、暴露服务和构建项目的命令等。

至此我们已经完成了 vue create 命令执行过程中第一个要点：**获取用户配置信息**，并且根据配置信息获取了所需的插件，接下来便是另一个核心**依赖的安装**，我们来接着看 create 函数：

```js
async create (cliOptions = {}, preset = null) {
  ...
  const packageManager = (
    cliOptions.packageManager ||
    loadOptions().packageManager ||
    (hasYarn() ? 'yarn' : null) ||
    (hasPnpm3OrLater() ? 'pnpm' : 'npm')
  )

  await clearConsole()
  const pm = new PackageManager({ context, forcePackageManager: packageManager })

  log(`✨  Creating project in ${chalk.yellow(context)}.`)
  this.emit('creation', { event: 'creating' })

  // get latest CLI plugin version
  const { latestMinor } = await getVersions()
  // generate package.json with plugin dependencies
  const pkg = {
    name,
    version: '0.1.0',
    private: true,
    devDependencies: {},
    ...resolvePkg(context)
  }
  const deps = Object.keys(preset.plugins)
  deps.forEach(dep => {
    if (preset.plugins[dep]._isPreset) {
      return
    }

    let { version } = preset.plugins[dep]

    if (!version) {
      if (isOfficialPlugin(dep) || dep === '@vue/cli-service' || dep === '@vue/babel-preset-env') {
        version = isTestOrDebug ? `latest` : `~${latestMinor}`
      } else {
        version = 'latest'
      }
    }

    pkg.devDependencies[dep] = version
  })

  // write package.json
  await writeFileTree(context, {
    'package.json': JSON.stringify(pkg, null, 2)
  })
  ...
}
```

此处代码主要有两处作用：**获取最新 CLI （包含插件）的版本** 和 **生成 package.json**，这里 getVersions 方法用于判断 @vue/cli 是否需要更新以及初始化项目中相关插件的版本，需要注意的是，获取 CLI 的版本并不是直接获取， 而是通过 `vue-cli-version-marker` npm 包获取的 CLI 版本，为什么会这样做，主要原因有两点：

- vue-cli 从 3.0（@vue/cli） 开始就放在了 @vue 下面，即是一个 scoped package, 而 **scoped package 又不支持通过 npm registry 来获取 latest 版本信息**。比如 vue-cli-version-marker/latest 可以正常访问，而 @vue/cli/latest 则不可以。
- 获取 scoped packages 的数据比获取 unscoped package 通常要慢 300ms。

正是由于上述两个原因，因此通过 unscoped package `vue-cli-version-marker` 来获取 CLI 版本，`vue-cli-version-marker` 的内容比较简单，就是一个 package.json，通过获取里面 devDependencies 的版本信息，从而获取 @vue/cli 以及一些插件的版本号。

获取了插件版本之后遍历 preset 中所有 plugin 为其初始化版本号。并调用 writeFileTree 生成 package.json 。

在生成 package.json 文件之后便开始依赖的正式安装，在这之前会先进行判断是否需要 git 初始化：

```js
async create (cliOptions = {}, preset = null) {
  ...
  // intilaize git repository before installing deps
  // so that vue-cli-service can setup git hooks.
  const shouldInitGit = this.shouldInitGit(cliOptions)
  if (shouldInitGit) {
    log(`🗃  Initializing git repository...`)
    this.emit('creation', { event: 'git-init' })
    await run('git init')
  }

  // install plugins
  log(`⚙\u{fe0f}  Installing CLI plugins. This might take a while...`)
  log()
  this.emit('creation', { event: 'plugins-install' })

  if (isTestOrDebug && !process.env.VUE_CLI_TEST_DO_INSTALL_PLUGIN) {
    // in development, avoid installation process
    await require('./util/setupDevProject')(context)
  } else {
    await pm.install()
  }
  ...
}
```

这段代码会先调用 shouldInitGit 来判断是否需要 git 初始化，判断的情形有以下几种：

- 没有安装 git (!hasGit())：false；
- vue create 含有 --git 或者 -g 选项：true；
- vue create 含有 --no-git 或者 -n 选项：false；
- 生成项目的目录是否已经含有 git （!hasProjectGit(this.context)）：如果有，则返回 false，否则返回 true。

在判断完是否需要 git 初始化项目后，接下来就会调用 installDeps 安装依赖，还是看下 installDeps 的源码：

