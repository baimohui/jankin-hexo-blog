---
title: Webpack 详解
categories:
- 模块打包
tags:
- 浏览器
- Webpack
- 前端工程化
---

`Webpack` 本质上是一个**静态模块打包器**：从入口出发，构建依赖图，经过 Loader 转换与 Plugin 扩展，最终输出可部署资源。本文以 `webpack5` 为主。<!-- more -->

## 一、Webpack 解决了什么问题

### 一句话版本
`Webpack` 本质上解决了前端项目在「模块化开发 → 浏览器运行 → 工程化交付」这一完整链路中的断层问题。

### 深入理解这三个痛点

在没有 Webpack 的时代，前端开发面临以下困境：

#### 1. 模块化开发的困境
传统方式下，我们需要手动在 HTML 中按顺序引入 `<script>` 标签：
```html
<script src="utils.js"></script>
<script src="component.js"></script>
<script src="app.js"></script>
```
**问题：**
- 必须严格按照依赖顺序排列，一旦顺序错误就会报错
- 全局变量污染，不同文件间变量可能冲突
- 难以追踪依赖关系，代码维护成本高
- 无法按需加载，所有代码一次性加载

**Webpack 的解决方案：**
- 支持多种模块规范：ESM（`import/export`）、CJS（`require/module.exports`）、AMD 等
- 自动分析模块依赖关系，构建完整的依赖图（Dependency Graph）
- 消除全局变量污染，每个模块都有独立作用域
- 支持动态导入，实现按需加载

#### 2. 浏览器兼容性与资源处理
现代前端开发使用了大量浏览器原生不支持的技术：
- TypeScript、CoffeeScript 等超集语言
- Sass、Less、Stylus 等 CSS 预处理器
- Vue、React 组件文件
- 图片、字体等静态资源

**问题：**
- 浏览器无法直接运行这些文件，需要转换
- 每种资源需要不同的工具处理，配置繁琐
- 资源引用路径管理困难
- 难以优化资源加载策略

**Webpack 的解决方案：**
- 通过 Loader 机制，将任意类型文件转换为可识别的模块
- 统一的资源处理管道，从入口开始递归处理所有依赖
- 内置 Asset Modules 处理静态资源，无需额外 loader
- 支持资源内联、分离、哈希命名等策略

#### 3. 工程化交付的缺失
前端项目需要考虑：
- 代码压缩混淆
- 资源哈希缓存
- 代码分割按需加载
- 开发调试体验
- 构建性能优化

**问题：**
- 需要手动配置多个工具（gulp、grunt、babel 等）
- 配置复杂且容易出错
- 工具间集成困难
- 缺乏统一的最佳实践

**Webpack 的解决方案：**
- 内置完整的工程化能力
- 通过 Plugin 机制扩展功能
- 成熟的生态系统，大量现成解决方案
- webpack5 进一步优化了默认配置，开箱即用

### 可展开为三点总结

1. **模块依赖管理**：支持 ESM/CJS 等多种模块体系，从入口出发递归分析，构建完整的模块依赖图，自动处理模块间的依赖关系。
2. **非 JS 资源纳入构建**：CSS、图片、字体、TS、Vue/React 代码等所有前端资源都可作为模块参与打包，通过 Loader 统一转换处理。
3. **工程能力集成**：代码分割、持久化缓存、代码压缩、Tree Shaking、SourceMap 调试、DevServer 本地服务、HMR 热更新等，一站式解决前端工程化需求。

## 二、核心概念（高频）

理解 Webpack 的核心概念是掌握其工作原理的关键。这部分内容在面试中出现频率极高，务必吃透。

---

### 1. Entry / Output / Mode

#### Entry（入口）
`entry` 是 webpack 构建依赖图的起点，告诉 webpack 从哪个文件开始分析。

**常见配置方式：**

```js
// 1. 单入口（字符串形式）
entry: './src/main.js'

// 2. 单入口（对象形式，更清晰）
entry: {
  main: './src/main.js'
}

// 3. 多入口（多页面应用场景）
entry: {
  app: './src/app.js',
  admin: './src/admin.js'
}
```

**入口的作用：**
- 从入口文件开始，webpack 递归解析所有依赖的模块
- 每个入口对应一个初始 chunk
- 多入口适合多页面应用或需要独立打包的功能模块

#### Output（输出）
`output` 配置 webpack 如何输出构建产物，包括文件名、输出目录、publicPath 等。

**核心配置项：**

```js
output: {
  // 输出目录（必须是绝对路径）
  path: path.resolve(__dirname, 'dist'),
  
  // 主 bundle 文件名
  filename: 'js/[name].[contenthash:8].js',
  
  // 非入口 chunk 的文件名（动态导入等产生）
  chunkFilename: 'js/[name].[contenthash:8].chunk.js',
  
  // 每次构建前清理输出目录（webpack5 新增）
  clean: true,
  
  // 资源访问路径（CDN 部署时常用）
  publicPath: '/'
}
```

**文件名占位符详解：**
- `[name]`：chunk 名称（入口名称或动态导入指定的名称）
- `[contenthash]`：基于文件内容生成的哈希值，内容不变哈希不变
- `[chunkhash]`：基于 chunk 内容生成的哈希值
- `[hash]`：基于整个构建生成的哈希值
- `[id]`：chunk 的唯一 ID

**publicPath 的作用：**
- 指定浏览器访问资源时的基础路径
- 本地开发通常设为 `/`
- CDN 部署时设为 CDN 地址，如 `https://cdn.example.com/`
- 影响 CSS 中图片、字体等资源的引用路径

#### Mode（模式）
`mode` 告诉 webpack 使用哪种内置优化策略。

```js
mode: 'development' // 或 'production' 或 'none'
```

**三种模式对比：**

| 模式 | 特点 | 适用场景 |
|------|------|----------|
| `development` | 开启调试工具、不压缩代码、保留完整路径、快速构建 | 开发环境 |
| `production` | 启用 Tree Shaking、代码压缩、作用域提升、删除调试代码 | 生产环境 |
| `none` | 不使用任何默认优化 | 自定义配置场景 |

**production 模式下的默认优化：**
- 启用 `TerserPlugin` 压缩 JavaScript
- 启用 `MiniCssExtractPlugin` 提取 CSS
- 启用 Tree Shaking 移除未使用代码
- 设置 `process.env.NODE_ENV = 'production'`

---

### 2. Loader vs Plugin

这是面试中最常被问到的问题，必须清晰区分两者的职责。

#### Loader（加载器）

**核心职责：** 处理模块内容的转换，将非 JavaScript 文件转换为 webpack 可识别的模块。

**工作原理：**
- webpack 原生只能理解 JavaScript 和 JSON
- Loader 让 webpack 能够处理其他类型的文件
- Loader 是函数，接收源文件内容，返回转换后的内容
- 多个 Loader 可以链式调用，**从右到左、从下到上**执行

**常见 Loader：**

| Loader | 作用 |
|--------|------|
| `babel-loader` | 将 ES6+ 转译为 ES5 |
| `ts-loader` | 将 TypeScript 转为 JavaScript |
| `css-loader` | 解析 CSS 文件中的 `@import` 和 `url()` |
| `style-loader` | 将 CSS 注入到 DOM 的 `<style>` 标签中 |
| `sass-loader` | 将 Sass/SCSS 转为 CSS |
| `file-loader` | 将文件输出到指定目录，返回文件路径 |
| `raw-loader` | 将文件作为字符串导入 |

**Loader 链式调用示例：**

```js
module: {
  rules: [
    {
      test: /\.scss$/,
      use: [
        'style-loader',       // 3. 将 CSS 注入 DOM
        'css-loader',         // 2. 解析 CSS 中的导入
        'sass-loader'         // 1. 将 SCSS 转为 CSS
      ]
    }
  ]
}
```

**执行顺序：** `sass-loader` → `css-loader` → `style-loader`

#### Plugin（插件）

**核心职责：** 介入 webpack 构建生命周期的各个阶段，执行更广泛的任务。

**工作原理：**
- Plugin 基于 webpack 的 Tapable 钩子系统
- 可以在构建的任意时机执行自定义逻辑
- 比 Loader 更强大，可以访问 compiler 和 compilation 对象
- 可以修改输出资源、添加新资源、优化构建等

**常见 Plugin：**

| Plugin | 作用 |
|--------|------|
| `HtmlWebpackPlugin` | 自动生成 HTML 文件并注入 bundle |
| `MiniCssExtractPlugin` | 将 CSS 提取为独立文件 |
| `CleanWebpackPlugin` | 清理输出目录（webpack5 已内置） |
| `DefinePlugin` | 定义全局常量 |
| `CopyWebpackPlugin` | 复制静态资源到输出目录 |
| `OptimizeCssAssetsWebpackPlugin` | 压缩 CSS |

