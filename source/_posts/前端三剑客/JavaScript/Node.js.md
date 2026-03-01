---
title: Node.js 详解
categories:
- JavaScript
tags:
- JavaScript
- Node.js
- 后端
---

Node.js 是一个基于 Chrome V8 引擎的 JavaScript 运行时环境，它让 JavaScript 可以在浏览器之外运行。本文将从概念到实践，全面介绍 Node.js，并深入分析它与浏览器 JavaScript 的差异。<!-- more -->

---

## 一、Node.js 是什么？

### 1.1 官方定义

> Node.js 是一个基于 Chrome V8 引擎的 JavaScript **运行时环境**。

**关键点解析：**
- **运行时环境（Runtime）**：提供 JavaScript 执行所需的环境和 API
- **基于 V8 引擎**：使用 Google Chrome 的 V8 JavaScript 引擎，执行速度快
- **非浏览器环境**：让 JavaScript 脱离浏览器，在服务器端运行

---

### 1.2 为什么要创造 Node.js？

在 Node.js 出现之前，JavaScript 只能运行在浏览器中。Ryan Dahl 在 2009 年创造了 Node.js，解决了以下问题：

| 问题 | Node.js 的解决方案 |
|------|-------------------|
| JavaScript 只能在浏览器运行 | 让 JavaScript 可以在服务器端运行 |
| 传统服务器（如 Apache）高并发性能差 | 基于事件驱动、非阻塞 I/O，高并发性能好 |
| 前后端语言不一致 | 前后端都用 JavaScript，降低学习成本 |
| 线程模型资源消耗大 | 单线程 + 事件循环，资源占用少 |

---

### 1.3 Node.js 的核心特性

#### 特性一：单线程

Node.js 使用**单线程**模型，所有 JavaScript 代码在主线程执行。

**优点：**
- 避免了多线程的上下文切换开销
- 编程模型简单，没有死锁问题
- 内存占用少

**缺点：**
- 不适合 CPU 密集型任务
- 一个请求阻塞会影响其他请求

---

#### 特性二：非阻塞 I/O

Node.js 的 I/O 操作（文件读写、网络请求等）都是**非阻塞**的。

```javascript
// 同步读取文件（阻塞）
const fs = require('fs');
const data = fs.readFileSync('file.txt'); // 阻塞等待
console.log('文件内容：', data);
console.log('这行代码等待文件读取完成后才执行');

// 异步读取文件（非阻塞）
fs.readFile('file.txt', (err, data) => {
  if (err) throw err;
  console.log('文件内容：', data);
});
console.log('这行代码立即执行，不等待文件读取');
```

---

#### 特性三：事件驱动

Node.js 使用**事件驱动**架构，通过事件循环处理异步操作。

```
┌─────────────────────────────────────────────────────────┐
│                      事件循环 (Event Loop)                │
├─────────────────────────────────────────────────────────┤
│  1. 执行同步代码                                          │
│  2. 遇到异步操作，交给底层系统处理                          │
│  3. 继续执行后续代码                                        │
│  4. 异步操作完成，将回调函数放入任务队列                     │
│  5. 事件循环检测任务队列，执行回调函数                       │
└─────────────────────────────────────────────────────────┘
```

---

## 二、Node.js 的核心概念

### 2.1 模块系统（CommonJS）

Node.js 使用 **CommonJS** 模块规范，通过 `require` 和 `module.exports` 进行模块化。

#### 导出模块

```javascript
// math.js
function add(a, b) {
  return a + b;
}

function subtract(a, b) {
  return a - b;
}

// 导出多个成员
module.exports = {
  add,
  subtract
};

// 或者单独导出
exports.multiply = (a, b) => a * b;
```

#### 导入模块

```javascript
// app.js
const math = require('./math.js');

console.log(math.add(1, 2));        // 3
console.log(math.subtract(5, 3));   // 2
console.log(math.multiply(2, 3));   // 6

// 解构导入
const { add, subtract } = require('./math.js');
console.log(add(1, 2));  // 3
```

