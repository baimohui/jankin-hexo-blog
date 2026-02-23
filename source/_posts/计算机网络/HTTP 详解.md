---
title: HTTP 协议详解
categories:
- 计算机网络
tags:
- HTTP
- HTTPS
- 计算机网络
- 负载均衡
- 代理
---

HTTP（Hyper Text Transfer Protocol，超文本传输协议）是 Web 开发中最重要的协议之一。本文将从基础到进阶，全面讲解 HTTP 协议的核心知识点。<!-- more -->

---

## 一、HTTP 简介

### 1.1 什么是 HTTP？

HTTP 是一种用于从万维网（WWW）服务器传输超文本到本地浏览器的应用层协议，基于 TCP/IP 通信协议来传递数据（HTML 文件、图片文件、查询结果等）。

HTTP 工作于客户端 - 服务端架构上：浏览器作为 HTTP 客户端通过 URL 向 HTTP 服务器发送请求，Web 服务器根据接收到的请求向客户端发送响应信息。

---

### 1.2 HTTP 的主要特点

| 特点 | 说明 |
|------|------|
| **简单快速** | 只需传送请求方法和路径，常用方法有 GET、HEAD、POST |
| **灵活** | 允许传输任意类型的数据，由 Content-Type 标记 |
| **无连接** | 每次连接只处理一个请求，处理完后断开（HTTP/1.1 后默认长连接）|
| **无状态** | 协议对事务处理没有记忆能力，后续请求需要前面的信息时必须重传 |

---

### 1.3 HTTP 发展历程

| 版本 | 发布时间 | 主要特性 |
|------|---------|---------|
| **HTTP/0.9** | 1991 | 只支持 GET 方法；没有首部；只能获取纯文本 |
| **HTTP/1.0** | 1996 | 增加了首部、状态码、权限、缓存；默认短连接 |
| **HTTP/1.1** | 1999 | 默认长连接；强制 Host 首部；管线化；Cache-Control、ETag 等缓存扩展 |
| **HTTP/2** | 2015 | 二进制分帧；多路复用；头部压缩；服务器推送 |
| **HTTP/3** | 2022 | 基于 QUIC 协议；解决队头阻塞；更快的连接建立 |

---

## 二、HTTP 报文

HTTP 报文是 HTTP 通信的信息载体，分为请求报文和响应报文。

### 2.1 请求报文结构

```
请求行     GET /index.html HTTP/1.1
请求头     Host: www.example.com
           User-Agent: Mozilla/5.0
           Accept: text/html
空行       （CRLF）
请求体     （POST/PUT 等请求的数据）
```

**示例：**
```http
GET /api/user/1 HTTP/1.1
Host: www.example.com
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64)
Accept: application/json
Connection: keep-alive

```

---

### 2.2 响应报文结构

```
响应行     HTTP/1.1 200 OK
响应头     Content-Type: application/json
           Content-Length: 123
空行       （CRLF）
响应体     { "id": 1, "name": "张三" }
```

**示例：**
```http
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 45
Date: Wed, 18 Feb 2026 12:00:00 GMT

{"id":1,"name":"张三","email":"zhangsan@example.com"}
```

---

### 2.3 HTTP 请求方法

| 方法 | 说明 | 幂等 | 安全 |
|------|------|------|------|
| **GET** | 请求资源 | ✅ | ✅ |
| **POST** | 提交数据 | ❌ | ❌ |
| **PUT** | 替换资源 | ✅ | ❌ |
| **PATCH** | 部分更新 | ❌ | ❌ |
| **DELETE** | 删除资源 | ✅ | ❌ |
| **HEAD** | 获取头部 | ✅ | ✅ |
| **OPTIONS** | 获取支持的方法 | ✅ | ✅ |

**幂等性**：同一个请求执行多次和仅执行一次的效果完全相同。
**安全性**：请求不会改变服务器状态（只读）。

---

### 2.4 GET vs POST 对比

