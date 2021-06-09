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

vue-cli 在 3.0 版本进行了彻底的重构，为了区别也将其普遍称为 Vue CLI，是目前 Vue 官方推荐的 Vue 项目快速开发的完整系统，它基于 Webpack 实现，提供了终端命令行工具、零配置脚手架、插件体系、图形化管理界面等诸多功能，近乎提供了前端项目工程化的所有步骤的完整工具链，也是当前 Vue 项目构建的主流工具。

### 一、整体架构

vue-cli 为了尽可能覆盖项目工程化需求所以模版项目往往引入了大量第三方库，但实际开发过程中开发者可能并不需要这些功能模块，虽然可以通过在模版项目的 meta.js 或 meta.json 文件配置 prompts 在命令行交互然后在 filterFiles 函数中对生成项目的文件目录结构进行筛选，但是依然可配置性不强，会存在较多冗余依赖或功能，并且依赖项的升级极为痛苦，因此为了给开发者提供更灵活的配置能力 Vue CLI 实现了一种极为巧妙的架构设计：

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
- 加载其他 CLI 插件内部的核心服务。

3、@vue/cli-plugin-xxx: 个人觉得 Vue CLI 设计最出彩的地方就在于插件机制，项目本身创建最开始的时候只是根据用户在命令行的交互问题生成 package.json，然后根据所拥有的模块动态完成 Webpack 的配置并生成项目结构，因此 Vue CLI 的插件模块主要完成如下功能：

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

vue add 相当于只是 vue create 中的一部分，vue create 包含了插件的安装以及调用，vue add 命令只是将此功能分离了出来，而 vue invoke 命令其实是 vue add 命名去掉了插件包安装的那一部分，其他 vue 命令由于篇幅限制（wo tai lan le）此处不再介绍。

### 三、@vue/cli-service

在 Vue CLI 中将 webpack 及相关插件提供的功能都收敛到 @vue/cli-service 内部来实现，提供了 vue-cli-service 命令帮助使用者最基本的日常开发，下面就开始分析 @vue/cli-service 的具体实现。

#### 1、入口

vue-cli-service m命令的入口在 @vue/cli-service/bin/vue-service-service.js 中，代码如下：

```js
// @vue/cli-service/bin/vue-service-service.js

#!/usr/bin/env node

const { semver, error } = require('@vue/cli-shared-utils')
const requiredVersion = require('../package.json').engines.node

// Node 版本校验
if (!semver.satisfies(process.version, requiredVersion, { includePrerelease: true })) {
  error(
    `You are using Node ${process.version}, but vue-cli-service ` +
    `requires Node ${requiredVersion}.\nPlease upgrade your Node version.`
  )
  process.exit(1)
}

const Service = require('../lib/Service')
const service = new Service(process.env.VUE_CLI_CONTEXT || process.cwd())

// 获取命令参数 vue-cli-service serve --https
const rawArgv = process.argv.slice(2)
// 在 boolean 选项当中的参数会被解析成 true，例如 args.https: true, args.modern: false
const args = require('minimist')(rawArgv, {
  boolean: [
    // build
    // FIXME: --no-module, --no-unsafe-inline, no-clean, etc.
    'modern',
    'report',
    'report-json',
    'inline-vue',
    'watch',
    // serve
    'open',
    'copy',
    'https',
    // inspect
    'verbose'
  ]
})
const command = args._[0] // serve

service.run(command, args, rawArgv).catch(err => {
  error(err)
  process.exit(1)
})
```

代码看着非常简洁，与一般 node 命令文件有点不同，第一就是并没有依赖 commander.js，第二就是并没有直接提供相关 CLI 命令的服务。 在 @vue/vue-cli-service 中，对于第一点是通过 Service 这个类来处理 node 命令，而对于第二点，所有的 CLI 服务都是动态注册的。从上面这段代码可以看出，执行 CLI 命令后，主要有两个操作：实例化 Service 和调用实例的 run 方法。

#### 2、Service 类实例化

我们接着查看 Service 类的构造函数：

```js
// @vue/cli-service/lib/Service.js

module.exports = class Service {
  constructor (context, { plugins, pkg, inlineOptions, useBuiltIn } = {}) {
    checkWebpack(context)

    process.VUE_CLI_SERVICE = this
    this.initialized = false
    this.context = context
    this.inlineOptions = inlineOptions
    this.webpackChainFns = []
    this.webpackRawConfigFns = []
    this.devServerConfigFns = []
    this.commands = {}
    // Folder containing the target package.json for plugins
    this.pkgContext = context
    // package.json containing the plugins
    this.pkg = this.resolvePkg(pkg)
    // If there are inline plugins, they will be used instead of those
    // found in package.json.
    // When useBuiltIn === false, built-in plugins are disabled. This is mostly
    // for testing.
    // 如果有 inline plugins 的话，就不会去加载 package.json 里 devDependencies 和 dependencies 的插件，useBuiltIn 为 false 时 builtInPlugins 会被禁用
    this.plugins = this.resolvePlugins(plugins, useBuiltIn)
    // pluginsToSkip will be populated during run()
    this.pluginsToSkip = new Set()
    // resolve the default mode to use for each command
    // this is provided by plugins as module.exports.defaultModes
    // so we can get the information without actually applying the plugin.
    // 注册的插件可以通过 module.exports.defaultModes 指定特定的模式
    /*{
      serve: 'development',
      build: 'production',
      inspect: 'development',
      'test:unit': 'test'
    }*/
    this.modes = this.plugins.reduce((modes, { apply: { defaultModes } }) => {
      return Object.assign(modes, defaultModes)
    }, {})
  }
  ...
}
```

实例化的过程主要进行了插件的解析和为每一种 CLI 命令指定模式，先看一下插件的解析：

