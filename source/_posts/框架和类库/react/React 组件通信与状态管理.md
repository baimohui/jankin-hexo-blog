---
title: React 组件通信与状态管理 — props、Context、Redux/Zustand
categories: 
- React
tags:
- React
- 组件通信
- Context
- Redux
- Zustand
---

## 【面试速答版】

<!-- more -->
### Q1: "React 组件之间如何通信？"

分三种关系。**父子通信**：父传子用 props，子传父用回调——父组件传一个函数给子组件，子组件调用它来通知父组件。如果需要跨多层传递，可以用**组合（children）**避免逐层传。**跨层级通信**：用 Context——`createContext` + `Provider` + `useContext`，跳过中间层直接把数据注入到底层组件，适合主题、语言等低频全局数据。**任意组件通信**：用全局状态管理库——中小项目推荐 Zustand（极简、按需订阅），大型项目推荐 Redux Toolkit（强约束、DevTools 完善）。总结：父子用 props + 回调，跨层级用 Context，全局用 Zustand/Redux。

### Q2: "Redux 的核心概念是什么？单向数据流是如何工作的？"

Redux 三个核心概念：**Store**（存全局状态的唯一对象）、**Action**（描述"发生了什么"的普通对象，如 `{ type: 'INCREMENT', payload: 1 }`）、**Reducer**（纯函数，接收旧 state 和 action，返回新 state）。数据流是单向的：组件调用 `dispatch(action)` → Redux 内部调用 reducer 计算新 state → store 更新 → 通知所有订阅的组件重渲染。单向的好处是数据变化路径清晰，从头到尾追踪一遍就能定位 bug。现代 Redux 用 Redux Toolkit 省去手写 switch-case 和 action type 常量，通过 `createSlice` 定义 reducer，`createAsyncThunk` 处理异步。

### Q3: "Zustand 和 Redux 有什么异同？"

相同点：都是全局状态管理，都按需订阅避免无关重渲染。不同点：Zustand 零样板代码——一个 `create` 搞定，不需要 Provider 包裹，返回的就是 hook；Redux 需要 `createSlice + configureStore + Provider`，概念更多。性能上两者都可以（都支持按需订阅），但 Zustand 按 selector 订阅，Redux 按 useSelector 订阅，原理相同。选型建议：中小项目、追求开发效率用 Zustand；大项目、多人协作、需要强约束和 DevTools 用 Redux Toolkit。

## 【深入理解版】

### 1. 这个知识点要解决什么问题？

假设你在开发一个电商应用：首页展示商品列表，用户点击商品进入详情页加入购物车，购物车页面展示已选商品。在这个过程中，"购物车数据"需要被多个页面共享——首页不知道购物车里有什么，详情页加了商品后购物车页要能看到。

在没有专门的状态管理之前，React 开发者遇到三个痛点：

**① Props Drilling（逐层传递）**：如果组件嵌套很深（A → B → C → D），D 需要用到 A 的数据，那 B 和 C 即使不关心这个数据也必须帮忙传一遍。代码变成了"中间搬运工"。

**② 兄弟组件通信困难**：同级组件 A 改了数据，B 要知道，只能通过共同的父组件中转。父组件变得臃肿。

**③ 数据不一致**：每个页面各自维护购物车数据副本，用户可能在三个页面看到三个不同的数量。

状态管理的本质，就是**把"需要被多个组件共享的数据"从组件树中抽离出来**，放到一个统一的地方管理。任何组件都能读、都能写，且数据变化后所有相关组件自动更新。这个"统一的地方"就是 Redux 的 store 或 Zustand 的 store。

### 2. 核心原理/执行过程

#### 2.1 props + 回调：最原始但最可靠的通信

先从最简单的父子通信开始。假设有一个按钮组件，点击后让父组件知道：

```jsx
function Child({ onAction }) {
  return <button onClick={() => onAction('按钮被点了')}>点击</button>
}

function Parent() {
  const [message, setMessage] = useState('')
  const handleMessage = (msg) => setMessage(msg)
  return (
    <div>
      <p>{message}</p>
      <Child onAction={handleMessage} />
    </div>
  )
}
```