| 特性 | GET | POST |
|------|-----|------|
| **数据位置** | URL 查询参数 | 请求体 |
| **数据长度** | 受 URL 长度限制（浏览器一般 2KB-8KB）| 无限制 |
| **数据类型** | 只允许 ASCII 字符 | 无限制 |
| **安全性** | 数据在 URL 中，易被缓存/记录 | 数据在请求体中，相对安全 |
| **刷新/后退** | 无害 | 可能重复提交 |
| **书签** | 可以保存为书签 | 不适合 |
| **缓存** | 可被缓存 | 默认不缓存 |

---

### 2.5 PUT vs POST vs PATCH

| 方法 | 语义 | 幂等 | 示例 |
|------|------|------|------|
| **POST** | 在集合中创建资源 | ❌ | `POST /articles` |
| **PUT** | 替换整个资源 | ✅ | `PUT /articles/1` |
| **PATCH** | 部分更新资源 | ❌ | `PATCH /articles/1` |

**示例：**
```javascript
// 原文章
const article = {
  id: 1,
  title: 'HTTP 详解',
  author: '张三',
  content: '...'
};

// PUT：替换整个资源
PUT /articles/1
{
  id: 1,
  title: 'HTTP 详解',
  author: '李四',  // 修改了
  content: '...'
}

// PATCH：只更新部分字段
PATCH /articles/1
{
  author: '李四'
}
```

---

### 2.6 HTTP 响应状态码

状态码由三位数字组成，第一个数字定义了响应的类别：

| 分类 | 说明 |
|------|------|
| **1xx** | 信息性状态码，表示请求已接收，继续处理 |
| **2xx** | 成功状态码，表示请求已成功被服务器接收、理解、并接受 |
| **3xx** | 重定向状态码，表示需要客户端采取进一步的操作才能完成请求 |
| **4xx** | 客户端错误状态码，表示客户端可能发生了错误，妨碍了服务器的处理 |
| **5xx** | 服务器错误状态码，表示服务器在处理请求的过程中有错误或者异常状态发生 |

---

#### 2.6.1 2xx 成功

| 状态码 | 原因短语 | 说明 |
|--------|---------|------|
| **200** | OK | 请求成功 |
| **201** | Created | 资源创建成功 |
| **202** | Accepted | 请求已接受，但还未处理 |
| **204** | No Content | 请求成功，但无返回内容 |
| **206** | Partial Content | 范围请求成功 |

---

#### 2.6.2 3xx 重定向

| 状态码 | 原因短语 | 说明 | 保留方法 |
|--------|---------|------|---------|
| **301** | Moved Permanently | 永久重定向 | 可能变 GET |
| **302** | Found | 临时重定向 | 可能变 GET |
| **303** | See Other | 临时重定向，强制 GET | ❌ 变 GET |
| **304** | Not Modified | 资源未修改，使用缓存 | - |
| **307** | Temporary Redirect | 临时重定向，保留方法 | ✅ |
| **308** | Permanent Redirect | 永久重定向，保留方法 | ✅ |

---

#### 2.6.3 4xx 客户端错误

| 状态码 | 原因短语 | 说明 |
|--------|---------|------|
| **400** | Bad Request | 请求语法错误 |
| **401** | Unauthorized | 需要认证 |
| **403** | Forbidden | 服务器拒绝请求 |
| **404** | Not Found | 资源不存在 |
| **405** | Method Not Allowed | 方法不允许 |
| **408** | Request Timeout | 请求超时 |
| **409** | Conflict | 资源冲突 |
| **413** | Payload Too Large | 请求体过大 |
| **429** | Too Many Requests | 请求过多（限流）|

---

#### 2.6.4 5xx 服务器错误

| 状态码 | 原因短语 | 说明 |
|--------|---------|------|
| **500** | Internal Server Error | 服务器内部错误 |
| **501** | Not Implemented | 服务器不支持该功能 |
| **502** | Bad Gateway | 网关错误 |
| **503** | Service Unavailable | 服务器暂时不可用 |
| **504** | Gateway Timeout | 网关超时 |