#### 内置模块 vs 第三方模块 vs 自定义模块

```javascript
// 内置模块 - 直接 require，不需要安装
const fs = require('fs');
const path = require('path');
const http = require('http');

// 第三方模块 - 需要先 npm install
const express = require('express');
const lodash = require('lodash');

// 自定义模块 - 使用相对路径或绝对路径
const myModule = require('./myModule');
const utils = require('../utils');
```

---

### 2.2 核心模块详解

Node.js 提供了丰富的内置模块，无需安装即可使用。

#### fs（文件系统）

```javascript
const fs = require('fs');
const fsPromises = require('fs').promises;

// 同步读取
const data = fs.readFileSync('file.txt', 'utf8');

// 异步读取（回调）
fs.readFile('file.txt', 'utf8', (err, data) => {
  if (err) {
    console.error('读取失败：', err);
    return;
  }
  console.log('文件内容：', data);
});

// 异步读取（Promise）
async function readFile() {
  try {
    const data = await fsPromises.readFile('file.txt', 'utf8');
    console.log('文件内容：', data);
  } catch (err) {
    console.error('读取失败：', err);
  }
}

// 写入文件
fs.writeFileSync('output.txt', 'Hello Node.js');

// 追加内容
fs.appendFileSync('output.txt', '\nNew line');

// 检查文件是否存在
const exists = fs.existsSync('file.txt');

// 创建目录
fs.mkdirSync('new-folder', { recursive: true });

// 读取目录内容
const files = fs.readdirSync('./');
console.log(files);
```

---

#### path（路径处理）

```javascript
const path = require('path');

// 拼接路径（自动处理不同操作系统的路径分隔符）
const fullPath = path.join(__dirname, 'folder', 'file.txt');
// Windows: C:\project\folder\file.txt
// Linux/Mac: /project/folder/file.txt

// 解析路径
const parsed = path.parse('/home/user/file.txt');
console.log(parsed);
// {
//   root: '/',
//   dir: '/home/user',
//   base: 'file.txt',
//   ext: '.txt',
//   name: 'file'
// }

// 获取文件名
console.log(path.basename('/home/user/file.txt'));      // file.txt
console.log(path.basename('/home/user/file.txt', '.txt')); // file

// 获取目录名
console.log(path.dirname('/home/user/file.txt'));       // /home/user

// 获取扩展名
console.log(path.extname('/home/user/file.txt'));       // .txt

// 绝对路径
console.log(path.resolve('file.txt'));                  // 当前目录下的绝对路径

// 相对路径
console.log(path.relative('/data/orange', '/data/pear')); // ../pear
```

---

#### http（创建 Web 服务器）

```javascript
const http = require('http');

// 创建服务器
const server = http.createServer((req, res) => {
  // 设置响应头
  res.setHeader('Content-Type', 'text/html; charset=utf-8');
  
  // 根据 URL 路由
  if (req.url === '/') {
    res.statusCode = 200;
    res.end('<h1>首页</h1>');
  } else if (req.url === '/about') {
    res.statusCode = 200;
    res.end('<h1>关于我们</h1>');
  } else if (req.url === '/api/users') {
    res.setHeader('Content-Type', 'application/json');
    res.statusCode = 200;
    res.end(JSON.stringify({
      users: [
        { id: 1, name: '张三' },
        { id: 2, name: '李四' }
      ]
    }));
  } else {
    res.statusCode = 404;
    res.end('<h1>404 页面不存在</h1>');
  }
});

// 监听端口
const PORT = 3000;
server.listen(PORT, () => {
  console.log(`服务器运行在 http://localhost:${PORT}`);
});
```

---

#### events（事件触发器）

```javascript
const EventEmitter = require('events');

// 创建事件发射器实例
const emitter = new EventEmitter();

// 监听事件
emitter.on('message', (data) => {
  console.log('收到消息：', data);
});

// 监听一次性事件
emitter.once('connect', () => {
  console.log('连接成功（只执行一次）');
});

