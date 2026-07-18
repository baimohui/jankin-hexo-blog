---
title: Preact 介绍与 React 差异对比
categories: 
- React
tags:
- React
- Preact
- 轻量框架
- 性能优化
---

## 【面试速答版】

<!-- more -->
### Q1: "Preact 是什么？它和 React 的核心区别是什么？"

Preact 是 React 的**轻量级替代品**，体积仅约 3KB（React + ReactDOM 约 40KB）。它实现了 React 的核心 API：JSX、组件、props/state、hooks（useState/useEffect/useRef 等），对大多数应用兼容度超过 95%。核心区别在于**内部实现更加精简**：去掉合成事件系统（SyntheticEvent，直接使用原生事件）、简化 diff 算法（不用完整 Fiber 架构，用递归同步 diff）、不实现 Suspense/Profiler 等边缘功能。通过 `preact/compat` 模块可以达到近乎 100% 的 React API 兼容（增加约 2KB），可以直接使用 React 生态库（如 Ant Design、React Router）。

### Q2: "Preact 如何做到 3KB 的体积？它砍掉了 React 的哪些能力？"

Preact 缩小体积的策略：① **去掉合成事件系统**——React 的事件包装 + 事件池 + 跨浏览器兼容占了约 10KB，Preact 直接用原生事件对象；② **简化协调引擎**——不用完整的 Fiber 架构（React 的 diff 核心约 3000 行），Preact 用约 200 行的递归同步 diff；③ **不实现不常用 API**：如 createPortal 需要 compat、Suspense 仅有限支持、Profiler 不支持、Children API 需要 compat。这些被砍掉的功能对大多数业务应用来说用不到或很少用到。

### Q3: "什么场景下适合用 Preact 替代 React？迁移成本高吗？"

适合场景：**体积敏感的移动端 H5、广告落地页、浏览器扩展、低端设备**。这些场景下首屏加载时间直接关乎转化率，省掉几十 KB 框架体积非常有价值。迁移成本很低——配置构建工具的 alias，把 `react` 和 `react-dom` 指向 `preact/compat`，大多数应用无需改动代码就能跑通。需要注意的坑：用了 `useSyncExternalStore`（React 18 新 API）的库不兼容、Suspense 相关的代码需要测试。

## 【深入理解版】

### 1. 这个知识点要解决什么问题？

React 18 + ReactDOM 打包后约 40KB（gzip 后约 12-15KB）。对于大多数 Web 应用这个体积是合理的，但在一些特殊场景下每一 KB 都很珍贵：

- **移动端 H5 页面**——首屏加载时间每多 100ms，转化率下降约 7%
- **低端设备 / 新兴市场**——内存小、网络慢，40KB 的框架是显著负担
- **浏览器扩展**——Chrome Web Store 对包体积有限制
- **广告落地页 / 营销页面**——只需展示简单交互，不需要完整框架
- **微前端子应用**——每个子应用都引入 React 会导致基座体积膨胀

Preact 的作者 Jason Miller 的目标是：**"用 3KB 实现 React 的核心能力"**。它不是一个"独立框架"，而是 React 的一个兼容层——你在 Preact 中写的代码几乎就是 React 代码。

### 2. 核心原理/执行过程

#### 2.1 Preact 事件系统的差异

React 的事件系统是自己实现的 **SyntheticEvent（合成事件）**，所有事件委托到 root 节点，做跨浏览器兼容包装。这个系统大约占 React 体积的 25%。Preact 直接使用**原生事件**：

```javascript
// React：事件委托 + 合成事件包装
<button onClick={handleClick}>点击</button>
// React 内部：把所有 onClick 委托到 root 节点
// 回调接收的是 SyntheticEvent 对象（包装了原生事件）

// Preact：原生事件直接绑定
<button onClick={handleClick}>点击</button>
// Preact 内部：直接调用 addEventListener
// 回调接收的是浏览器原生 Event 对象
```

这带来的影响：① **体积更小**（不用实现事件包装层）；② **行为更接近 DOM 规范**（没有合成事件的那些边界特性）；③ **一些 React 特有的模式不兼容**（如 `e.persist()` 在 Preact 中没有意义，因为不存在事件池）。

#### 2.2 diff 算法的简化