这里的核心设计是 React 的**单向数据流**：数据只能从父组件通过 props 往下流，子组件不能直接修改父组件的状态，而是调用父组件传进来的函数来"请求父组件修改"。这样做的好处是数据变化的路径非常清晰——你永远知道数据是从哪来的、经过哪里、最后去了哪。

**那跨多层传递怎么办？** 考虑这个结构：`A → B → C → D`，D 需要用到 A 的数据。如果不做优化：

```jsx
function A() {
  const [user] = useState({ name: '张三' })
  return <B user={user} />
}
function B({ user }) { return <C user={user} /> } // B 不关心 user，被迫传
function C({ user }) { return <D user={user} /> } // C 也不关心
function D({ user }) { return <div>{user.name}</div> }
```

这就是 **Props Drilling（逐层传递）**。中间的 B 和 C 完全不关心 `user`，但必须声明并传递，否则 D 就拿不到。解决方式有两种：**组合** 和 **Context**。

#### 2.2 组合（Component Composition）— 绕过 Props Drilling

组合的思想是：**既然 B、C 不关心 user，那 user 根本不应该经过它们**。直接在 A 中使用 D：

```jsx
function A() {
  const [user] = useState({ name: '张三' })
  return <B><D user={user} /></B>
  //              ↑ user 直接给了 D，不经过 B、C
}
function B({ children }) {
  return <div className="b-wrapper">{children}</div>
  //                        ↑ B 只负责布局，children 自动渲染
}
```

即使 B 内部嵌套多层，`children` 都能直接透传。这种方式比 Context 更轻量，适合"父组件掌控子组件内容布局"的场景。但如果 D 的嵌套深度不确定或者 D 需要在 B 内部的任意位置出现，组合就不够灵活了，此时需要 Context。

#### 2.3 Context：跨层级注入

Context 允许你在组件树中"打一个洞"，跳过中间层直接把数据注入到底层组件：

```jsx
// 1. 创建一个 Context 对象
const UserContext = React.createContext(null)

function A() {
  const [user] = useState({ name: '张三' })
  return (
    // 2. 用 Provider 包裹，value 就是要传递的数据
    <UserContext.Provider value={user}>
      <B />  {/* B 和它的所有子孙都能拿到 user */}
    </UserContext.Provider>
  )
}

function D() {
  // 3. 底层组件用 useContext 读取
  const user = useContext(UserContext)
  return <div>{user.name}</div>
}
```

`React.createContext(defaultValue)` 创建一个 Context 容器。`<Provider value={...}>` 包裹子组件树，value 就是穿透传递的数据。`useContext(MyContext)` 在函数组件中读取最近一个 Provider 的 value。如果向上找不到 Provider，就使用 `createContext(defaultValue)` 中的默认值。

**Context 的原理**：React 内部在 Fiber 节点上记录了 `_context` 属性。当组件调用 `useContext(MyContext)` 时，React 从当前 Fiber 节点向上遍历组件树，找到最近的 `MyContext.Provider` 对应的 Fiber 节点，取出它的 value 返回。这个查找过程是 O(n) 的（n 为 Fiber 深度），但通常深度有限，性能可以接受。

**但 Context 有一个严重的性能问题**：Provider 的 value 变化时，**所有**消费该 Context 的组件都会强制重新渲染，即使父组件包裹了 `React.memo` 也无法阻止。

```jsx
const ThemeContext = createContext('light')

function App() {
  const [theme] = useState('light')
  const [count, setCount] = useState(0)

  return (
    <ThemeContext.Provider value={theme}>
      <button onClick={() => setCount(c => c + 1)}>点了{count}次</button>
      <ExpensiveComponent />
    </ThemeContext.Provider>
  )
}

const ExpensiveComponent = React.memo(() => {
  // 这里没有使用 ThemeContext
  // 但点击按钮时 ExpensiveComponent 依然会重渲染！
  return <div>我很重</div>
})
```

