---
title: React Router 6 路由
categories: 
- React
tags:
- React
- React Router
- 路由
---

## 【面试速答版】

<!-- more -->
### Q1: "React Router 6 相比 React Router 5 有哪些主要变化？"

核心变化是 **API 全面升级**：`<Switch>` 改为 `<Routes>`（自动匹配最佳路由）；`component={X}` 改为 `element={<X />}`；`useHistory()` 改为 `useNavigate()`；路由参数从 `props.match.params` 改为 `useParams()` hook。V6.4+ 引入了基于数据路由的 API——`createBrowserRouter` + `RouterProvider`，支持 `loader`（路由组件渲染前自动执行数据预加载）和 `action`（表单提交处理），类似 Next.js 的约定式数据加载。此外，嵌套路由通过 `<Outlet>` 组件渲染子路由，不再需要手动嵌套 Route 组件。

### Q2: "React Router 6 中的嵌套路由和布局路由是怎么实现的？"

通过 `<Outlet />` 组件。在父路由的 element 中放入 `<Outlet />`，子路由的组件就会渲染在这个位置。结合布局组件（如 Header + Footer + Outlet），可以实现"页眉/页脚不变，中间内容区域切换"的效果。在路由配置中，父子关系通过 `children` 数组体现：

```jsx
const router = createBrowserRouter([
  {
    path: '/',
    element: <Layout />,  // Layout 内含 <Outlet />
    children: [
      { index: true, element: <Home /> },
      { path: 'about', element: <About /> },
    ],
  },
])
```

### Q3: "React Router 6 的 loader 和 action 是什么？"

`loader` 是路由级别的**数据预加载函数**——进入路由前自动执行，返回的数据通过 `useLoaderData()` 在组件中获取。`action` 是路由级别的**表单提交处理函数**——当 `<Form>` 组件提交时自动调用，返回的数据通过 `useActionData()` 获取。两者都属于 V6.4+ 的"数据路由 API"（需要 `createBrowserRouter` + `RouterProvider`），让"获取数据 → 等待 → 渲染页面"这个流程更声明式，不需要在组件中用 `useEffect` 手动控制。

## 【深入理解版】

### 1. 这个知识点要解决什么问题？

在单页应用（SPA）中，路由解决三个核心问题：

1. **URL 与 UI 的对应**：访问 `/user/123` 显示用户详情页，访问 `/about` 显示关于页。刷新页面后不会回到首页，而是停留在当前路由。
2. **前进/后退**：浏览器的前进/后退按钮正常工作。
3. **导航过程中的控制**：登录检查、权限验证、数据预加载、离开确认。

React Router 是 React 生态中最主流的路由方案。V6 是 2022 年发布的重大升级，核心改进是**让路由和数据加载更紧密地配合**——V6.4+ 的 loader 允许你在渲染组件之前就把数据准备好，不再需要在组件中写 `useEffect` 发请求。

### 2. 核心原理/执行过程

#### 2.1 两种路由创建方式

React Router 6 提供了两种方式创建路由，适用不同场景：

**方式一：组件式（适合中小项目）**

```jsx
import { BrowserRouter, Routes, Route, Link } from 'react-router-dom'

function App() {
  return (
    <BrowserRouter>
      <nav>
        <Link to="/">首页</Link>
        <Link to="/about">关于</Link>
      </nav>
      <Routes>
        <Route path="/" element={<Home />} />
        <Route path="/about" element={<About />} />
        <Route path="/user/:id" element={<User />} />
      </Routes>
    </BrowserRouter>
  )
}
```

`<BrowserRouter>` 使用 HTML5 History API（`pushState`、`popstate`）来同步 URL 和 UI，不依赖 URL 中的 `#`。URL 看起来像普通网站：`https://example.com/user/123`。

**方式二：数据路由 API（适合中大型项目）**