---

### 2.7 HTTP 首部字段

#### 2.7.1 通用首部

| 首部 | 说明 |
|------|------|
| **Cache-Control** | 控制缓存行为 |
| **Connection** | 连接管理（keep-alive、close）|
| **Date** | 报文创建时间 |
| **Transfer-Encoding** | 传输编码（chunked）|
| **Upgrade** | 升级协议 |
| **Via** | 代理服务器信息 |

---

#### 2.7.2 请求首部

| 首部 | 说明 |
|------|------|
| **Accept** | 客户端可接受的媒体类型 |
| **Accept-Encoding** | 客户端可接受的编码 |
| **Accept-Language** | 客户端可接受的语言 |
| **Authorization** | 认证信息（Token）|
| **Cookie** | Cookie |
| **Host** | 请求的主机名 |
| **If-Modified-Since** | 比较资源修改时间 |
| **If-None-Match** | 比较 ETag |
| **Range** | 范围请求 |
| **Referer** | 来源页面 |
| **User-Agent** | 用户代理信息 |

---

#### 2.7.3 响应首部

| 首部 | 说明 |
|------|------|
| **Access-Control-Allow-Origin** | CORS 允许的源 |
| **Cache-Control** | 缓存控制 |
| **Content-Encoding** | 内容编码 |
| **Content-Length** | 内容长度 |
| **Content-Type** | 内容类型 |
| **ETag** | 资源唯一标识 |
| **Last-Modified** | 最后修改时间 |
| **Location** | 重定向地址 |
| **Set-Cookie** | 设置 Cookie |
| **Server** | 服务器信息 |

---

### 2.8 常用 Content-Type

| MIME 类型 | 说明 |
|-----------|------|
| `text/html` | HTML 文档 |
| `text/plain` | 纯文本 |
| `text/css` | CSS 样式 |
| `application/javascript` | JavaScript |
| `application/json` | JSON 数据 |
| `application/xml` | XML 数据 |
| `application/x-www-form-urlencoded` | 表单数据 |
| `multipart/form-data` | 表单文件上传 |
| `image/png` | PNG 图片 |
| `image/jpeg` | JPEG 图片 |

---

## 三、URI & URL

### 3.1 URI vs URL

| 概念 | 说明 |
|------|------|
| **URI** | Uniform Resource Identifier，统一资源标识符，用来唯一标识一个资源 |
| **URL** | Uniform Resource Locator，统一资源定位符，是 URI 的子集，不仅标识资源，还提供了定位方式 |

**关系**：所有的 URL 都是 URI，但不是所有的 URI 都是 URL。

---

### 3.2 URL 结构

```
协议://用户名:密码@主机名:端口/路径?查询#片段
  ↓      ↓    ↓    ↓    ↓  ↓   ↓
http://user:pass@www.example.com:8080/path/to/file?a=1&b=2#section
```

| 部分 | 说明 | 示例 |
|------|------|------|
| **协议** | 使用的协议 | `http://` |
| **用户名密码** | 认证信息（可选）| `user:pass@` |
| **主机名** | 服务器域名或 IP | `www.example.com` |
| **端口** | 端口号（可选）| `:8080` |
| **路径** | 资源路径 | `/path/to/file` |
| **查询** | 查询参数 | `?a=1&b=2` |
| **片段** | 锚点（不发送到服务器）| `#section` |

---

### 3.3 URL 编码

URL 只能使用 ASCII 字符集，非 ASCII 字符需要编码：

- 使用 `%` 跟随两位十六进制数
- 空格通常替换为 `+` 或 `%20`

**示例：**
```
原始：https://example.com/search?keyword=前端开发
编码：https://example.com/search?keyword=%E5%89%8D%E7%AB%AF%E5%BC%80%E5%8F%91
```

---

## 四、HTTP 数据传输

### 4.1 内容编码

内容编码用于压缩报文实体内容，提高传输效率。