**原因**：`count` 变化 → `App` 重渲染 → 重新执行 `ThemeContext.Provider` 的渲染 → `value={theme}` 虽然 `theme` 没变，但 React 每次都创建一个新的 props 对象引用 → 所有消费该 Context 的组件（包括子组件树中任何使用了 `useContext(ThemeContext)` 的组件）都会重渲染。

**解法**：把 Provider 单独抽出来并用 `useMemo` 缓存 value，或对于高频变化的数据直接使用 Zustand 而非 Context。

#### 2.4 从零理解 Zustand：一个发布订阅系统

当我们说"状态管理库"时，本质上就是一个**发布订阅模式**——数据发生变化时通知所有关心这个数据的"订阅者"。Zustand 就是这样一个极简的实现。我们先看一个从零手写的 Zustand 原型，帮你理解它的本质：

```javascript
// 极简版 Zustand
function createStore(createState) {
  let state                    // 存储实际数据
  const listeners = new Set()  // 存储所有"订阅者"（即组件）

  const getState = () => state                         // 读取当前数据
  const setState = (partial) => {                       // 修改数据
    state = typeof partial === 'function'
      ? partial(state)          // 支持函数式更新：set(s => ({...s, count: s.count+1}))
      : { ...state, ...partial} // 支持对象合并更新
    listeners.forEach(listener => listener())  // 通知所有订阅者
  }
  const subscribe = (listener) => {
    listeners.add(listener)
    return () => listeners.delete(listener)  // 返回取消订阅函数
  }

  state = createState(setState, getState)
  return { getState, setState, subscribe }
}
```

这里 `getState` 是读取接口，`setState` 是修改接口（修改后自动通知），`subscribe` 是订阅接口。当你在 React 组件中调用 `const count = useStore(s => s.count)` 时，Zustand 内部就是这样做的：`subscribe(组件)` 把组件注册为订阅者，`setState` 时触发组件重新渲染。

Zustand 在 React 中的具体连接依赖于 React 18 提供的 `useSyncExternalStore` hook——这个 hook 专门用来把"外部的状态仓库"和"React 的渲染周期"连接起来。当外部 store 变化时，它通知 React 重新渲染组件。Zustand 在这个基础上还做了**按 selector 订阅**的优化：组件只关心 `s.count`，那只有 `count` 变化时组件才重渲染，`store` 中其他字段变化不影响它。

#### 2.5 Redux 的单向数据流

Redux 和 Zustand 一样是发布订阅，但多了几个强制约束，让数据流更规范。完整的 Redux 数据流是这样的：

```text
组件调用 dispatch(action)
  ↓
Action 是一个普通对象：{ type: 'INCREMENT', payload: 1 }
  ↓
Redux 内部调用 Reducer：reducer(当前state, action) → 新state
  ↓
新 state 存入 Store
  ↓
Store 通知所有通过 subscribe 注册的监听者
  ↓
React-Redux 的 useSelector 或 connect 收到通知 → 组件重渲染
```

**Redux 为什么要求 reducer 是纯函数？** 纯函数的意思是：同样的输入永远得到同样的输出，且不产生副作用（不发起 API 请求、不修改 DOM、不修改传入的参数）。原因有二：① **可预测性**——如果你在 reducer 里发了请求，那 `dispatch({ type: 'INCREMENT' })` 到底干了什么就无法确定了；② **调试**——Redux DevTools 的"时间旅行"（撤销/重放）依赖 reducer 是可重现的纯函数。

现代 Redux 使用 Redux Toolkit，它提供了几个关键工具来减少模板代码。这些 API 会在下一节的实际场景中逐一解释。

### 3. 实际应用场景

#### 场景1：主题切换 — 适合 Context（低频变化数据）

这个场景中，主题变化频率很低（用户手动点一下切换按钮），用 Context 完全合理。

