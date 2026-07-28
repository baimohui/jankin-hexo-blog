---
title: mitt——轻量级事件发射器
categories: 
- 框架和类库
tags:
- mitt
- EventEmitter
- 事件总线
- 发布订阅
---

## 一、mitt 是什么

mitt 是一个**只有 200 字节**的轻量级事件发射器（Event Emitter），由 Vue 作者尤雨溪编写。它提供了 `on`、`off`、`emit` 三个核心方法，用于实现模块间的解耦通信。<!--more-->

```bash
npm install mitt
```

```js
import mitt from 'mitt';
const emitter = mitt();

// 订阅事件
emitter.on('some-event', (payload) => { console.log(payload); });

// 触发事件
emitter.emit('some-event', { data: 'hello' });

// 取消订阅
emitter.off('some-event', handler);
```

### 与其他事件库对比

| 库 | 体积 | 特性 | 适用场景 |
|----|------|------|----------|
| mitt | **200B** | on/off/emit，无依赖 | 轻量事件总线 |
| Node.js EventEmitter | ~8KB | 完整 API，支持 `once`/`listeners` | 服务端 |
| EventEmitter3 | ~3KB | 性能优化 | 对性能敏感的 Node 场景 |
| Vue 2 EventBus | 需引入 Vue | 依赖 Vue 实例 | 旧 Vue 项目 |

## 二、基本使用

### 2.1 创建与销毁

```js
import mitt from 'mitt';

// 创建一个 emitter 实例
const emitter = mitt();

// 不需要时清除所有事件（组件卸载时调用）
emitter.all.clear();
```

### 2.2 订阅事件

```js
// 带类型定义
type Events = {
  foo: string;
  bar?: number;
  baz: void;  // 无 payload
};

const emitter = mitt<Events>();

// 监听 foo 事件
emitter.on('foo', (payload) => {
  console.log(payload);  // type: string
});

// 通配符 `*`：监听所有事件
emitter.on('*', (type, payload) => {
  console.log(`事件 ${type} 触发，数据：`, payload);
});
```

### 2.3 触发事件

```js
emitter.emit('foo', 'hello');     // 触发 foo 事件，传字符串
emitter.emit('bar', 42);          // 触发 bar 事件，传数字
emitter.emit('baz');              // 触发 baz 事件，无参数
```

### 2.4 取消订阅

```js
const handler = (payload: string) => { console.log(payload); };
emitter.on('foo', handler);

// 取消特定处理函数
emitter.off('foo', handler);

// 取消 foo 事件的所有处理函数
emitter.off('foo');

// 清除所有事件
emitter.all.clear();
```

## 三、在框架中使用

### React 中的全局事件

```tsx
// utils/eventBus.ts
import mitt from 'mitt';
type AppEvents = {
  userLogin: { id: number; name: string };
  userLogout: void;
  cartUpdate: { count: number };
  notification: { message: string; type: 'success' | 'error' };
};
export const eventBus = mitt<AppEvents>();
```

```tsx
// ComponentA.tsx（订阅）
import { useEffect } from 'react';
import { eventBus } from './utils/eventBus';

function NotificationBar() {
  useEffect(() => {
    const handler = (data: { message: string }) => {
      alert(data.message);
    };
    eventBus.on('notification', handler);
    return () => eventBus.off('notification', handler);  // 清理
  }, []);

  return null;
}
```

```tsx
// ComponentB.tsx（触发）
import { eventBus } from './utils/eventBus';

function Header() {
  const onLogin = () => {
    eventBus.emit('notification', {
      message: '欢迎回来！',
      type: 'success',
    });
  };
  return <button onClick={onLogin}>登录</button>;
}
```

### Vue 3 中的事件总线

Vue 3 移除了 `$on`/`$off`/`$emit`（Vue 2 的 EventBus），mitt 是官方推荐的替代方案：

```ts
// utils/eventBus.ts
import mitt from 'mitt';
export const emitter = mitt();
```