**Plugin 使用示例：**

```js
const HtmlWebpackPlugin = require('html-webpack-plugin');
const MiniCssExtractPlugin = require('mini-css-extract-plugin');

plugins: [
  new HtmlWebpackPlugin({
    template: './public/index.html',
    filename: 'index.html',
    minify: {
      collapseWhitespace: true,
      removeComments: true
    }
  }),
  new MiniCssExtractPlugin({
    filename: 'css/[name].[contenthash:8].css'
  })
]
```

#### Loader vs Plugin 对比总结

| 维度 | Loader | Plugin |
|------|--------|--------|
| **关注点** | 文件内容转换 | 构建流程扩展 |
| **作用对象** | 单个模块 | 整个构建过程 |
| **执行时机** | 模块加载时 | 生命周期各个阶段 |
| **本质** | 函数 | 类（含 apply 方法） |
| **典型场景** | 转译代码、处理样式 | 生成 HTML、压缩代码、资源优化 |

**一句话总结：**
- **Loader** 负责"把某类文件变成 webpack 能处理的模块"
- **Plugin** 负责"在编译的某个时机做一件事"

---

### 3. Chunk / Bundle / Module

这三个概念容易混淆，理解它们的区别对掌握 webpack 至关重要。

#### Module（模块）
**定义：** 源码中的单个文件，是构建的最小单元。

**什么是模块：**
- 一个 JavaScript 文件（ESM/CJS/AMD）
- 一个 CSS 文件
- 一张图片
- 一个 Vue/React 组件文件

**Module 的特点：**
- 每个模块都有独立的作用域
- 可以通过 `import` 或 `require` 引用其他模块
- webpack 会递归解析所有模块的依赖关系

#### Chunk（代码块）
**定义：** 若干模块按规则组合在一起形成的代码块，是构建过程中的中间产物。

**Chunk 的来源：**
1. **入口 Chunk：** 从 entry 配置产生的 chunk
2. **动态导入 Chunk：** 通过 `import()` 动态导入产生的 chunk
3. **SplitChunks：** 通过 SplitChunksPlugin 拆分出的公共 chunk
4. **Runtime Chunk：** webpack 运行时代码单独拆分的 chunk

**Chunk 的作用：**
- 实现代码分割，按需加载
- 优化缓存策略
- 提升首屏加载速度

#### Bundle（打包文件）
**定义：** 最终输出到磁盘的文件，是一个或多个 Chunk 打包后的结果。

**Chunk vs Bundle 的关系：**
- 通常一个 Chunk 对应一个 Bundle
- 但有时多个 Chunk 可能合并成一个 Bundle
- Bundle 是最终产物，Chunk 是中间过程

**图解示例：**

```
源码文件（Module）
├── main.js        ┐
├── utils.js       ├──→ Chunk: main  ──→ Bundle: main.[hash].js
├── component.js   ┘
├── lodash.js      ──→ Chunk: vendors ──→ Bundle: vendors.[hash].js
└── chart.js       ──→ Chunk: chart   ──→ Bundle: chart.[hash].js (动态导入)
```

---

### 4. Compiler / Compilation

这两个对象是 webpack 插件开发的核心，理解它们对深入理解 webpack 很重要。

#### Compiler（编译器）
**定义：** webpack 运行期的全局唯一实例，管理整个构建流程。

**特点：**
- 每次启动 webpack 只创建一个 Compiler 实例
- 包含完整的 webpack 配置
- 管理所有 Compilation
- 提供生命周期钩子供 Plugin 订阅

**Compiler 常用钩子：**
- `entryOption`：处理完入口配置后
- `beforeRun`：开始编译前
- `run`：开始读取 records 时
- `emit`：输出资源前（最后一个可以修改输出的钩子）
- `done`：编译完成时

#### Compilation（编译上下文）
**定义：** 一次完整构建过程的上下文对象，每次重新编译都会创建新的实例。

**特点：**
- 包含当前构建的所有模块、依赖、输出资源
- 开发环境下，每次文件修改都会触发新的 Compilation
- 提供模块级别的钩子

**Compilation 常用钩子：**
- `buildModule`：开始构建模块时
- `succeedModule`：模块构建成功时
- `optimize`：优化阶段开始时
- `seal`：开始封装输出时

**Compiler vs Compilation 的关系：**

```
webpack 启动
    ↓
创建 Compiler（唯一实例）
    ↓
    ├─→ 创建 Compilation #1 → 构建完成 → 输出文件
    │
    │  (开发模式下，文件修改)
    │
    ├─→ 创建 Compilation #2 → 构建完成 → 输出文件
    │
    └─→ ...
```

**为什么需要两个对象？**
- Compiler 管理全局配置和流程
- Compilation 关注单次构建的具体内容
- 开发环境下，Compiler 保留，Compilation 不断重建，性能更好

---

这四个核心概念构成了 webpack 的骨架，掌握它们是深入理解 webpack 的第一步。

## 三、Webpack 构建流程

理解 webpack 的完整构建流程，对于排查问题、优化性能和编写自定义 Plugin 都至关重要。

---

### 完整构建流程图

```
┌─────────────────────────────────────────────────────────────────┐
│  1. 初始化阶段                                                    │
│  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐      │
│  │  读取配置    │ → │  创建Compiler │ → │  注册Plugin  │      │
│  └──────────────┘   └──────────────┘   └──────────────┘      │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  2. 编译阶段                                                      │
│  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐      │
│  │  开始编译    │ → │  解析入口    │ → │  构建模块    │      │
│  └──────────────┘   └──────────────┘   └──────────────┘      │
│                        ↓                  ↓                      │
│                  递归解析依赖      Loader 转换                    │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  3. 封装阶段                                                      │
│  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐      │
│  │  构建依赖图  │ → │  生成Chunk   │ → │  优化Chunk   │      │
│  └──────────────┘   └──────────────┘   └──────────────┘      │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  4. 输出阶段                                                      │
│  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐      │
│  │  Template    │ → │  生成资源    │ → │  输出文件    │      │
│  └──────────────┘   └──────────────┘   └──────────────┘      │
└─────────────────────────────────────────────────────────────────┘
```

---

### 详细步骤说明

#### 第一步：初始化阶段

**1.1 读取并合并配置**
- 读取 `webpack.config.js`（或指定的配置文件）
- 合并命令行参数和配置文件
- 应用默认配置

**1.2 创建 Compiler 实例**
```js
const webpack = require('webpack');
const config = require('./webpack.config.js');
const compiler = webpack(config);
```
- Compiler 是全局唯一实例
- 包含完整配置信息
- 提供生命周期钩子

**1.3 注册所有 Plugin**
- 遍历 `plugins` 数组
- 调用每个 Plugin 的 `apply(compiler)` 方法
- Plugin 在此时订阅需要的钩子

---

#### 第二步：编译阶段（Compilation）

**2.1 触发 make 钩子**
- 开始一次新的编译
- 创建 Compilation 对象

**2.2 从 Entry 入口开始解析**
```js
// entry: './src/main.js'
```
- 读取入口文件内容
- 识别模块类型
- 确定使用哪些 Loader

**2.3 构建模块（Build Module）**

这是最核心的步骤，包含：

**a) 模块解析**
- 根据模块路径找到文件位置
- 解析别名（alias）
- 处理扩展名（extensions）

**b) Loader 转换**
```
原始文件 → Loader1 → Loader2 → ... → 最终 JS 模块
```
- 按规则匹配 `module.rules`
- 从右到左执行 Loader 链
- 每个 Loader 接收上一个的输出

**c) 解析 AST，收集依赖**
- 将转换后的代码解析成 AST（抽象语法树）
- 遍历 AST 找到 `import`、`require` 等依赖声明
- 记录所有依赖模块

**d) 递归构建依赖模块**
- 对每个依赖模块重复上述步骤
- 直到所有依赖都处理完毕
- 形成完整的依赖图

---

#### 第三步：封装阶段（Seal）

**3.1 构建依赖图（Dependency Graph）**
- 所有模块及其依赖关系已确定
- 记录模块间的导入导出关系

**3.2 生成 Chunk**
- 根据入口和动态导入划分 Chunk
- 应用 `splitChunks` 配置拆分公共模块
- 确定每个 Chunk 包含哪些模块

**Chunk 分组规则：**
- 每个入口生成一个初始 Chunk
- 每个 `import()` 生成一个异步 Chunk
- 公共依赖可能被提取到单独的 Chunk

**3.3 优化 Chunk**
- Tree Shaking：移除未使用的代码
- Scope Hoisting：作用域提升，减少闭包
- 模块合并：将小模块合并

---

#### 第四步：输出阶段（Emit）

**4.1 模板渲染（Template）**
- 为每个 Chunk 生成最终代码
- 注入 webpack 运行时代码
- 处理模块间的依赖关系

