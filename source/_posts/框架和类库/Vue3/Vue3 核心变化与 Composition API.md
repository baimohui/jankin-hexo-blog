---
title: Vue3 核心变化与 Composition API
categories: 
- Vue
tags:
- Vue3
- Composition API
- 面试
---

## 【面试速答版】

### Q1: "Vue3 对比 Vue2 有哪些核心变化？"

三大核心变化：**响应式系统**——用 Proxy 替代 Object.defineProperty，解决了 Vue2 无法监听数组下标修改、新增/删除属性、Map/Set 的问题，且性能更好。**Composition API**——把组件逻辑按功能组织而非按选项（data/methods/computed）分割，解决了 Vue2 mixin 的命名冲突和来源不明问题。**编译器优化**——引入 PatchFlag（编译时标记动态节点类型）、静态提升（静态节点提升到 render 外复用）、树摇（按需引入运行时 API），渲染性能提升约 1.3-2 倍。此外还新增了 Fragments（多根节点）、Teleport（传送门——把组件渲染到指定 DOM 位置）、Suspense（异步依赖统一管理）等特性。

### Q2: "Composition API 解决了什么痛点？Vue2 的 Options API 有什么问题？"

Vue2 中一个 200 行的组件，逻辑分散在 data、methods、computed、watch 四个选项中——搜索功能相关的代码分散在四个地方，分页功能也是。你想理解"搜索"整个功能，要在文件内来回跳转。这就是"逻辑分散"问题。逻辑复用靠 mixin——但 mixin 有命名冲突（两个 mixin 都定义了 handleSearch）、来源不明（模板里的 handleSearch 是哪个 mixin 提供的？）、隐式依赖（mixin 依赖宿主组件的某个 data，但没有任何类型提示）。Composition API 让你按功能组织：搜索的逻辑写在一起，分页的逻辑写在一起，然后通过 composables（组合函数）把逻辑提取为可复用的 `useSearch`、`usePagination`，没有冲突、来源清晰。

### Q3: "setup 和 script setup 是什么？ref 和 reactive 有什么区别？"

`setup` 是 Composition API 的入口函数，在组件创建前执行。`<script setup>` 是它的语法糖，编译后等价于 setup 函数，模板中直接使用顶层变量不需要 return。`ref` 和 `reactive` 都是创建响应式数据的方式：`ref` 通过 `.value` 访问/修改，适用于基础类型和需要重新赋值的变量；`reactive` 直接操作属性，适用于深层嵌套的对象。在模板中 ref 自动解包（不需要 .value），在 reactive 中也会自动解包。建议优先用 ref——赋值时不会丢失响应式，且解构时用 `toRefs` 即可。

## 【深入理解版】

### 1. 这个知识点要解决什么问题？

Vue2 当年（2016）的设计在现在看来有一些历史局限。用 Vue2 写过 300 行以上的组件可能都有这种体验：一个表格组件里同时有搜索、分页、筛选、导出四个功能，每个功能的数据在 `data` 中初始化、方法在 `methods` 中定义、计算属性在 `computed` 中。当你想理解"搜索"这个功能时，你需要在 created（初始化请求）、methods（搜索方法）、computed（搜索结果）、watch（搜索条件变化）之间来回跳转。这就是 **Options API 的逻辑分散问题**。

除此之外：Vue2 的响应式用 `Object.defineProperty` 在初始化时递归遍历 data 所有属性，导致两个问题——新增/删除属性无法监听（需要 `Vue.set`/`Vue.delete`）、数组下标修改无法监听。Vue3 用 Proxy 重写了响应式系统，同时引入了 Composition API 解决逻辑组织问题。

### 2. 核心原理/执行过程

#### 2.1 createApp 替代 new Vue()

```javascript
// Vue2 — 全局 API 污染
Vue.mixin({ created() { /* 全局混入 */ } })  // 影响所有 Vue 实例
new Vue({ el: '#app', render: h => h(App) })

// Vue3 — 应用实例隔离
const app = createApp(App)
app.mixin({ created() { /* 仅影响这个应用 */ } })
app.mount('#app')
```

