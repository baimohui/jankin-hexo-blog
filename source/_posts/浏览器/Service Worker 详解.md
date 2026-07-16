---
title: Service Worker 详解
categories: 
- 浏览器
tags:
- Service Worker
- 浏览器
- 离线缓存
- PWA
- 网络请求
- Web Worker
---

# Service Worker 详解

## 【面试速答版】

### Q1: Service Worker 是什么？它和 Web Worker 有什么区别？

Service Worker 是浏览器在后台**独立于网页运行**的脚本，本质是一个**网络请求的代理服务器**。它可以拦截、修改、缓存页面的所有网络请求，实现离线访问、消息推送、后台同步等功能。

| 维度 | Service Worker | Web Worker |
|:---|:---|:---|
| **目的** | 拦截网络请求、管理缓存 | 执行 CPU 密集型计算 |
| **生命周期** | 独立于页面，安装后可长期存在 | 随页面关闭而销毁 |
| **访问 DOM** | ❌ 不能 | ❌ 不能 |
| **访问网络** | ✅ 可以拦截请求 | ❌ 只能通过 fetch |
| **与页面通信** | postMessage | postMessage |
| **持久化** | 可安装、激活、更新 | 无持久化概念 |
| **使用 HTTPS** | ✅ 必须 | ❌ 不需要 |

核心区别：**Web Worker 是「计算线程」，Service Worker 是「网络代理」**。

### Q2: Service Worker 的生命周期有哪些阶段？

Service Worker 的生命周期完全独立于网页，包含以下阶段：

```
注册 → 安装（install） → 等待（waiting） → 激活（activate） → 运行（fetch/message/...） → 更新/终止
```

**注册（Registration）**：调用 `navigator.serviceWorker.register('/sw.js')`，浏览器下载并解析脚本。

**安装（install 事件）**：首次安装或检测到脚本有字节差异时触发。通常在此事件中预缓存静态资源。

**等待（waiting）**：如果已有一个激活的 SW，新的 SW 会进入 waiting 状态，直到所有页面关闭或调用 `skipWaiting()`。

**激活（activate 事件）**：SW 开始接管页面。通常在此事件中清理旧缓存。

**运行（fetch/message 等事件）**：SW 处于激活状态后，拦截网络请求、接收消息。

**更新**：每次导航到页面时，浏览器会重新下载 SW 脚本。如果内容有变化，新 SW 进入 install → waiting → activate 循环。

**终止**：浏览器为了节省资源，会在 SW 空闲一段时间后终止它。下次有事件时重新启动。

### Q3: Service Worker 常用的缓存策略有哪些？

| 策略 | 适用场景 | 行为 |
|:---|:---|:---|
| **Cache First** | 静态资源（JS/CSS/图片） | 有缓存直接返回，无缓存则网络请求并缓存 |
| **Network First** | API 请求、最新内容 | 先网络请求，超时或失败则用缓存 |
| **Stale While Revalidate** | 非关键但需要新鲜度 | 立即返回缓存，同时后台更新缓存下次用 |
| **Network Only** | 表单提交、支付 | 始终走网络，不使用缓存 |
| **Cache Only** | 预缓存的应用 Shell | 只从缓存读取，从不请求网络 |

**开发中最常用：Stale While Revalidate**。用户获得即时响应（缓存 -> 屏幕），同时后台悄悄更新缓存，下次响应就是最新的。

---

## 【深入理解版】

### 1. Service Worker 要解决什么问题？

#### 1.1 传统 Web 应用的离线困境

假设你正在做一个笔记应用，用户在地铁上打开网页时信号很差：

```
传统 Web 应用的加载流程：
1. 浏览器请求 HTML → ❌ 无网络 → 白屏
2. 浏览器请求 CSS → ❌ 无网络 → 白屏
3. 浏览器请求 JS → ❌ 无网络 → 白屏
4. 浏览器请求 API → ❌ 无网络 → 无数据

用户看见：白屏 + 「无法连接到互联网」
```

这暴露了传统 Web 模型的根本问题：**网络是现代 Web 应用的唯一依赖。没有网络，一切归零**。

有几种「伪离线」方案，但都不完美：

**方案 1：浏览器 HTTP 缓存**
```javascript
// 服务器通过 Cache-Control 控制缓存
Cache-Control: public, max-age=3600
// 问题：浏览器缓存不可编程，缓存逻辑由服务器控制
// 问题：无法缓存 POST 请求
// 问题：无法在离线时提供自定义的降级页面
```

**方案 2：Application Cache（AppCache）**
```html
<html manifest="app.appcache">
<!-- 已被废弃，因为设计有严重缺陷：
  一旦 manifest 文件不更新，缓存永远不更新
  缓存更新后不推送到页面，需要用户手动刷新两次
  ... -->
</html>
```

**方案 3：LocalStorage / IndexedDB**
```javascript
// 手动存储数据
localStorage.setItem('data', JSON.stringify(data))
// 问题：只能存数据，不能拦截网络请求
// 问题：需要手动管理缓存逻辑
```

**Service Worker 的解决思路**：在浏览器和网络之间插入一个可编程的代理层。

```
传统模式：
浏览器 ←→ 网络

Service Worker 模式：
浏览器 ←→ Service Worker ←→ 网络
               ↓
            缓存（Cache Storage）

SW 可以决定：
- 请求直接走缓存（不上网）
- 先走缓存，后台更新网络结果
- 先走网络，失败时用缓存兜底
- 甚至完全不使用网络（完全离线应用）
```

#### 1.2 Service Worker 不是什么

在深入之前，先澄清几个常见的误解：

**Service Worker 不是**：
1. **不是页面的一部分**：SW 运行在独立的上下文（`ServiceWorkerGlobalScope`）中，不能访问 `window`、`document`、`DOM`
2. **不是持久进程**：浏览器会在 SW 空闲时终止它，下次有事件时重新启动。不能假设 SW 会一直运行
3. **不是 localStorage 替代品**：SW 使用 `Cache Storage` API 缓存网络请求，用 `IndexedDB` 存结构化数据
4. **不是所有浏览器都支持**：IE 不支持、部分低版本移动浏览器不支持

**一个 SW 可以控制多个页面**，但每个域名下的 SW 是独立的。

### 2. 核心原理与执行过程

#### 2.1 前提条件与注册

Service Worker 有三个硬性前提：

1. **HTTPS 必须**（或 localhost 用于开发）
2. **同源策略**：SW 只能控制同源页面
3. **浏览器支持**：需要 `navigator.serviceWorker` 存在

