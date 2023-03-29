---
title: Nuxt 项目梳理
categories: 
- Nuxt
tag: 
- Nuxt
- Vue
---

在公司实习时，参与的第一个任务是负责从头搭建某医疗机构的 h5 页面端。h5 端和 pc 端的页面仅供浏览信息使用，不像 app 端有用户注册登录及一系列交互操作的页面，因此整体难度不高。因为上司没有指明要使用何种框架，所以我照搬了公司用来测试前端实习生水平的一个伪项目所用的框架——Nuxt。本文用来梳理在搭建这个 Nuxt 项目时需要了解的一些知识点。<!--more-->

### 一、Nuxt.js 介绍

传统的 web 开发（`jsp`、`php`、`asp`）中，服务端收到客户端的请求后，查询数据库，将整个 HTML 页面返回给客户端，浏览器相当于打开了一个新页面。也就是传统的`MVC`的开发。

后来如 Vue、React 等单页面应用（Single Page Application, SPA），以其优秀的用户体验逐渐成为主流。它的网页渲染过程是，客户端初次请求服务端时，服务端会返回一个只有基本结构的`HTML`。客户端收到后开始执行对应`javaScript`渲染`template`。之后当客户端再向服务端发送请求，服务端返回的都是`json`格式的数据。客户端接收数据，然后完成最终渲染。SPA 的页面整体是 JavaScript 渲染出来的，即客户端渲染（Client Side Rendering, CSR）。

SPA 虽然减轻了服务器的压力，但也存在缺点：

- 首屏渲染时间较长：必须等待`JavaScript`加载和执行完毕后才能渲染出首屏；
- `SEO`不友好：爬虫只能拿到一个`div`元素，会当作空页面，不利于`SEO`。

为了解决如上问题，服务端渲染（Server Side Rendering, SSR）应运而生：由服务端负责将数据渲染到首屏的`DOM`结构并返回 HTML，后续其它路由的页面仍然由客户端渲染。这样用户在浏览首屏时速度会很快，因为无需对首屏发送`ajax`请求。尽管添加了 SSR，本质上它仍然是一个 SPA 应用。而 Nuxt.js 是由 Vue.js 衍生出的框架，最常就是用于 SSR。

### 二、Nuxt.js 目录结构

```
└─my-nuxt-demo
  ├─.nuxt               // build后的目标文件夹
  ├─assets              // 用于组织未编译的静态资源如LESS、SASS或JavaScript,对于不需要通过 Webpack 处理的静态资源文件，可以放置在 static 目录中
  ├─components          // 用于自己编写的Vue组件，比如日历组件、分页组件
  ├─layouts             // 布局目录，用于组织应用的布局组件，不可更改⭐
  ├─middleware          // 用于存放中间件
  ├─node_modules
  ├─pages               // 用于组织应用的路由及视图,Nuxt.js根据该目录结构自动生成对应的路由配置，文件名不可更改⭐
  ├─plugins             // 用于组织那些需要在根vue.js应用实例化之前需要运行的 Javascript 插件。
  ├─static              // 用于存放应用的静态文件，此类文件不会被 Nuxt.js 调用 Webpack 进行构建编译处理。 服务器启动的时候，该目录下的文件会映射至应用的根路径 / 下。文件夹名不可更改。⭐
  └─store               // 用于组织应用的Vuex 状态管理。文件夹名不可更改。⭐
  ├─.editorconfig       // 开发工具格式配置
  ├─.eslintrc.js        // ESLint的配置文件，用于检查代码格式
  ├─.gitignore          // 配置git忽略文件
  ├─nuxt.config.js      // 用于组织Nuxt.js 应用的个性化配置，以便覆盖默认配置。文件名不可更改。⭐
  ├─package-lock.json   // npm自动生成，用于帮助package的统一设置的，yarn也有相同的操作
  ├─package.json        // npm 包管理配置文件
  ├─README.md
```

### 三、Nuxt 路由配置

#### 1. 基本路由

Nuxt.js 依据 `pages` 目录结构自动生成 `vue-router` 模块的路由配置。

假设 `pages` 的目录结构如下：

```
└─pages
    ├─index.vue
    └─user
      ├─index.vue
      ├─one.vue
```

那么，Nuxt.js 自动生成的路由配置如下：

```
router: {
  routes: [
    {
      name: 'index',
      path: '/',
      component: 'pages/index.vue'
    },
    {
      name: 'user',
      path: '/user',
      component: 'pages/user/index.vue'
    },
    {
      name: 'user-one',
      path: '/user/one',
      component: 'pages/user/one.vue'
    }
  ]
}
```

有如下两种方法进行页面跳转，推荐前者。

- `<nuxt-link to="/users"></nuxt-link>`
- `this.$router.push('/users');`

#### 2. 动态路由

定义带参数的动态路由，需要创建对应的**以下划线作为前缀**的 Vue 文件或目录。可以通过`{{$route.params.id}}`获取动态参数。

![image-20210624171203973](https://cdn.jsdelivr.net/gh/baimohui/FigureBed/img/20210624171212.png)

### 四、plugin 的使用

