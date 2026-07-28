---
title: Vue 性能优化
categories: 
- 性能优化
tags:
- Vue
- 性能优化
- 渲染优化
- computed
- 虚拟滚动
---

## 一、Vue 性能问题的本质

Vue 的响应式系统会自动追踪依赖，理论上只有真正依赖的数据变化时才会触发更新。但实践中常见的问题包括：**模板中不合理的计算**、**watch 过度触发**、**列表渲染缺乏 key** 以及 **组件粒度过大**。<!--more-->

```text
响应式数据变化
  │
  ├── Vue 的优化优势
  │   ├── 自动追踪依赖 → 不需要手动 memo
  │   ├── 虚拟 DOM diff → 最小化 DOM 操作
  │   └── 异步更新队列 → 批量合并更新
  │
  └── 常见性能陷阱
      ├── 模板中执行昂贵计算 → 每次渲染都执行
      ├── 列表没有 key → 全部重新创建
      ├── watch 监听大对象 → 深度监听开销高
      └── 组件嵌套过深 → 大量 diff 开销
```

## 二、优化手段详解

### 计算属性 vs 方法——缓存计算结果

```vue
<!-- ❌ 方法：每次渲染都重新计算 -->
<template>
  <div>{{ formatList(items) }}</div>
</template>
<script setup>
function formatList(items) {
  return items.sort((a, b) => b.value - a.value).map(formatItem);
}
</script>

<!-- ✅ 计算属性：只有依赖变化时才重新计算 -->
<script setup>
import { computed } from 'vue';

const formattedList = computed(() => {
  return props.items.sort((a, b) => b.value - a.value).map(formatItem);
});
</script>
<template>
  <div>{{ formattedList }}</div>
</template>
```

```text
├── computed → 有缓存，依赖不变就不重算 → ✅ 推荐
├── method  → 无缓存，每次渲染都执行 → ❌ 昂贵计算避免使用
└── watch   → 监听变化执行副作用 → 不要用于模板数据的衍生
```

### v-memo——跳过不变列表项的重渲染

```vue
<!-- v-memo：依赖数组中的值未变化时，跳过该块的重渲染 -->
<template>
  <div v-for="item in list" :key="item.id" v-memo="[item.id, item.updatedAt]">
    <ListItem :item="item" />
  </div>
</template>
```

```text
适用场景：
├── 大列表中，只有少数项变化 → v-memo 跳过未变项
├── 配合虚拟滚动 → 进一步减少 diff 工作量
└── 列表项包含复杂子组件 → 避免深层 diff
```

### v-once / v-memo——稳定内容只渲染一次

```vue
<!-- v-once：只在首次渲染，不再更新 -->
<template>
  <div v-once>
    <h1>固定的标题</h1>
    <p>不会变化的内容</p>
  </div>
</template>

<!-- v-memo="[]"：空依赖表示永不更新 -->
<template>
  <div v-memo="[]">
    <UserInfo :user="user" />
  </div>
</template>
```

### 函数式组件——轻量无状态组件

```vue
<!-- 函数式组件：没有实例、没有 this、开销极小 -->
<template functional>
  <div class="cell">{{ props.value }}</div>
</template>

<!-- Vue 3 中推荐用普通组件，因为编译优化已使其接近函数式组件的性能 -->
```

### 组件拆分——控制更新粒度

```vue
<!-- ❌ 单个组件包含大量不相关的状态 -->
<template>
  <div>
    <input v-model="search" placeholder="搜索" />   <!-- 搜索输入 -->
    <div v-for="item in filteredList">...</div>     <!-- 列表渲染 -->
    <aside>
      <UserProfile />                              <!-- 用户信息 -->
      <NotificationList />                         <!-- 通知列表 -->
    </aside>
  </div>
</template>

<!-- ✅ 拆分为独立组件，各自追踪自己的响应式依赖 -->
<template>
  <div>
    <SearchPanel />
    <ItemList :items="items" />
    <SidePanel />
  </div>
</template>
```

```text
原则：
├── 每个组件只关注自己的数据 → 数据变化时只有该组件更新
├── 不要在一个组件里混合频繁变化和几乎不变的内容
└── 状态尽量下放到叶子组件
```

### 列表与 Key

```vue
<!-- ❌ key 使用 index → 删除/插入时所有列表项重新创建 -->
<div v-for="(item, index) in items" :key="index">

<!-- ✅ key 使用唯一 id → Vue 正确复用 DOM -->
<div v-for="item in items" :key="item.id">
```

## 三、虚拟滚动

```vue
<template>
  <div ref="container" class="virtual-list" @scroll="onScroll">
    <div class="scroll-bar" :style="{ height: totalHeight + 'px' }">
      <div
        class="visible-items"
        :style="{ transform: `translateY(${offsetY}px)` }"
      >
        <div
          v-for="item in visibleItems"
          :key="item.id"
          class="list-item"
        >
          {{ item.name }}
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed } from 'vue';

const props = defineProps({ items: Array, itemHeight: { type: Number, default: 50 } });
const container = ref(null);
const scrollTop = ref(0);
const containerHeight = ref(600);

const totalHeight = computed(() => props.items.length * props.itemHeight);

const visibleCount = computed(() => Math.ceil(containerHeight.value / props.itemHeight) + 2);
const startIndex = computed(() => Math.floor(scrollTop.value / props.itemHeight));

const visibleItems = computed(() => {
  return props.items.slice(startIndex.value, startIndex.value + visibleCount.value);
});

const offsetY = computed(() => startIndex.value * props.itemHeight);

function onScroll() {
  scrollTop.value = container.value.scrollTop;
}
</script>
```

