---
title: Pinia 状态管理
categories: 
- Vue
tags:
- Vue3
- Pinia
- 状态管理
- Vuex
---

## 【面试速答版】

### Q1: "Pinia 和 Vuex 有什么核心区别？为什么官方用 Pinia 替代 Vuex？"

Pinia 相比 Vuex 做了三个核心简化：① **去掉了 mutations**——Vuex 中修改 state 必须经过 mutation（同步）+ action（异步），Pinia 直接在 actions 中修改 state（这本来也是大部分开发者的实际做法）；② **模块化无需嵌套**——Vuex 用 modules 组织模块（有命名空间 namespace 的麻烦），Pinia 每个 store 独立的 `defineStore`，天然模块化；③ **完整的 TypeScript 支持**——无需额外类型声明就能获得完整类型推断。Pinia 也完全支持 Composition API（setup store 写法类似 composable）。体积仅约 1KB（Vuex 约 10KB）。Vue 官方已在 2022 年正式推荐 Pinia 替代 Vuex。

### Q2: "Pinia 中 Options Store 和 Setup Store 有什么区别？"

Options Store 类似 Vuex 的写法：`state` + `getters` + `actions`。Setup Store 类似 Composition API 的写法：用 `ref`/`reactive` 定义 state，`computed` 定义 getters，普通函数定义 actions，返回一个对象。两种写法功能等价。Setup Store 更灵活——内部可以使用 `watch`、生命周期钩子等。推荐新项目用 Setup Store。

### Q3: "从 store 中解构会丢失响应式吗？如何正确解构？"

会——直接解构 `const { count, name } = useStore()` 得到的是普通值，不是响应式的。需要用 `storeToRefs` 解构 state 和 getters，保持响应式。actions（函数）可以直接解构，不需要 `storeToRefs`。

## 【深入理解版】

### 1. 这个知识点要解决什么问题？

当应用中有多个组件/页面需要共享数据（用户登录状态、购物车、全局配置），用 props 层层传递太麻烦，用 Context/Provide 跨层级注入了又无法做精细的按需更新。状态管理库把共享数据抽离出来，放到一个统一的地方管理，任何组件都可以读和写。

Vuex 是 Vue2 时代的解决方案，但它有两个让人头疼的设计：mutations 强制要求"修改 state 必须通过 mutation"（导致一个简单的 `count++` 需要写 mutation + action 两套代码），以及 modules 的命名空间配置复杂。Pinia 在 Vue3 时代重新设计了状态管理——去掉 mutation、模块独立、TypeScript 原生支持。

### 2. 核心原理/执行过程

#### 2.1 Pinia store 的本质

```javascript
import { defineStore } from 'pinia'

// defineStore 接收两个参数：
// 参数1: store 的唯一 ID（类似 Vuex 的模块名）
// 参数2: 配置对象（Options Store）或函数（Setup Store）
export const useCounterStore = defineStore('counter', {
  state: () => ({ count: 0 }),
  getters: {
    doubleCount: (state) => state.count * 2,
  },
  actions: {
    increment() {
      // Pinia 中 action 直接修改 state，不需要通过 mutation
      this.count++
    },
  },
})
```

Pinia store 的 `state` 本质上是一个 `reactive()` 对象。`getters` 是 `computed()`。`actions` 就是普通的函数。这也是为什么 Pinia 这么简洁——它直接复用了 Vue3 的响应式系统，没有自己再封装一层。

#### 2.2 Setup Store（组合式写法）

```javascript
export const useCounterStore = defineStore('counter', () => {
  // state → ref/reactive
  const count = ref(0)
  const name = ref('Eduardo')

  // getters → computed
  const doubleCount = computed(() => count.value * 2)

  // actions → 普通函数
  function increment() {
    count.value++
  }

  // 返回对象，暴露给外部
  return { count, name, doubleCount, increment }
})
```

Setup Store 的写法与 Vue3 的 Composition API 完全一致——你可以使用 `watch`、`onMounted` 等。一个实际例子：

```javascript
export const useAuthStore = defineStore('auth', () => {
  const user = ref(null)
  const token = ref(localStorage.getItem('token') || '')

  // 监听 token 变化，自动同步到 localStorage
  watch(token, (val) => {
    if (val) localStorage.setItem('token', val)
    else localStorage.removeItem('token')
  })

  const isLoggedIn = computed(() => !!token.value)

  async function login(credentials) {
    const res = await api.login(credentials)
    token.value = res.token
    user.value = res.user
  }

  function logout() {
    token.value = ''
    user.value = null
  }

  return { user, token, isLoggedIn, login, logout }
})
```

#### 2.3 storeToRefs 的作用

```vue
<script setup>
import { useCounterStore } from './stores/counter'
import { storeToRefs } from 'pinia'

const store = useCounterStore()

// ❌ 直接解构：count 和 doubleCount 丢失响应式
const { count, doubleCount } = store

// ✅ storeToRefs 保持响应式
const { count, doubleCount } = storeToRefs(store)

// ✅ actions 可以直接解构（函数没有响应式的问题）
const { increment } = store
</script>
```

`storeToRefs` 的作用类似 `toRefs`——把 store 中的响应式数据（state、getters）转换为 ref，解构后仍然保持响应式。

### 3. 实际应用场景

#### 场景1：购物车 store

