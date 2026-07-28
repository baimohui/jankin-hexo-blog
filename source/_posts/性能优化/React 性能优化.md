---
title: React 性能优化
categories: 
- 性能优化
tags:
- React
- 性能优化
- 渲染优化
- useMemo
- React.memo
---

## 一、React 性能问题的本质

React 的性能问题几乎都指向同一个根源：**不必要的重新渲染**。当组件的 state、props 或 context 变化时，React 会重新调用该组件及其所有子组件的 render 函数（或函数体），即使子组件的 props 没有变化。<!--more-->

```text
父组件状态更新
  │
  ├── 父组件重新渲染
  │   ├── 子组件 A（props 没变）→ ❌ 不必要地重渲染
  │   ├── 子组件 B（props 变了）→ ✅ 需要重渲染
  │   └── 子组件 C（props 没变）→ ❌ 不必要地重渲染
  │
  └── 优化手段
      ├── React.memo → 阻止 props 未变时的重渲染
      ├── useMemo → 缓存计算结果，避免无效计算
      ├── useCallback → 稳定函数引用，配合 memo 生效
      └── 状态下移 → 减少变动的范围
```

## 二、优化手段详解

### React.memo——防止 props 未变时的重渲染

```jsx
// ❌ 父组件每次渲染，Child 也会跟着渲染
function Child({ name }) {
  return <div>{name}</div>;
}

// ✅ 只有 name 变化时才重渲染
const MemoChild = React.memo(function Child({ name }) {
  return <div>{name}</div>;
});

// 也支持自定义比较函数
const MemoChild = React.memo(Child, (prevProps, nextProps) => {
  return prevProps.name === nextProps.name;
});
```

```text
使用原则：
├── 组件接收 props 且渲染次数频繁 → memo
├── 组件本身渲染很重（大列表、图表）→ memo
├── 组件 props 基本不变 → memo
└── 组件几乎每次都重新传入新 props → memo 无效，需配合 useCallback
```

### useMemo——缓存计算结果

```jsx
// ❌ 每次渲染都重新计算，即使依赖没变
function Dashboard({ data, filter }) {
  const filtered = data.filter(d => d.type === filter)
                     .sort((a, b) => b.value - a.value);
  return <Chart data={filtered} />;
}

// ✅ 只有 data 或 filter 变化时才重新计算
function Dashboard({ data, filter }) {
  const filtered = useMemo(() => {
    return data.filter(d => d.type === filter)
               .sort((a, b) => b.value - a.value);
  }, [data, filter]);
  return <Chart data={filtered} />;
}
```

```text
适用场景：
├── 大数组的 map/filter/sort 操作
├── 深拷贝或复杂对象转换
├── React.createElement 结果
└── 需要引用的稳定对象（配合 memo）
```

### useCallback——稳定函数引用

```jsx
// ❌ 每次渲染新建函数 → 即使子组件用了 memo 也无法阻止重渲染
function Parent() {
  const handleClick = () => console.log('clicked');
  return <MemoButton onClick={handleClick} />;
}

// ✅ 引用稳定 → MemoButton 不会因为 handleClick 变化而重渲染
function Parent() {
  const handleClick = useCallback(() => console.log('clicked'), []);
  return <MemoButton onClick={handleClick} />;
}
```

```text
使用原则：
├── 传给被 React.memo 包裹的子组件 props → useCallback
├── 作为 useEffect 的依赖 → useCallback
├── 自定义 Hook 返回的函数 → useCallback
└── 简单内联函数（如 onChange）且无 memo 子组件 → 不需要
```

### useMemo 和 useCallback 的误区

```text
❌ 不要把所有变量都用 useMemo 包裹
   → 缓存本身有内存和比较开销
   → 只对昂贵的计算或引用稳定性有意义

❌ 不要用 useCallback 包裹所有函数
   → 每次渲染依然创建函数，只是多了个 useCallback 调用
   → 只有配合 memo 子组件时才有意义

❌ 不要用 useMemo 做"提前优化"
   → 先测量再优化
   → 简单的计算（小于 1000 项的 map）不需要 useMemo
```

### 状态下移——减少状态变化影响范围

