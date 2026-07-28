---
title: Express 详解
categories: 
- 后端
tags:
- Node.js
- Express
- 后端框架
- API
---

## 一、Express 是什么

Express 是 Node.js 最流行的 Web 应用框架，提供了一套简洁灵活的 API 用于构建 Web 服务器和 RESTful API。它是 Node.js 后端开发的入门标准。<!--more-->

### 为什么需要 Express

Node.js 原生 `http` 模块可以直接创建服务器，但处理路由、中间件、请求解析等工作需要大量手写代码：

```js
// 原生 http 模块
const http = require('http');
http.createServer((req, res) => {
  if (req.url === '/api/users' && req.method === 'GET') {
    // 手动解析 query、body、处理 CORS...
  }
});
```

Express 将上述重复工作抽象为简洁的 API：

```js
const express = require('express');
const app = express();
app.get('/api/users', (req, res) => res.json(users));
```

## 二、快速开始

```bash
npm init -y
npm install express
```

```js
const express = require('express');
const app = express();
const port = 3000;

app.get('/', (req, res) => {
  res.send('Hello Express');
});

app.listen(port, () => {
  console.log(`Server running at http://localhost:${port}`);
});
```

## 三、路由

### 基础路由

```js
app.get('/users', (req, res) => res.send('GET /users'));
app.post('/users', (req, res) => res.send('POST /users'));
app.put('/users/:id', (req, res) => res.send(`PUT /users/${req.params.id}`));
app.delete('/users/:id', (req, res) => res.send(`DELETE /users/${req.params.id}`));

// 匹配所有 HTTP 方法
app.all('/api/*', (req, res, next) => {
  console.log(`${req.method} ${req.path}`);
  next();
});
```

### 路由参数

```js
// :id 是动态参数
app.get('/users/:userId/orders/:orderId', (req, res) => {
  console.log(req.params);
  // { userId: '123', orderId: '456' }
});

// 查询参数
app.get('/search', (req, res) => {
  console.log(req.query);   // { q: 'express', page: '1' }
});
```

### 路由模块化（Router）

```js
// routes/users.js
const router = express.Router();

router.get('/', (req, res) => { /* GET /api/users */ });
router.get('/:id', (req, res) => { /* GET /api/users/:id */ });
router.post('/', (req, res) => { /* POST /api/users */ });

module.exports = router;
```

```js
// app.js
const userRouter = require('./routes/users');
app.use('/api/users', userRouter);    // 挂载到 /api/users 前缀
```

## 四、中间件

中间件是 Express 的核心机制。它是一个函数，在请求处理流程中按顺序执行。

```js
function middleware(req, res, next) {
  // 1. 处理请求
  // 2. 调用 next() 传递给下一个中间件
  // 3. 或直接 res.send() 结束响应
}
```

### 应用级中间件

```js
// 全局中间件：所有请求都会经过
app.use((req, res, next) => {
  console.log(`${new Date().toISOString()} ${req.method} ${req.path}`);
  next();
});

// 路径匹配的中间件
app.use('/api', (req, res, next) => {
  req.apiRequest = true;
  next();
});
```

### 内置中间件

```js
// 解析 JSON 请求体（express.json 替代了 body-parser）
app.use(express.json());

// 解析 URL-encoded 请求体
app.use(express.urlencoded({ extended: true }));

// 提供静态文件
app.use(express.static('public'));
// 访问 http://localhost:3000/image.png → 返回 public/image.png
```

### 第三方中间件

```bash
npm install cors morgan helmet
```

```js
const cors = require('cors');
const morgan = require('morgan');
const helmet = require('helmet');

app.use(helmet());                          // 安全头
app.use(cors());                            // 跨域
app.use(morgan('combined'));                // 请求日志
```

### 错误处理中间件

```js
// 同步错误
app.get('/error', (req, res) => {
  throw new Error('something broke');
});

// 异步错误（Express 5 自动捕获，Express 4 需手动传给 next）
app.get('/async-error', (req, res, next) => {
  asyncOperation().catch(next);
});