```js
// @vue/cli-service/lib/Service.js
module.exports = class Service {
  ...
  resolvePlugins (inlinePlugins, useBuiltIn) {
    const idToPlugin = (id, absolutePath) => ({
      id: id.replace(/^.\//, 'built-in:'),
      apply: require(absolutePath || id)
    })

    let plugins
 
    // 内置插件
    const builtInPlugins = [
      './commands/serve',
      './commands/build',
      './commands/inspect',
      './commands/help',
      // config plugins are order sensitive
      './config/base',
      './config/assets',
      './config/css',
      './config/prod',
      './config/app'
    ].map((id) => idToPlugin(id)) // [{ id: 'built-in:commands/serve', apply:{ [Function] defaultModes: [Object] } },...]

    // 实例化 Service 时传入插件
    if (inlinePlugins) {
      plugins = useBuiltIn !== false
        ? builtInPlugins.concat(inlinePlugins)
        : inlinePlugins
    } else {
      const projectPlugins = Object.keys(this.pkg.devDependencies || {})
        .concat(Object.keys(this.pkg.dependencies || {}))
        .filter(isPlugin) // isPlugin = id => /^(@vue\/|vue-|@[\w-]+\/vue-)cli-plugin-/.test(id)
        .map(id => {
          if (
            this.pkg.optionalDependencies &&
            id in this.pkg.optionalDependencies
          ) {
            let apply = loadModule(id, this.pkgContext)
            if (!apply) {
              warn(`Optional dependency ${id} is not installed.`)
              apply = () => {}
            }

            return { id, apply }
          } else {
            return idToPlugin(id, resolveModule(id, this.pkgContext))
          }
        })

      // Add the plugin automatically to simplify the webpack-4 tests
      // so that a simple Jest alias would suffice, avoid changing every
      // preset used in the tests
      if (
        process.env.VUE_CLI_TEST &&
        process.env.VUE_CLI_USE_WEBPACK4 &&
        !projectPlugins.some((p) => p.id === '@vue/cli-plugin-webpack-4')
      ) {
        builtInPlugins.push(idToPlugin('@vue/cli-plugin-webpack-4'))
      }

      plugins = builtInPlugins.concat(projectPlugins)
    }

    // Local plugins
    // 项目本地的插件，针对于只需要在项目里直接访问插件 API 而不需要创建一个完整的插件
    if (this.pkg.vuePlugins && this.pkg.vuePlugins.service) {
      const files = this.pkg.vuePlugins.service
      if (!Array.isArray(files)) {
        throw new Error(`Invalid type for option 'vuePlugins.service', expected 'array' but got ${typeof files}.`)
      }
      plugins = plugins.concat(files.map(file => ({
        id: `local:${file}`,
        apply: loadModule(`./${file}`, this.pkgContext)
      })))
    }
    debug('vue:plugins')(plugins)

    const orderedPlugins = sortPlugins(plugins)
    debug('vue:plugins-ordered')(orderedPlugins)

    return orderedPlugins
  }
  ...
}
```

可以将解析的插件分为4类：

- 内置插件
- inlinePlugins
- package.json 插件
- package.vuePlugins 插件

**内置插件**指的是 @vue/cli-service 内部提供的插件，又可以大致分为两类，serve、build、inspect、help 这一类插件在内部动态注册新的 CLI 命令， 开发者即可通过 npm script 的形式去启动对应的 CLI 命令服务，base ,css, dev, prod, app 这一类插件主要是完成 webpack 本地编译构建时的各种相关的配置。 @vue/cli-service 将 webpack 的开发构建功能收敛到内部来完成。

**inlinePlugins** 指的是直接在实例化 Service 时传入，执行 vue serve 和 vue build 命令时会创建一个 Service 实例，并传入 inlinePlugins。

**package.json 插件**指的是 devDependencies 和 dependencies 中的 vue 插件，比如 @vue/cli-plugin-eslint。

**package.vuePlugins** 也是在 package.json 中的插件，不过是在 vuePlugins 字段中，该类插件是针对于只需要在项目里直接访问插件 API 而不需要创建一个完整的插件。

我们接着查看 CLI 指定模式：

CLI 模式是 vue cli 中一个重要的概念，默认情况下，一个 Vue CLI 项目有三个模式：

- development 模式用于 vue-cli-service serve
- test 模式用于 vue-cli-service test:unit
- production 模式用于 vue-cli-service build 和 vue-cli-service test:e2e

