---
title: Web Worker 详解
categories: 
- 浏览器
tags:
- 浏览器
- Worker
- 多线程
- 性能
---

## 一、Web Worker 是什么

Web Worker 是浏览器提供的**多线程运行环境**，允许 JavaScript 在后台线程中执行耗时任务，不阻塞主线程的 UI 渲染和用户交互。<!--more-->

### 为什么需要 Worker

JavaScript 是单线程的——主线程既要处理 DOM 渲染，又要执行 JS 代码。当 JS 执行耗时任务时，页面会卡死：

```js
// 主线程执行耗时计算 → 页面冻结
function heavyTask() {
  let sum = 0;
  for (let i = 0; i < 1e10; i++) sum += i;
  return sum;
}
heavyTask();  // 用户无法点击、滚动、输入
```

Web Worker 将耗时任务放到独立线程，主线程保持流畅：

```js
// 主线程
const worker = new Worker('worker.js');
worker.postMessage(1e10);
worker.onmessage = (e) => console.log('结果：', e.data);
// 页面仍然可以正常交互
```

## 二、Worker 的限制

```text
┌─────────────────┐     postMessage      ┌─────────────────┐
│    主线程        │ ◄────────────────►   │    Worker 线程   │
│                 │    事件通信          │                 │
│  ✅ DOM 操作    │                     │  ❌ 不能操作 DOM  │
│  ✅ window 对象 │                     │  ❌ 不能访问 window│
│  ✅ document    │                     │  ❌ 不能访问 document│
│  ✅ navigator  │                     │  ✅ navigator 部分 │
│  ✅ console     │                     │  ✅ console       │
│  ✅ fetch       │                     │  ✅ fetch / XHR   │
│  ✅ localStorage│                     │  ❌ 不能访问 localStorage│
│  ✅ Canvas/WebGL│                     │  ✅ Canvas/WebGL(Offscreen)│
└─────────────────┘                     └─────────────────┘
```

**Worker 不能做的事**：
- 操作 DOM 或访问 `document`、`window`、`parent`
- 访问 `localStorage`、`sessionStorage`
- 访问主线程的 JavaScript 执行上下文

**Worker 能做的事**：
- 纯计算（数据处理、加密、压缩）
- `fetch` 请求
- `setTimeout`/`setInterval`
- `XMLHttpRequest`
- `OffscreenCanvas`（离屏 Canvas 渲染）

## 三、基本用法

### 主线程

```js
// main.js
const worker = new Worker('worker.js');

// 发送消息给 Worker
worker.postMessage({ type: 'calculate', payload: 1000000000 });

// 接收 Worker 的消息
worker.onmessage = (event) => {
  console.log('Worker 返回：', event.data);
};

// 错误处理
worker.onerror = (error) => {
  console.error('Worker 错误：', error.message);
};

// 终止 Worker
worker.terminate();
```

### Worker 线程

```js
// worker.js
self.onmessage = (event) => {
  const { type, payload } = event.data;

  if (type === 'calculate') {
    const result = heavyCompute(payload);
    self.postMessage({ type: 'result', data: result });
  }
};

function heavyCompute(n) {
  let sum = 0;
  for (let i = 0; i < n; i++) sum += i;
  return sum;
}
```

### 完整示例

```js
// main.js
const worker = new Worker('fib-worker.js');

document.getElementById('calcBtn').addEventListener('click', () => {
  const n = document.getElementById('input').value;
  worker.postMessage(parseInt(n));
});

worker.onmessage = (e) => {
  document.getElementById('result').textContent = e.data;
};
```

```js
// fib-worker.js
function fib(n) {
  if (n <= 1) return n;
  return fib(n - 1) + fib(n - 2);
}

self.onmessage = (e) => {
  const result = fib(e.data);
  self.postMessage(result);
};
```

## 四、数据传递方式

### 拷贝传递（默认）

`postMessage` 默认使用**结构化克隆算法**（structured clone）复制数据。Worker 修改数据不会影响主线程：