// 触发事件
emitter.emit('message', 'Hello World');
emitter.emit('connect');
emitter.emit('connect'); // 不会执行

// 移除监听器
const callback = (data) => console.log(data);
emitter.on('event', callback);
emitter.off('event', callback);  // Node.js 10+ 或 removeListener

// 查看监听器数量
console.log(emitter.listenerCount('message'));
```

---

#### stream（流）

流是 Node.js 处理数据的高效方式，特别适合处理大文件。

```javascript
const fs = require('fs');

// 读取流
const readStream = fs.createReadStream('large-file.txt', {
  encoding: 'utf8',
  highWaterMark: 1024  // 每次读取 1KB
});

// 写入流
const writeStream = fs.createWriteStream('output.txt');

// 管道：将读取流的数据直接写入写入流
readStream.pipe(writeStream);

// 监听事件
readStream.on('data', (chunk) => {
  console.log('收到数据块：', chunk.length);
});

readStream.on('end', () => {
  console.log('读取完成');
});

readStream.on('error', (err) => {
  console.error('读取错误：', err);
});

// 手动写入
writeStream.write('Hello ');
writeStream.write('World');
writeStream.end();  // 结束写入
```

---

### 2.3 npm 包管理器

npm（Node Package Manager）是 Node.js 的包管理工具，也是世界上最大的软件注册表。

#### 常用命令

```bash
# 初始化项目（创建 package.json）
npm init
npm init -y  # 使用默认配置

# 安装依赖
npm install lodash
npm i lodash

# 安装开发依赖
npm install --save-dev nodemon
npm i -D nodemon

# 全局安装
npm install -g @vue/cli

# 卸载依赖
npm uninstall lodash

# 更新依赖
npm update lodash

# 安装 package.json 中的所有依赖
npm install
npm i

# 运行脚本
npm run start
npm run build
npm test
```

#### package.json 详解

```json
{
  "name": "my-project",
  "version": "1.0.0",
  "description": "项目描述",
  "main": "index.js",
  "scripts": {
    "start": "node index.js",
    "dev": "nodemon index.js",
    "test": "jest"
  },
  "dependencies": {
    "express": "^4.18.0",
    "lodash": "~4.17.0"
  },
  "devDependencies": {
    "nodemon": "^2.0.0",
    "jest": "^29.0.0"
  },
  "engines": {
    "node": ">=14.0.0"
  }
}
```

**版本号说明：**
- `^4.18.0`：兼容次要版本和补丁版本（4.x.x，但不包括 5.0.0）
- `~4.17.0`：只兼容补丁版本（4.17.x）
- `4.18.0`：固定版本
- `*`：最新版本

---

## 三、Node.js 实践示例

### 3.1 创建 RESTful API

```javascript
// server.js
const http = require('http');
const url = require('url');

// 模拟数据库
let users = [
  { id: 1, name: '张三', age: 25 },
  { id: 2, name: '李四', age: 30 }
];

