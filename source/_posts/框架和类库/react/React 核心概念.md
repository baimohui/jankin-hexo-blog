---
title: React 核心概念 — JSX、组件化、生命周期、Virtual DOM
categories: 
- React
tags:
- React
- JSX
- 组件
- 生命周期
- Virtual DOM
---

## 【面试速答版】

<!-- more -->
### Q1: "JSX 的本质是什么？为什么 React 要引入 JSX？"

JSX 是 JavaScript 的**语法扩展**，本质是 `React.createElement(type, props, ...children)` 的语法糖。Babel 在编译时将 JSX 转换成普通的 JS 函数调用。举个例：`<div className="box">Hello</div>` 编译后变成 `React.createElement('div', { className: 'box' }, 'Hello')`。React 引入 JSX 而不是像 Vue 那样用模板，是因为 JSX 本身就是 JS——你可以在 JSX 中用变量、表达式、循环、条件判断，不需要学一套额外的模板语法（如 v-if/v-for）。对于 JavaScript 开发者来说更自然，也更灵活。

### Q2: "Class Component 和 Function Component 有什么区别？"

Class Component 继承 `React.Component`，有 `this.state` + `this.setState` + 生命周期方法（componentDidMount 等），逻辑复用靠 HOC 或 Render Props。Function Component 只是普通函数，React 16.8 之前不能管理状态和副作用，之后通过 Hooks（useState、useEffect 等）获得了完整能力。现在 React 官方**推荐全部使用 Function Component + Hooks**，因为代码更简洁，Hooks 的逻辑复用（自定义 hooks）比 HOC 更直观且没有嵌套地狱。Class Component 只在维护旧项目时出现。

### Q3: "React 生命周期有哪些？React 16 之后有什么变化？"

React 生命周期分三个阶段：**挂载**（constructor → render → componentDidMount）、**更新**（render → componentDidUpdate）、**卸载**（componentWillUnmount）。React 16 的变化：废弃了 componentWillMount、componentWillReceiveProps、componentWillUpdate（加了 UNSAFE_ 前缀），新增 `getDerivedStateFromProps(props, state)` 替代 componentWillReceiveProps，新增 `getSnapshotBeforeUpdate(prevProps, prevState)` 在 DOM 更新前捕获信息。原因是 React 16 的 Fiber 架构允许渲染被中断，而旧生命周期在中断后可能被多次调用，容易出 bug。新生命周期更安全。

## 【深入理解版】

### 1. 这个知识点要解决什么问题？

先想想没有 React 的时候前端怎么写页面。假设有一个计数器按钮，点击后数字 +1：

```html
<button id="btn">0</button>
<script>
  let count = 0
  const btn = document.getElementById('btn')
  btn.addEventListener('click', () => {
    count++
    btn.textContent = count  // 手动操作 DOM
  })
</script>
```

这种命令式写法在简单场景下很直观，但页面复杂后问题就来了：你需要手动管理 DOM 的每一次创建、更新、删除。如果 10 个地方引用了 `count`，你就得写 10 行 `xxx.textContent = count`。

React 要解决的核心问题就是：**让开发者只描述"UI 应该长什么样"，框架负责"怎么去更新"**。开发者只需要声明 `UI = f(state)`（UI 是状态的函数），React 在状态变化时自动计算出需要更新的最小 DOM 操作，然后执行。这背后的关键是 JSX、Virtual DOM 和 reconciliation。

### 2. 核心原理/执行过程

#### 2.1 JSX 编译链

先看 JSX 代码在 React 中是怎么变成真实 DOM 的：

```jsx
// Step 1: 你写的 JSX
const element = <h1 className="title">Hello, React!</h1>
```

```javascript
// Step 2: Babel 编译后的结果（@babel/preset-react 负责转换）
const element = React.createElement('h1', { className: 'title' }, 'Hello, React!')
```

`React.createElement` 接收三个参数：**type**（标签名或组件）、**props**（属性对象）、**children**（子节点）。它返回一个**普通的 JavaScript 对象**，叫作 ReactElement：

```javascript
// Step 3: createElement 返回的 ReactElement（≈ Virtual DOM 节点）
const element = {
  type: 'h1',
  props: {
    className: 'title',
    children: 'Hello, React!'
  },
  key: null,
  ref: null,
  // ... 其他内部字段
}
```

这个对象就是 **Virtual DOM（虚拟 DOM）** 的最小单元——一个普通的 JS 对象，描述了真实 DOM 应该长什么样。JSX 本质上就是帮你生成这个对象的语法捷径。