```javascript
// 注册 Service Worker（通常在页面的主 JS 中）
if ('serviceWorker' in navigator) {
  navigator.serviceWorker.register('/sw.js', {
    scope: '/' // SW 控制的路径范围（默认为 SW 脚本所在目录）
  })
    .then(registration => {
      console.log('SW 注册成功, 状态:', registration.active ? '已激活' : '未激活')
    })
    .catch(err => {
      console.error('SW 注册失败:', err)
    })
}
```

**scope 的作用**：

```javascript
// 假设 SW 文件在 /app/sw.js
navigator.serviceWorker.register('/app/sw.js')
// scope 默认是 /app/，即只控制 /app/ 路径下的页面

// 如果希望控制整个站点：
navigator.serviceWorker.register('/app/sw.js', { scope: '/' })
// 但浏览器会检查 SW 文件的路径，如果 scope 超出了 SW 文件所在目录
// 需要在响应头中设置：
// Service-Worker-Allowed: /
```

**注册后的内部流程**：

```
1. 浏览器下载 /sw.js
2. 与已有的 SW 比较字节内容
   ├── 首次注册 → 直接进入 install
   └── 已注册且内容无变化 → 跳过，使用已有 SW
       └── 已注册但内容变化 → 新的 SW 进入 install（等待激活）
3. 注册返回 ServiceWorkerRegistration 对象
```

#### 2.2 install 事件：预缓存

install 事件在 SW 首次安装或更新时触发，且**只触发一次**。这是做预缓存的时机——在用户还没访问任何页面之前，先把关键资源缓存起来。

```javascript
// sw.js — install 事件
const CACHE_NAME = 'my-app-v1'
const PRECACHE_URLS = [
  '/',
  '/index.html',
  '/styles/main.css',
  '/scripts/app.js',
  '/images/logo.png',
  '/offline.html',       // 离线降级页面
]

self.addEventListener('install', (event) => {
  console.log('SW: install 事件触发')
  
  // event.waitUntil 告诉浏览器 SW 安装尚未完成
  // 直到传入的 Promise  resolve
  event.waitUntil(
    caches.open(CACHE_NAME)
      .then(cache => {
        console.log('SW: 预缓存资源')
        return cache.addAll(PRECACHE_URLS)
      })
      .then(() => {
        // 强制新 SW 跳过 waiting 阶段，立即激活
        // 不调用的话，新 SW 会等到所有页面关闭后才激活
        return self.skipWaiting()
      })
  )
})
```

**`event.waitUntil()` 的关键作用**：它延长 install/activate 事件的存活时间，直到传入的 Promise 完成。如果 Promise reject，浏览器认为安装失败，该 SW 被丢弃。

**`self.skipWaiting()` 的作用**：让处于 waiting 状态的新 SW 立即激活，不等所有页面关闭。

#### 2.3 activate 事件：清理旧缓存

activate 事件在新 SW 激活时触发。通常在这里清理旧版本的缓存，避免磁盘空间浪费。

```javascript
// sw.js — activate 事件
self.addEventListener('activate', (event) => {
  console.log('SW: activate 事件触发')
  
  const cacheWhitelist = [CACHE_NAME] // 当前版本的缓存名
  
  event.waitUntil(
    // 获取所有缓存名称
    caches.keys().then(cacheNames => {
      return Promise.all(
        cacheNames.map(cacheName => {
          // 删除不在白名单中的旧缓存
          if (!cacheWhitelist.includes(cacheName)) {
            console.log('SW: 删除旧缓存', cacheName)
            return caches.delete(cacheName)
          }
        })
      )
    }).then(() => {
      // 立即接管所有页面，不需要等页面刷新
      // clients.claim 让 SW 立即控制所有已打开的页面
      return self.clients.claim()
    })
  )
})
```

**`self.clients.claim()` 的作用**：新 SW 激活后，默认只控制**激活后**打开的页面。已有的页面依然被旧 SW 控制（直到页面刷新）。`claim()` 让新 SW 立即控制所有页面。

**为什么需要版本化缓存名**：

```javascript
// 使用版本化的缓存名，便于更新时识别和清理
const CACHE_NAME = 'my-app-v1'
// 更新版本时改为 my-app-v2，旧缓存 my-app-v1 在 activate 中被删除
```

#### 2.4 fetch 事件：请求拦截与缓存策略

fetch 事件是 Service Worker 的核心——每次页面发出网络请求时，都会先经过 SW。

```javascript
// sw.js — fetch 事件（使用 Stale While Revalidate 策略）
self.addEventListener('fetch', (event) => {
  // event.request 是浏览器发出的 Request 对象
  console.log('SW: 拦截请求', event.request.url)
  
  // event.respondWith 告诉浏览器：我来处理这个请求
  // 传入 Response 对象或 Promise<Response>
  event.respondWith(
    // 策略：Stale While Revalidate
    fromCache(event.request)         // 1. 从缓存中获取
      .then(cachedResponse => {
        // 后台发起网络请求更新缓存（不阻塞页面）
        const fetchPromise = fromNetwork(event.request)
          .then(networkResponse => {
            // 更新缓存
            updateCache(event.request, networkResponse.clone())
            return networkResponse
          })
          .catch(() => cachedResponse) // 网络失败时返回缓存
        
        // 立即返回缓存内容（如果有）或等网络结果
        return cachedResponse || fetchPromise
      })
  )
})

// 从缓存读取
function fromCache(request) {
  return caches.match(request).then(response => {
    return response || Promise.reject('no-match')
  })
}

// 从网络获取
function fromNetwork(request) {
  return fetch(request).then(response => {
    // 只缓存成功的响应
    if (!response || response.status !== 200) {
      return response
    }
    return response
  })
}

// 更新缓存
function updateCache(request, response) {
  if (response && response.status === 200) {
    const clone = response.clone() // Response 只能消费一次，需要 clone
    caches.open(CACHE_NAME).then(cache => {
      cache.put(request, clone)
    })
  }
}
```

**`event.respondWith()` 的关键作用**：它告诉浏览器「这个请求由我来响应」。必须同步调用（在 event 回调内部），不能异步调用。

**Response 只能消费一次**：调用 `caches.put()` 或 `response.text()` 后，Response body 被消耗。如果需要在多处使用，必须先 `.clone()`。

#### 2.5 完整的缓存策略实现

**Cache First（缓存优先）**：