```jsx
// 1. 定义主题颜色
const themes = {
  light: { bg: '#fff', text: '#333' },
  dark:  { bg: '#333', text: '#fff' },
}

// 2. 创建 Context
// createContext 创建一个"容器"，可以放任意值。参数是默认值（当找不到 Provider 时使用）
const ThemeContext = createContext(themes.light)

// 3. Provider 组件：包裹整个应用，提供主题数据
function ThemeProvider({ children }) {
  const [mode, setMode] = useState('light')
  
  // useMemo 缓存 value 对象：只有当 mode 变化时才创建新对象
  // 如果不加 useMemo，每次 ThemeProvider 重渲染都会生成新对象
  // → Context value 引用变化 → 所有消费者重渲染
  const value = useMemo(() => ({
    theme: themes[mode],
    toggle: () => setMode(m => m === 'light' ? 'dark' : 'light'),
  }), [mode])

  return <ThemeContext.Provider value={value}>{children}</ThemeContext.Provider>
}

// 4. 任意底层组件消费 Context
function ThemedButton() {
  const { theme, toggle } = useContext(ThemeContext)
  return (
    <button
      style={{ background: theme.bg, color: theme.text }}
      onClick={toggle}
    >
      切换主题
    </button>
  )
}
```

这里的新 API 解释：
- **`createContext(defaultValue)`**：创建一个 Context 对象。参数是默认值，当组件使用 `useContext` 但找不到对应的 Provider 时使用。
- **`useContext(MyContext)`**：读取最近一个 `<MyContext.Provider>` 的 `value`。如果 Provider 嵌套，取最近的那个。
- **`useMemo(() => obj, [deps])`**：缓存计算结果。只有当 `[deps]` 中的值变化时才重新执行函数。这里保证 `mode` 没变时 value 的引用不变，避免触发不必要的 Context 重渲染。

#### 场景2：购物车数据共享 — 用 Zustand（高频变化、多页面共享）

购物车数据在多个页面中频繁增删改查，且需要持久化到 localStorage。如果用 Context，每次 `addItem` 都会导致所有消费购物车 Context 的组件重渲染，性能不可接受。这时需要 Zustand。

先来看 Zustand store 的定义——你可能会好奇这里的 `create` 和 `set` 是什么：

```javascript
import { create } from 'zustand'
// create 是 Zustand 的核心函数。它接收一个"创建函数"，返回一个自定义 hook。
// 创建函数的参数 set 是一个函数，调用它来修改 state。set 可以传新值或函数。

const useCartStore = create((set, get) => ({
  // state
  items: [],

  // actions（可以直接修改 state）
  addItem: (product) => set((state) => {
    // 这里的 set((state) => ...) 是"函数式更新"，state 就是当前的最新状态
    // set 内部会做 state 合并 + 通知所有订阅者
    const existing = state.items.find(i => i.id === product.id)
    if (existing) {
      return {
        items: state.items.map(i =>
          i.id === product.id ? { ...i, quantity: i.quantity + 1 } : i
        ),
      }
    }
    return { items: [...state.items, { ...product, quantity: 1 }] }
  }),

  removeItem: (id) => set((state) => ({
    items: state.items.filter(i => i.id !== id),
  })),

  // get() 是另一个接口：获取当前 state，用于在 action 中读取数据但不订阅
  total: () => get().items.reduce((sum, i) => sum + i.price * i.quantity, 0),
}))
```

这里需要解释几个点：
- **`set` 为什么可以直接修改 state？** 因为 Zustand 内部用了 immer 类似的合并机制。`set((state) => ({ items: [...] }))` 返回的部分对象，Zustand 会浅合并到现有 state 上。
- **`get` 是干什么的？** `get()` 获取当前 state 的快照，与 `useStore(s => s.items)` 不同——它不会建立订阅关系。所以 `total()` 中用 `get()` 获取 items 来计算总额，组件调用 `total()` 时不会因为 items 变化而自动重渲染（只有显式订阅了 `total` 才重渲染）。

在组件中使用：

```jsx
function CartPage() {
  // useStore 接收一个"selector"函数，选择组件关心的那部分 state
  // 这里只订阅 items——store 中其他字段变化时 CartPage 不会重渲染
  const items = useCartStore((state) => state.items)
  const removeItem = useCartStore((state) => state.removeItem)

  return (
    <div>
      {items.length === 0 ? <EmptyCart /> : (
        items.map(item => (
          <div key={item.id}>
            {item.name} x{item.quantity}
            <button onClick={() => removeItem(item.id)}>删除</button>
          </div>
        ))
      )}
    </div>
  )
}

function ProductCard({ product }) {
  const addItem = useCartStore((state) => state.addItem)
  return <button onClick={() => addItem(product)}>加入购物车</button>
}
```