**4.2 生成资源（Assets）**
- JS Bundle：主代码 + 运行时
- CSS 文件（如果提取）
- SourceMap 文件
- 其他静态资源

**4.3 输出到文件系统**
- 触发 `emit` 钩子（最后修改机会）
- 将所有资源写入 `output.path` 目录
- 触发 `done` 钩子，构建完成

---

### 关键钩子时机

| 阶段 | 钩子 | 说明 | Plugin 可做的事 |
|------|------|------|----------------|
| 初始化 | `environment` | 环境准备阶段 | 修改环境变量 |
| 初始化 | `afterPlugins` | Plugin 注册完成 | 可以注册更多 Plugin |
| 编译前 | `beforeCompile` | 编译开始前 | 修改编译参数 |
| 编译中 | `compilation` | Compilation 创建 | 订阅 Compilation 钩子 |
| 编译中 | `buildModule` | 开始构建模块 | 修改模块内容 |
| 编译中 | `succeedModule` | 模块构建成功 | 处理构建完成的模块 |
| 封装前 | `seal` | 开始封装 | 优化 Chunk |
| 输出前 | `emit` | 输出资源前 | 修改或添加输出资源 |
| 完成 | `done` | 编译完成 | 统计构建信息 |

---

### 构建流程示例追踪

假设项目结构：
```
src/
├── main.js      (入口)
├── utils.js
├── component.js
└── style.css
```

**执行流程：**
1. 读取配置，创建 Compiler
2. Compiler 触发 `make` 钩子
3. 创建 Compilation，开始处理 `main.js`
4. 解析 `main.js`，发现依赖 `utils.js`、`component.js`、`style.css`
5. 递归处理每个依赖：
   - `utils.js`：直接是 JS，无需 Loader
   - `component.js`：直接是 JS
   - `style.css`：经过 `css-loader` → `style-loader` 处理
6. 所有模块处理完成，形成依赖图
7. 生成 Chunk（默认所有模块在一个 Chunk）
8. 优化 Chunk（压缩、Tree Shaking）
9. 输出 `main.[hash].js` 到 dist 目录
10. 触发 `done`，构建完成

---

理解这个完整流程，你就能明白：
- 为什么需要 Loader 和 Plugin
- Plugin 为什么能在不同阶段做事
- 如何排查构建问题
- 如何优化构建性能

## 四、webpack5 配置骨架详解

这是一个完整的 webpack5 配置示例，每个配置项都有其重要作用。让我们逐一详解。

---

### 完整配置代码

```js
// webpack.config.js
const path = require('path');
const HtmlWebpackPlugin = require('html-webpack-plugin');

module.exports = {
  mode: process.env.NODE_ENV || 'development',
  entry: {
    app: './src/main.js'
  },
  output: {
    path: path.resolve(__dirname, 'dist'),
    filename: 'js/[name].[contenthash:8].js',
    chunkFilename: 'js/[name].[contenthash:8].chunk.js',
    clean: true,
    publicPath: '/'
  },
  devtool: 'source-map',
  module: {
    rules: [
      {
        test: /\.css$/i,
        use: ['style-loader', 'css-loader']
      },
      {
        test: /\.(png|jpe?g|gif|svg)$/i,
        type: 'asset',
        parser: {
          dataUrlCondition: {
            maxSize: 8 * 1024
          }
        },
        generator: {
          filename: 'img/[name].[contenthash:8][ext]'
        }
      },
      {
        test: /\.m?js$/,
        exclude: /node_modules/,
        use: {
          loader: 'babel-loader',
          options: {
            presets: ['@babel/preset-env']
          }
        }
      }
    ]
  },
  plugins: [
    new HtmlWebpackPlugin({
      template: './public/index.html'
    })
  ],
  optimization: {
    splitChunks: {
      chunks: 'all'
    },
    runtimeChunk: 'single'
  },
  devServer: {
    port: 3000,
    open: true,
    hot: true,
    historyApiFallback: true
  }
};
```

---

### 配置项逐行详解

#### 1. 基础配置

```js
const path = require('path');
const HtmlWebpackPlugin = require('html-webpack-plugin');
```

**说明：**
- `path`：Node.js 内置模块，用于处理文件路径
- `HtmlWebpackPlugin`：自动生成 HTML 文件的插件

---

#### 2. mode（构建模式）

```js
mode: process.env.NODE_ENV || 'development',
```

**详解：**
- `process.env.NODE_ENV`：从环境变量读取模式
- 默认值：`'development'`（开发模式）
- 可选值：
  - `'development'`：开启调试、不压缩、快速构建
  - `'production'`：启用所有优化、压缩代码、适合部署
  - `'none'`：不使用任何默认优化

**使用建议：**
```json
// package.json
{
  "scripts": {
    "dev": "NODE_ENV=development webpack serve",
    "build": "NODE_ENV=production webpack"
  }
}
```

---

#### 3. entry（入口配置）

```js
entry: {
  app: './src/main.js'
},
```

**详解：**
- 对象形式的配置，key 为 chunk 名称，value 为入口文件路径
- `app`：chunk 名称，会影响输出文件名
- `'./src/main.js'`：入口文件相对路径

**其他配置方式：**

```js
// 方式一：字符串（单入口）
entry: './src/main.js'

// 方式二：数组（多文件合并为一个入口）
entry: ['./src/polyfill.js', './src/main.js']

// 方式三：对象（多入口，多页面应用）
entry: {
  app: './src/app.js',
  admin: './src/admin.js'
}
```

---

#### 4. output（输出配置）

```js
output: {
  path: path.resolve(__dirname, 'dist'),
  filename: 'js/[name].[contenthash:8].js',
  chunkFilename: 'js/[name].[contenthash:8].chunk.js',
  clean: true,
  publicPath: '/'
},
```

**逐行详解：**

**a) path：输出目录**
```js
path: path.resolve(__dirname, 'dist'),
```
- `__dirname`：当前配置文件所在目录
- `path.resolve()`：生成绝对路径（webpack 要求必须是绝对路径）
- 输出目录：`项目根目录/dist`

**b) filename：主 bundle 文件名**
```js
filename: 'js/[name].[contenthash:8].js',
```
- 输出到 `dist/js/` 目录
- `[name]`：占位符，对应 entry 中的 key（这里是 `app`）
- `[contenthash:8]`：基于文件内容生成 8 位哈希值（内容不变哈希不变）
- 最终文件名示例：`app.abc123def.js`

**c) chunkFilename：非入口 chunk 文件名**
```js
chunkFilename: 'js/[name].[contenthash:8].chunk.js',
```
- 用于动态导入（`import()`）和 splitChunks 产生的 chunk
- 最终文件名示例：`chart.xyz789.chunk.js`

**d) clean：自动清理输出目录**
```js
clean: true,
```
- **webpack5 新增**：每次构建前自动清理 `dist` 目录
- 替代了旧版本的 `clean-webpack-plugin`
- 避免旧文件堆积

**e) publicPath：资源访问路径**
```js
publicPath: '/'
```
- 浏览器访问资源时的基础路径
- 本地开发：`'/'` 或 `''`
- CDN 部署：`'https://cdn.example.com/'`
- 影响 HTML 中 script 标签的 src 和 CSS 中图片路径

---

#### 5. devtool（SourceMap 配置）

```js
devtool: 'source-map',
```

**详解：**
- 控制是否生成以及如何生成 SourceMap
- `'source-map'`：生成完整的 `.map` 文件，适合生产环境
- 更多选项见第五部分「SourceMap」详细说明

---

#### 6. module.rules（Loader 规则）

```js
module: {
  rules: [
    // ...
  ]
},
```

`rules` 是一个数组，每个元素是一个规则对象，包含：

| 字段 | 作用 |
|------|------|
| `test` | 匹配文件的正则表达式 |
| `use` | 使用的 Loader（数组或字符串） |
| `exclude` | 排除的文件（正则或路径） |
| `include` | 只处理的文件（正则或路径） |

**规则一：处理 CSS 文件**

```js
{
  test: /\.css$/i,
  use: ['style-loader', 'css-loader']
},
```

**详解：**
- `test: /\.css$/i`：匹配所有 `.css` 文件（i 表示忽略大小写）
- `use: ['style-loader', 'css-loader']`：Loader 链
  - 执行顺序：**从右到左**
  - 第一步：`css-loader` 解析 CSS 中的 `@import` 和 `url()`
  - 第二步：`style-loader` 将 CSS 注入到 DOM 的 `<style>` 标签中

**生产环境优化：** 使用 `MiniCssExtractPlugin` 提取 CSS 为独立文件

**规则二：处理图片资源（Asset Modules）**

