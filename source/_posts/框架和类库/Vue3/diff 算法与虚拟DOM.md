---
title: Vue3 diff 算法与虚拟 DOM
categories: 
- Vue
tags:
- Vue3
- diff
- 虚拟DOM
- 性能优化
---

## 【面试速答版】

### Q1: "Vue3 的 diff 算法相比 Vue2 有哪些优化？"

两大优化：**PatchFlag + Block Tree**。Vue3 在编译阶段给每个 VNode 打上 PatchFlag（标记哪些属性是动态的），运行时 patch 只比较带标记的属性，静态节点完全跳过。同时通过 Block Tree 把所有动态节点收集到一个扁平数组中，运行时直接遍历这个数组而不递归整棵 VNode 树。列表 diff 方面，Vue3 用**快速 diff**（基于最长递增子序列）替代了 Vue2 的双端 diff，预处理时先跳过前缀/后缀的相同节点，然后对中间未知序列用 LIS 计算最少移动次数，性能更优。

### Q2: "什么是 Block Tree？它是如何工作的？"

Block Tree 是 Vue3 在 VNode 树之上的一层"动态节点索引"。render 函数执行时，`_openBlock()` + `_createBlock()` 把所有带 PatchFlag 的动态节点**扁平化收集**到 `dynamicChildren` 数组中。patch 时只遍历 `dynamicChildren`，跳过整棵静态子树。这样一来，即使组件有 100 个节点，实际只需要比较其中 3-5 个动态节点。

### Q3: "v-for 的 key 有什么用？Vue3 中 key 的优化点是什么？"

key 帮助 diff 算法"识别"同一个节点在不同渲染中是否可复用。没有 key 时，列表的 diff 退化为"逐个位置比较"——头部插入新元素会导致后面所有元素被"更新"而非"移动"。有正确的 key 时，Vue3 的快速 diff 通过最长递增子序列计算出最少移动次数。Vue3 中 key 不仅可以用在 `v-for` 中，也可以用在 `<template v-for>` 上（Vue2 不支持）。**始终使用唯一且稳定的 key**（如后端 id），不要用 index。

## 【深入理解版】

### 1. 这个知识点要解决什么问题？

当响应式数据变化时，Vue 需要更新真实 DOM。最粗暴的方式是把整个 DOM 树销毁重建——但这样性能极差。diff 算法的目标就是：**用最小的 DOM 操作来完成新旧状态的迁移**。

Vue2 的 diff 是全量递归——从根节点开始，逐层比较新旧两棵 VNode 树的所有节点。即使 90% 的节点是静态的（从未变化），每次也要全部比较一遍。Vue3 的目标是：**只比较那些可能变化的部分**。

### 2. 核心原理/执行过程

#### 2.1 整体 patch 流程

```text
响应式数据变化
  ↓
组件重新执行 render → 生成新的 VNode 树 + dynamicChildren
  ↓
patch(prevVNode, nextVNode)
  ↓
判断是否是同一节点（tag + key）
  ├─ 不是 → 卸载旧节点，挂载新节点
  └─ 是 → patchElement
       ├─ 根据 PatchFlag 更新属性/事件/样式
       └─ patchChildren
            ├─ 文本 → 直接替换
            ├─ 静态 → 跳过
            └─ 动态 → patchKeyedChildren（快速 diff）
```

#### 2.2 Block Tree 与动态节点收集

Vue3 在 VNode 上增加了两种数据结构：`patchFlag` 标记动态类型，`dynamicChildren` 收集动态子节点。

```vue
<template>
  <div>
    <span class="static">固定文字</span>
    <span :class="cls">{{ msg }}</span>
    <div>
      <span>完全静态</span>
    </div>
  </div>
</template>
```

编译后的 render 函数（简化）：

```javascript
function render(_ctx, _cache) {
  return (_openBlock(), _createBlock('div', null, [
    _createVNode('span', { class: 'static' }, '固定文字'),
    // 上一行：没有 PatchFlag，也没有被收集到 dynamicChildren

    _createVNode('span', { class: _ctx.cls }, _ctx.msg, 3 /* TEXT, CLASS */),
    // 上一行：PatchFlag = 3，被收集到 dynamicChildren

    _createVNode('div', null, [
      _createVNode('span', null, '完全静态')
    ])
    // 外层 div 的 children 有静态内容，但没有任何动态标记
    // 这个子 div 整体也不会被收集到 dynamicChildren
  ]))
}
```

patch 时：

```javascript
// dynamicChildren 只有一条：[span(PatchFlag=3)]
// patch 只需要比较这一个节点，其他全部跳过
```

如果这个组件有 100 个节点，其中 95 个是静态的，那么 dynamicChildren 可能只有 5 个节点。diff 只需要比较 5 个节点——这就是 Vue3 比 Vue2 快的原因。

