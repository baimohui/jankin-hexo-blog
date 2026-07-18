---
title: React 渲染机制与性能优化 — Fiber、diff、reconciliation
categories: 
- React
tags:
- React
- Fiber
- diff
- reconciliation
- 性能优化
---

## 【面试速答版】

<!-- more -->
### Q1: "React 的 reconciliation 过程是怎样的？Fiber 是什么？解决了什么问题？"

Reconciliation 是 React **比较新旧 Virtual DOM 树**的过程。React 16 之前的 Stack Reconciler 是**同步递归**的——一旦开始渲染，必须遍历完整棵组件树才能停止。这导致大更新阻塞主线程，用户无法点击、输入、滚动。**Fiber** 是 React 16 引入的新协调引擎，把渲染拆分为可中断的小任务（Fiber Node），每个节点执行完后检查是否有更高优先级的任务（如用户输入）。Fiber 通过链表遍历替代递归，实现了**可中断渲染**、**优先级调度**和**错误边界**。渲染分为两个阶段：**Render 阶段**（可中断，对比新旧 VNode 生成 effect list）和 **Commit 阶段**（不可中断，把 effect list 应用到真实 DOM）。

### Q2: "React.memo、useMemo、useCallback 的区别和适用场景是什么？"

三者都是为了**减少不必要的重渲染**。`React.memo(Component)` 包裹组件，对 props 做浅比较——props 没变就跳过重渲染。`useMemo` 缓存**计算结果**，依赖不变时复用上次结果。`useCallback` 缓存**函数引用**，保证子组件的 memo 不因内联函数而失效。适用场景：对象/函数作为 props 传给被 memo 包裹的子组件时，必须用 useMemo/useCallback 保持引用稳定。**不需要滥用**——简单计算或不涉及子组件的场景下，useMemo 反而有额外开销。

### Q3: "React 18 的自动批处理和 startTransition 是什么？"

**自动批处理（Automatic Batching）**：React 18 默认把多个 `setState` 合并到一次重新渲染中，不管它们写在 setTimeout、Promise 还是原生事件中。React 17 及之前只在 React 合成事件中批处理。**`startTransition`** 标记一个更新为"非紧急"——如果同时有紧急更新（用户输入）和非紧急更新（过滤列表结果），React 优先处理紧急更新，非紧急更新可被中断。`useDeferredValue` 是 `startTransition` 的另一种形式——延迟一个值的更新，常用于输入框 + 大列表渲染场景。

## 【深入理解版】

### 1. 这个知识点要解决什么问题？

React 的渲染机制要回答的根本问题是：**数据变了，UI 怎么更新？**

最简单的方式是：每次 setState 时把整个 DOM 树销毁重建。但这样性能极差。React 的策略是：
1. 用 **Virtual DOM**（JavaScript 对象树）模拟真实 DOM
2. 数据变化时生成**新的 Virtual DOM 树**
3. 用 **diff（reconciliation）** 比较新旧两棵树，找出最小差异
4. **只更新有差异的那部分真实 DOM**

在 React 16 之前，第 3 步（diff）是**同步递归**的——从根节点一直递归到叶子节点，遇到复杂的组件树和大量数据变化时，这个过程可能持续几百毫秒。浏览器在这几百毫秒里无法响应用户输入、点击、滚动——页面"卡死了"。

**Fiber 就是为了解决这个问题而生的。**它把递归遍历改成了链表遍历，把一次不可中断的大任务拆成了很多可中断的小任务，每个小任务执行完后问浏览器："还有没有更高优先级的任务需要处理？比如用户输入？" 这就是"可中断渲染"。

### 2. 核心原理/执行过程

#### 2.1 从递归到链表：Fiber 的数据结构

先看传统递归 diff 的问题。假设组件树的结构是：

```text
App
 ├─ Header
 ├─ Sidebar
 │    └─ Menu
 └─ Content
      ├─ Article
      └─ Comments
```

传统的递归遍历：从 App 开始 → Header → Sidebar → Menu → Content → Article → Comments。**一旦开始，必须全部遍历完才能停止。**

Fiber 的做法：把每个组件/元素变成一个 **Fiber 节点**，用三个指针把树变成链表：

- `return`：指向父节点
- `child`：指向第一个子节点  
- `sibling`：指向下一个兄弟节点