#### 4.1.1 编码流程

```
服务器：原始内容 → 压缩（gzip）→ 发送
客户端：接收 → 解压 → 原始内容
```

#### 4.1.2 常用编码算法

| 算法 | 说明 | 压缩率 |
|------|------|--------|
| **gzip** | GNU zip，最常用 | 高 |
| **deflate** | zlib 格式 | 中 |
| **br** | Brotli，Google 开发 | 更高 |
| **compress** | Unix 压缩 | 低 |
| **identity** | 不编码（默认）| - |

#### 4.1.3 相关首部

```http
请求：
Accept-Encoding: gzip, deflate, br

响应：
Content-Encoding: gzip
Content-Length: 1234
```

#### 4.1.4 Node.js 开启 gzip 示例

```javascript
const express = require('express');
const compression = require('compression');
const app = express();

app.use(compression()); // 开启 gzip

app.get('/', (req, res) => {
  res.send('Hello World');
});

app.listen(3000);
```

---

### 4.2 传输编码

传输编码用于改变报文的传输方式，主要是分块传输。

#### 4.2.1 长连接 vs 短连接

| 特性 | 短连接 | 长连接 |
|------|--------|--------|
| HTTP/1.0 | 默认 | 需要 Connection: keep-alive |
| HTTP/1.1 | 需要 Connection: close | 默认 |
| 握手开销 | 每次请求都握手 | 一次握手多次请求 |
| 适用场景 | 请求少 | 请求多 |

**长连接优点：**
- 减少 CPU 和内存使用
- 降低拥塞控制
- 减少后续请求延迟

---

#### 4.2.2 Content-Length

Content-Length 用于指定响应体的长度：

```javascript
const http = require('http');

const server = http.createServer((req, res) => {
  const body = 'Hello World';
  res.setHeader('Content-Type', 'text/plain');
  res.setHeader('Content-Length', Buffer.byteLength(body));
  res.end(body);
});

server.listen(3000);
```

---

#### 4.2.3 分块传输（Transfer-Encoding: chunked）

当内容长度不确定时，使用分块传输：

```http
HTTP/1.1 200 OK
Content-Type: text/plain
Transfer-Encoding: chunked

b\r\n
Hello World\r\n
3\r\n
123\r\n
0\r\n
\r\n
```

**分块规则：**
1. 每个分块包含 16 进制的长度值和真实数据
2. 长度值独占一行，用 CRLF（\r\n）分隔
3. 最后以长度为 0 的分块标记结束

**Node.js 示例：**
```javascript
const http = require('http');

const server = http.createServer((req, res) => {
  res.setHeader('Content-Type', 'text/html; charset=utf-8');
  res.setHeader('Transfer-Encoding', 'chunked');

  res.write('开始传输...<br/>');

  setTimeout(() => {
    res.write('第一块数据<br/>');
  }, 1000);

  setTimeout(() => {
    res.write('第二块数据<br/>');
    res.end();
  }, 2000);
});

server.listen(3000);
```

---

## 五、HTTP 缓存

### 5.1 缓存的作用

- **减少网络传输**：缓存命中时不请求服务器
- **降低服务器负载**：减少请求数量
- **提升用户体验**：页面加载更快

---

### 5.2 缓存分类

| 分类 | 说明 |
|------|------|
| **强制缓存** | 直接使用本地缓存，不请求服务器 |
| **协商缓存** | 询问服务器缓存是否过期 |

---

### 5.3 强制缓存

#### 5.3.1 Expires (HTTP/1.0)

指定缓存过期的绝对时间：

```http
Expires: Wed, 18 Feb 2026 12:00:00 GMT
```

**缺点**：依赖客户端时间，可能不准确。

---

#### 5.3.2 Cache-Control (HTTP/1.1)

更灵活的缓存控制：