// 统一错误处理中间件（4 个参数）
app.use((err, req, res, next) => {
  console.error(err.stack);
  res.status(500).json({
    message: err.message,
    ...(process.env.NODE_ENV === 'development' && { stack: err.stack }),
  });
});
```

## 五、请求与响应

### Request 对象

```js
app.post('/users', (req, res) => {
  req.body;          // POST 请求体（需 express.json 解析）
  req.query;         // 查询参数  ?page=1 → { page: '1' }
  req.params;        // 路由参数  /users/:id → { id: '123' }
  req.headers;       // 请求头
  req.ip;            // 客户端 IP
  req.path;          // 请求路径
  req.method;        // HTTP 方法
});
```

### Response 对象

```js
res.status(201);                    // 设置状态码
res.json({ id: 1, name: 'Alice' });  // JSON 响应（自动设置 Content-Type）
res.send('<h1>Hello</h1>');         // HTML 字符串
res.redirect('/login');             // 重定向
res.sendFile('/path/to/file.pdf');  // 发送文件
res.download('/path/to/file.pdf');  // 下载文件
```

### 链式调用

```js
res.status(201).json({ id: 1 });
```

## 六、静态文件

```js
// 单个目录
app.use(express.static('public'));

// 多个目录（按顺序查找）
app.use(express.static('public'));
app.use(express.static('uploads'));

// 虚拟路径前缀
app.use('/static', express.static('public'));
// 访问 /static/style.css → 返回 public/style.css
```

## 七、模板引擎

Express 支持多种模板引擎（EJS、Pug、Handlebars）。

```bash
npm install ejs
```

```js
app.set('view engine', 'ejs');
app.set('views', './views');

// 渲染模板
app.get('/profile', (req, res) => {
  res.render('profile', { name: 'Alice', age: 25 });
});
```

```html
<!-- views/profile.ejs -->
<h1><%= name %></h1>
<p>Age: <%= age %></p>
```

## 八、项目结构规范

```
project/
├── app.js                    # 入口文件
├── routes/                   # 路由
│   ├── users.js
│   └── orders.js
├── controllers/              # 控制器（处理业务逻辑）
│   ├── userController.js
│   └── orderController.js
├── models/                   # 数据模型
│   └── user.js
├── middleware/               # 自定义中间件
│   ├── auth.js
│   └── validate.js
├── config/                   # 配置
│   └── index.js
├── public/                   # 静态文件
│   ├── css/
│   └── js/
└── views/                    # 模板
    └── index.ejs
```

### 控制器模式

```js
// controllers/userController.js
exports.list = async (req, res, next) => {
  try {
    const users = await User.find();
    res.json(users);
  } catch (err) {
    next(err);
  }
};
```

```js
// routes/users.js
const router = require('express').Router();
const userCtrl = require('../controllers/userController');

router.get('/', userCtrl.list);
router.get('/:id', userCtrl.getById);
router.post('/', userCtrl.create);
```

## 九、常见问题

### Q1: req.body 始终为 undefined

```js
// 缺少 JSON 解析中间件
app.use(express.json());       // ✅ 必须添加
```

### Q2: 跨域问题

```js
const cors = require('cors');
app.use(cors());               // 允许所有来源

// 或限制特定来源
app.use(cors({
  origin: 'https://example.com',
  methods: ['GET', 'POST'],
}));
```

### Q3: 异步错误捕获

```js
// Express 4 不自动捕获异步异常，需用 next
app.get('/async', (req, res, next) => {
  asyncFunction().catch(next);
});

// 或用包装函数
const asyncHandler = (fn) => (req, res, next) =>
  Promise.resolve(fn(req, res, next)).catch(next);
```

### Q4: Express vs Koa 怎么选

Express 社区更成熟、中间件生态丰富、学习曲线低。Koa 利用 async/await 更优雅地处理异步，但社区较小。大多数新项目仍首选 Express。

## 十、推荐学习路径

1. 掌握基础路由和中间件机制
2. 使用 express.Router 模块化路由
3. 集成数据库（Mongoose / Sequelize）
4. 实现 JWT 鉴权和错误处理
5. 使用 cors、helmet、morgan 等中间件加固生产环境
6. 结合 Docker 和 PM2 部署
