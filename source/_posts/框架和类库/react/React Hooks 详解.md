---
title: React Hooks 详解 — useState、useEffect、自定义 hooks
categories: 
- React
tags:
- React
- Hooks
- useState
- useEffect
- useCallback
- useMemo
---

## 【面试速答版】

<!-- more -->
### Q1: "useState 和 useReducer 的适用场景分别是什么？setState 是同步还是异步？"

`useState` 适用于**独立、简单的状态**（如数字、字符串、布尔值），`useReducer` 适用于**多个关联的状态或复杂更新逻辑**（如表单字段校验、购物车多步骤操作）。`useState` 的 setter 在 React 18 的 `createRoot` 下**默认都是异步的**（自动批处理），无论写在事件处理、setTimeout 还是 Promise 中，多个 setState 会被合并到一次渲染。`useReducer` 的 dispatch 也是同样的批处理规则。需要强制同步读取最新值时，用 `useEffect` 或在事件处理中通过 `e.target` 等原生方式获取。

### Q2: "useEffect 的依赖数组是怎么工作的？空数组和不传有什么区别？"

`useEffect(fn, deps)` 在**渲染完成后**执行 `fn`。依赖数组控制执行时机：**不传 deps** → 每次渲染都执行；**传 `[]`** → 只在组件挂载时执行一次（fn 中如果有清理函数，组件卸载时执行）；**传 `[a, b]`** → a 或 b 变化时才执行（初次挂载也执行）。注意 useEffect 不是在"监听"变化，而是在"每次渲染后检查依赖是否变了"——初次渲染时"依赖从无到有"也算变化，所以一定执行一次。

### Q3: "useMemo 和 useCallback 的区别是什么？什么时候必须使用？"

`useMemo` 缓存计算**结果**，`useCallback` 缓存**函数引用**。两者都为了配合 `React.memo` 避免子组件不必要的重渲染。**必须使用的场景**：当子组件被 `memo` 包裹，且父组件传入了对象/函数/数组给子组件时——因为每次父组件重渲染都会创建新引用，导致 memo 失效。其他场景**不需要滥用**，简单的加法 `useMemo(() => a + b, [a, b])` 反而因比较依赖数组的开销而不划算。优先写好不加 memo 的代码，遇到性能问题时再用 Profiler 定位优化点。

## 【深入理解版】

### 1. 这个知识点要解决什么问题？

在 React 16.8（2019 年）之前，函数组件是"无状态组件"——只能接收 props 返回 JSX，不能管理自己的状态，也不能执行副作用（发请求、订阅、操作 DOM）。如果你需要这些能力，就必须写成 Class Component，然后面对 `this` 绑定问题、生命周期方法逻辑分散的问题。

看一个 Class Component 的典型例子——鼠标位置追踪器：

```jsx
class MouseTracker extends React.Component {
  constructor(props) {
    super(props)
    this.state = { x: 0, y: 0 }
    this.handleMouseMove = this.handleMouseMove.bind(this)
  }

  // 相关的逻辑（绑定事件）在 componentDidMount
  componentDidMount() {
    window.addEventListener('mousemove', this.handleMouseMove)
  }

  handleMouseMove(e) {
    this.setState({ x: e.clientX, y: e.clientY })
  }

  // 相关的逻辑（解绑事件）在 componentWillUnmount
  componentWillUnmount() {
    window.removeEventListener('mousemove', this.handleMouseMove)
  }

  render() {
    return <div>鼠标位置: {this.state.x}, {this.state.y}</div>
  }
}
```

这里"监听鼠标移动"这个逻辑被拆到了三个方法中：`constructor`（初始化）、`componentDidMount`（绑定）、`componentWillUnmount`（解绑）。如果还有多个这样的逻辑（窗口 resize、键盘事件），它们都会交叉分散在这些生命周期方法中——这就是所谓的"逻辑分散"问题。

Hooks 的核心理念是：**按功能组织逻辑，而不是按生命周期组织**。用 hooks 重写上面的例子：

```jsx
function MouseTracker() {
  const [pos, setPos] = useState({ x: 0, y: 0 })

  // 绑定/解绑的逻辑放在一起，功能内聚
  useEffect(() => {
    const handler = (e) => setPos({ x: e.clientX, y: e.clientY })
    window.addEventListener('mousemove', handler)
    return () => window.removeEventListener('mousemove', handler)
  }, [])  // 空数组 = 只在挂载时绑定、卸载时解绑

  return <div>鼠标位置: {pos.x}, {pos.y}</div>
}
```

