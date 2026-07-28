---
title: Vue Router 4 路由
categories: 
- Vue
tags:
- Vue3
- Vue Router
- 路由
---

## 【面试速答版】

### Q1: "Vue Router 4 相比 3 有哪些主要变化？"

创建从 `new VueRouter()` 变为 `createRouter({ history: createWebHistory() })`。全面拥抱 Composition API——`useRouter` 和 `useRoute`。导航守卫 `next()` 变为可选，不 return 即放行，不再有"忘记调用 next 页面卡死"的 bug。移除了 `<transition>` + `<keep-alive>` 的旧嵌套用法，改用 `<router-view v-slot>` 灵活控制。TypeScript 类型全面增强。动态路由 API 更完善（`addRoute`/`removeRoute`）。<!--more-->

### Q2: "导航守卫的执行顺序是怎样的？"

```text
1. 路由失活组件的 onBeforeRouteLeave
2. 全局 beforeEach
3. 路由配置的 beforeEnter（如适用）
4. 组件内 onBeforeRouteUpdate（如复用）
5. 解析异步路由组件
6. 全局 beforeResolve
7. 导航确认
8. 全局 afterEach
```

Vue Router 4 的守卫通过 return 控制导航方向。return `false` 中止；return 路径对象进行重定向；不 return 则放行。

### Q3: "Hash 模式和 History 模式有什么区别？"

Hash 模式通过 `hashchange` 事件监听 URL `#` 后的变化，兼容性最好（无需服务器配置）。History 模式通过 History API（`pushState`/`replaceState` + `popstate`），URL 美观不带 `#`，但需要服务器配置 fallback（否则刷新 404）。History 模式还能利用 `state` 传递数据。Vue Router 4 源码层面两者只差一个监听函数和一个状态读取方式。

### Q4: "路由组件如何传递 props？"

```js
{ path: '/user/:id', component: User, props: true }        // params 作为 props
{ path: '/user/:id', component: User, props: { role: 'admin' } }  // 静态 props
{ path: '/user/:id', component: User, props: route => ({ id: route.params.id }) }  // 动态
```

推荐用 `props: true`，组件通过 `defineProps(['id'])` 接收，不再依赖 `useRoute()`。

### Q5: "如何实现路由鉴权？"

```js
router.beforeEach((to, from) => {
  const token = localStorage.getItem('token');
  if (to.meta.requiresAuth && !token) {
    return { name: 'login', query: { redirect: to.fullPath } };
  }
  if (to.meta.role && !hasRole(to.meta.role)) {
    return '/403';
  }
});
```

注意加上 `to.name !== 'login'` 判断，避免循环重定向。


## 【深入理解版】

### 1. 这个知识点要解决什么问题？

SPA 路由需要解决三个核心问题：① **URL 与 UI 的映射**——`/user/123` 展示用户详情页；② **浏览器前进/后退**——利用 History API 或 hash 实现；③ **导航过程控制**——鉴权、权限、数据预加载、离开确认。

Vue Router 4 是为 Vue 3 重新设计的版本，不兼容 Vue 2。主要变化原因：Vue 3 不再支持 `Vue.use` 插件机制（改为 `app.use`），全面拥抱 Composition API，TypeScript 成为一等公民。

### 2. 核心原理

#### 2.1 创建流程

```js
import { createRouter, createWebHistory } from 'vue-router';

const router = createRouter({
  history: createWebHistory(),
  routes: [
    { path: '/', name: 'home', component: () => import('./views/Home.vue') },
    { path: '/user/:id', name: 'user', component: () => import('./views/User.vue') },
  ],
});

const app = createApp(App);
app.use(router);
```

`app.use(router)` 内部做了三件事：

```text
1. router.install(app)
   ├── 调用 app.component('RouterLink', RouterLink)
   ├── 调用 app.component('RouterView', RouterView)
   ├── 通过 provide 注入 router 和 route 到全局
   └── 调用 router.push(router.currentRoute) 初始化路由
```

#### 2.2 Hash 模式 vs History 模式

```js
// Hash 模式
import { createWebHashHistory } from 'vue-router';
// 底层监听 window.addEventListener('hashchange', handler)
// URL: http://example.com/#/user/123
// 修改: location.hash = '#/user/123'

// History 模式
import { createWebHistory } from 'vue-router';
// 底层监听 window.addEventListener('popstate', handler)
// URL: http://example.com/user/123
// 修改: history.pushState(state, '', '/user/123')
```

**Hash 模式**的兼容性好（无需服务器配置），但 URL 不美观，且 hash 部分不会被搜索引擎收录。**History 模式**需要服务器配置 fallback，否则用户直接在地址栏输入 `/user/123` 会返回 404：

