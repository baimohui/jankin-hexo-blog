---
title: Webpack 详解
categories: 
- 模块打包
tags:
- 浏览器
- Webpack
---



前端开发已经模块化，它改进了代码库的封装和结构。打包工具已经成为了一个项目必不可少的部分，如今这儿有几种可能的选择，例如 webpack，grunt，gulp 等。webpack 因为他的功能和扩展性在过去的几年中，受到非常大的欢迎。

<img src="https://cdn.jsdelivr.net/gh/baimohui/FigureBed/img/20211106173720.png" alt="image-20210325155353232" style="zoom:67%;" />

**不像大多数的模块打包机，webpack 是把项目当作一个整体，通过一个给定的的主文件，webpack 将从这个文件开始找到你的项目的所有依赖文件，使用 loaders 处理它们，最后打包成一个或多个浏览器可识别的 js 文件。**<!-- more -->

## （一）webpack 作用

- **模块打包**。可以将不同模块的文件打包整合在一起，并且保证它们之间的引用正确，执行有序。利用打包我们就可以在开发的时候根据我们自己的业务自由划分文件模块，保证项目结构的清晰和可读性。
- **编译兼容**。在前端发展早期阶段，手写一堆浏览器兼容代码一直是令前端工程师头皮发麻的事情，而在今天这个问题被大大弱化了，通过`webpack`的`Loader`机制，不仅仅可以帮助我们对代码做`polyfill`，还可以编译转换诸如`.less, .vue, .jsx`这类在浏览器无法识别的格式文件，让我们在开发的时候可以使用新特性和新语法做开发，提高开发效率。
- **能力扩展**。通过`webpack`的`Plugin`机制，我们在实现模块化打包和编译兼容的基础上，可以进一步实现诸如按需加载，代码压缩等一系列功能，帮助我们进一步提高自动化程度，工程效率以及打包输出的质量。

## （二）模块打包运行原理

`Webpack`究竟是如何把这些模块合并到一起，并且保证其正常工作的。首先我们应了解`webpack`的整个打包流程：

1. 读取`webpack`的配置参数；
2. 启动`webpack`，创建`Compiler`对象并开始解析项目；
3. 从入口文件（`entry`）开始解析，并且找到其导入的依赖模块，递归遍历分析，形成依赖关系树；
4. 对不同文件类型的依赖模块文件使用对应的`Loader`进行编译，最终转为`Javascript`文件；
5. 整个过程中`webpack`会通过发布订阅模式，向外抛出一些`hooks`，而`webpack`的插件即可通过监听这些关键的事件节点，执行插件任务进而达到干预输出结果的目的。

其中文件的解析与构建是一个比较复杂的过程，在`webpack`源码中主要依赖于`compiler`和`compilation`两个核心对象实现。

- `compiler`对象是一个全局单例，它负责把控整个`webpack`打包的构建流程。 
- `compilation`对象是每一次构建的上下文对象，它包含了当次构建所需要的所有信息，每次热更新和重新构建，`compiler`都会重新生成一个新的`compilation`对象，负责此次更新的构建过程。

而每个模块间的依赖关系，则依赖于`AST`抽象语法树。每个模块文件在通过`Loader`解析完成之后，会通过`acorn`库生成模块代码的`AST`语法树，通过语法树就可以分析这个模块是否还有依赖的模块，进而继续循环执行下一个模块的编译解析。最终`Webpack`打包出来的`bundle`文件是一个`IIFE`（立即调用函数表达式）的执行函数。

```javascript
// webpack 5 打包的 bundle 文件内容

(() => { // webpackBootstrap
    var __webpack_modules__ = ({
        'file-A-path': ((modules) => { // ... })
        'index-file-path': ((__unused_webpack_module, __unused_webpack_exports, __webpack_require__) => { // ... })
    })
    
    // The module cache
    var __webpack_module_cache__ = {};
    
    // The require function
    function __webpack_require__(moduleId) {
        // Check if module is in cache
        var cachedModule = __webpack_module_cache__[moduleId];
        if (cachedModule !== undefined) {
                return cachedModule.exports;
        }
        // Create a new module (and put it into the cache)
        var module = __webpack_module_cache__[moduleId] = {
                // no module.id needed
                // no module.loaded needed
                exports: {}
        };

        // Execute the module function
        __webpack_modules__[moduleId](module, module.exports, __webpack_require__);

        // Return the exports of the module
        return module.exports;
    }
    
    // startup
    // Load entry module and return exports
    // This entry module can't be inlined because the eval devtool is used.
    var __webpack_exports__ = __webpack_require__("./src/index.js");
})
```



