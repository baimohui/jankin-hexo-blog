---
title: RESTful API 设计与鉴权
categories: 
- 后端
tags:
- API
- RESTful
- HTTP
- 设计规范
- 鉴权
---

## 一、什么是 REST

REST（Representational State Transfer）是一种架构风格，而不是协议或标准。它定义了一组约束，遵循这些约束设计的 API 被称为 RESTful API。<!--more-->

### REST 的 6 个约束

| 约束 | 含义 | 体现 |
|------|------|------|
| **无状态** | 服务端不保存客户端状态，每个请求包含所有必要信息 | Token 随请求发送 |
| **客户端-服务端** | 关注点分离 | 前端只关心 UI，后端只关心数据 |
| **可缓存** | 响应应隐式或显式标记为可缓存或不可缓存 | `Cache-Control` 头 |
| **统一接口** | 资源通过 URL 标识，通过表述操作资源，自描述消息 | HTTP 动词 + 状态码 |
| **分层系统** | 客户端不知道是否直接连接端服务器 | 中间可加代理、网关 |
| **按需代码（可选）** | 服务器可临时扩展客户端功能 | JS 脚本下发（极少用） |

## 二、资源命名规范

### 核心原则：名词复数

```text
# 好的命名
GET    /api/users
GET    /api/users/:id
POST   /api/users
PUT    /api/users/:id
DELETE /api/users/:id

# 关系的表示
GET  /api/users/:id/orders         # 用户的订单列表
GET  /api/orders/:id/items         # 订单的商品列表

# 过滤、排序、分页（全部通过查询参数）
GET /api/orders?status=pending&page=1&pageSize=20&sort=-createdAt
```

### 常见反模式

```text
❌ 动词在 URL 中：/api/getUsers、/api/deleteUser
✅ 用 HTTP 动词表达：GET /api/users、DELETE /api/users/:id

❌ 模糊的端点：/api/doSomething
✅ 清晰的资源：/api/orders/:id/cancel（cancel 适合用 POST 动作）

❌ 大小写混用：/api/GetUsers、/api/userOrders
✅ 全小写 + 连字符：/api/order-items

❌ 文件扩展名：/api/users.json、/api/users.xml
✅ 通过 Accept 头协商：Accept: application/json
```

### 动作（Action）的处理

对于不适合 CRUD 的操作，用 POST 发送动作：

```text
POST /api/orders/:id/cancel     # 取消订单
POST /api/orders/:id/refund     # 退款
POST /api/users/:id/reset-password  # 重置密码

# 或者将动作作为请求体字段
POST /api/orders/:id/actions
{ "action": "cancel", "reason": "..." }
```

## 三、请求与响应设计

### 统一响应格式

```json
// 成功响应
{
  "code": 0,
  "message": "ok",
  "data": {
    "id": 1,
    "name": "Alice"
  }
}

// 列表响应（包含分页）
{
  "code": 0,
  "data": [...],
  "pagination": {
    "page": 1,
    "pageSize": 20,
    "total": 100,
    "totalPages": 5
  }
}

// 错误响应
{
  "code": "VALIDATION_ERROR",
  "message": "邮箱格式不正确",
  "field": "email",
  "details": "user@"
}
```

### 请求头约定

```text
Content-Type: application/json          # 请求体格式
Accept: application/json                # 期望的响应格式
Authorization: Bearer <token>            # 鉴权
X-Request-Id: uuid                      # 请求追踪 ID
X-Tenant-Code: tenant-a                 # 租户标识（SaaS）
Accept-Language: zh-CN                  # 国际化
If-None-Match: "abc123"                 # 条件请求（ETag）
```

### HTTP 状态码使用

```text
2xx 成功
├── 200 OK           GET / PUT / PATCH 成功
├── 201 Created      POST 创建资源成功
├── 202 Accepted     异步任务已接受
├── 204 No Content   DELETE 成功（无响应体）

3xx 重定向
├── 301 Moved Permanently    资源路径永久变更
├── 302 Found                临时重定向
├── 304 Not Modified          缓存有效

4xx 客户端错误
├── 400 Bad Request           参数错误/校验失败
├── 401 Unauthorized          未登录/Token 过期
├── 403 Forbidden             无权限（已登录但无权）
├── 404 Not Found             资源不存在
├── 409 Conflict              冲突（重复创建、版本冲突）
├── 422 Unprocessable Entity  语义错误（如邮箱格式错）
├── 429 Too Many Requests     限流

5xx 服务端错误
├── 500 Internal Server Error 服务器内部错误
├── 502 Bad Gateway           网关错误
├── 503 Service Unavailable   服务暂时不可用
├── 504 Gateway Timeout       网关超时
```