```text
App (return: null, child: Header)
 ↓
Header (return: App, sibling: Sidebar, child: null)
 ↓
Sidebar (return: App, sibling: Content, child: Menu)
 ↓
Menu (return: Sidebar, sibling: null, child: null)
 ↓  ← 这里可以暂停！检查是否有高优先级任务
Content (return: App, sibling: null, child: Article)
...
```

遍历顺序变成了**深度优先的链表遍历**：先往下走（child），走到底再往右走（sibling），再往上（return）。而且每访问一个节点，都可以停下来问浏览器："还要继续吗？"

**Fiber 节点的核心字段**（简化版）：

```javascript
const fiberNode = {
  tag: FunctionComponent | ClassComponent | HostComponent,  // 节点类型
  key: null,                  // 用于 diff 的 key
  type: null,                 // 组件函数/类，或 HTML 标签名
  stateNode: null,            // 真实 DOM 或组件实例

  return: null,  // 父节点
  child: null,   // 第一个子节点
  sibling: null, // 下一个兄弟节点

  pendingProps: null,   // 新 props
  memoizedProps: null,  // 旧 props
  memoizedState: null,  // 旧 state（hooks 链表也挂在这里）

  effectTag: null,  // 标记：Placement（新增）/ Update（更新）/ Deletion（删除）
  nextEffect: null, // effect list 指针

  alternate: null,  // workInProgress ↔ current 双缓冲
}
```

#### 2.2 双缓冲（Double Buffering）

React 内部同时维护两棵 Fiber 树：

- **current 树**：对应当前已渲染到页面上的 UI
- **workInProgress（WIP）树**：正在构建中的新的 UI

```text
current Fiber tree（已显示）
  ↓ setState 触发更新
创建 workInProgress 树（复用 current 的节点，浅拷贝）
  ↓ Render 阶段
逐个处理 WIP 树的节点 → 标记 effect（增/删/改）
  ↓ Commit 阶段
把 effect list 应用到真实 DOM
  ↓
将 current 指针指向 WIP 树（下一次更新时，原来的 current 成为 WIP 的复用源）
```

双缓冲的好处：**构建过程中不操作真实 DOM**，所有变化都打在 effect tag 上，最后一次性提交。用户不会看到"渲染到一半"的中间状态。

#### 2.3 Render 阶段 vs Commit 阶段

```text
setState / useState dispatch
  ↓
调度器（Scheduler）：决定优先级，安排任务
  ↓
┌──────────────────────────────────┐
│ Render 阶段（可中断）             │
│                                  │
│ beginWork(fiber) → 处理当前节点   │
│   └─ 对比 current 和新的 ReactElement │
│   └─ 如果不同 → 标记 effectTag   │
│   └─ 返回 child → 处理子节点     │
│                                  │
│ completeWork(fiber) → 子节点处理完 │
│   └─ 构建 effect list            │
│                                  │
│ ← 可被高优先级任务中断 →          │
└──────────────────────────────────┘
  ↓
┌──────────────────────────────────┐
│ Commit 阶段（不可中断，同步执行）  │
│                                  │
│ 遍历 effect list                 │
│ 执行 DOM 操作（增/删/改）         │
│ 执行 useLayoutEffect（同步）      │
│ 浏览器绘制                        │
│ 执行 useEffect（异步）            │
└──────────────────────────────────┘
```

**关键区别**：Render 阶段可以被打断，Commit 阶段不行——因为一旦开始操作 DOM，中断会导致 UI 不一致。

#### 2.4 React 的 diff 策略

React 的 diff 算法基于三个假设（保证了 O(n) 复杂度，而非 O(n³)）：

1. **同层比较**——只比较同一层级的节点，不跨层级移动
2. **不同类型 = 不同内容**——`<div>` 变成 `<span>`，直接销毁重建
3. **key 属性**——同一层级通过 key 匹配哪些节点可以复用

```jsx
// 例1：不同类型 → 直接替换
// 旧：<div><Child /></div>
// 新：<span><Child /></span>
// → React 销毁整个 div+Child，创建新的 span+Child

// 例2：同类型无 key → 逐个比较
// 旧：<li>A</li> <li>B</li> <li>C</li>
// 新：<li>A</li> <li>B</li> <li>D</li>
// → 前两个复用，第三个从 C 更新为 D

// 例3：有 key → 通过 key 匹配
// 旧：<li key="a">A</li> <li key="b">B</li>
// 新：<li key="b">B</li> <li key="a">A</li>
// → React 识别出只是交换位置，标记为"移动"而非"销毁+新建"
```