```js
{
  test: /\.(png|jpe?g|gif|svg)$/i,
  type: 'asset',
  parser: {
    dataUrlCondition: {
      maxSize: 8 * 1024
    }
  },
  generator: {
    filename: 'img/[name].[contenthash:8][ext]'
  }
},
```

**详解（webpack5 新特性）：**

**a) type：资源类型**
- `'asset'`：自动选择（默认）
  - 小于 `maxSize`：内联为 Data URL
  - 大于 `maxSize`：输出为独立文件
- `'asset/resource'`：总是输出为独立文件（替代 file-loader）
- `'asset/inline'`：总是内联为 Data URL（替代 url-loader）
- `'asset/source'`：导出文件源码（替代 raw-loader）

**b) parser.dataUrlCondition.maxSize**
```js
maxSize: 8 * 1024  // 8KB
```
- 小于 8KB 的图片内联为 Data URL（减少 HTTP 请求）
- 大于 8KB 的图片输出为独立文件（利用浏览器缓存）

**c) generator.filename**
```js
filename: 'img/[name].[contenthash:8][ext]'
```
- 输出到 `dist/img/` 目录
- `[ext]`：原文件扩展名（带点，如 `.png`）
- 最终文件名示例：`logo.abc123def.png`

**规则三：处理 JavaScript 文件（Babel）**

```js
{
  test: /\.m?js$/,
  exclude: /node_modules/,
  use: {
    loader: 'babel-loader',
    options: {
      presets: ['@babel/preset-env']
    }
  }
}
```

**详解：**

**a) test：匹配 JS 文件**
```js
test: /\.m?js$/,
```
- 匹配 `.js` 和 `.mjs` 文件

**b) exclude：排除 node_modules**
```js
exclude: /node_modules/,
```
- 不处理 `node_modules` 中的文件
- 原因：第三方库通常已经转译过，无需重复处理
- 大幅提升构建速度

**c) babel-loader 配置**
```js
loader: 'babel-loader',
options: {
  presets: ['@babel/preset-env']
}
```
- `babel-loader`：调用 Babel 转译 JS
- `@babel/preset-env`：根据目标浏览器自动转译 ES6+

**更好的做法：** 将 Babel 配置独立到 `babel.config.js`

```js
// babel.config.js
module.exports = {
  presets: [
    ['@babel/preset-env', {
      targets: '> 0.25%, not dead',
      useBuiltIns: 'usage',
      corejs: 3
    }]
  ]
};
```

---

#### 7. plugins（插件配置）

```js
plugins: [
  new HtmlWebpackPlugin({
    template: './public/index.html'
  })
],
```

**HtmlWebpackPlugin 详解：**
- 自动生成 HTML 文件
- 自动注入所有生成的 bundle（script、link 标签）
- `template`：使用自定义 HTML 模板

**模板文件示例（`public/index.html`）：**

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>我的应用</title>
</head>
<body>
  <div id="app"></div>
</body>
</html>
```

**更多配置选项：**

```js
new HtmlWebpackPlugin({
  template: './public/index.html',
  filename: 'index.html',
  title: 'Webpack App',
  minify: {
    collapseWhitespace: true,    // 压缩 HTML
    removeComments: true        // 删除注释
  }
})
```

---

#### 8. optimization（优化配置）

```js
optimization: {
  splitChunks: {
    chunks: 'all'
  },
  runtimeChunk: 'single'
},
```

**详解：**

**a) splitChunks：代码分割**

```js
splitChunks: {
  chunks: 'all'
}
```

- `chunks: 'all'`：对所有类型的 chunk 进行分割（同步和异步）
- 自动提取第三方库（如 `react`、`lodash`）到单独的 chunk
- 作用：
  - 业务代码和第三方库分离
  - 第三方库可以长期缓存
  - 减少单个 bundle 体积

**默认分割规则：**
- 至少被 2 个 chunk 引用
- 来自 `node_modules`
- 体积超过 30KB（压缩前）

**b) runtimeChunk：运行时代码分离**

```js
runtimeChunk: 'single'
```

- `'single'`：将 webpack 运行时代码提取到单独的 chunk
- 作用：
  - 运行时代码频繁变化，单独提取避免影响其他 chunk 的缓存
  - 业务代码和第三方库不变时，只有 runtime chunk 变化
- 配合 `contenthash` 使用，实现最佳缓存策略

---

#### 9. devServer（开发服务器配置）

```js
devServer: {
  port: 3000,
  open: true,
  hot: true,
  historyApiFallback: true
}
```

**详解：**

**a) port：端口号**
```js
port: 3000,
```
- 开发服务器监听端口
- 访问地址：`http://localhost:3000`

**b) open：自动打开浏览器**
```js
open: true,
```
- 启动服务器后自动在默认浏览器打开页面

**c) hot：开启 HMR（热模块替换）**
```js
hot: true,
```
- 模块更新时只替换变更模块，不刷新整个页面
- 保留应用状态，提升开发体验

**d) historyApiFallback：SPA 路由支持**
```js
historyApiFallback: true,
```
- 单页应用（SPA）使用 HTML5 History API 路由时
- 任意 404 响应都返回 `index.html`
- 解决刷新页面 404 问题

**更多常用配置：**

```js
devServer: {
  port: 3000,
  open: true,
  hot: true,
  historyApiFallback: true,
  proxy: {
    '/api': {
      target: 'http://localhost:8080',
      changeOrigin: true,
      pathRewrite: { '^/api': '' }
    }
  },
  static: {
    directory: path.join(__dirname, 'public')
  },
  client: {
    overlay: {
      errors: true,
      warnings: false
    }
  }
}
```

---

### webpack5 重要改进说明

1. **`output.clean`**：可替代老方案的 `clean-webpack-plugin`，内置自动清理功能。
2. **Asset Modules**：webpack5 默认支持，可完全替代 `file-loader/url-loader/raw-loader`，配置更简洁。
3. **持久化缓存**：通过 `cache: { type: 'filesystem' }` 启用，大幅提升二次构建速度。
4. **`splitChunks + runtimeChunk`**：长缓存最佳实践组合，实现最优缓存策略。
5. **更好的 Tree Shaking**：对嵌套模块和副作用标记支持更好。

## 五、SourceMap 详解

SourceMap 是一个信息文件，它将构建后的代码位置映射回原始源代码的位置，让我们能够在调试或排错时看到源码，而不是混淆压缩后的代码。

---

### 为什么需要 SourceMap？

**构建前后的代码对比：**

```js
// 源码
const multiply = (a, b) => a * b;
const result = multiply(2, 3);
console.log(result);
```

```js
// 构建压缩后
!function(e){var t=function(n,r){return n*r},n=t(2,3);console.log(n)}();
```

**问题：**
- 压缩后的代码没有变量名、没有换行，难以阅读
- 报错时的行号列号与源码完全对应不上
- 调试时根本不知道哪里出了问题

**SourceMap 的作用：**
- 记录压缩后代码与源码的映射关系
- 浏览器能自动还原出源码进行调试
- 错误堆栈能正确指向源码位置

---

### SourceMap 工作原理

SourceMap 本质是一个 JSON 文件，包含以下信息：

```json
{
  "version": 3,
  "file": "bundle.js",
  "sourceRoot": "",
  "sources": ["main.js", "utils.js"],
  "sourcesContent": ["源码内容...", "源码内容..."],
  "names": ["multiply", "result", "console", "log"],
  "mappings": "AAAA,SAASA,SAASC,CAACC,CAAD,CAAQC,CAAR,CACb,OAAOC,CAACC,CAAD,CAAQC,CAAR,CADa,CAGvB,SAASC,MAAMC,SAASC,CAACC,CAAD,CAAQC,CAAR,CACtB"
}
```

**关键字段说明：**

| 字段 | 说明 |
|------|------|
| `version` | SourceMap 版本，目前是 3 |
| `sources` | 原始源文件列表 |
| `sourcesContent` | 原始源文件内容（可选） |
| `names` | 变量名和函数名列表 |
| `mappings` | 核心映射信息（VLQ 编码） |

**浏览器如何使用：**
1. 下载构建后的 JS 文件
2. 检测到 `//# sourceMappingURL=bundle.js.map`
3. 下载对应的 .map 文件
4. 解析映射关系，在 DevTools 中显示源码

---

### devtool 配置选项详解

webpack 提供了多种 SourceMap 生成策略，通过 `devtool` 配置。

**常见配置对比：**