```vue
<!-- ComponentA.vue -->
<script setup lang="ts">
import { onMounted, onUnmounted } from 'vue';
import { emitter } from './utils/eventBus';

onMounted(() => {
  emitter.on('refresh', handleRefresh);
});

onUnmounted(() => {
  emitter.off('refresh', handleRefresh);
});

function handleRefresh(data: { id: number }) {
  console.log('刷新：', data.id);
}
</script>
```

```vue
<!-- ComponentB.vue -->
<script setup lang="ts">
import { emitter } from './utils/eventBus';

function refreshList() {
  emitter.emit('refresh', { id: 1 });
}
</script>
```

### 跨组件/跨模块通信场景

```text
├── 兄弟组件通信（不经过父组件）
├── 路由切换时触发数据刷新
├── 用户登录/登出通知多个模块
├── WebSocket 消息分发到不同组件
├── 模态框 / Toast 全局控制
└── 非父子组件间的状态同步
```

## 四、mitt 的源码分析

mitt 的源码只有约 30 行，核心逻辑非常精巧：

```ts
type EventHandler<Events, K extends keyof Events>
  = (payload: Events[K]) => void;

type WildcardHandler<Events>
  = (type: keyof Events, payload: Events[keyof Events]) => void;

type Handler<Events, K extends keyof Events>
  = EventHandler<Events, K> | WildcardHandler<Events>;

export default function mitt<Events extends Record<string, unknown>>(
  all?: Map<keyof Events, Handler<Events, any>[]>
) {
  // 用 Map 存储事件名 → 处理函数数组的映射
  all = all || new Map();

  return {
    // 订阅
    on<K extends keyof Events>(type: K, handler: Handler<Events, K>) {
      const handlers = all!.get(type);
      if (handlers) {
        handlers.push(handler as any);
      } else {
        all!.set(type, [handler as any]);
      }
    },

    // 取消订阅
    off<K extends keyof Events>(type: K, handler?: Handler<Events, K>) {
      const handlers = all!.get(type);
      if (!handlers) return;

      if (handler) {
        // 移除指定处理函数
        handlers.splice(handlers.indexOf(handler) >>> 0, 1);
      } else {
        // 移除该类型所有处理函数
        all!.set(type, []);
      }
    },

    // 触发
    emit<K extends keyof Events>(type: K, evt?: Events[K]) {
      // 调用该类型的处理函数
      (all!.get(type) || []).slice().forEach((handler: any) => {
        handler(evt);
      });
      // 调用通配符处理函数
      (all!.get('*') as any)?.slice().forEach((handler: any) => {
        handler(type, evt);
      });
    },

    // 所有事件（可用于外部访问或清理）
    get all() { return all; },
  };
}
```

### 设计亮点

```text
├── 用 Map 而非对象存储 → 性能更好、key 不限字符串
├── handlers.slice() 再遍历 → 防止遍历期间修改数组导致错乱
├── handlers.indexOf(handler) >>> 0 → 找不到返回 -1 → >>> 0 转为 0xFFFFFFFF
│   相当于 splice(4294967295, 1)，不会越界报错
├── TypeScript 类型安全 → 事件名和 payload 的类型对应
└── 支持通配符 * → 一次性监听所有事件（调试/日志）
```

## 五、实际场景

### 场景一：WebSocket 消息分发

```ts
// websocket.ts
import mitt from 'mitt';

type WSEvents = {
  message: { type: string; data: any };
  connected: void;
  disconnected: { code: number };
  error: { message: string };
};

export const wsEvents = mitt<WSEvents>();

let socket: WebSocket;

export function connect(url: string) {
  socket = new WebSocket(url);

  socket.onopen = () => wsEvents.emit('connected');
  socket.onclose = (e) => wsEvents.emit('disconnected', { code: e.code });
  socket.onerror = () => wsEvents.emit('error', { message: '连接异常' });
  socket.onmessage = (e) => {
    const data = JSON.parse(e.data);
    wsEvents.emit('message', { type: data.type, data });
  };
}
```

```tsx
// ChatPanel.tsx
import { useEffect } from 'react';
import { wsEvents, connect } from './websocket';

function ChatPanel() {
  useEffect(() => {
    connect('wss://example.com/ws');

    wsEvents.on('message', (msg) => {
      if (msg.type === 'chat') handleChat(msg.data);
    });

    wsEvents.on('disconnected', () => showReconnect());

    return () => {
      // 清理事件绑定
      wsEvents.all.clear();
    };
  }, []);

  return <div>...</div>;
}
```