```javascript
// 对比：不用 JSX 写 React 组件
React.createElement('div', { className: 'container' },
  React.createElement('h1', null, '标题'),
  React.createElement('p', null, '段落')
)

// 用 JSX
<div className="container">
  <h1>标题</h1>
  <p>段落</p>
</div>
```

**为什么 JSX 比模板（Vue template）更灵活？** 因为 JSX 是 JS——你可以在 `{}` 中写任意 JS 表达式：

```jsx
{condition && <div>条件渲染</div>}
{list.map(item => <div key={item.id}>{item.name}</div>)}
{/* 不需要学 v-if/v-for，直接用 JS 语法 */}
```

而 Vue 的模板是 HTML 扩展，里面有 `v-if`、`v-for`、`v-bind` 等指令，学习成本更高。JSX 的策略是：**你既然已经会 JavaScript，那就直接用 JavaScript**。

#### 2.2 组件的两种定义方式

**Class Component（旧方式）：**

```jsx
class Welcome extends React.Component {
  // constructor 中初始化 state 和绑定事件
  constructor(props) {
    super(props)
    this.state = { name: 'World' }
    this.handleClick = this.handleClick.bind(this)
    // 这里 bind 是因为 class 方法默认不绑定 this
  }

  handleClick() {
    this.setState({ name: 'React' })
    // setState 触发重新渲染
  }

  render() {
    // render 返回 JSX → createElement → Virtual DOM → 真实 DOM
    return <h1 onClick={this.handleClick}>Hello, {this.state.name}</h1>
  }
}
```

**Function Component + Hooks（新方式）：**

```jsx
function Welcome() {
  const [name, setName] = useState('World')
  // useState 返回 [当前值, 更新函数]，不需要 this
  return <h1 onClick={() => setName('React')}>Hello, {name}</h1>
}
```

对比来看，Function Component 不需要 `this`、不需要 `constructor`、不需要 `bind`、不需要 `render` 方法——整个组件就是一个函数，返回 JSX 即可。

**一个常见的困惑：函数组件每次渲染都会重新执行整个函数，那它的"状态"是怎么保持的？**

关键在于 React 内部维护了一个"hooks 链表"。每次组件渲染时，React 按照 hooks 的调用顺序，把上一次存储在链表上的值取出来返回。所以 hooks 不能放在 if/for 中——调用顺序变了就无法匹配到上一次的值。

```jsx
function Counter() {
  // 第一次渲染：React 创建 hook1，存 count=0
  const [count, setCount] = useState(0)
  // 第二次渲染（点击后）：React 找到 hook1，取出最新的 count
  // ...
}
```

#### 2.3 生命周期（Class Component）

Class 组件的生命周期分为三个阶段：

```text
挂载（Mount）:
constructor()          → 初始化 state、绑定事件
  ↓
render()               → 生成 Virtual DOM 树
  ↓
componentDidMount()    → DOM 已渲染，适合发请求、设定时器

更新（Update）:
render()               → 生成新的 Virtual DOM
  ↓
componentDidUpdate()   → DOM 已更新，适合基于新 DOM 做操作

卸载（Unmount）:
componentWillUnmount() → 组件消失前清理（定时器、取消订阅）
```

**React 16 的变化"历史背景"：**

在 React 16（Fiber 架构）之前，渲染是同步不可中断的，所以 `componentWillMount` 等生命周期可以安全使用。Fiber 引入了**可中断渲染**——渲染过程可能被高优先级任务打断，之后再恢复。这意味着一个生命周期可能在一次更新中多次被调用。`componentWillMount`、`componentWillReceiveProps`、`componentWillUpdate` 这三个生命周期在 Fiber 下变得不安全，因为它们可能在渲染过程中被多次触发。

被移除的三个生命周期（加了 `UNSAFE_` 前缀，不推荐使用）：
- `UNSAFE_componentWillMount()` — 用 `constructor` 或 `componentDidMount` 替代
- `UNSAFE_componentWillReceiveProps(nextProps)` — 用 `getDerivedStateFromProps` 替代
- `UNSAFE_componentWillUpdate(nextProps, nextState)` — 用 `getSnapshotBeforeUpdate` 替代

新增的两个：
- `static getDerivedStateFromProps(props, state)` — 根据 props 计算新 state，返回 null 表示 state 不变
- `getSnapshotBeforeUpdate(prevProps, prevState)` — 在 DOM 更新前捕获信息（如滚动位置），返回值传给 `componentDidUpdate` 的第三个参数

#### 2.4 Function Component 的"生命周期"（Hooks）

