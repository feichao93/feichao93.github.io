# vscode-temme：快速抓取网页中的数据

[temme](https://github.com/shinima/temme) 是用于从 HTML 中提取 JSON 数据的选择器，其在 CSS 选择器语法的基础上添加了一些额外的语法，实现了多字段抓取、列表抓取等功能，适用于 Node.js 网页爬虫。[上一篇专栏文章](https://zhuanlan.zhihu.com/p/31624732) 介绍了 temme 选择器在命令行中的用法，本篇文章将以更直观的方式介绍该选择器。

从名字上也可以看出来，vscode-temme 是 temme 的 vscode 插件，实际使用效果如下图所示。

![screenshot](https://raw.githubusercontent.com/shinima/vscode-temme/master/docs/vscode-temme.gif)

上图展示了使用该插件来抓取 [芳文社番剧列表](http://bangumi.tv/anime/tag/%E8%8A%B3%E6%96%87%E7%A4%BE/?sort=date) 的整个过程。抓取结果为一个列表，每个列表元素包含了 id、番剧名称、图片链接、评分等信息。下图展示了网页的页面结构和对应的 CSS 选择器，这些选择器也都出现在了temme 选择器中。如果你熟悉 CSS 选择器的话，那么对照上下两张图片，很容易理解其中各个选择器的含义。

![selector-descriptions](selector-descriptions.jpg)

## 完整步骤说明

下面将以四个步骤来说明动图中的操作流程。

**第一步**

打开 vscode 编辑器，安装插件（在插件市场中搜索 temme 即可，安装前可能需要将编辑器升级到最新版），打开 temme 文件（[这里可以下载上图中的文件](https://github.com/shinima/vscode-temme/blob/master/fixtures/bangumi.temme)）。打开命令面板，选择 `Temme: Start watching`，然后选择 `<宇宙芳文社>`，插件会去根据链接去下载 HTML 文档，下载完成之后，插件会进入 watch mode，编辑器状态栏中会出现 `⠼ temme: watching`。在 watch mode 下，每次选择器发生变化，插件会重新执行选择器并更新输出。这样我们就可以愉快地编辑选择器了。

**第二步**

用浏览器打开 [芳文社番剧列表](http://bangumi.tv/anime/tag/%E8%8A%B3%E6%96%87%E7%A4%BE/?sort=date) 的话，我们可以看到上图所示的页面。我们要抓取的番剧信息列表位于 `ul#browserItemList` 对应的元素（褐色）中，每一个番剧的信息对应其中的一个 `li` 元素（绿色）。将这两个选择器写下来，并在后面添加 `@list` 表示抓取 `li` 列表。为了指定在每一个 `li` 元素中抓取哪些内容，我们还需要添加一堆花括号来放置子选择器。我们得到以下选择器：

`ul#browserItemList li@list { /* 子选择器将出现在这里 */ }`

**第三步**

上图中子选择器共有五个，每一个子选择器抓取一个对应字段，这里一个一个进行分析：

1. `&[id=$id];` 表示将父元素（也就是 li 元素）的 **id特性** 抓取到结果的 `id` 字段。`&` 符号表示父元素引用，和 CSS 预处理器（Less/Sass/Stylus）中的含义是一样的。
2. `.inner a{$name}` 表示将 `.inner a` 对应的元素的 **文本** 抓取到结果的 `name` 字段。
3. `img[src=$imgUrl]` 表示将 `img` 对应的元素的 **src特性** 抓取到结果的 `imgUrl` 字段。在 CSS 选择器中，`img[src=imgUrl]` 表示选取 src 为 imgUrl 的那些 img 元素；temme 这里添加了 `$` 符号，其含义就变成捕获，这个语法还是挺容易记住的 (o´ω`o)。
4. `.fade{$rate|Number}` 和 2 类似，不过这里多了 `|Number` 用来将结果从字符串转化为数字。
5. `.rank{$rank|firstNumber}` 和 4 类似，不过这里的 `firstNumber` 是自定义的过滤器，用来获取一个字符串中第一个数字，该过滤器定义在选择器文件的下方。

**第四步**

我们不仅可以在 `$xxx` 之后可以添加过滤器来处理某个数据字段，也可以在 `@yyy` 之后添加过滤器处理数组。`sortByRate` 和 `rateBetween` 是两个自定义的过滤器，前者按照评分对番剧列表进行排序，后者用来挑选评分位于一定区间的番剧。当我们应用这两个过滤器的时候，可以看到右侧的 JSON 数据也会发生相应变化。自定义过滤器的定义方式和 JavaScript 函数定义方式一样，只不过关键字从 function 变为了 filter，注意在自定义过滤器中需要使用 `this` 来引用捕获的结果。

## 插件碎碎念

插件会高亮显示那些符合模式 `// <tag> link` 的文本，我们称之为 tagged-link。link 可以为 http 链接，或者是本地文件路径。因为插件下载 HTML 的功能较为简单，所以我比较推荐使用插件 [vscode-restclient](https://github.com/Huachao/vscode-restclient) 先下载网页文档，然后再使用本地路径来启动 temme watch mode。另外，要在编辑器中执行 temme 选择器的话，文件中至少需要存在一个 tagged-link。

插件除了提供语法高亮以外，还会报告选择器语法错误。在 watch 模式下，因为选择器在不断执行，插件也会报告运行时错误，不过目前插件还尚未完善，运行时错误总是显示在文件的第一行，但应该问题不大。

## 选择器碎碎念

在 CSS 选择器语法的基础上，temme 要记的内容不多，一般来说记住以下几点就可以了：`$` 表示捕获字段，`@` 表示捕获列表，`|xxx` 表示应用过滤器来处理结果，选择器结束需要使用分号 `;`。temme 其他的语法和功能还请移步 [GitHub 文档](https://github.com/shinima/temme/blob/master/readme-zh.md)。

temme 发布在 NPM 上，使用 `yarn global add temme` 可以全局安装 temme；将选择器保存在文件 bangumi.temme 中，那么上面的例子也可以运行在命令行中：

```bash
url=http://bangumi.tv/anime/tag/%E8%8A%B3%E6%96%87%E7%A4%BE/?sort=date
curl -s $url | temme bangumi.temme --format
```

当然，我们也可以在 Node 中使用 temme。一般来说，对于每一种不同的网页结构，我们可以先使用该插件调试好选择器；等爬虫运行的时候下载得到 HTML 文档，我们直接执行相应的选择器即可，这样子爬虫开发效率大大提升。

## 感想与总结

选择合适的工具可以提高工作效率；编译原理很重要，也很有用。最后再给自己的选择器求一波 [star](https://github.com/shinima/temme)，谢谢各位观众老爷 (๑¯◡¯๑)。