**`useSelector` 是 Zustand 的核心模式**：`useStore(selector)` 中的 `selector` 是一个函数，从全量 state 中提取组件需要的那部分。Zustand 内部通过 `===` 比较 selector 的返回值——如果这次和上次相同，就不触发重渲染。所以 `useCartStore(s => s.items)` 只在 `items` 引用变化时重渲染。

如果需要持久化到 localStorage，Zustand 提供了 `persist` 中间件：

```javascript
import { persist } from 'zustand/middleware'

const useCartStore = create(
  persist(
    (set, get) => ({ /* 同上 */ }),
    { name: 'cart-storage' } // 自动存到 localStorage 的 key
  )
)
```

每次 state 变化后，`persist` 中间件自动将 state 序列化存入 localStorage。应用启动时自动读取恢复。

#### 场景3：用户信息获取 + 异步流程 — 用 Redux Toolkit

当需要处理复杂的异步流程（请求 → loading → 成功/失败 → 超时重试），Redux Toolkit 提供了 `createAsyncThunk` 来管理异步状态。这个场景中，我们获取用户信息并显示：

```javascript
// store/userSlice.js
import { createSlice, createAsyncThunk } from '@reduxjs/toolkit'

// createAsyncThunk：创建一个"异步 action 创建器"
// 参数1: 动作名称（用于 DevTools 调试）
// 参数2: 异步函数，接收 (参数, thunkAPI) → 必须返回 Promise
// 自动生成三个 action type: user/fetchUser/pending, /fulfilled, /rejected
export const fetchUser = createAsyncThunk(
  'user/fetchUser',
  async (userId, { rejectWithValue }) => {
    try {
      const res = await fetch(`/api/users/${userId}`)
      if (!res.ok) throw new Error('请求失败')
      return await res.json()  // fulfilled 时，payload = 返回值
    } catch (e) {
      return rejectWithValue(e.message) // rejected 时，payload = 错误信息
    }
  }
)

// createSlice：定义 reducer + action 的"切片"
// 它帮我们省去了手写 action type 常量和 switch-case 的样板代码
const userSlice = createSlice({
  name: 'user',
  initialState: {
    data: null,
    loading: false,  // 是否正在请求
    error: null,      // 错误信息
  },
  // 同步 action（不需要额外 dispatch 函数）
  reducers: {
    clearUser: (state) => { state.data = null; state.error = null }
    // ↑ Immer 允许我们"直接修改"state，其实是 immer 在背后生成了不可变更新
  },
  // 异步 action 的处理
  extraReducers: (builder) => {
    builder
      .addCase(fetchUser.pending, (state) => {
        state.loading = true    // 请求开始 → loading
        state.error = null
      })
      .addCase(fetchUser.fulfilled, (state, action) => {
        state.loading = false   // 请求成功 → 存数据
        state.data = action.payload
      })
      .addCase(fetchUser.rejected, (state, action) => {
        state.loading = false   // 请求失败 → 存错误
        state.error = action.payload
      })
  },
})
```

这个文件中出现了几个新 API，逐一解释：

- **`createAsyncThunk(name, asyncFn)`**：创建一个异步 thunk。它自动 dispatch 三个 action：`pending`（请求开始）、`fulfilled`（请求成功）、`rejected`（请求失败）。你不需要手动写 `dispatch({ type: 'LOADING' })` 再 `dispatch({ type: 'SUCCESS' })`，它全自动处理。
- **`createSlice(options)`**：创建一个"reducer 切片"。`name` 是命名空间，`initialState` 是初始数据，`reducers` 是同步 action，`extraReducers` 是处理其他 slice 或异步 thunk 产生的 action。
- **`builder.addCase(actionCreator, reducer)`**：在 `extraReducers` 中指定某个 action 被 dispatch 时如何更新 state。三个 case 分别处理请求的三个状态。

