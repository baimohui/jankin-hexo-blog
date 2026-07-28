---
title: Express 详解
categories: 
- 后端
tags:
- Node.js
- Express
- 后端框架
- API
- Mongoose
---

## 一、Express 是什么

Express 是 Node.js 最流行的 Web 应用框架，它基于 Node.js 原生 `http` 模块构建，在请求响应流程上抽象出**中间件**机制，使开发者能以声明式的方式组织请求处理逻辑。<!--more-->

### 设计哲学

Express 的核心设计只有三条：

```text
1. 中间件（Middleware）：请求按顺序经过一系列函数
2. 路由（Routing）：根据 HTTP 方法和路径匹配到对应处理函数
3. 增强 req/res：在中间件中为请求/响应对象添加属性和方法
```

### Express 解决了什么问题

原生 `http` 模块的重复工作：

```js
// 原生 http：手动解析路径、手动处理 CORS、手动解析 body
const http = require('http');
http.createServer((req, res) => {
  // 手动解析 URL
  const url = require('url').parse(req.url);
  // 手动处理 CORS
  res.setHeader('Access-Control-Allow-Origin', '*');
  // 手动解析 Body
  let body = '';
  req.on('data', chunk => body += chunk);
  req.on('end', () => {
    if (url.pathname === '/api/users' && req.method === 'GET') {
      res.end(JSON.stringify(users));
    }
  });
});
```

Express 将上述工作抽象为**声明式的中间件链**：

```js
const express = require('express');
const app = express();

app.use(cors());               // 跨域
app.use(express.json());       // Body 解析
app.get('/api/users', listUsers);  // 路由
```

## 二、中间件原理

### 中间件执行模型

Express 的内部架构本质上是一个**中间件队列**。每次请求进来，Express 将匹配到的所有中间件按顺序串起来执行：

```text
请求到达
  │
  ├── middleware1（日志记录）
  │    next()
  ├── middleware2（登录验证）
  │    next()
  ├── middleware3（路由处理）
  │    res.json(...)   ← 响应结束
  │
  └── 响应返回客户端
```

每个中间件要么调用 `next()` 将控制权交给下一个中间件，要么直接结束响应（`res.send()` / `res.json()` 等）。

### 中间件的类型

```js
// 1. 普通中间件（匹配所有请求）
app.use((req, res, next) => {
  console.log('Time:', Date.now());
  next();
});

// 2. 路径匹配中间件（只匹配特定路径开头）
app.use('/api', (req, res, next) => {
  req.isApiRequest = true;
  next();
});

// 3. 错误处理中间件（4 个参数）
app.use((err, req, res, next) => {
  console.error(err);
  res.status(500).json({ error: err.message });
});
```

### 内置中间件

```js
const app = express();

// 解析 JSON body（替换了已废弃的 body-parser）
app.use(express.json());
app.use(express.json({ limit: '10mb' }));

// 解析 URL-encoded body
app.use(express.urlencoded({ extended: true }));
// extended: true  → 使用 qs 库，支持嵌套对象
// extended: false → 使用 querystring 库，不支持嵌套

// 静态文件服务
app.use(express.static('public'));
app.use('/static', express.static('public'));

// 静态文件缓存配置
app.use(express.static('public', {
  maxAge: '1d',
  immutable: true,  // 不可变文件（文件名带 hash）
}));
```

### 错误处理中间件的原理

Express 识别错误处理中间件的依据是**参数个数**——当参数个数为 4 时视为错误处理中间件。`next(err)` 被调用时，Express 跳过后面的所有普通中间件，直接进入第一个错误处理中间件。

```js
// next()           → 下一个普通中间件
// next('route')    → 跳过当前路由的剩余中间件
// next(err)        → 跳转到错误处理中间件
// next(new Error('...')) → 同上，但携带错误信息
```

#### 异步错误捕获

