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

这些指令都是在 `@vue/cli/bin/vue.js` 文件中通过 command 库的 command 方法指定的，此处比较有意思的是 @vue/cli 提供了 suggestCommands 函数，用于对用户在命令行的错误输入分析并给出猜测指令，具体实现如下：

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


首先我们开始分析项目创建最核心的 create 命令：

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

具体的执行则主要是调用了 `../lib/create.js` 文件暴露出的 create 方法：

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

此处经过认真的代码分析我们成功发现真正实现的核心都在 `../lib/Creator` 的 Creator 类中😓，，，但是这儿 getPromptModules 方法的作用是什么呢？

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

- presetPrompt： 预设选项 prompt，加入上次以 Manually 模式进行了预设选项，并且保存到了 ~/.vuerc 中，那么在初始化项目时就会列出已经保存的 preset，并提供选择。
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

PromptModuleAPI 实例会调用它的实例方法，然后将 injectFeature，injectPrompt，injectOptionForPrompt，onPromptComplete 保存到 Creator 实例对应的变量中。最后遍历前面 getPromptModules 获取的 promptModules，传入实例 promptAPI，初始化 Creator 实例中 featurePrompt, injectedPrompts, promptCompleteCbs 变量。

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

先判断 vue create 命令是否带有 -p 选项，如果有的话会调用 resolvePreset 去解析 preset。resolvePreset 函数会先获取 `～/.vuerc` 中保存的 preset， 然后进行遍历，如果里面包含了 -p 中的 `<presetName>`，则返回 `～/.vuerc` 中的 preset。如果没有则判断是否是采用内联的 JSON 字符串预设选项，如果是就会解析 `.json` 文件，并返回 preset，还有一种情况就是从远程获取 preset（利用 download-git-repo 下载远程的 preset.json 并返回。

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

我们接着查看 create 方法在获取到 preset 之后，还会向 preset 的插件里面注入核心插件 `@vue/cli-service`， 它是调用 `vue-cli-service <command> [...args]` 时创建的类。 负责管理内部的 webpack 配置、暴露服务和构建项目的命令等。

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

此处代码主要有两处作用：**获取最新 CLI （包含插件）的版本** 和 **生成 package.json**，这里 getVersions 方法用于判断 @vue/cli 是否需要更新以及初始化项目中相关插件的版本，需要注意的是，getVersions 获取 CLI 的版本并不是直接获取， 而是通过 `vue-cli-version-marker` npm 包获取的 CLI 版本，为什么会这样做，主要原因有两点：

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

在判断完是否需要 git 初始化项目后，接下来就会调用 `pm.install` 安装依赖，还是看下其源码：

```ts
async install () {
  const args = []

  if (this.needsPeerDepsFix) {
    args.push('--legacy-peer-deps')
  }

  if (process.env.VUE_CLI_TEST) {
    args.push('--silent', '--no-progress')
  }

  return await this.runCommand('install', args)
}
async runCommand (command, args) {
  const prevNodeEnv = process.env.NODE_ENV
  // In the use case of Vue CLI, when installing dependencies,
  // the `NODE_ENV` environment variable does no good;
  // it only confuses users by skipping dev deps (when set to `production`).
  delete process.env.NODE_ENV

  await this.setRegistryEnvs()
  await executeCommand(
    this.bin,
    [
      ...PACKAGE_MANAGER_CONFIG[this.bin][command],
      ...(args || [])
    ],
    this.context
  )

  if (prevNodeEnv) {
    process.env.NODE_ENV = prevNodeEnv
  }
}
```

源码中调用了 setRegistryEnvs 函数，它的作用是指定安装源，具体实现如下：

```js
// Any command that implemented registry-related feature should support
// `-r` / `--registry` option
async getRegistry (scope) {
  const cacheKey = scope || ''
  if (this._registries[cacheKey]) {
    return this._registries[cacheKey]
  }

  const args = minimist(process.argv, {
    alias: {
      r: 'registry'
    }
  })

  let registry
  if (args.registry) {
    registry = args.registry
  } else if (!process.env.VUE_CLI_TEST && await shouldUseTaobao(this.bin)) {
    registry = registries.taobao
  } else {
    try {
      if (scope) {
        registry = (await execa(this.bin, ['config', 'get', scope + ':registry'])).stdout
      }
      if (!registry || registry === 'undefined') {
        registry = (await execa(this.bin, ['config', 'get', 'registry'])).stdout
      }
    } catch (e) {
      // Yarn 2 uses `npmRegistryServer` instead of `registry`
      registry = (await execa(this.bin, ['config', 'get', 'npmRegistryServer'])).stdout
    }
  }

  this._registries[cacheKey] = stripAnsi(registry).trim()
  return this._registries[cacheKey]
}
```