这还不是最大的优势。Hooks 真正的威力是**自定义 hooks**——把上面的逻辑提取成一个可复用的函数：

```javascript
function useMousePosition() {
  const [pos, setPos] = useState({ x: 0, y: 0 })

  useEffect(() => {
    const handler = (e) => setPos({ x: e.clientX, y: e.clientY })
    window.addEventListener('mousemove', handler)
    return () => window.removeEventListener('mousemove', handler)
  }, [])

  return pos
}

// 在任何组件中使用：
function MouseTracker() {
  const pos = useMousePosition()
  return <div>位置: {pos.x}, {pos.y}</div>
}

function GameCanvas() {
  const pos = useMousePosition()
  // 用鼠标位置控制画布上的元素
}
```

这就是 hooks 解决的核心问题：**逻辑复用**。Class Component 时代，复用逻辑要靠 HOC（高阶组件）或 Render Props，两者都会导致组件树嵌套层层加深，调试困难。自定义 hooks 只是一个函数，没有嵌套、没有额外组件。

### 2. 核心原理/执行过程

#### 2.1 useState — 状态管理的起点

`useState` 是最基础的 hook。先看它的用法，然后理解它背后是如何"记住"状态的：

```jsx
function Counter() {
  // useState(initialValue) 返回 [当前值, 更新函数]
  const [count, setCount] = useState(0)

  const increment = () => {
    // setCount 可以传新值
    setCount(count + 1)
    // 也可以传函数（函数式更新），接收上一次的值
    // setCount(prev => prev + 1)
  }

  return (
    <div>
      <p>{count}</p>
      <button onClick={increment}>+1</button>
    </div>
  )
}
```

**useState 是如何跨渲染周期"记住"状态的？** 每次组件渲染，React 内部做了三件事：

1. 在 Fiber 节点上找到这个组件对应的 **hooks 链表**
2. 按 hooks 的**调用顺序**从链表上取出对应的 hook 节点
3. 读取 hook 节点上存储的 `memoizedState` 作为当前值返回

```javascript
// 简化版：React 内部维护一个"当前正在处理的 hook 指针"
let workInProgressHook = null

function useState(initialValue) {
  // 获取当前 hook 节点（第一次创建，后续复用）
  const hook = workInProgressHook?.next || {
    memoizedState: typeof initialValue === 'function' ? initialValue() : initialValue,
    queue: null,
    next: null,
  }

  workInProgressHook = hook

  const dispatch = (action) => {
    const newState = typeof action === 'function' ? action(hook.memoizedState) : action
    hook.memoizedState = newState
    scheduleRender()  // 调度重新渲染
  }

  return [hook.memoizedState, dispatch]
}
```

这就是为什么 hooks **不能放在条件或循环中**——React 依赖调用顺序来匹配 hook 和它的状态：

```jsx
function BadComponent({ condition }) {
  // 第一次渲染：condition = true → 第1个useState是count
  if (condition) {
    const [count] = useState(0)      // hook #1
  }
  const [name] = useState('')       // hook #2

  // 第二次渲染：condition = false → 第1个useState变成了name！
  // 顺序错乱了 → React 读到错误的值
}
```

#### 2.2 useEffect — 副作用管理器

`useEffect` 用来处理"副作用"——那些不属于渲染逻辑的操作：发请求、订阅事件、操作 DOM、设置定时器。

```jsx
useEffect(
  () => {
    // 副作用逻辑（渲染后执行）
    document.title = `你点了 ${count} 次`

    return () => {
      // 清理函数（下一次副作用执行前或卸载时执行）
      // 用于取消订阅、清除定时器、移除事件监听
    }
  },
  [count]  // 依赖数组
)
```

**useEffect 的执行时点：**

```text
组件渲染（生成 Virtual DOM）
  ↓
React 更新真实 DOM
  ↓
浏览器绘制（用户看到新 UI）
  ↓  ← 这里才是 useEffect 的执行时机
执行 useEffect 中的函数
```

**为什么在浏览器绘制之后才执行？** 因为 React 不希望副作用阻塞用户看到新 UI。如果你需要同步（在浏览器绘制之前）执行副作用，用 `useLayoutEffect`。

**依赖数组的三种传法对比：**

```javascript
// 1. 不传依赖：每次渲染都执行
useEffect(() => {
  console.log('每次渲染后都执行')
})  // 没有第二个参数

// 2. 空数组：只在挂载时执行一次，卸载时执行清理
useEffect(() => {
  const timer = setInterval(() => tick(), 1000)
  return () => clearInterval(timer)  // 组件卸载时清理
}, [])  // [] 表示不需要依赖任何值

// 3. 有依赖：依赖变化时执行（初次挂载也会执行一次）
useEffect(() => {
  fetch(`/api/users/${userId}`)  // userId 变化时重新请求
}, [userId])
```