```js
// Express 4 不自动捕获异步错误
app.get('/users', async (req, res, next) => {
  try {
    const users = await User.find();
    res.json(users);
  } catch (err) {
    next(err);  // 必须手动传给 next
  }
});

// 封装辅助函数（推荐）
const asyncHandler = (fn) => (req, res, next) =>
  Promise.resolve(fn(req, res, next)).catch(next);

app.get('/users', asyncHandler(async (req, res) => {
  const users = await User.find();
  res.json(users);
}));

// Express 5 自动捕获 async handler 中的异常，无需手动 catch
```

### 自定义中间件示例

```js
// middlewares/requestLogger.js
module.exports = (req, res, next) => {
  const start = Date.now();
  res.on('finish', () => {
    const duration = Date.now() - start;
    console.log(`${req.method} ${req.originalUrl} ${res.statusCode} ${duration}ms`);
  });
  next();
};

// middlewares/rateLimiter.js
const rateLimit = new Map();

module.exports = (windowMs = 60000, maxRequests = 100) =>
  (req, res, next) => {
    const ip = req.ip;
    const now = Date.now();

    if (!rateLimit.has(ip)) {
      rateLimit.set(ip, []);
    }

    const requests = rateLimit.get(ip).filter(t => now - t < windowMs);

    if (requests.length >= maxRequests) {
      return res.status(429).json({ error: 'Too many requests' });
    }

    requests.push(now);
    rateLimit.set(ip, requests);
    next();
  };
```

## 三、路由

### 路由方法

Express 对每个 HTTP 方法提供了对应 API：

```js
app.get(path, handler)
app.post(path, handler)
app.put(path, handler)
app.patch(path, handler)
app.delete(path, handler)
app.options(path, handler)
app.head(path, handler)
app.all(path, handler)    // 匹配所有 HTTP 方法
```

### 路由参数与模式匹配

```js
// 命名参数
app.get('/users/:userId/posts/:postId', (req, res) => {
  req.params.userId;   // 字符串
  req.params.postId;
});

// 可选参数
app.get('/users/:id?', (req, res) => {
  // /users/123 和 /users 都匹配
});

// 正则匹配
app.get(/\/users\/(\d+)/, (req, res) => {
  req.params[0];  // 正则捕获组通过 params[0] 访问
});

// 通配符匹配
app.get('/files/*', (req, res) => {
  req.params[0];  // 通配符捕获的内容
});
```

### 路由的链式定义

```js
app.route('/users/:id')
  .get((req, res) => { /* 获取用户 */ })
  .put((req, res) => { /* 更新用户 */ })
  .delete((req, res) => { /* 删除用户 */ });
```

### Router——路由模块化

```js
// routes/auth.js
const router = require('express').Router();

router.post('/login', (req, res) => { /* 登录 */ });
router.post('/register', (req, res) => { /* 注册 */ });
router.post('/logout', (req, res) => { /* 登出 */ });

module.exports = router;
```

```js
// routes/users.js
const router = require('express').Router();
const userController = require('../controllers/userController');
const auth = require('../middlewares/auth');

// 路由内的中间件——只在此路由内生效
router.use(auth.requireLogin);

router.get('/', userController.list);
router.get('/:id', userController.getById);

module.exports = router;
```

```js
// app.js
const authRouter = require('./routes/auth');
const userRouter = require('./routes/users');

app.use('/api/auth', authRouter);
app.use('/api/users', userRouter);
```

## 四、请求与响应对象

### Request（req） 的属性和方法

```js
req.params      // 路由参数：/users/:id → { id: '123' }
req.query       // 查询字符串：?page=1 → { page: '1' }
req.body        // POST 请求体（需 express.json 中间件）
req.headers     // 请求头对象
req.get('Content-Type')  // 读取特定请求头

req.path        // 请求路径：/api/users
req.method      // HTTP 方法：GET / POST
req.url         // 完整 URL（含查询字符串）
req.originalUrl // 原始 URL（不受路由挂载影响）

req.ip          // 客户端 IP 地址
req.protocol    // 协议：http / https
req.hostname    // 主机名
req.subdomains  // 子域名数组

req.cookies     // Cookie（需 cookie-parser 中间件）
req.signedCookies // 签名 Cookie

req.xhr         // 是否为 AJAX 请求（判断 X-Requested-With 头）
req.accepts('json')  // 内容协商
```

