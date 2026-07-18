---
title: Node.js 与全栈基础（Express / Koa）
categories: 
- 后端
tags:
- Node.js
- Express
- Koa
- 全栈
---

## 为什么 Node.js 是前端转后端的最佳入口

<!-- more -->
- **语言一致**：JavaScript/TypeScript 前后端通用
- **NPM 生态**：前端熟悉的包管理，后端也有 Express/Koa/Prisma
- **事件循环模型**：前端理解的事件队列，Node.js 后端也是同一个
- **BFF（Backend For Frontend）**：前端团队维护中间层，Node.js 是最佳选择

## Express 基础

最流行的 Node.js Web 框架，核心就是**中间件 + 路由**：

```javascript
const express = require('express')
const app = express()
app.use(express.json()) // 中间件：解析 JSON 请求体
// 路由 + 处理函数
app.get('/api/users', (req, res) => {
  res.json([{ id: 1, name: '张三' }])
})
app.post('/api/users', (req, res) => {
  console.log(req.body) // 前端传来的数据
  res.status(201).json({ id: 2, ...req.body })
})
app.listen(3000)
```

## 中间件机制（与前端类比）

**Express/Koa 的核心**：一个请求依次经过一系列中间件，每个中间件可以修改请求/响应或终止处理。

```text
请求 → 日志中间件 → 鉴权中间件 → 路由处理器 → 响应
         ↓            ↓
      记录日志     校验 Token
```

**前端类比：** `Axios 拦截器` — `request.interceptors` 在请求发出前处理，`response.interceptors` 在收到响应后处理，就是中间件模式。

**对比 Express vs Koa：**

| 维度 | Express | Koa |
|---|---|---|
| 作者 | 同一个人 | 同一个人（升级版） |
| 中间件模型 | 线性（回调） | 洋葱模型（async/await） |
| 错误处理 | 内置 error handler | try/catch + ctx.throw |
| 体积 | 大（内置路由、静态文件等） | 小（核心极简，功能靠中间件） |
| 适用场景 | 快速开发、教程多 | 需要精细控制的场景 |
| TypeScript | 社区 @types | 原生友好 |

## Koa 的洋葱模型

```javascript
const Koa = require('koa')
const app = new Koa()

app.use(async (ctx, next) => {
  console.log('① 请求进来')
  await next()
  console.log('④ 响应出去')
})

app.use(async (ctx, next) => {
  console.log('② 处理中')
  await next()
  console.log('③ 处理完返回')
})

app.use(async ctx => {
  ctx.body = 'Hello World'
})
// 输出顺序：① → ② → ③ → ④
```

**前端类比：** Vue Router 的导航守卫 `beforeEach → beforeResolve → afterEach` 也是类似的洋葱模型。

## 连接前后端：BFF 层

```text
前端 (React/Vue)
  ↓ 请求
BFF 层 (Node.js/Express)   ← 前端团队维护
  ↓ 转发/聚合
后端微服务 (Java/Go/Python)
```

**BFF 的作用：**
- 聚合多个后端接口的数据，按前端需要返回
- 处理与前端相关的逻辑（登录态、权限）
- 数据格式转换（后端返回的 snake_case → 前端的 camelCase）

## .js → .ts 切换到后端

后端更需要 TypeScript：接口参数类型验证、数据库查询结果类型、错误类型都有明确的类型定义。使用 NestJS（基于 Express/Koa 的 TypeScript 框架）是目前 Node.js 后端的最佳实践。

```typescript
// NestJS 示例 — 接近前端熟悉的装饰器/注解风格
@Controller('users')
export class UserController {
  @Get(':id')
  findOne(@Param('id') id: string) {
    return this.userService.findById(id)
  }
}
```