```js
const obj = { a: 1, b: [2, 3] };
worker.postMessage(obj);
// Worker 中修改 obj，主线程不受影响
```

支持的类型：Object、Array、Map、Set、Date、RegExp、Blob、ArrayBuffer、ImageData 等。

不支持的类型：Function、DOM 元素、Error、Symbol。

### 转移传递（Transferable）

对于 `ArrayBuffer` 等大块数据，可以使用**转移所有权**方式，零拷贝传递：

```js
// 主线程
const buffer = new ArrayBuffer(1024 * 1024 * 100);  // 100MB
worker.postMessage(buffer, [buffer]);                // 转移所有权
// 此时 buffer.byteLength === 0（主线程不再拥有该内存）
```

```js
// Worker
self.onmessage = (e) => {
  const buffer = e.data;
  // buffer.byteLength === 100MB（Worker 拥有所有权）
};
```

使用转移传递时，发送方会**失去**对该数据的访问权，适合大文件处理、WebAssembly 等场景。

## 五、Worker 类型

### Dedicated Worker（专用 Worker）

一个 Worker 实例只能被创建它的页面使用，这是最常用的类型（以上示例均为 Dedicated Worker）。

### Shared Worker（共享 Worker）

多个页面（同源）可以共享同一个 Worker 实例：

```js
// main.js（页面 A 和页面 B 都可以连接）
const worker = new SharedWorker('shared-worker.js');
worker.port.start();
worker.port.postMessage('Hello from Page A');
worker.port.onmessage = (e) => console.log(e.data);
```

```js
// shared-worker.js
const connections = [];

self.onconnect = (e) => {
  const port = e.ports[0];
  connections.push(port);

  port.onmessage = (event) => {
    // 广播给所有连接
    connections.forEach(p => p.postMessage(event.data));
  };

  port.start();
};
```

### Service Worker

Service Worker 是浏览器和网络之间的代理脚本，用于拦截请求、实现离线缓存和消息推送。它不是为计算设计的，而是为网络代理设计的。

```js
// service-worker.js
self.addEventListener('install', (event) => {
  event.waitUntil(caches.open('v1').then(cache => {
    return cache.addAll(['/', '/index.html', '/app.js']);
  }));
});

self.addEventListener('fetch', (event) => {
  event.respondWith(
    caches.match(event.request).then(cached => {
      return cached || fetch(event.request);
    })
  );
});
```

### Worker 类型对比

| 类型 | 共享 | 生命周期 | 用途 |
|------|------|----------|------|
| Dedicated Worker | 仅创建它的页面 | 页面关闭或 terminate() 终止 | 耗时计算、数据处理 |
| Shared Worker | 同源多页面共享 | 最后一个页面关闭后终止 | 跨页面状态同步、共享连接 |
| Service Worker | 同源多页面共享 | 由浏览器管理（事件驱动） | 离线缓存、推送通知 |

## 六、实际应用场景

### 场景一：大文件哈希计算

```js
// main.js
async function calculateHash(file) {
  return new Promise((resolve) => {
    const worker = new Worker('hash-worker.js');
    worker.postMessage(file);
    worker.onmessage = (e) => {
      resolve(e.data);
      worker.terminate();
    };
  });
}

// 用于文件上传前的秒传校验
const hash = await calculateHash(file);
// 检查 hash 是否已在服务器上存在
```

```js
// hash-worker.js
self.onmessage = async (e) => {
  const file = e.data;
  const buffer = await file.arrayBuffer();
  const hashBuffer = await crypto.subtle.digest('SHA-256', buffer);
  const hashArray = Array.from(new Uint8Array(hashBuffer));
  const hashHex = hashArray.map(b => b.toString(16).padStart(2, '0')).join('');
  self.postMessage(hashHex);
};
```

### 场景二：数据流处理

```js
// main.js
const worker = new Worker('stream-worker.js');

// 实时数据处理（如 WebSocket 消息的大量数据）
websocket.onmessage = (event) => {
  worker.postMessage(event.data);
};

worker.onmessage = (e) => {
  updateChart(e.data);   // 主线程只负责渲染
};
```