## （三）sourceMap

`sourceMap`是一项将编译、打包、压缩后的代码映射回源代码的技术，由于打包压缩后的代码并没有阅读性可言，一旦在开发中报错或者遇到问题，直接在混淆代码中`debug`问题会带来非常糟糕的体验，`sourceMap`可以帮助我们快速定位到源代码的位置，提高我们的开发效率。`sourceMap`并非`Webpack`特有的功能，像`JQuery`也支持`sourceMap`。

既然是一种源码的映射，那必然就需要有一份映射的文件，来标记混淆代码里对应的源码的位置，通常这份映射文件以`.map`结尾，里边的数据结构大概长这样：

```js
{
  "version" : 3,                          // Source Map 版本
  "file": "out.js",                       // 输出文件（可选）
  "sourceRoot": "",                       // 源文件根目录（可选）
  "sources": ["foo.js", "bar.js"],        // 源文件列表
  "sourcesContent": [null, null],         // 源内容列表（可选，和源文件列表顺序一致）
  "names": ["src", "maps", "are", "fun"], // mappings 使用的符号名称列表
  "mappings": "A,AAAB;;ABCDE;"            // 带有编码映射数据的字符串
}
```

其中`mappings`数据有如下规则：

- 生成文件中的一行的每个组用“;”分隔；
- 每一段用“,”分隔；
- 每个段由 1、4 或 5 个可变长度字段组成；

有了这份映射文件，我们只需要在压缩代码的最末端加上这句注释，即可让 sourceMap 生效：

```js
//# sourceURL=/path/to/file.js.map
```

有了这段注释后，浏览器就会通过`sourceURL`去获取这份映射文件，通过解释器解析后，实现源码和混淆代码之间的映射。因此 sourceMap 其实也是一项需要浏览器支持的技术。

如果我们仔细查看 webpack 打包出来的 bundle 文件，就可以发现在默认的`development`开发模式下，每个`_webpack_modules__`文件模块的代码最末端，都会加上`//# sourceURL=webpack://file-path?`，从而实现对 sourceMap 的支持。




## （四）webpack 使用

### install

首先添加我们即将使用的包：

```
npm install webpack webpack-dev-server --save-dev
```

webpack 是我们需要的模块打包机，webpack-dev-server 用来创建本地服务器，监听你的代码修改，并自动刷新修改后的结果。这些是 webpack.config.js 文件中有关 devServer 的配置。

- `contentBase`：为文件提供本地服务器
- `port`：监听端口，默认 8080
- `inline`：设置为 true，源文件发生改变自动刷新页面
- `historyApiFallback`：依赖 HTML5 history API，如果设置为 true，所有的页面跳转指向 index.html

```javascript
// exemple
devServer:{
    contentBase: './src' //本地服务器所加载的页面所在的目录
    historyApiFallback: true, //不跳转
    inline: true // 实时刷新
}
```

```
// 在'package.json'添加两个命令用于本地开发和生产发布
"scripts": {
    "start": "webpack-dev-server",
    "build": "webpack"
}
```

在使用 webpack 命令的时候，他将接受 webpack 的配置文件，除非我们使用其他的操作

### Entry

entry: 用来写入口文件，它将是整个依赖关系的根

```javascript
var baseConfig = {
	entry: './src/index.js'
}
```

当我们需要多个入口文件的时候，可以把 entry 写成一个对象

```javascript
var baseConfig = {
	entry: {
    	main: './src/index.js'
    }
}
```

我建议使用后面一种方法，因为他的规模会随你的项目增大而变得繁琐

### Output

output: 即使入口文件有多个，但是只有一个输出配置

```javascript
var path = require('path')
var baseConfig = {
	entry: {
    	main: './src/index.js'
    },
    output: {
    	filename: 'main.js',
        path: path.resolve('./build')
    }
}
module.exports = baseConfig
```

如果你定义的入口文件有多个，那么我们需要使用占位符来确保输出文件的唯一性

