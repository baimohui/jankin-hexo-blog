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

**关联知识点索引**
- `模板编译与渲染.md` — PatchFlag、Block Tree 的编译过程
- `响应式原理.md` — 响应式变化如何触发 patch