组件中使用时，需要理解 `useDispatch` 和 `useSelector` 这两个 hook：

```jsx
import { useDispatch, useSelector } from 'react-redux'

function UserProfile({ userId }) {
  // useDispatch：返回 store 的 dispatch 函数
  // Redux 不允许直接修改 store，必须通过 dispatch(action) 来触发
  const dispatch = useDispatch()

  // useSelector：从 store 中提取组件需要的 state
  // 参数是"selector"函数，从全量 state 中取一部分
  // Redux 内部用 === 比较返回值，只有变化时才重渲染
  const { data, loading, error } = useSelector((state) => state.user)

  // 挂载时 dispatch 异步 thunk
  useEffect(() => {
    dispatch(fetchUser(userId))
  }, [userId, dispatch])

  if (loading) return <div>加载中...</div>
  if (error) return <div>出错了: {error}</div>
  return <div>{data?.name}</div>
}
```

这里的关键：
- **`useDispatch()`**：返回 store 的 `dispatch` 函数。Redux 中**唯一修改 state 的方式**就是 `dispatch(someAction)`。
- **`useSelector(selectorFn)`**：从 Redux store 中提取数据。`selectorFn` 接收整个 state 树，返回你关心的部分。当 dispatch 导致 state 变化时，useSelector 会用 `===` 比较 selector 的上次返回值和当前返回值——不同则触发重渲染。

**整个异步流程的时间线**：

```text
组件挂载 → useEffect 中 dispatch(fetchUser(1))
  ↓
createAsyncThunk 自动 dispatch 'user/fetchUser/pending'
  → state.user.loading = true, state.user.error = null
  → useSelector 发现 loading 变化 → UserProfile 重渲染 → 显示"加载中..."
  ↓
执行 async 函数：fetch('/api/users/1')
  ↓
请求成功 → dispatch 'user/fetchUser/fulfilled', payload = 用户数据
  → state.user.loading = false, state.user.data = 用户数据
  → useSelector 发现 data 变化 → UserProfile 重渲染 → 显示用户信息
  ↓
（如果失败 → dispatch 'user/fetchUser/rejected', payload = 错误信息
  → state.user.loading = false, state.user.error = 错误信息
  → UserProfile 重渲染 → 显示"出错了: xxx"）
```

### 4. 常见误区 & 实际项目中的坑

#### 误区1：把 Context 当成"万能状态管理"

```javascript
// ❌ 错误做法：所有数据塞到一个 Context
const AppContext = createContext()

function App() {
  const [user, setUser] = useState(null)
  const [cart, setCart] = useState([])
  const [theme, setTheme] = useState('light')
  const [notifications, setNotifications] = useState([])

  return (
    <AppContext.Provider value={{ user, cart, theme, notifications, setUser, setCart, setTheme, setNotifications }}>
      <Main />
    </AppContext.Provider>
  )
}
```

**为什么错？** 任何一次 `setCart` 都会导致所有使用 `useContext(AppContext)` 的组件重渲染，哪怕它只关心 `theme`。如果 30 个组件消费这个 Context，一次购物车更新触发 30 次重渲染，其中 29 次是不必要的。

**正确做法**：把 Context 拆分成多个（`UserContext`、`ThemeContext`、`NotificationContext`），或用 Zustand 代替。

#### 误区2：在 Redux reducer 中做异步操作

```javascript
// ❌ 错误做法
const userSlice = createSlice({
  name: 'user',
  initialState: { data: null },
  reducers: {
    fetchUser: (state, action) => {
      fetch(`/api/users/${action.payload}`).then(r => r.json()).then(data => {
        state.data = data  // reducer 必须是纯函数，不能有异步！
      })
    }
  }
})
```

**为什么错？** Redux 强制要求 reducer 是纯函数。这里在 reducer 内部调用了 `fetch`，违反了"相同的输入永远得到相同的输出"的约定。如果你用 Redux DevTools 做"时间旅行"（回退到之前的状态），reducer 会重新执行，那就等于重新发了一遍 HTTP 请求。