### Response（res）的属性和方法

```js
// 响应方法
res.status(code)           // 设置状态码，返回 res 以支持链式调用
res.json(obj)              // JSON 响应（自动设置 Content-Type: application/json）
res.jsonp(obj)             // JSONP 响应
res.send(data)             // 通用响应（自动推断类型）
res.sendFile(path)         // 发送文件
res.download(path)         // 下载文件
res.redirect(url)          // 重定向
res.render(view, data)     // 渲染模板

// 设置响应头
res.set('X-Custom-Header', 'value');
res.set({ 'Header1': 'v1', 'Header2': 'v2' });
res.type('application/json');  // 设置 Content-Type

// Cookie 操作
res.cookie('token', 'xxx', { httpOnly: true, maxAge: 3600000, secure: true });
res.clearCookie('token');

// 链式调用
res.status(201).json({ id: 1 });
```

## 五、项目结构与架构

### 推荐的分层架构

```
project/
├── config/               # 配置文件
│   ├── index.js          # 全局配置
│   ├── db.js             # 数据库连接
│   └── env.js            # 环境变量
├── models/               # 数据模型（Mongoose Schema）
│   ├── user.js
│   └── order.js
├── routes/               # 路由定义（薄层）
│   ├── auth.js
│   ├── users.js
│   └── orders.js
├── controllers/          # 控制器（处理请求/响应）
│   ├── authController.js
│   ├── userController.js
│   └── orderController.js
├── services/             # 业务逻辑层（可选，进一步解耦）
│   ├── authService.js
│   └── orderService.js
├── middlewares/          # 中间件
│   ├── auth.js
│   ├── errorHandler.js
│   └── validate.js
├── validators/           # 请求校验（如 Joi/Zod Schema）
│   └── userValidator.js
├── utils/                # 工具函数
│   ├── ApiError.js
│   └── asyncHandler.js
├── tests/                # 测试
├── app.js                # Express 实例创建与中间件注册
└── server.js             # 入口文件（启动服务器）
```

各层职责明确：

```text
routes       → 只做路由分发，不包含业务逻辑
controllers  → 处理请求参数、调用 service、构造响应
services     → 纯业务逻辑（可复用、可测试）
models       → 数据模型定义
middlewares  → 跨路由的横切关注点（鉴权、日志、限流）
```

### 入口文件示例

```js
// server.js
const app = require('./app');
const config = require('./config');

app.listen(config.port, () => {
  console.log(`Server running on port ${config.port}`);
});
```

```js
// app.js
const express = require('express');
const cors = require('cors');
const morgan = require('morgan');
const helmet = require('helmet');
const { errorHandler, notFoundHandler } = require('./middlewares/errorHandler');
const routes = require('./routes');

const app = express();

// 全局中间件（顺序重要）
app.use(helmet());
app.use(cors());
app.use(morgan('dev'));
app.use(express.json({ limit: '10mb' }));
app.use(express.urlencoded({ extended: true }));

// 路由
app.use('/api/auth', require('./routes/auth'));
app.use('/api/users', require('./routes/users'));

// 404 处理
app.use(notFoundHandler);

// 错误处理（必须放在最后）
app.use(errorHandler);

module.exports = app;
```

## 六、数据库集成（Mongoose）

### 连接配置

```bash
npm install mongoose
```

```js
// config/db.js
const mongoose = require('mongoose');

const connectDB = async () => {
  try {
    await mongoose.connect(process.env.MONGODB_URI || 'mongodb://localhost:27017/myapp', {
      // Mongoose 7+ 不再需要这些选项
    });
    console.log('MongoDB connected');
  } catch (err) {
    console.error('MongoDB connection error:', err);
    process.exit(1);
  }
};

module.exports = connectDB;
```