```jsx
import { createBrowserRouter, RouterProvider } from 'react-router-dom'

const router = createBrowserRouter([
  {
    path: '/',
    element: <Layout />,
    children: [
      { index: true, element: <Home /> },
      { path: 'about', element: <About /> },
      { path: 'user/:id', element: <User /> },
    ],
  },
])

root.render(<RouterProvider router={router} />)
```

`createBrowserRouter` 接收一个路由配置数组（而不是 JSX 组件），返回一个 router 实例，然后通过 `<RouterProvider>` 注入到应用中。这种方式的优势是：**支持 loader 和 action**，且 router 实例可以在组件外部访问（用于编程式导航）。

#### 2.2 嵌套路由与布局路由

嵌套路由是 V6 的核心设计——父路由的 `element` 中放置 `<Outlet />`，子路由渲染在该位置：

```jsx
// 路由配置
const router = createBrowserRouter([
  {
    path: '/',
    element: <RootLayout />,
    children: [
      { index: true, element: <Home /> },
      { path: 'dashboard', element: <Dashboard /> },
      {
        path: 'settings',
        element: <SettingsLayout />,  // 二级布局
        children: [
          { index: true, element: <Profile /> },
          { path: 'account', element: <Account /> },
        ],
      },
    ],
  },
])

// RootLayout.jsx
function RootLayout() {
  return (
    <div>
      <header>顶部导航</header>
      <main>
        <Outlet />  {/* 子路由（Home/Dashboard/Settings...）渲染在这里 */}
      </main>
      <footer>页脚</footer>
    </div>
  )
}
```

`<Outlet />` 可以理解为"子路由内容插槽"。父组件做布局（Header + Outlet + Footer），子组件只负责自己的内容。

**索引路由（index route）**：`{ index: true, element: <Home /> }` 表示当路径就是 `/` 时渲染 Home，不需要另一个 `path: ''`。

#### 2.3 导航与参数

```jsx
import { useNavigate, useParams, useSearchParams } from 'react-router-dom'

function User() {
  // 获取 URL 参数：/user/123 → params.id = "123"
  const { id } = useParams()

  const navigate = useNavigate()

  // 获取 query 参数：/user/123?tab=info
  const [searchParams] = useSearchParams()
  const tab = searchParams.get('tab') || 'info'

  return (
    <div>
      <h1>用户 {id}</h1>
      <p>当前标签: {tab}</p>
      <button onClick={() => navigate('/')}>返回首页</button>
      {/* navigate 也可以传相对路径：navigate('..') 回到上级路由 */}
    </div>
  )
}
```

这里出现的三个 hook：
- **`useParams()`**：返回 URL 中的动态参数（路由路径中 `:id` 匹配的部分）
- **`useNavigate()`**：返回编程式导航函数，替代 V5 的 `useHistory().push()`
- **`useSearchParams()`**：类似 `useState`，但数据源是 URL query 参数。`searchParams.get('key')` 读取，`setSearchParams({ key: 'val' })` 写入

#### 2.4 loader 与 action — 数据路由的核心

这是 V6.4+ 最值得关注的新能力。**loader** 在路由组件渲染之前自动执行，用于预加载数据：

```jsx
const router = createBrowserRouter([
  {
    path: '/products/:id',
    loader: async ({ params }) => {
      // loader 在组件渲染前执行
      // params.id 对应路径中的 :id
      const response = await fetch(`/api/products/${params.id}`)
      if (!response.ok) {
        throw new Response('Not Found', { status: 404 })
      }
      return response.json()  // 返回的数据通过 useLoaderData 获取
    },
    element: <Product />,
    errorElement: <ErrorPage />,  // loader 抛异常时渲染这个组件
  },
])
```

```jsx
function Product() {
  // useLoaderData 获取 loader 的返回值
  const product = useLoaderData()
  return <div>{product.name} - ¥{product.price}</div>
}
```

