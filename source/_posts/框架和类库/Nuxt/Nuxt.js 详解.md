---
title: Nuxt.js 详解
categories: 
- 框架和类库
tags:
- Nuxt
- Vue
- SSR
- 服务端渲染
- SEO
---

## 一、Nuxt.js 是什么

Nuxt.js 是一个基于 Vue.js 的**通用应用框架**，它封装了 Vue、Vue Router、Vite/Webpack 等服务端渲染所需的工具链，提供了一套约定优于配置的开发模式，支持 SSR（服务端渲染）、SSG（静态生成）和 SPA 三种渲染模式。<!--more-->

### 为什么需要 Nuxt

传统 Vue SPA 有两个主要问题：

- **SEO 差**：HTML 中只有一个 `div#app`，搜索引擎爬虫抓取不到内容
- **首屏慢**：需要先加载 JS 再渲染页面，首屏白屏时间长

Nuxt 通过**服务端渲染**（在服务器上生成完整 HTML 再发送给浏览器）解决了这两个问题。

### 三种渲染模式

| 模式 | 生成时机 | 适用场景 |
|------|----------|----------|
| **SSR**（Universal） | 每次请求时在服务器渲染 | 电商、内容站、需要动态数据 |
| **SSG**（Static） | 构建时生成静态 HTML | 博客、文档站、着陆页 |
| **SPA** | 客户端渲染（同普通 Vue） | 后台管理系统、仪表盘 |

```js
// nuxt.config.ts
export default defineNuxtConfig({
  ssr: true,   // true = SSR（默认）
  // ssr: false,  // = SPA 模式
  // 静态生成使用 `npx nuxi generate`
});
```

## 二、快速开始

```bash
npx nuxi init my-app
cd my-app
npm install
npm run dev    # 开发服务器 http://localhost:3000
```

## 三、目录结构与约定

```
my-app/
├── app.vue              # 根组件
├── pages/               # 页面（自动路由）
├── components/          # 组件（自动导入）
├── layouts/             # 布局
├── middleware/          # 路由中间件
├── composables/         # 组合式函数（自动导入）
├── server/              # API 服务端路由
├── public/              # 静态资源
├── nuxt.config.ts       # 配置
└── package.json
```

## 四、自动路由（Pages）

Nuxt 根据 `pages/` 目录结构自动生成路由，无需手动配置 Vue Router。

```text
pages/
├── index.vue           → /
├── about.vue           → /about
├── users/
│   ├── index.vue       → /users
│   └── [id].vue        → /users/:id
└── blog/
    ├── index.vue       → /blog
    ├── [slug].vue      → /blog/:slug
    └── category/
        └── [cat].vue   → /blog/category/:cat
```

```vue
<!-- pages/users/[id].vue -->
<script setup>
const route = useRoute();
const { data: user } = await useFetch(`/api/users/${route.params.id}`);
</script>

<template>
  <div>{{ user.name }}</div>
</template>
```

### 导航

```vue
<NuxtLink to="/about">关于</NuxtLink>
<NuxtLink :to="`/users/${user.id}`">{{ user.name }}</NuxtLink>
```

## 五、数据获取

Nuxt 提供了两个核心组合式函数来获取数据，它们会在服务端执行并将数据注入 HTML。

### useFetch

```vue
<script setup>
const { data, pending, error, refresh } = await useFetch('/api/users', {
  method: 'GET',
  // params: { page: 1 },
});

// useFetch 会自动拼接 baseURL（来自 runtimeConfig）
</script>
```

### useAsyncData

当需要更精细的控制时，用 `useAsyncData` 配合自定义请求：

```vue
<script setup>
const { data, pending } = await useAsyncData('users', async () => {
  const response = await $fetch('/api/users');
  return response.data;
});
</script>
```

### 参数变化时重新获取

```vue
<script setup>
const route = useRoute();
const { data } = await useFetch(`/api/posts/${route.params.id}`, {
  watch: [route.params.id],    // id 变化时自动重新请求
});
</script>
```