const server = http.createServer((req, res) => {
  // 设置 CORS 头
  res.setHeader('Access-Control-Allow-Origin', '*');
  res.setHeader('Access-Control-Allow-Methods', 'GET, POST, PUT, DELETE');
  res.setHeader('Content-Type', 'application/json');
  
  const parsedUrl = url.parse(req.url, true);
  const path = parsedUrl.pathname;
  const method = req.method;
  
  // 路由处理
  if (path === '/api/users' && method === 'GET') {
    // 获取所有用户
    res.statusCode = 200;
    res.end(JSON.stringify({ success: true, data: users }));
  } else if (path.match(/\/api\/users\/\d+/) && method === 'GET') {
    // 获取单个用户
    const id = parseInt(path.split('/')[3]);
    const user = users.find(u => u.id === id);
    if (user) {
      res.statusCode = 200;
      res.end(JSON.stringify({ success: true, data: user }));
    } else {
      res.statusCode = 404;
      res.end(JSON.stringify({ success: false, message: '用户不存在' }));
    }
  } else if (path === '/api/users' && method === 'POST') {
    // 创建用户
    let body = '';
    req.on('data', chunk => body += chunk);
    req.on('end', () => {
      const newUser = JSON.parse(body);
      newUser.id = users.length + 1;
      users.push(newUser);
      res.statusCode = 201;
      res.end(JSON.stringify({ success: true, data: newUser }));
    });
  } else if (path.match(/\/api\/users\/\d+/) && method === 'PUT') {
    // 更新用户
    const id = parseInt(path.split('/')[3]);
    let body = '';
    req.on('data', chunk => body += chunk);
    req.on('end', () => {
      const updateData = JSON.parse(body);
      const index = users.findIndex(u => u.id === id);
      if (index !== -1) {
        users[index] = { ...users[index], ...updateData };
        res.statusCode = 200;
        res.end(JSON.stringify({ success: true, data: users[index] }));
      } else {
        res.statusCode = 404;
        res.end(JSON.stringify({ success: false, message: '用户不存在' }));
      }
    });
  } else if (path.match(/\/api\/users\/\d+/) && method === 'DELETE') {
    // 删除用户
    const id = parseInt(path.split('/')[3]);
    const index = users.findIndex(u => u.id === id);
    if (index !== -1) {
      users.splice(index, 1);
      res.statusCode = 200;
      res.end(JSON.stringify({ success: true, message: '删除成功' }));
    } else {
      res.statusCode = 404;
      res.end(JSON.stringify({ success: false, message: '用户不存在' }));
    }
  } else {
    res.statusCode = 404;
    res.end(JSON.stringify({ success: false, message: '接口不存在' }));
  }
});

const PORT = 3000;
server.listen(PORT, () => {
  console.log(`API 服务器运行在 http://localhost:${PORT}`);
});
```

---

### 3.2 使用 Express 框架

Express 是 Node.js 最流行的 Web 框架。

```javascript
// express-server.js
const express = require('express');
const app = express();

// 中间件：解析 JSON 请求体
app.use(express.json());

// 中间件：日志记录
app.use((req, res, next) => {
  console.log(`${new Date().toISOString()} - ${req.method} ${req.url}`);
  next();
});

// 路由
app.get('/', (req, res) => {
  res.json({ message: '欢迎使用 Express API' });
});

app.get('/api/users', (req, res) => {
  res.json({ success: true, data: users });
});

app.get('/api/users/:id', (req, res) => {
  const user = users.find(u => u.id === parseInt(req.params.id));
  if (user) {
    res.json({ success: true, data: user });
  } else {
    res.status(404).json({ success: false, message: '用户不存在' });
  }
});

app.post('/api/users', (req, res) => {
  const newUser = {
    id: users.length + 1,
    ...req.body
  };
  users.push(newUser);
  res.status(201).json({ success: true, data: newUser });
});

app.put('/api/users/:id', (req, res) => {
  const index = users.findIndex(u => u.id === parseInt(req.params.id));
  if (index !== -1) {
    users[index] = { ...users[index], ...req.body };
    res.json({ success: true, data: users[index] });
  } else {
    res.status(404).json({ success: false, message: '用户不存在' });
  }
});

app.delete('/api/users/:id', (req, res) => {
  const index = users.findIndex(u => u.id === parseInt(req.params.id));
  if (index !== -1) {
    users.splice(index, 1);
    res.json({ success: true, message: '删除成功' });
  } else {
    res.status(404).json({ success: false, message: '用户不存在' });
  }
});

// 错误处理中间件
app.use((err, req, res, next) => {
  console.error(err.stack);
  res.status(500).json({ success: false, message: '服务器内部错误' });
});

const PORT = 3000;
app.listen(PORT, () => {
  console.log(`Express 服务器运行在 http://localhost:${PORT}`);
});
```

---

### 3.3 文件上传服务器

```javascript
// upload-server.js
const http = require('http');
const fs = require('fs');
const path = require('path');