```jsx
// ❌ 状态在父组件，整个子树跟着重渲染
function Page() {
  const [count, setCount] = useState(0);
  return (
    <div>
      <Counter count={count} setCount={setCount} />
      <Header />            // ❌ 因为 count 变化而重渲染
      <Sidebar />
      <Footer />
    </div>
  );
}

// ✅ 状态下移到使用它的组件
function Page() {
  return (
    <div>
      <CounterBlock />
      <Header />
      <Sidebar />
      <Footer />
    </div>
  );
}

function CounterBlock() {
  const [count, setCount] = useState(0);
  return <Counter count={count} setCount={setCount} />;
}
```

### Context 优化

```jsx
// ❌ Context value 每次新建，所有消费者都重渲染
<ThemeContext.Provider value={{ theme, setTheme }}>

// ✅ useMemo 稳定 value 引用
function App() {
  const [theme, setTheme] = useState('light');
  const value = useMemo(() => ({ theme, setTheme }), [theme]);
  return (
    <ThemeContext.Provider value={value}>
      <Content />
    </ThemeContext.Provider>
  );
}
```

## 三、懒加载与代码分割

### 路由级代码分割

```jsx
const Dashboard = lazy(() => import('./pages/Dashboard'));

function App() {
  return (
    <Suspense fallback={<Loading />}>
      <Route path="/dashboard" element={<Dashboard />} />
    </Suspense>
  );
}
```

### 组件级按需加载

```jsx
function Editor() {
  const [showChart, setShowChart] = useState(false);
  return (
    <div>
      <button onClick={() => setShowChart(true)}>显示图表</button>
      {showChart && (
        <Suspense fallback={<div>加载中...</div>}>
          <HeavyChart />
        </Suspense>
      )}
    </div>
  );
}
const HeavyChart = lazy(() => import('./HeavyChart'));
```

## 四、排查手段

### React DevTools Profiler

```text
操作步骤：
1. 安装 React DevTools 浏览器扩展
2. 打开 Profiler 面板，点击录制按钮
3. 执行需要分析的操作
4. 停止录制，查看火焰图

排查问题：
├── 灰色柱状条 → 应该跳过但没跳过 → 检查 React.memo
├── 渲染耗时长的组件 → useMemo 缓存数据
├── 不应该渲染的组件高亮了 → props 未稳定或 context 变化
└── 渲染次数过多的组件 → 状态提升过高
```

### Highlight Updates

```text
操作步骤：
1. React DevTools → Settings → Highlight updates
2. 在页面中交互
3. 观察绿/蓝色闪烁范围

排查问题：
├── 闪烁范围过大 → 不必要的重渲染
├── 闪烁频率过高 → 状态变化触发过多渲染
└── 应该闪烁的没闪烁 → props 或 state 未正确传递
```

### Why Did You Render

```bash
npm install @welldone-software/why-did-you-render
```

```jsx
import React from 'react';

if (process.env.NODE_ENV === 'development') {
  const whyDidYouRender = require('@welldone-software/why-did-you-render');
  whyDidYouRender(React, {
    trackAllPureComponents: true,
    trackHooks: true,
  });
}

const MyComponent = React.memo(function MyComponent(props) {
  return <div>{props.name}</div>;
});
MyComponent.whyDidYouRender = true;
```

```text
输出示例：
"WhyDidYouRender - MyComponent
  Props Diff:
  - name: 'Alice' → 'Alice' (same value, different reference)"

→ name 值相同但引用不同 → 需要 useCallback/useMemo
```

## 五、排查清单

| 现象 | 排查手段 | 常用方案 |
|------|----------|----------|
| 输入框输入卡顿 | Profiler 火焰图 | 状态下移、React.memo |
| 列表重排序卡顿 | Highlight Updates | 唯一 key、虚拟滚动 |
| 弹窗打开慢 | Network 瀑布图 | React.lazy + 代码分割 |
| 父更新子组件全重渲染 | Why Did You Render | React.memo + useCallback |
| Context 变化大面积重渲染 | Profiler | 拆分 Context、useMemo value |
| 图表渲染慢 | flame graph | useMemo 缓存数据 |
| 页面切换白屏 | Network 图 | Suspense + 路由级分割 |