`createApp` 返回一个应用实例，所有全局 API（`mixin`、`component`、`directive`）都挂在这个实例上，不影响其他 Vue 应用。这在微前端场景下很重要——两个子应用可以各有独立的 Vue 配置。

#### 2.2 Composition API 的执行流程

```text
<script setup> 编译阶段
  ↓
编译为 setup() 函数，在组件创建前执行（beforeCreate 之前）
  ↓
setup 执行：
  ref(0)          → 创建响应式状态
  reactive({})    → 创建响应式对象
  computed(...)   → 注册计算属性
  watch(...)      → 注册监听器
  onMounted(...)  → 注册生命周期钩子
  ↓
这些响应式变量和方法暴露给模板
  ↓
模板渲染 → 访问响应式变量触发 get → 自动收集依赖
```

**setup 执行时组件实例还没创建**，所以 `setup` 中不能访问 `this`（`this` 是 undefined）。如果想在 setup 中获取当前组件实例，可以用 `getCurrentInstance()`（但很少需要这样做）。

#### 2.3 ref 和 reactive 的底层关系

```javascript
import { ref, reactive } from 'vue'

const count = ref(0)
// ref 内部等价于：
// const count = reactive({ value: 0 })

const state = reactive({ count: 0 })
// reactive 直接返回一个 Proxy 对象
```

`ref` 的本质是：如果传入基础类型（string/number/boolean），把它包在一个 `{ value: ... }` 对象中，然后让这个对象变成响应式的。如果传入对象，内部调用 `reactive` 处理。所以 `ref(0)` 和 `reactive({ value: 0 })` 的本质是相同的。

**模板中 ref 自动解包**：

```vue
<template>
  <p>{{ count }}</p>   <!-- 不需要 count.value -->
  <button @click="count++">+1</button>
</template>
<script setup>
const count = ref(0)
</script>
```

**但不是所有地方都解包**：

```javascript
const count = ref(0)
const arr = reactive([count])
console.log(arr[0])      // RefImpl 对象，不是 0！
console.log(arr[0].value) // 0

const state = reactive({ count })
console.log(state.count)  // 0 ✅ reactive 中自动解包
```

### 3. 实际应用场景

#### 场景1：useSearch composable（逻辑复用）

```javascript
// composables/useSearch.js
import { ref, watch } from 'vue'
import { debounce } from 'lodash-es'

export function useSearch(fetchApi) {
  const keyword = ref('')
  const results = ref([])
  const loading = ref(false)

  // 防抖搜索：用户停止输入 300ms 后自动搜索
  const search = debounce(async (kw) => {
    loading.value = true
    results.value = await fetchApi(kw)
    loading.value = false
  }, 300)

  // watch keyword 变化时触发搜索
  watch(keyword, (val) => {
    if (val) search(val)
    else results.value = []
  })

  return { keyword, results, loading }
}
```

```vue
<script setup>
import { useSearch } from './composables/useSearch'

const { keyword, results, loading } = useSearch(
  (kw) => fetch(`/api/search?q=${kw}`).then(r => r.json())
)
</script>

<template>
  <input v-model="keyword" placeholder="搜索..." />
  <div v-if="loading">搜索中...</div>
  <ul v-else>
    <li v-for="item in results">{{ item.title }}</li>
  </ul>
</template>
```

这个 composable 封装了搜索的完整逻辑（防抖、请求、loading），任何需要搜索功能的组件直接引入使用，没有 mixin 的冲突问题。

#### 场景2：Teleport 传送弹窗

```vue
<!-- Modal.vue -->
<script setup>
defineProps({ visible: Boolean })
</script>

<template>
  <!-- Teleport 把弹窗渲染到 body 下，避免被父组件的 overflow:hidden 裁剪 -->
  <Teleport to="body">
    <div v-if="visible" class="modal-overlay">
      <div class="modal-content">
        <slot />
      </div>
    </div>
  </Teleport>
</template>

<style scoped>
.modal-overlay {
  position: fixed; inset: 0; background: rgba(0,0,0,0.5);
  display: flex; align-items: center; justify-content: center;
}
</style>
```

没有 `Teleport` 时，如果 Modal 在父组件中层级很深，父组件的 `overflow: hidden`、`transform`、`z-index` 都可能影响弹窗的显示。`<Teleport to="body">` 把模板内容渲染到 `document.body` 下，但逻辑上仍然属于当前组件的生命周期。