Function Component 通过 hooks 来模拟生命周期，但设计理念不同——不再是"在某个时间点执行"，而是"数据变化后做什么"：

```jsx
import { useEffect, useLayoutEffect } from 'react'

function LifecycleDemo({ id }) {
  // componentDidMount + componentDidUpdate 的组合：
  // id 变化时重新请求（初次挂载时也会执行一次）
  useEffect(() => {
    fetchData(id)
  }, [id])
  //            ↑ 依赖数组——只有 id 变化时才重新执行

  // componentDidMount 只执行一次（空依赖）：
  useEffect(() => {
    const timer = setInterval(tick, 1000)
    return () => clearInterval(timer)  // ← componentWillUnmount
  }, [])

  // 同步版本——在浏览器绘制前执行（罕见场景）：
  useLayoutEffect(() => {
    // 测量 DOM 尺寸、同步更新布局
  }, [])
}
```

`useEffect` 和 `useLayoutEffect` 的区别在于执行时机：`useEffect` 在浏览器**绘制之后**异步执行，不阻塞用户看到新 UI；`useLayoutEffect` 在浏览器**绘制之前**同步执行。绝大多数场景用 `useEffect`，只有在需要读取 DOM 布局信息（如测量元素尺寸）时用 `useLayoutEffect`。

### 3. 实际应用场景

#### 场景1：条件渲染 + 列表渲染（JSX 的灵活性）

```jsx
function UserList({ users, loading, error }) {
  if (loading) return <Spinner />
  if (error)   return <Error message={error} />

  if (users.length === 0) return <EmptyState />

  // JS 的 map 直接搞定列表渲染
  return (
    <ul>
      {users.map(user => (
        // key 帮助 React 识别哪些元素变了（后面 diff 算法会用到）
        <li key={user.id}>
          {user.name} - {user.role === 'admin' ? '管理员' : '用户'}
        </li>
      ))}
    </ul>
  )
}
```

这里没有 `v-if`、`v-for`，就是用普通的 JS 条件判断和数组 map。`loading`、`error`、空数组各对应一种渲染结果，逻辑清晰。

#### 场景2：错误边界（Error Boundary）

Class Component 独有的能力——捕获子组件抛出的错误，防止整个应用崩溃：

```jsx
class ErrorBoundary extends React.Component {
  constructor(props) {
    super(props)
    this.state = { hasError: false, errorMessage: '' }
  }

  // 当子组件抛出错误时触发，返回新的 state
  static getDerivedStateFromError(error) {
    return { hasError: true, errorMessage: error.message }
  }

  // 错误日志上报
  componentDidCatch(error, errorInfo) {
    console.error('捕获到错误:', error, errorInfo)
    // 可以在这里把错误信息上报给监控系统
  }

  render() {
    if (this.state.hasError) {
      // 显示降级 UI，而不是白屏
      return <div className="error-ui">
        <h2>出错了</h2>
        <p>{this.state.errorMessage}</p>
        <button onClick={() => this.setState({ hasError: false })}>
          重试
        </button>
      </div>
    }
    return this.props.children
  }
}

// 使用：
<ErrorBoundary>
  <UserProfile userId={id} />  {/* 如果 UserProfile 抛出错误，ErrorBoundary 捕获 */}
</ErrorBoundary>
```

注意：错误边界只能捕获**渲染期间**的错误（生命周期、render 方法），不能捕获事件处理中的错误（事件错误用 try/catch）。

### 4. 常见误区 & 实际项目中的坑

#### 误区1：认为 JSX 中的 `{}` 可以直接放对象

```jsx
// ❌ 错误：对象不能直接作为 React 子节点渲染
function Bad() {
  return <div>{{ name: 'test' }}</div>
  // Error: Objects are not valid as a React child
}

// ✅ 正确：用 JSON.stringify 或提取属性
function Good() {
  const obj = { name: 'test' }
  return <div>{JSON.stringify(obj)}</div>   // {"name":"test"}
  // 或 <div>{obj.name}</div>                // test
}
```

React 子节点必须是可渲染的内容：字符串、数字、数组、null/undefined/boolean（不渲染）、ReactElement。对象不能直接渲染。

#### 误区2：class 组件中忘记 bind this

```jsx
class Counter extends React.Component {
  constructor(props) {
    super(props)
    this.state = { count: 0 }
    // 如果这里忘了 bind...
  }

  handleClick() {
    // 这里的 this 在事件触发时为 undefined（class 方法默认不绑定）
    this.setState({ count: this.state.count + 1 })
  }

  render() {
    return <button onClick={this.handleClick}>+1</button>
    // TypeError: Cannot read properties of undefined (reading 'setState')
  }
}
```