### 定义 Model

```js
// models/user.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

const userSchema = new mongoose.Schema({
  name: {
    type: String,
    required: [true, '用户名不能为空'],
    trim: true,
    minlength: [2, '用户名至少 2 个字符'],
    maxlength: [50, '用户名最多 50 个字符'],
  },
  email: {
    type: String,
    required: [true, '邮箱不能为空'],
    unique: true,               // 唯一索引
    lowercase: true,             // 自动转小写
    match: [/^\S+@\S+\.\S+$/, '邮箱格式不正确'],
  },
  password: {
    type: String,
    required: [true, '密码不能为空'],
    minlength: [6, '密码至少 6 位'],
    select: false,              // 查询时默认不返回 password
  },
  role: {
    type: String,
    enum: ['user', 'admin'],
    default: 'user',
  },
  isActive: {
    type: Boolean,
    default: true,
  },
}, {
  timestamps: true,              // 自动添加 createdAt / updatedAt
  toJSON: {
    transform(doc, ret) {
      ret.id = ret._id.toString();
      delete ret._id;
      delete ret.__v;
      delete ret.password;
      return ret;
    },
  },
});

// 保存前加密密码
userSchema.pre('save', async function(next) {
  if (!this.isModified('password')) return next();
  this.password = await bcrypt.hash(this.password, 12);
  next();
});

// 实例方法：验证密码
userSchema.methods.comparePassword = async function(candidatePassword) {
  return bcrypt.compare(candidatePassword, this.password);
};

module.exports = mongoose.model('User', userSchema);
```

### 控制器实现

```js
// controllers/userController.js
const User = require('../models/user');
const ApiError = require('../utils/ApiError');

exports.list = async (req, res) => {
  const { page = 1, limit = 20, role, isActive } = req.query;

  const filter = {};
  if (role) filter.role = role;
  if (isActive !== undefined) filter.isActive = isActive === 'true';

  const users = await User.find(filter)
    .sort({ createdAt: -1 })
    .skip((page - 1) * limit)
    .limit(Number(limit))
    .select('-password');

  const total = await User.countDocuments(filter);

  res.json({
    data: users,
    pagination: {
      page: Number(page),
      limit: Number(limit),
      total,
      totalPages: Math.ceil(total / limit),
    },
  });
};

exports.getById = async (req, res) => {
  const user = await User.findById(req.params.id);
  if (!user) throw new ApiError(404, '用户不存在');
  res.json({ data: user });
};

exports.create = async (req, res) => {
  const { name, email, password, role } = req.body;

  const existing = await User.findOne({ email });
  if (existing) throw new ApiError(409, '邮箱已被注册');

  const user = await User.create({ name, email, password, role });
  res.status(201).json({ data: user });
};

exports.update = async (req, res) => {
  const { name, role } = req.body;
  const user = await User.findByIdAndUpdate(
    req.params.id,
    { name, role },
    { new: true, runValidators: true }
  );
  if (!user) throw new ApiError(404, '用户不存在');
  res.json({ data: user });
};

exports.remove = async (req, res) => {
  const user = await User.findByIdAndDelete(req.params.id);
  if (!user) throw new ApiError(404, '用户不存在');
  res.status(204).end();
};
```

### 路由注册

```js
// routes/users.js
const router = require('express').Router();
const userController = require('../controllers/userController');
const auth = require('../middlewares/auth');
const validate = require('../middlewares/validate');
const { createUserSchema, updateUserSchema } = require('../validators/userValidator');

router.use(auth.requireLogin);  // 以下路由都需要登录

router.get('/', userController.list);
router.post('/', validate(createUserSchema), userController.create);
router.get('/:id', userController.getById);
router.put('/:id', validate(updateUserSchema), userController.update);
router.delete('/:id', userController.remove);

module.exports = router;
```