```javascript
// 适合静态资源：JS、CSS、字体、图片
self.addEventListener('fetch', (event) => {
  if (isStaticResource(event.request.url)) {
    event.respondWith(
      caches.match(event.request).then(cached => {
        // 有缓存 → 返回缓存
        // 无缓存 → 请求网络并缓存
        return cached || fetch(event.request).then(response => {
          return cacheResponse(event.request, response)
        })
      })
    )
  }
})

function isStaticResource(url) {
  const extensions = ['.js', '.css', '.png', '.jpg', '.svg', '.woff2', '.ttf']
  return extensions.some(ext => url.includes(ext))
}
```

**Network First（网络优先）**：

```javascript
// 适合需要最新数据的 API 请求
self.addEventListener('fetch', (event) => {
  if (event.request.url.includes('/api/')) {
    event.respondWith(
      fetch(event.request)
        .then(response => {
          // 网络成功 → 更新缓存，返回网络响应
          return cacheResponse(event.request, response.clone())
        })
        .catch(() => {
          // 网络失败 → 返回缓存（如果有）
          return caches.match(event.request).then(cached => {
            if (cached) return cached
            // 连缓存都没有，返回离线降级页面
            return caches.match('/offline.html')
          })
        })
    )
  }
})
```

**Network Only（仅网络）**：

```javascript
// 适合表单提交、支付等需要实时确认的操作
self.addEventListener('fetch', (event) => {
  if (event.request.method === 'POST' || event.request.url.includes('/api/payment')) {
    // 不调用 event.respondWith → 浏览器正常发起网络请求
    // 或者显式转发
    event.respondWith(fetch(event.request))
  }
})
```

**Cache Only（仅缓存）**：

```javascript
// 适合预缓存的应用 Shell 资源
self.addEventListener('fetch', (event) => {
  if (event.request.url.includes('/shell/')) {
    event.respondWith(
      caches.match(event.request).then(cached => {
        // 如果缓存中没有 → 请求失败（不会回退到网络）
        return cached || new Response('', { status: 404 })
      })
    )
  }
})
```

#### 2.6 message 事件：与页面通信

SW 和页面之间通过 `postMessage` 通信：

```javascript
// sw.js — 接收消息
self.addEventListener('message', (event) => {
  console.log('SW 收到来自页面的消息:', event.data)
  
  if (event.data.type === 'SKIP_WAITING') {
    self.skipWaiting()
  }
  
  if (event.data.type === 'CACHE_URLS') {
    // 页面通知 SW 缓存某些 URL
    event.waitUntil(
      caches.open(CACHE_NAME).then(cache => {
        return cache.addAll(event.data.urls)
      })
    )
  }
  
  // 回复消息给页面
  event.source.postMessage({
    type: 'ACK',
    received: event.data
  })
})

// 页面 — 发送消息给 SW
if (navigator.serviceWorker.controller) {
  navigator.serviceWorker.controller.postMessage({
    type: 'CACHE_URLS',
    urls: ['/api/data', '/images/new.png']
  })
}

// 页面 — 接收来自 SW 的消息
navigator.serviceWorker.addEventListener('message', (event) => {
  console.log('页面收到来自 SW 的消息:', event.data)
})
```

#### 2.7 Service Worker 的更新机制

SW 的更新是**自动检查、手动控制激活时机**的：

```javascript
// 页面中监听 SW 更新
let swRegistration

if ('serviceWorker' in navigator) {
  swRegistration = navigator.serviceWorker.register('/sw.js')
  
  // 监听 SW 状态变化
  swRegistration.addEventListener('updatefound', () => {
    // 检测到新 SW
    const newSW = swRegistration.installing
    console.log('发现新版本的 SW')
    
    newSW.addEventListener('statechange', () => {
      if (newSW.state === 'installed' && swRegistration.active) {
        // 新 SW 已安装，但处于 waiting 状态
        // 通知用户「有新版本可用」
        showUpdateNotification()
      }
    })
  })
}

// 用户点击「更新」按钮时
function applyUpdate() {
  if (swRegistration && swRegistration.waiting) {
    // 让等待中的新 SW 激活
    swRegistration.waiting.postMessage({ type: 'SKIP_WAITING' })
  }
}

// SW 端的 skipWaiting 处理
self.addEventListener('message', (event) => {
  if (event.data.type === 'SKIP_WAITING') {
    self.skipWaiting()
  }
})

// 页面被新 SW 接管时触发
navigator.serviceWorker.addEventListener('controllerchange', () => {
  console.log('新 SW 已激活，刷新页面使用新版本')
  window.location.reload()
})
```

**更新检查的触发时机**：
1. 用户导航到 SW scope 内的页面（每次）
2. 调用 `registration.update()`
3. 浏览器每 24 小时自动检查一次

### 3. 实际应用场景与代码示例

#### 3.1 场景 1：全栈离线笔记应用

**Step 1：注册 SW**

```javascript
// main.js
async function registerSW() {
  if (!('serviceWorker' in navigator)) {
    console.log('当前浏览器不支持 Service Worker')
    return
  }
  
  try {
    const registration = await navigator.serviceWorker.register('/sw.js', {
      scope: '/'
    })
    
    console.log('SW 注册成功:', registration.scope)
    
    // 检测新 SW
    registration.addEventListener('updatefound', () => {
      const newWorker = registration.installing
      newWorker.addEventListener('statechange', () => {
        if (newWorker.state === 'installed' && registration.active) {
          showUpdateBanner()
        }
      })
    })
  } catch (err) {
    console.error('SW 注册失败:', err)
  }
}

registerSW()
```

**Step 2：SW 缓存策略**