### 场景二：表单跨步骤通信

```ts
// 多步骤表单中，不同步骤组件通过 mitt 通信
type FormEvents = {
  'step:validated': { step: number; isValid: boolean; data: any };
  'step:next': void;
  'step:prev': void;
  'form:submit': void;
  'form:saved': { draftId: string };
};

export const formBus = mitt<FormEvents>();
```

### 场景三：Toast 全局控制

```tsx
// ToastContainer.tsx
import { useEffect, useState } from 'react';
import { eventBus } from './eventBus';

function ToastContainer() {
  const [toasts, setToasts] = useState<{ id: number; msg: string }[]>([]);

  useEffect(() => {
    let id = 0;
    eventBus.on('notification', (data) => {
      const toastId = ++id;
      setToasts(prev => [...prev, { id: toastId, msg: data.message }]);
      setTimeout(() => {
        setToasts(prev => prev.filter(t => t.id !== toastId));
      }, 3000);
    });
  }, []);

  return (
    <div className="toast-container">
      {toasts.map(t => <div key={t.id}>{t.msg}</div>)}
    </div>
  );
}
```

## 六、与 Vue 2 EventBus 的对比

```text
Vue 2 模式：
  // main.js
  Vue.prototype.$bus = new Vue();
  // 组件中
  this.$bus.$on('event', handler);
  this.$bus.$emit('event', data);
  this.$bus.$off('event');

  // 问题：
  // 1. 依赖 Vue 实例，额外消耗内存
  // 2. Vue 3 已移除 $on/$off
  // 3. 类型支持不完善

mitt 模式：
  import mitt from 'mitt';
  const bus = mitt();
  bus.on('event', handler);
  bus.emit('event', data);
  bus.off('event');

  // 优势：
  // 1. 200 字节，无依赖
  // 2. 框架无关（React/Vue/Vanilla）
  // 3. 完美的 TypeScript 类型推断
  // 4. 通配符支持
```

## 七、注意事项

```text
├── 避免事件名冲突
│   项目较大时建议统一管理事件名，用命名空间：
│   'user:login'、'cart:update'、'order:paid'

├── 组件卸载时务必清理
│   不清理会导致内存泄漏和重复执行：
│   useEffect(() => {
│     bus.on('event', handler);
│     return () => bus.off('event', handler);
│   }, []);

├── 不要滥用事件总线
│   父子组件通信优先用 props（显式、可追溯）。
│   事件总线适合：跨层级无关组件、跨模块通信。

└── 调试技巧
    // 开发环境打印所有事件
    if (import.meta.env.DEV) {
      bus.on('*', (type, payload) => {
        console.log(`[Event] ${type}`, payload);
      });
    }
```

## 八、面试题

### Q1: mitt 的实现原理

```text
核心是用 Map 存储事件名与处理函数数组的映射。
on：向 Map[key] 数组中 push handler
emit：遍历 Map[key] 数组，依次调用 handler
off：从 Map[key] 数组中 splice 移除 handler

额外支持：
  - 通配符 *：emit 触发时额外调用通配符 handler，传 (type, payload)
  - type-safe：通过泛型约束事件名与 payload 的类型对应
```

### Q2: mitt 和 EventEmitter 的区别

```text
mitt：200B，只有 on/off/emit，不支持 once/listeners/removeAllListeners
EventEmitter：~8KB，完整的发布订阅 API

mitt 的优势正是它的"少"：
  - 体积小（200B vs 8KB）
  - 框架无关
  - TypeScript 类型完美
  - 源码 30 行，无黑盒
```

### Q3: 什么时候用 mitt 什么时候用 Redux/Pinia

```text
mitt：非父子组件通信、全局事件、跨模块消息。
  无需全局状态管理时使用，简单轻量。

Redux/Pinia：跨组件共享的状态。
  多个组件读写同一份数据时使用，数据流可追踪。

两者互补：mitt 传递事件消息，状态管理库管理应用状态。
```