**`loader` 解决了什么问题？** 在 V6.4 之前，获取数据要在组件中用 `useEffect` + `fetch`。这导致两个问题：① 组件先渲染空内容，数据回来后再填充（闪白）；② 每个页面都要重复写 `useEffect` 里的 loading/error 逻辑。`loader` 让数据获取在渲染之前完成，路由切换时会等待 loader 返回再渲染页面。如果 loader 抛异常，渲染 `errorElement` 而不是白屏。

**action** 处理表单提交，与 loader 对称：

```jsx
const router = createBrowserRouter([
  {
    path: '/products/new',
    action: async ({ request }) => {
      // request.formData() 获取表单数据
      const formData = await request.formData()
      const product = Object.fromEntries(formData)

      const res = await fetch('/api/products', {
        method: 'POST',
        body: JSON.stringify(product),
      })

      if (!res.ok) {
        // 返回数据，表单中通过 useActionData 获取，用于显示错误
        return { error: '保存失败', fields: product }
      }
      // 成功后跳转到列表页
      return redirect('/products')
    },
    element: <NewProduct />,
  },
])
```

### 3. 实际应用场景

#### 场景1：权限控制（路由守卫）

React Router 6 没有像 Vue Router 那样的 `beforeEach` 钩子，但可以通过 loader 做权限验证：

```jsx
function authLoader() {
  const token = localStorage.getItem('token')
  if (!token) {
    // 未登录 → 重定向到登录页，同时保留来源 URL 以便登录后跳回
    return redirect(`/login?redirect=${window.location.pathname}`)
  }
  return null  // 继续渲染
}

const router = createBrowserRouter([
  {
    path: '/dashboard',
    loader: authLoader,  // 进入 dashboard 前检查登录
    element: <Dashboard />,
  },
  {
    path: '/admin',
    loader: async () => {
      const user = await fetch('/api/user').then(r => r.json())
      if (user.role !== 'admin') {
        return redirect('/403')  // 非管理员 → 跳转 403
      }
      return null
    },
    element: <AdminPanel />,
  },
])
```

`redirect()` 函数中断当前导航，跳转到指定路径。注意它要在 loader 中 return，而不是 throw。

#### 场景2：导航离开确认

```jsx
import { useBlocker } from 'react-router-dom'

function Editor() {
  const [content, setContent] = useState('')
  const [saved, setSaved] = useState(true)

  useBlocker(
    // 当内容未保存且有导航发生时，阻止离开
    ({ currentLocation, nextLocation }) =>
      !saved && currentLocation.pathname !== nextLocation.pathname
  )

  // 如果需要确认弹窗，配合 window.confirm（V6 无内置 UI）
  // useBlocker 返回 blocker 对象，可以自定义 UI

  return (
    <div>
      <textarea value={content} onChange={e => setContent(e.target.value)} />
      <button onClick={() => { setSaved(true) }}>保存</button>
    </div>
  )
}
```

### 4. 常见误区 & 实际项目中的坑

#### 误区1：在组件外部使用 useNavigate

```javascript
// ❌ useNavigate 必须在 React 组件中调用
function handleApiError() {
  const navigate = useNavigate() // Error! 不在组件中
  navigate('/login')
}
```

**解法**：使用 `createBrowserRouter` 返回的 router 实例：

```javascript
const router = createBrowserRouter([...])
// 在组件外部（如 Axios 拦截器）：
router.navigate('/login')
```

#### 误区2：忘记给 History 模式配置服务器 fallback

```nginx
# 如果使用 BrowserRouter（HTML5 History 模式）
# 用户直接访问 /user/123 → 浏览器发请求到服务器 → 服务器没有 /user/123 路径 → 404

# 解决方案：Nginx 把所有路径指向 index.html
location / {
  try_files $uri $uri/ /index.html
}
```

如果是 Vite 开发环境，`vite.config.js` 中需要加 `historyApiFallback: true`。

#### 误区3：loader 中直接修改组件状态