社区推荐库：

| 库 | Vue 版本 | 特点 |
|----|----------|------|
| vue-virtual-scroller | Vue 2/3 | 功能全面，支持动态高度 |
| vueuc (VUse) | Vue 3 | Naive UI 同作者，轻量 |
| TanStack Virtual | 框架无关 | 最灵活 |

## 四、watch 优化

### 避免深度监听大对象

```vue
<!-- ❌ 深度监听大对象 → 每次变更都要遍历整个对象 -->
<script setup>
watch(
  () => props.data,
  (val) => { /* 处理 */ },
  { deep: true }
);
</script>

<!-- ✅ 明确指定监听的路径 -->
<script setup>
watch(
  () => props.data.id,
  (id) => { /* 只在 id 变化时触发 */ }
);
</script>
```

### 避免 watch 中执行昂贵操作

```vue
<!-- ❌ watch 中直接执行耗时操作 -->
<script setup>
watch(keyword, async (val) => {
  results.value = await fetch(`/api/search?q=${val}`);
});
</script>

<!-- ✅ 防抖 + 取消上次请求 -->
<script setup>
import { ref, watch } from 'vue';

let abortController = null;

watch(keyword, (val) => {
  if (abortController) abortController.abort();
  abortController = new AbortController();

  const timer = setTimeout(async () => {
    results.value = await fetch(`/api/search?q=${val}`, {
      signal: abortController.signal,
    }).then(r => r.json());
  }, 300);

  // 下次 watch 触发时清除定时器
  watch(keyword, () => clearTimeout(timer), { once: true });
});
</script>
```

## 五、Immutable 数据模式

Vue 的响应式系统基于 Proxy，不需要像 React 那样手动保证不可变。但过度使用 reactive 包裹大对象会带来额外开销：

```vue
<!-- ✅ 对于大型静态数据，用 ref 或普通变量 -->
<script setup>
import { ref } from 'vue';

// ✅ 大数组用 ref 包裹整体
const items = ref([]);

// ✅ 更新时整体替换，而非逐个修改
function updateItems(newData) {
  items.value = newData;  // 整体替换，Vue 只追踪引用变化
}
</script>
```

## 六、编译优化（Vue 3）

Vue 3 在编译层面做了大量优化，通常不需要手动干预：

| 优化 | 说明 |
|------|------|
| 静态提升 | 不变的模板内容提升到 render 函数之外，只创建一次 |
| 补丁标记 | 编译时标记动态节点，diff 时只检查动态部分 |
| 缓存事件 | 内联事件处理函数自动缓存 |
| 树结构打平 | 减少递归遍历的深度 |

```text
Vue 3 默认优化：
├── 模板中静态内容 → 只创建一次，后续跳过
├── 动态绑定（:class / :style / v-if）→ 精确标记，只 diff 变化
├── 事件监听 → 自动缓存 handler 引用
└── v-for + v-if → 编译警告（优先用 computed 过滤）
```

## 七、排查手段

### Vue DevTools

```text
操作步骤：
1. 安装 Vue DevTools 浏览器扩展
2. 打开 DevTools → Vue 面板
3. Components → 查看组件树和更新频率
4. Performance → 录制操作，查看渲染时间
```

```text
排查问题：
├── 频繁更新的组件高亮 → 是否更新了不必要的数据？
├── 组件渲染耗时过高 → 是否模板中有昂贵计算？
├── 大量组件同时更新 → 是否状态提升过高？
└── Timeline 中渲染频率过高 → 是否 watch 未加防抖？
```

### 组件渲染计数

```vue
<script setup>
// 开发阶段监控组件渲染次数
const renderCount = ref(0);

onBeforeUpdate(() => {
  renderCount.value++;
  if (renderCount.value > 5) {
    console.warn('组件渲染次数过多，请检查优化');
  }
});
</script>
```

### 检查 watcher 数量

```vue
<script setup>
import { onMounted, getCurrentInstance } from 'vue';

onMounted(() => {
  const instance = getCurrentInstance();
  const watchers = instance?.proxy?.$forceUpdate
    ? '请使用 Vue DevTools 查看'
    : instance?.proxy;

  // 查看当前组件的 watcher 数量
  console.log('组件内部 watcher:', Object.keys(instance?.proxy || {}).length);
});
</script>
```

## 八、排查清单

| 现象 | 排查手段 | 常用方案 |
|------|----------|----------|
| 输入框输入卡顿 | Vue DevTools Timeline | 拆分组件、computed |
| 列表滚动卡顿 | 渲染计数 | 虚拟滚动、v-memo |
| 大量列表渲染慢 | Performance 火焰图 | 虚拟滚动、唯一 key |
| 页面切换白屏 | Network 瀑布图 | defineAsyncComponent |
| watch 频繁触发 | 检查依赖路径 | 明确指定字段、防抖 |
| 模板中复杂计算 | Vue DevTools | computed 替代方法 |
| 组件更新范围过大 | 组件树检查 | 拆分组件、状态下移 |