## 七、JWT 鉴权完整实现

### 生成与验证 Token

```bash
npm install jsonwebtoken bcryptjs
```

```js
// services/authService.js
const jwt = require('jsonwebtoken');
const User = require('../models/user');

const generateTokens = (userId) => {
  const accessToken = jwt.sign(
    { userId },
    process.env.JWT_SECRET,
    { expiresIn: '15m' }
  );
  const refreshToken = jwt.sign(
    { userId },
    process.env.JWT_REFRESH_SECRET,
    { expiresIn: '7d' }
  );
  return { accessToken, refreshToken };
};

exports.login = async (email, password) => {
  const user = await User.findOne({ email }).select('+password');
  if (!user) throw new ApiError(401, '邮箱或密码错误');

  const isMatch = await user.comparePassword(password);
  if (!isMatch) throw new ApiError(401, '邮箱或密码错误');

  const tokens = generateTokens(user._id);
  return { user, ...tokens };
};

exports.register = async (userData) => {
  const existing = await User.findOne({ email: userData.email });
  if (existing) throw new ApiError(409, '邮箱已被注册');

  const user = await User.create(userData);
  const tokens = generateTokens(user._id);
  return { user, ...tokens };
};

exports.refreshToken = async (refreshToken) => {
  try {
    const decoded = jwt.verify(refreshToken, process.env.JWT_REFRESH_SECRET);
    return generateTokens(decoded.userId);
  } catch (err) {
    throw new ApiError(401, 'Token 已过期，请重新登录');
  }
};
```

### 鉴权中间件

```js
// middlewares/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/user');

exports.requireLogin = async (req, res, next) => {
  try {
    const authHeader = req.headers.authorization;
    if (!authHeader?.startsWith('Bearer ')) {
      throw new ApiError(401, '未提供认证 Token');
    }

    const token = authHeader.split(' ')[1];
    const decoded = jwt.verify(token, process.env.JWT_SECRET);

    const user = await User.findById(decoded.userId);
    if (!user) throw new ApiError(401, '用户不存在');

    req.user = user;   // 将用户信息挂到 req 上，后续 handler 可用
    next();
  } catch (err) {
    if (err.name === 'JsonWebTokenError') {
      next(new ApiError(401, 'Token 无效'));
    } else if (err.name === 'TokenExpiredError') {
      next(new ApiError(401, 'Token 已过期'));
    } else {
      next(err);
    }
  }
};

exports.requireRole = (...roles) => (req, res, next) => {
  if (!roles.includes(req.user.role)) {
    return next(new ApiError(403, '权限不足'));
  }
  next();
};
```

### 鉴权路由

```js
// routes/auth.js
const router = require('express').Router();
const authService = require('../services/authService');
const auth = require('../middlewares/auth');

router.post('/login', async (req, res) => {
  const { email, password } = req.body;
  const result = await authService.login(email, password);
  res.json(result);
});

router.post('/register', async (req, res) => {
  const result = await authService.register(req.body);
  res.status(201).json(result);
});

router.post('/refresh', async (req, res) => {
  const { refreshToken } = req.body;
  const tokens = await authService.refreshToken(refreshToken);
  res.json(tokens);
});

router.get('/me', auth.requireLogin, async (req, res) => {
  res.json({ data: req.user });
});

module.exports = router;
```

## 八、错误处理体系

### 自定义错误类

```js
// utils/ApiError.js
class ApiError extends Error {
  constructor(statusCode, message) {
    super(message);
    this.statusCode = statusCode;
    this.isOperational = true;  // 标记为可预见的操作错误
    Error.captureStackTrace(this, this.constructor);
  }
}

module.exports = ApiError;
```

### 全局错误处理中间件