```javascript
// ❌ loader 中不能访问 React 状态
const router = createBrowserRouter([{
  path: '/',
  loader: () => {
    // ❌ 不能调用 useState 或任何 hook
    // loader 在 React 组件树之外执行
  }
}])
```

**loader 的定位**：它只是一个普通的异步函数，返回数据供组件消费，不在 loader 中调用 React 的 API。

### 5. 与相关知识的关联 & 对比

| 对比维度 | React Router 5 | React Router 6 | Vue Router 4 |
|---|---|---|---|
| 核心组件 | Switch → Route | Routes → Route / RouterProvider | router-view / router-link |
| 编程式导航 | useHistory() | useNavigate() | router.push() |
| 路由参数 | props.match.params | useParams() | useRoute().params |
| 嵌套路由 | Route 嵌套 Route | <Outlet /> 插槽 | <router-view> |
| 数据加载 | 组件内 useEffect | loader（路由级别） | 无内置 |
| 表单处理 | 手动提交 | action + Form 组件 | 无内置 |
| 路由守卫 | 手动实现 | loader 中 redirect | router.beforeEach |
| 支持数据路由 | 否 | 是（V6.4+） | 否 |

### 6. 现代最佳实践（2024-2025）

1. **新项目使用 `createBrowserRouter` + `RouterProvider`**，而不是 `<BrowserRouter>`。数据路由 API 提供了 loader/action，未来会是 React Router 的主要发展方向。
2. **利用 loader 做数据预加载和权限校验**，不再在组件中用 `useEffect` 发请求和检查登录。
3. **嵌套路由 + Outlet 实现布局复用**：一个布局组件 + 多个子路由，避免在每个路由组件中重复写 Header/Footer。
4. **配合 React.lazy 做路由级代码分割**：

```jsx
const Dashboard = React.lazy(() => import('./Dashboard'))
{
  path: 'dashboard',
  element: <Suspense fallback={<Loading />}><Dashboard /></Suspense>,
}
```

5. **路由配置集中管理**：把路由配置放在独立的 `router.jsx` 文件中，不要散落在 App 组件中。

### 7. 常见疑问解答

**Q：`createBrowserRouter` 和 `<BrowserRouter>` 有什么区别？用哪个？**

A：`createBrowserRouter` 是 V6.4+ 新增的"数据路由 API"，基于标准 History API，但返回的是一个 router 实例（普通 JS 对象），通过 `RouterProvider` 注入 React 树。`<BrowserRouter>` 是传统组件式用法，直接在 React 树中用组件配置路由。区别在于：`createBrowserRouter` 支持 `loader`/`action`，且 router 实例可以在组件外访问。**新项目建议用 `createBrowserRouter`**，因为它代表了 React Router 未来的方向。

**Q：loader 和直接在 useEffect 中 fetch 有什么不同？**

A：核心区别是**执行时机**。loader 在组件渲染之前执行——React Router 会等待 loader 完成后再渲染路由组件。`useEffect` + fetch 在组件**渲染之后**执行——组件先渲染"空"状态，数据回来后再渲染"有数据"状态，用户会看到从空到有数据的闪烁。loader 避免了这种闪烁，同时也把数据获取逻辑从组件中移出，让组件更纯粹。

**Q：`<Link>` 和 `<a>` 标签在 React Router 中有什么区别？**

A：`<Link>` 组件拦截了 `click` 事件，调用 `event.preventDefault()` 阻止浏览器的页面刷新，然后通过 History API（`pushState`）更新 URL，最后触发 React Router 的路由匹配和组件渲染。整个过程没有页面请求，是"前端路由"。`<a>` 标签会触发页面刷新，相当于从服务器重新请求页面，破坏了 SPA 的体验。

**关联知识点索引**
- `React 核心概念.md` — 组件基础
- `React vs Vue 对比.md` — 与 Vue Router 的完整对比