| devtool | 构建速度 | 重新构建速度 | 生产可用 | 质量 | 说明 |
|---------|----------|--------------|----------|------|------|
| `(none)` | 最快 | 最快 | 是 | 无 | 不生成 SourceMap |
| `eval` | 最快 | 最快 | 否 | 生成后的代码 | 每个模块用 `eval()` 包裹，带 `sourceURL` |
| `eval-cheap-source-map` | 快 | 快 | 否 | 转换后的代码（无列） | 基于 eval，只映射行 |
| `eval-cheap-module-source-map` | 中等 | 快 | 否 | 源码（无列） | 基于 eval，映射到源码，只映射行 |
| `cheap-source-map` | 慢 | 中等 | 否 | 转换后的代码（无列） | 单独 .map 文件，只映射行 |
| `cheap-module-source-map` | 慢 | 中等 | 否 | 源码（无列） | 单独 .map 文件，映射到源码，只映射行 |
| `source-map` | 最慢 | 最慢 | 是 | 源码（完整） | 单独 .map 文件，完整映射 |
| `hidden-source-map` | 最慢 | 最慢 | 是 | 源码（完整） | 生成 .map 文件，但不注入引用 |
| `nosources-source-map` | 最慢 | 最慢 | 是 | 源码（无内容） | 生成 .map 文件，但不包含源码内容 |

**命名规则说明：**

- `eval`：使用 `eval()` 包裹模块代码
- `cheap`：只映射行，不映射列（更快）
- `module`：映射到 loader 处理前的源码
- `source-map`：生成单独的 .map 文件
- `hidden`：生成 .map 文件但不引用
- `nosources`：不包含源码内容

---

### 推荐配置方案

#### 开发环境

```js
devtool: 'eval-cheap-module-source-map'
```

**理由：**
- 构建速度快
- 重新构建更快（HMR 友好）
- 调试时能看到源码
- 不映射列，性能更好

**进阶选择（需要更好的调试体验）：**
```js
devtool: 'eval-source-map'
```

#### 生产环境

**方案一：完整 SourceMap（推荐用于有错误监控平台）**

```js
devtool: 'source-map'
```

**使用场景：**
- 配合 Sentry 等错误监控平台
- 需要在线上精准定位问题
- SourceMap 文件不对外公开（可放在内网或删除）

**方案二：隐藏 SourceMap（安全优先）**

```js
devtool: 'hidden-source-map'
```

**使用场景：**
- 生成 .map 文件但不注入 `sourceMappingURL`
- 浏览器不会自动下载
- 可以手动上传到 Sentry 用于错误分析

**方案三：无 SourceMap（极简）**

```js
devtool: false
```

**使用场景：**
- 对代码保密性要求极高
- 不需要线上错误定位
- 减少构建产物体积

---

### SourceMap 安全注意事项

**风险：**
- 如果 SourceMap 对外公开，任何人都能看到你的源码
- 可能泄露敏感逻辑、API 密钥、注释等

**最佳实践：**

1. **生产环境不要暴露 SourceMap**
   ```js
   devtool: 'hidden-source-map'
   ```

2. **将 .map 文件上传到错误监控平台（如 Sentry）**
   - 只有平台能访问
   - 普通用户看不到

3. **构建后删除 .map 文件**
   ```bash
   # 如果不需要线上排错
   rm dist/*.map
   ```

4. **使用 `nosources-source-map`**
   - 保留堆栈映射
   - 不暴露源码内容

---

### 实际使用示例

```js
// webpack.config.js
module.exports = {
  devtool: process.env.NODE_ENV === 'production'
    ? 'hidden-source-map'  // 生产环境
    : 'eval-cheap-module-source-map'  // 开发环境
}
```

```json
// package.json
{
  "scripts": {
    "dev": "webpack serve --mode development",
    "build": "webpack --mode production"
  }
}
```

## 六、代码分割与懒加载详解

代码分割是 webpack 最强大的特性之一，它可以将代码拆分成多个 bundle，实现按需加载，大幅提升应用性能。

---

### 为什么需要代码分割？

**问题场景：**
```
一个单页应用包含：
- 首页代码（100KB）
- 图表模块（500KB）
- 管理后台（1MB）
- 其他模块（400KB）
总计：2MB
```

**不做代码分割的问题：**
- 用户访问首页需要下载全部 2MB 代码
- 首屏加载慢，用户体验差
- 浪费带宽和流量

**代码分割后的效果：**
- 首页只下载 100KB，首屏快
- 点击图表按钮时再下载 500KB
- 进入管理后台时再下载 1MB
- 用户按需加载，体验大幅提升

---

### 1. 代码分割的三种常见方式

#### 方式一：多入口配置（Entry Points）

**适用场景：** 多页面应用（MPA），每个页面有独立的入口。

```js
// webpack.config.js
module.exports = {
  entry: {
    home: './src/home.js',
    about: './src/about.js',
    contact: './src/contact.js'
  },
  output: {
    filename: '[name].[contenthash:8].js'
  }
};
```

**输出结果：**
```
dist/
├── home.abc123.js
├── about.def456.js
└── contact.ghi789.js
```

**优点：**
- 配置简单，天然分割
- 适合传统多页面应用

**缺点：**
- 如果多个入口共享代码，会重复打包
- 需要配合 `splitChunks` 提取公共代码

---

#### 方式二：动态导入（Dynamic Imports）

**适用场景：** 单页面应用（SPA），按需加载路由或功能模块。

**ES6 动态导入语法：**
```js
// 普通导入（同步）
import { renderChart } from './chart';

// 动态导入（异步，返回 Promise）
import('./chart').then(module => {
  const { renderChart } = module;
  renderChart();
});
```

**async/await 写法：**
```js
button.onclick = async () => {
  const { default: renderChart } = await import('./chart');
  renderChart();
};
```

**React 路由懒加载示例：**
```jsx
import { lazy, Suspense } from 'react';

const Home = lazy(() => import('./Home'));
const About = lazy(() => import('./About'));
const Chart = lazy(() => import('./Chart'));

function App() {
  return (
    <Suspense fallback={<div>Loading...</div>}>
      <Routes>
        <Route path="/" element={<Home />} />
        <Route path="/about" element={<About />} />
        <Route path="/chart" element={<Chart />} />
      </Routes>
    </Suspense>
  );
}
```

**Vue 路由懒加载示例：**
```js
const routes = [
  {
    path: '/',
    component: () => import('./Home.vue')
  },
  {
    path: '/chart',
    component: () => import('./Chart.vue')
  }
];
```

**命名 Chunk（webpackChunkName）：**
```js
import(/* webpackChunkName: "chart" */ './chart');
```

输出文件名：`chart.[contenthash:8].chunk.js`，而不是随机 ID。

**预加载（Prefetch/Preload）：**
```js
// Preload：提前加载当前页面可能用到的资源
import(/* webpackPreload: true */ './chart');

// Prefetch：预加载未来可能用到的资源
import(/* webpackPrefetch: true */ './chart');
```

---

#### 方式三：SplitChunksPlugin（内置优化）

**适用场景：** 自动提取公共代码和第三方库。

**默认配置（webpack5）：**
```js
module.exports = {
  optimization: {
    splitChunks: {
      chunks: 'all',  // 对所有 chunk 生效
      minSize: 20000,  // 最小 20KB 才分割
      minRemainingSize: 0,
      minChunks: 1,  // 至少被引用 1 次
      maxAsyncRequests: 30,  // 最大异步请求数
      maxInitialRequests: 30,  // 最大初始请求数
      enforceSizeThreshold: 50000,
      cacheGroups: {
        defaultVendors: {
          test: /[\\/]node_modules[\\/]/,
          priority: -10,
          reuseExistingChunk: true
        },
        default: {
          minChunks: 2,
          priority: -20,
          reuseExistingChunk: true
        }
      }
    }
  }
};
```

**核心配置说明：**

| 配置项 | 说明 |
|--------|------|
| `chunks: 'all'` | 对同步和异步 chunk 都生效 |
| `minSize` | chunk 最小体积（字节），小于则不分割 |
| `minChunks` | 至少被多少个 chunk 引用才分割 |
| `cacheGroups` | 缓存组，定义分割规则 |

**自定义 cacheGroups 示例：**
```js
splitChunks: {
  chunks: 'all',
  cacheGroups: {
    // 提取 React 相关
    react: {
      test: /[\\/]node_modules[\\/](react|react-dom)[\\/]/,
      name: 'react',
      chunks: 'all',
      priority: 20
    },
    // 提取其他第三方库
    vendors: {
      test: /[\\/]node_modules[\\/]/,
      name: 'vendors',
      chunks: 'all',
      priority: 10
    },
    // 提取公共业务代码
    common: {
      name: 'common',
      minChunks: 2,
      chunks: 'all',
      priority: 5
    }
  }
}
```

---

### 2. 懒加载完整示例

**项目结构：**
```
src/
├── main.js
├── Home.js
├── Chart.js
└── utils.js
```

**main.js（入口）：**
```js
import Home from './Home';

// 渲染首页
Home.render();

// 点击按钮时加载图表
document.getElementById('loadChart').addEventListener('click', async () => {
  const { default: Chart } = await import(
    /* webpackChunkName: "chart" */
    './Chart'
  );
  Chart.render();
});
```