| 值 | 说明 |
|----|------|
| `max-age=3600` | 缓存 3600 秒 |
| `no-cache` | 强制协商缓存 |
| `no-store` | 不缓存 |
| `public` | 可被任何中间节点缓存 |
| `private` | 只能被浏览器缓存 |
| `must-revalidate` | 缓存过期后必须验证 |

**示例：**
```http
Cache-Control: public, max-age=3600
```

---

### 5.4 协商缓存

#### 5.4.1 Last-Modified / If-Modified-Since

基于最后修改时间：

```http
响应：
Last-Modified: Wed, 18 Feb 2026 10:00:00 GMT

请求：
If-Modified-Since: Wed, 18 Feb 2026 10:00:00 GMT

响应（未修改）：
HTTP/1.1 304 Not Modified
```

---

#### 5.4.2 ETag / If-None-Match

基于资源唯一标识：

```http
响应：
ETag: "abc123"

请求：
If-None-Match: "abc123"

响应（未修改）：
HTTP/1.1 304 Not Modified
```

**ETag 优势：**
- 精度更高（秒级以下）
- 可以区分内容相同但修改时间不同的情况

---

### 5.5 缓存流程图

```
发起请求
   ↓
有强制缓存？
   ├─ 是 → 未过期？
   │        ├─ 是 → 使用缓存（200 from disk cache）
   │        └─ 否 → 协商缓存
   └─ 否 → 协商缓存
              ↓
         有 ETag/Last-Modified？
              ├─ 是 → 发送 If-None-Match/If-Modified-Since
              │        ↓
              │     304？
              │        ├─ 是 → 使用缓存（200 from disk cache）
              │        └─ 否 → 返回新内容（200）
              └─ 否 → 返回新内容（200）
```

---

## 六、HTTP/2

### 6.1 HTTP/1.1 的问题

| 问题 | 说明 |
|------|------|
| **队头阻塞** | 一个请求慢会阻塞后面的请求 |
| **多个 TCP 连接** | 浏览器限制并发 6-8 个连接 |
| **头部冗余** | 每个请求都带重复的头部 |
| **客户端主动请求** | 服务器不能主动推送 |

---

### 6.2 HTTP/2 的核心改进

#### 6.2.1 二进制分帧层

HTTP/2 采用二进制格式而非文本格式：

| 概念 | 说明 |
|------|------|
| **帧（Frame）** | 最小通信单位 |
| **消息（Message）** | 一个完整的请求或响应，由一个或多个帧组成 |
| **流（Stream）** | TCP 连接上的双向字节流，可承载多个消息 |

---

#### 6.2.2 多路复用

所有通信在一个 TCP 连接上完成，真正实现并发：

```
HTTP/1.1：
连接1: [请求1] [响应1]
连接2:     [请求2]     [响应2]
连接3:         [请求3]         [响应3]

HTTP/2：
连接: [请求1][请求2][请求3]
      [响应1][响应2][响应3]
```

---

#### 6.2.3 头部压缩（HPACK）

使用静态字典和动态字典压缩头部：

- 静态字典：预定义常见头部
- 动态字典：动态添加的头部
- Huffman 编码：进一步压缩

**示例：**
```
method: GET → 静态字典索引（1 字节）
user-agent: xxx → 首次传完整，后续传索引
```

---

#### 6.2.4 服务器推送

服务器可以主动推送资源：

```
客户端请求 index.html
   ↓
服务器返回 index.html
   ↓
服务器主动推送 style.css、script.js
```

---

### 6.3 HTTP/1.1 vs HTTP/2 对比

| 特性 | HTTP/1.1 | HTTP/2 |
|------|----------|--------|
| **传输格式** | 文本 | 二进制 |
| **多路复用** | ❌ | ✅ |
| **头部压缩** | ❌ | ✅ |
| **服务器推送** | ❌ | ✅ |
| **队头阻塞** | ✅（TCP 层）| ❌（应用层）|

---

## 七、HTTPS

### 7.1 HTTP vs HTTPS

