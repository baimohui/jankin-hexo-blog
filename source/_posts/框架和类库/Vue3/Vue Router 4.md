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

### Q1: "Vue Router 4 相比 Vue Router 3 有哪些主要变化？"

创建方式从 `new VueRouter()` 变为 `createRouter({ history: createWebHistory() })`。全面拥抱 Composition API，提供 `useRouter` 和 `useRoute` 组合式函数。导航守卫中 `next()` 变为可选（Vue Router 3 中必须调用 next，不调用就卡住；Vue Router 4 中不 return 或 return undefined 表示放行）。移除了 `<transition>` + `<keep-alive>` 的旧嵌套用法，改用 `<router-view v-slot>` 更灵活。TypeScript 支持大幅增强，路由配置和 RouteRecord 都有完整类型推断。

### Q2: "Vue Router 4 中如何使用 Composition API？"

使用 `useRouter()` 获取 router 实例（用于编程式导航：`router.push('/home')`），使用 `useRoute()` 获取当前路由的响应式对象（用于读取 params/query/meta）。还可以用 `onBeforeRouteLeave` 和 `onBeforeRouteUpdate` 在组件内设置导航守卫——不需要写在路由配置中。这些都在 `<script setup>` 中直接使用。

### Q3: "导航守卫有哪几种？它们分别在什么时机执行？"

三种：**全局守卫**——`router.beforeEach`（任何导航前触发）、`router.beforeResolve`（导航确认前）、`router.afterEach`（导航完成后）。**路由独享守卫**——路由配置中的 `beforeEnter`。**组件内守卫**——`onBeforeRouteLeave`（离开当前路由时）、`onBeforeRouteUpdate`（参数变化但组件复用时）。执行顺序：全局 beforeEach → 路由 beforeEnter → 组件 onBeforeRouteUpdate → 全局 beforeResolve → 导航完成 → 全局 afterEach。

## 【深入理解版】

### 1. 这个知识点要解决什么问题？

SPA 中，路由解决三个问题：① **URL 与 UI 的映射**——`/user/123` 显示用户详情页；② **浏览器前进/后退**——利用 History API 实现；③ **导航过程控制**——登录检查、权限验证、数据预加载。

Vue Router 4 是为 Vue3 重新设计的版本，主要变化是因为 Vue3 不再支持 Vue2 的插件机制（Vue.use）、不再有 `new Vue()` 实例、全面拥抱 Composition API。

### 2. 核心原理/执行过程

#### 2.1 createRouter 创建流程

```javascript
import { createRouter, createWebHistory } from 'vue-router'

const router = createRouter({
  history: createWebHistory(),  // 使用 HTML5 History 模式
  routes: [
    { path: '/', name: 'home', component: () => import('./views/Home.vue') },
    { path: '/user/:id', name: 'user', component: () => import('./views/User.vue') },
  ],
})

// 安装到 Vue 应用
const app = createApp(App)
app.use(router)
app.mount('#app')
```

`createWebHistory()` 使用 HTML5 History API（`pushState`、`popstate`），URL 中不带 `#`。`createWebHashHistory()` 使用 URL hash（`#/user/123`），兼容性更好但 URL 不美观。

#### 2.2 导航解析流程

```text
用户点击 <router-link> 或调用 router.push()
  ↓
1. 导航被触发
2. 失活的组件调用 onBeforeRouteLeave 守卫
3. 全局 beforeEach 守卫
4. 路由配置中的 beforeEnter 守卫（如果有）
5. 解析异步路由组件
6. 复用的组件调用 onBeforeRouteUpdate
7. 全局 beforeResolve 守卫
8. 导航确认
9. 全局 afterEach 钩子
10. 触发 DOM 更新
```

#### 2.3 动态路由

```javascript
// 添加路由（运行时动态注册）
router.addRoute({
  path: '/admin',
  name: 'admin',
  component: () => import('./views/Admin.vue'),
  meta: { requiresAuth: true },
})

// 添加嵌套路由
router.addRoute('admin', {
  path: 'users',
  component: () => import('./views/AdminUsers.vue'),
})

// 移除路由
router.removeRoute('admin')
```

`addRoute` 可以在应用运行时动态添加路由，适合权限管理场景——根据用户角色动态注册可访问的路由。

### 3. 实际应用场景

#### 场景1：组合式函数中使用路由

```vue
<script setup>
import { useRouter, useRoute, onBeforeRouteLeave } from 'vue-router'

const router = useRouter()
const route = useRoute()

function goToUser(id) {
  router.push({ name: 'user', params: { id } })
}

// 监听路由参数变化（同一组件内切换用户）
import { watch } from 'vue'
watch(() => route.params.id, (newId) => {
  fetchUser(newId)
})

// 离开前确认
onBeforeRouteLeave((to, from) => {
  if (hasUnsavedChanges) {
    const leave = window.confirm('有未保存的修改，确定离开？')
    if (!leave) return false
  }
})
</script>
```