```javascript
// sw.js
const CACHE_NAME = 'notes-app-v1'
const STATIC_CACHE = 'notes-static-v1'

const PRECACHE_URLS = [
  '/',
  '/index.html',
  '/styles/app.css',
  '/scripts/app.js',
  '/offline.html',
  '/icons/icon-192.png',
]

// install — 预缓存静态资源
self.addEventListener('install', (event) => {
  event.waitUntil(
    (async () => {
      const cache = await caches.open(STATIC_CACHE)
      await cache.addAll(PRECACHE_URLS)
      await self.skipWaiting()
    })()
  )
})

// activate — 清理旧缓存
self.addEventListener('activate', (event) => {
  event.waitUntil(
    (async () => {
      const cacheNames = await caches.keys()
      await Promise.all(
        cacheNames.map(name => {
          if (name !== STATIC_CACHE && name !== CACHE_NAME) {
            return caches.delete(name)
          }
        })
      )
      await self.clients.claim()
    })()
  )
})

// fetch — 混合策略
self.addEventListener('fetch', (event) => {
  const { request } = event
  const url = new URL(request.url)
  
  // 策略选择：
  if (isStaticAsset(url.pathname)) {
    // 静态资源：缓存优先
    event.respondWith(cacheFirst(request))
  } else if (url.pathname.startsWith('/api/notes')) {
    // 笔记 API：网络优先，缓存兜底
    event.respondWith(networkFirst(request))
  } else if (request.method === 'POST' && url.pathname === '/api/notes/sync') {
    // 同步请求：仅网络 + 后台队列
    event.respondWith(networkWithQueue(request))
  } else {
    // 其他请求：stale-while-revalidate
    event.respondWith(staleWhileRevalidate(request))
  }
})

// 缓存优先策略
async function cacheFirst(request) {
  const cached = await caches.match(request)
  if (cached) return cached
  
  try {
    const response = await fetch(request)
    if (response.ok) {
      const cache = await caches.open(STATIC_CACHE)
      cache.put(request, response.clone())
    }
    return response
  } catch (err) {
    return caches.match('/offline.html')
  }
}

// 网络优先策略
async function networkFirst(request) {
  try {
    const response = await fetch(request)
    if (response.ok) {
      const cache = await caches.open(CACHE_NAME)
      cache.put(request, response.clone())
    }
    return response
  } catch (err) {
    const cached = await caches.match(request)
    if (cached) return cached
    return new Response(
      JSON.stringify({ error: '离线模式，数据可能不是最新的' }),
      { status: 503, headers: { 'Content-Type': 'application/json' } }
    )
  }
}

// Stale While Revalidate
async function staleWhileRevalidate(request) {
  const cache = await caches.open(CACHE_NAME)
  const cached = await cache.match(request)
  
  const fetchPromise = fetch(request).then(response => {
    if (response.ok) {
      cache.put(request, response.clone())
    }
    return response
  }).catch(() => cached)
  
  return cached || fetchPromise
}

// 带后台队列的网络请求
async function networkWithQueue(request) {
  try {
    return await fetch(request)
  } catch (err) {
    // 离线时，将请求存入 IndexedDB 队列
    await saveToSyncQueue(request.clone())
    return new Response(
      JSON.stringify({ queued: true, message: '将在恢复连接后同步' }),
      { headers: { 'Content-Type': 'application/json' } }
    )
  }
}

// IndexedDB 同步队列
async function saveToSyncQueue(request) {
  const db = await openDB('sync-queue', 1, {
    upgrade(db) {
      db.createObjectStore('requests', { autoIncrement: true })
    }
  })
  
  const body = await request.text()
  const tx = db.transaction('requests', 'readwrite')
  tx.store.add({
    url: request.url,
    method: request.method,
    headers: [...request.headers.entries()],
    body
  })
  await tx.done
}

function isStaticAsset(path) {
  return /\.(js|css|png|jpg|svg|woff2?)$/.test(path)
}

// 后台同步事件（需要 `sync` 权限）
self.addEventListener('sync', (event) => {
  if (event.tag === 'sync-notes') {
    event.waitUntil(processSyncQueue())
  }
})

async function processSyncQueue() {
  const db = await openDB('sync-queue', 1)
  const tx = db.transaction('requests', 'readonly')
  const requests = await tx.store.getAll()
  
  for (const req of requests) {
    try {
      await fetch(req.url, {
        method: req.method,
        headers: new Headers(req.headers),
        body: req.body
      })
    } catch (err) {
      console.error('同步失败，下次再试:', req.url)
      return // 失败一个就停止，下次 sync 继续
    }
  }
  
  // 清空队列
  const clearTx = db.transaction('requests', 'readwrite')
  await clearTx.store.clear()
}
```

#### 3.2 场景 2：SW 实现消息推送（Web Push）

Service Worker 也可以接收服务器推送的消息：

```javascript
// sw.js — 推送事件
self.addEventListener('push', (event) => {
  console.log('收到推送消息:', event.data.text())
  
  let data
  try {
    data = event.data.json()
  } catch {
    data = { title: '新消息', body: event.data.text() }
  }
  
  const options = {
    body: data.body || '有新内容',
    icon: '/icons/icon-192.png',
    badge: '/icons/badge.png',
    vibrate: [200, 100, 200],
    data: {
      url: data.url || '/'
    },
    actions: [
      { action: 'view', title: '查看' },
      { action: 'close', title: '关闭' }
    ]
  }
  
  event.waitUntil(
    self.registration.showNotification(data.title, options)
  )
})

// 用户点击通知
self.addEventListener('notificationclick', (event) => {
  event.notification.close()
  
  if (event.action === 'close') return
  
  // 打开或聚焦到指定页面
  event.waitUntil(
    clients.matchAll({ type: 'window' }).then(windowClients => {
      const url = event.notification.data.url
      
      // 如果已有页面打开，聚焦它并导航到目标 URL
      for (const client of windowClients) {
        if (client.url === url && 'focus' in client) {
          return client.focus()
        }
      }
      
      // 否则打开新窗口
      if (clients.openWindow) {
        return clients.openWindow(url)
      }
    })
  )
})
```

**页面上请求推送权限**：

```javascript
// main.js
async function subscribeToPush() {
  if (!('Notification' in window)) {
    console.log('当前浏览器不支持推送通知')
    return
  }
  
  // 请求权限
  const permission = await Notification.requestPermission()
  if (permission !== 'granted') {
    console.log('用户拒绝了推送通知')
    return
  }
  
  // 获取推送订阅对象
  const registration = await navigator.serviceWorker.ready
  const subscription = await registration.pushManager.subscribe({
    userVisibleOnly: true, // 必须为 true，保证每条推送都显示通知
    applicationServerKey: urlBase64ToUint8Array('你的VAPID公钥')
  })
  
  console.log('推送订阅成功:', subscription)
  
  // 将 subscription 发送给后端
  await fetch('/api/push/subscribe', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(subscription)
  })
}
```

#### 3.3 场景 3：SW 配合 Workbox 使用

Workbox 是 Google 推出的 SW 工具库，封装了常用的缓存策略和工具：

```bash
npm install workbox-webpack-plugin --save-dev
```