**Chart.js（大模块）：**
```js
import * as echarts from 'echarts';  // 500KB+

export default {
  render() {
    const chart = echarts.init(document.getElementById('chart'));
    chart.setOption({
      // 图表配置...
    });
  }
};
```

**网络请求时序：**
```
1. 页面加载 → 下载 main.js (100KB) → 首页显示
2. 用户点击按钮 → 触发 import()
3. 下载 chart.chunk.js (500KB)
4. 图表渲染完成
```

---

### 代码分割最佳实践

1. **按路由分割**：每个路由一个 chunk
2. **按功能分割**：大功能模块独立分割
3. **提取第三方库**：React、Vue、Lodash 等单独打包
4. **提取公共代码**：多个页面共享的代码单独打包
5. **合理设置 minSize**：避免产生过多小 chunk
6. **使用 webpackChunkName**：让 chunk 名称更易读

---

### 验证代码分割效果

**查看输出：**
```bash
npm run build
```

**输出示例：**
```
Asset                          Size
app.abc123.js                 100 KiB
react.def456.js               120 KiB
vendors.ghi789.js             300 KiB
chart.jkl012.chunk.js         500 KiB
runtime.mno345.js              5 KiB
```

**webpack-bundle-analyzer 分析：**
```js
const BundleAnalyzerPlugin = require('webpack-bundle-analyzer').BundleAnalyzerPlugin;

module.exports = {
  plugins: [
    new BundleAnalyzerPlugin()
  ]
};
```
运行后会打开一个可视化页面，直观展示每个 bundle 的大小和内容。

## 七、缓存策略详解

缓存策略的核心目标是：用户二次访问时尽可能命中缓存，同时代码更新后又能精准失效，避免使用旧代码。

---

### 为什么需要缓存策略？

**场景说明：**

| 访问次数 | 情况 | 结果 |
|---------|------|------|
| 第 1 次 | 用户首次访问 | 下载所有资源 |
| 第 2 次 | 代码未更新 | 从缓存读取，秒开 |
| 第 3 次 | 代码已更新 | 下载变更的资源，复用未变更的 |

**没有缓存策略的问题：**
- 每次访问都下载全部资源
- 浪费带宽，用户体验差
- CDN 成本高

---

### 缓存原理：HTTP 缓存机制

浏览器缓存流程：

```
1. 首次请求
   浏览器 → 服务器 → 下载资源 → 缓存到本地

2. 再次请求
   浏览器检查缓存 → 有效？→ 使用缓存（200 from disk cache）
                    ↓ 无效
                 向服务器验证 → 未过期？→ 304 Not Modified（使用缓存）
                                      ↓ 已过期
                                   200 OK（下载新资源）
```

**关键响应头：**
- `Cache-Control: max-age=31536000, immutable`：强缓存 1 年
- `ETag` / `Last-Modified`：协商缓存验证

---

### webpack 缓存策略三件套

#### 1. 文件名哈希（contenthash）

**三种哈希对比：**

| 哈希类型 | 特点 | 适用场景 |
|---------|------|----------|
| `[hash]` | 基于整个构建，任何文件变更都会变 | 不推荐 |
| `[chunkhash]` | 基于 chunk 内容，chunk 内任一文件变更都会变 | 旧版本常用 |
| `[contenthash]` | 基于单个文件内容，只随文件内容变化 | **推荐** |

**为什么推荐 contenthash？**
```js
// 场景：只修改业务代码，不修改第三方库

使用 chunkhash：
- main.js 变更 → chunkhash 变 → vendor.js 的 chunkhash 也变 → 全部重新下载 ❌

使用 contenthash：
- main.js 变更 → main.js 的 contenthash 变
- vendor.js 不变 → vendor.js 的 contenthash 不变 → 复用缓存 ✅
```

**配置示例：**
```js
output: {
  filename: 'js/[name].[contenthash:8].js',
  chunkFilename: 'js/[name].[contenthash:8].chunk.js'
}
```

---

#### 2. 运行时代码分离（runtimeChunk）

**什么是运行时代码？**
- webpack 注入的模块加载器代码
- 包含模块 ID 映射关系
- 很小（几 KB），但频繁变化

**问题：**
```
修改业务代码 → 模块 ID 变化 → runtime 变化 → main chunk 也变化 → 缓存失效 ❌
```

**解决方案：**
```js
optimization: {
  runtimeChunk: 'single'  // 提取为单独的 chunk
}
```

**效果：**
```
修改业务代码 → runtime 变化（只有几 KB）
               main.js 不变 → 复用缓存 ✅
               vendor.js 不变 → 复用缓存 ✅
```

---

#### 3. 公共依赖拆包（splitChunks）

**问题场景：**
```
多个页面都引用了 React、Lodash → 每个 bundle 都包含 → 重复下载，缓存不共享 ❌
```

**解决方案：**
```js
optimization: {
  splitChunks: {
    chunks: 'all'  // 提取公共代码
  }
}
```

**效果：**
```
React、Lodash → 单独的 vendors chunk → 所有页面共享 → 下载一次，永久缓存 ✅
```

---

### 完整缓存配置示例

```js
module.exports = {
  output: {
    path: path.resolve(__dirname, 'dist'),
    filename: 'js/[name].[contenthash:8].js',
    chunkFilename: 'js/[name].[contenthash:8].chunk.js',
    clean: true
  },
  optimization: {
    // 1. 代码分割
    splitChunks: {
      chunks: 'all',
      cacheGroups: {
        vendors: {
          test: /[\\/]node_modules[\\/]/,
          name: 'vendors',
          priority: 10
        },
        common: {
          name: 'common',
          minChunks: 2,
          priority: 5
        }
      }
    },
    // 2. 提取 runtime
    runtimeChunk: 'single'
  }
};
```

---

### 输出结果示例

```
dist/
├── index.html
├── js/
│   ├── runtime.xyz123.js          (5KB，运行时代码)
│   ├── vendors.abc456.js           (300KB，第三方库)
│   ├── common.def789.js            (50KB，公共代码)
│   ├── main.ghi012.js              (100KB，业务代码)
│   └── chart.jkl345.chunk.js       (500KB，懒加载模块)
└── css/
    └── main.mno678.css
```

**缓存命中情况：**
- 修改业务代码：只有 `main.ghi012.js` 和 `runtime.xyz123.js` 变更
- 修改第三方库：只有 `vendors.abc456.js` 变更
- 所有未变更的文件都复用缓存

---

### Nginx 缓存配置配合

```nginx
server {
  location / {
    # HTML 文件：不缓存，每次请求最新
    add_header Cache-Control 'no-cache';
    
    # JS/CSS/图片：强缓存 1 年
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg)$ {
      expires 1y;
      add_header Cache-Control 'public, immutable';
    }
  }
}
```

---

## 八、Tree Shaking（原理 + 限制）详解

Tree Shaking（摇树优化）是一种消除未使用代码（Dead Code Elimination）的技术，它可以显著减小产物体积。

---

### 什么是 Tree Shaking？

**形象类比：**
- 你的代码是一棵树
- 使用的代码是绿叶
- 未使用的代码是枯叶
- Tree Shaking 把枯叶摇掉

**代码示例：**

```js
// utils.js
export const add = (a, b) => a + b;
export const multiply = (a, b) => a * b;
export const subtract = (a, b) => a - b;

// main.js
import { add } from './utils';
console.log(add(1, 2));
```

**Tree Shaking 后：**
```js
// utils.js 中只保留 add 函数
const add = (a, b) => a + b;
console.log(add(1, 2));
```

`multiply` 和 `subtract` 被摇掉了！

---

### Tree Shaking 原理

**为什么 Tree Shaking 能工作？**
- ESM（ES6 Modules）是静态的
- `import` 和 `export` 只能在顶层，不能在条件语句中
- 编译时就能确定哪些代码被使用

**对比 CommonJS（不能 Tree Shaking）：**
```js
// CommonJS：动态的，编译时无法确定
if (Math.random() > 0.5) {
  require('./moduleA');
} else {
  require('./moduleB');
}
```

**ESM（可以 Tree Shaking）：**
```js
// ESM：静态的，编译时就能分析
import { add } from './utils';  // 确定只导入 add
```

---

### Tree Shaking 前提条件

1. **使用 ESM 语法**
   ```js
   // ✅ ESM（推荐）
   import { foo } from './module';
   export const bar = 1;
   
   // ❌ CommonJS（无法 Tree Shaking）
   const { foo } = require('./module');
   module.exports = { bar: 1 };
   ```

2. **生产模式**
   ```js
   mode: 'production'  // 自动启用压缩和 Tree Shaking
   ```

3. **使用支持 Tree Shaking 的工具**
   - webpack 4+
   - Terser（webpack5 内置）

---

### 副作用（Side Effects）的影响