```nginx
# Nginx fallback
location / {
  try_files $uri $uri/ /index.html;
}
```

#### 2.3 路由匹配原理

Vue Router 内部将用户配置的 `routes` 编译为**路由记录（RouteRecord）** 的扁平列表，再匹配时基于路径模式进行评分（类似正则匹配）：

```js
// 简化匹配逻辑
const records = [
  { path: '/', regex: /^\/(\?|$)/, components: Home },
  { path: '/user/:id', regex: /^\/user\/([^/]+)$/, components: User },
  { path: '/user/:id/post/:postId', regex: /^\/user\/([^/]+)\/post\/([^/]+)$/, components: Post },
];

function match(path) {
  for (const record of records) {
    const match = path.match(record.regex);
    if (match) {
      return { ...record, params: extractParams(record, match) };
    }
  }
  return null;  // 404
}
```

路径参数 `:id` 会被编译为捕获组 `([^/]+)`。`/user/123` 匹配后 `params.id` 就是 `'123'`。

常用匹配模式：

```js
{ path: '/user/:id' }                // 匹配 /user/123 或 /user/abc
{ path: '/user/:id(\\d+)' }          // 只匹配数字
{ path: '/article/:slug+' }          // 多段：/article/a/b/c
{ path: '/file/:path(.*)' }          // 任意路径：/file/a/b/c.txt
{ path: '/:optional?' }              // 可选参数：/ 或 /hello
```

#### 2.4 嵌套路由

```js
const routes = [
  {
    path: '/dashboard',
    component: DashboardLayout,
    children: [
      { path: '', name: 'dashboard', component: DashboardHome },
      { path: 'settings', name: 'settings', component: Settings },
      { path: 'users/:id', name: 'user-detail', component: UserDetail },
    ],
  },
];
```

```vue
<!-- DashboardLayout.vue -->
<template>
  <div class="layout">
    <aside>侧边栏</aside>
    <main>
      <router-view />  <!-- 渲染子路由组件 -->
    </main>
  </div>
</template>
```

子路由的 path 会自动拼接父路由路径：`/dashboard/settings`。空字符串 `''` 表示默认子路由，匹配 `/dashboard`。

Vue Router 4 中**一个 `<router-view>` 组件内部还可以有子 `<router-view>`**，由 `children` 配置决定渲染哪一层。

#### 2.5 编程式导航

```js
const router = useRouter();

// 字符串路径
router.push('/user/123');

// 路径对象
router.push({ path: '/user/123' });

// 命名路由 + params
router.push({ name: 'user', params: { id: '123' } });

// 带查询参数
router.push({ path: '/search', query: { q: 'vue' } });

// 带 hash
router.push({ path: '/settings', hash: '#profile' });

// 替换（不产生历史记录）
router.replace('/user/123');

// 前进/后退
router.go(1);   // 前进 1 步
router.go(-1);  // 后退 1 步
```

**注意**：如果提供了 `path`，`params` 会被忽略。推荐使用 `name` + `params` 的组合，更稳定。

#### 2.6 RouterLink

```vue
<router-link to="/user/123">用户</router-link>
<router-link :to="{ name: 'user', params: { id: user.id } }">用户</router-link>
<router-link to="/user/123" replace>替换</router-link>
<router-link to="/user/123" active-class="active">当前激活</router-link>
<router-link to="/user/123" exact-active-class="exact">精确匹配</router-link>

<!-- v-slot 自定义渲染 -->
<router-link to="/home" v-slot="{ href, route, navigate, isActive, isExactActive }">
  <a :href="href" @click="navigate" :class="{ active: isExactActive }">
    <slot />
  </a>
</router-link>
```

#### 2.7 RouterView 高级用法

```vue
<!-- 配合 transition 动画 -->
<router-view v-slot="{ Component, route }">
  <transition :name="route.meta.transition || 'fade'" mode="out-in">
    <component :is="Component" :key="route.path" />
  </transition>
</router-view>

<!-- 配合 keep-alive -->
<router-view v-slot="{ Component }">
  <keep-alive :include="['Home', 'User']">
    <component :is="Component" />
  </keep-alive>
</router-view>
```

`<router-view v-slot>` 是 Vue Router 4 提供的灵活模式，开发者可以自己控制组件的渲染、过渡动画和缓存策略。

#### 2.8 导航守卫详解

守卫按作用范围分三类：