**一个常见的误解**：认为 `useEffect` 像 Vue 的 `watch` 一样在"监听"变化。不对——`useEffect` 的逻辑其实是："每次渲染之后，检查依赖列表中的值有没有变化，如果有就执行函数"。所以**初次挂载时一定会执行一次**，因为"从无到有"本身就是变化。

#### 2.3 useRef — 可变引用

`useRef` 创建一个**跨渲染周期保持不变**的可变对象。它的 `.current` 属性可以存任意值，改变它不会触发重渲染：

```javascript
function Timer() {
  const [count, setCount] = useState(0)

  // useRef 返回 { current: initialValue }，这个对象在组件整个生命周期内不变化
  const timerRef = useRef(null)

  useEffect(() => {
    // 存定时器 ID，方便卸载时清理
    timerRef.current = setInterval(() => {
      setCount(c => c + 1)
    }, 1000)

    return () => clearInterval(timerRef.current)
  }, [])

  return <div>{count}</div>
}
```

`useRef` 常见的两个用途：
1. **访问 DOM 元素**：`<div ref={myRef}>` → `myRef.current` 拿到真实 DOM
2. **保存可变值**：存定时器 ID、事件监听器引用、任何需要跨渲染周期"记住"但不触发渲染的值

#### 2.4 useMemo 和 useCallback — 性能优化

这两个 hook 的作用相同——**缓存引用**——只是缓存的东西不同：

```javascript
// useMemo：缓存计算结果
const sortedList = useMemo(
  () => bigList.sort((a, b) => a.score - b.score),  // 只有 bigList 变化时才重新排序
  [bigList]
)

// useCallback：缓存函数引用（等价于 useMemo(() => fn, deps)）
const handleClick = useCallback(
  () => setCount(c => c + 1),  // 只要 [] 不变，handleClick 引用就不变
  []
)
```

**它们什么时候必须用？** 当一个**被 React.memo 包裹的子组件**接收了父组件的对象/函数/数组时：

```jsx
const Child = React.memo(function Child({ onClick }) {
  console.log('Child render')  // 只有 props 变化时才执行
  return <button onClick={onClick}>点击</button>
})

function Parent() {
  const [count, setCount] = useState(0)

  // ❌ 每次 Parent 渲染都创建新函数 → Child 的 memo 失效
  return <Child onClick={() => setCount(c => c + 1)} />

  // ✅ useCallback 保证引用不变 → Child 不会因函数引用变化而重渲染
  const handleClick = useCallback(() => setCount(c => c + 1), [])
  return <Child onClick={handleClick} />
}
```

**不要滥用**：简单的数值计算（`a + b`）用 `useMemo` 反而增加了比较依赖数组的开销。React 官方建议先不加优化，遇到性能瓶颈时用 DevTools Profiler 定位后再加上。

### 3. 实际应用场景

#### 场景1：表单输入（useState + 受控组件）

```jsx
function LoginForm() {
  const [form, setForm] = useState({ username: '', password: '' })

  // 用一个函数处理所有字段更新
  const handleChange = (field) => (e) => {
    setForm(prev => ({ ...prev, [field]: e.target.value }))
  }

  const handleSubmit = (e) => {
    e.preventDefault()
    console.log('提交:', form)
  }

  return (
    <form onSubmit={handleSubmit}>
      <input value={form.username} onChange={handleChange('username')} placeholder="用户名" />
      <input value={form.password} onChange={handleChange('password')} type="password" placeholder="密码" />
      <button type="submit">登录</button>
    </form>
  )
}
```

这里 `setForm(prev => ({ ...prev, [field]: value }))` 是函数式更新——用 `prev` 保证你拿到的永远是最新的 state，而不是闭包中捕获的旧值。

#### 场景2：防抖搜索（自定义 hooks）

防抖是典型场景：用户输入时，等 300ms 没有再输入才执行搜索：

```javascript
// 自定义 hook：useDebounce
function useDebounce(value, delay = 300) {
  const [debounced, setDebounced] = useState(value)

  useEffect(() => {
    // 每次 value 变化时设定定时器
    const timer = setTimeout(() => setDebounced(value), delay)

    // 如果 value 再次变化（新输入），清理上一个定时器 → 重新等待 delay
    return () => clearTimeout(timer)
  }, [value, delay])

  return debounced
}
```