**什么是副作用？**
- 模块执行时，除了导出，还做了其他事情
- 修改全局变量、修改原型、立即执行的函数等

**有副作用的模块示例：**
```js
// utils.js
export const add = (a, b) => a + b;

// 副作用：修改了全局对象
window.PLUGIN_INITIALIZED = true;

// 副作用：立即执行了一段代码
console.log('utils.js loaded');
```

**问题：**
```js
import { add } from './utils';
```
即使只使用 `add`，也不能把整个 utils.js 摇掉，因为有副作用代码需要执行。

---

### 标记无副作用的模块

在 `package.json` 中标记：

```json
{
  "sideEffects": false
}
```

表示整个包都没有副作用，可以放心 Tree Shaking。

**部分文件有副作用：**
```json
{
  "sideEffects": [
    "*.css",
    "*.scss",
    "./src/polyfill.js"
  ]
}
```

表示 CSS 文件和 polyfill.js 有副作用，其他文件没有。

---

### 常见限制与陷阱

1. **只对 ESM 有效**
   - CommonJS 模块无法 Tree Shaking
   - 确保第三方库也提供 ESM 版本

2. **动态导入无法完全优化**
   ```js
   // 无法确定实际使用哪些导出
   import * as utils from './utils';
   ```

3. **类的方法可能不会被摇掉**
   ```js
   class Calculator {
     add() {}
     multiply() {}  // 即使没调用，也可能不会被摇掉
   }
   ```

4. **注意 Babel 配置**
   确保 Babel 不会把 ESM 转成 CommonJS：
   ```js
   // babel.config.js
   module.exports = {
     presets: [
       ['@babel/preset-env', {
         modules: false  // 不转换模块，保留 ESM
       }]
     ]
   };
   ```

---

### 验证 Tree Shaking 效果

使用 `webpack-bundle-analyzer` 查看：
```js
const BundleAnalyzerPlugin = require('webpack-bundle-analyzer').BundleAnalyzerPlugin;

module.exports = {
  plugins: [
    new BundleAnalyzerPlugin()
  ]
};
```

检查 bundle 中是否包含未使用的代码。

---

## 九、Babel 和 Webpack 的关系（高频误区）

很多人会混淆 Babel 和 webpack 的职责，这里彻底理清。

---

### 两者的核心职责

| 工具 | 定位 | 核心作用 |
|------|------|----------|
| **Babel** | JavaScript 编译器 | 把新语法转成旧语法，让代码在老浏览器运行 |
| **Webpack** | 模块打包器 | 管理模块依赖，把多个文件打包成一个或多个 bundle |

---

### 类比理解

**场景：** 你要出国旅游，需要准备行李。

- **Babel**：翻译官，把你的中文需求翻译成英文
- **Webpack**：行李箱，把所有东西打包整理好

**两者配合：**
1. Babel 把你的代码"翻译"成浏览器能懂的语言
2. Webpack 把所有"翻译好"的代码"打包"起来

---

### Babel 能做什么？

1. **语法转译**
   ```js
   // 输入（ES6+）
   const arr = [1, 2, 3];
   const doubled = arr.map(n => n * 2);
   
   // 输出（ES5）
   var arr = [1, 2, 3];
   var doubled = arr.map(function(n) {
     return n * 2;
   });
   ```

2. **Polyfill（垫片）**
   为老浏览器提供新 API：
   - `Array.prototype.includes`
   - `Promise`
   - `async/await`

3. **JSX、TypeScript 转换**
   ```jsx
   // JSX
   <div>Hello</div>
   
   // 转换后
   React.createElement('div', null, 'Hello');
   ```

---

### Webpack 能做什么？

1. **模块打包**
   ```
   main.js
   ├── utils.js
   ├── component.js
   ├── style.css
   └── logo.png
   
   ↓ webpack 打包
   
   bundle.js（包含所有依赖）
   ```

2. **代码分割**
3. **资源处理**（图片、CSS、字体等）
4. **开发服务器**
5. **热更新**

---

### 两者如何配合？

**通过 babel-loader：**

```
源码 → babel-loader → Babel 转译 → webpack 打包 → 输出
```

**配置示例：**

```js
// webpack.config.js
module.exports = {
  module: {
    rules: [
      {
        test: /\.m?js$/,
        exclude: /node_modules/,
        use: {
          loader: 'babel-loader'  // webpack 调用 babel-loader
        }
      }
    ]
  }
};
```

```js
// babel.config.js（Babel 自己的配置）
module.exports = {
  presets: [
    ['@babel/preset-env', {
      targets: '> 0.25%, not dead',
      useBuiltIns: 'usage',
      corejs: 3
    }]
  ]
};
```

---

### 常见误区

**误区一：webpack 可以转译新语法**
- webpack 本身不做语法转译
- 需要通过 babel-loader 调用 Babel

**误区二：Babel 可以打包**
- Babel 只做单个文件的转译
- 不处理模块依赖，不打包

**误区三：只用 webpack 就行**
- 如果只用 webpack，新语法在老浏览器会报错
- 必须配合 Babel

---

## 十、HMR 热更新详解

HMR（Hot Module Replacement，热模块替换）是 webpack 最实用的特性之一，它可以在不刷新页面的情况下更新模块。

---

### 什么是 HMR？

**传统开发流程：**
```
修改代码 → 重新构建 → 刷新页面 → 状态丢失 → 重新操作
```
问题：
- 每次刷新都要重新登录
- 表单数据丢失
- 调试效率低

**HMR 开发流程：**
```
修改代码 → 只更新变更的模块 → 页面不刷新 → 状态保留 ✅
```

---

### HMR 原理

**架构图：**

```
浏览器                       Webpack Dev Server
  │                              │
  │  1. 建立 WebSocket 连接       │
  │─────────────────────────────>│
  │                              │
  │  2. 文件修改                  │
  │                              │─── 检测变更
  │                              │─── 重新编译
  │                              │
  │  3. 发送更新通知              │
  │<─────────────────────────────│
  │  (hash: abc123)              │
  │                              │
  │  4. 请求 manifest.json        │
  │─────────────────────────────>│
  │                              │
  │  5. 返回变更的模块列表        │
  │<─────────────────────────────│
  │                              │
  │  6. 请求变更的模块            │
  │─────────────────────────────>│
  │                              │
  │  7. 返回新模块代码            │
  │<─────────────────────────────│
  │                              │
  │  8. 替换旧模块                │
  │─── 执行 accept 回调          │
  │─── 更新页面                  │
```

---

### HMR 配置

```js
// webpack.config.js
module.exports = {
  devServer: {
    hot: true,  // 开启 HMR
    open: true
  }
};
```

---

### 模块热替换逻辑

**手动处理 HMR：**

```js
// main.js
import { render } from './App';

render();

// HMR 接受代码
if (module.hot) {
  module.hot.accept('./App', () => {
    // App 模块更新时执行
    console.log('App updated!');
    render();
  });
}
```

**框架已内置：**
- React：`react-refresh`
- Vue：`vue-loader` 内置支持
- 大多数现代框架都有对应的 HMR 方案

---

### HMR 的限制

1. **不是所有模块都能热替换**
   - CSS 模块：通常可以
   - React/Vue 组件：通常可以
   - 工具函数：可能需要手动处理

2. **状态保留**
   - 组件内部状态可能保留
   - 全局状态可能需要额外处理

3. **回退机制**
   - 如果 HMR 失败，会自动回退到刷新页面

---

## 十一、性能优化详解

webpack 性能优化主要从两个维度入手：**构建速度**和**产物体积**。

---

### 1. 提升构建速度

#### 1.1 缩小 Loader 处理范围

```js
module: {
  rules: [
    {
      test: /\.m?js$/,
      include: path.resolve(__dirname, 'src'),  // 只处理 src 目录
      exclude: /node_modules/,                   // 排除 node_modules
      use: 'babel-loader'
    }
  ]
}
```

**效果：** 不处理第三方库，大幅减少需要转译的文件数。

---

#### 1.2 使用持久化缓存（webpack5）

```js
module.exports = {
  cache: {
    type: 'filesystem',  // 使用文件系统缓存
    buildDependencies: {
      config: [__filename]  // 配置文件变更时清除缓存
    }
  }
};
```

**效果：**
- 首次构建：正常速度
- 二次构建：速度提升 80%+

---

#### 1.3 多进程构建

```js
module: {
  rules: [
    {
      test: /\.m?js$/,
      use: [
        'thread-loader',  // 多进程
        'babel-loader'
      ]
    }
  ]
}
```

**注意：**
- 小项目不推荐，进程启动有开销
- 大项目（1000+ 模块）效果明显

---

#### 1.4 优化 SourceMap

| 开发环境 | 生产环境 |
|---------|---------|
| `eval-cheap-module-source-map` | `source-map` 或 `hidden-source-map` |