`onBeforeRouteLeave` 是 Vue Router 4 新增的组合式守卫——在 setup 中直接调用，不需要在路由配置中声明。

#### 场景2：路由守卫 + 登录验证

```javascript
// router/index.js
const routes = [
  {
    path: '/dashboard',
    meta: { requiresAuth: true },
    component: () => import('./views/Dashboard.vue'),
  },
  {
    path: '/admin',
    meta: { requiresAuth: true, role: 'admin' },
    component: () => import('./views/Admin.vue'),
    beforeEnter: (to, from) => {
      // 路由独享守卫：检查是否管理员
      if (!userIsAdmin()) return '/403'
    }
  },
]

// 全局前置守卫
router.beforeEach((to, from) => {
  const token = localStorage.getItem('token')

  if (to.meta.requiresAuth && !token) {
    // 未登录 → 跳登录页，带来源 URL 以便登录后跳回
    return { name: 'login', query: { redirect: to.fullPath } }
  }
  // 不 return 或 return undefined → 放行
})
```

注意 Vue Router 4 的守卫中 **不调用 next 也不会卡住导航**。如果你不 return 任何东西，导航正常进行。如果你要阻断导航，return `false` 或 return 一个路由地址。

### 4. 常见误区 & 实际项目中的坑

#### 误区1：在 `<script setup>` 中直接使用 `this.$router`

```vue
<script setup>
this.$router.push('/home') // ❌ setup 中没有 this
</script>
```

**正确**：使用 `useRouter()`。

#### 误区2：History 模式部署时没有配置服务器 fallback

```nginx
# Nginx 配置
location / {
  try_files $uri $uri/ /index.html;
}
```

否则用户刷新 `/user/123` 页面时，服务器找不到这个路径，返回 404。

#### 坑：编码导航守卫中 return 的路径可能造成循环

```javascript
router.beforeEach((to) => {
  if (!isLogged && to.name !== 'login') {
    return '/login'  // 这里不加判断会循环：跳 /login → /login 又被拦截 → 又跳 /login...
  }
})
```

注意在 return 重定向之前加上 `to.name !== 'login'` 的判断。

### 5. 与相关知识的关联 & 对比

| 对比维度 | Vue Router 3 (Vue2) | Vue Router 4 (Vue3) |
|---|---|---|
| 创建方式 | `new VueRouter({...})` | `createRouter({...})` |
| 路由模式 | `mode: 'history'` 字符串 | `history: createWebHistory()` |
| Composition API | 不支持 | `useRouter`/`useRoute` |
| 导航守卫 next | 必须调用 | 可选（return 替代） |
| TypeScript | 弱 | 强 |
| 动态路由 | 支持但限制多 | 更灵活（addRoute/removeRoute） |

### 6. 现代最佳实践（2024-2025）

1. **优先使用 HTML5 History 模式**（`createWebHistory`），URL 更美观。只用 Hash 模式在无法配置服务器 fallback 的 CDN 部署场景。
2. **路由配置按模块拆分**——大型应用的路由按功能模块拆分为独立文件，通过 `addRoute` 按需注册。
3. **利用 `props: true` 解耦**——让路由组件通过 props 接收参数，而非依赖 `$route`：

```javascript
{ path: '/user/:id', component: User, props: true }
// User 组件中：defineProps(['id']) 而不是 useRoute().params.id
```

4. **导航守卫尽量在全局统一处理**（如登录验证），路由独享守卫处理细粒度权限。
5. **配合 `defineAsyncComponent` 做路由级代码分割**：`component: () => import('./views/xxx.vue')`。

### 7. 常见疑问解答

**Q：`useRoute()` 返回的是响应式对象吗？**

A：是的，`useRoute()` 返回的是一个响应式对象。当路由参数变化时，使用 `useRoute().params.id` 的组件会自动重新渲染。但如果只在模板中使用了 `$route`，Vue Router 内部是通过 `provide/inject` 注入的，模板中直接使用 `$route.params.id` 即可。

**Q：导航守卫中不调用 next 会卡住页面吗？**

A：Vue Router 3 中会——你必须调用 `next()` 来"放行"导航。Vue Router 4 中 `next` 是可选参数，如果你不调用也不 return，导航会正常进行。这是 Vue Router 4 的重大改进——减少了忘记调用 next 导致的卡死 bug。

**关联知识点索引**
- `Vue3 核心变化与 Composition API.md` — Composition API 基础
- `组件通信.md` — provide/inject 与路由守卫