```javascript
// webpack.config.js — 在构建时生成 SW
const { GenerateSW } = require('workbox-webpack-plugin')

module.exports = {
  plugins: [
    new GenerateSW({
      clientsClaim: true,        // 相当于 self.clients.claim()
      skipWaiting: true,          // 相当于 self.skipWaiting()
      
      // 预缓存：自动收集构建产物
      // 不需要手动写 PRECACHE_URLS
      
      // 运行时缓存策略
      runtimeCaching: [
        {
          // 图片资源：缓存优先
          urlPattern: /\.(png|jpg|jpeg|svg|gif)$/,
          handler: 'CacheFirst',
          options: {
            cacheName: 'images',
            expiration: {
              maxEntries: 50,     // 最多缓存 50 张
              maxAgeSeconds: 30 * 24 * 60 * 60, // 30 天
            },
          },
        },
        {
          // API 请求：网络优先
          urlPattern: /\/api\//,
          handler: 'NetworkFirst',
          options: {
            cacheName: 'api-cache',
            networkTimeoutSeconds: 3, // 网络超时 3 秒
            expiration: {
              maxEntries: 100,
              maxAgeSeconds: 5 * 60, // 5 分钟
            },
          },
        },
        {
          // 其他请求：Stale While Revalidate
          urlPattern: /.*/,
          handler: 'StaleWhileRevalidate',
          options: {
            cacheName: 'others',
            expiration: {
              maxEntries: 200,
            },
          },
        },
      ],
    }),
  ],
}
```

这样配置后，Webpack 构建时会自动生成 SW 脚本，并在 `dist/service-worker.js` 中注入预缓存和运行时策略。

**手动使用 Workbox 库**（非构建工具集成）：

```javascript
// sw.js — 使用 Workbox 库
importScripts('https://storage.googleapis.com/workbox-cdn/releases/7.0.0/workbox-sw.js')

if (workbox) {
  console.log('Workbox 加载成功')
  
  // 预缓存（需要配合 workbox-webpack-plugin 或 injectManifest）
  
  // 路由缓存策略
  workbox.routing.registerRoute(
    /\.(?:js|css|html)$/,
    new workbox.strategies.StaleWhileRevalidate({
      cacheName: 'static-resources',
    })
  )
  
  workbox.routing.registerRoute(
    /\.(?:png|jpg|jpeg|svg|gif|webp)$/,
    new workbox.strategies.CacheFirst({
      cacheName: 'image-cache',
      plugins: [
        new workbox.expiration.ExpirationPlugin({
          maxEntries: 60,
          maxAgeSeconds: 30 * 24 * 60 * 60, // 30 天
        }),
      ],
    })
  )
  
  workbox.routing.registerRoute(
    /\/api\//,
    new workbox.strategies.NetworkFirst({
      cacheName: 'api-cache',
      networkTimeoutSeconds: 3,
    })
  )
}
```

### 4. 常见误区 & 实际项目中的坑

#### 4.1 误区：SW 注册后立即生效

**错误理解**：注册 SW 后，当前页面立即被 SW 控制。

```javascript
navigator.serviceWorker.register('/sw.js')
// 认为从此页面由 SW 控制
```

**实际情况**：

```javascript
navigator.serviceWorker.register('/sw.js').then(registration => {
  console.log('SW 注册成功')
  
  // 当前页面是否已经被 SW 控制？
  console.log(navigator.serviceWorker.controller)
  // 第一次注册时 → null（当前页面未被控制）
  // 刷新页面后 → [ServiceWorker]（被控制）
  
  // SW 控制的是「注册之后打开的页面」，不包括当前页面
})
```

**原因**：SW 在 install 和 activate 完成后才接管页面。当前页面已经在没有 SW 的情况下加载完成，不会回退到 SW 获取资源。只有**下次导航**（刷新或新标签页）才会走 SW。

**正确做法**：如果希望当前页面也受 SW 控制，可以在 activate 中调用 `self.clients.claim()`，但当前页面的初始资源已经加载完了，后续的资源请求才会走 SW。

#### 4.2 误区：SW 脚本不能有任何缓存头

**错误理解**：SW 脚本必须设置 `Cache-Control: no-cache`，否则浏览器不会更新 SW。

**实际情况**：浏览器**始终**会检查 SW 脚本的更新，即使有 HTTP 缓存。但行为取决于 `Cache-Control` 头：

```javascript
// 浏览器检查 SW 更新的机制：
// 1. 导航到页面时，浏览器重新下载 SW 脚本
// 2. 使用 HTTP 缓存（如果可用）快速返回
// 3. 然后对比新旧脚本的字节内容
```

**`Cache-Control` 的影响**：

- 如果设置了 `Cache-Control: max-age=86400`，浏览器在 24 小时内不会真正下载新 SW 脚本，直接以缓存中的脚本为准。
- 但这不代表 SW 不会更新——浏览器每 24 小时会强制检查一次。

**推荐配置**：

```nginx
# Nginx 配置
location /sw.js {
  add_header Cache-Control "no-cache";
  # 或者设置很短的 max-age
  # add_header Cache-Control "public, max-age=300";
}
```

**但注意**：即使 `Cache-Control` 设置了 `no-cache`，SW 的更新也不是实时的。浏览器必须在**下一次导航**时才会检查更新。

#### 4.3 坑：Cache Storage 不是无限空间

每个浏览器对 Cache Storage 的配额不同，且会和其他存储（IndexedDB、localStorage）共享配额：

```javascript
// 检查缓存配额
if ('storage' in navigator && 'estimate' in navigator.storage) {
  navigator.storage.estimate().then(estimate => {
    console.log('已使用:', estimate.usage, 'bytes')
    console.log('配额:', estimate.quota, 'bytes')
    console.log('使用率:', (estimate.usage / estimate.quota * 100).toFixed(1) + '%')
  })
}
```

```javascript
// 估算各浏览器的 Cache Storage 配额：
// Chrome：可用磁盘空间的 60%（通常几百 MB 到几 GB）
// Firefox：可用磁盘空间的 50%（最多 2GB）
// Safari：< 1GB（移动端更小，约 50MB）
// iOS Safari：约 50MB（非常有限）
```

**缓存管理策略**：