```js
// middlewares/errorHandler.js
const ApiError = require('../utils/ApiError');

const errorHandler = (err, req, res, next) => {
  // 默认服务器错误
  let statusCode = err.statusCode || 500;
  let message = err.message || '服务器内部错误';

  // Mongoose 校验错误
  if (err.name === 'ValidationError') {
    statusCode = 422;
    const errors = Object.values(err.errors).map(e => ({
      field: e.path,
      message: e.message,
    }));
    return res.status(422).json({ code: 'VALIDATION_ERROR', errors });
  }

  // Mongoose 重复键错误
  if (err.code === 11000) {
    const field = Object.keys(err.keyValue)[0];
    statusCode = 409;
    message = `${field} 已存在`;
  }

  // Mongoose 无效 ID
  if (err.name === 'CastError') {
    statusCode = 400;
    message = '无效的 ID 格式';
  }

  // JWT 错误
  if (err.name === 'JsonWebTokenError') {
    statusCode = 401;
    message = 'Token 无效';
  }

  // 开发环境返回错误堆栈
  const response = { code: statusCode, message };
  if (process.env.NODE_ENV === 'development') {
    response.stack = err.stack;
  }

  res.status(statusCode).json(response);
};

const notFoundHandler = (req, res) => {
  res.status(404).json({
    code: 404,
    message: `路径 ${req.originalUrl} 不存在`,
  });
};

module.exports = { errorHandler, notFoundHandler };
```

## 九、数据校验

```bash
npm install joi
```

```js
// validators/userValidator.js
const Joi = require('joi');

const createUserSchema = Joi.object({
  name: Joi.string().min(2).max(50).required(),
  email: Joi.string().email().required(),
  password: Joi.string().min(6).required(),
  role: Joi.string().valid('user', 'admin').default('user'),
});

const updateUserSchema = Joi.object({
  name: Joi.string().min(2).max(50),
  role: Joi.string().valid('user', 'admin'),
}).min(1);  // 至少有一个字段

module.exports = { createUserSchema, updateUserSchema };
```

```js
// middlewares/validate.js
const ApiError = require('../utils/ApiError');

const validate = (schema) => (req, res, next) => {
  const { error, value } = schema.validate(req.body, {
    abortEarly: false,       // 返回所有错误
    stripUnknown: true,      // 移除未定义的字段
  });

  if (error) {
    const errors = error.details.map(d => ({
      field: d.path.join('.'),
      message: d.message,
    }));
    throw new ApiError(422, errors[0].message);
  }

  req.body = value;  // 替换为校验后的值
  next();
};

module.exports = validate;
```

## 十、文件上传

```bash
npm install multer
```

```js
// middlewares/upload.js
const multer = require('multer');
const path = require('path');
const ApiError = require('../utils/ApiError');

const storage = multer.diskStorage({
  destination: (req, file, cb) => {
    cb(null, 'uploads/');
  },
  filename: (req, file, cb) => {
    const uniqueSuffix = Date.now() + '-' + Math.round(Math.random() * 1E9);
    cb(null, uniqueSuffix + path.extname(file.originalname));
  },
});

const fileFilter = (req, file, cb) => {
  const allowed = ['image/jpeg', 'image/png', 'image/webp'];
  if (!allowed.includes(file.mimetype)) {
    return cb(new ApiError(400, '只允许上传 JPEG、PNG、WebP 图片'), false);
  }
  cb(null, true);
};

const upload = multer({
  storage,
  fileFilter,
  limits: { fileSize: 5 * 1024 * 1024 },  // 5MB
});

module.exports = upload;
```

```js
// routes/upload.js
const router = require('express').Router();
const upload = require('../middlewares/upload');

router.post('/avatar', upload.single('avatar'), (req, res) => {
  res.json({ data: { url: `/uploads/${req.file.filename}` } });
});

router.post('/gallery', upload.array('photos', 10), (req, res) => {
  const files = req.files.map(f => ({ url: `/uploads/${f.filename}` }));
  res.json({ data: files });
});
```

## 十一、测试

```bash
npm install --save-dev jest supertest
```