**正确做法**：异步操作放在 `createAsyncThunk` 中（上面场景3已经展示），或放在自定义 middleware（redux-saga/thunk）中。

#### 误区3：useSelector 返回新对象导致无限循环

```javascript
// ❌ 错误做法
function UserProfile() {
  const user = useSelector(state => ({
    name: state.user.name,
    role: state.user.role,
  }))
  // 每次 dispatch 都会执行这个函数 → 返回一个新对象 { name, role }
  // 新对象 !== 旧对象（即使内容相同）→ useSelector 认为变化了 → 重渲染
  // 如果这个组件又 dispatch 了一个 action → 循环...
  return <div>{user.name}</div>
}
```

**原因**：`useSelector` 使用 `===`（严格引用相等）比较 selector 的返回值。每次执行 `state => ({ name: ..., role: ... })` 都创建一个新的对象字面量，所以即使 `name` 和 `role` 没变，对象的引用也不同了。

**两种解法**：

```javascript
// 解法1：shallowEqual 做浅比较（比较对象的每个字段是否相等，而不是比较引用）
import { shallowEqual } from 'react-redux'
const user = useSelector(state => ({
  name: state.user.name,
  role: state.user.role,
}), shallowEqual)

// 解法2：拆分为独立的 useSelector，每个只取一个基础类型值
const name = useSelector(state => state.user.name)  // 基础类型 string
const role = useSelector(state => state.user.role)  // 基础类型 string
// 基础类型直接用 === 比较，没问题
```

### 5. 与相关知识的关联 & 对比

#### 通信方式总览

| 通信方式 | 数据方向 | 适用场景 | 优点 | 缺点 |
|---|---|---|---|---|
| props | 父 → 子 | 直接父子 | 显式、类型安全 | 层级深时 Props Drilling |
| 回调函数 | 子 → 父 | 直接父子 | 简单直接 | 层级深时传递繁琐 |
| 组合 (children) | 父 → 子（模板） | 布局组件 | 绕过 Props Drilling | 灵活性有限 |
| Context | 祖先 → 后代 | 跨层级、低频数据 | 跳过中间层 | value 变化导致大范围重渲染 |
| Zustand | 任意组件 | 全局共享、中小项目 | 零样板代码、按需订阅 | 生态小于 Redux |
| Redux Toolkit | 任意组件 | 全局共享、大型项目 | 可预测、强约束、DevTools | 样板代码较多 |

#### 细粒度对比：Redux Toolkit vs Zustand vs Context

| 维度 | Redux Toolkit | Zustand | Context |
|---|---|---|---|
| 样板代码 | 中等（createSlice + Provider） | 极少（一个 create） | 少 |
| 性能（避免无关重渲染） | useSelector 按需订阅 | 按 selector 订阅 | 差（所有消费者重渲染） |
| 异步支持 | createAsyncThunk 内置 | 自行处理（action 中 async/await） | 不适用 |
| 中间件 | 内置 thunk + 可选 saga | 插件式（immer/devtools/persist） | 无 |
| Provider 包裹 | 必要 | 不需要 | 必要 |
| TypeScript 支持 | 好 | 极好（天然推导） | 好 |
| 服务器端渲染 (SSR) | 支持 | 支持（需注意 store 隔离） | 支持 |
| bundle 体积 | ~12KB（压缩） | ~1KB（压缩） | 内置 |

### 6. 现代最佳实践（2024-2025）

**核心原则：能不引入全局状态管理，就不引入。** 很多团队一上来就上 Redux，结果项目只有三四个页面、十几个组件，90% 的 Redux 代码是为了用 Redux 而写的模板。

**决策流程**：

```text
数据需要跨组件共享吗？
  ├─ 仅父子 → props + 状态提升
  ├─ 跨层级但低频（主题/语言/用户） → Context + useMemo 缓存
  └─ 跨层级且高频（购物车/实时数据）/ 多页面共享 → 状态管理库
       ├─ 中小型项目 → Zustand
       ├─ 大型项目 / 多人协作 → Redux Toolkit
       └─ 服务端数据缓存 → TanStack Query（React Query），不要放 Redux
```