React 的 Fiber 架构把 diff 拆分为可中断的小任务（7000+ 行核心代码）。Preact 使用**简化的递归 diff**，与 React 16 之前的 Stack Reconciler 类似——同步递归遍历，不可中断：

```javascript
// Preact diff 核心（极度简化）
function diff(dom, newVNode, oldVNode) {
  if (oldVNode == null) {
    // 创建新 DOM
    dom = document.createElement(newVNode.type)
  } else if (newVNode.type !== oldVNode.type) {
    // 不同类型 → 替换
    dom = replaceElement(dom, newVNode)
  }

  // 更新属性
  diffAttributes(dom, newVNode.props, oldVNode?.props)

  // 递归 diff 子节点
  diffChildren(dom, newVNode.children, oldVNode?.children)

  return dom
}
```

这意味着 Preact **没有并发模式**、**没有 Suspense**、**没有时间切片**。但 Preact 作者认为：对于 99% 的应用，单次 diff 耗时在 5ms 以内，Fiber 的可中断能力是"为未来准备的冗余复杂度"。

#### 2.3 preact/compat 的兼容层

`preact/compat` 是一个兼容模块，它为 Preact 添加 React 的缺失功能：

```javascript
// 大致实现思路
// compat 把 React 的 API 映射到 Preact 的实现

// createPortal — compat 中用 DOM API 实现
function createPortal(vnode, container) {
  const dom = render(vnode, container, null, true)
  return { $$typeof: Symbol.for('react.portal'), children: vnode, containerInfo: container }
}

// Children API — 用工具函数补上
const Children = {
  map: (children, fn) => toArray(children).map(fn),
  forEach: (children, fn) => toArray(children).forEach(fn),
  count: (children) => toArray(children).length,
  // ...
}
```

配置 alias 后（Vite 中 `resolve.alias.react = 'preact/compat'`），项目中所有 `import from 'react'` 的代码都指向 `preact/compat`，包括 React Router、Zustand、Ant Design 等三方库。

### 3. 实际应用场景

#### 场景1：直接使用 Preact（体积优先）

```jsx
// 不经过 compat，直接使用 preact
import { render, h } from 'preact'
import { useState } from 'preact/hooks'

function Counter() {
  const [count, setCount] = useState(0)
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>
}

render(<Counter />, document.getElementById('app'))
```

注意：这里的 `h` 函数是 Preact 的 `createElement` 别名。如果配置了 JSX pragma（`/** @jsx h */`），JSX 会被编译成 `h(...)` 调用。

#### 场景2：通过 compat 使用 React 生态

```jsx
// vite.config.js
import { defineConfig } from 'vite'
import preact from '@preact/preset-vite'

export default defineConfig({
  plugins: [preact()],
  // preact() 插件自动配置了 alias：react → preact/compat, react-dom → preact/compat
})

// 然后在代码中正常写 React 语法，引入 React 生态库
import React, { useState } from 'react'
import { Button } from 'antd'
import { BrowserRouter, Route } from 'react-router-dom'

function App() {
  const [count, setCount] = useState(0)
  return <Button onClick={() => setCount(c => c + 1)}>{count}</Button>
}
```

#### 场景3：Preact Signals — 比 hooks 更高效的响应式方案

Preact 团队推出了 `@preact/signals`，一种新的响应式原语，更新粒度精确到 DOM 节点：

```jsx
import { signal, computed } from '@preact/signals'

// signal 创建一个响应式值（类似 Vue 的 ref）
const count = signal(0)
const doubled = computed(() => count.value * 2)

function Counter() {
  return (
    <div>
      {/* 直接在 JSX 中使用 .value，Preact 自动追踪依赖 */}
      <p>{doubled}</p>
      <button onClick={() => count.value++}>+1</button>
    </div>
  )
}
```

Signals 的工作原理与 Vue 的 `ref` + `computed` 相似：当 `count.value` 变化时，只更新包含 `doubled` 的文本节点，整个组件不会重新渲染。这比 React hooks 的"重新执行整个组件函数然后 diff"更高效。

### 4. 常见误区 & 实际项目中的坑

#### 误区1：认为 Preact API 和 React 完全一致