示例中如果 vue create 还有 -r 选项则采用设置的安装源，否则调用 shouldUseTaobao 函数来判断是否需要使用淘宝 NPM 镜像源，如果两者都不匹配则会调用 [execa](https://www.npmjs.com/package/execa) 来直接执行 `this.bin` 文件，这儿的 bin 属性实际上就是 PackageManager 类创建的时候传入的 forcePackageManager 参数，即 yarn / npm / pnpm，在选定安装源后紧接着调用 executeCommand 函数，executeCommand 函数利用了 execa 执行 npm 或者 yarn 安装命令。至此**安装依赖**的流程就全部结束了，接下来就是 `vue create` 最核心的部分**代码生成**：

```js
async create (cliOptions = {}, preset = null) {
  ...
  // run generator
  log(`🚀  Invoking generators...`)
  this.emit('creation', { event: 'invoking-generators' })
  const plugins = await this.resolvePlugins(preset.plugins, pkg)
  const generator = new Generator(context, {
    pkg,
    plugins,
    afterInvokeCbs,
    afterAnyInvokeCbs
  })
  await generator.generate({
    extractConfigFiles: preset.useConfigFiles
  })
  ...
}
```

示例中在实例化 Generator 实例之前调用了 resolvePlugins 对插件进行初始化，举个例子：`[{ '@vue/cli-plugin-babel': { useTsWithBabel: true } }]` 的入参将被转化生成 `[{ id: '@vue/cli-plugin-babel', apply, options: { useTsWithBabel: true }}]`，这儿主要的不同是多了 apply 对象，实际上了对应插件下 `generator` 文件的加载对象，在示例中也就是 `@vue/cli-plugin-babel/generator` 文件。

再完成了对插件的初始化和每个插件下的 `generator.js` 文件的加载后正式开始 Generator 实例的创建，在实例化一个 Generator 的时候会初始化一些成员变量，最重要的就是调用插件的 generators，不同于 1.x/2.x 基于模板的脚手架，Vue-cli3.0 采用了一套 基于插件的架构，到这里就会交给各个插件去执行了，看一下 Generator 实例化的代码：

```js
module.exports = class Generator {
  constructor (context, {
    pkg = {},
    plugins = [],
    afterInvokeCbs = [],
    afterAnyInvokeCbs = [],
    files = {},
    invoking = false
  } = {}) {
    this.context = context
    this.plugins = plugins
    this.originalPkg = pkg
    this.pkg = Object.assign({}, pkg)
    this.pm = new PackageManager({ context })
    this.imports = {}
    this.rootOptions = {}
    this.afterInvokeCbs = afterInvokeCbs
    this.afterAnyInvokeCbs = afterAnyInvokeCbs
    this.configTransforms = {} // 插件通过 GeneratorAPI 暴露的 addConfigTransform 方法添加如何提取配置文件
    this.defaultConfigTransforms = defaultConfigTransforms // 默认的配置文件
    this.reservedConfigTransforms = reservedConfigTransforms // 保留的配置文件 vue.config.js
    this.invoking = invoking
    // for conflict resolution
    this.depSources = {}
    // virtual file tree
    this.files = Object.keys(files).length
      // when execute `vue add/invoke`, only created/modified files are written to disk
      ? watchFiles(files, this.filesModifyRecord = new Set())
      // all files need to be written to disk
      : files
    this.fileMiddlewares = []
    this.postProcessFilesCbs = []
    // exit messages
    this.exitLogs = []

    // load all the other plugins
    this.allPluginIds = Object.keys(this.pkg.dependencies || {})
      .concat(Object.keys(this.pkg.devDependencies || {}))
      .filter(isPlugin)

    const cliService = plugins.find(p => p.id === '@vue/cli-service')
    const rootOptions = cliService
      ? cliService.options
      : inferRootOptions(pkg)

    this.rootOptions = rootOptions
  }
  ...
}
```

然后查看其具体被调用实例的 generate 方法：

```js
async generate ({
  extractConfigFiles = false,
  checkExisting = false
} = {}) {
  await this.initPlugins()

  // save the file system before applying plugin for comparison
  const initialFiles = Object.assign({}, this.files)
  // extract configs from package.json into dedicated files.
  this.extractConfigFiles(extractConfigFiles, checkExisting)
  // wait for file resolve
  await this.resolveFiles()
  // set package.json
  this.sortPkg()
  this.files['package.json'] = JSON.stringify(this.pkg, null, 2) + '\n'
  // write/update file tree to disk
  await writeFileTree(this.context, this.files, initialFiles, this.filesModifyRecord)
}
```

这儿 initPlugins 的主要作用是为每个插件对应生成一个 GeneratorAPI 实例，同时将实例作为参数传递给前面插件初始化时生成的插件下 `generator` 文件的加载对象，也可以理解为将实例 api 传入插件暴露出来的 generator 函数，这儿的最重要的就是 GeneratorAPI 了。

前面说到 Vue CLI 是基于一套插件架构的，那么如果插件需要自定义项目模板、修改模板中的一些文件或者添加一些依赖的话怎么处理呢？方法是 @vue/cli 插件所提供的 generator 向外暴露一个函数，接收的第一个参数 api，然后通过该 api 提供的方法去完成应用的拓展工作，这里所说 的 api 就是 GeneratorAPI，下面看一下 GeneratorAPI 提供了哪些方法。

- hasPlugin：判断项目中是否有某个插件
- extendPackage：拓展 package.json 配置
- render：利用 ejs 渲染模板文件
- onCreateComplete：内存中保存的文件字符串全部被写入文件后的回调函数
- exitLog：当 generator 退出的时候输出的信息
- genJSConfig：将 json 文件生成为 js 配置文件
- injectImports：向文件当中注入import语法的方法
- injectRootOptions：向 Vue 根实例中添加选项
- ...

下面以 `@vue/cli-service/generator/index.js` 为例：

```js
module.exports = (api, options) => {
  /* 渲染 ejs 模板 */
  api.render('./template', {
    doesCompile: api.hasPlugin('babel') || api.hasPlugin('typescript')
  })
  
  // 扩展 package.json
  api.extendPackage({
    scripts: {
      'serve': 'vue-cli-service serve',
      'build': 'vue-cli-service build'
    },
    dependencies: {
      'vue': '^2.5.17'
    },
    devDependencies: {
      'vue-template-compiler': '^2.5.17'
    },
    'postcss': {
      'plugins': {
        'autoprefixer': {}
      }
    },
    browserslist: [
      '> 1%',
      'last 2 versions',
      'not ie <= 8'
    ]
  })
  // 如果 preset 中包含 vue-router
  if (options.router) {
    require('./router')(api, options)
  }
  
  // 如果 preset 中包含 vuex
  if (options.vuex) {
    require('./vuex')(api, options)
  }
  
  // 如果 preset 中包含 cssPreprocessor，即选择了 css 预处理器
  if (options.cssPreprocessor) {
    const deps = {
      sass: {
        'node-sass': '^4.9.0',
        'sass-loader': '^7.0.1'
      },
      less: {
        'less': '^3.0.4',
        'less-loader': '^4.1.0'
      },
      stylus: {
        'stylus': '^0.54.5',
        'stylus-loader': '^3.0.2'
      }
    }

    api.extendPackage({
      devDependencies: deps[options.cssPreprocessor]
    })
  }

  // additional tooling configurations
  if (options.configs) {
    api.extendPackage(options.configs)
  }
}
```

通过示例我们可以感受到 vue-cli 3.0 的插件机制的核心：**通过 GeneratorAPI 所暴露的方法将所有功能都交给插件去完成**。对于 vue-cli 3.0 内置的插件，比如：@vue/cli-plugin-eslint 、@vue/cli-plugin-pwa 等等，以及其他第三方插件，他们的 generator 作用都是一样都可以向项目的 package.json 中注入额外的依赖或字段，并向项目中添加文件。我们也可以结合 GeneratorAPI 所暴露的方法自定义插件，因此 vue-cli 3.0 的插件机制给予了我们极大的灵活性与扩展性。

在完成对每个插件的处理之后开始进入到了**生成项目文件**的阶段，大致可以分为三部分，**extractConfigFiles（提取配置文件）**，**resolveFiles（模板渲染）** 和 **writeFileTree（在磁盘上生成文件）**。

1、 extractConfigFiles

提取配置文件指的是将一些插件（比如 eslint，babel）的配置从 package.json 的字段中提取到专属的配置文件中。下面以 eslint 为例进行分析： 在初始化项目的时候，如果选择了 eslint 插件，在调用 @vue/cli-plugin-eslint 的 generator 的时候，就会向 package.json 注入 eslintConfig 字段：

```js
module.exports = (api, { config, lintOn = [] }, _, invoking) => {
  if (typeof lintOn === 'string') {
    lintOn = lintOn.split(',')
  }

  const eslintConfig = require('../eslintOptions').config(api)

  const pkg = {
    scripts: {
      lint: 'vue-cli-service lint'
    },
    eslintConfig,
    // TODO:
    // Move these dependencies to package.json in v4.
    // Now in v3 we have to add redundant eslint related dependencies
    // in order to keep compatibility with v3.0.x users who defaults to ESlint v4.
    devDependencies: {
      'babel-eslint': '^10.0.1',
      'eslint': '^5.8.0',
      'esliint-plugin-vue': '^5.0.0-0'
    }
  }

  const injectEditorConfig = (config) => {
    const filePath = api.resolve('.editorconfig')
    if (fs.existsSync(filePath)) {
      // Append to existing .editorconfig
      api.render(files => {
        const configPath = path.resolve(__dirname, `./template/${config}/_editorconfig`)
        const editorconfig = fs.readFileSync(configPath, 'utf-8')

        files['.editorconfig'] += `\n${editorconfig}`
      })
    } else {
      api.render(`./template/${config}`)
    }
  }

  if (config === 'airbnb') {
    eslintConfig.extends.push('@vue/airbnb')
    Object.assign(pkg.devDependencies, {
      '@vue/eslint-config-airbnb': '^4.0.0'
    })
    injectEditorConfig('airbnb')
  } else if (config === 'standard') {
    eslintConfig.extends.push('@vue/standard')
    Object.assign(pkg.devDependencies, {
      '@vue/eslint-config-standard': '^4.0.0'
    })
    injectEditorConfig('standard')
  } else if (config === 'prettier') {
    eslintConfig.extends.push('@vue/prettier')
    Object.assign(pkg.devDependencies, {
      '@vue/eslint-config-prettier': '^4.0.0'
    })
    // prettier & default config do not have any style rules
    // so no need to generate an editorconfig file
  } else {
    // default
    eslintConfig.extends.push('eslint:recommended')
  }

  api.extendPackage(pkg)

}
```

这是 `@vue/cli-plugin-eslint/generator/index.js` 中的一部分代码，从代码中可以看出，利用 GeneratorAPI 的 extendPackage 方法向 package.josn 里面注入了 scripts，eslintConfig 以及 devDependencies 字段，另外也会根据选择的 eslint 模式添加对应的依赖和修改对应的配置文件，例如选择了 airbnb 模式，就会向 eslintConfig.extends 添加 @vue/airbnb 配置，并且添加 @vue/eslint-config-airbnb 依赖和修改 .editorconfig 配置文件。此时 项目 package.json 中 eslintConfig 字段内容如下：

```json
{
"eslintConfig": {
    "root": true,
    "env": {
      "node": true
    },
    "extends": [
      "plugin:vue/essential",
      "@vue/airbnb"
    ],
    "rules": {},
    "parserOptions": {
      "parser": "babel-eslint"
    }
  }
}
```

如果 preset 的 useConfigFiles 为 true ，或者以 Manually 模式初始化 preset 的时候选择 In dedicated config files 存放配置文件，那么 extractConfigFiles 方法就会将 package.json 中 eslintConfig 字段内容提取到 .eslintrc.js 文件中，内存中 .eslintrc.js 内容如下：

```js
module.exports = {
  root: true,
  env: {
    node: true,
  },
  extends: [
    'plugin:vue/essential',
    '@vue/airbnb',
  ],
  rules: {
    'no-console': process.env.NODE_ENV === 'production' ? 'error' : 'off',
    'no-debugger': process.env.NODE_ENV === 'production' ? 'error' : 'off',
  },
  parserOptions: {
    parser: 'babel-eslint',
  },
};
```

extractConfigFiles 方法的具体实现主要是调用 ConfigTransform 实例的 transform 方法，代码实现的比较清晰，各位同学可以自己看下。这里就不做详细 分析了，在配置文件提取完了以后接下来就是执行 resolveFiles 函数了。

2、 resolveFiles

resolveFiles 主要分为以下三个部分执行：

- fileMiddlewares
- injectImportsAndOptions
- postProcessFilesCbs

fileMiddlewares 里面包含了 ejs render 函数，所有插件调用 api.render 时候只是把对应的渲染函数 push 到了 fileMiddlewares 中，等所有的 插件执行完以后才会遍历执行 fileMiddlewares 里面的所有函数，即在内存中生成模板文件字符串。

injectImportsAndOptions 就是将 generator 注入的 import 和 rootOption 解析到对应的文件中，比如选择了 vuex, 会在 src/main.js 中添加 import store from './store'，以及在 vue 根实例中添加 router 选项。

postProcessFilesCbs 是在所有普通文件在内存中渲染成字符串完成之后要执行的遍历回调。例如将 @vue/cli-service/generator/index.js 中的 render 是放在了 fileMiddlewares 里面，而将 @vue/cli-service/generator/router/index.js 中将替换 src/App.vue 文件的方法放在了 postProcessFiles 里面，原因是对 src/App.vue 文件的一些替换一定是发生在 render 函数之后，如果在之前，修改后的 src/App.vue 在之后 render 函数执行时又会被覆盖，这样显然不合理。


3、 writeFileTree

在提取了配置文件和模板渲染之后调用了 sortPkg 对 package.json 的字段进行了排序并将 package.json 转化为 json 字符串添加到项目的 files 中。 此时整个项目的文件已经在内存中生成好了（在源码中就是对应的 this.files），接下来就调用 writeFileTree 方法将内存中的字符串模板文件生成在磁盘中。到这里 vue create 核心部分 generator 基本上就分析完了。


<img src="/assets/emoji/04.png" width="280" />

接下来在 `../lib/Creator.js` 文件的 create 方法中还有一些其他额外处理，主要包括：

- 安装额外依赖：这里的依赖来源于 preset 的 option，比如选择了 scss css 预处理器，那么就需要额外安装 node-sass 和 sass-loader 两个依赖。
- 执行 createCompleteCbs：所有文件都写在磁盘后执行的遍历回调。
- 生成 README.md：根据 package.json 的 script 的字段生成生成对应的 README.md。
- git 初始化提交：调用 shouldInitGit 来判断是否需要 git 初始化提交，如果需要初始化提交就会执行 git add 和 git commmit 命令。
- 日志输出：插件的 generator 可以利用 GeneratorAPI 暴露的 exitLog 方法在项目输出其他所有的 message 之后输出一些日志。

**总结：**

vue create 命令总的来说就实例化了2个类，然后各自调了一个方法。当执行 create 命令时会创建一个 Creator 实例， 然后调用其 create 方法。在 create 方法中又会创建一个 Generator 实例，然后再调用其 generator 方法，初始化整个项目会交给各个插件去完成。比如核心插件 @vue/cli-service 会渲染整个项目的文件，@vue/cli-plugin-eslint 会进行 eslint 的一些配置， @vue/cli-plugin-unit-jest 插件也会进行一些 jest 的配置，各个功能都会交给对应的插件去完成，而 GeneratorAPI 又允许一个插件的 generator 向 package.json 注入额外的依赖或字段，并向项目中添加文件，这也让插件对整个项目拥有更高的可配置性。

#### 2、add 命令

文章前面通过解析 vue create 命令的具体执行流程介绍了项目创建初始化的主要过程，其最核心的就是插件机制，其使得用户对项目有了更高的可配置性，再完成项目初始构建后我们依然可以通过 vue add 指令添加插件，那么其实现原理有是什么呢？

vue add 命令的入口在 packages/@vue/cli/bin/vue.js 中:

```js
program
  .command('add <plugin> [pluginOptions]')
  .description('install a plugin and invoke its generator in an already created project')
  .option('--registry <url>', 'Use specified npm registry when installing dependencies (only for npm)')
  .allowUnknownOption()
  .action((plugin) => {
    require('../lib/add')(plugin, minimist(process.argv.slice(3)))
  })
```

从代码中可以看出，vue add 命令接受两个参数:

- plugin： 插件名称，必填。
- registry： 安装插件指定的安装源，只针对于 npm 包管理器，选填。

当执行了 vue add 命令后会加载 @vue/cli/lib/add.js，下面就逐步开始分析。

```js
module.exports = (...args) => {
  return add(...args).catch(err => {
    error(err)
    if (!process.env.VUE_CLI_TEST) {
      process.exit(1)
    }
  })
}
```

我们接着来查看 add 方法：

```js
async function add (pluginToAdd, options = {}, context = process.cwd()) {
  if (!(await confirmIfGitDirty(context))) {
    return
  }

  // for `vue add` command in 3.x projects
  const servicePkg = loadModule('@vue/cli-service/package.json', context)
  if (servicePkg && semver.satisfies(servicePkg.version, '3.x')) {
    // special internal "plugins"
    if (/^(@vue\/)?router$/.test(pluginToAdd)) { // 匹配 @vue/router，router
      return addRouter(context)
    }
    if (/^(@vue\/)?vuex$/.test(pluginToAdd)) { // 匹配 @vue/vuex，vuex
      return addVuex(context)
    }
  }

  // 解析插件名称
  // @bar/foo => @bar/vue-cli-plugin-foo
  // @vue/foo => @vue/cli-plugin-foo
  // foo => vue-cli-plugin-foo
  const pluginRe = /^(@?[^@]+)(?:@(.+))?$/
  const [
    // eslint-disable-next-line
    _skip,
    pluginName,
    pluginVersion
  ] = pluginToAdd.match(pluginRe)
  const packageName = resolvePluginId(pluginName)

  log()
  log(`📦  Installing ${chalk.cyan(packageName)}...`)
  log()

  const pm = new PackageManager({ context })

  if (pluginVersion) {
    await pm.add(`${packageName}@${pluginVersion}`)
  } else if (isOfficialPlugin(packageName)) {
    const { latestMinor } = await getVersions()
    await pm.add(`${packageName}@~${latestMinor}`)
  } else {
    await pm.add(packageName, { tilde: true })
  }

  log(`${chalk.green('✔')}  Successfully installed plugin: ${chalk.cyan(packageName)}`)
  log()

  const generatorPath = resolveModule(`${packageName}/generator`, context)
  if (generatorPath) {
    invoke(pluginName, options, context)
  } else {
    log(`Plugin ${packageName} does not have a generator to invoke`)
  }
}
```

add 函数的主要功能就是安装插件包，对于 vue-cli 内部一些特殊的"插件"，比如 router，vuex，就不会通过包管理器安装，而是直接加载 @vue/cli-service/generator/router 和 @vue/cli-service/generator/vuex，这两个文件也是两个 generator，可以向 package.json 注入额外的依赖或字段，并向项目中添加文件，对于普通的第三方插件，则需要通过包管理器安装，示例调用的 `pm.add` 方法实现如下所示：

```js
// @vue/cli/lib/util/ProjectPackageManager.js
async add (packageName, {
  tilde = false,
  dev = true
} = {}) {
  const args = dev ? ['-D'] : []
  if (tilde) {
    if (this.bin === 'yarn') {
      args.push('--tilde')
    } else {
      process.env.npm_config_save_prefix = '~'
    }
  }

  if (this.needsPeerDepsFix) {
    args.push('--legacy-peer-deps')
  }

  return await this.runCommand('add', [packageName, ...args])
}
```

如果用户指定插件包名称时未指定版本号那么在调用 `pm.add` 方法时会传入参数 `tilde: true`，在 `pm.add` 的内部会根据包管理器的具体类型进行判断，如果是 yarn 会添加 --tilde 参数，如果是 npm 或 pnpm 则会将 npm_config_save_prefix 的配置设为 '~'，这个配置的默认值为 '^', 例如一个包的版本是 1.2.3，默认情况下那么在 package.json 中就会被设置为 '^1.2.3'，这样，对于 minor 部分的更新依然是可以的。但是如果设置了 npm config set save-prefix=”~”，那么就会被设置为 ”~1.2.3”，此时只允许最后一位数字的版本更新。最后再通过 runCommand 函数执行具体包管理器（如 yarn）的 add 指令。

在 vue-cli 官方文档中对插件 的命名有着明确的要求，即命名方式为：`vue-cli-plugin-<name>`，插件遵循命名约定之后就可以：

- 被 @vue/cli-service 发现；
- 被其它开发者搜索到；
- 通过 `vue add <name>` 或 `vue invoke <name>` 安装下来。

在获取第三方插件名称后，就会调用 invoke 方法安装插件包：

```js
async function invoke (pluginName, options = {}, context = process.cwd()) {
  if (!(await confirmIfGitDirty(context))) {
    return
  }

  delete options._
  const pkg = getPkg(context) // 解析 package.json

  // attempt to locate the plugin in package.json
  const findPlugin = deps => {
    if (!deps) return
    let name
    // official
    if (deps[(name = `@vue/cli-plugin-${pluginName}`)]) {
      return name
    }
    // full id, scoped short, or default short
    if (deps[(name = resolvePluginId(pluginName))]) {
      return name
    }
  }

  const id = findPlugin(pkg.devDependencies) || findPlugin(pkg.dependencies)
  if (!id) {
    throw new Error(
      `Cannot resolve plugin ${chalk.yellow(pluginName)} from package.json. ` +
        `Did you forget to install it?`
    )
  }

  const pluginGenerator = loadModule(`${id}/generator`, context)
  if (!pluginGenerator) {
    throw new Error(`Plugin ${id} does not have a generator.`)
  }

  // resolve options if no command line options (other than --registry) are passed,
  // and the plugin contains a prompt module.
  // eslint-disable-next-line prefer-const
  let { registry, $inlineOptions, ...pluginOptions } = options
  if ($inlineOptions) {
    try {
      pluginOptions = JSON.parse($inlineOptions)
    } catch (e) {
      throw new Error(`Couldn't parse inline options JSON: ${e.message}`)
    }
  } else if (!Object.keys(pluginOptions).length) {
    let pluginPrompts = loadModule(`${id}/prompts`, context)
    if (pluginPrompts) {
      const prompt = inquirer.createPromptModule()

      if (typeof pluginPrompts === 'function') {
        pluginPrompts = pluginPrompts(pkg, prompt)
      }
      if (typeof pluginPrompts.getPrompts === 'function') {
        pluginPrompts = pluginPrompts.getPrompts(pkg, prompt)
      }
      pluginOptions = await prompt(pluginPrompts)
    }
  }

  const plugin = {
    id,
    apply: pluginGenerator,
    options: {
      registry,
      ...pluginOptions
    }
  }

  await runGenerator(context, plugin, pkg)
}
```

该方法先调用 findPlugin 判断插件是否安装，如果安装也返回插件名，接着获取目标插件下的 generator 方法。然后就是判断命令行是否传入行内参数，如果没有并且传入插件参数为空则调用 inquirer.prompt 获取插件的 option，并传给其 generator，在完成这些以后，就是 runGenerator。这儿的 runGenerator 的具体实现与 vue create 命令中项目生成的阶段基本一致，实质上就是构造一个 Generator 实例，并调用其 generate 方法。在实例的 generator 方法调用完成之后执行以下命令：

```sh
git ls-files --exclude-standard --modified --others
```

因为插件的 generator 可以通过 GeneratorAPI 暴露的 render 和 extendPackage 方法修改项目的文件，因此通过执行该命令将变化的文件显示在终端上，这对开发者十分地友好。

vue add 相当于只是 vue create 中的一部分，vue create 包含了插件的安装以及调用，vue add 命令只是将此功能分离了出来，而 vue invoke 命令其实是 vue add 命名去掉了插件包安装的那一部分，由于篇幅限制（wo tai lan le）不再介绍。

<img src="/assets/emoji/05.png" width="280" />


## 参考资料

[vue-cli-analysis](https://kuangpf.com/vue-cli-analysis)

[剖析 Vue CLI 实现原理](https://cloud.tencent.com/developer/article/1781202)