```javascript
// stores/cart.js
export const useCartStore = defineStore('cart', () => {
  const items = ref([])

  const total = computed(() =>
    items.value.reduce((sum, i) => sum + i.price * i.quantity, 0)
  )

  const itemCount = computed(() => items.value.length)

  function addItem(product) {
    const existing = items.value.find(i => i.id === product.id)
    if (existing) {
      existing.quantity++
    } else {
      items.value.push({ ...product, quantity: 1 })
    }
  }

  function removeItem(id) {
    items.value = items.value.filter(i => i.id !== id)
  }

  return { items, total, itemCount, addItem, removeItem }
})
```

```vue
<script setup>
import { useCartStore } from './stores/cart'

const cart = useCartStore()
</script>

<template>
  <button @click="cart.addItem({ id: 1, name: '商品A', price: 99 })">加入购物车</button>
  <div>商品总数: {{ cart.itemCount }}</div>
  <div>总计: ¥{{ cart.total }}</div>
</template>
```

#### 场景2：跨 store 互相调用

```javascript
// stores/order.js
import { useCartStore } from './cart'
import { useAuthStore } from './auth'

export const useOrderStore = defineStore('order', () => {
  const orders = ref([])
  const loading = ref(false)

  async function checkout() {
    const auth = useAuthStore()
    const cart = useCartStore()

    if (!auth.isLoggedIn) {
      throw new Error('请先登录')
    }

    loading.value = true
    const order = await api.checkout({
      items: cart.items,
      token: auth.token,
    })

    orders.value.push(order)
    cart.items = []  // 清空购物车
    loading.value = false
    return order
  }

  return { orders, loading, checkout }
})
```

### 4. 常见误区 & 实际项目中的坑

#### 误区1：从 store 中解构 state 后丢失响应式

```javascript
const store = useCounterStore()
const { count } = store  // ❌ count 变为普通值
```

**正确**：`const { count } = storeToRefs(store)`

#### 误区2：在 Setup Store 中使用 this

```javascript
export const useStore = defineStore('store', () => {
  const count = ref(0)

  function increment() {
    this.count++  // ❌ Setup Store 中没有 this
  }

  return { count, increment }
})
```

**正确**：直接使用 `count.value++`。

#### 坑：使用 `$patch` 批量更新

```javascript
// 多次修改会触发多次更新
store.count++
store.name = 'new name'

// ✅ 用 $patch 批量更新，只触发一次
store.$patch({
  count: store.count + 1,
  name: 'new name',
})
// 或函数式：
store.$patch((state) => {
  state.count++
  state.name = 'new name'
})
```

### 5. 与相关知识的关联 & 对比

| 对比维度 | Vuex 4 | Pinia |
|---|---|---|
| mutations | 必须定义 | 无（直接修改 state） |
| 模块化 | modules 嵌套 | 独立 defineStore |
| TypeScript | 需额外声明 | 自动类型推导 |
| Composition API | 支持有限 | 原生支持 |
| 体积 | ~10KB | ~1KB |
| DevTools | 支持 | 支持 |

| API | Vuex | Pinia |
|---|---|---|
| 定义 store | `new Vuex.Store({...})` | `defineStore(id, options/setup)` |
| 读取 state | `$store.state.count` | `store.count` |
| 修改 state | `commit('increment')` | `store.count++` / `$patch` |
| 异步操作 | dispatch(action) | 直接在 action 中 await |
| getter | `$store.getters.double` | `store.double` |

### 6. 现代最佳实践（2024-2025）

1. **新项目直接使用 Pinia**，不再使用 Vuex。
2. **优先使用 Setup Store 语法**——更灵活、天然支持 TypeScript、内部可以使用 watch 和生命周期。
3. **每个功能/领域一个独立的 store 文件**——如 `useAuthStore`、`useCartStore`、`useThemeStore`。
4. **使用 `storeToRefs` 解构 state 和 getters**，actions 直接解构。
5. **批量更新用 `$patch`** 减少更新次数。
6. **使用 `$subscribe` 监听 store 变化做持久化**：

```javascript
cartStore.$subscribe((mutation, state) => {
  localStorage.setItem('cart', JSON.stringify(state.items))
})
```

### 7. 常见疑问解答

**Q：Pinia 的 setup store 和 composable 有什么区别？**

A：setup store 是一个在 Pinia 管理下的"特殊的 composable"。它与普通 composable 的区别在于：① Pinia store 是全局单例的——你在任何组件中调用 `useCounterStore()` 都返回同一个实例；② 它与 Vue DevTools 集成——状态变化在 DevTools 中可见，可以时间旅行调试；③ 它支持 `$subscribe`、`$patch`、`$reset` 等内置方法。普通的 composable 没有这些能力。结论：跨组件/跨页面共享状态用 Pinia store；单个组件内的逻辑封装用 composable。

**Q：如果两个 store 互相引用，会发生什么？**

A：在 Options Store 中，可以在 action 内调用 `useOtherStore()`，因为 action 在执行时才被调用，不会在 store 初始化时产生循环引用。Setup Store 同理——`useOtherStore()` 在函数内部调用（而不是在模块顶层）。Pinia 已经处理了这种相互引用的场景，不会产生死循环。

**关联知识点索引**
- `Vue3 核心变化与 Composition API.md` — 理解 Setup Store 的写法基础
- `响应式原理.md` — Pinia state 基于 reactive 的实现