避免使用 `source-map` 开发环境，构建慢。

---

#### 1.5 排除不必要的插件

只在生产环境使用压缩插件：

```js
const isProd = process.env.NODE_ENV === 'production';

module.exports = {
  plugins: [
    new HtmlWebpackPlugin(),
    ...(isProd ? [new MiniCssExtractPlugin()] : [])
  ]
};
```

---

### 2. 优化产物体积

#### 2.1 Tree Shaking

见第八部分详细说明。

---

#### 2.2 代码分割与按需加载

见第六部分详细说明。

---

#### 2.3 压缩资源

**JS 压缩（webpack5 内置）：**
```js
module.exports = {
  optimization: {
    minimize: true,
    minimizer: [
      `...`,  // 使用默认的 TerserPlugin
      new CssMinimizerPlugin()
    ]
  }
};
```

**CSS 压缩：**
```js
const CssMinimizerPlugin = require('css-minimizer-webpack-plugin');

module.exports = {
  optimization: {
    minimizer: [
      `...`,
      new CssMinimizerPlugin()
    ]
  }
};
```

**图片压缩：**
使用 `image-webpack-loader` 或 `sharp-loader`。

---

#### 2.4 拆分第三方依赖

```js
optimization: {
  splitChunks: {
    chunks: 'all',
    cacheGroups: {
      react: {
        test: /[\\/]node_modules[\\/](react|react-dom)[\\/]/,
        name: 'react',
        chunks: 'all'
      },
      vendors: {
        test: /[\\/]node_modules[\\/]/,
        name: 'vendors',
        chunks: 'all',
        priority: -10
      }
    }
  }
}
```

---

#### 2.5 按需引入第三方库

**Bad：**
```js
import _ from 'lodash';  // 引入全部，体积大
```

**Good：**
```js
import debounce from 'lodash/debounce';  // 只引入需要的
```

或使用 `babel-plugin-import` 自动按需引入。

---

### 3. 分析工具

**webpack-bundle-analyzer：**
```js
const BundleAnalyzerPlugin = require('webpack-bundle-analyzer').BundleAnalyzerPlugin;

module.exports = {
  plugins: [
    new BundleAnalyzerPlugin()
  ]
};
```

可视化分析 bundle 构成，找出大模块。

**speed-measure-webpack-plugin：**
分析各个 Loader 和 Plugin 的耗时。

---

## 十二、概念速览

### 1. Loader 和 Plugin 区别？
**Loader**：文件内容转换器，把非 JS 文件转成 JS 模块。
**Plugin**：生命周期钩子的订阅者，在构建的各个阶段执行自定义任务。

### 2. 为什么需要 `contenthash`？
让缓存粒度细化到单个文件，只有文件内容变化时哈希才变化，避免不必要的缓存失效。

### 3. `chunkhash` 和 `contenthash` 怎么选？
现代实践优先 `contenthash`，它对单文件内容变化更敏感，能实现更细粒度的缓存控制。

### 4. 为什么生产建议单独配置？
开发和生产目标冲突：
- 开发：重速度、重调试体验
- 生产：重体积、重稳定性、重缓存策略

### 5. webpack5 相比旧版本有什么明显变化？
- 内置 Asset Modules，替代 file-loader/url-loader
- 持久化缓存（`cache: { type: 'filesystem' }`），二次构建极快
- 默认优化策略更现代化
- 更好的 Tree Shaking
- Node.js polyfill 移除，需手动引入

---

## 十三、一个简化的"中级面试答案框架"

当被问"讲讲你项目里的 webpack 优化"时，可以按这个顺序回答：

### 第一步：先说目标
"我们的优化目标是：启动快、构建快、首屏快、可缓存、可排障。"

### 第二步：再说具体动作

**1. 分环境配置**
- 开发环境：开启 HMR、快速 SourceMap、不压缩
- 生产环境：开启所有优化、代码压缩、Tree Shaking

**2. 代码分割**
- 按路由分割，每个路由独立 chunk
- 动态导入大模块，按需加载
- 提取第三方库到 vendors chunk

**3. 缓存策略**
- filename 用 `[contenthash]`
- runtime 单独提取
- splitChunks 拆分公共代码
- 配合 Nginx 强缓存

**4. SourceMap 策略**
- 开发：`eval-cheap-module-source-map`
- 生产：`hidden-source-map`，上传到 Sentry

**5. 构建性能优化**
- 持久化缓存
- include/exclude 缩小 Loader 范围
- 大项目用 thread-loader

### 第三步：最后说结果
"优化后，构建耗时从 2 分钟降到 30 秒，首屏资源体积从 2MB 降到 500KB，线上错误定位效率提升了 80%。"

---

## 十四、实践建议

### 1. 开发和生产配置拆分

**项目结构：**
```
build/
├── webpack.base.js     # 基础配置
├── webpack.dev.js      # 开发环境
└── webpack.prod.js     # 生产环境
```

**使用 webpack-merge：**
```js
// webpack.dev.js
const { merge } = require('webpack-merge');
const base = require('./webpack.base');

module.exports = merge(base, {
  mode: 'development',
  devtool: 'eval-cheap-module-source-map',
  devServer: {
    hot: true,
    open: true
  }
});
```

---

### 2. 每次优化都做数据对比

| 指标 | 优化前 | 优化后 | 提升 |
|------|--------|--------|------|
| 构建时间 | 120s | 30s | 75% ↓ |
| 首屏 JS | 2MB | 500KB | 75% ↓ |
| LCP | 3.2s | 1.5s | 53% ↓ |

**关键指标：**
- 构建时长
- 首屏资源体积
- LCP（最大内容绘制）
- FCP（首次内容绘制）

---

### 3. 不要"为优化而优化"，先找瓶颈再下手

**优化步骤：**
1. 使用分析工具定位问题
2. 针对瓶颈优化
3. 验证效果
4. 有提升才保留

**反例：**
- 小项目也上 thread-loader（反而更慢）
- 为了分割而分割，产生过多小 chunk（反而影响加载）

---

## 十五、从 0 手写 Loader 和 Plugin（中级面试实战）

### 1. 手写一个简单 Loader

Loader 本质是一个接收文件内容并返回转换后内容的函数。

#### 示例：实现一个 `markdown-loader`（将 Markdown 转 HTML）

```js
// loaders/markdown-loader.js
const marked = require('marked');

module.exports = function (source) {
  // 将 Markdown 转为 HTML
  const html = marked.parse(source);
  // 返回 CommonJS 模块代码
  return `module.exports = ${JSON.stringify(html)}`;
};
```

#### 使用 Loader

```js
// webpack.config.js
module.exports = {
  module: {
    rules: [
      {
        test: /\.md$/,
        use: [
          'html-loader', // 处理 HTML 中的资源引用
          path.resolve(__dirname, 'loaders/markdown-loader.js')
        ]
      }
    ]
  }
};
```

#### Loader 核心要点

- 函数接收 `source`（源文件内容），返回转换后的字符串或 Buffer
- Loader 链从右到左执行
- 可通过 `this.callback` 或 `this.async()` 处理异步逻辑
- 可通过 `this.query` 或 `this.getOptions()` 获取 loader 配置

---

### 2. 手写一个简单 Plugin

Plugin 基于 webpack 的 Tapable 钩子系统，在构建生命周期的特定时机执行自定义逻辑。

#### 示例：实现一个 `FileListPlugin`（生成构建文件清单）

```js
// plugins/file-list-plugin.js
class FileListPlugin {
  apply(compiler) {
    // 订阅 emit 钩子（输出资源前触发）
    compiler.hooks.emit.tapAsync('FileListPlugin', (compilation, callback) => {
      // 创建文件清单内容
      let filelist = 'In this build:\n\n';

      // 遍历所有输出资源
      for (const filename in compilation.assets) {
        filelist += `- ${filename}\n`;
      }

      // 将清单添加为新的输出资源
      compilation.assets['filelist.md'] = {
        source: () => filelist,
        size: () => filelist.length
      };

      // 调用回调继续构建流程
      callback();
    });
  }
}

module.exports = FileListPlugin;
```

#### 使用 Plugin

```js
// webpack.config.js
const FileListPlugin = require('./plugins/file-list-plugin');

module.exports = {
  plugins: [
    new FileListPlugin()
  ]
};
```

#### Plugin 核心要点

- 必须是一个类，包含 `apply` 方法
- `apply` 方法接收 `compiler` 对象
- 通过 `compiler.hooks.xxx.tap` 订阅钩子
- `compilation` 对象包含当前构建的上下文
- 常见钩子：`emit`（输出前）、`done`（构建完成）、`beforeCompile`（编译前）

---

这两部分是中级面试最容易拉开差距的实战题，建议亲自手写一遍加深理解。