我们也可以通过传递 --mode 选项参数为命令行覆写默认的模式。详情可查看[官方文档](https://cli.vuejs.org/zh/guide/mode-and-env.html#%E6%A8%A1%E5%BC%8F)。

在 Service 的构造函数中解析完插件之后就为每种插件命令指定模式，插件命令的模式可以 通过 module.exports.defaultModes 以 { [commandName]: mode } 的形式来暴露：

```js
module.exports = api => {
  api.registerCommand('build', () => {
    // ...
  })
}

module.exports.defaultModes = {
  build: 'production'
}
```

解析命令模式利用 js 内建函数 reduce 实现:

```js
this.modes = this.plugins.reduce((modes, { apply: { defaultModes }}) => {
  return Object.assign(modes, defaultModes)
}, {})
```

#### 3、实例的 run 方法

在 vue-cli-service 的入口处完成 Service 类的实例化后又调用了实例的 run 方法，我们接着查看其具体实现：

```js
// @vue/cli-service/lib/Service.js
module.exports = class Service {
  ...
  async run (name, args = {}, rawArgv = []) {
    // resolve mode
    // prioritize inline --mode
    // fallback to resolved default modes from plugins or development if --watch is defined
    const mode = args.mode || (name === 'build' && args.watch ? 'development' : this.modes[name])

    // --skip-plugins arg may have plugins that should be skipped during init()
    this.setPluginsToSkip(args)

    // load env variables, load user config, apply plugins
    await this.init(mode)

    args._ = args._ || []
    let command = this.commands[name] // 这里的 commands 就是加载插件时通过 api.registerCommand 注册的 command，后续详细介绍
    if (!command && name) {
      error(`command "${name}" does not exist.`)
      process.exit(1)
    }
    if (!command || args.help || args.h) {
      command = this.commands.help
    } else {
      args._.shift() // remove command itself
      rawArgv.shift()
    }
    const { fn } = command
    return fn(args, rawArgv)
  }
  ...
}
```

run 方法开始会获取该命令所对应的模式值，然后调用实例的 init 方法，init 主要有三个功能：

- 加载对应模式下本地的环境变量文件
- 解析 vue.config.js 或者 package.vue
- 执行所有被加载的插件

我们来查看 init 方法的具体实现：

```js
module.exports = class Service {
  ...
  init (mode = process.env.VUE_CLI_MODE) {
    if (this.initialized) {
      return
    }
    this.initialized = true
    this.mode = mode

    // load mode .env
    // 加载指定的模式环境文件
    if (mode) {
      this.loadEnv(mode)
    }
    // load base .env
    // 加载普通环境文件
    this.loadEnv()

    // load user config
    const userOptions = this.loadUserOptions()
    const loadedCallback = (loadedUserOptions) => {
      this.projectOptions = defaultsDeep(loadedUserOptions, defaults())

      debug('vue:project-config')(this.projectOptions)

      // apply plugins.
      this.plugins.forEach(({ id, apply }) => {
        if (this.pluginsToSkip.has(id)) return
        apply(new PluginAPI(id, this), this.projectOptions)
      })

      // apply webpack configs from project config file
      if (this.projectOptions.chainWebpack) {
        this.webpackChainFns.push(this.projectOptions.chainWebpack)
      }
      if (this.projectOptions.configureWebpack) {
        this.webpackRawConfigFns.push(this.projectOptions.configureWebpack)
      }
    }

    if (isPromise(userOptions)) {
      return userOptions.then(loadedCallback)
    } else {
      return loadedCallback(userOptions)
    }
  }
  ...
}
```

init 执行了两次实例的 loadEnv 函数，第一次是加载指定的模式环境文件（.env.development, .env.development.local），第二次执行是加载普通环境文件 (.env, .env.local)，看一下实例 loadEnv 函数代码：

```js
// 加载本地的环境文件，环境文件的作用就是设置某个模式下特有的环境变量
// 加载环境变量其实要注意的就是优先级的问题，下面的代码已经体现的非常明显了，先加载 .env.mode.local，然后加载 .env.mode 最后再加载 .env
// 由于环境变量不会被覆盖，因此 .env.mode.local 的优先级最高，.env.mode.local 与 .env.mode 的区别就是前者会被 git 忽略掉。另外一点要
// 注意的就是环境文件不会覆盖Vue CLI 启动时已经存在的环境变量。
loadEnv (mode) {
  const logger = debug('vue:env')
  // path/.env.production || path/.env.development || ...
  const basePath = path.resolve(this.context, `.env${mode ? `.${mode}` : ``}`)
  const localPath = `${basePath}.local`

  const load = envPath => {
    try {
      const env = dotenv.config({ path: envPath, debug: process.env.DEBUG }) // 加载指定路径的环境变量
      dotenvExpand(env)
      logger(envPath, env)
    } catch (err) {
      // only ignore error if file is not found
      if (err.toString().indexOf('ENOENT') < 0) {
        error(err)
      }
    }
  }

  load(localPath)
  load(basePath)

  // by default, NODE_ENV and BABEL_ENV are set to "development" unless mode
  // is production or test. However the value in .env files will take higher
  // priority.
  if (mode) {
    // always set NODE_ENV during tests
    // as that is necessary for tests to not be affected by each other
    const shouldForceDefaultEnv = (
      process.env.VUE_CLI_TEST &&
      !process.env.VUE_CLI_TEST_TESTING_ENV
    )
    const defaultNodeEnv = (mode === 'production' || mode === 'test')
      ? mode
      : 'development'
    if (shouldForceDefaultEnv || process.env.NODE_ENV == null) {
      process.env.NODE_ENV = defaultNodeEnv
    }
    if (shouldForceDefaultEnv || process.env.BABEL_ENV == null) {
      process.env.BABEL_ENV = defaultNodeEnv
    }
  }
}
```

通过代码我们可以看到 loadEnv 方法的主要作用就是向 node process 中添加环境变量。我们接着查看 loadUserOptions：

```js
// Note: we intentionally make this function synchronous by default
// because eslint-import-resolver-webpack does not support async webpack configs.
loadUserOptions () {
  // fileConfig: import(fileConfigPath)/require(fileConfigPath), fileConfigPath: path.resolve(context, './vue.config.js')
  const { fileConfig, fileConfigPath } = loadFileConfig(this.context)

  if (isPromise(fileConfig)) {
    return fileConfig
      .then(mod => mod.default)
      .then(loadedConfig => resolveUserConfig({
        inlineOptions: this.inlineOptions,
        pkgConfig: this.pkg.vue,
        fileConfig: loadedConfig,
        fileConfigPath
      }))
  }

  return resolveUserConfig({
    inlineOptions: this.inlineOptions,
    pkgConfig: this.pkg.vue, // package.json 里面的 vue config
    fileConfig,
    fileConfigPath
  })
}
```

示例代码中的 loadFileConfig 方法的主要作用是配置文件路径的获取及内容解析，其首先会在遍历 './vue.config.js'、'./vue.config.cjs'、'./vue.config.mjs' 查看当前项目目录下是否存在对应文件，如果存在则判断文件内容是否遵从 ESModule 规范？如果遵从则通过 import 方法获取文件内容，如果不是则通过 require 方法获取。在获取到配置文件内容后紧接着调用了 resolveUserConfig 函数，我们接着查看其实现：

```js
module.exports = function resolveUserConfig ({
  inlineOptions,
  pkgConfig,
  fileConfig,
  fileConfigPath
}) {
  if (fileConfig) {
    if (typeof fileConfig === 'function') {
      fileConfig = fileConfig()
    }

    if (!fileConfig || typeof fileConfig !== 'object') {
      throw new Error(
        `Error loading ${chalk.bold(fileConfigPath)}: ` +
        `should export an object or a function that returns object.`
      )
    }
  }

  // package.vue
  if (pkgConfig && typeof pkgConfig !== 'object') {
    throw new Error(
      `Error loading Vue CLI config in ${chalk.bold(`package.json`)}: ` +
      `the "vue" field should be an object.`
    )
  }

  let resolved, resolvedFrom
  // 既有 vue.config.js 而且在 package.json 里面又包含了 vue 的配置，将会取 vue.config.js 的配置
  if (fileConfig) {
    const configFileName = path.basename(fileConfigPath)
    if (pkgConfig) {
      warn(
        `"vue" field in package.json ignored ` +
        `due to presence of ${chalk.bold(configFileName)}.`
      )
      warn(
        `You should migrate it into ${chalk.bold(configFileName)} ` +
        `and remove it from package.json.`
      )
    }
    resolved = fileConfig
    resolvedFrom = configFileName
  } else if (pkgConfig) {
    resolved = pkgConfig
    resolvedFrom = '"vue" field in package.json'
  } else {
    resolved = inlineOptions || {}
    resolvedFrom = 'inline options'
  }

  // normalize some options
  if (resolved.publicPath !== 'auto') {
    ensureSlash(resolved, 'publicPath')
  }
  if (typeof resolved.publicPath === 'string') {
    resolved.publicPath = resolved.publicPath.replace(/^\.\//, '')
  }
  removeSlash(resolved, 'outputDir')

  // validate options
  validate(resolved, msg => {
    error(`Invalid options in ${chalk.bold(resolvedFrom)}: ${msg}`)
  })

  return resolved
}
```

示例代码首先会加载项目中 vue.config.js，然后会加载 package.json 中的 vue 字段中的配置信息。如果既有 vue.config.js 而且在 package.json 里面又包含了 vue 的配置，将会取 vue.config.js 的配置，如果两者都没有配置信息的话会取 this.inlineOptions || {}， 在获取到配置以后还会进行一些处理和验证，最后返回配置 resolved 。

在通过 loadUserOptions 方法完成了配置文件的解析及获取后紧接着便开始加载插件，核心代码如下：

```js
// apply plugins.
this.plugins.forEach(({ id, apply }) => {
  if (this.pluginsToSkip.has(id)) return
  // service 插件接受两个参数，一个 PluginAPI 实例，一个包含 vue.config.js 内指定的项目本地选项的对象，或者在 package.json 内的 vue 字段。
  apply(new PluginAPI(id, this), this.projectOptions)
})
```

PluginAPI 类的核心代码如下：

```js
// Note: if a plugin-registered command needs to run in a specific default mode,
// the plugin needs to expose it via `module.exports.defaultModes` in the form
// of { [commandName]: mode }. This is because the command mode needs to be
// known and applied before loading user options / applying plugins.

class PluginAPI {
  /**
   * @param {string} id - Id of the plugin.
   * @param {Service} service - A vue-cli-service instance.
   */
  constructor (id, service) {
    this.id = id
    this.service = service
  }

  get version () {
    return require('../package.json').version
  }

  assertVersion (range) {
    if (typeof range === 'number') {
      if (!Number.isInteger(range)) {
        throw new Error('Expected string or integer value.')
      }
      range = `^${range}.0.0-0`
    }
    if (typeof range !== 'string') {
      throw new Error('Expected string or integer value.')
    }

    if (semver.satisfies(this.version, range, { includePrerelease: true })) return

    throw new Error(
      `Require @vue/cli-service "${range}", but was loaded with "${this.version}".`
    )
  }

  /**
   * Current working directory.
   */
  getCwd () {
    return this.service.context
  }

  /**
   * Resolve path for a project.
   *
   * @param {string} _path - Relative path from project root
   * @return {string} The resolved absolute path.
   */
  resolve (_path) {
    return path.resolve(this.service.context, _path)
  }

  /**
   * Check if the project has a given plugin.
   *
   * @param {string} id - Plugin id, can omit the (@vue/|vue-|@scope/vue)-cli-plugin- prefix
   * @return {boolean}
   */
  hasPlugin (id) {
    return this.service.plugins.some(p => matchesPluginId(id, p.id))
  }

  /**
   * Register a command that will become available as `vue-cli-service [name]`.
   *
   * @param {string} name
   * @param {object} [opts]
   *   {
   *     description: string,
   *     usage: string,
   *     options: { [string]: string }
   *   }
   * @param {function} fn
   *   (args: { [string]: string }, rawArgs: string[]) => ?Promise
   */
  registerCommand (name, opts, fn) {
    if (typeof opts === 'function') {
      fn = opts
      opts = null
    }
    this.service.commands[name] = { fn, opts: opts || {} }
  }

  /**
   * Register a function that will receive a chainable webpack config
   * the function is lazy and won't be called until `resolveWebpackConfig` is
   * called
   *
   * @param {function} fn
   */
  chainWebpack (fn) {
    this.service.webpackChainFns.push(fn)
  }

  /**
   * Register
   * - a webpack configuration object that will be merged into the config
   * OR
   * - a function that will receive the raw webpack config.
   *   the function can either mutate the config directly or return an object
   *   that will be merged into the config.
   *
   * @param {object | function} fn
   */
  configureWebpack (fn) {
    this.service.webpackRawConfigFns.push(fn)
  }

  /**
   * Register a dev serve config function. It will receive the express `app`
   * instance of the dev server.
   *
   * @param {function} fn
   */
  configureDevServer (fn) {
    this.service.devServerConfigFns.push(fn)
  }

  /**
   * Resolve the final raw webpack config, that will be passed to webpack.
   *
   * @param {ChainableWebpackConfig} [chainableConfig]
   * @return {object} Raw webpack config.
   */
  resolveWebpackConfig (chainableConfig) {
    return this.service.resolveWebpackConfig(chainableConfig)
  }

  /**
   * Resolve an intermediate chainable webpack config instance, which can be
   * further tweaked before generating the final raw webpack config.
   * You can call this multiple times to generate different branches of the
   * base webpack config.
   * See https://github.com/mozilla-neutrino/webpack-chain
   *
   * @return {ChainableWebpackConfig}
   */
  resolveChainableWebpackConfig () {
    return this.service.resolveChainableWebpackConfig()
  }

  /**
   * Generate a cache identifier from a number of variables
   */
  genCacheConfig (id, partialIdentifier, configFiles = []) {
    const fs = require('fs')
    const cacheDirectory = this.resolve(`node_modules/.cache/${id}`)

    // replace \r\n to \n generate consistent hash
    const fmtFunc = conf => {
      if (typeof conf === 'function') {
        return conf.toString().replace(/\r\n?/g, '\n')
      }
      return conf
    }

    const variables = {
      partialIdentifier,
      'cli-service': require('../package.json').version,
      env: process.env.NODE_ENV,
      test: !!process.env.VUE_CLI_TEST,
      config: [
        fmtFunc(this.service.projectOptions.chainWebpack),
        fmtFunc(this.service.projectOptions.configureWebpack)
      ]
    }

    try {
      variables['cache-loader'] = require('cache-loader/package.json').version
    } catch (e) {
      // cache-loader is only intended to be used for webpack 4
    }

    if (!Array.isArray(configFiles)) {
      configFiles = [configFiles]
    }
    configFiles = configFiles.concat([
      'package-lock.json',
      'yarn.lock',
      'pnpm-lock.yaml'
    ])

    const readConfig = file => {
      const absolutePath = this.resolve(file)
      if (!fs.existsSync(absolutePath)) {
        return
      }

      if (absolutePath.endsWith('.js')) {
        // should evaluate config scripts to reflect environment variable changes
        try {
          return JSON.stringify(require(absolutePath))
        } catch (e) {
          return fs.readFileSync(absolutePath, 'utf-8')
        }
      } else {
        return fs.readFileSync(absolutePath, 'utf-8')
      }
    }

    variables.configFiles = configFiles.map(file => {
      const content = readConfig(file)
      return content && content.replace(/\r\n?/g, '\n')
    })

    const cacheIdentifier = hash(variables)
    return { cacheDirectory, cacheIdentifier }
  }
}

module.exports = PluginAPI
```

简单介绍一些 PluginAPI 常用的方法：

- registerCommand: 注册 cli 命令服务
- chainWebpack: 通过 webpack-chain 修改 webpack 配置
- configureWebpack: 通过 webpack-merge 对 webpack 配置进行合并
- resolveWebpackConfig: 调用之前通过 chainWebpack 和 configureWebpack 上完成的对于 webpack 配置的改造，并返回最终的 webpack 配置
- genCacheConfig: 返回 cacheDirectory, cacheIdentifier
- ...

系统的内置插件 builtInPlugins 例如 './commands/serve'、'./commands/build' 等也是通过调用 PluginAPI 提供的具体方法实现其功能，我们以 serve 命令为例：

```js
// @vue/cli-service/lib/commands/serve.js
module.exports = (api, options) => {
  api.registerCommand('serve', {
    description: 'start development server',
    usage: 'vue-cli-service serve [options] [entry]',
    options: {
      '--open': `open browser on server start`,
      '--copy': `copy url to clipboard on server start`,
      '--mode': `specify env mode (default: development)`,
      '--host': `specify host (default: ${defaults.host})`,
      '--port': `specify port (default: ${defaults.port})`,
      '--https': `use https (default: ${defaults.https})`,
      '--public': `specify the public network URL for the HMR client`
    }
  }, async function serve (args) {
   // resolveWebpackConfig
   // create server
   // ....
  })
}
```

利用 api.registerCommand 注册了 serve 命名，并将 serve 命令的处理函数挂载到 Service 实例的 serve 命令中，当然你还可以通过 module.exports.defaultModes 以 `{ [commandName]: mode }` 的形式来指定命令的运行模式。

分析到这里你应该逐渐熟悉 vue-cli 3.0 的插件机制了，vue-cli 3.0 将所有的工作都交给插件去执行，开发模式执行内置 serve 插件，打包执行内置 build 插件，检查代码规范由 @vue/cli-plugin-eslint 插件完成。

在加载完所有的插件以后，实例的 init 方法在最后会读取项目配置中的 webpack 配置信息，即 chainWebpack 和 configureWebpack，代码如下：

```js
// apply webpack configs from project config file
if (this.projectOptions.chainWebpack) {
  this.webpackChainFns.push(this.projectOptions.chainWebpack)
}
if (this.projectOptions.configureWebpack) {
  this.webpackRawConfigFns.push(this.projectOptions.configureWebpack)
}
```

在实例的 run 函数中执行了实例 init 函数对 Service 实例的属性进行初始化后就会解析 CLI 命令，具体实现如下：

```js
...
args._ = args._ || []
let command = this.commands[name] // 加载插件时注册了 command，api.registerCommand
if (!command && name) { // 非法命令
  error(`command "${name}" does not exist.`)
  process.exit(1)
}
if (!command || args.help) { // vue-cli-service || vue-cli-service -h
  command = this.commands.help
} else {
  args._.shift() // remove command itself
  rawArgv.shift()
}
const { fn } = command
return fn(args, rawArgv)
```

会先对 CLI 命令进行一个判断，主要有一下三种情况：

- 输入了命令 name ，但是并没有通过 api.registerCommand 注册，即非法命令，process.exit(1)
- 直接输入了 vue-cli-service 或者 vue-cli-service --help，加载内置 help 插件
- 正常输入，eg: vue-cli-service test:unit，这种情况会加载对应地单元测试插件 @vue/cli-plugin-unit-jest || @vue/cli-plugin-unit-mocha，并执行插件内与对应 test 命令指定的处理函数。

#### 4、总结

至此 @vue/cli-service 的分析就完成了，@vue/cli-service 的主要作用就是提供了 vue-cli-service 命令，但是与一般 node 命令文件有点不同，@vue/cli-service 并没有直接提供相关 serve、build 等 CLI 命令的服务。具体实现主要是通过 Service 实例的 run 方法，run 方法主要执行了环境变量文件加载，获取项目配置信息，合并项目配置，加载插件，加载项目配置中的 webpack 信息，最后 执行 CLI 服务，常见的 serve 和 build 指令也是通过系统提供的内置插件实现的，在下面我们将详细介绍 Vue CLI 的最后一个核心内容：插件。

### 四、插件

（已经六万字了，，，我当时论文都没这么多字😭）

通过前文对于 @vue/cli 和 @vue/cli-service 的介绍我们也已经初步知道了 Vue CLI 中插件的重要性，一个 CLI 插件是一个 npm 包，它能够为 Vue CLI 创建的项目添加额外的功能，这些功能包括：

- 修改项目的 webpack 配置 - 例如，如果你的插件希望去针对某种类型的文件工作，你可以为这个特定的文件扩展名添加新的 webpack 解析规则。比如说，@vue/cli-plugin-typescript 就添加这样的规则来解析 .ts 和 .tsx 扩展的文件；
- 添加新的 vue-cli-service 命令 - 例如，@vue/cli-plugin-unit-jest 添加了 test:unit 命令，允许开发者运行单元测试；
- 扩展 package.json - 当你的插件添加了一些依赖到项目中，你需要将他们添加到 package 的 dependencies 部分时，这是一个有用的选项；
- 在项目中创建新文件、或者修改老文件。有时创建一个示例组件或者通过给入口文件（main.js）添加导入（imports）是一个好的主意；
- 提示用户选择一个特定的选项 - 例如，你可以询问用户是否创建我们前面提到的示例组件。

CLI 插件应该总是包含一个 service 插件 做为主的导出，并且他能够选择性的包含 generator, prompt 文件 和 Vue UI 集成。

作为一个 npm 包，CLI 插件必须有一个 package.json 文件。通常建议在 README.md 中包含插件的描述，来帮助其他人在 npm 上发现你的插件。

所以，通常的 CLI 插件目录结构看起来像下面这样：

```sh
.
├── README.md
├── generator.js  # generator（可选）
├── index.js      # service 插件
├── package.json
├── prompts.js    # prompt 文件（可选）
└── ui.js         # Vue UI 集成（可选）
```

#### 1、package.json

为了让一个 CLI 插件在 Vue CLI 项目中被正常使用，它必须遵循 vue-cli-plugin-<name> 或者 @scope/vue-cli-plugin-<name> 这样的命名惯例。这样你的插件才能够：

- 被 @vue/cli-service 发现;
- 被其他开发者通过搜索发现;
- 通过 `vue add <name>` 或者 `vue invoke <name>` 安装;

为了能够被用户在搜索时更好的发现，keywords 尽可能多写一些项目相关的词，同时可以将插件的关键描述放到 description 字段中。

例如：

```json
{
  "name": "vue-cli-plugin-apollo",
  "version": "0.7.7",
  "description": "vue-cli plugin to add Apollo and GraphQL"
}
```

你应该在 homepage 或者 repository 字段添加创建插件的官网地址或者仓库的地址，这样你的插件详情里就会出现一个 查看详情 按钮：

```json
{
  "repository": {
    "type": "git",
    "url": "git+https://github.com/Akryum/vue-cli-plugin-apollo.git"
  },
  "homepage": "https://github.com/Akryum/vue-cli-plugin-apollo#readme"
}
```

#### 2、generator

插件的 Generator 部分通常在你想要为项目扩展包依赖，创建新的文件或者编辑已经存在的文件时需要。在 CLI 插件内部，generator 应该放在 generator.js 或者 generator/index.js 文件中。它将在以下两个场景被调用：

- 项目初始创建期间，CLI 插件被作为项目创建 preset 的一部分被安装时。
- 当插件在项目创建完成和通过 vue add 或者 vue invoke 单独调用被安装时。

一个 generator 应该导出一个接收三个参数的函数：

1、一个 GeneratorAPI 实例；

2、插件的 generator 选项。这些选项在项目创建，或者从 ~/.vuerc 载入预设时被解析。例如：如果保存的 ~/.vuerc 像这样：

```json
{
  "presets" : {
    "foo": {
      "plugins": {
        "@vue/cli-plugin-foo": { "option": "bar" }
      }
    }
  }
}
```

如果用户使用 preset foo 创建了一个项目，那么 @vue/cli-plugin-foo 的 generator 就会收到 { option: 'bar' } 作为第二个参数。

对于第三方插件，这个选项将在用户执行 vue invoke 时，从提示或者命令行参数中被解析。

3、整个 preset (presets.foo) 将会作为第三个参数传入。

关于 GeneratorAPI 我们已经在前面解析 @vue/cli 的过程中进行了详细介绍，那么我们通过 GeneratorAPI 可以实现哪些功能呢？

**1、创建新的模板**

当你调用 api.render('./template') 时，该 generator 将会使用 EJS 渲染 ./template 中的文件 (相对于 generator 中的文件路径进行解析)

想象我们正在创建 vue-cli-auto-routing 插件，我们希望当插件在项目中被引用时做以下的改变：

- 创建一个 layouts 文件夹包含默认布局文件；
- 创建一个 pages 文件夹包含 about 和 home 页面；
- 在 src 文件夹中添加 router.js 文件

为了渲染这个结构，你需要在 generator/template 文件夹内创建它：

```sh
.
├── src
 ├── layouts
  ├── default.vue
 ├── pages
  ├── about.vue
  ├── home.vue
 ├── router
  └── index.js
```

模板创建完之后，我们就可以在 generator/index.js 文件中添加 api.render('./template') 来调用。

不过需要注意文件名的边界情况，如果你想要渲染一个以点开头的模板文件 (例如 .env)，则需要遵循一个特殊的命名约定，因为以点开头的文件会在插件发布到 npm 的时候被忽略：

```sh
# 以点开头的模板需要使用下划线取代那个点：

/generator/template/_env

# 当调用 api.render('./template') 时，它在项目文件夹中将被渲染为：

/generator/template/.env
```

同时这也意味着当你想渲染以下划线开头的文件时，同样需要遵循一个特殊的命名约定：

```sh
# 这种模板需要使用两个下划线来取代单个下划线：

/generator/template/__variables.scss

# 当调用 api.render('./template') 时，它在项目文件夹中将被渲染为：

/generator/template/_variable.scss
```

**2、编辑已经存在的模板**

此外，你可以使用 YAML 前置元信息继承并替换已有的模板文件的一部分（即使来自另一个包）：

```yaml
---
extend: '@vue/cli-service/generator/template/src/App.vue'
replace: !!js/regexp /<script>[^]*?<\/script>/
---

<script>
export default {
  // 替换默认脚本
}
</script>
```

也可以替换多处，只不过你需要将替换的字符串包裹在 <%# REPLACE %> 和 <%# END_REPLACE %> 块中：

```yaml
---
extend: '@vue/cli-service/generator/template/src/App.vue'
replace:
  - !!js/regexp /Welcome to Your Vue\.js App/
  - !!js/regexp /<script>[^]*?<\/script>
---

<%# REPLACE %>
替换欢迎信息
<%# END_REPLACE %>

<%# REPLACE %>
<script>
export default {
  // 替换默认脚本
}
</script>
<%# END_REPLACE %>
```

**3、扩展包**

如果你需要向项目中添加额外的依赖，创建一个 npm 脚本或者修改 package.json 的其他任何一处，你可以使用 API extendPackage 方法。

```js
// generator/index.js

module.exports = api => {
  api.extendPackage({
    dependencies: {
      'vue-router-layout': '^0.1.2'
    }
  })
}
```

在上面这个例子中，我们添加了一个依赖：vue-router-layout。在插件调用时，这个 npm 模块将被安装，这个依赖将被添加到用户项目的 package.json 文件。

同样使用这个 API 我们可以添加新的 npm 任务到项目中。为了实现这个，我们需要定义一个任务名和一个命令，这样他才能够在用户 package.json 文件的 scripts 部分运行：

```js
// generator/index.js

module.exports = api => {
  api.extendPackage({
    scripts: {
      greet: 'vue-cli-service greet'
    }
  })
}
```

在上面这个例子中，我们添加了一个新的 greet 任务来执行一个创建在 Service 部分 的自定义 vue-cli 服务命令。

**4、修改主文件**

通过 generator 方法你能够修改项目中的文件。最有用的场景是针对 main.js 或 main.ts 文件的一些修改：新的导入，新的 Vue.use() 调用等。

让我们来思考一个场景，当我们通过 模板 创建了一个 router.js 文件，现在我们希望导入这个路由到主文件中。我们将用到两个 generator API 方法： entryFile 将返回项目的主文件（main.js 或 main.ts），injectImports 用于添加新的导入到主文件中：

```js
// generator/index.js

api.injectImports(api.entryFile, `import router from './router'`)
```

现在，当我们路由被导入时，我们可以在主文件中将这个路由注入到 Vue 实例。我们可以使用 afterInvoke 钩子，这个钩子将在文件被写入硬盘之后被调用。

首先，我们需要通过 Node 的 fs 模块（提供了文件交互 API）读取文件内容，将内容拆分

```js
// generator/index.js

module.exports.hooks = (api) => {
  api.afterInvoke(() => {
    const fs = require('fs')
    const contentMain = fs.readFileSync(api.resolve(api.entryFile), { encoding: 'utf-8' })
    const lines = contentMain.split(/\r?\n/g)
  })
}
```

然后我们需要找到包含 render 单词的字符串（它通常是 Vue 实例的一部分），router 就是下一个字符串：

```js
// generator/index.js

module.exports.hooks = (api) => {
  api.afterInvoke(() => {
    const fs = require('fs')
    const contentMain = fs.readFileSync(api.resolve(api.entryFile), { encoding: 'utf-8' })
    const lines = contentMain.split(/\r?\n/g)

    const renderIndex = lines.findIndex(line => line.match(/render/))
    lines[renderIndex] += `\n router,`
  })
}
```

最后，你需要将内容写入主文件：

```js
// generator/index.js

module.exports.hooks = (api) => {
  api.afterInvoke(() => {
    const { EOL } = require('os')
    const fs = require('fs')
    const contentMain = fs.readFileSync(api.resolve(api.entryFile), { encoding: 'utf-8' })
    const lines = contentMain.split(/\r?\n/g)

    const renderIndex = lines.findIndex(line => line.match(/render/))
    lines[renderIndex] += `${EOL}  router,`

    fs.writeFileSync(api.entryFile, lines.join(EOL), { encoding: 'utf-8' })
  })
}
```

#### 3、service 插件

Service 插件可以修改 webpack 配置，创建新的 vue-cli service 命令或者修改已经存在的命令（如 serve 和 build）。

Service 插件在 Service 实例被创建后自动加载 - 例如，每次 vue-cli-service 命令在项目中被调用的时候。它位于 CLI 插件根目录的 index.js 文件。

一个 service 插件应该导出一个函数，这个函数接受两个参数：

- 一个 PluginAPI 实例
- 一个包含 vue.config.js 内指定的项目本地选项的对象，或者在 package.json 内的 vue 字段。

PluginAPI 为 service 插件提供了针对不同的环境扩展/修改内部的 webpack 配置的能力。例如，这里我们在 webpack-chain 中添加 vue-auto-routing 这个 webpack 插件，并指定参数：

```js
const VueAutoRoutingPlugin = require('vue-auto-routing/lib/webpack-plugin')

module.exports = (api, options) => {
  api.chainWebpack(webpackConfig => {
    webpackConfig
    .plugin('vue-auto-routing')
      .use(VueAutoRoutingPlugin, [
        {
          pages: 'src/pages',
          nested: true
        }
      ])
  })
}
```

你也可以使用 configureWebpack 方法修改 webpack 配置或者返回一个对象，返回的对象将通过 webpack-merge 被合并到配置中。

除了修改 webpack 配置之外我们还可以通过 Service 插件实现如下功能：

**1、添加一个新的 cli-service 命令**

通过 service 插件你可以注册一个新的 cli-service 命令，除了标准的命令（即 serve 和 build）。你可以使用 registerCommand API 方法实现。

下面的例子创建了一个简单的新命令，可以向开发控制台输出一条问候语：

```js
api.registerCommand(
  'greet',
  {
    description: 'Write a greeting to the console',
    usage: 'vue-cli-service greet'
  },
  () => {
    console.log(`👋  Hello`)
  }
)
```

在这个例子中，我们提供了命令的名字（'greet'）、一个有 description 和 usage 选项的对象，和一个在执行 vue-cli-service greet 命令时会调用的函数。

如果你在已经安装了插件的项目中运行新命令，你将看到下面的输出：

```sh
$ vue-cli-service greet
👋 Hello!
```

你也可以给新命令定义一系列可能的选项。接下来我们添加一个 --name 选项，并修改实现函数，当提供了 name 参数时把它也打印出来。

```js
api.registerCommand(
  'greet',
  {
    description: 'Writes a greeting to the console',
    usage: 'vue-cli-service greet [options]',
    options: { '--name': 'specifies a name for greeting' }
  },
  args => {
    if (args.name) {
      console.log(`👋 Hello, ${args.name}!`);
    } else {
      console.log(`👋 Hello!`);
    }
  }
)
```

现在，如果 greet 命令携带了特定的 --name 选项，这个 name 被添加到控制台输出：

```sh
$ vue-cli-service greet --name 'John Doe'
👋 Hello, John Doe!
```

**2、修改已经存在的 cli-service 命令**

如果你想修改一个已经存在的 cli-service 命令，你可以使用 api.service.commands 获取到命令对象并且做些改变。我们将在应用程序运行的端口打印一条信息到控制台：

```js
const { serve } = api.service.commands

const serveFn = serve.fn

serve.fn = (...args) => {
  return serveFn(...args).then(res => {
    if(res && res.url) {
      console.log(`Project is running now at ${res.url}`)
    }
  })
}
```

在上面的这个例子中，我们从已经存在的命令列表中获取到命令对象 serve；然后我们修改了他的 fn 部分（fn 是创建这个新命令时传入的第三个参数；它定义了在执行这个命令时要执行的函数）。修改完后，这个控制台消息将在 serve 命令成功运行后打印。

**3、为命令指定模式**

如果一个已注册的插件命令需要运行在特定的默认模式下，则该插件需要通过 module.exports.defaultModes 以 { [commandName]: mode } 的形式来暴露：

```js
module.exports = api => {
  api.registerCommand('build', () => {
    // ...
  })
}

module.exports.defaultModes = {
  build: 'production'
}
```

这是因为我们需要在加载环境变量之前知道该命令的预期模式，所以需要提前加载用户选项/应用插件。

#### 4、prompt 文件

对话是在创建一个新的项目或者在已有项目中添加新的插件时处理用户选项时需要的。所有的对话逻辑都存储在 prompts.js 文件中。对话内部是通过 inquirer 实现。

当用户通过调用 vue invoke 初始化插件时，如果插件根目录包含 prompts.js，它将在调用时被使用。这个文件应该导出一个问题数组 -- 将被 Inquirer.js 处理。

```js
// prompts.js 
// 直接返回问题数组
module.exports = [
  {
    type: 'input',
    name: 'locale',
    message: 'The locale of project localization.',
    validate: input => !!input,
    default: 'en'
  }
  // ...
]

// 返回问题数组的函数
module.exports = pkg => {
  const prompts = [
    {
      type: 'input',
      name: 'locale',
      message: 'The locale of project localization.',
      validate: input => !!input,
      default: 'en'
    }
  ]

  // 添加动态对话
  if ('@vue/cli-plugin-eslint' in (pkg.devDependencies || {})) {
    prompts.push({
      type: 'confirm',
      name: 'useESLintPluginVueI18n',
      message: 'Use ESLint plugin for Vue I18n ?'
    })
  }

  return prompts
}
```

解析到的答案对象将作为选项传入到插件的 generator。如果我们想在 generator 中使用用户的选择结果，可以通过对话名字获得。例如我们可以修改一下 generator/index.js：

```js
if (options.useESLintPluginVueI18n) {
  ...
}
```

#### 5、Vue UI 集成

Vue CLI 有一个非常强大的 UI 工具 -- 允许用户通过图形接口来架构和管理项目。Vue CLI 插件能够集成到接口中。UI 为 CLI 插件提供了额外的功能：

- 可以执行 npm 任务，直接在 UI 中执行插件中定义的命令；
- 可以展示插件的自定义配置。
- 当创建项目时，你可以展示对话
- 如果你想支持多种语言，你可以为你的插件添加本地化
- 你可以使插件在 Vue UI 搜索中被搜索到

图形化配置及管理也是 Vue CLI 最新架构的一个重要亮点，个人感觉实际插件开发比较少用，所以此处不再展开，详细配置方法请查看[官方文档](https://cli.vuejs.org/zh/dev-guide/plugin-dev.html#ui-%E9%9B%86%E6%88%90)

至此对于 Vue CLI 的架构的主要分析全部完成，这种基于插件机制的架构为开发者提供了终端命令行工具、零配置脚手架、插件体系、图形化管理界面等诸多功能，相较于早前的  vue-cli 其可配置性强，极为灵活并且版本依赖易于管理与升级，就很牛皮！

<img src="/assets/emoji/05.jpg" width="280" />

## 参考资料

[vue-cli-analysis](https://kuangpf.com/vue-cli-analysis)

[剖析 Vue CLI 实现原理](https://cloud.tencent.com/developer/article/1781202)

[Vue CLI官方文档](https://cli.vuejs.org/)