### 场景三：离屏 Canvas 渲染

```js
// main.js
const canvas = document.getElementById('game');
const offscreen = canvas.transferControlToOffscreen();
const worker = new Worker('render-worker.js');
worker.postMessage({ canvas: offscreen }, [offscreen]);
```

```js
// render-worker.js
self.onmessage = (e) => {
  const canvas = e.data.canvas;
  const ctx = canvas.getContext('2d');
  // 在 worker 中直接操作 canvas
  function render() {
    ctx.clearRect(0, 0, canvas.width, canvas.height);
    // ... 渲染逻辑
    requestAnimationFrame(render);
  }
  render();
};
```

## 七、注意事项

### 1. Worker 文件的同源限制

Worker 脚本必须与页面同源。可以使用 `Blob URL` 创建内联 Worker：

```js
const code = `
  self.onmessage = (e) => {
    self.postMessage(e.data * 2);
  };
`;
const blob = new Blob([code], { type: 'application/javascript' });
const worker = new Worker(URL.createObjectURL(blob));
worker.postMessage(5);
worker.onmessage = (e) => console.log(e.data);  // 10
```

### 2. 使用 importScripts 加载库

```js
// worker.js
importScripts('https://cdn.example.com/lodash.min.js');
// 现在可以在 Worker 中使用 lodash
```

ES Module Workers（部分浏览器支持）：

```js
const worker = new Worker('worker.js', { type: 'module' });
```

```js
// worker.js（type: module）
import { heavyCompute } from './utils.js';
```

### 3. 不要频繁创建和销毁 Worker

Worker 的创建有开销。对于频繁的任务，考虑复用 Worker 实例：

```js
const worker = new Worker('worker.js');
const pending = {};
let msgId = 0;

function sendToWorker(data) {
  return new Promise((resolve) => {
    const id = msgId++;
    pending[id] = resolve;
    worker.postMessage({ id, data });
  });
}

worker.onmessage = (e) => {
  const { id, result } = e.data;
  pending[id](result);
  delete pending[id];
};
```

### 4. React/Vue 中使用 Worker

```js
// React Hook 封装
function useWorker(workerPath) {
  const workerRef = useRef(null);

  useEffect(() => {
    workerRef.current = new Worker(workerPath);
    return () => workerRef.current?.terminate();
  }, [workerPath]);

  const post = useCallback((data) => {
    return new Promise((resolve) => {
      workerRef.current.onmessage = (e) => resolve(e.data);
      workerRef.current.postMessage(data);
    });
  }, []);

  return post;
}
```

## 八、常见问题

### Q1: Worker 中能使用 DOM 吗

不能。Worker 中没有 `document`、`window`、`parent` 对象。如果需要在 Worker 中操作 Canvas，使用 `OffscreenCanvas`（浏览器支持有限）。

### Q2: Worker 的全局对象是什么

Worker 的全局对象是 `self`（`DedicatedWorkerGlobalScope`），不是 `window`。

### Q3: 怎么调试 Worker

Chrome DevTools → Sources → Threads 面板可以看到所有 Worker 线程，可以像调试主线程一样设置断点。

### Q4: Worker 中能发起网络请求吗

可以。`fetch` 和 `XMLHttpRequest` 在 Worker 中均可使用。

### Q5: 什么时候不应该用 Worker

- 任务非常轻量（Worker 创建开销可能大于任务本身）
- 需要频繁 DOM 操作
- 只是简单的异步操作（`setTimeout` / `Promise` 即可）

## 九、推荐学习路径

1. 理解主线程和 Worker 线程的关系与限制
2. 掌握 `new Worker()`、`postMessage`、`onmessage` 基础通信
3. 了解结构化克隆和 Transferable 的区别
4. 实际场景：用 Worker 做文件哈希或大数据排序
5. 了解 Shared Worker 和 Service Worker 的适用场景