这就是为什么列表渲染中 key 如此重要——没有 key 时，列表头部插入会导致所有元素被重建；有正确的 key 时，React 能精准复用。

### 3. 实际应用场景

#### 场景1：使用 React.memo 阻止不必要的重渲染

```jsx
// 一个"重量级"子组件
const ExpensiveList = React.memo(function ExpensiveList({ items }) {
  console.log('ExpensiveList render')
  return (
    <ul>
      {items.map(item => (
        <li key={item.id}>{item.name}</li>
      ))}
    </ul>
  )
})

function App() {
  const [count, setCount] = useState(0)
  const [items] = useState([
    { id: 1, name: 'A' },
    { id: 2, name: 'B' },
  ])

  return (
    <div>
      <button onClick={() => setCount(c => c + 1)}>{count}</button>
      <ExpensiveList items={items} />
    </div>
  )
}
```

如果不加 `React.memo`，每次点击按钮（`count` 变化）都会触发 `App` 重渲染 → `ExpensiveList` 跟着重渲染。加上 `React.memo` 后，`items` 引用没变，`ExpensiveList` 跳过渲染。注意这里 `items` 是用 `useState` 初始化的，引用稳定。如果 `items` 每次都重新创建（比如 `const items = [1,2,3]` 写在组件内部），那 memo 就没效果了，因为每次渲染都创建新数组。

**`React.memo` 的原理**：它对新旧 props 做**浅比较**——比较每个 props 字段的引用是否变化。如果所有字段的引用都相同，就复用上次渲染结果，跳过本次渲染。

#### 场景2：useTransition 保持交互流畅

```jsx
import { useState, useTransition } from 'react'

function SearchPage() {
  const [query, setQuery] = useState('')
  const [isPending, startTransition] = useTransition()
  const [results, setResults] = useState([])

  const handleChange = (e) => {
    // 紧急更新：更新输入框的值（用户立刻能看到输入的文字）
    setQuery(e.target.value)

    // 非紧急更新：过滤结果可以"等一下"
    startTransition(() => {
      // 这里的更新在紧急更新之后才执行，如果输入频繁变化，会被中断
      setResults(filterBigList(e.target.value))
    })
  }

  return (
    <div>
      <input value={query} onChange={handleChange} />
      {isPending ? <Spinner /> : <ResultsList results={results} />}
      {/* isPending 表示"非紧急更新进行中" */}
    </div>
  )
}
```

没有 `startTransition` 时，大列表的过滤渲染会阻塞输入——每按一个键都要等待列表过滤完成才能看到新输入。有了 `startTransition`，输入更新优先，列表过滤可以延迟或被中断。

#### 场景3：虚拟列表 + key（大数据量）

```jsx
function BigList({ items }) {
  return (
    <div>
      {items.map((item, index) => (
        // ❌ 错误：用 index 作为 key
        // 如果列表头部插入新项，所有已有项的 key 都变了 → 全部重建
        <ListItem key={index} data={item} />
      ))}
    </div>
  )
}

// ✅ 正确：用唯一 id
{items.map(item => (
  <ListItem key={item.id} data={item} />
))}

// ✅ 如果数据量巨大（数万行），配合虚拟滚动（react-window）
import { FixedSizeList } from 'react-window'

<FixedSizeList height={600} itemCount={items.length} itemSize={50}>
  {({ index, style }) => (
    <div style={style}>
      <ListItem key={items[index].id} data={items[index]} />
    </div>
  )}
</FixedSizeList>
```

### 4. 常见误区 & 实际项目中的坑

#### 误区1：React.memo 一定能阻止子组件重渲染

```jsx
// ❌ 父组件每次渲染都创建新对象 → memo 无效
function Parent() {
  const config = { theme: 'dark' }  // 每次渲染新对象
  return <Child config={config} />
}

// ✅ 用 useMemo 保持引用稳定
function Parent() {
  const config = useMemo(() => ({ theme: 'dark' }), [])
  return <Child config={config} />
}
```

`React.memo` 的浅比较只是检查"引用是否变了"，如果每次都创建新引用，memo 相当于没起作用。

#### 误区2：认为 key 只影响性能，不影响功能

```jsx
const [items, setItems] = useState(['a', 'b', 'c'])

// 渲染：
{items.map((item, i) => <Input key={i} defaultValue={item} />)}

// 操作：在头部插入 'x'
setItems(['x', ...items])
```

