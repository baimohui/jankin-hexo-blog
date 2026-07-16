---
title: Vue2 vs Vue3 — 全方位异同对比
categories: 
- Vue
tags:
- Vue2
- Vue3
- 对比
- 迁移
---

## 【面试速答版】

### Q1: "Vue3 到底好在哪？你有 Vue2 迁移到 Vue3 的经验吗？"

Vue3 是 Vue2 的**架构重构**而非简单升级。四个维度：**响应式**——Proxy 替代 Object.defineProperty，解决了数组/新增属性监听问题；**API 范式**——Composition API + composables 替代 mixin，解决逻辑复用冲突；**性能**——编译时 PatchFlag + 静态提升 + Block Tree，渲染性能提升 1.3-2 倍；**生态**——Pinia 替代 Vuex（去掉了烦人的 mutations），Vite 替代 Webpack。迁移经验：Options API 和 Composition API 可以混用，建议渐进式迁移——先升级构建工具和依赖库，再逐组件改造。

### Q2: "Vue2 和 Vue3 有哪些破坏性变更？迁移时需要特别注意什么？"

**必知变更**：v-model 默认 prop 从 `value` + `input` 变为 `modelValue` + `update:modelValue`；`.sync` 修饰符废弃，用多 v-model 替代；`$on/$off/$once` 移除（EventBus 不再内置，推荐 mitt）；filters 移除（用 computed 或方法代替）；`$listeners` 合并到 `$attrs`；`v-if` 优先级高于 `v-for`（Vue2 相反）；生命周期 `beforeDestroy` → `beforeUnmount`、`destroyed` → `unmounted`。**迁移建议**：先安装 `@vue/compat` 兼容模式，逐一修复控制台的废弃警告，再移除兼容层。

### Q3: "Vue2 项目如何迁移到 Vue3？必须全部重写吗？"

不需要全部重写。Options API 在 Vue3 中完全兼容，Composition API 可以混用。推荐的迁移步骤：① 升级构建工具到 Vite 或 Webpack 5 并安装 Vue3；② 安装 `@vue/compat`（Vue3 的 Vue2 兼容模式）；③ 一条条修复兼容警告（先解决 filters 和 v-model，再处理生命周期改名和 EventBus）；④ 移除 `@vue/compat`，使用原生 Vue3；⑤ 新组件用 `<script setup>` 写，旧组件逐步改造。核心原则：**基础设施先升级，业务代码逐步改，不急着重写所有组件**。

## 【深入理解版】

### 1. 这个知识点要解决什么问题？

Vue2 发布于 2016 年，随着前端发展，它的几个设计逐渐暴露出问题：

- **Object.defineProperty 的局限**——无法监听数组下标修改、新增/删除属性，需要 hack（重写 7 个数组方法）和 `Vue.set`/`Vue.delete`。
- **Options API 在复杂组件中的维护问题**——200 行的组件，功能逻辑分散在 data/methods/computed/watch 四个区域。
- **mixin 的缺陷**——命名冲突、来源不明、隐式依赖。
- **Tree-shaking 不支持**——即使不用过渡动画、指令，打包时也被包含。
- **全局 API 污染**——`Vue.mixin` 影响所有实例。

Vue3 在 2020 年发布，2022 年成为默认版本，重写了核心模块但保持了 90% 以上的 API 兼容。

### 2. 核心差异对比

#### 2.1 创建实例

```javascript
// Vue2 — 全局 API 可能导致冲突
Vue.mixin({ created() { /* 影响所有实例 */ } })
new Vue({ el: '#app', render: h => h(App) })

// Vue3 — 应用实例隔离
const app = createApp(App)
app.mixin({ created() { /* 仅影响当前应用 */ } })
app.mount('#app')
```

#### 2.2 响应式系统

```javascript
// Vue2 — 需要额外 API 处理新增/删除
this.obj.newProp = 'hello'  // ❌ 不是响应式
Vue.set(this.obj, 'newProp', 'hello')  // ✅
this.arr[0] = 'x'  // ❌ 不是响应式
this.arr.splice(0, 1, 'x')  // ✅

// Vue3 — Proxy 原生支持
state.newProp = 'hello'  // ✅
state.arr[0] = 'x'  // ✅
delete state.obj.prop  // ✅
```

#### 2.3 生命周期对照

| Vue2 | Vue3 | 说明 |
|---|---|---|
| beforeCreate / created | setup() | setup 在 beforeCreate 之前执行 |
| beforeMount | onBeforeMount | 设名 |
| mounted | onMounted | hooks 化 |
| beforeUpdate | onBeforeUpdate | hooks 化 |
| updated | onUpdated | hooks 化 |
| beforeDestroy | onBeforeUnmount | 更名，语义更清晰 |
| destroyed | onUnmounted | 更名 |
| errorCaptured | onErrorCaptured | hooks 化 |

#### 2.4 模板语法

```vue
<!-- Vue2 — 单根节点 -->
<template>
  <div>
    <header />
    <main />
  </div>
</template>

<!-- Vue3 — 多根节点（Fragments）-->
<template>
  <header />
  <main />
</template>
```