```javascript
// 1. 限制每个缓存的条目数量
async function addToCacheWithLimit(request, response, maxEntries = 50) {
  const cache = await caches.open(CACHE_NAME)
  
  // 获取缓存中的键
  const keys = await cache.keys()
  
  // 如果超出限制，删除最旧的
  if (keys.length >= maxEntries) {
    await cache.delete(keys[0])
  }
  
  await cache.put(request, response)
}

// 2. 设置过期时间
async function isCacheFresh(request, maxAgeMs) {
  const response = await caches.match(request)
  if (!response) return false
  
  const cachedDate = new Date(response.headers.get('date') || 0).getTime()
  const now = Date.now()
  
  return (now - cachedDate) < maxAgeMs
}

// 3. 定期清理（在 activate 事件中）
const MAX_CACHE_AGE = 7 * 24 * 60 * 60 * 1000 // 7 天

async function cleanOldCache() {
  const cache = await caches.open(CACHE_NAME)
  const requests = await cache.keys()
  
  const now = Date.now()
  
  for (const request of requests) {
    const response = await cache.match(request)
    const cachedDate = new Date(response.headers.get('date') || now).getTime()
    
    if (now - cachedDate > MAX_CACHE_AGE) {
      await cache.delete(request)
    }
  }
}
```

#### 4.4 坑：请求被 SW 拦截后出现 CORS 问题

当 SW 拦截了一个跨域请求（如 CDN 资源），缓存和返回时需要处理 CORS：

```javascript
// 问题场景：CDN 资源
self.addEventListener('fetch', (event) => {
  if (event.request.url.includes('cdn.example.com')) {
    event.respondWith(
      fetch(event.request).then(response => {
        // response 可能是 opaque 响应
        // opaque 响应：status = 0，body 不可读
        // 不能调用 response.clone() 或 response.text()
        console.log(response.type) // 'opaque'
        
        // 尝试缓存这个响应会失败
        // caches.put(event.request, response.clone()) ❌ Error
        return response
      })
    )
  }
})
```

**Opaque 响应**：当请求设置了 `mode: 'no-cors'` 或资源服务器没有返回 CORS 头时，浏览器返回 opaque 响应。这个响应的 body 不可读，status 为 0。

**解决方案**：

```javascript
// 方案 1：确保资源服务器返回 CORS 头
// 服务器设置：Access-Control-Allow-Origin: *
// 客户端不需要特殊处理，响应 type 为 'cors'

// 方案 2：如果无法控制服务器，改用 no-cors 并缓存 opaque 响应
self.addEventListener('fetch', (event) => {
  if (event.request.url.includes('cdn.example.com')) {
    event.respondWith(
      caches.match(event.request).then(cached => {
        if (cached) return cached
        
        // 发起 no-cors 请求
        return fetch(event.request, { mode: 'no-cors' }).then(response => {
          // Opaque 响应可以缓存，但无法读取其内容
          // 缓存后下次从缓存读取时也是 opaque
          const cache = event.waitUntil(
            caches.open('cdn-cache').then(cache => cache.put(event.request, response))
          )
          
          // 注意：这里 response 已经被 consume 了
          // 需要 clone
          return response.clone()
        })
      })
    )
  }
})

// 方案 3：使用 CORS 代理
// 将请求转发到支持 CORS 的代理
```

**更安全的做法：只缓存可读的响应**：

```javascript
async function safeCache(request, response) {
  if (response.type === 'opaque') {
    // Opaque 响应：缓存但不 guarantee 可用性
    // 不 clone，直接 put
    const cache = await caches.open('opaque-cache')
    cache.put(request, response)
    return response
  }
  
  // 正常 CORS 或 same-origin 响应
  const cache = await caches.open(CACHE_NAME)
  cache.put(request, response.clone())
  return response
}
```

#### 4.5 坑：SW 中 IndexedDB 的操作需要处理版本升级

```javascript
// 在 SW 中操作 IndexedDB
// 问题：如果数据库版本升级，需要处理 onupgradeneeded

// 不好的写法
self.addEventListener('fetch', (event) => {
  // 每次 fetch 都尝试打开数据库
  const request = indexedDB.open('my-db', 1)
  // 没有处理 onupgradeneeded：如果数据库不存在，请求会失败
})

// 正确写法
function openDB() {
  return new Promise((resolve, reject) => {
    const request = indexedDB.open('my-db', 1)
    
    request.onupgradeneeded = (event) => {
      const db = event.target.result
      // 创建对象存储
      if (!db.objectStoreNames.contains('sync-queue')) {
        db.createObjectStore('sync-queue', { autoIncrement: true })
      }
    }
    
    request.onsuccess = (event) => resolve(event.target.result)
    request.onerror = (event) => reject(event.target.error)
  })
}
```

### 5. 与相关知识的关联 & 对比

#### 5.1 Service Worker vs Cache-Control（HTTP 缓存）

| 维度 | Service Worker | HTTP Cache-Control |
|:---|:---|:---|
| **控制粒度** | 请求级别，可编程（JS） | 资源级别，配置文件 |
| **离线支持** | ✅ 完整离线 | ❌ 只能降低请求频率 |
| **更新及时性** | 需处理 skipWaiting + claim | 自动 |
| **存储容量** | 大（几十 MB 到 GB） | 小（浏览器自动管理） |
| **缓存策略** | 完全可编程 | 固定模式 |
| **透明性** | 开发者完全控制 | 浏览器控制 |
| **复杂度** | 高 | 低 |

**两者关系**：互补，不是替代。

```javascript
// 建议的组合使用：
// 1. HTTP Cache-Control 处理「快速失效」的场景
//    Cache-Control: no-cache 或 max-age=0
//    确保浏览器每次都会验证资源是否新鲜

// 2. Service Worker 处理「离线」和「高级缓存策略」的场景
//    SW 可以选择从缓存读取，或发起网络请求
//    甚至可以忽略 Cache-Control，完全按自己的逻辑处理
```

#### 5.2 Service Worker vs Web Worker vs Shared Worker

| 维度 | Service Worker | Dedicated Worker | Shared Worker |
|:---|:---|:---|:---|
| **实例数** | 每个域名一个 | 每个页面多个 | 多个同源页面共享一个 |
| **生命周期** | 独立于页面 | 随页面关闭销毁 | 随最后一个引用页面关闭 |
| **拦截网络** | ✅ 可以 | ❌ 不能 | ❌ 不能 |
| **访问 DOM** | ❌ 不能 | ❌ 不能 | ❌ 不能 |
| **可用 API** | 只能异步 API | 大部分 API | 大部分 API |
| **使用场景** | 离线、缓存、推送 | 计算密集型任务 | 跨页面共享状态 |
| **持久化** | ✅ 可安装/激活 | ❌ 不持久 | ❌ 不持久 |

#### 5.3 Service Worker vs PWA

Service Worker 是 PWA 的核心技术之一，但不是全部。PWA 的三要素：

```
PWA = Web App Manifest（可安装性）
    + Service Worker（离线 + 推送）
    + 其他增强能力（IndexedDB、支付、蓝牙等）
```

