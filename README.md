# lerna 管理你的应用


## multirepo VS monorepo

在介绍我们今天的主角 lerna 之前，首先了解下什么是multirepo？什么是monorepo？

multirepo指的是将模块分为多个仓库，monorepo指的是将多个模块放在一个仓库中。

multirepo可以让每个团队都拥有自己的仓库，他们可以使用自己的构建流程、代码规范等，但是同时也会存在很多问题，比如模块之间如果存在相互依赖，就必须到目标仓库里面进行bug修复、构建、发版本等，相互依赖关系越复杂，处理起来就越困难。

monorepo可以让多个模块共享同一个仓库，因此他们可以共享同一套构建流程、代码规范也可以做到统一，特别是如果存在模块间的相互依赖的情况，查看代码、修改bug、调试等会更加方便，因此也越来越受到大家的关注，像 Babel、React、Vue等主流的开源仓库都采用的monorepo。

## lerna

**lerna 是一个管理工具，用于管理包含多个软件包（package）的 JavaScript 项目**，最早是 Babel 自己用来维护自己的 monorepo 并开源出的一个项目，针对使用 git 和 npm 管理多软件包代码仓库的工作流程进行优化，解决多个包互相依赖，且发布需要手动维护多个包的问题。

总结一下，使用lerna可以帮我们解决如下几个痛点:

**多个仓库之间可以管理管理公共的依赖包，或者单独管理各自的依赖包**

**方便模块之间的相互引用，模块之间的调试不必发版本，lerna内部会自动进行link**

**lerna提供了两种模式，支持选择单独针对某个包发版本或者统一发版本**

**多个仓库之间可以共享统一的代码规范，版本管理更加规范**


以下我会分两个部分介绍下 lerna，一个是介绍lerna的常规用法，然后介绍下lerna的最佳实践。

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

可以通过参数 `--independent` 使用 Independent 模式，该模式可以单独发版本。

初始化之后的工程目录结构如下：
```
lerna-repo
    ├── lerna.json
    ├── package.json
    └── packages
```

lerna.json
```json
{
  "packages": [
    "packages/*"
  ],
  "version": "0.0.0"
}
```
package.json
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

给pkg1和pkg2这两个包都安装 fs-extra 这个包，pkg1 和 pkg2的package.json的dependency会同时包含fs-extra这个包。

```bash
$ lerna add fs-extra
```

安装 fs-extra 之后的目录结构:
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

比如给pkg1安装一个glob包，给pkg2安装一个ora包:

```bash
$ lerna add glob --scope pkg1
$ lerna add ora --scope pkg2
```

其中 `--scope` 参数用来指定具体给哪个package安装包，注意 `--scope` 跟着的是对应模块的 `package.json` 中的 `name` 字段名。

### 添加内部模块之间的依赖

将pkg1作为pkg2的依赖进行安装：

```bash
$ lerna add pkg1 --scope pkg2
```

需要注意的是，通过这种方式安装的依赖，并不会将 `pkg1` 安装到 `pkg2` 的 `node_modules` 里，而是通过 `symlink` 的形式进行关联。

### 发布

以上包确认没有问题之后，就可以通过执行 `lerna publish` 进行发布了。

在进行publish之前需要首先提交你的代码，否则lerna会报错：

```bash
lerna ERR! ENOCOMMIT No commits in this repository. Please commit something before using version.
```

提交代码并关联到git仓库：

```bash
$ git add .
$ git commit -m 'init'
$ git remote add origin git@github.com:astonishqft/lerna-demo.git
$ git push -u origin main
```

### 删除某个包

将pkg1里面的glob包删除：

```bash
$ lerna exec --scope=pkg1 npm uninstall glob
```
### 抽离公共的包

上面可以看到，pkg1 和 pkg2 都依赖了 fs-extra 这个包，而各自 package 下面的 node_modules 都进行了一次安装，因此我们可以通过 --hoist 来抽取重复的依赖到最外层的 node_modules 目录下，同时最外层的 package.josn 的依赖信息也不会进行更新。

```bash
$ lerna bootstrap --hoist
```

### 最佳实践

前面我们已经介绍了 `lerna` 的相关概念和基本用法，目前最常见的解决方案是基于 `lerna` 和 `yarn workspace` 的 `monorepo` 工作流。由于 `yarn` 和 `lerna` 在功能上有较多的重叠，我们采用 `yarn` 官方推荐的做法，用 `yarn` 来处理依赖问题，用 `lerna` 来处理发布问题。

### yarn workspaces 与 lerna

`yarn workspaces` 是 `yarn` 提供的 `monorepo` 的依赖管理机制，用于在代码仓库的根目录下管理多个 `package` 依赖，与 `lerna` 不同的是，`yarn workspace` 只会在根目录下安装一个 `node_modules`，而 `lerna` 会进到每个 `package` 中执行 `yarn/npm install`，因此会在每个 `package` 下生成一个 `node_modules`。

因此，如果能在 `lerna` 中使用 `yarn workspaces` 特性，那就非常棒了！

幸运的是，在 `lerna` 中可以很方便的启用 `yarn workspaces`。

首先在工程的根目录下的 `package.json` 中增加 `"private": true` 和 `"workspaces”: [ "packages/*"]` 配置项。`"private": true` 可以确保根目录不会被发布到npm上，`"workspaces”: [ "packages/*"]` 声明了 `workspaces` 中所包含的项目路径。

`package.json` 配置文件增加如下配置。

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
