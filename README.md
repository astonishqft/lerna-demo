# lerna 管理你的应用

## multirepo VS monorepo

在介绍我们今天的主角 [lerna](https://www.npmjs.com/package/lerna) 之前，首先了解下什么是 `multirepo` ？什么是 `monorepo` ？

`multirepo` 指的是将模块分为多个仓库，`monorepo` 指的是将多个模块放在一个仓库中。

`multirepo` 可以让每个团队都拥有自己的仓库，他们可以使用自己的构建流程、代码规范等，但是同时也会存在很多问题，比如模块之间如果存在相互依赖，就必须到目标仓库里进行bug修复、构建、发版本等，相互依赖关系越复杂，处理起来就越困难。

`monorepo` 可以让多个模块共享同一个仓库，因此他们可以共享同一套构建流程、代码规范也可以做到统一，特别是如果存在模块间的相互依赖的情况，查看代码、修改bug、调试等会更加方便，因此也越来越受到大家的关注，像 `Babel`、`React`、`Vue` 等主流的开源仓库都采用的 `monorepo`。

## lerna

**lerna 是一个管理工具，用于管理包含多个软件包（package）的 JavaScript 项目**，最早是 `Babel` 自己用来维护自己的 `monorepo` 并开源出的一个项目，针对使用 `git` 和 `npm` 管理多软件包代码仓库的工作流程进行优化，解决多个包互相依赖，且发布需要手动维护多个包的问题。

总结一下，使用 `lerna` 可以帮我们解决如下几个痛点:

**多个仓库之间可以管理管理公共的依赖包，或者单独管理各自的依赖包**

**方便模块之间的相互引用，模块之间的调试不必发版本，lerna内部会自动进行link**

**lerna提供了两种模式，支持选择单独针对某个包发版本或者统一发版本**

**多个仓库之间可以共享统一的代码规范，版本管理更加规范**

以下我会分两个部分介绍下 `lerna`，首先是介绍 `lerna` 的常规用法，然后介绍下 `lerna` 的最佳实践。

## 基本用法

### 安装

```bash
$ npm install --global lerna
```

### 创建一个git仓库

```bash
$ git init lerna-repo && cd lerna-repo
```

### 初始化一个 lerna 仓库

```bash
$ lerna init
```

注意，`lerna init` 可以通过参数 `--independent` 进入 `Independent` 模式，该模式可以单独发版本。

初始化之后的工程目录结构如下：

```
lerna-repo
    ├── lerna.json
    ├── package.json
    └── packages
```

lerna.json:

```json
{
  "packages": [
    "packages/*"
  ],
  "version": "0.0.0"
}
```

package.json:

```json
{
  "name": "root",
  "private": true,
  "devDependencies": {
    "lerna": "^4.0.0"
  }
}
```

### 新增package包

使用 lerna create 创建两个包 pkg1 和 pkg2

```bash
$ lerna create pkg1
$ lerna create pkg2
```

创建完成后的目录结构如下:
```
lerna-demo
    ├── README.md
    ├── lerna.json
    ├── package.json
    └── packages
        ├── pkg1
        │   ├── README.md
        │   ├── __tests__
        │   ├── lib
        │   └── package.json
        └── pkg2
            ├── README.md
            ├── __tests__
            ├── lib
            └── package.json
```

### 给两个package增加公共依赖

给 `pkg1` 和 `pkg2` 这两个包都安装 `fs-extra` 这个包，`pkg1` 和 `pkg2` 的 `package.json` 的 `dependency` 会同时包含 `fs-extra` 这个包。

```bash
$ lerna add fs-extra
```

安装 `fs-extra` 之后的目录结构:

```
lerna-demo
    ├── README.md
    ├── lerna.json
    ├── package.json
    └── packages
        ├── pkg1
        │   ├── README.md
        │   ├── __tests__
        │   ├── lib
        │   ├── node_modules
        │   ├── package-lock.json
        │   └── package.json
        └── pkg2
            ├── README.md
            ├── __tests__
            ├── lib
            ├── node_modules
            ├── package-lock.json
            └── package.json
```

### 给某个包单独安装指定依赖

比如给 `pkg1` 安装一个 `glob` 包，给 `pkg2` 安装一个 `ora` 包:

```bash
$ lerna add glob --scope pkg1
$ lerna add ora --scope pkg2
```

其中 `--scope` 参数用来指定具体给哪个 `package` 安装包，注意 `--scope` 后面的参数是对应模块的 `package.json` 中的 `name` 字段名。

### 添加内部模块之间的依赖

将 `pkg1` 作为 `pkg2` 的依赖进行安装：

```bash
$ lerna add pkg1 --scope pkg2
```

需要注意的是，通过这种方式安装的依赖，并不会将 `pkg1` 安装到 `pkg2` 的 `node_modules` 里，而是通过 `symlink` 的形式进行关联。

### 发布

以上包确认没有问题之后，就可以通过执行 `lerna publish` 进行发布了。

在进行 `publish` 之前需要首先提交你的代码，否则 `lerna` 会报错：

```bash
lerna ERR! ENOCOMMIT No commits in this repository. Please commit something before using version.
```

提交代码并关联到 `git` 仓库：

```bash
$ git add .
$ git commit -m 'init'
$ git remote add origin git@github.com:astonishqft/lerna-demo.git // 关联到远程git仓库
$ git push -u origin main
```

### 删除某个包

将 `pkg1` 里面的 `glob` 包删除：

```bash
$ lerna exec --scope=pkg1 npm uninstall glob
```

### 抽离公共的包

上面可以看到，`pkg1` 和 `pkg2` 都依赖了 `fs-extra` 这个包，而各自 `package` 下面的 `node_modules` 都进行了一次安装，因此我们可以通过 `--hoist` 来抽取重复的依赖到最外层的 `node_modules` 目录下，同时最外层的 `package.josn` 的依赖信息也不会进行更新。

```bash
$ lerna bootstrap --hoist
```

## 最佳实践

前面我们已经介绍了 `lerna` 的相关概念和基本用法，目前最常见的解决方案是基于 `lerna` 和 `yarn workspace` 的 `monorepo` 工作流。

由于 `yarn` 和 `lerna` 在功能上有较多的重叠，我们采用 `yarn` 官方推荐的做法: 

用 `yarn` 来处理依赖问题，用 `lerna` 来处理发布问题。

### yarn workspaces 与 lerna

`yarn workspaces` 是 `yarn` 提供的 `monorepo` 的依赖管理机制，用于在代码仓库的根目录下管理多个 `package` 依赖，与 `lerna` 不同的是，`yarn workspaces` 会将所有的依赖尽量安装到根目录的 `node_modules` 中，只有在各个 `package` 依赖了不同版本的同一个依赖或者存在自己的特有依赖时才会独自安装依赖，而 `lerna` 会进到每个 `package` 中执行 `yarn/npm install`，因此会在每个 `package` 下生成一个 `node_modules`。

`yarn workspaces` 首先在工程的根目录下的 `package.json` 中增加 `"private": true` 和 `"workspaces”: [ "packages/*"]` 配置项。`"private": true` 可以确保根目录不会被发布出去，`"workspaces”: [ "packages/*"]` 声明了 `workspaces` 中所包含的项目路径。

`package.json` 配置文件增加如下配置。

### 开启yarn workspaces

```json
{
  "private": true,
  "workspaces": [
    "packages/*"
  ],
}
```

`lerna.json` 配置文件增加如下配置。
```json
{
  "useWorkspaces": true,
  "npmClient": "yarn",
}
```

### 工程初始化

对于一个已经存在的 `monorepo` 仓库，使用 `yarn install` 安装依赖。`yarn install` 会自动安装依赖并且解决同一个仓库之间多个`package` 之间的 `link` 问题。

`yarn install` 等价于 `lerna bootstrap --npm-client yarn --use-workspace`。

### 清理环境

使用 `lerna clean` 可以清理每个 `package` 下的 `node_modules`，但是没有办法清理根目录下的 `node_modules` 目录，因此,我们可以在根目录下的 `package.json` 目录的 `scripts` 中增加一条 `clean` 命令，用于清理环境。

package.json:

```json
{
  "clear-all": "rimraf node_modules && lerna clean -y"
}
```

### 安装依赖

安装依赖一般分为三种情况：

- 安装到workspace-root

  对于一些打包工具或者代码规范校验工具，可以使用 `yarn -W add [package] [--dev]` 进行安装，比如 typescript、eslint、cross-env、babel、rollup等。这类包一般都是一些开发依赖，比如将 `ts` 代码转换成 `es5` 代码或者一些代码校验工具等。通过这种方式安装的依赖包是装在根目录下的 `node_modules` 中。

- 给所有的 `package` 都安装依赖
  
  比如如果想给每个 `package` 都安装一个 `lodash` 包，就可以使用 `yarn workspace add lodash` 给每个 `package` 都安装 `lodash`。

- 给指定的某个 package 安装依赖

  通过 `yarn workspace pkgA add pkgB` 可以将 `pkgB` 作为依赖安装到 `pkgA` 中，需要注意的是，如果是 packages 之间的相互安装，安装的时候可以指定到具体的版本号，否则安装的时候回去npm上搜索，但是因为某个包还没有发包出去，导致安装失败。

删除包的时候只需要把上述 `add` 换成 `remove` 即可。

### 运行workspace的command

通过运行 `yarn workspace <workspace_name> <command>` 命令运行某个执行 `package` 下的某个 `script` 命令。

比如执行 `pkgA` 下的 `build` 命令，可以运行 `yarn workspace pkgA run build`。如果想运行所以 `package` 下的 `build` 命令，可以运行 `yarn workspaces run build`。

### 代码提交

代码编写完毕后接下来就涉及到代码的提交，为了规范代码提交格式，方便自动生成 changelog，这里需要借助一下几个工具。

#### commitizen && cz-conventional-changelog

[commitizen](https://www.npmjs.com/package/commitizen) 的作用主要是为了生成标准化的 `commit message`，符合 [Angular规范](https://github.com/angular/angular.js/blob/master/DEVELOPERS.md#-git-commit-guidelines)。

一个标准化的 `commit message` 应该包含三个部分：`Header`、`Body` 和 `Footer`，其中的 `Header` 是必须的，`Body` 和 `Footer` 可以选填。

```
<type>(<scope>): <subject>
// 空一行
<body>
// 空一行
<footer>
```
`Header` 部分由三个字段组成：`type`（必需）、`scope`（可选）、`subject`（必需）

**Type**

`type` 必须是下面的其中之一：

- feat: 增加新功能
- fix: 修复 bug
- docs: 只改动了文档相关的内容
- style: 不影响代码含义的改动，例如去掉空格、改变缩进、增删分号
- refactor: 代码重构时使用，既不是新增功能也不是代码的bud修复
- perf: 提高性能的修改
- test: 添加或修改测试代码
- build: 构建工具或者外部依赖包的修改，比如更新依赖包的版本
- ci: 持续集成的配置文件或者脚本的修改
- chore: 杂项，其他不需要修改源代码或不需要修改测试代码的修改
- revert: 撤销某次提交

**scope**

用于说明本次提交的影响范围。`scope` 依据项目而定，例如在业务项目中可以依据菜单或者功能模块划分，如果是组件库开发，则可以依据组件划分。

**subject**

主题包含对更改的简洁描述：

注意三点：

1. 使用祈使语气，现在时，比如使用 "change" 而不是 "changed" 或者 ”changes“
2. 第一个字母不要大写
3. 末尾不要以.结尾

**Body**

主要包含对主题的进一步描述，同样的，应该使用祈使语气，包含本次修改的动机并将其与之前的行为进行对比。

**Footer**

包含此次提交有关重大更改的信息，引用此次提交关闭的issue地址，如果代码的提交是不兼容变更或关闭缺陷，则Footer必需，否则可以省略。

使用方法：

安装 `commitizen`，如果需要在项目中使用 `commitizen` 生成符合 `AngularJS` 规范的提交说明，还需要安装 `cz-conventional-changelog` 适配器：

```bash
$ yarn -W add commitizen cz-conventional-changelog -D
```

`package.json` 中增加一条 script: `commit: "git-cz"`，并且在 `config` 字段中指定 `cz-conventional-changelog` 路径：

```json
{
  "name": "root",
  "private": true,
  "workspaces": [
    "packages/*"
  ],
  "scripts": {
    "commit": "git-cz"
  },
  "config": {
    "commitizen": {
      "path": "cz-conventional-changelog"
    }
  },
  "devDependencies": {
    "commitizen": "^4.2.4",
    "cz-conventional-changelog": "^3.3.0",
    "lerna": "^4.0.0"
  }
}
```

接下来就可以使用 `yarn commit` 来代替 `git commit` 进行代码提交了。

![image](https://user-images.githubusercontent.com/15138753/154808442-93a67399-d759-4cec-a8a7-d17081575f87.png)


#### commitlint && husky

前面我们提到，通过 `commitizen` && `z-conventional-changelog` 可以规范我们的 `commit message`，但是同时也存在一个问题，如果用户不通过 `yarn commit` 来提交代码，而是直接通过 `git commit` 命令来提交代码，就能绕开 `commit message` 检查，这是我们不希望看到的。

因此接下来我们使用 `commitlint` 结合 `husky` 来对我们的提交行为进行约束。在 `git commit` 提交之前使用 `git` 钩子来验证信息。提交不符合规则的信息将会被阻止提交。

安装 `commitlint` 和 `husky`：

```bash
$ yarn -W add @commitlint/cli @commitlint/config-conventional husky -D
```

在工程根目录下增加 `commitlint.config.js` 配置文件，指定 `commitlint` 的校验配置文件：

**commitlint.config.js**:

已经废弃的配置：
```js
module.exports = { 
	extends: ['@commitlint/config-conventional'] 
}
```

需要注意的是，如果安装的 `husky` 的版本是 `7.x` 的，以往直接在 `package.json` 中 `hooks` 字段增加的配置项已经被废弃了，具体可以参考[这里](https://cloud.tencent.com/developer/article/1876208)

```json
"husky": { 
  "hooks": { 
    "commit-msg": "commitlint -E HUSKY_GIT_PARAMS" 
  }
},
```

下面在 `husky 7.x` 版本下进行 `commit-msg` 的配置。

1. 执行 `npm husky install`

或者如果想在安装后自动启动 husky，在 package.json 的scripts中增加一条script命令：

```json
"scripts": {
  "prepare": "husky install"
},
```

`prepare` 是 `NPM` 操作生命周期中的一环，在执行 `install` 的时候会按生命周期顺序执行相应钩子：`preinstall -> install -> postinstall -> prepublish -> preprepare -> prepare -> postprepare`

执行完毕后就会在根目录下创建一个 `.husky` 目录。

2. 通过 `husky add <file> [cmd]` 指令来添加一条 `hook`

执行 `npx husky add .husky/commit-msg 'npx --no-install commitlint --edit "$1"'` 会在 `.husky` 下生成一个 `commit-msg` 的 `shell` 文件：

```bash
#!/bin/sh
. "$(dirname "$0")/_/husky.sh"

npx --no-install commitlint --edit "$1"
```

这样，当我们在执行 `git commit -m 'xxx'` 的时候，如果提交的 `commit message` 不符合规范就会报如下的错误，只有提交信息符合提交规范才会允许代码提交。

![image](https://user-images.githubusercontent.com/15138753/154849774-0627270a-86d9-4938-8782-bf5eb71a36c8.png)

### eslint 配置

配置 `eslint` 对代码进行统一的规范校验，配合 `lint-staged` 可以对已经提交的代码进行校验。

安装 `eslint` 和 `lint-stage`:

```bash
$ yarn -W add eslint lint-staged @typescript-eslint/parser @typescript-eslint/eslint-plugin -D
```

增加 `.eslintrc.js` 配置文件:

```
module.exports = {
  'parser': '@typescript-eslint/parser',
  'plugins': ['@typescript-eslint'],
  'rules': {
    'no-var': 'error',// 不能使用var声明变量
    'no-extra-semi': 'error',
    '@typescript-eslint/indent': ['error', 2],
    'import/extensions': 'off',
    'linebreak-style': [0, 'error', 'windows'],
    'indent': ['error', 2, { SwitchCase: 1 }], // error类型，缩进2个空格
    'space-before-function-paren': 0, // 在函数左括号的前面是否有空格
    'eol-last': 0, // 不检测新文件末尾是否有空行
    'semi': ['error', 'always'], // 在语句后面加分号
    'quotes': ['error', 'single'],// 字符串使用单双引号,double,single
    'no-console': ['error', { allow: ['log', 'warn'] }],// 允许使用console.log()
    'arrow-parens': 0,
    'no-new': 0,//允许使用 new 关键字
    'comma-dangle': [2, 'never'], // 数组和对象键值对最后一个逗号， never参数：不能带末尾的逗号, always参数：必须带末尾的逗号，always-multiline多行模式必须带逗号，单行模式不能带逗号
    'no-undef': 0
  },
  'parserOptions': {
    'ecmaVersion': 6,
    'sourceType': 'module',
    'ecmaFeatures': {
      'modules': true
    }
  }
};
```

`lint-staged staged` 是 `Git` 里的概念，表示暂存区，`lint-staged` 表示只检查暂存区中的文件。

`package.json` 中增加如下配置：

```json
"lint-staged": {
    "*.ts": [
      "eslint --fix",
      "git add"
    ]
}
```

`husky` 中增加 `pre-commit` 校验：

```bash
$ npx husky add .husky/pre-commit "npx --no-install lint-staged"
```

### 自动生成提交日志

上面这些操作结束之后，就可以方便的对版本进行管理，比如生成 changelog 或者 发布到 npm 等。

#### lerna puplish

- lerna publish 的时候会做一下操作：

- 找出从上一个版本发布以来有过变更的 package

- 提示开发者确定要发布的版本号

- 将所有更新过的的 package 中的package.json的version字段更新

- 将依赖更新过的 package 的 包中的依赖版本号更新

- 更新 lerna.json 中的 version 字段

- 提交上述修改，并打一个 tag

- 推送到 git 仓库

使用 lerna publish 来发布我们的包，只要我们按规范提交后，在 `lerna version` 的过程中会便会自动生成当前这个版本的 CHANGELOG。为了方便，不用每次输入参数，可以配置在 `lerna.json` 中，并且有的文件的变更我们可能不希望版本的更新，比如 md 文档，可以在 `ignoreChanges` 配置项中进行忽略。

在 `lerna.json` 增加如下配置：

```json
{
  "command": {
    "version": {
      "conventionalCommits": true
    }
  },
  "ignoreChanges": [
    "**/*.md"
  ]
}
```