```json
// manifest.json — 让网页可安装
{
  "name": "笔记应用",
  "short_name": "笔记",
  "start_url": "/",
  "display": "standalone",
  "background_color": "#ffffff",
  "theme_color": "#007AFF",
  "icons": [
    { "src": "/icons/icon-192.png", "sizes": "192x192", "type": "image/png" },
    { "src": "/icons/icon-512.png", "sizes": "512x512", "type": "image/png" }
  ]
}
```

```
PWA 的 3 个核心能力：
1. 离线可用（Service Worker + Cache Storage）
2. 可安装到桌面（manifest.json）
3. 消息推送（Service Worker push event + Web Push API）
```

### 6. 现代最佳实践（2025-2026）

#### 6.1 使用 Workbox 而非手写 SW

手写 SW 管理缓存、版本、更新策略容易出错。Workbox 封装了绝大多数最佳实践：

```bash
npm install workbox-webpack-plugin workbox-precaching workbox-routing workbox-strategies
```

```javascript
// webpack.config.js
const { InjectManifest } = require('workbox-webpack-plugin')

module.exports = {
  plugins: [
    new InjectManifest({
      swSrc: './src/sw.js',        // 你的 SW 源码
      swDest: 'service-worker.js', // 输出的 SW 文件
      // 自动注入预缓存清单
      maximumFileSizeToCacheInBytes: 5 * 1024 * 1024, // 5MB
    })
  ]
}
```

```javascript
// src/sw.js — 使用 Workbox 的 SW 模板
import { precacheAndRoute } from 'workbox-precaching'
import { registerRoute } from 'workbox-routing'
import { StaleWhileRevalidate, CacheFirst, NetworkFirst } from 'workbox-strategies'
import { ExpirationPlugin } from 'workbox-expiration'

// 使用构建工具注入的预缓存清单
precacheAndRoute(self.__WB_MANIFEST)

// 图片：缓存优先，限制数量
registerRoute(
  /\.(?:png|jpg|jpeg|svg|gif|webp)$/,
  new CacheFirst({
    cacheName: 'images',
    plugins: [
      new ExpirationPlugin({
        maxEntries: 60,
        maxAgeSeconds: 30 * 24 * 60 * 60,
      }),
    ],
  })
)

// API：网络优先，超时回退
registerRoute(
  /\/api\//,
  new NetworkFirst({
    cacheName: 'api',
    networkTimeoutSeconds: 3,
    plugins: [
      new ExpirationPlugin({
        maxEntries: 100,
        maxAgeSeconds: 5 * 60,
      }),
    ],
  })
)

// 静态资源：Stale While Revalidate
registerRoute(
  /\.(?:js|css|html)$/,
  new StaleWhileRevalidate({
    cacheName: 'static',
  })
)
```

**使用 Workbox 的好处**：
- 自动生成预缓存清单（不需要手动维护 PRECACHE_URLS）
- 内置过期策略、缓存大小限制
- 自动处理 SW 更新逻辑
- TypeScript 支持

#### 6.2 开发环境中的 SW 调试

```javascript
// 开发环境不注册 SW
if ('serviceWorker' in navigator && location.hostname !== 'localhost') {
  navigator.serviceWorker.register('/sw.js')
}

// 或在构建时通过环境变量控制
if ('serviceWorker' in navigator && process.env.NODE_ENV === 'production') {
  navigator.serviceWorker.register('/service-worker.js')
}
```

**浏览器 DevTools 调试**：

```
Chrome DevTools → Application → Service Workers
可以：
- 查看 SW 状态
- 更新/卸载 SW
- 模拟离线模式
- 查看缓存的 Cache Storage 内容
- 查看推送通知
```

#### 6.3 优雅的更新提示

```javascript
// main.js
class SWManager {
  constructor() {
    this.registration = null
    this.updateCallbacks = []
  }
  
  async init() {
    if (!('serviceWorker' in navigator)) return
    
    this.registration = await navigator.serviceWorker.register('/sw.js')
    
    this.registration.addEventListener('updatefound', () => {
      const newWorker = this.registration.installing
      
      newWorker.addEventListener('statechange', () => {
        if (newWorker.state === 'installed' && this.registration.active) {
          // 有新版本可用
          this.updateCallbacks.forEach(cb => cb())
        }
      })
    })
  }
  
  onUpdate(callback) {
    this.updateCallbacks.push(callback)
  }
  
  applyUpdate() {
    if (this.registration?.waiting) {
      this.registration.waiting.postMessage({ type: 'SKIP_WAITING' })
    }
  }
}

// 使用
const swManager = new SWManager()
swManager.init()

swManager.onUpdate(() => {
  // 显示一个 Toast 或 Banner
  const toast = document.createElement('div')
  toast.className = 'update-toast'
  toast.innerHTML = `
    <span>新版本可用</span>
    <button onclick="updateApp()">立即更新</button>
  `
  document.body.appendChild(toast)
})

// 全局函数供按钮调用
window.updateApp = () => {
  swManager.applyUpdate()
}

// 页面被新 SW 接管时刷新
navigator.serviceWorker.addEventListener('controllerchange', () => {
  window.location.reload()
})
```

#### 6.4 SW 的测试策略

```javascript
// 使用 Puppeteer 测试 SW 行为
// npm install -D puppeteer

const puppeteer = require('puppeteer')

async function testServiceWorker() {
  const browser = await puppeteer.launch()
  const page = await browser.newPage()
  
  // 等待 SW 注册
  await page.goto('http://localhost:3000')
  await page.evaluate(() => {
    return navigator.serviceWorker.ready
  })
  
  console.log('SW 已激活')
  
  // 测试离线
  await page.setOfflineMode(true)
  
  // 刷新页面（应该从缓存加载）
  await page.reload({ waitUntil: 'networkidle0' })
  
  const title = await page.title()
  console.log('离线时页面标题:', title)
  
  // 验证页面内容完整
  const content = await page.evaluate(() => document.body.textContent)
  console.log('离线时页面内容长度:', content.length)
  
  await browser.close()
}
```

#### 6.5 安全性考虑