### 4. 常见误区 & 实际项目中的坑

#### 误区1：reactive 解构后丢失响应式

```javascript
const state = reactive({ count: 0, name: 'foo' })

// ❌ 解构后 count 和 name 是普通值
const { count, name } = state
count++  // 不会触发更新

// ✅ 用 toRefs 保持响应式
const { count, name } = toRefs(state)
count.value++  // ✅ 触发更新
```

`toRefs` 把 reactive 对象的每个属性转为 ref。这在从 composable 返回多个响应式值时特别有用——调用方可以解构而不丢失响应式。

#### 误区2：watchEffect 中的异步操作导致依赖追踪不完整

```javascript
watchEffect(async () => {
  const res = await fetch(`/api/user/${state.id}`)
  // 上一行的 state.id 会被追踪（同步阶段）
  state.data = await res.json()
  // 这里的 state.data 不会被追踪（异步阶段——await 之后脱离了同步追踪）
})
```

`watchEffect` 只在同步阶段收集依赖。`await` 之后的代码已经不在同步执行上下文中了，不会被视为依赖。解法：用 `watch` 显式指定依赖，或在同步阶段读取所有需要的响应式变量。

#### 坑：ref 在普通对象和数组中不会自动解包

```javascript
const count = ref(0)
const obj = { count }     // 不会解包，obj.count 是 RefImpl
const arr = reactive([count])  // 不会解包，arr[0] 是 RefImpl

// reactive 中会自动解包
const state = reactive({ count })  // state.count 是 number
```

### 5. 与相关知识的关联 & 对比

| 对比维度 | Options API (Vue2) | Composition API (Vue3) |
|---|---|---|
| 逻辑组织 | 按选项分割（data/methods/computed） | 按功能聚合（composables） |
| 逻辑复用 | mixin（命名冲突、来源不明） | composables（显式导入、无冲突） |
| Tree-shaking | 不支持 | 支持（按需 import） |
| TypeScript | 弱（this 上下文推断困难） | 强 |
| 学习曲线 | 低 | 中 |

### 6. 现代最佳实践（2024-2025）

1. **新项目一律用 `<script setup>`**，不使用 options API。
2. **优先用 `ref` 而不是 `reactive`**——ref 在赋值时不会丢失响应式，解构用 toRefs。
3. **逻辑复用一律用 composables**（`useXxx` 函数），不再使用 mixin。
4. **异步依赖优先用 Suspense**，避免手动维护 loading 状态。
5. **新项目标准技术栈：Vue3 + Vite + TypeScript + Pinia + Vue Router 4**。

### 7. 常见疑问解答

**Q：setup 函数中为什么不能访问 this？**

A：因为 `setup()` 在组件实例创建之前执行。此时组件实例还没有被初始化，data、methods、computed 等都还没创建。`this` 指向的是 `undefined`（严格模式下）。设计意图是让 setup 与组件的其他选项解耦——你在 setup 中定义的变量和方法不依赖组件的 `this` 上下文，更容易提取为复用的 composable。

**Q：`<script setup>` 编译后到底变成了什么？**

A：它编译为一个 `setup()` 函数，所有顶层的 import 和变量声明都成为这个函数的局部变量，模板中可以直接使用。不需要 return 语句，编译器会自动处理。此外 `<script setup>` 还让编译器能更准确地分析模板中绑定的变量来源——比手写 setup() 函数能多做一些编译优化。

**Q：ref 的 .value 自动解包规则到底是什么？**

A：三个规则：① 在模板中自动解包——`{{ count }}` 就是 `count.value`；② 在 `reactive` 中自动解包——`reactive({ count })` 后 `state.count` 就是 number；③ 在普通对象和 `reactive` 数组中不会解包——`obj.count` 和 `arr[0]` 都需要 `.value`。

**关联知识点索引**
- `响应式原理.md` — ref/reactive 的 Proxy 实现细节
- `组件通信.md` — provide/inject 在 Composition API 中的用法
- `模板编译与渲染.md` — `<script setup>` 的编译优化原理