```jsx
function SearchPage() {
  const [query, setQuery] = useState('')
  const debouncedQuery = useDebounce(query, 500)  // 500ms 防抖

  // 只有 debouncedQuery 变化时才请求（用户实际停止输入后 500ms）
  useEffect(() => {
    if (debouncedQuery) {
      fetch(`/api/search?q=${debouncedQuery}`)
        .then(r => r.json())
        .then(setResults)
    }
  }, [debouncedQuery])

  return (
    <div>
      <input value={query} onChange={e => setQuery(e.target.value)} />
      <SearchResults results={results} />
    </div>
  )
}
```

这个自定义 hook 封装了防抖的完整逻辑，任何组件需要防抖时直接引用即可。

#### 场景3：useReducer + Context 代替小型 Redux

当组件树中多个深层组件需要共享状态时，可以用 `useReducer` + `Context` 组合成一个轻量级状态管理：

```javascript
// store/CounterContext.js
import { createContext, useContext, useReducer } from 'react'

// 1. 定义 reducer
function counterReducer(state, action) {
  switch (action.type) {
    case 'INCREMENT': return { count: state.count + 1 }
    case 'DECREMENT': return { count: state.count - 1 }
    case 'RESET':     return { count: 0 }
    default: return state
  }
}

// 2. 创建 Context
const CounterContext = createContext(null)

// 3. Provider 组件
export function CounterProvider({ children }) {
  const [state, dispatch] = useReducer(counterReducer, { count: 0 })
  return (
    <CounterContext.Provider value={{ state, dispatch }}>
      {children}
    </CounterContext.Provider>
  )
}

// 4. 自定义 hook 简化使用
export function useCounter() {
  const ctx = useContext(CounterContext)
  if (!ctx) throw new Error('useCounter must be used inside CounterProvider')
  return ctx
}
```

```jsx
// 任意深度的子组件
function DeepChild() {
  const { state, dispatch } = useCounter()

  return (
    <div>
      <p>{state.count}</p>
      <button onClick={() => dispatch({ type: 'INCREMENT' })}>+</button>
    </div>
  )
}
```

`useReducer` 在这里代替了 `useState`——因为更新逻辑（INCREMENT/DECREMENT/RESET）有固定的模式，用 reducer 更好组织和阅读。这个模式其实就是 Zustand 和 Redux 的简化雏形。

### 4. 常见误区 & 实际项目中的坑

#### 误区1：useEffect 里直接用 async 函数

```javascript
// ❌ 错误：useEffect 不能直接接收 async 函数
useEffect(async () => {
  const data = await fetchData()
  setData(data)
}, [])
// useEffect 期望第一个参数返回一个函数（清理函数）或 undefined
// async 函数总是返回 Promise，不符合预期
```

```javascript
// ✅ 正确：内部定义 async 并调用
useEffect(() => {
  let cancelled = false  // 用于防止组件卸载后更新已卸载组件的状态

  async function load() {
    const data = await fetchData()
    if (!cancelled) {  // 组件还在，安全更新
      setData(data)
    }
  }

  load()

  return () => { cancelled = true }  // 清理：标记组件已卸载
}, [])
```

#### 误区2：闭包陷阱（Stale Closure）

```javascript
function DelayedLogger() {
  const [count, setCount] = useState(0)

  const log = () => {
    setTimeout(() => {
      console.log(count)  // 输出的是点击按钮时的 count，不是 3 秒后的最新值
    }, 3000)
  }

  return (
    <>
      <p>{count}</p>
      <button onClick={() => setCount(count + 1)}>+1</button>
      <button onClick={log}>3秒后打印</button>
    </>
  )
}
```

**为什么会这样？** 每次组件渲染，`log` 都是一个**新函数**。这个新函数的闭包捕获了**本次渲染的 count**。3 秒后定时器执行时，使用的是被捕获的那个 count，而不是最新的 count。

**两种解法**：

```javascript
// 解法1：函数式更新（适用于 setState 场景）
const increment = () => setCount(c => c + 1)  // c 永远是最新的

// 解法2：useRef 保存最新值（适用于需要读取最新值但不触发渲染的场景）
const countRef = useRef(count)
useEffect(() => { countRef.current = count }, [count])
// 3 秒后：console.log(countRef.current)  // 输出最新值
```

#### 误区3：useMemo 过度优化