| 特性 | HTTP | HTTPS |
|------|------|-------|
| **端口** | 80 | 443 |
| **安全性** | 明文传输 | 加密传输 |
| **证书** | 不需要 | 需要 SSL/TLS 证书 |
| **协议** | HTTP | HTTP + TLS/SSL |

---

### 7.2 加密方式

#### 7.2.1 对称加密

- 使用同一个密钥加密和解密
- 优点：速度快
- 缺点：密钥分发困难

**示例：AES、DES**

---

#### 7.2.2 非对称加密

- 使用公钥和私钥
- 公钥加密，私钥解密
- 优点：安全
- 缺点：速度慢

**示例：RSA、ECC**

---

### 7.3 HTTPS 的工作原理

HTTPS 结合对称加密和非对称加密：

```
1. 客户端 → 服务器：请求 HTTPS，支持的加密套件
2. 服务器 → 客户端：返回证书（包含公钥）
3. 客户端验证证书
4. 客户端生成随机对称密钥，用服务器公钥加密
5. 客户端 → 服务器：加密后的对称密钥
6. 服务器用私钥解密，得到对称密钥
7. 双方使用对称密钥通信
```

---

### 7.4 TLS 1.2 握手流程

```
┌─────────────┐                    ┌─────────────┐
│   Client    │                    │   Server    │
└──────┬──────┘                    └──────┬──────┘
       │     1. Client Hello              │
       │  (随机数、加密套件)              │
       │──────────────────────────────────>│
       │                                   │
       │     2. Server Hello              │
       │  (随机数、选择的加密套件)         │
       │<──────────────────────────────────│
       │                                   │
       │     3. Certificate               │
       │  (服务器证书)                     │
       │<──────────────────────────────────│
       │                                   │
       │     4. Server Key Exchange       │
       │  (可选)                           │
       │<──────────────────────────────────│
       │                                   │
       │     5. Server Hello Done         │
       │<──────────────────────────────────│
       │                                   │
       │     6. Client Key Exchange       │
       │  (预主密钥，用服务器公钥加密)     │
       │──────────────────────────────────>│
       │                                   │
       │     7. Change Cipher Spec        │
       │  (切换到加密通信)                 │
       │──────────────────────────────────>│
       │                                   │
       │     8. Finished                  │
       │  (加密的握手信息摘要)             │
       │──────────────────────────────────>│
       │                                   │
       │     9. Change Cipher Spec        │
       │<──────────────────────────────────│
       │                                   │
       │     10. Finished                 │
       │<──────────────────────────────────│
       │                                   │
       │     11. 应用数据加密传输          │
       │<──────────────────────────────────>│
```

---

### 7.5 证书与 CA

**证书内容：**
- 签发者
- 证书用途
- 使用者公钥
- 证书到期时间
- 数字签名

**验证流程：**
1. 浏览器用 CA 公钥解密证书中的数字签名
2. 浏览器计算证书内容的 Hash
3. 对比两者，一致则证书有效

---

### 7.6 会话恢复

#### 7.6.1 Session ID

- 服务器保存会话信息
- 客户端发送 Session ID
- 缺点：只能在单台服务器上恢复

---

#### 7.6.2 Session Ticket

- 服务器加密会话信息发给客户端
- 客户端下次请求时带回
- 优点：支持多服务器

---

## 八、Web 即时通讯

### 8.1 方案对比

| 方案 | 原理 | 优点 | 缺点 | 适用场景 |
|------|------|------|------|---------|
| **短轮询** | 定时请求 | 简单 | 浪费资源 | 实时性要求低 |
| **长轮询** | 挂起请求 | 减少请求 | 连接挂起 | 实时性要求中 |
| **SSE** | 服务器推送 | 简单、标准 | 只能服务器到客户端 | 单向推送 |
| **WebSocket** | 全双工 | 双向、实时 | 复杂 | 实时聊天 |

---

### 8.2 短轮询

```javascript
// 客户端
setInterval(() => {
  fetch('/api/messages')
    .then(res => res.json())
    .then(data => console.log(data));
}, 3000); // 每 3 秒请求一次
```