```js
// 1. 全局守卫
const router = createRouter({ ... });

router.beforeEach((to, from) => { });    // 导航触发时
router.beforeResolve((to, from) => { }); // 导航确认前（组件已解析）
router.afterEach((to, from, failure) => { });  // 导航完成后

// 2. 路由独享守卫
{
  path: '/admin',
  component: Admin,
  beforeEnter: (to, from) => {
    if (!isAdmin()) return '/403';
  },
}

// 3. 组件内守卫
import { onBeforeRouteLeave, onBeforeRouteUpdate } from 'vue-router';

onBeforeRouteLeave((to, from) => {
  if (hasUnsavedChanges) return false;  // false 中止导航
});

onBeforeRouteUpdate((to, from) => {
  // 同一组件但参数变化时（如 /user/1 → /user/2）
  fetchUser(to.params.id);
});
```

守卫的返回值控制导航行为：

```text
return undefined / 不 return  →  放行
return false                   →  中止导航
return { name: 'login' }      →  重定向到登录页
return '/login'                →  同上的字符串写法
```

完整执行顺序：

```text
导航被触发
  ↓
1. 失活组件的 onBeforeRouteLeave
   ↓ 依次执行
2. 全局 beforeEach
   ↓ 依次执行
3. 路由配置的 beforeEnter（只在参数变化时触发）
   ↓
4. 解析异步路由组件
   ↓
5. 复用组件的 onBeforeRouteUpdate
   ↓
6. 全局 beforeResolve
   ↓
7. 导航确认
   ↓
8. 全局 afterEach
   ↓
9. DOM 更新
   ↓
10. 触发路由完成回调
```

#### 2.9 滚动行为

```js
const router = createRouter({
  scrollBehavior(to, from, savedPosition) {
    if (savedPosition) {
      return savedPosition;  // 后退时恢复滚动位置
    }
    if (to.hash) {
      return { el: to.hash, behavior: 'smooth' };  // 滚动到锚点
    }
    return { top: 0, behavior: 'smooth' };  // 默认回到顶部
  },
});
```

### 3. 实际应用场景

#### 场景 1：权限管理系统

```js
// router/index.js
const routes = [
  { path: '/login', name: 'login', component: () => import('@/views/Login.vue') },
  {
    path: '/admin',
    component: AdminLayout,
    meta: { requiresAuth: true, role: 'admin' },
    children: [
      { path: '', name: 'admin-dashboard', component: () => import('@/views/AdminDashboard.vue') },
      { path: 'users', name: 'admin-users', component: () => import('@/views/AdminUsers.vue') },
    ],
  },
];

const router = createRouter({ history: createWebHistory(), routes });

// 全局前置守卫
router.beforeEach(async (to, from) => {
  const token = localStorage.getItem('token');

  // 未登录 → 跳登录
  if (to.meta.requiresAuth && !token) {
    return { name: 'login', query: { redirect: to.fullPath } };
  }

  // 已登录但访问登录页 → 跳首页
  if (to.name === 'login' && token) {
    return '/';
  }

  // 角色权限检查
  if (to.meta.role) {
    const user = await fetchUserInfo();
    if (!user.roles.includes(to.meta.role)) {
      return '/403';
    }
  }
});

export default router;
```

#### 场景 2：动态路由——后端返回权限配置

```js
// 登录成功后
async function initRouter(user) {
  // 清空旧路由
  router.removeRoute('dashboard');

  // 根据用户角色动态添加路由
  const dynamicRoutes = await fetchUserRoutes(user.role);

  dynamicRoutes.forEach(route => {
    router.addRoute({
      path: route.path,
      name: route.name,
      component: () => import(`@/views/${route.component}.vue`),
      meta: { ...route.meta },
    });
  });
}
```

#### 场景 3：表单离开确认

```vue
<script setup>
import { onBeforeRouteLeave } from 'vue-router';

const hasUnsavedChanges = ref(false);

onBeforeRouteLeave((to, from) => {
  if (hasUnsavedChanges.value) {
    const leave = window.confirm('有未保存的内容，确定离开吗？');
    if (!leave) return false;
  }
});
</script>
```

#### 场景 4：路由过渡动画

```vue
<template>
  <router-view v-slot="{ Component, route }">
    <transition
      :name="route.meta.transition || 'fade'"
      mode="out-in"
    >
      <component :is="Component" :key="route.path" />
    </transition>
  </router-view>
</template>

<style>
.fade-enter-active, .fade-leave-active {
  transition: opacity 0.2s ease;
}
.fade-enter-from, .fade-leave-to {
  opacity: 0;
}
</style>
```

### 4. 常见误区 & 实际项目中的坑

#### 误区 1：在 `<script setup>` 中用 `this.$router`

```vue
<script setup>
this.$router.push('/home');  // ❌ 没有 this
const router = useRouter();  // ✅
</script>
```