#### 2.3 快速 diff（patchKeyedChildren）

当新旧两个子节点列表都是动态的（如 `v-for` 列表），Vue3 使用快速 diff 算法。过程：

```text
旧列表：[a, b, c, d, e]
新列表：[a, b, e, d, c, f]

步骤1: 从头开始比较，找到相同的前缀
  a === a → 跳过
  b === b → 跳过
  ↓
  旧剩余: [c, d, e]
  新剩余: [e, d, c, f]

步骤2: 从尾开始比较，找到相同的后缀
  旧尾: e
  新尾: f → 不同，停止
  ↓
  旧剩余: [c, d, e]
  新剩余: [e, d, c, f]

步骤3: 对剩余部分，用 key 建立映射表，找最长递增子序列
  剩余新列表的 key 序列: [e, d, c, f]
  映射到旧列表索引: [4, 3, 2, -1]（f 是新增）
  最长递增子序列: [2, 3] → 对应 c 和 d → 不用移动
  e 需要移动到 d 后面
  f 是新增，插入到最后
```

结果是：c 和 d 不动，e 移动位置，f 新增。**用最少操作完成了列表更新。**

**为什么最长递增子序列能算出最少移动？** 递增序列中的元素说明它们在旧列表和新列表中的相对顺序一致——不需要移动。剩下的元素才需要移动或新增删除。找最长递增子序列 = 找到最多"不用动"的元素。

### 3. 实际应用场景

#### 场景1：理解 key 的重要性

```vue
<!-- ❌ 不推荐：用 index 做 key -->
<li v-for="(item, index) in list" :key="index">{{ item.name }}</li>

<!-- ✅ 推荐：用唯一 id -->
<li v-for="item in list" :key="item.id">{{ item.name }}</li>
```

当 `list` 头部插入新元素时，用 index 做 key 会导致所有已有 li 的 key 都变了（索引全部后移），diff 认为它们都是新节点——所有 li 都会重新创建。用唯一 id 时，key 不变，diff 能识别出哪些是已有节点（只需移动位置而不是重建）。

#### 场景2：理解 Block Tree 的效果

```vue
<template>
  <div>
    <!-- 下面这个静态列表完全不会被 diff -->
    <ul>
      <li>固定项 1</li>
      <li>固定项 2</li>
      <li>固定项 3</li>
    </ul>

    <!-- 只有这个动态文本需要 diff -->
    <p>{{ message }}</p>
  </div>
</template>
```

即使这个模板看起来有很多节点，Vue3 的 Block Tree 确保只有 `<p>{{ message }}</p>` 被收集到 `dynamicChildren` 中。静态的 `<ul>` 列表在整个组件生命周期中都不会被 diff。

### 4. 常见误区 & 实际项目中的坑

#### 误区1：认为加了一定 key 就一定性能好

key 帮助 diff 准确识别节点，但如果你只有一个静态列表（没有新增/删除/排序），加不加 key 没有区别。另外用随机值作为 key（`:key="Math.random()"`）会导致每次渲染所有节点都被重新创建——性能极差，且会丢失组件状态。

#### 误区2：v-for 中 index 作为 key "也可以"

```vue
<!-- 问题：输入框内容错乱 -->
<div v-for="(item, i) in list" :key="i">
  <input v-model="item.name" />
</div>
<!-- 头部插入新项后，每个输入框的内容会错位 -->
```

用 `:key="i"`，在列表头部插入后，原来索引 0 的输入框现在对应了新数据，React 和 Vue 都会认为"这个 DOM 节点还是之前的"——输入框里残留的旧内容不会清除。

#### 坑：不要在 v-for 中创建内联对象作为 props

```vue
<Child v-for="item in items" :key="item.id" :config="{ name: item.name }" />
```

每次渲染都会创建新的 `{ name: item.name }` 对象，导致 Child 每次都会收到"新的" props 引用，触发不必要的更新。如果 Child 能稳定复用，建议提取为方法或 computed。

### 5. 与相关知识的关联 & 对比

| 对比维度 | Vue2 diff | Vue3 diff |
|---|---|---|
| 整体策略 | 全量递归比较 | Block Tree + 动态节点收集 |
| 静态节点 | 每次重新创建并比较 | 静态提升 + 完全跳过 |
| 属性 diff | 全量比较所有属性 | PatchFlag 按需比较 |
| 列表 diff | 双端 diff（4 指针） | 快速 diff（LIS） |
| key 的作用 | sameVnode 判断 | 基本一致 |

### 6. 现代最佳实践（2024-2025）

1. **始终用唯一且稳定的 key（后端 id 或 uuid）**，不要用 index。
2. **避免在 v-for 循环中创建内联对象/函数**——每次渲染都创建新引用，导致不必要的更新。
3. **如果列表完全静态，不需要加 key**，Block Tree 会直接跳过整个列表。
4. **大规模静态内容用 `v-once`**——渲染一次后永不更新。
5. **合理拆分组件**——组件是独立的 patch 单元，经常变化的部分单独拆分可缩小 diff 范围。