---

### 8.3 长轮询

```javascript
// 客户端
async function longPolling() {
  try {
    const res = await fetch('/api/messages');
    const data = await res.json();
    console.log(data);
    longPolling(); // 立即发起下一次请求
  } catch (err) {
    setTimeout(longPolling, 1000);
  }
}

longPolling();
```

```javascript
// 服务器（Node.js）
app.get('/api/messages', async (req, res) => {
  while (true) {
    const messages = await getNewMessages();
    if (messages.length > 0) {
      return res.json(messages);
    }
    await sleep(1000);
  }
});
```

---

### 8.4 SSE (Server-Sent Events)

```javascript
// 客户端
const eventSource = new EventSource('/api/events');

eventSource.onmessage = (event) => {
  console.log('收到消息：', event.data);
};

eventSource.onerror = (err) => {
  console.error('错误：', err);
};
```

```javascript
// 服务器（Node.js）
app.get('/api/events', (req, res) => {
  res.setHeader('Content-Type', 'text/event-stream');
  res.setHeader('Cache-Control', 'no-cache');
  res.setHeader('Connection', 'keep-alive');

  setInterval(() => {
    res.write(`data: ${new Date().toISOString()}\n\n`);
  }, 1000);
});
```

---

### 8.5 WebSocket

```javascript
// 客户端
const ws = new WebSocket('ws://localhost:3000');

ws.onopen = () => {
  console.log('连接成功');
  ws.send('Hello Server');
};

ws.onmessage = (event) => {
  console.log('收到：', event.data);
};

ws.onclose = () => {
  console.log('连接关闭');
};
```

```javascript
// 服务器（Node.js + ws）
const WebSocket = require('ws');
const wss = new WebSocket.Server({ port: 3000 });

wss.on('connection', (ws) => {
  console.log('客户端连接');

  ws.on('message', (message) => {
    console.log('收到：', message);
    ws.send(`服务器收到：${message}`);
  });

  ws.on('close', () => {
    console.log('客户端断开');
  });
});
```

---

## 九、代理与负载均衡

### 9.1 正向代理 vs 反向代理

| 特性 | 正向代理 | 反向代理 |
|------|---------|---------|
| **代理对象** | 代理客户端 | 代理服务器 |
| **隐藏对象** | 隐藏真实客户端 | 隐藏真实服务器 |
| **用途** | 翻墙、访问控制 | 负载均衡、缓存、安全 |
| **示例** | VPN | Nginx |

---

### 9.2 正向代理

```
客户端 → 正向代理 → 目标服务器
         (隐藏客户端)
```

**用途：**
- 访问被墙网站
- 加速访问
- 控制上网行为

---

### 9.3 反向代理

```
客户端 → 反向代理 → 真实服务器 1
                      真实服务器 2
                      真实服务器 3
         (隐藏服务器)
```

**用途：**
- 负载均衡
- 缓存静态资源
- SSL 终端
- 安全防护

**Nginx 示例：**
```nginx
upstream backend {
    server 127.0.0.1:3000;
    server 127.0.0.1:3001;
    server 127.0.0.1:3002;
}

server {
    listen 80;
    server_name example.com;

    location / {
        proxy_pass http://backend;
    }
}
```

---

### 9.4 负载均衡算法

| 算法 | 说明 | 优点 | 缺点 |
|------|------|------|------|
| **轮询** | 依次分配 | 简单 | 不考虑服务器负载 |
| **加权轮询** | 按权重分配 | 考虑服务器性能 | 配置复杂 |
| **IP Hash** | 按 IP 哈希 | 会话保持 | 可能不均 |
| **最小连接** | 分配给连接最少的 | 负载均衡 | 需要维护连接数 |
| **最小响应时间** | 分配给响应最快的 | 性能最优 | 需要统计响应时间 |

---

### 9.5 队头阻塞解决方案