const server = http.createServer((req, res) => {
  if (req.url === '/upload' && req.method === 'POST') {
    const boundary = req.headers['content-type'].split('boundary=')[1];
    let data = '';
    
    req.on('data', chunk => {
      data += chunk;
    });
    
    req.on('end', () => {
      // 解析 multipart/form-data（简化版）
      const parts = data.split(`--${boundary}`);
      const filePart = parts.find(p => p.includes('filename='));
      
      if (filePart) {
        const filenameMatch = filePart.match(/filename="(.+)"/);
        const filename = filenameMatch ? filenameMatch[1] : 'uploaded-file';
        
        // 提取文件内容
        const contentStart = filePart.indexOf('\r\n\r\n') + 4;
        const contentEnd = filePart.lastIndexOf('\r\n');
        const fileContent = filePart.slice(contentStart, contentEnd);
        
        // 保存文件
        const uploadPath = path.join(__dirname, 'uploads', filename);
        fs.mkdirSync(path.dirname(uploadPath), { recursive: true });
        fs.writeFileSync(uploadPath, fileContent, 'binary');
        
        res.writeHead(200, { 'Content-Type': 'application/json' });
        res.end(JSON.stringify({
          success: true,
          message: '文件上传成功',
          filename: filename
        }));
      }
    });
  } else if (req.url === '/' && req.method === 'GET') {
    // 返回上传表单
    res.writeHead(200, { 'Content-Type': 'text/html' });
    res.end(`
      <!DOCTYPE html>
      <html>
      <body>
        <h2>文件上传</h2>
        <form action="/upload" method="post" enctype="multipart/form-data">
          <input type="file" name="file" />
          <button type="submit">上传</button>
        </form>
      </body>
      </html>
    `);
  } else {
    res.writeHead(404);
    res.end('Not Found');
  }
});

server.listen(3000, () => {
  console.log('上传服务器运行在 http://localhost:3000');
});
```

---

## 四、Node.js vs 浏览器 JavaScript

### 4.1 核心差异对比

| 特性 | Node.js | 浏览器 JavaScript |
|------|---------|-------------------|
| **运行环境** | 服务器端 | 客户端（浏览器） |
| **主要用途** | 构建服务器、工具脚本、CLI | 网页交互、DOM 操作 |
| **模块系统** | CommonJS (`require`/`module.exports`) | ES Modules (`import`/`export`) |
| **全局对象** | `global` | `window` |
| **DOM 操作** | ❌ 没有 DOM | ✅ 可以操作 DOM |
| **文件系统** | ✅ 可以读写文件 | ❌ 不能直接访问文件系统 |
| **操作系统** | ✅ 可以调用操作系统 API | ❌ 受限的沙箱环境 |
| **网络请求** | ✅ 可以作为服务器接收请求 | ❌ 只能作为客户端发送请求 |
| **数据库** | ✅ 可以连接数据库 | ❌ 不能直接连接数据库 |

---

### 4.2 全局对象差异

```javascript
// ========== Node.js ==========
// 全局对象是 global
global.myVar = 'hello';
console.log(global.myVar);  // hello

// 常用全局对象
console.log(__dirname);   // 当前文件所在目录
console.log(__filename);  // 当前文件的完整路径
console.log(process);     // 进程信息
console.log(Buffer);      // 二进制数据处理

// ========== 浏览器 ==========
// 全局对象是 window
window.myVar = 'hello';
console.log(window.myVar);  // hello

// 常用全局对象
console.log(document);    // DOM 文档对象
console.log(location);    // URL 信息
console.log(history);     // 浏览历史
console.log(navigator);   // 浏览器信息
```

---

### 4.3 模块系统差异

```javascript
// ========== Node.js (CommonJS) ==========
// 导入
const fs = require('fs');
const myModule = require('./myModule');

// 导出
module.exports = { foo, bar };
exports.baz = baz;

// 特点：
// - 同步加载
// - 运行时加载
// - 值拷贝

// ========== 浏览器 (ES Modules) ==========
// 导入
import fs from 'fs';           // 默认导入
import { readFile } from 'fs'; // 命名导入
import * as utils from './utils'; // 命名空间导入