```js
// tests/auth.test.js
const request = require('supertest');
const app = require('../app');
const mongoose = require('mongoose');
const User = require('../models/user');

beforeAll(async () => {
  // 连接测试数据库
  await mongoose.connect(process.env.TEST_DB_URI);
});

afterEach(async () => {
  // 每个测试后清理数据
  await User.deleteMany({});
});

afterAll(async () => {
  await mongoose.disconnect();
});

describe('POST /api/auth/register', () => {
  it('should register a new user', async () => {
    const res = await request(app)
      .post('/api/auth/register')
      .send({ name: 'Alice', email: 'alice@test.com', password: '123456' })
      .expect(201);

    expect(res.body.data).toHaveProperty('id');
    expect(res.body).toHaveProperty('accessToken');
    expect(res.body).toHaveProperty('refreshToken');
  });

  it('should reject duplicate email', async () => {
    await User.create({ name: 'Alice', email: 'alice@test.com', password: '123456' });

    const res = await request(app)
      .post('/api/auth/register')
      .send({ name: 'Bob', email: 'alice@test.com', password: '123456' })
      .expect(409);

    expect(res.body.message).toContain('已被注册');
  });
});
```

## 十二、常见问题

### Q1: Express 4 vs Express 5 的主要区别

```text
Express 5 改进：
  1. 自动捕获 async handler 中的异常（不再需要 try/catch + next）
  2. app.disable('x-powered-by') 默认启用
  3. res.send() 自动识别 Buffer、TypedArray、BigInt
  4. 废弃 app.del()，统一用 app.delete()
  5. req.host 不再包含端口号（改用 req.hostname）
```

### Q2: app.use 和 app.get 有什么区别

```text
app.use(path, handler)：匹配所有以 path 开头的路径
  app.use('/api', handler) → 匹配 /api、/api/users、/api/users/123

app.get(path, handler)：精确匹配 GET 请求
  app.get('/api', handler) → 只匹配 GET /api

app.use 用于挂载中间件和子路由（不关注具体方法），
app.get/post 用于定义具体路由。
```

### Q3: next() 执行后的代码还会运行吗

```text
next() 之后的代码仍然会执行。不会自动 return。
推荐在调用 next() 后加 return 避免意外执行后续代码：

  app.use((req, res, next) => {
    if (!req.headers.authorization) {
      return next(new Error('Unauthorized'));  // return 阻止后续执行
    }
    // 继续处理
    next();
  });
```

### Q4: 如何管理环境变量

```bash
npm install dotenv
```

```js
// config/env.js
const dotenv = require('dotenv');
const path = require('path');

dotenv.config({ path: path.join(__dirname, '../../.env') });

// config/index.js
module.exports = {
  port: process.env.PORT || 3000,
  mongodbUri: process.env.MONGODB_URI || 'mongodb://localhost:27017/myapp',
  jwtSecret: process.env.JWT_SECRET || 'dev-secret',
  jwtRefreshSecret: process.env.JWT_REFRESH_SECRET || 'dev-refresh-secret',
  nodeEnv: process.env.NODE_ENV || 'development',
};
```

### Q5: Express vs Koa vs Fastify 选型

```text
Express：生态最大、中间件最多、学习成本最低。适合大多数项目。
Koa：利用 async/await 消除回调，更现代。但第三方中间件偏少。
Fastify：性能极强（每秒请求数约为 Express 的 2-3 倍），内置 JSON Schema
        校验和日志。适合高并发场景。

大多数项目首选 Express，它的生态和社区资源是不可替代的。
```

## 十三、推荐学习路径

1. 理解中间件执行模型（`next()` 如何串联）
2. 掌握路由定义和 Router 模块化
3. 集成 Mongoose，实现 CRUD API
4. 实现 JWT 鉴权体系（登录/注册/Token 刷新）
5. 加入数据校验（Joi/Zod）和错误处理体系
6. 加入文件上传（Multer）
7. 编写自动化测试（Jest + Supertest）
8. 使用 PM2 + Nginx 部署到生产环境