```javascript
// 1. 始终验证请求来源
self.addEventListener('message', (event) => {
  // 只处理来自自己页面的消息
  if (event.origin !== location.origin) {
    console.warn('收到来自不可信来源的消息:', event.origin)
    return
  }
  
  // 处理消息...
})

// 2. 不要在 SW 中存储敏感信息
// Cache Storage 在不同浏览器中的安全级别不同
// 避免缓存包含 token 的请求响应

// 3. 使用 Cache Storage 的安全策略
async function cacheSafely(request, response) {
  // 不缓存包含认证信息的请求
  if (request.url.includes('/api/auth') || request.url.includes('/token')) {
    return response
  }
  
  // 不缓存的响应类型
  if (response.status === 401 || response.status === 403) {
    return response
  }
  
  const cache = await caches.open(CACHE_NAME)
  cache.put(request, response.clone())
  return response
}
```

### 7. 常见疑问解答（自问自答）

#### Q1: Service Worker 能缓存流媒体（视频/音频）吗？

可以，但要谨慎。大文件的缓存策略和普通资源不同：

```javascript
// 视频缓存策略：首次访问时流式传输，后台逐步缓存
self.addEventListener('fetch', (event) => {
  if (event.request.url.endsWith('.mp4')) {
    event.respondWith(
      // 使用 Range 请求支持分段缓存
      handleRangeRequest(event.request)
    )
  }
})

async function handleRangeRequest(request) {
  const cached = await caches.match(request)
  if (cached) return cached
  
  // 不缓存大视频，直接透传到网络
  return fetch(request)
}
```

**注意事项**：
- 视频文件通常很大，容易耗尽缓存配额
- 移动端缓存的视频不会被系统媒体扫描器识别（不会出现在相册中）
- 建议只缓存短视频（< 10MB），大视频流式播放即可

#### Q2: SW 在浏览器关闭后还能运行吗？

**不能**。SW 的生命周期规则：

- 浏览器关闭 → SW 被终止
- 浏览器重新打开 → SW 不会自动启动
- 用户导航到受 SW 控制的页面 → SW 启动
- SW 收到推送事件 → SW 启动（即使浏览器关闭，但需要浏览器在后台运行）

所以不能依赖 SW 做「浏览器关闭后还在后台运行的任务」。如果需要这样的能力，使用：
- **Background Sync**：浏览器恢复网络时自动同步数据
- **Periodic Background Sync**：定期在后台同步（需要权限）
- **Push Events**：服务器推送唤醒 SW

#### Q3: 多个页面共用一个 SW 会有什么问题？

```javascript
// 问题：多个页面共享同一个 SW 的 fetch handler
// 页面 A 正在离线查看缓存
// 页面 B 需要网络最新数据
// SW 无法区分两个页面的不同需求

// 解决方案：通过请求头传递页面意图
// 页面 B 在获取 API 数据时添加标记
fetch('/api/data', {
  headers: {
    'X-Cache-Strategy': 'network-only'
  }
})

// SW 根据请求头选择策略
self.addEventListener('fetch', (event) => {
  if (event.request.headers.get('X-Cache-Strategy') === 'network-only') {
    event.respondWith(fetch(event.request))
    return
  }
  
  // 默认策略
  event.respondWith(staleWhileRevalidate(event.request))
})
```

**实践建议**：SW 的粒度通常是整个站点一个。如果应用的不同部分需要不同的缓存策略，通过 URL 模式或请求头区分。

#### Q4: SW 注册后，怎么确认它是否正常工作？

```javascript
// 方法 1：检查 controller
if (navigator.serviceWorker.controller) {
  console.log('当前页面由 SW 控制')
} else {
  console.log('当前页面未被 SW 控制')
}

// 方法 2：获取注册信息
navigator.serviceWorker.getRegistration().then(registration => {
  console.log('SW 状态:', registration.active?.state) // 'activated'
  console.log('SW scope:', registration.scope)
  console.log('SW 脚本:', registration.active?.scriptURL)
})

// 方法 3：Chrome DevTools
// Application → Service Workers → 查看当前 SW 状态

// 方法 4：在 SW 中记录日志
self.addEventListener('fetch', (event) => {
  console.log('[SW] 拦截请求:', event.request.url)
  event.respondWith(fetch(event.request))
})
```

#### Q5: 同一域名下可以注册多个 SW 吗？

可以，但**每个 scope 只能有一个 SW**。scope 由注册时的路径或选项决定：

```javascript
// 注册两个 scope 不同的 SW
navigator.serviceWorker.register('/sw-dashboard.js', { scope: '/dashboard/' })
navigator.serviceWorker.register('/sw-admin.js', { scope: '/admin/' })

// /dashboard/ 路径下的页面由 sw-dashboard.js 控制
// /admin/ 路径下的页面由 sw-admin.js 控制
// 其他路径可能由根 scope 的 SW 控制
```

**注意**：scope 必须有包含关系，不能交叉重叠。如果两个 SW 的 scope 重叠，注册会失败。

实际项目中**绝大多数情况只需要一个 SW**（scope: '/'）。多个 SW 会增加维护复杂度，且容易产生冲突。

#### Q6: 如何优雅地卸载 Service Worker？

```javascript
// 方法 1：调用 unregister
if ('serviceWorker' in navigator) {
  navigator.serviceWorker.getRegistration().then(registration => {
    if (registration) {
      registration.unregister().then(success => {
        if (success) {
          console.log('SW 已卸载')
        }
      })
    }
  })
}

// 方法 2：清空缓存并卸载
async function unregisterSW() {
  // 清空所有缓存
  const keys = await caches.keys()
  await Promise.all(keys.map(key => caches.delete(key)))
  
  // 卸载 SW
  const registration = await navigator.serviceWorker.getRegistration()
  if (registration) {
    await registration.unregister()
  }
  
  // 刷新页面，移除 SW 控制
  window.location.reload()
}
```

### 8. 推荐学习路径

```
1. 阅读本文【面试速答版】 → 建立整体认知
2. 在 Chrome DevTools 的 Application 面板中熟悉 SW 相关功能
3. 写一个最简单的 SW：只输出日志，不缓存任何东西
4. 实现 Cache First 策略缓存静态资源
5. 在 DevTools 中模拟离线，验证缓存是否生效
6. 实现 Network First 策略缓存 API 请求
7. 学习 SW 的更新机制：修改 SW 脚本，观察更新流程
8. 引入 Workbox 替代手写 SW
9. 学习 Background Sync（后台同步）
10. 学习 Web Push 通知
```

**关联知识点索引**：
- PWA 中 SW 的角色 → [PWA 详解](./PWA%20详解.md)
- 浏览器的 HTTP 缓存机制 → [浏览器原理详解](./浏览器原理详解.md)
- IndexedDB 存储 → [前端性能优化](../性能优化/前端性能优化.md)
- Web Worker 多线程 → JavaScript 目录下相关文档
