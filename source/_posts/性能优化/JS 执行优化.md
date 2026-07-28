---
title: JavaScript 执行优化
categories: 
- 性能优化
tags:
- 性能优化
- JavaScript
- 代码分割
- 延迟加载
---

## 一、延迟脚本加载

### 合理放置脚本位置

`<script>` 标签会阻塞页面解析。应将其放在 `<body>` 末尾：<!--more-->

```html
<body>
  <div id="app"></div>
  <script src="app.js"></script>  <!-- 页面内容先展示，再加载脚本 -->
</body>
```

### defer 与 async

```html
<script defer src="app.js"></script>  <!-- 并行下载，DOM 解析完成后按顺序执行 -->
<script async src="analytics.js"></script>  <!-- 并行下载，下载完成后立即执行 -->
```

- `defer`：DOM 完成加载前不执行，多个 defer 脚本按顺序执行
- `async`：下载完成后立即执行，顺序无保证

### 动态添加脚本

```js
function loadScript(url, callback) {
  const script = document.createElement('script');
  script.type = 'text/javascript';
  script.onload = () => callback();
  script.src = url;
  document.head.appendChild(script);
}
```

## 二、代码分割（Code Splitting）

### 静态代码分割

```js
const getModal = () => import('./src/modal.js');
Listener.on('click', () => {
  getModal().then(module => module.initModal());
});
```

适用场景：
- 模态框、对话框、tooltip 等临时性组件
- 路由级别的代码分割
- 体积很大的第三方库

路由级代码分割（Vue）：

```js
const Home = () => import('views/home/Home.vue');
const routes = [{ path: '/home', component: Home }];
```

### 动态代码分割

```js
const getTheme = (name) => import(`./src/themes/${name}`);
getTheme('stylish').then(module => module.applyTheme());
```

适用场景：A/B Test、动态加载主题。

### 魔术注释（Webpack）

```js
import(/* webpackChunkName: "my-chunk" */ './footer');
```

配合配置：

```js
output: { filename: 'bundle.js', chunkFilename: '[name].chunk.js' }
```

## 三、逻辑后移

将主体内容的请求前移，非主体逻辑后移。

```text
优化前：
  请求头部 → 请求导航 → 请求正文 → 请求侧边栏 → 请求底部

优化后：
  请求正文 → 请求头部 → 请求导航 → 请求侧边栏 → 请求底部
```

突出主体逻辑，让用户尽快看到页面主要内容，可以极大提升感知体验。