### 7. 常见疑问解答

**Q：Vue3 的快速 diff 和 Vue2 的双端 diff 有什么区别？**

A：双端 diff 用 4 个指针（oldStart、oldEnd、newStart、newEnd）两两比较，匹配成功的节点移动到对应位置。快速 diff 先预处理前缀/后缀的相同节点跳过，然后对中间未知序列用 key → index 映射 + 最长递增子序列计算最少移动次数。快速 diff 代码更简洁（Vue3 核心约 100 行，Vue2 约 300 行），且在大多数场景下效率更高。两者理论上都是 O(n) 复杂度，但快速 diff 的常数更小。

**Q：Block Tree 中的 dynamicChildren 是扁平的动态节点数组，嵌套组件中的动态节点怎么处理？**

A：每个组件实例是一个独立的 patch 单元。父组件的 Block Tree 只收集父组件模板中直接定义的动态节点，不穿透到子组件内部。子组件内部的动态节点由子组件自己的 Block Tree 处理。所以 dynamicChildren 是"当前组件模板这一层"的动态节点，不是整个组件树的。

## 七、Vapor Mode——无虚拟 DOM 的未来

### 7.1 什么是 Vapor Mode

Vapor Mode 是 Vue 团队正在开发的一种**新的编译策略**（非默认，是可选的编译目标）。它的核心思路是：**在编译阶段将模板直接编译为原生 DOM 操作，彻底移除虚拟 DOM**。

```text
传统 Vue 3 运行流程：
  模板 → 编译为 render 函数 → 执行 render → 生成 VNode → diff → patch DOM

Vapor Mode 运行流程：
  模板 → 编译为 DOM 操作函数 → 直接调用 DOM API → 更新真实 DOM
```

### 7.2 为什么需要 Vapor Mode

虚拟 DOM 虽然有 diff 性能优势，但也有不可忽视的开销：

```text
虚拟 DOM 的开销：
├── 运行时要额外创建 VNode 对象（内存分配 + GC 压力）
├── 即使只更新一个属性，也要执行整棵树的 diff 遍历
├── 对于纯静态内容，每次都创建相同的 VNode 再直接 patch
└── Block Tree 虽能跳过静态节点，但 VNode 仍然存在
```

Vapor Mode 直接跳过 VNode 创建和 diff 阶段，将模板编译为等价的命令式 DOM 操作：

```vue
<!-- 模板 -->
<div class="card">
  <p>{{ title }}</p>
  <p>静态描述</p>
</div>
```

```js
// Virtual DOM 模式编译产物
function render(ctx, cache) {
  return (_openBlock(), _createBlock('div', { class: 'card' }, [
    _createVNode('p', null, ctx.title, 1 /* TEXT */),
    _createVNode('p', null, '静态描述'),
  ]))
}

// Vapor Mode 编译产物（示意）
function render(ctx) {
  const div = document.createElement('div');
  div.className = 'card';

  const p1 = document.createElement('p');
  effect(() => { p1.textContent = ctx.title; });  // 只有这一行是动态的

  const p2 = document.createElement('p');
  p2.textContent = '静态描述';  // 只创建一次，永不更新

  div.append(p1, p2);
  return div;
}
```

### 7.3 Virtual DOM vs Vapor Mode 对比

| 维度 | Virtual DOM 模式 | Vapor Mode |
|------|-----------------|------------|
| VNode 创建 | 每次更新都创建 | 无 |
| diff 开销 | 有（即使 Block Tree 也有） | 无 |
| 内存占用 | 高（VNode 对象 + 响应式系统） | 低（仅响应式系统） |
| 首屏渲染 | 需要执行 render + diff | 直接创建 DOM |
| 更新粒度 | 组件级别（组件内比完后应用更新） | 变量级别（精确到具体 DOM 属性） |
| 动态内容 > 静态内容 | 优秀 | 优秀 |
| 静态内容 > 动态内容 | 有浪费（VNode 创建） | 极优（静态内容只执行一次） |
| 缓存友好性 | 低（每次都创建新对象） | 高 |
| bundle 大小 | 完整 Vue 运行时（含 diff 算法） | 小（不含 virtual DOM 相关代码） |

### 7.4 Vapor Mode 的更新粒度

Vapor Mode 中每个动态绑定被直接编译为响应式副作用：

