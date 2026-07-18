---
title: RESTful API 设计与鉴权机制
categories: 
- 后端
tags:
- API
- RESTful
- JWT
- OAuth
- 鉴权
---

## RESTful API 设计规范（前端视角）

<!-- more -->
RESTful 是前后端约定接口的最主流方式。作为前端，理解它比后端更重要——因为你是 API 的调用方。

### 核心概念

```text
资源（Resource）：用户、文章、订单 ...
URL 表示资源：/api/users、/api/articles
HTTP 动词表示操作：
  GET    /api/users       → 获取用户列表
  GET    /api/users/:id   → 获取单个用户
  POST   /api/users       → 新增用户
  PUT    /api/users/:id   → 全量更新用户
  PATCH  /api/users/:id   → 部分更新用户
  DELETE /api/users/:id   → 删除用户
```

### 前端友好的 API 设计要点

```javascript
// 好的 API：
GET /api/articles?page=1&pageSize=20
→ { "data": [...], "total": 100, "page": 1, "pageSize": 20 }

// 好的错误响应：
POST /api/users  → 400
{ "code": "VALIDATION_ERROR", "message": "邮箱格式不正确", "field": "email" }
```

### HTTP 状态码速查（前端最常遇到的）

| 状态码 | 含义 | 前端处理 |
|---|---|---|
| 200 | 成功 | 正常渲染 |
| 201 | 创建成功 | POST 请求成功 |
| 204 | 无内容 | DELETE 成功，无响应体 |
| 301/302 | 重定向 | 浏览器自动跳转或手动处理 |
| 400 | 参数错误 | 提示用户检查输入 |
| 401 | 未登录/Token 过期 | 跳转登录页 + 刷新 Token |
| 403 | 无权限 | 提示无权访问 |
| 404 | 资源不存在 | 提示 404 或展示空状态 |
| 409 | 冲突 | 如重复提交、版本冲突 |
| 422 | 参数校验失败 | 后端返回具体字段错误 |
| 429 | 请求太频繁 | 限流，等待后重试 |
| 500 | 服务器错误 | 友好提示 + 错误上报 |

## 常见鉴权方式

作为前端，你每天都在和鉴权打交道——登录、Token 过期、刷新 Token。

### Session-Cookie

```text
用户登录 → 服务器创建 Session → 返回 Cookie（含 sessionId）
  ↓
后续请求 → 浏览器自动带上 Cookie
  ↓
服务器对比 Session → 确认身份
```

**前端视角：** Cookie 是浏览器自动带的，前端不需要手动处理。但跨域时 Cookie 需要 `credentials: 'include'`。

### JWT（JSON Web Token）— 目前最主流

```text
用户登录 → 服务器生成 JWT（签名加密）→ 返回给前端
  ↓
前端存到 localStorage / 请求头 Authorization: Bearer <token>
  ↓
后续请求 → 前端手动带 Token → 服务器验证签名
```

**JWT 的结构：**
```text
header.payload.signature
↓          ↓       ↓
算法类型   用户信息    签名（防篡改）
{"alg":"HS256","typ":"JWT"}.{"id":1,"role":"admin","exp":1700000000}.xxxxx...
```

**前端注意事项：**

```javascript
// 登录后存 Token
const login = async (username, password) => {
  const { token, refreshToken } = await api.login({ username, password })
  localStorage.setItem('token', token)
  localStorage.setItem('refreshToken', refreshToken)
}

// Axios 拦截器统一带 Token 并处理过期
api.interceptors.request.use(config => {
  config.headers.Authorization = `Bearer ${localStorage.getItem('token')}`
  return config
})

api.interceptors.response.use(
  response => response,
  error => {
    if (error.response?.status === 401) {
      // Token 过期 → 用 refreshToken 刷新
      return refreshTokenAPI().then(({ token }) => {
        localStorage.setItem('token', token)
        error.config.headers.Authorization = `Bearer ${token}`
        return api(error.config) // 重发原请求
      }).catch(() => {
        // 刷新也失败 → 跳登录
        window.location.href = '/login'
      })
    }
    return Promise.reject(error)
  }
)
```

### OAuth 2.0（第三方登录）

```text
用户点击"微信登录"
  ↓
跳转微信授权页 → 用户确认
  ↓
回调到你的服务器 → 服务器拿 code 换 token
  ↓
服务器用 token 获取用户信息 → 创建/匹配账号 → 返回 JWT
```

**前端视角：** OAuth 流程中前端主要负责跳转和接收回调（处理 redirect_uri）。

## 鉴权模式对比

| 模式 | 适用场景 | 前端复杂度 |
|---|---|---|
| Session-Cookie | 传统 Web 应用、SSR | 低（浏览器自动处理） |
| JWT | SPA、移动端、跨服务鉴权 | 中（需手动管理 Token） |
| OAuth 2.0 | 第三方登录、开放平台 | 中（处理跳转和回调） |
| API Key | 服务间调用、SDK | 低（固定请求头） |

## 接口对接的常见问题

1. **跨域 (CORS)**：后端需要配置 `Access-Control-Allow-Origin`，开发环境用 Vite proxy
2. **Token 过期**：401 后自动刷新，并在刷新期间"挂起"其他请求，避免多个 401 同时刷
3. **接口字段名风格**：后端 `snake_case` → 前端 `camelCase`，BFF 层转换或用 Axios transformResponse
4. **接口版本**：`/api/v1/users`、`/api/v2/users`，前端按版本对接
5. **数据一致性**：后端返回的数据可能变化（新增字段/删除字段），前端要做好兜底和类型检查
