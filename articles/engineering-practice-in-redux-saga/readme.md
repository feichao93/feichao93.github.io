# redux-saga 中的开源工程实践

接触 redux-saga 也有好长一段时间了，这篇文章将主要介绍 redux-saga 中用到的开源工具。维护开源项目是相当耗费精力的一件事，维护者除了需要写代码之外，还需要花时间在文档、教程、解决 issue、答疑等方面上。很多工作包含了大量重复劳动，redux-saga 配置了各种各样的开源工具，通过自动化的方式减少大量的重复工作。这其中许多工具也可以应用在我们普通开发者的日常开发中，提升开发效率。

## 依赖管理与启动脚本 —— npm

npm 已经是前端的标配了。大部分时候，我们将前端项目从 GitHub 克隆下来之后，便施展一套熟练的起手式： `npm install && npm start`。redux-saga 也是一样，package.json 文件详细记录了项目的信息：名称、版本、开源协议、代码仓库地址等，以及一个长长的依赖列表。后面介绍的工具都会出现在该依赖列表中，运行 `npm install` 时，npm 会将这些依赖安装到项目的 node_modules 文件夹下。

不同前端项目开启调试环境（例如开启 webpack-dev-server）的方式不尽相同，一般开发者会将开启项目调试环境的命令写入 start 脚本，`npm start` 也成为了通用的启动调试环境的脚本。redux-saga 自带若干示例项目（examples/\*），我们可以通过在示例目录下运行 `npm start` 来运行各个示例。

## 多 package 管理 —— lerna

与我们的日常项目不一样的是，redux-saga 仓库中包含多个 package。其中 package 两个需要发布在 npm 上：

- 一个 redux-saga 类库本身，npm 的包名即为 `redux-saga`
- 另一个是 babel-plugin-redux-saga，用于提升 saga 报错信息的可读性

其他 package 为仓库中的示例项目，不会发布在 npm 中（即 package.json 中 private 字段为 `true`）。每个示例项目都是独立且完整的 node/npm 工程，用户可以进入示例目录运行 `npm install && npm start` 来查看示例的实际效果。

redux-saga 仓库下各个 package 目录结构大致如下：

```
<redux-saga仓库>
    package.json
    lerna.json
    packages/
        core/
            pacakge.json
            src/
            test/
        babel-plugin-redux-saga/
            pacakge.json    src/    test/
    exmaples/
        async/
            pacakge.json    src/    test/
        cancellable-counter/
            pacakge.json    src/    test/
        ..... 其他示例项目
```

从上面的目录我们可以看出来整个仓库中有多个 package.json 分散在不同的目录下，如果需要手动地跑多次 `npm install` 来安装所有的依赖，那将是相当繁琐的。lerna 提供了 bootstrap 命令来解决这个问题，在仓库目录执行 `lerna bootstrap`，lerna 将会为所有的 package 安装依赖。当然有的同学会问我用 shell 脚本加上一个循环也可以做到这个啊，为什么一定要使用 lerna 呢？\_(:з」∠)\_。这位同学你先坐下，下面我们来看 「lerna bootstrap」相比于「依次 npm install」的优势。

**优势一：lerna bootstrap 有更快的安装速度，更少的依赖占用空间**

在 redux-saga 中共有 8 个示例项目，其中 6 个使用 webpack 来进行打包构建，且不同示例项目使用的 webpack 版本相同。如果我们依次安装依赖，那么我们将会安装 6 份 webpack，每一份安装位于各个示例的 node_modules 目录下。显然，这 6 份安装中有 5 份都是多余的。根据 node 查找模块的算法，我们只需要在这些模块的父文件夹（即仓库文件夹）安装一份 webpack 即可。