// 导出
export const foo = 'bar';      // 命名导出
export default myFunction;     // 默认导出

// 特点：
// - 异步加载
// - 编译时加载
// - 值引用（实时绑定）
```

**注意：** Node.js 12+ 也支持 ES Modules，但需要将文件后缀改为 `.mjs` 或在 `package.json` 中设置 `"type": "module"`。

---

### 4.4 API 差异

#### Node.js 特有的 API

```javascript
// 文件系统
const fs = require('fs');
fs.readFileSync('file.txt');

// 路径处理
const path = require('path');
path.join(__dirname, 'folder', 'file.txt');

// 操作系统
const os = require('os');
console.log(os.cpus());        // CPU 信息
console.log(os.totalmem());    // 总内存
console.log(os.freemem());     // 空闲内存

// 进程管理
process.exit(1);               // 退出进程
process.env.NODE_ENV;          // 环境变量
process.argv;                  // 命令行参数

// 子进程
const { exec, spawn } = require('child_process');
exec('ls -la', (err, stdout) => {
  console.log(stdout);
});
```

#### 浏览器特有的 API

```javascript
// DOM 操作
document.getElementById('app');
document.querySelector('.class');
element.addEventListener('click', handler);

// BOM 操作
window.alert('Hello');
window.localStorage.setItem('key', 'value');
window.location.href = 'https://example.com';

// 定时器（两者都有，但实现不同）
setTimeout(() => {}, 1000);
setInterval(() => {}, 1000);

// Fetch API（两者都有）
fetch('/api/data')
  .then(res => res.json())
  .then(data => console.log(data));
```

---

### 4.5 this 指向差异

```javascript
// ========== Node.js ==========
// 全局作用域
console.log(this);  // {} (空对象，不是 global)

// 函数内部
function test() {
  console.log(this);  // global (严格模式下是 undefined)
}
test();

// 模块内部
// this 指向 module.exports
console.log(this === module.exports);  // true

// ========== 浏览器 ==========
// 全局作用域
console.log(this);  // window

// 函数内部
function test() {
  console.log(this);  // window (严格模式下是 undefined)
}
test();

// 事件处理函数
document.getElementById('btn').addEventListener('click', function() {
  console.log(this);  // 被点击的元素
});
```

---

### 4.6 运行环境差异

```javascript
// ========== Node.js ==========
// 可以访问操作系统资源
const fs = require('fs');
const os = require('os');

// 可以创建服务器
const http = require('http');
http.createServer((req, res) => {
  res.end('Hello');
}).listen(3000);

// 可以执行系统命令
const { exec } = require('child_process');
exec('node --version', (err, stdout) => {
  console.log(stdout);  // v18.x.x
});

// ========== 浏览器 ==========
// 运行在沙箱环境，安全性限制
// 不能直接访问文件系统
// 不能执行系统命令
// 不能创建服务器监听端口

// 可以操作 DOM
document.title = 'New Title';
document.body.style.backgroundColor = 'red';

// 可以使用浏览器存储
localStorage.setItem('user', JSON.stringify({ name: '张三' }));
const user = JSON.parse(localStorage.getItem('user'));
```

---

### 4.7 事件循环差异

虽然 Node.js 和浏览器都使用事件循环，但实现细节有所不同。

#### 浏览器事件循环

```
┌─────────────────────────────────────────────────────────┐
│                    浏览器事件循环                         │
├─────────────────────────────────────────────────────────┤
│  调用栈 (Call Stack)                                     │
│  微任务队列 (Microtask): Promise.then, MutationObserver  │
│  宏任务队列 (Macrotask): setTimeout, setInterval, I/O    │
│                                                         │
│  流程：                                                  │
│  1. 执行同步代码                                         │
│  2. 执行所有微任务                                       │
│  3. 执行一个宏任务                                       │
│  4. 重复步骤 2-3                                         │
└─────────────────────────────────────────────────────────┘
```

#### Node.js 事件循环

```
┌─────────────────────────────────────────────────────────┐
│                   Node.js 事件循环                        │
├─────────────────────────────────────────────────────────┤
│  1. timers: setTimeout, setInterval                      │
│  2. pending callbacks: 系统操作的回调                     │
│  3. idle, prepare: 内部使用                              │
│  4. poll: 获取新的 I/O 事件                              │
│  5. check: setImmediate                                  │
│  6. close callbacks: socket.on('close', ...)             │
│                                                         │
│  微任务：process.nextTick, Promise.then                  │
│  process.nextTick 优先级高于 Promise                     │
└─────────────────────────────────────────────────────────┘
```

```javascript
// Node.js 特有的定时器
setTimeout(() => console.log('setTimeout'), 0);
setImmediate(() => console.log('setImmediate'));
process.nextTick(() => console.log('nextTick'));
Promise.resolve().then(() => console.log('Promise'));

