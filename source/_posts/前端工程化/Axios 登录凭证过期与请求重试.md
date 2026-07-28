---
title: Axios 登录凭证过期与请求重试
categories: 
- 前端工程化
tags:
- Axios
- Token
- 鉴权
- HTTP
- 拦截器
---

## 一、问题描述

用户在一个页面上同时发起了多个 API 请求，此时登录凭证（Token）刚好过期。这会导致：<!--more-->

```text
页面加载
  │
  ├── GET /api/user/info          → 401（Token 过期）
  ├── GET /api/user/orders        → 401（Token 过期）
  └── GET /api/user/notifications → 401（Token 过期）
  │
  └── 三个请求各自尝试刷新 Token → 并发刷新 3 次 → 浪费 + 竞态
```

理想处理：

```text
页面加载
  │
  ├── GET /api/user/info          → 401（Token 过期）
  ├── GET /api/user/orders        → 401（Token 过期）
  └── GET /api/user/notifications → 401（Token 过期）
  │
  ├── 统一拦截：只发起 1 次刷新 Token 请求
  ├── 其余请求排队等待
  │
  ├── 刷新成功 → 三个请求全部重放
  └── 刷新失败 → 跳转登录页
```

## 二、核心实现

### 1. 响应拦截器 + 刷新队列

```ts
import axios, { AxiosError, InternalAxiosRequestConfig } from 'axios';

const api = axios.create({
  baseURL: '/api',
  timeout: 10000,
});

// 刷新 Token 相关的状态
let isRefreshing = false;          // 是否正在刷新
let pendingQueue: PendingTask[] = []; // 等待重试的请求队列

interface PendingTask {
  resolve: (value: any) => void;
  reject: (reason: any) => void;
  config: InternalAxiosRequestConfig;
}
```

### 2. 请求拦截器——注入 Token

```ts
api.interceptors.request.use((config) => {
  const token = localStorage.getItem('token');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});
```

### 3. 响应拦截器——处理 401

```ts
api.interceptors.response.use(
  (response) => response,
  async (error: AxiosError) => {
    const { config, response } = error;

    // 非 401 或没有 config（网络错误等）直接抛出
    if (!config || response?.status !== 401) {
      return Promise.reject(error);
    }

    // 401 但已经是刷新 Token 的请求本身 → 不再重试，直接跳转登录
    if (config.url?.includes('/auth/refresh')) {
      redirectToLogin();
      return Promise.reject(error);
    }

    // 正在刷新中 → 将当前请求加入队列
    if (isRefreshing) {
      return new Promise((resolve, reject) => {
        pendingQueue.push({ resolve, reject, config });
      });
    }

    // 开始刷新 Token
    isRefreshing = true;

    try {
      const newToken = await refreshToken();

      // 更新本地 Token
      localStorage.setItem('token', newToken);
      config.headers.Authorization = `Bearer ${newToken}`;

      // 重放当前请求
      const result = await api(config);

      // 重放队列中的请求
      pendingQueue.forEach((task) => {
        task.config.headers.Authorization = `Bearer ${newToken}`;
        api(task.config).then(task.resolve).catch(task.reject);
      });
      pendingQueue = [];

      return result;
    } catch (refreshError) {
      // 刷新也失败 → 清空队列，跳转登录
      pendingQueue.forEach((task) => task.reject(refreshError));
      pendingQueue = [];
      redirectToLogin();
      return Promise.reject(refreshError);
    } finally {
      isRefreshing = false;
    }
  }
);
```

### 4. 刷新 Token 函数

```ts
async function refreshToken(): Promise<string> {
  const refreshToken = localStorage.getItem('refreshToken');
  if (!refreshToken) {
    throw new Error('No refresh token available');
  }

  const res = await axios.post('/api/auth/refresh', {
    refreshToken,
  });

  const { token, refreshToken: newRefreshToken } = res.data;

  // 更新 refreshToken
  localStorage.setItem('refreshToken', newRefreshToken);

  return token;
}
```

### 5. 跳转登录