| 方案 | 说明 |
|------|------|
| **并发连接** | 同一域名建立多个 TCP 连接（Chrome 默认 6 个）|
| **域名分片** | 使用多个二级域名（如 img1.example.com、img2.example.com）|
| **HTTP/2** | 多路复用 |

---

## 十、HTTP 与 TCP 的关系

### 10.1 OSI 七层模型

| 层级 | 协议 | 说明 |
|------|------|------|
| **应用层** | HTTP、FTP、SMTP | 应用程序通信 |
| **表示层** | SSL/TLS | 数据加密、压缩 |
| **会话层** | RPC | 建立、维护会话 |
| **传输层** | TCP、UDP | 端到端传输 |
| **网络层** | IP | 路由选择 |
| **数据链路层** | Ethernet | 介质访问 |
| **物理层** | - | 物理传输 |

---

### 10.2 TCP/IP 四层模型

| 层级 | 协议 |
|------|------|
| **应用层** | HTTP、FTP、DNS |
| **传输层** | TCP、UDP |
| **网络层** | IP |
| **网络接口层** | Ethernet |

---

### 10.3 HTTP 与 TCP 的关系

- HTTP 是应用层协议，基于 TCP
- TCP 是传输层协议，提供可靠传输
- HTTP 使用 TCP 作为传输协议

**关系图：**
```
HTTP（应用层）
   ↓
TCP（传输层）
   ↓
IP（网络层）
   ↓
数据链路层
   ↓
物理层
```

---

## 十一、常见面试题

### 11.1 HTTP 常见问题

**Q1：GET 和 POST 的区别？**

参考 2.4 节的对比表格，从数据位置、长度限制、安全性、缓存等方面回答。

---

**Q2：HTTP 状态码有哪些？**

参考 2.6 节，按 1xx-5xx 分类回答。

---

**Q3：HTTP 缓存机制？**

参考第五章，讲强制缓存（Expires、Cache-Control）和协商缓存（Last-Modified、ETag）。

---

**Q4：HTTP/1.1 和 HTTP/2 的区别？**

参考 6.3 节，讲二进制分帧、多路复用、头部压缩、服务器推送。

---

**Q5：HTTPS 的工作原理？**

参考 7.3-7.4 节，讲对称加密、非对称加密、证书、握手流程。

---

**Q6：正向代理和反向代理的区别？**

参考 9.1 节，对比表格回答。

---

### 11.2 最佳实践

1. **使用 HTTPS**：全站 HTTPS，使用 HSTS
2. **开启缓存**：合理配置 Cache-Control
3. **使用 Gzip/Brotli**：压缩文本资源
4. **升级到 HTTP/2**：享受多路复用等特性
5. **合理使用请求方法**：GET 查询、POST 创建、PUT 更新、DELETE 删除
6. **正确使用状态码**：200 成功、404 不存在、500 服务器错误
7. **API 版本化**：`/api/v1/`、`/api/v2/`

---

## 十二、总结

### 12.1 核心知识点回顾

| 知识点 | 关键内容 |
|--------|---------|
| **HTTP 报文** | 请求行/响应行、头部、空行、体 |
| **请求方法** | GET、POST、PUT、PATCH、DELETE |
| **状态码** | 2xx、3xx、4xx、5xx |
| **缓存** | 强制缓存（Cache-Control）、协商缓存（ETag）|
| **HTTP/2** | 二进制分帧、多路复用、头部压缩、服务器推送 |
| **HTTPS** | 对称 + 非对称加密、证书、TLS 握手 |
| **即时通讯** | 短轮询、长轮询、SSE、WebSocket |
| **代理** | 正向代理、反向代理、负载均衡 |

### 12.2 学习建议

1. **理解基础**：先掌握 HTTP/1.1 的核心概念
2. **动手实践**：用浏览器开发者工具查看 HTTP 请求
3. **关注演进**：了解 HTTP/2、HTTP/3 的改进
4. **结合实际**：在项目中应用缓存、HTTPS 等最佳实践