```javascript
output: {
    filename: '[name].js',
    path: path.resolve('./build')
}
```

如今这么少的配置，就能够让你运行一个服务器并在本地使用命令 npm start 或者 npm run build 来打包我们的代码进行发布

### Loader

`Webpack`最后打包出来的成果是一份`Javascript`代码，实际上在`Webpack`内部默认也只能够处理`JS`模块代码，在打包过程中，会默认把所有遇到的文件都当作 `JavaScript`代码进行解析，因此当项目存在非`JS`类型文件时，我们需要先对其进行必要的转换，才能继续执行打包任务，这也是`Loader`机制存在的意义。

**loader 的作用**：

1. 实现对不同格式的文件的处理，比如说将 scss 转换为 css，或者 typescript 转化为 js

2. 转换这些文件，从而使其能够被添加到依赖图中

loader 是 webpack 最重要的部分之一，通过使用不同的 Loader，我们能够调用外部的脚本或者工具，实现对不同格式文件的处理，loader 需要在 webpack.config.js 里边单独用 module 进行配置，配置如下：

- test：匹配所处理文件的扩展名的正则表达式（必须）
- loader：loader 的名称（必须）
- include/exclude：手动添加处理的文件，屏蔽不需要处理的文件（可选）
- query：为 loaders 提供额外的设置选项

```javascript
// exemple 
var baseConfig = {
    // ...
    module: {
        rules: [
            {
                test: /*匹配文件后缀名的正则*/,
                use: [
                	loader: /*loader 名字*/,
                	query: /*额外配置*/
                ]
    		}
    	]
	}
}
```

要使 loader 工作，我们需要一个正则表达式来标识我们要修改的文件，然后用一个数组表示我们即将使用的 Loader，当然我们需要的 loader 需要通过 npm 进行安装。例如我们需要解析 less 的文件，那么 webpack.config.js 的配置如下：

```javascript
var baseConfig = {
    entry: {
        main: './src/index.js'
    },
    output: {
        filename: '[name].js',
        path: path.resolve('./build')
    },
    devServer: {
        contentBase: './src',
        historyApiFallBack: true,
        inline: true
    },
    module: {
        rules: [
            {
                test: /\.less$/,
                use: [
                    {loader: 'style-loader'},
                    {loader: 'css-loader'},
                    {loader: 'less-loader'}
                ],
                exclude: /node_modules/
            }
        ]
    }
}
```


- babel-loader：让下一代的 js 文件转换成现代浏览器能够支持的 JS 文件。
- babel：有些复杂，所以大多数都会新建一个`.babelrc` 进行配置
- css-loader，style-loader：两个建议配合使用，用来解析 css 文件，能够解释`@import url()`，如果需要解析 less 就在后面加一个 less-loader
- file-loader：生成的文件名就是文件内容的 MD5 哈希值，并会保留所引用资源的原始扩展名
- url-loader：功能类似 file-loader，但是文件大小低于指定的限制时，可以返回一个`DataURL`。事实上，在使用 less,scss,stylus 时，npm 会提示你差什么插件，差什么你安上就行了

### Plugins

如果说`Loader`负责文件转换，那么`Plugin`便是负责功能扩展。`Loader`和`Plugin`作为`Webpack`的两个重要组成部分，承担着两部分不同的职责。上文已经说过，`webpack`基于发布订阅模式，在运行的生命周期中会广播出许多事件，插件通过监听这些事件，就可以在特定的阶段执行自己的插件任务，从而实现自己想要的功能。