```vue
<template>
  <p :style="{ color: theme }">{{ msg }}</p>
</template>

<!-- Vapor Mode 编译产物（伪代码） -->
import { effect, template, setText, setStyle } from 'vue/vapor';

const tpl = template('<p></p>');  // 只创建一次模板

function render(ctx) {
  const el = tpl();  // 克隆模板

  effect(() => { setText(el, ctx.msg); });     // msg 变化只更新 textContent
  effect(() => { setStyle(el, 'color', ctx.theme); });  // theme 变化只更新 color

  return el;
}
```

当 `msg` 变化时，只有 `setText(el, ctx.msg)` 被执行——**不需要 diff，不需要知道其他节点**。

### 7.5 Vapor Mode 的局限性

```text
├── 不是 "drop-in" 替代
│   Vapor Mode 是一个编译目标，不是默认的运行时策略。
│   目前的规划是：Vapor Mode 编译的组件可以和 Virtual DOM 组件混用。

├── 不直接支持部分功能
│   ├── `<Teleport>`、`<Suspense>` 等内置组件（需额外运行时支持）
│   ├── `<KeepAlive>`（依赖 VNode 缓存）
│   ├── functional components 的某些用法
│   └── 动态组件 `<component :is="...">`（需要运行时解析）

├── 编译产物更大
│   每个模板被直接编译为 DOM 操作代码，会有些体积膨胀。
│   但不需要包含运行时 diff 算法，gzip 后整体可能更小。

├── 与库 / 插件的兼容性
│   第三方组件库可能依赖 VNode 或 render 函数的某些特性，
│   需要适配才能完全在 Vapor Mode 下工作。
```

### 7.6 混合模式（Vapor + Virtual DOM）

Vue 团队的计划不是"二选一"，而是**两者共存**：

```text
┌─────────────────────────────────────────┐
│            应用容器                      │
│  ┌──────────────────────────────────┐   │
│  │ Virtual DOM 组件（兼容层）        │   │
│  │ （使用现有组件库、插件）           │   │
│  └────────────┬─────────────────────┘   │
│               │ props / slots           │
│  ┌────────────▼─────────────────────┐   │
│  │ Vapor 组件（高性能内层）           │   │
│  │ （核心业务、大列表、频繁更新）      │   │
│  └──────────────────────────────────┘   │
│                                         │
│  两者通过统一的跨组件通信机制互操作      │
└─────────────────────────────────────────┘
```

在同一个应用中，大部分组件继续使用 Virtual DOM 模式（兼容性优先），对性能敏感的组件（大列表、频繁更新的图表、动画）使用 Vapor Mode。

### 7.7 当前状态与启用方式

```text
Vue 3.4：Vapor Mode 进入实验阶段（RFC + 原型实现）
Vue 3.5（预计）：可选的 Vapor Mode 编译器
Vue 3.6+（预计）：稳定版 Vapor Mode

启用方式（未来）：
  // vite.config.ts
  import { defineConfig } from 'vite';
  import vue from '@vitejs/plugin-vue';

  export default defineConfig({
    plugins: [
      vue({
        vapor: true,  // 整个应用开启 Vapor Mode
        // 或按文件粒度
      }),
    ],
  });

  // 或按组件选择（通过文件扩展名或配置）
  // Button.vapor.vue → 使用 Vapor Mode
  // Button.vue → 使用 Virtual DOM
```

### 7.8 面试题

#### Q1: Vapor Mode 和虚拟 DOM 哪个更快

```text
不能简单说哪个"更快"，取决于场景：

静态内容多 → Vapor Mode 快（静态只创建一次，无需 VNode）
动态更新频繁 → 两者接近（Vapor Mode 精确到变量，Virtual DOM 到组件）
首次渲染 → Vapor Mode 快（跳过 VNode 创建 + diff）
内存敏感（移动端）→ Vapor Mode 优（无 VNode 对象开销）

Vapor Mode 消除的是虚拟 DOM 的"固定开销"（VNode 分配 + diff），
在静态内容越多的场景下优势越明显。
```

#### Q2: Vapor Mode 会替代虚拟 DOM 吗

```text
不会完全替代。Vapor Mode 和 Virtual DOM 是互补关系：

Vapor Mode 适合：新项目、性能敏感组件、移动端、静态内容多的页面
Virtual DOM 适合：需要兼容第三方库、使用 Teleport/Suspense/KeepAlive、
                动态组件、依赖 render 函数的场景

Vue 团队的目标是让两者无缝共存。
```

#### Q3: Vapor Mode 对现有项目的影响

```text
对现有项目基本无影响——Vapor Mode 是一个可选的编译目标，
默认还是 Virtual DOM 模式。升级 Vue 版本不会自动切换。

如果需要使用 Vapor Mode，需要对依赖库进行兼容性检查
（element-plus / ant-design-vue 等是否适配）。
```

**关联知识点索引**
- `模板编译与渲染.md` — PatchFlag、Block Tree 的编译过程
- `响应式原理.md` — 响应式变化如何触发 patch
- Vue 官方 Vapor Mode RFC