```javascript
// ❌ 滥用：简单计算不值得 useMemo
const total = useMemo(() => a + b, [a, b])
// useMemo 本身有内存开销（存储依赖数组 + 比较）

// ✅ 仅在以下场景使用：
// 1. 计算开销大（大数据排序、复杂运算）
const sorted = useMemo(() => bigList.sort(...), [bigList])
// 2. 作为 React.memo 子组件的 props
const items = useMemo(() => [1, 2, 3], [])
```

### 5. 与相关知识的关联 & 对比

| Hook | 作用 | Class 对应 | 使用场景 |
|---|---|---|---|
| useState | 声明响应式状态 | this.state + setState | 所有局部状态 |
| useEffect | 处理副作用 | componentDidMount/Update/WillUnmount | 请求、订阅、DOM 操作 |
| useLayoutEffect | 同步副作用 | getSnapshotBeforeUpdate | 测量 DOM、更新布局 |
| useMemo | 缓存计算结果 | shouldComponentUpdate（部分） | 开销大的计算 |
| useCallback | 缓存函数引用 | 无直接对应 | 配合 React.memo |
| useRef | 保存可变值/DOM 引用 | React.createRef / 实例属性 | DOM 引用、定时器 ID |
| useContext | 消费 Context | Context.Consumer | 主题、语言 |
| useReducer | 复杂状态逻辑 | 无 | 多关联状态 |
| useImperativeHandle | 暴露子组件方法 | 无 | 配合 forwardRef |

| 对比 | Class Component | Function Component + Hooks |
|---|---|---|
| 代码量 | 多 | 少 30%-50% |
| 逻辑组织 | 按生命周期分散 | 按功能聚合 |
| 逻辑复用 | HOC / Render Props | 自定义 hooks |
| this | 需要 bind | 不需要 |
| 调用顺序限制 | 无 | hooks 不能放在条件/循环中 |

### 6. 现代最佳实践（2024-2025）

1. **ESLint 插件 `eslint-plugin-react-hooks` 必须开启**——它能在编译时检查 hooks 的调用顺序和依赖完整性，是 hooks 开发的"安全带"。
2. **自定义 hooks 是逻辑复用的唯一方式**——不再使用 HOC 或 Render Props 复用逻辑。任何可复用的状态逻辑都应封装为 `useXxx` 函数。
3. **状态提升优先于 Context**——props 下钻 2-3 层是可以接受的，不要过早使用 Context。
4. **React 18 的 useDeferredValue 和 useTransition** 处理非紧急更新：

```javascript
const [query, setQuery] = useState('')
const deferredQuery = useDeferredValue(query)  // 低优先级版本
// 输入框实时更新，结果列表延迟渲染 → 保持输入流畅
```

5. **不要把所有请求放在 useEffect 中**——考虑使用 TanStack Query 等数据请求库，它们比手写 useEffect + fetch 更健壮（缓存、重新请求、加载状态管理）。

### 7. 常见疑问解答

**Q：为什么 hooks 不能放在条件或循环中？**

A：因为 React 依赖**调用顺序**来匹配 hook 和它的状态。每次组件渲染时，React 按顺序遍历 hooks 链表，从链表的第 1 个节点取第 1 个 hook 的值、第 2 个节点取第 2 个 hook 的值……如果某次渲染时 `if (condition)` 导致第 2 个 hook 没有被调用，那 React 就会把原来属于第 3 个 hook 的值错认为是第 2 个的，导致状态错乱。

**Q：useEffect 的清理函数在开发环境下为什么执行了两次？**

A：这是 React 18 的严格模式（`<StrictMode>`）下的故意行为。React 在开发环境中会**卸载并重新挂载**每个组件，以此来暴露哪些 useEffect 的清理逻辑写得不对（比如没有正确清理定时器、没有取消订阅）。如果你的清理函数正确，双重执行不会导致问题。生产环境中不会出现。

**Q：class 组件的 componentDidMount 和 useEffect([], []) 的执行时机有什么不同？**

A：`componentDidMount` 在**真实 DOM 渲染完成后、浏览器绘制之前**同步执行。`useEffect([], [])` 在**真实 DOM 渲染完成后、浏览器绘制之后**异步执行。这意味着如果你在 `useEffect` 中读取 DOM 尺寸，得到的是"用户看到之后"的值——但无所谓，因为 React 会在下一次渲染中使用新值。如果你必须在浏览器绘制之前读取 DOM，用 `useLayoutEffect`。

**关联知识点索引**
- `React 核心概念.md` — Class Component 生命周期与 Function Component 的对应
- `React 渲染机制与性能优化.md` — useMemo/useCallback 的渲染优化原理
- `React 组件通信与状态管理.md` — useContext + useReducer 模式的延伸