既然基于发布订阅模式，那么知道`Webpack`到底提供了哪些事件钩子供插件开发者使用是非常重要的，上文提到过`compiler`和`compilation`是`Webpack`两个非常核心的对象，其中`compiler`暴露了和 `Webpack`整个生命周期相关的钩子（[compiler-hooks](https://webpack.js.org/api/compiler-hooks/)），而`compilation`则暴露了与模块和依赖有关的粒度更小的事件钩子（[Compilation Hooks](https://webpack.js.org/api/compilation-hooks/)）。

`Webpack`的事件机制基于`webpack`自己实现的一套`Tapable`事件流方案（[github](https://github.com/webpack/tapable)）

```js
// Tapable 的简单使用
const { SyncHook } = require("tapable");

class Car {
    constructor() {
        // 在 this.hooks 中定义所有的钩子事件
        this.hooks = {
            accelerate: new SyncHook(["newSpeed"]),
            brake: new SyncHook(),
            calculateRoutes: new AsyncParallelHook(["source", "target", "routesList"])
        };
    }

    /* ... */
}

const myCar = new Car();
// 通过调用 tap 方法即可增加一个消费者，订阅对应的钩子事件了
myCar.hooks.brake.tap("WarningLampPlugin", () => warningLamp.on());
```

`Plugin`的开发和开发`Loader`一样，需要遵循一些开发上的规范和原则：

- 插件必须是一个函数或者是一个包含 `apply` 方法的对象，这样才能访问`compiler`实例；
- 传给每个插件的 `compiler` 和 `compilation` 对象都是同一个引用，若在一个插件中修改了它们的属性，会影响后面的插件;
- 异步的事件需要在插件处理完任务时调用回调函数通知 `Webpack` 进入下一个流程，不然会卡住;

了解了以上这些内容，想要开发一个 `Webpack Plugin`，其实也并不困难。

```js
class MyPlugin {
  apply (compiler) {
    // 找到合适的事件钩子，实现自己的插件功能
    compiler.hooks.emit.tap('MyPlugin', compilation => {
        // compilation: 当前打包构建流程的上下文
        console.log(compilation);
        
        // do something...
    })
  }
}
```

**loaders 负责的是处理源文件的如 css、jsx，一次处理一个文件。而 plugins 并不是直接操作单个文件，**它直接对整个构建过程起作用。下面列举了一些我们常用的 plugins 和对应用法。

**`ExtractTextWebpackPlugin`**:

它会将入口中引用的 css 文件，都打包在独立的 css 文件中，而不是内嵌在 js 打包文件中。应用如下：

```javascript
var ExtractTextPlugin = require('extract-text-webpack-plugin')
var lessRules = {
    use: [
        {loader: 'css-loader'},
        {loader: 'less-loader'}
    ]
}

var baseConfig = {
    // ... 
    module: {
        rules: [
            // ...
            {test: /\.less$/, use: ExtractTextPlugin.extract(lessRules)}
        ]
    },
    plugins: [
        new ExtractTextPlugin('main.css')
    ]
}
```

**`HtmlWebpackPlugin`**

依据一个简单的 index.html 模版，生成一个自动引用你打包后的 js 文件的新 index.html

```javascript
var HTMLWebpackPlugin = require('html-webpack-plugin')
var baseConfig = {
    // ...
    plugins: [
        new HTMLWebpackPlugin()
    ]
}
```

**`HotModuleReplacementPlugin`**

它允许在修改组件代码时自动进行刷新，以实时预览修改后的结果。注意不要在生产环境中使用 HMR（一般情况分为开发环境，测试环境，生产环境）。用法为： `new webpack.HotModuleReplacementPlugin()`

**webpack.config.js 的全部内容示例**

```javascript
const webpack = require("webpack")
const HtmlWebpackPlugin = require("html-webpack-plugin")
var ExtractTextPlugin = require('extract-text-webpack-plugin')
var lessRules = {
    use: [
        {loader: 'css-loader'},
        {loader: 'less-loader'}
    ]
}
module.exports = {
    entry: {
        main: './src/index.js'
    },
    output: {
        filename: '[name].js',
        path: path.resolve('./build')
    },
    devServer: {
        contentBase: '/src',
        historyApiFallback: true,
        inline: true,
        hot: true
    },
    module: {
        rules: [
            {test: /\.less$/, use: ExtractTextPlugin.extract(lessRules)}
        ]
    },
    plugins: [
        new ExtractTextPlugin('main.css')
    ]
}
```

### 产品阶段的构建

目前为止，在开发阶段的东西已经基本完成。但在产品阶段还需要对资源进行别的处理，例如压缩，优化，缓存，分离 CSS 和 JS。首先我们来定义产品环境：

```javascript
var ENV = process.env.NODE_ENV
var baseConfig = {
    // ... 
    plugins: [
        new webpack.DefinePlugin({
            'process.env.NODE_ENV': JSON.stringify(ENV)
        })
    ]
}
```

然后还需要修改 script 命令：

```
"scripts": {
	"start": "NODE_ENV=development webpack-dev-server",
	"build": "NODE_ENV=production webpack"
}
```

`process.env.NODE_ENV` 将被一个字符串替代，它运行压缩器排除那些不可到达的开发代码分支。
当你引入那些不会进行生产的代码，下面这个代码将非常有用。

```javascript
if (process.env.NODE_ENV === 'development') {
    console.warn('这个警告会在生产阶段消失')
}
```

### 优化插件

下面介绍几个插件用来优化代码

- `OccurrenceOrderPlugin`：为组件分配 ID，通过这个插件 webpack 可以分析和优先考虑使用最多的模块，然后为它们分配最小的 ID
- `UglifyJsPlugin`：压缩代码

下面是他们的使用方法

```javascript
var baseConfig = {
    // ...
     new webpack.optimize.OccurenceOrderPlugin()
     new webpack.optimize.UglifyJsPlugin()
}
```

然后在我们使用 npm run build 会发现代码是压缩的。

## （五）编写自定义 Plugin

插件是 webpack 的支柱功能。`webpack` 自身也是构建于，你在 `webpack` 配置中用到的相同的插件系统之上！插件目的在于解决 `loader` 无法实现的其他事。要想写好插件就要知道`Webpack`中的两个比较核心的概念`compiler`、`compilation`、`tapable`。在[webpack 编译流程]()已经都要记录。 `Webpack` 通过 `Plugin` 机制让其更加灵活，以适应各种应用场景。在 `Webpack` 运行的生命周期中会广播出许多事件，`Plugin` 可以监听这些事件，在合适的时机通过 `Webpack` 提供的 `API` 改变输出结果。

### 实现一个 plugin

------

一个 webpack plugin 基本包含以下几步：

1. 一个 JavaScript 函数或者类
2. 在函数原型（prototype）中定义一个注入`compiler`对象的`apply`方法。
3. `apply`函数中通过`compiler`插入指定的事件钩子，在钩子回调中拿到`compilation`对象
4. 使用`compilation`操纵修改`webapack`内部实例数据。
5. 异步插件，数据处理完后使用`callback`回调

最后会实现一个简单的`clean-webpack-plugin`。

### 一个简单的插件

```javascript
class WebpackCleanupPlugin {
  // 构造函数
  constructor(options) {
    console.log("WebpackCleanupPlugin", options);
  }
  // 应用函数
  apply(compiler) {
    console.log(compiler);
    // 绑定钩子事件
    compiler.plugin("done", compilation => {
      console.log(compilation);
    });
  }
}
```

如何使用在 webpack.config.js 中引入并且使用如下：

```javascript
const MiniCssExtractPlugin = require("mini-css-extract-plugin");
// 引入自己的插件
const WebpackCleanupPlugin = require("./WebpackCleanupPlugin");
module.exports = {
  devtool: "source-map",
  mode: "production",
  entry: {
    index: "./src/index.js",
    chunk1: "./src/chunk1.js"
  },
  output: {
    filename: "[name].[chunkhash].js"
  },
  module: {
    rules: [
      {
        test: /\.css$/,
        use: [MiniCssExtractPlugin.loader, "css-loader"]
      }
    ]
  },
  plugins: [
    // 提取 css 插件
    new MiniCssExtractPlugin({
      // Options similar to the same options in webpackOptions.output
      // both options are optional
      filename: "[name].[contenthash].css"
    }),
    // 使用自己的插件
    new WebpackCleanupPlugin()
  ]
};
```

自己写的插件如下执行：

- `webpack` 启动后，在读取配置的过程中会先执行 `new WebpackCleanupPlugin()` ，初始化一个 `WebpackCleanupPlugin` 。
- 在初始化 `compiler` 对象后，再调用 `WebpackCleanupPlugin.apply(compiler)` 给插件实例传入 `compiler` 对象。
- 插件实例在获取到 `compiler` 对象后，就可以通过 `compiler.plugin`(事件名称，回调函数) 监听到 `Webpack` 广播出来的事件。
- 并且可以通过 `compiler` 对象去操作 `webpack`。

**Compiler、Compilation**

- **Compiler 对象包含了 Webpack 环境所有的的配置信息**，包含 `options`，`hook`，`loaders`，`plugins` 这些信息，这个对象在 `Webpack` 启动时候被实例化，它是**全局唯一**的，可以简单地把它理解为 `Webpack` 实例；`Compiler`中包含的东西如下所示：

![webpacl-plugin](https://user-gold-cdn.xitu.io/2019/9/5/16d003f3b01c273c?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

- **Compilation 对象包含了当前的模块资源、编译生成资源、变化的文件等**。当 `Webpack` 以开发模式运行时，每当检测到一个文件变化，一次新的 `Compilation` 将被创建。`Compilation` 对象也提供了很多事件回调供插件做扩展。通过 `Compilation` 也能读取到 `Compiler` 对象。

`Compilation`中包含的东西如下所示：

![webpacl-plugin](https://user-gold-cdn.xitu.io/2019/9/5/16d003fa8e2cc44a?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

> **Compiler 和 Compilation 的区别在于**：`Compiler` 代表了整个 `Webpack` 从启动到关闭的生命周期，而 `Compilation` 只是代表了一次新的编译。

[Compiler 钩子](https://www.webpackjs.com/api/compiler-hooks/)和[compilation 钩子](https://www.webpackjs.com/api/compilation-hooks/)

### 一个简单的清除文件插件

------

每次打包如果文件有修改会生成新的文件，文件的**hash**也会跟着变化，那么这个改变了的文件，他以前的文件就是无效的了，要把以前的文件清除掉，我们使用比较多的就是`clean-webpack-plugin`，这里自己实现一个简单的文件清除。如果不知道[hash、contenthash、chunkhash]()的区别可以看这一片文章。

大致分为以下几步：

- 获取`output`路径，也就是出口路径一般为`dist`
- 绑定钩子事件 `compiler.plugin('done', (stats) => {})`
- 编译文件，与原来文件对比，删除未匹配文件（同时可以 options 设置要忽略的文件）

代码实现如下

```javascript
const recursiveReadSync = require("recursive-readdir-sync");
const minimatch = require("minimatch");
const path = require("path");
const fs = require("fs");
const union = require("lodash.union");

// 匹配文件
function getFiles(fromPath, exclude = []) {
  const files = recursiveReadSync(fromPath).filter(file =>
    exclude.every(
      excluded =>
        !minimatch(path.relative(fromPath, file), path.join(excluded), {
          dot: true
        })
    )
  );
  // console.log(files);
  return files;
}

class WebpackCleanupPlugin {
  constructor(options = {}) {
    // 配置文件
    this.options = options;
  }
  apply(compiler) {
    // 获取 output 路径
    const outputPath = compiler.options.output.path;
    // 绑定钩子事件
    compiler.plugin("done", stats => {
      if (
        compiler.outputFileSystem.constructor.name !== "NodeOutputFileSystem"
      ) {
        return;
      }
      // 获取编译完成 文件名
      const assets = stats.toJson().assets.map(asset => asset.name);
      console.log(assets);
      // 多数组合并并且去重
      const exclude = union(this.options.exclude, assets);
      console.log(exclude);
      // console.log('outputPath', outputPath);
      // 获取未匹配文件
      const files = getFiles(outputPath, exclude);
      // const files = [];
      console.log("files", files);
      if (this.options.preview) {
        // console.log('%s file(s) would be deleted:', files.length);
        // 输出文件
        files.forEach(file => console.log("    %s", file));
        // console.log();
      } else {
        // 删除未匹配文件
        files.forEach(fs.unlinkSync);
      }
      if (!this.options.quiet) {
        // console.log('\nWebpackCleanupPlugin: %s file(s) deleted.', files.length);
      }
    });
  }
}
module.exports = WebpackCleanupPlugin;
```

上面的这个插件实现了一个清除编译文件的效果。在这里就不做实验了，如果有兴趣可以自己把代码 copy 到本地，运行一下看一下结果。

### 总结

在上面大致知道怎么写一个简单的清除文件的`webpack`的**插件**，其实还可以做更多的事情如下：

- 读取输出资源、代码块、模块及其依赖（在 `emit` 事件发生）
- 监听文件变化 `watch-run`
- 修改输出资源 `compilation.assets`

具体实现可以看一下一下[webpack 深入浅出](http://webpack.wuhaolin.cn/5原理/5-4编写Plugin.html)