**三种解法**：

```javascript
// 解法1：constructor 中 bind（推荐，只 bind 一次）
constructor(props) {
  super(props)
  this.handleClick = this.handleClick.bind(this)
}

// 解法2：箭头函数定义方法（实验性语法，需要 babel）
handleClick = () => { this.setState(...) }

// 解法3：render 中箭头函数（不推荐，每次渲染创建新函数）
<button onClick={() => this.handleClick()}>+1</button>
```

这也是为什么 Function Component 更受欢迎——它没有 `this` 的问题。

### 5. 与相关知识的关联 & 对比

| 对比维度 | Class Component | Function Component + Hooks |
|---|---|---|
| 状态 | this.state + this.setState | useState / useReducer |
| 生命周期 | 专用方法（componentDidMount 等） | useEffect（按功能聚合） |
| 逻辑复用 | HOC / Render Props（嵌套地狱） | 自定义 hooks（无嵌套） |
| this | 需要关心绑定问题 | 没有 this |
| 代码量 | 多（constructor + bind 等模板代码） | 少 |
| 错误边界 | 支持（componentDidCatch） | 不支持（需用 Class 包裹） |
| 官方推荐 | 旧项目维护 | 新项目首选 |

| 概念 | 本质 | 类比 |
|---|---|---|
| JSX | `createElement` 的语法糖 | 编译时转换，不是运行时模板 |
| ReactElement | `createElement` 返回的 JS 对象 | Virtual DOM 的最小单元 |
| Virtual DOM | 用 JS 对象模拟的 DOM 树结构 | diff 的"原材料" |
| Fiber | React 16 的可中断调度引擎 | 能暂停/恢复的渲染流水线 |

### 6. 现代最佳实践（2024-2025）

1. **新项目都用 Function Component + Hooks**，TypeScript + .tsx。
2. **用 React 18 的 `createRoot` 代替 `ReactDOM.render`**：

```jsx
import { createRoot } from 'react-dom/client'
const root = createRoot(document.getElementById('root'))
root.render(<App />)
```

3. **按路由/组件做代码分割**——`React.lazy` + `Suspense`：

```jsx
const Dashboard = React.lazy(() => import('./Dashboard'))
<Suspense fallback={<Loading />}>
  <Dashboard />
</Suspense>
```

4. **Error Boundary 包裹每个路由**，防止一个页面的崩溃影响整个应用。
5. **不要再使用 UNSAFE_ 开头的生命周期**，它们会在未来的 React 版本中彻底移除。

### 7. 常见疑问解答

**Q：JSX 中 `class` 为什么要写成 `className`？**

A：因为 JSX 最终被转译为 JavaScript，而 `class` 是 JavaScript 的**保留关键字**（用于定义类：`class Foo {}`）。如果 JSX 中用 `class`，会导致解析冲突。所以 React 选择了 `className`、`htmlFor`（替代 `for`）、`tabIndex`（替代 `tabindex`）等 DOM 属性的 JS 风格写法。在 React 17/18 的 JSX 新转换中，这个规则依然不变。

**Q：函数组件每次渲染都重新执行，那 useState 是怎么保持上一次的值的？**

A：React 内部在每个组件对应的 Fiber 节点上维护了一个 **hooks 链表**。第一次调用 `useState` 时，React 在链表上创建一个新的 hook 节点，存入初始值。以后每次渲染，React 按照**调用顺序**从链表上依次取出对应的 hook 节点，读取上次存储的值。所以 hooks 的调用顺序必须稳定——不能放在 if/for 中。如果第一次渲染时 useState 是第 1 个 hook，第二次渲染时因为某种原因变成了第 2 个，React 就会读到错误的值。

**Q：Error Boundary 为什么不能用 Function Component？**

A：因为 Error Boundary 依赖 `componentDidCatch` 和 `getDerivedStateFromError` 这两个方法，而这两个方法是 Class Component 的专属能力，Function Component 的 hooks 体系中没有对应的替代品。React 官方表示未来可能会通过 Suspense 的某些机制来提供类似能力，但目前仍需用 Class Component 实现 Error Boundary。已有的解决方案是封装一个通用的 `ErrorBoundary` 类组件（像上面场景2那样），然后在函数组件中直接用它包裹即可。

**关联知识点索引**
- `React Hooks 详解.md` — Function Component 中 useState/useEffect 等 hooks 的深入使用
- `React 渲染机制与性能优化.md` — Fiber、reconciliation、diff 算法的完整原理
- `React vs Vue 对比.md` — 与 Vue 模板/组件的设计对比