当使用 [--hoist 参数](https://github.com/lerna/lerna/blob/master/doc/hoist.md) 时，lerna 会分析出不同模块的公共依赖，并将这些公共依赖安装在仓库文件夹。当公共依赖版本不一致时（例如上述例子中 5 份 webpack 版本要求为 4.x，另外一份 webpack 要求 3.x），lerna 会将**最常用的版本**安装在仓库文件夹，不一致的那些版本仍会被安装在各自的模块文件夹中。

下图列举了不同情况下整个 redux-saga 项目的大小和文件数量，我们可以看到**使用 hoist 可以减少约 60% 的空间占用**。（下图中**初始大小**较大，这是因为初始情况下也包含了完整的 git 记录）

![comparation](comparation.jpg)

**优势二：lerna bootstrap [会为 package 之间的相互引用创建符号链接](https://github.com/lerna/lerna/tree/master/commands/bootstrap#usage)**

例如我们的示例项目 example/async 依赖于 redux-saga package，并在 package.json 添加了 redux-saga 依赖这一行。那么在执行 lerna bootstrap 时，lerna 不会再去下载 npm registry 中的版本，而是直接在 example/async/node_modules 文件夹创建一个符号链接，指向 packages/core，这样我们就可以在示例项目中用到最新的 redux-saga 版本。值得一提的是，npm/yarn [也提供了 link 的功能](https://docs.npmjs.com/cli/link)，方便用户调试自己本地的 package。

redux-saga 中的示例项目都带有一定的测试，在使用符号链接的情况下，这些测试都将使用最新的 redux-saga 版本，帮助我们发现最新版本出现的问题。为了确保测试时使用的是最新版本，redux-saga 将 npm pretest 脚本设置为 `npm run build`，确保每次测试都使用最新打包出来的文件。

## 代码风格 —— prettier & ESLint

代码风格本应是仁者见仁智者见智的一件事，但当开发人员较多，且开发人员不可控（redux-saga 社区活跃，不知道谁在什么时候会贡献代码）的时候，选择偏向性更强、规则更严的格式化工具更为合适。redux-saga 使用 prettier 作为格式化工具，并设置了 lint-staged 工具确保所有代码在提交时都会经过了格式化。prettier 是一个 opinionated 的格式化工具，工具自带一套代码风格，可供开发者配置的选项并不多；prettier 也是一个非常严格的代码格式化工具，只要代码的 AST（抽象语法树）相同，使用该工具就能得到相同的输出（除了少数空行、换行等例外）。

ESLint 是一个功能丰富且强大的静态检查工具，提供了武装到牙齿的配置。ESLint 默认包含了 250+ 不同的规则，每个规则拥有若干选项来对单个规则进行配置；ESLint 的插件机制允许开发者安装插件来使用其他规则，例如非常流行的 [eslint-react-plugin](https://github.com/yannickcr/eslint-plugin-react) 提供了约 80 个 react/JSX 相关规则。

ESLint 规则的粒度非常细致，例如规则 [generator-star-spacing](https://eslint.org/docs/rules/generator-star-spacing) 可用来配置「生成器函数的星号两边是否需要空格」，该规则允许我们选择 `before` / `after` / `both` / `neither` 中的其中一种，此外，该规则还允许我们针对不同的生成器声明方式（命名函数 / 匿名函数 / 方法）单独设置上述空格配置 \_(:з」∠)\_。

从零开始配置 ESLint 是一件很繁琐的事情，好在 ESLint 提供了拓展机制，允许我们基于已有的规则集合进行二次配置。ESLint 也提供了 eslint:recommended，该规则集合包含了针对一些常见的错误（未定义的变量，无法到达的代码等）的规则。 redux-saga 使用了 eslint:recommended 与 plugin:react/recommended，这两个集合基本能够覆盖代码检查需求。

## 自动化测试 —— tape & jest

自动化测试这个词我们已经听过好多遍，几乎每本讲编程的书，都会有那么几个小节介绍自动化测试以及其带来的好处。自动化测试其实也挺讲究，我个人认为测试质量有如下几个阶段：

第一阶段，从无到有：我们开始书写测试用例，我们会写一些简单的测试覆盖一些常见的情况。即使这些测试很简单，但通过这个测试，我们至少能够保证代码在大部分情况下将正常运行。

第二阶段，从低覆盖率到高覆盖率：我们开始关注一些不太常见的情况，并构造用例来测试代码在一些边界条件下是否正常运行。一些工具（例如 jest 所使用的 [istanbul](https://istanbul.js.org/)）会生成测试覆盖率（语句覆盖率，行覆盖率，分支覆盖率）报告，会告诉我们每一行代码是否被执行，执行了多少次。通过这些工具我们不断补充缺失的测试用例，直至覆盖率达到一个较高的值。

第三阶段，从写测试到设计测试：我们开始思考如何更好地设计测试用例，我们开始考虑以下这些问题「测试是否足够小，小到恰好测试我们想测试的代码单元？」，「多个测试之间是否有重复，是否可以移除一些不必要的测试？」，「测试是否足够直观，输入输出的可读性如何，单元测试是否易于构造？」…… 我们不再满足于「让代码通过测试」，而是像设计软件一样去设计测试用例，并像核心代码一样去维护测试代码。

redux-saga 包含了非常完善的自动化测试，每一个 effect 类型都有若干相应的用例来保证其在不同情况下运行正常，同时丰富的测试还涵盖了 sagaHelper（例如 takeEvery、takeLatest）、数据结构（例如 buffer 与 channel）、typescript 类型、saga monitor 等。测试用例一般会在实现功能时就准备好（和功能代码放在一个 pull request），也会在日常的维护中被不断改进。

redux-saga 使用 tape 作为自动化测试工具。tape 是一个非常简单的测试工具，我们需要在测试文件引入 tape，然后使用其提供的函数来书写测试用例。tape 只是一个简单的 node 模块，也没有什么魔法，故测试文件都是能够独立运行的 js 文件，我们可以直接使用 node 来运行测试文件。当测试文件较多时，我们可以新建一个文件（例如叫做 index.js），并在该文件中 require 其他测试文件，然后运行 index.js 便能运行所有测试。

## 打包转换

JS 打包转换工具
babel rollup webpack

## 更多的自动化 —— git-hook & npm-scripts

## 其他工具

gitbook

lint-staged

bundlesize

travis-ci

templates：issue template and pull-request template

gitignore