#### 误区 2：History 模式未配置服务器 fallback

部署到 Nginx 时未配置 `try_files`，用户刷新子页面返回 404。解决方案见上文。

#### 误区 3：导航守卫循环重定向

```js
router.beforeEach((to) => {
  if (!isLogin) {
    return '/login';   // ❌ 没有判断 to.name !== 'login'
  }
});
```

正确的写法是加条件判断：`if (!isLogin && to.name !== 'login')`。

#### 误区 4：watch route 不及时

```vue
<script setup>
const route = useRoute();
watch(() => route.params.id, (id) => { fetch(id); });
</script>
```

在组件复用时（如从 `/user/1` 到 `/user/2`），`watch` 可以监听到参数变化。但如果组件不复用（如 `/user/1` 到 `/article/2`），组件会重新挂载，应使用 `onMounted` 或路由守卫。

#### 误区 5：path 和 params 混用

```js
router.push({ path: '/user', params: { id: '123' } });  // ❌ params 被忽略
router.push({ name: 'user', params: { id: '123' } });   // ✅
```

使用 `path` 时 `params` 会被忽略。用 `name` + `params` 替代。

### 5. 与相关知识的关联 & 对比

| 对比维度 | Vue Router 3 | Vue Router 4 |
|----------|-------------|-------------|
| 创建方式 | `new VueRouter({ ... })` | `createRouter({ ... })` |
| 路由模式 | `mode: 'history'` | `history: createWebHistory()` |
| 组合式 API | ❌ 不支持 | ✅ useRouter / useRoute |
| 导航守卫 next | 必须调用（否则卡住） | 可选（return 替代） |
| TypeScript | 弱类型 | 强类型（RouteRecordRaw 等） |
| 动态路由 | 限制多 | addRoute / removeRoute 完善 |
| router-view | 不支持 v-slot | v-slot + component 灵活控制 |
| 滚动行为 | 支持 | 支持，且返回 Promise |
| 编码大小 | ~20KB | 更小 |

### 6. 现代最佳实践

1. **始终使用 History 模式**（`createWebHistory`），除非无法配置服务器 fallback
2. **路由配置按模块拆分**，通过 `addRoute` 按需加载
3. **用 `props: true` 解耦路由组件**，组件不依赖 `$route`
4. **导航守卫统一管理鉴权**，避免分散在各组件中
5. **配合 `defineAsyncComponent`** 做路由级代码分割
6. **路由 meta 字段规范**：统一在 meta 中定义 `requiresAuth`、`role`、`title`、`transition`、`keepAlive` 等字段
7. **使用命名路由**（`name`）替代路径字符串，方便重构和维护
8. **用 navigation failure 判断导航结果**：

```js
const result = await router.push('/admin');
if (result === undefined) {
  // 导航成功
} else {
  // 导航被中止（result 是 NavigationFailure 对象）
  if (result.type === 'aborted') { }       // 返回 false
  if (result.type === 'duplicated') { }    // 已在目标位置
}
```

### 7. 常见疑问解答

**Q：`useRoute()` 返回的是响应式对象吗？**

是的。它通过 Vue 的 `shallowRef` 实现响应式。当导航发生时，Vue Router 内部更新 `currentRoute`，所有使用了 `useRoute()` 的组件都会自动更新。

**Q：`app.use(router)` 时内部做了什么？**

```text
1. 注册全局组件 RouterLink + RouterView
2. 通过 provide 向全应用注入 router 实例和当前路由
3. 调用 router.push(router.currentRoute) 初始化
4. 在根组件上注册 `app.config.globalProperties.$router` 等
```

**Q：`router.addRoute` 添加的路由什么时候生效？**

立即生效。`addRoute` 将新路由记录添加到路由表中，下一次导航匹配就会包含新路由。无须重新调用 `createRouter`。

**Q：嵌套路由可以无限嵌套吗？**

理论上可以，但实际建议嵌套不超过 3 层。每层嵌套都会在 `RouterView` 中再套一个 `RouterView`，层级过深会影响渲染性能。

**Q：路由懒加载的原理是什么？**

```js
{ component: () => import('./views/Home.vue') }
```

`() => import()` 返回一个 Promise，Vue Router 在首次导航到该路由时才执行 import，Webpack/Vite 会将动态 import 的模块打包为独立的 chunk。导航到该路由时，路由守卫中的 beforeResolve 会等待异步组件解析完成，然后才渲染。

**关联知识点索引**
- `Vue3 核心变化与 Composition API.md` — Composition API 基础
- `响应式原理.md` — shallowRef 与 provide/inject
- `Vite 详解.md` — 动态 import 与代码分割