用 `key={i}` 时，插入 'x' 后，原来 key=0 的 Input 显示的是 'a'，现在 key=0 的 Input 显示的是 'x'。React 认为 key=0 的 Input 还是同一个组件（key 没变），所以**不会重新创建**——Input 内部保留下来的值是 'a' 对应的状态。**你看到的输入框内容错乱了。**用唯一的 `key={item}` 才能解决。

#### 误区3：把 "render 阶段" 的重渲染当成性能问题过度优化

```javascript
function App() {
  // ❌ 过度优化：父组件重渲染不意味着子组件一定重绘 DOM
  // React 的 diff 只在变化处更新 DOM
  return <HeavyComponent />
}

// 正确的优化时机：
// 1. 通过 Profiler 确实测到了性能瓶颈
// 2. 子组件渲染开销大（大数据列表、复杂图表）
// 3. 子组件频繁因父组件无关的状态变化而重渲染
```

React 的 diff 本来就会跳过未变化的部分。`React.memo` 只是避免"执行组件函数生成 VNode"这一步——如果组件函数本身不重，加 memo 反而不划算。

### 5. 与相关知识的关联 & 对比

| 对比维度 | React 15 (Stack) | React 16+ (Fiber) |
|---|---|---|
| 协调引擎 | 同步递归遍历 | 可中断的链表遍历 |
| 优先级 | 无 | Lane 模型 |
| 中断恢复 | 不支持 | 支持 |
| 错误边界 | 无 | componentDidCatch |
| 并发模式 | 无 | React 18 createRoot |
| Suspense | 无 | 支持 |

| 优化手段 | 作用 | 配合使用 |
|---|---|---|
| React.memo | 浅比较 props，跳过组件渲染 | + useCallback/useMemo |
| useMemo | 缓存计算结果 | 高开销计算 |
| useCallback | 缓存函数引用 | + React.memo |
| useTransition | 标记非紧急更新 | 输入框 + 列表过滤 |
| useDeferredValue | 延迟低优先级值 | 同上 |
| 虚拟列表（react-window） | 只渲染可见区域 | 大列表 |
| 代码分割（React.lazy） | 按路由/组件拆分 | Suspense |

### 6. 现代最佳实践（2024-2025）

1. **用 `createRoot` 而不是 `ReactDOM.render`**——开启 React 18 的自动批处理和并发特性。
2. **先不加 memo/useMemo/useCallback，Profiler 定位瓶颈后再加**——不要猜测性能问题。
3. **大型列表必须用虚拟滚动**——`react-window` 或 `@tanstack/react-virtual`。
4. **`startTransition` 和 `useDeferredValue` 是 React 18 最值得使用的 API**——用它们替代手动防抖或 setTimeout 来做"非紧急更新"。
5. **避免渲染时创建新引用**——对象、函数、数组尽量从 props 中取或用 useMemo/useCallback 包装。
6. **拆散大的 state**——一个组件里的多个 useState 比一个大的 useState 更好，避免无关变化触发不必要的渲染。

### 7. 常见疑问解答

**Q：Fiber 是如何实现"可中断"的？浏览器怎么知道什么时候该中断？**

A：依赖浏览器的 **requestIdleCallback** 或 React 自己实现的 Scheduler。每处理完一个 Fiber 节点后，React 检查"当前帧是否还有剩余时间"。如果剩余时间为 0（或即将为 0），就把控制权交还给浏览器，等下一帧再继续。这就是时间切片（Time Slicing）。

**Q：Commit 阶段为什么不能中断？**

A：因为 Commit 阶段已经**直接操作真实 DOM**了——可能正在插入一个元素、修改一个文本节点、删除一个子节点。如果在中间中断，DOM 会处于"半更新"状态：一部分是新 UI，一部分还是旧 UI，用户看到的就是一个"撕裂"的页面。所以 Commit 阶段必须同步一次性完成。

**Q：`key` 用了随机值（如 `key={Math.random()}`）会怎么样？**

A：每次渲染时 key 都不同，React 认为所有节点都是全新的→全部卸载重建→性能极差。更严重的是：如果组件有 `useState` 状态，重建后会丢失；如果有输入框，用户输入的内容会被清空。永远不要用随机值做 key。

**关联知识点索引**
- `React 核心概念.md` — Virtual DOM、ReactElement 基础
- `React Hooks 详解.md` — hooks 与渲染周期的关系
- `React 组件通信与状态管理.md` — Context 重渲染的性能问题