```vue
<!-- Vue2 v-model -->
<Child v-model="title" />

<!-- Vue3 v-model（默认 prop 变化）-->
<Child v-model="title" />
<!-- 等价于 :modelValue="title" @update:modelValue="val => title = val" -->

<!-- Vue3 多 v-model -->
<Child v-model:title="title" v-model:content="content" />
```

### 3. 迁移中的常见坑

#### 坑 1：v-model 默认行为变化

Vue2 的 `v-model="count"` 对应 `value` prop 和 `input` 事件。Vue3 改为 `modelValue` prop 和 `update:modelValue` 事件。如果自定义组件升级到 Vue3 但没有改 props 声明，双向绑定功能会失效。

#### 坑 2：filters 移除

```vue
<!-- Vue2 -->
<div>{{ price | currency }}</div>

<!-- Vue3 — 改为方法或 computed -->
<script setup>
const currency = (val) => `¥${val.toFixed(2)}`
</script>
<template>
  <div>{{ currency(price) }}</div>
</template>
```

#### 坑 3：EventBus 移除

Vue2 中 `new Vue()` 作为事件总线 + `$on/$off`。Vue3 中不再支持。推荐：状态共享用 Pinia，跨组件事件通信用 `mitt`（200 字节）。

#### 坑 4：`$listeners` 合并到 `$attrs`

Vue2 中 `$attrs` 只包含非 props 的 attribute，`$listeners` 包含所有监听器。Vue3 中两者合并到 `$attrs`，且 `class` 和 `style` 也包含在 `$attrs` 中。

### 4. 迁移策略

```text
Step 1: 升级构建环境
  将 Webpack 升级到 v5 或迁移到 Vite
  安装 Vue3、Vue Router 4、Pinia（替代 Vuex）

Step 2: 安装 @vue/compat（Vue3 的 Vue2 兼容模式）
  在项目中启用兼容模式，它会自动检测废弃用法并给出警告

Step 3: 逐条修复兼容警告
  优先级从上到下：
  ├─ filters → 改为 computed/method
  ├─ v-model 调整（value → modelValue）
  ├─ $listeners → $attrs
  ├─ 移除 $on/$off/$once
  ├─ 生命周期改名（beforeDestroy → beforeUnmount）
  ├─ .sync → 多 v-model
  └─ v-if / v-for 优先级问题

Step 4: 移除 @vue/compat，使用原生 Vue3

Step 5: 逐步将组件迁移到 Composition API + <script setup>
  新组件直接用新语法，旧组件保持 Options API 也可以
```

### 5. 对比总结

| 对比维度 | Vue2 (2016) | Vue3 (2020) |
|---|---|---|
| 响应式核心 | Object.defineProperty | Proxy |
| API 范式 | Options API | + Composition API |
| 逻辑复用 | mixin | composables |
| TypeScript | 困难 | 一等公民 |
| 渲染性能 | 全量递归 diff | PatchFlag + 静态提升 |
| 包体积 | ~30KB（不可摇树） | ~13KB（可摇树） |
| 多根节点 | 不支持 | Fragment 支持 |
| 状态管理 | Vuex（需 mutations） | Pinia（无 mutations） |
| 构建工具 | Webpack（默认） | Vite（官方推荐） |

### 6. 现代最佳实践（2024-2025）

1. **新项目直接选 Vue3 + Vite + TypeScript + Pinia + Vue Router 4**。
2. **渐进式迁移**——不必一次性重写所有组件。Options API 依然受支持，可以和 Composition API 混用。
3. **放弃 mixin**——所有 mixin 逻辑改为 composables（`useXxx` 函数）。
4. **放弃 EventBus**——状态共享用 Pinia，事件通信用 mitt。
5. **利用 `@vue/compat` 做平滑迁移**——不要直接在旧项目上装 Vue3 原生版。

### 7. 常见疑问解答

**Q：Vue3 中为什么把 beforeDestroy 改成 beforeUnmount？**

A：语义更准确。`destroy` 在英文中暗示"彻底销毁"，而 React 也有类似生命周期方法。Vue 团队认为 `unmount`（卸载）更准确地描述了"组件从 DOM 树中移除"这个行为。同理 `destroyed` → `unmounted`。

**Q：我能在 Vue2 项目中使用 Composition API 吗？**

A：可以——安装 `@vue/composition-api` 插件后，Vue2 项目也能用 `ref`、`reactive`、`computed`、`watch`、`onMounted` 等。但有一些限制：① `defineComponent` 是零等的；② `<script setup>` 不可用（这是编译器特性，Vue2 编译器不支持）；③ `onUnmounted` 在 Vue2 中映射为 `destroyed`。这只是一种过渡方案，最终还是需要迁移到 Vue3。

**Q：`@vue/compat` 兼容模式会拖慢性能吗？**

A：会有一点性能损耗，因为兼容层需要额外做 Vue2 API 的适配转换。但它只应该作为**迁移过程的工具**——你逐条修复废弃警告，修复一条就少一条兼容代码。当所有 warnings 都修完后，就可以移除 `@vue/compat`，使用原生 Vue3。**不要长期在生产环境使用兼容模式**。

**关联知识点索引**
- 所有 Vue3 文档（本目录）
- 所有 Vue2 文档（`../Vue2/`）