// 输出顺序：
// nextTick (微任务，优先级最高)
// Promise (微任务)
// setTimeout (timers 阶段)
// setImmediate (check 阶段)
```

---

## 五、使用场景对比

### 5.1 什么时候用 Node.js？

| 场景 | 说明 |
|------|------|
| **Web 服务器** | 构建 API 服务、实时应用（配合 WebSocket） |
| **微服务** | 轻量级、高并发的服务架构 |
| **工具脚本** | 自动化构建、文件处理、代码转换 |
| **CLI 工具** | 命令行工具（如 Vue CLI、Create React App） |
| **实时应用** | 聊天室、在线游戏、股票行情等 |
| **流式处理** | 大文件处理、视频流、日志处理 |
| **中间层** | BFF（Backend For Frontend）、代理服务器 |

### 5.2 什么时候用浏览器 JavaScript？

| 场景 | 说明 |
|------|------|
| **网页交互** | 表单验证、动画效果、用户交互 |
| **DOM 操作** | 动态修改页面内容、样式 |
| **前端框架** | React、Vue、Angular 等框架运行 |
| **数据可视化** | 图表库（ECharts、D3.js） |
| **客户端存储** | LocalStorage、IndexedDB、Cookie |
| **单页应用（SPA）** | 路由切换、状态管理 |

---

## 六、总结

### 6.1 Node.js 核心要点

| 要点 | 内容 |
|------|------|
| **本质** | 基于 V8 引擎的 JavaScript 运行时 |
| **特点** | 单线程、非阻塞 I/O、事件驱动 |
| **模块** | CommonJS 规范（require/module.exports） |
| **核心模块** | fs、path、http、events、stream |
| **包管理** | npm / yarn / pnpm |
| **适用场景** | 服务器、工具脚本、CLI、实时应用 |

### 6.2 Node.js vs 浏览器 JavaScript 对比表

| 对比项 | Node.js | 浏览器 JavaScript |
|--------|---------|-------------------|
| 运行环境 | 服务器 | 浏览器 |
| 全局对象 | `global` | `window` |
| 模块系统 | CommonJS | ES Modules |
| DOM 操作 | ❌ 不支持 | ✅ 支持 |
| 文件系统 | ✅ 支持 | ❌ 不支持 |
| 数据库连接 | ✅ 支持 | ❌ 不支持 |
| 操作系统 API | ✅ 支持 | ❌ 受限 |
| 事件循环 | 6 个阶段 | 宏任务 + 微任务 |
| 主要用途 | 后端服务、工具 | 前端交互、DOM |

### 6.3 学习建议

1. **先学好 JavaScript 基础**：Node.js 和浏览器 JS 语法相同，基础通用
2. **理解异步编程**：Promise、async/await 在两者中都重要
3. **掌握模块系统**：CommonJS 和 ES Modules 都要了解
4. **多写实践项目**：从简单的 CLI 工具到 Web 服务器
5. **了解核心模块**：fs、path、http 是最常用的

Node.js 让 JavaScript 从一门浏览器脚本语言成长为全栈开发语言，掌握 Node.js 意味着你可以用同一种语言完成前后端开发，大大提高开发效率！