```ts
function redirectToLogin() {
  // 清除本地凭证
  localStorage.removeItem('token');
  localStorage.removeItem('refreshToken');

  // 记录当前页面地址，登录后跳回
  const currentPath = window.location.pathname + window.location.search;
  window.location.href = `/login?redirect=${encodeURIComponent(currentPath)}`;
}
```

## 三、完整实现

```ts
// utils/http.ts
import axios, { AxiosError, InternalAxiosRequestConfig } from 'axios';

const api = axios.create({
  baseURL: import.meta.env.VITE_API_BASE,
  timeout: 15000,
});

let isRefreshing = false;
let pendingQueue: Array<{
  resolve: (value: any) => void;
  reject: (reason: any) => void;
  config: InternalAxiosRequestConfig;
}> = [];

function getToken(): string | null {
  return localStorage.getItem('token');
}

function setToken(token: string, refreshToken: string) {
  localStorage.setItem('token', token);
  localStorage.setItem('refreshToken', refreshToken);
}

function clearAuth() {
  localStorage.removeItem('token');
  localStorage.removeItem('refreshToken');
}

function redirectToLogin() {
  clearAuth();
  const currentPath = window.location.pathname + window.location.search;
  window.location.href = `/login?redirect=${encodeURIComponent(currentPath)}`;
}

async function refreshTokenRequest(): Promise<{ token: string; refreshToken: string }> {
  const rt = localStorage.getItem('refreshToken');
  if (!rt) throw new Error('No refresh token');

  const res = await axios.post(`${import.meta.env.VITE_API_BASE}/auth/refresh`, {
    refreshToken: rt,
  });
  return res.data;
}

// 请求拦截器
api.interceptors.request.use((config) => {
  const token = getToken();
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

// 响应拦截器
api.interceptors.response.use(
  (response) => response,
  async (error: AxiosError) => {
    const originalRequest = error.config;

    if (!originalRequest || error.response?.status !== 401) {
      return Promise.reject(error);
    }

    // 刷新接口本身 401 → 直接跳登录
    if (originalRequest.url?.includes('/auth/refresh')) {
      redirectToLogin();
      return Promise.reject(error);
    }

    // 已有刷新请求进行中 → 排队
    if (isRefreshing) {
      return new Promise((resolve, reject) => {
        pendingQueue.push({ resolve, reject, config: originalRequest });
      });
    }

    isRefreshing = true;

    try {
      const { token, refreshToken } = await refreshTokenRequest();
      setToken(token, refreshToken);

      // 重放当前请求
      originalRequest.headers.Authorization = `Bearer ${token}`;
      const result = await api(originalRequest);

      // 重放队列中的请求
      pendingQueue.forEach((task) => {
        task.config.headers.Authorization = `Bearer ${token}`;
        api(task.config).then(task.resolve).catch(task.reject);
      });
      pendingQueue = [];

      return result;
    } catch (refreshError) {
      pendingQueue.forEach((task) => task.reject(refreshError));
      pendingQueue = [];
      redirectToLogin();
      return Promise.reject(refreshError);
    } finally {
      isRefreshing = false;
    }
  }
);

export default api;
```

## 四、扩展场景

### 场景一：Token 提前刷新（非 401 触发）

401 触发刷新是在请求失败后补救。更主动的做法是在 Token 过期前就刷新：

```ts
// 在请求拦截器中检查 Token 是否即将过期
function isTokenExpiring(): boolean {
  const token = getToken();
  if (!token) return false;

  try {
    const payload = JSON.parse(atob(token.split('.')[1]));
    // 过期前 5 分钟就刷新
    return payload.exp * 1000 - Date.now() < 5 * 60 * 1000;
  } catch {
    return false;
  }
}

api.interceptors.request.use(async (config) => {
  const token = getToken();

  if (token && isTokenExpiring() && !config.url?.includes('/auth/refresh')) {
    try {
      const { token: newToken, refreshToken } = await refreshTokenRequest();
      setToken(newToken, refreshToken);
      config.headers.Authorization = `Bearer ${newToken}`;
    } catch {
      // 静默失败，继续用旧 Token，等 401 时再处理
    }
  } else if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }

  return config;
});
```

### 场景二：刷新 Token 并发限制（使用锁 Token）