## 六、布局

Nuxt 的布局系统允许不同页面组使用不同的布局。

```vue
<!-- layouts/default.vue（默认布局） -->
<template>
  <div>
    <header>通用头部</header>
    <slot />     <!-- 页面内容注入到这里 -->
    <footer>通用底部</footer>
  </div>
</template>
```

```vue
<!-- layouts/blank.vue（空白布局） -->
<template>
  <slot />
</template>
```

页面指定布局：

```vue
<!-- pages/login.vue -->
<script setup>
definePageMeta({ layout: 'blank' });   // 使用空白布局
</script>
```

## 七、中间件

路由中间件在页面渲染前执行，常用于鉴权：

```vue
// middleware/auth.ts
export default defineNuxtRouteMiddleware((to, from) => {
  const token = useCookie('token');
  if (!token.value && to.path !== '/login') {
    return navigateTo('/login');
  }
});
```

```vue
<!-- 页面级别中间件 -->
<script setup>
definePageMeta({
  middleware: 'auth',
});
</script>
```

## 八、服务端 API

Nuxt 支持在 `server/` 目录中直接编写 API 端点，无需额外搭建 Express 服务器：

```ts
// server/api/users.get.ts        → GET /api/users
// server/api/users/[id].delete.ts → DELETE /api/users/:id

export default defineEventHandler(async (event) => {
  const query = getQuery(event);         // 查询参数
  const body = await readBody(event);    // 请求体
  const params = getRouterParams(event); // 路由参数

  return { users: [] };
});
```

```ts
// 服务器启动时初始化数据库连接等
export default defineNitroPlugin((nitroApp) => {
  console.log('Server initialized');
});
```

## 九、配置

```ts
// nuxt.config.ts
export default defineNuxtConfig({
  ssr: true,

  // 运行时配置（客户端和服务端都可访问）
  runtimeConfig: {
    public: {
      apiBase: '/api',
    },
  },

  app: {
    head: {
      title: 'My App',
      meta: [
        { name: 'description', content: 'description' },
      ],
    },
  },

  css: ['~/assets/styles/main.scss'],

  modules: ['@nuxt/ui', '@nuxtjs/tailwindcss'],
});
```

## 十、SEO 与 Meta

```vue
<script setup>
// 静态 SEO
useHead({
  title: '首页',
  meta: [
    { name: 'description', content: '页面描述' },
    { property: 'og:title', content: 'OG 标题' },
  ],
});

// 动态 SEO（根据数据设置）
const { data: post } = await useFetch(`/api/posts/${route.params.id}`);
useHead({
  title: post.value.title,
  meta: [
    { name: 'description', content: post.value.summary },
  ],
});
</script>
```

## 十一、常见问题

### Q1: Nuxt 2 vs Nuxt 3 怎么选

新项目一律使用 Nuxt 3（基于 Vue 3 + Vite + Nitro）。Nuxt 2 已进入维护期。

### Q2: 部署方式

```bash
# SSR 部署
npm run build
node .output/server/index.mjs

# 静态生成
npx nuxi generate
# 将 .output/public 部署到 Nginx / Vercel / Netlify
```

### Q3: 如何处理客户端独有的 API（如 localStorage）

```vue
<script setup>
// ❌ 服务端没有 localStorage
// const token = localStorage.getItem('token');

// ✅ 仅在客户端执行
const token = process.client ? localStorage.getItem('token') : null;

// 或使用 onMounted（只会客户端执行）
onMounted(() => {
  const token = localStorage.getItem('token');
});
</script>
```

## 十二、推荐学习路径

1. 理解 SSR 的概念和 Nuxt 的渲染模式
2. 掌握自动路由、layouts、components 的约定
3. 使用 `useFetch` / `useAsyncData` 获取数据
4. 编写中间件实现鉴权
5. 使用 `server/` 目录编写 API
6. 配置 SEO meta 和部署