```javascript
// React 可以渲染字符串
ReactDOM.render('hello', root)

// Preact 不能渲染字符串
render('hello', root) // ❌ Error
render(<span>hello</span>, root) // ✅ Preact 只接受 JSX/VNode
```

#### 误区2：以为直接替换 import 就能无缝迁移

```javascript
// ❌ 直接替换 import 可能遇到 API 差异
import { render } from 'react-dom'    // React: render(element, container, callback?)
import { render } from 'preact'       // Preact: render(element, container, replaceNode?)
```

**正确做法**：用 `preact/compat` 作为 alias，而不是手动替换 import。

#### 坑：Preact 不支持 Class Component 的 legacy Context

```javascript
class Button extends React.Component {
  // ❌ legacy contextTypes（prop-types 声明式）
  static contextTypes = { theme: PropTypes.object }
  render() { return <button className={this.context.theme} /> }
}
```

**解法**：改用新 Context API（`createContext` + `useContext`），或通过 compat 运行。

### 5. 与相关知识的关联 & 对比

| 对比维度 | React 18 | Preact | Vue 3 |
|---|---|---|---|
| 体积（gzip） | ~15KB | ~3KB | ~10KB |
| diff 引擎 | Fiber（可中断链表） | 简化递归（不可中断） | Block Tree + PatchFlag |
| 合成事件 | 有 | 无（原生事件） | 无 |
| 并发模式 | 支持 | 不支持 | 不支持 |
| Suspense | 支持 | compat 有限支持 | 内置 |
| Signals | 无 | @preact/signals | ref/reactive（内置） |
| 适用场景 | 大型复杂应用 | 体积敏感场景 | 中大型应用 |

| API | React | Preact | 兼容性 |
|---|---|---|---|
| createElement | React.createElement | h() | ✅ 别名 |
| render | ReactDOM.render(el, container) | render(el, container) | 参数略有不同 |
| hooks | react 包 | preact/hooks | 核心 hooks 全部支持 |
| Suspense | 支持 | 有限支持（compat） | ⚠️ |
| lazy | React.lazy | 不支持 | ⚠️ compat 中有 polyfill |
| createPortal | ReactDOM.createPortal | 不支持 | ⚠️ compat 中有 |

### 6. 现代最佳实践（2024-2025）

1. **新轻量项目直接使用 Preact**：通过 `npm create vite@latest my-app -- --template preact-ts` 脚手架。对首屏体积有要求的项目（移动端 H5、落地页）直接选 Preact。
2. **React 项目迁移到 Preact**：先用 alias 配置 `preact/compat`，跑通测试。注意 `useSyncExternalStore`（React 18 新 API）和 `Suspense` 相关库的兼容性。
3. **Signals 优先于 hooks**：对于状态变化频繁的组件（计数器、实时数据），Signals 的更新粒度更精细，性能更好。
4. **不要在生产环境混用 Preact 和 React**——如果你既有 Preact 又有 React 组件，最终的包体积反而更大。

### 7. 常见疑问解答

**Q：Preact 不用 Fiber，那大型列表渲染会不会比 React 卡？**

A：理论上 Preact 的同步递归 diff 在超大型列表（数万行）场景下可能出现长时间阻塞，因为一旦开始 diff 就不能中断。但实际上 React 的 Fiber 在这种情况下也只是"把阻塞拆分得更细"——总耗时是差不多的。对于绝大多数应用（几百到几千个节点），两者的 diff 耗时都在 5ms 以内，没有可感知的差异。如果你的列表真的到了数万行，应该用虚拟滚动（只渲染可见区域），而不是依赖框架的 diff 优化。

**Q：`@preact/signals` 对比 React 的 `useState` 好在哪？**

A：`useState` 的粒度是"组件级"——更新任何 state 都会导致整个组件函数重新执行。Signals 的粒度是"DOM 节点级"——`{doubled}` 这个文本节点在渲染时只订阅了 `doubled` 这个 signal，`doubled` 变化时只更新这个文本节点，组件函数不重新执行。在需要频繁更新的场景（实时数据、动画帧）中，Signals 优势明显。

**关联知识点索引**
- `React 核心概念.md` — JSX、组件基础（Preact 核心与其兼容）
- `React 渲染机制与性能优化.md` — Fiber vs Preact 简化 diff
- `React vs Vue 对比.md` — 框架设计哲学比较