极端情况下，用户同时在两个标签页操作，两个标签页同时检测 Token 过期，同时发起刷新：

```ts
// 使用 localStorage 作为锁，只有一个标签页能触发刷新
function acquireRefreshLock(): boolean {
  const lockKey = 'token:refresh:lock';
  const lockValue = Date.now().toString();
  localStorage.setItem(lockKey, lockValue);
  // 短暂延迟后检查是否仍是自己的锁
  return localStorage.getItem(lockKey) === lockValue;
}

async function refreshTokenWithLock(): Promise<{ token: string; refreshToken: string }> {
  const lockKey = 'token:refresh:lock';

  if (!acquireRefreshLock()) {
    // 另一个标签页在刷新，等待它完成
    return new Promise((resolve) => {
      const check = setInterval(() => {
        if (getToken() && !localStorage.getItem(lockKey)) {
          clearInterval(check);
          resolve({ token: getToken()!, refreshToken: localStorage.getItem('refreshToken')! });
        }
      }, 100);
    });
  }

  try {
    const result = await refreshTokenRequest();
    return result;
  } finally {
    localStorage.removeItem(lockKey);
  }
}
```

### 场景三：静默刷新（无感续期）

部分后端设计为每次请求都返回新 Token（通过响应头），前端拦截器统一更新：

```ts
api.interceptors.response.use((response) => {
  const newToken = response.headers['x-new-token'];
  const newRefreshToken = response.headers['x-new-refresh-token'];

  if (newToken) {
    localStorage.setItem('token', newToken);
  }
  if (newRefreshToken) {
    localStorage.setItem('refreshToken', newRefreshToken);
  }

  return response;
});
```

### 场景四：请求失败后的业务降级

对于非关键请求，401 失败后直接降级（不排队刷新也不跳转登录）：

```ts
// 为非关键请求添加标记
interface ExtendedConfig extends InternalAxiosRequestConfig {
  _silent401?: boolean;
}

// 使用
api.get('/api/analytics', { _silent401: true } as any);

// 拦截器中判断
if ((originalRequest as ExtendedConfig)._silent401) {
  return Promise.resolve({ data: null });  // 直接返回空数据
}
```

## 五、方案对比

| 方案 | 优点 | 缺点 | 适用场景 |
|------|------|------|----------|
| **401 触发刷新 + 队列重试** | 实现简单，覆盖所有请求 | 每次失败都要等网络往返 | 通用方案，推荐 |
| **请求前检查 Token 过期** | 提前刷新，避免 401 失败 | 多了一次 Token 解码 | 对失败容忍度低的场景 |
| **响应头静默续期** | 完全无感，不需要额外请求 | 需要后端配合 | 后端可控的场景 |
| **多标签页锁 Token** | 避免并发刷新 | 实现复杂，引入了分布式锁 | 多标签页同时操作的 SaaS |

## 六、常见问题

### Q1: 为什么刷新 Token 要用独立的 axios 实例

需要在响应拦截器中调用刷新接口，如果用同一个 `api` 实例，刷新请求也会走响应拦截器，形成死循环：

```ts
// ❌ 死循环：刷新接口 401 → 进入拦截器 → 又触发刷新 → ...
const newToken = await api.post('/auth/refresh', { ... });

// ✅ 用独立的 axios 实例（不走拦截器）
import axios from 'axios';
const newToken = await axios.post('/api/auth/refresh', { ... });
```

### Q2: 刷新 Token 本身也 401 了怎么办

```text
说明 refreshToken 也已过期或无效。
此时继续重试已经没有意义，应：
1. 清除本地所有凭证
2. 跳转到登录页（带上当前页面地址，方便登录后跳回）
3. 如有第三方登录，自动触发 OAuth 续期
```

### Q3: 队列中的请求是否需要超时

建议给队列增加超时机制，防止某些极端情况导致的死等：

```ts
function enqueueWithTimeout(task: PendingTask, timeoutMs = 10000) {
  const timer = setTimeout(() => {
    task.reject(new Error('Refresh queue timeout'));
  }, timeoutMs);

  const originalResolve = task.resolve;
  task.resolve = (value) => {
    clearTimeout(timer);
    originalResolve(value);
  };
}
```