**具体实践**：

1. **Context + useMemo 组合使用**——没有缓存 value 的 Context 就是性能炸弹，这两个必须一起用。

2. **Zustand + persist 自动持久化**——购物车、偏好设置等需要缓存的场景，一行代码搞定：

```javascript
const useStore = create(persist(
  (set) => ({ ... }),
  { name: 'local-storage-key' }
))
```

3. **服务端数据用 TanStack Query，不要放 Zustand/Redux**——API 返回的数据有缓存失效、重新请求、stale-while-revalidate 等特性，这些都是专业缓存库（TanStack Query / SWR）解决的问题。自己用 Zustand 存请求结果，还要手动处理 loading/error/重新请求，徒增工作量。

4. **Redux Toolkit 优于传统 Redux**——如果你的项目必须用 Redux（团队规范/遗留系统），一定要用 Redux Toolkit，不要再手写 action type 和 switch-case。

### 7. 常见疑问解答

**Q：Zustand 不用 Provider 包裹，那它是怎么让所有组件共享同一个 store 的？**

A：`create` 函数在模块级别（文件顶部）创建了一个普通的 JavaScript 对象，这个对象在模块加载时只创建一次。所有 `import { useStore } from './store'` 的组件导入的都是同一个 store 实例。这个 store 是独立于 React 组件树存在的，所以不需要 Provider 来"注入"到组件树中。而 Redux 虽然 store 也是模块级别的，但 React-Redux 的 `useSelector` 和 `useDispatch` 需要从 React Context 中获取 store 引用，所以必须用 `<Provider store={store}>` 包裹。Zustand 的 hook 内部通过闭包直接引用了 store 实例，绕过了 Context，因此不需要 Provider。

**Q：useSelector 的"按需订阅"是每次 selector 返回值都做深比较吗？**

A：不是深比较，是**引用比较（===）**。这就是为什么从 selector 返回新对象会导致无限重渲染（上面误区3）。Zustand 的按 selector 订阅也是同样的原理——用 `===` 比较 selector 返回值。如果你需要返回一个对象又不想每次都创建新引用，可以用 React 的 `useShallow`（Zustand 4.4+）或 Redux 的 `shallowEqual`。或者更好的做法是：一个组件拆多个 `useSelector`，每个只取一个基础类型。

**Q：createAsyncThunk 和直接在组件里 fetch + dispatch 有什么区别？**

A：区别在于"异步流程的状态管理放在了哪里"。直接在组件里 fetch + dispatch：

```javascript
// 组件中写：需要手动管理三个状态
function UserProfile({ userId }) {
  const [data, setData] = useState(null)
  const [loading, setLoading] = useState(false)
  const [error, setError] = useState(null)

  useEffect(() => {
    setLoading(true)
    fetch(`/api/users/${userId}`)
      .then(r => r.json())
      .then(d => { setData(d); setLoading(false) })
      .catch(e => { setError(e); setLoading(false) })
  }, [userId])

  if (loading) return <Loading />
  if (error) return <Error message={error} />
  return <div>{data.name}</div>
}
```

用 `createAsyncThunk` + `createSlice`：

```javascript
// store 中定义，组件中只需 dispatch + useSelector
function UserProfile({ userId }) {
  const dispatch = useDispatch()
  const { user, loading, error } = useSelector(s => s.user)

  useEffect(() => { dispatch(fetchUser(userId)) }, [userId])
  // loading/error/data 都在 selector 中自动获取
  // 不需要手写 setLoading(true)/setError(null)
}
```

`createAsyncThunk` 解决的问题就是：把异步流程的三个状态（loading/success/error）从组件中剥离出来，统一放在 slice 中管理。当多个组件都需要请求同一个接口时，你不需要在每个组件里重复写 `setLoading(true)` 和 `try/catch`。

**关联知识点索引**
- `React Hooks 详解.md` — useReducer、useContext 是理解状态管理的基础
- `React 渲染机制与性能优化.md` — 理解 Context 重渲染的性能问题根源
- `React vs Vue 对比.md` — React 的状态管理与 Vue 的 Pinia 设计理念对比