### 前端处理状态码

```ts
// Axios 统一处理
api.interceptors.response.use(
  response => response.data,
  error => {
    const status = error.response?.status;
    const data = error.response?.data;

    switch (status) {
      case 401:
        // Token 过期 → 刷新或跳转登录
        return refreshAndRetry(error);
      case 403:
        notification.warn('无权访问该资源');
        break;
      case 404:
        // 展示空状态而非报错
        return { code: 0, data: null };
      case 422:
        // 表单校验错误 → 回填到表单组件
        form.setFields(toFieldErrors(data.details));
        break;
      case 429:
        notification.warn('请求过于频繁，请稍后重试');
        break;
      default:
        errorReport.capture(error);
    }
    return Promise.reject(error);
  }
);
```

## 四、API 版本管理

```text
方案 A：URL 路径版本（推荐）
  /api/v1/users
  /api/v2/users

方案 B：请求头版本
  Accept: application/vnd.example.v1+json

方案 C：查询参数版本
  /api/users?version=1

最佳实践：
  - V1 稳定后不修改已有字段，只新增
  - 大版本使用 URL 路径方式（最直观）
  - 新增字段不要改变已有字段的语义
```

```ts
// 前端同时对接多个版本
const API_V1 = '/api/v1';
const API_V2 = '/api/v2';

// 降级策略：优先用 V2，不支持则回退 V1
async function getUsers() {
  try {
    return await api.get(`${API_V2}/users`);
  } catch {
    return await api.get(`${API_V1}/users`);
  }
}
```

## 五、安全设计

### 常见安全措施

```text
├── HTTPS：所有 API 必须走 HTTPS
├── 限流：按 IP/用户/Tenant 限制请求频率
├── 输入校验：服务端做白名单校验，前端仅做体验校验
├── 输出过滤：不返回敏感字段（密码、密钥）
├── 幂等性：POST 支持幂等 Key 防止重复提交
└── CORS：精确配置允许的来源，不开放给所有域名
```

### 防止重复提交

```ts
// 前端用 idempotencyKey
async function createOrder(orderData) {
  const idempotencyKey = crypto.randomUUID();

  return fetch('/api/orders', {
    method: 'POST',
    headers: {
      'Idempotency-Key': idempotencyKey,
    },
    body: JSON.stringify(orderData),
  });
}
```

## 六、REST 替代方案对比

```text
├── GraphQL
│   优势：客户端精确控制返回字段，一次请求获取关联数据
│   劣势：查询复杂度不可控，缓存比 REST 复杂
│   适用：数据模型复杂、前端需要灵活查询的场景

├── gRPC
│   优势：基于 Protobuf，性能高，强类型
│   劣势：浏览器支持有限，需 gRPC-Web 代理
│   适用：微服务间通信、后端到后端

├── WebSocket
│   优势：全双工实时通信
│   劣势：无请求-响应范式，无状态码
│   适用：实时协作、推送通知
└── REST
    优势：简单直观、缓存友好、工具链成熟
    适用：大多数 CRUD 类 Web API
```

## 七、面试题

### Q1: PUT 和 PATCH 有什么区别

```text
PUT：全量替换整个资源。客户端传完整对象，缺失字段会被置空。
PATCH：部分更新。客户端只传要修改的字段。

示例：
  PUT   /api/users/1  { name: "Alice", age: 25 }   必须传完整 user
  PATCH /api/users/1  { name: "Alice" }             只传 name 即可
```

### Q2: RESTful API 对前端最重要的设计原则是什么

```text
1. 统一接口：一套接口规则适用于所有资源，前端可以封装通用的请求函数
2. 无状态：每个请求独立，便于前端做重试和错误处理
3. 可缓存：合理的缓存策略减少前端请求量
4. 清晰的错误码：让前端能精确识别错误类型并给出友好提示
```

### Q3: 如何设计文件上传 API

```text
单文件：
  POST /api/upload
  Content-Type: multipart/form-data
  → { url: "https://cdn.example.com/file.pdf" }

分片上传：
  POST /api/upload/init      → { uploadId: "xxx" }
  POST /api/upload/part      → { uploadId, partNumber, body }
  POST /api/upload/complete   → { uploadId } → { url: "..." }

批量上传：
  POST /api/upload/batch
  → { files: [{ url: "..." }, { url: "..." }] }
```
