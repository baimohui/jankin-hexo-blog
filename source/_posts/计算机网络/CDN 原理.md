---
title: CDN 原理详解
categories:
- 计算机网络
tags:
- CDN
- 计算机网络
- 性能优化
- 缓存
---

CDN（Content Delivery Network，内容分发网络）是现代互联网基础设施的重要组成部分。本文将从概念到实践，全面讲解 CDN 的工作原理、关键技术及应用场景。<!-- more -->

---

## 一、什么是 CDN？

### 1.1 CDN 的定义

> CDN（Content Delivery Network，内容分发网络）是一种分布式服务器网络，通过将内容缓存到全球各地的边缘节点，使用户能够从最近的节点获取内容，从而提高访问速度和用户体验。

**通俗理解：**
> 想象你在北京开了一家餐厅（源站），顾客遍布全国。
> - **没有 CDN**：所有顾客都要来北京吃饭，距离远的顾客要花很长时间
> - **有了 CDN**：在全国各地开分店（边缘节点），顾客就近就餐，速度快多了

---

### 1.2 为什么需要 CDN？

#### 问题一：物理距离导致的延迟

光速是有限的，数据传输需要时间。

| 距离 | 理论延迟（往返 RTT） |
|------|---------------------|
| 北京 → 上海（约 1000km） | ~10ms |
| 北京 → 广州（约 2000km） | ~20ms |
| 北京 → 纽约（约 11000km） | ~110ms |
| 北京 → 悉尼（约 9000km） | ~90ms |

**实际延迟更高**，因为还要考虑：
- 路由器转发延迟
- 网络拥塞
- DNS 解析
- TCP 握手

---

#### 问题二：服务器负载过高

所有用户都访问同一个源站：
- 服务器压力大
- 带宽成本高
- 单点故障风险

---

#### 问题三：网络拥塞

互联网骨干网带宽有限，高峰期容易拥塞。

---

### 1.3 CDN 解决的问题

| 问题 | CDN 的解决方案 |
|------|---------------|
| 物理距离远 | 就近访问边缘节点 |
| 服务器负载高 | 分发流量到多个节点 |
| 带宽成本高 | 边缘节点承担大部分流量 |
| 单点故障 | 多节点冗余，高可用 |
| 网络拥塞 | 智能路由，避开拥塞节点 |

---

## 二、CDN 的核心架构

### 2.1 CDN 架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                          CDN 架构                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│    用户A          用户B          用户C          用户D            │
│      │             │              │              │              │
│      ▼             ▼              ▼              ▼              │
│  ┌────────┐   ┌────────┐    ┌────────┐    ┌────────┐           │
│  │边缘节点1│   │边缘节点2│    │边缘节点3│    │边缘节点4│           │
│  │ (北京) │   │ (上海) │    │ (广州) │    │ (成都) │            │
│  └────┬───┘   └────┬───┘    └────┬───┘    └────┬───┘           │
│       │            │             │             │                │
│       └────────────┴──────┬──────┴─────────────┘                │
│                           │                                     │
│                           ▼                                     │
│                    ┌─────────────┐                              │
│                    │  中间层节点  │                              │
│                    │ (区域中心)   │                              │
│                    └──────┬──────┘                              │
│                           │                                     │
│                           ▼                                     │
│                    ┌─────────────┐                              │
│                    │    源站     │                              │
│                    │ (Origin)    │                              │
│                    └─────────────┘                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

### 2.2 CDN 的核心组件

#### 组件一：边缘节点（Edge Server）

边缘节点是 CDN 的核心，部署在离用户最近的位置。

**特点：**
- 数量多，分布广
- 存储内容的缓存副本
- 直接响应用户请求
- 定期从源站或上级节点更新内容

---

#### 组件二：源站（Origin Server）

源站是内容的原始服务器，存储所有内容的最新版本。

**特点：**
- 内容的权威来源
- 当边缘节点没有缓存时，回源获取
- 可以是自建服务器或云服务器

---

#### 组件三：DNS 系统

CDN 的 DNS 系统负责将用户请求导向最近的边缘节点。

**工作流程：**
1. 用户访问 `www.example.com`
2. DNS 解析到 CDN 的 DNS 服务器
3. CDN DNS 根据用户 IP 返回最近的边缘节点 IP
4. 用户访问边缘节点

---

#### 组件四：内容分发系统

负责将内容从源站分发到各个边缘节点。

**分发方式：**
- **Push（推送）**：主动将内容推送到边缘节点
- **Pull（拉取）**：用户请求时按需拉取

---

### 2.3 CDN 节点层级

```
┌─────────────────────────────────────────────────────────────┐
│                     CDN 节点层级                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Level 1: 边缘节点（Edge Nodes）                             │
│  ├── 部署在用户附近（城市级别）                               │
│  ├── 直接响应用户请求                                        │
│  └── 缓存热门内容                                            │
│                                                             │
│  Level 2: 区域节点（Regional Nodes）                         │
│  ├── 部署在区域中心（省份级别）                               │
│  ├── 汇聚多个边缘节点的请求                                   │
│  └── 缓存更多内容                                            │
│                                                             │
│  Level 3: 中心节点（Central Nodes）                          │
│  ├── 部署在核心城市                                          │
│  ├── 连接源站                                                │
│  └── 缓存全量内容                                            │
│                                                             │
│  Level 4: 源站（Origin Server）                              │
│  └── 内容的原始来源                                          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 三、CDN 的工作原理

### 3.1 用户访问 CDN 的完整流程

```
┌─────────────────────────────────────────────────────────────────┐
│                    CDN 访问流程                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. 用户输入 URL                                                 │
│     └── 用户在浏览器输入 https://www.example.com/image.jpg       │
│                                                                 │
│  2. DNS 解析                                                     │
│     ├── 本地 DNS 缓存查询                                        │
│     ├── 递归查询到权威 DNS                                       │
│     ├── 权威 DNS 返回 CDN 的 DNS 服务器地址                       │
│     └── CDN DNS 返回最近的边缘节点 IP                             │
│                                                                 │
│  3. 建立连接                                                     │
│     ├── TCP 三次握手                                            │
│     └── TLS 握手（HTTPS）                                       │
│                                                                 │
│  4. 请求资源                                                     │
│     └── 浏览器向边缘节点发送 HTTP 请求                            │
│                                                                 │
│  5. 边缘节点处理                                                 │
│     ├── 缓存命中（Cache Hit）→ 直接返回                          │
│     └── 缓存未命中（Cache Miss）→ 回源获取                        │
│                                                                 │
│  6. 返回响应                                                     │
│     └── 边缘节点返回内容给用户，同时缓存                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

### 3.2 DNS 解析详解

CDN 的 DNS 解析是整个流程中最关键的一步。

#### 传统 DNS 解析

```
用户 → 本地 DNS → 根 DNS → 顶级域 DNS → 权威 DNS → 返回 IP
```

#### CDN DNS 解析

```
┌─────────────────────────────────────────────────────────────────┐
│                    CDN DNS 解析流程                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. 用户请求 www.example.com                                     │
│                    │                                            │
│                    ▼                                            │
│  2. 本地 DNS 服务器                                              │
│     └── 查询缓存，没有则向上查询                                  │
│                    │                                            │
│                    ▼                                            │
│  3. 根 DNS 服务器                                                │
│     └── 返回 .com 的顶级域 DNS                                   │
│                    │                                            │
│                    ▼                                            │
│  4. .com 顶级域 DNS                                              │
│     └── 返回 example.com 的权威 DNS                              │
│                    │                                            │
│                    ▼                                            │
│  5. 权威 DNS 服务器                                              │
│     └── 返回 CNAME: www.example.com.cdn.com                     │
│                    │                                            │
│                    ▼                                            │
│  6. CDN DNS 服务器（智能 DNS）                                    │
│     ├── 分析用户 IP 地理位置                                     │
│     ├── 分析网络状况                                             │
│     ├── 分析节点负载                                             │
│     └── 返回最佳边缘节点 IP                                      │
│                    │                                            │
│                    ▼                                            │
│  7. 用户访问边缘节点                                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

### 3.3 智能调度算法

CDN 如何选择最佳边缘节点？

#### 算法一：基于地理位置

根据用户 IP 判断地理位置，选择最近的节点。

```
用户 IP: 202.106.x.x (北京)
    │
    ▼
IP 地理位置库查询 → 北京
    │
    ▼
选择北京边缘节点
```

---

#### 算法二：基于网络质量

实时监测各节点的网络状况，选择延迟最低的节点。

| 指标 | 说明 |
|------|------|
| RTT（往返时延） | 用户到节点的往返时间 |
| 丢包率 | 网络传输的丢包比例 |
| 带宽 | 节点的可用带宽 |
| 负载 | 节点的当前负载 |

---

#### 算法三：基于负载均衡

选择负载较低的节点，避免单点过载。

```
节点负载情况：
├── 北京节点: CPU 80%, 带宽 90%  ← 负载高
├── 天津节点: CPU 30%, 带宽 40%  ← 选择这个
└── 石家庄节点: CPU 50%, 带宽 60%
```

---

### 3.4 缓存机制详解

#### 缓存命中与未命中

```
┌─────────────────────────────────────────────────────────────────┐
│                      缓存处理流程                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  用户请求资源                                                    │
│       │                                                         │
│       ▼                                                         │
│  边缘节点检查缓存                                                │
│       │                                                         │
│       ├── 缓存存在且未过期 ──────────────┐                       │
│       │                                 │                       │
│       │                                 ▼                       │
│       │                         Cache Hit                       │
│       │                         直接返回缓存内容                  │
│       │                         响应时间: ~10-50ms               │
│       │                                                         │
│       └── 缓存不存在或已过期 ────────────┐                       │
│                                          │                      │
│                                          ▼                      │
│                                  Cache Miss                     │
│                                       │                         │
│                                       ▼                         │
│                                  回源获取                        │
│                                       │                         │
│                    ┌──────────────────┼──────────────────┐      │
│                    │                  │                  │      │
│                    ▼                  ▼                  ▼      │
│              上级节点            上级节点            源站        │
│              (如有缓存)          (如有缓存)         (最终来源)   │
│                    │                  │                  │      │
│                    └──────────────────┴──────────────────┘      │
│                                       │                         │
│                                       ▼                         │
│                              获取内容并缓存                       │
│                                       │                         │
│                                       ▼                         │
│                              返回给用户                          │
│                              响应时间: ~100-500ms               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

#### 缓存判断依据

边缘节点如何判断是否使用缓存？

**1. Cache-Control 首部**

```http
Cache-Control: max-age=31536000    // 缓存 1 年
Cache-Control: no-cache            // 使用前需验证
Cache-Control: no-store            // 不缓存
Cache-Control: public              // 可被任何缓存存储
Cache-Control: private             // 只能被浏览器缓存
```

**2. Expires 首部**

```http
Expires: Wed, 21 Oct 2026 07:28:00 GMT  // 过期时间
```

**3. ETag / Last-Modified（协商缓存）**

```http
// 首次响应
ETag: "abc123"
Last-Modified: Wed, 21 Oct 2025 07:28:00 GMT

// 后续请求
If-None-Match: "abc123"
If-Modified-Since: Wed, 21 Oct 2025 07:28:00 GMT

// 源站响应
HTTP/1.1 304 Not Modified  // 未修改，使用缓存
```

---

#### 缓存分级策略

| 内容类型 | 缓存时间 | 更新策略 |
|---------|---------|---------|
| 静态资源（JS/CSS/图片） | 1 年 | 文件名加哈希 |
| HTML 页面 | 0-10 分钟 | 短缓存 + 协商缓存 |
| API 接口 | 不缓存或秒级 | 按需设置 |
| 视频流 | 分片缓存 | 边下边存 |

---

### 3.5 回源机制

当边缘节点没有缓存时，需要回源获取内容。

#### 回源流程

```
┌─────────────────────────────────────────────────────────────────┐
│                       回源流程                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. 边缘节点检查缓存                                             │
│     └── 缓存未命中                                              │
│                                                                 │
│  2. 边缘节点向上级节点请求                                        │
│     ├── 有上级节点 → 请求上级节点                                 │
│     └── 无上级节点 → 直接请求源站                                 │
│                                                                 │
│  3. 获取内容                                                     │
│     └── 上级节点或源站返回内容                                    │
│                                                                 │
│  4. 缓存内容                                                     │
│     └── 边缘节点缓存内容（根据 Cache-Control）                    │
│                                                                 │
│  5. 返回用户                                                     │
│     └── 将内容返回给用户                                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### 回源保护

大量请求同时回源会导致"回源风暴"。

**解决方案：**

```javascript
// 回源合并（请求合并）
// 多个用户同时请求同一资源，只回源一次
const pendingRequests = new Map();

async function fetchWithMerge(key) {
  if (pendingRequests.has(key)) {
    return pendingRequests.get(key);  // 等待正在进行的请求
  }
  
  const promise = fetchFromOrigin(key).finally(() => {
    pendingRequests.delete(key);
  });
  
  pendingRequests.set(key, promise);
  return promise;
}
```

---

## 四、CDN 的关键技术

### 4.1 内容分发技术

#### Push 模式（主动推送）

源站主动将内容推送到边缘节点。

**适用场景：**
- 内容更新频率低
- 文件数量少但体积大
- 需要确保内容实时可用

**示例：**
```
源站发布新版本
    │
    ▼
CDN 管理系统
    │
    ├── 推送到北京节点
    ├── 推送到上海节点
    ├── 推送到广州节点
    └── ...
```

---

#### Pull 模式（按需拉取）

用户请求时，边缘节点按需从源站拉取内容。

**适用场景：**
- 内容更新频率高
- 文件数量多
- 长尾内容（访问量低）

**示例：**
```
用户请求 /images/photo.jpg
    │
    ▼
边缘节点检查缓存
    │
    └── 未命中
        │
        ▼
    从源站拉取
        │
        ▼
    缓存并返回
```

---

### 4.2 负载均衡技术

CDN 需要在多个节点之间分配流量。

#### DNS 负载均衡

通过 DNS 返回不同的 IP 地址。

```
www.example.com
    │
    ├── 用户 A → 1.1.1.1 (北京节点)
    ├── 用户 B → 2.2.2.2 (上海节点)
    └── 用户 C → 1.1.1.1 (北京节点)
```

---

#### 应用层负载均衡

使用 L7 负载均衡器（如 Nginx、HAProxy）。

```nginx
upstream cdn_nodes {
    server 192.168.1.1:80 weight=3;
    server 192.168.1.2:80 weight=2;
    server 192.168.1.3:80 weight=1;
}

server {
    location / {
        proxy_pass http://cdn_nodes;
    }
}
```

---

### 4.3 缓存更新技术

#### 主动刷新

通过 API 主动刷新缓存。

```bash
# 阿里云 CDN 刷新 API
curl -X POST "https://cdn.aliyuncs.com/?Action=RefreshObjectCaches" \
  -d "ObjectPath=https://www.example.com/index.html" \
  -d "ObjectType=File"
```

#### 被动过期

根据 Cache-Control 自动过期。

---

### 4.4 安全防护技术

#### DDoS 防护

CDN 边缘节点可以吸收 DDoS 攻击流量。

```
攻击流量
    │
    ▼
CDN 边缘节点（分散攻击）
    │
    ├── 北京节点吸收 20%
    ├── 上海节点吸收 20%
    ├── 广州节点吸收 20%
    └── ...
    │
    ▼
源站（受保护）
```

#### WAF（Web 应用防火墙）

在边缘节点拦截恶意请求。

```
用户请求
    │
    ▼
CDN WAF 检测
    │
    ├── SQL 注入 → 拦截
    ├── XSS 攻击 → 拦截
    └── 正常请求 → 放行
```

---

## 五、CDN 的应用场景

### 5.1 静态资源加速

最常见的 CDN 应用场景。

**适用内容：**
- JavaScript、CSS 文件
- 图片（PNG、JPG、WebP）
- 字体文件
- 视频文件

**最佳实践：**
```html
<!-- 使用 CDN 加载静态资源 -->
<link rel="stylesheet" href="https://cdn.example.com/css/style.css">
<script src="https://cdn.example.com/js/app.js"></script>
<img src="https://cdn.example.com/images/logo.png">

<!-- 文件名加哈希，实现长期缓存 -->
<link rel="stylesheet" href="https://cdn.example.com/css/style.a1b2c3.css">
<script src="https://cdn.example.com/js/app.d4e5f6.js"></script>
```

---

### 5.2 动态内容加速

通过优化网络路径加速动态内容。

**技术手段：**
- 智能路由选择最优路径
- TCP 优化（减少握手次数）
- HTTP/2、HTTP/3 支持
- 动态压缩

```
用户请求 API
    │
    ▼
CDN 边缘节点（网络入口）
    │
    ▼
CDN 专用网络（优化路径）
    │
    ▼
源站（处理请求）
```

---

### 5.3 视频直播加速

直播场景对延迟要求高。

**技术架构：**
```
主播推流
    │
    ▼
源站（转码、切片）
    │
    ▼
CDN 分发网络
    │
    ├── 边缘节点 1
    ├── 边缘节点 2
    └── 边缘节点 3
    │
    ▼
观众拉流
```

---

### 5.4 文件下载加速

大文件分发场景。

**技术手段：**
- 分片缓存
- 断点续传
- 多线程下载

---

### 5.5 全站加速

同时加速静态和动态内容。

```
┌─────────────────────────────────────────────────────────────────┐
│                      全站加速                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  用户请求                                                        │
│      │                                                          │
│      ▼                                                          │
│  CDN 边缘节点                                                    │
│      │                                                          │
│      ├── 静态资源 (.js, .css, .png)                              │
│      │   └── 直接返回缓存                                        │
│      │                                                          │
│      └── 动态资源 (.php, .jsp, API)                              │
│          └── 通过优化路径转发到源站                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 六、CDN 配置实践

### 6.1 常用 CDN 服务商

| 服务商 | 特点 | 适用场景 |
|--------|------|---------|
| **阿里云 CDN** | 国内节点多，生态完善 | 国内业务 |
| **腾讯云 CDN** | 价格优惠，游戏加速 | 游戏、视频 |
| **Cloudflare** | 全球节点，免费套餐 | 海外业务 |
| **AWS CloudFront** | AWS 生态集成 | AWS 用户 |
| **Akamai** | 全球最大 CDN | 大型企业 |

---

### 6.2 CDN 配置示例

#### 阿里云 CDN 配置

```javascript
// 1. 添加加速域名
{
  "DomainName": "cdn.example.com",
  "SourceType": "oss",           // 源站类型：oss/ip/domain
  "Sources": [{
    "Type": "oss",
    "Content": "example.oss-cn-hangzhou.aliyuncs.com",
    "Priority": 20
  }]
}

// 2. 配置缓存规则
{
  "CacheContent": "/images/*",
  "CacheTTL": 31536000,          // 1 年
  "Weight": 100
}

// 3. 配置 HTTPS
{
  "CertName": "my-cert",
  "CertType": "upload",          // 上传证书
  "SSLProtocol": "on",
  "SSLProtocolVersion": "TLSv1.2,TLSv1.3"
}
```

---

#### Nginx 配置 CDN 回源

```nginx
server {
    listen 80;
    server_name origin.example.com;

    # 设置回源 Host
    set $origin_host "cdn.example.com";

    location / {
        # 允许 CDN 回源
        valid_referers none blocked cdn.example.com *.cdn.example.com;
        
        if ($invalid_referer) {
            return 403;
        }

        # 设置缓存头
        add_header Cache-Control "public, max-age=31536000";
        
        # 静态资源
        location ~* \.(js|css|png|jpg|jpeg|gif|ico|woff|woff2)$ {
            expires 1y;
            add_header Cache-Control "public, immutable";
        }

        # HTML 文件
        location ~* \.html$ {
            expires 10m;
            add_header Cache-Control "public, must-revalidate";
        }
    }
}
```

---

### 6.3 缓存策略最佳实践

```javascript
// Webpack 配置文件名哈希
module.exports = {
  output: {
    filename: '[name].[contenthash:8].js',
    chunkFilename: '[name].[contenthash:8].chunk.js',
  },
  plugins: [
    new MiniCssExtractPlugin({
      filename: '[name].[contenthash:8].css',
    }),
  ],
};

// Nginx 缓存配置
// 长期缓存：文件名带哈希的资源
location ~* \.[a-f0-9]{8}\.(js|css|png|jpg|gif|ico|woff2)$ {
    expires 1y;
    add_header Cache-Control "public, immutable";
}

// 短期缓存：HTML 文件
location ~* \.html$ {
    expires 10m;
    add_header Cache-Control "public, must-revalidate";
}

// 不缓存：API 接口
location /api/ {
    add_header Cache-Control "no-store, no-cache, must-revalidate";
}
```

---

## 七、CDN 性能优化

### 7.1 关键性能指标

| 指标 | 说明 | 目标值 |
|------|------|--------|
| **首字节时间（TTFB）** | 从请求到收到第一个字节 | < 100ms |
| **缓存命中率** | 缓存命中次数 / 总请求次数 | > 95% |
| **回源率** | 回源请求次数 / 总请求次数 | < 5% |
| **下载速度** | 资源下载速度 | > 1MB/s |
| **可用性** | 服务可用时间占比 | > 99.9% |

---

### 7.2 提高缓存命中率的策略

#### 策略一：合理设置缓存时间

```http
// 静态资源：长期缓存
Cache-Control: public, max-age=31536000, immutable

// HTML：短期缓存 + 协商缓存
Cache-Control: public, max-age=600, must-revalidate

// API：不缓存
Cache-Control: no-store
```

---

#### 策略二：使用文件名哈希

```html
<!-- 不推荐：文件名不变，更新后用户可能获取旧版本 -->
<script src="/js/app.js"></script>

<!-- 推荐：文件名带哈希，内容变化时文件名变化 -->
<script src="/js/app.a1b2c3d4.js"></script>
```

---

#### 策略三：预热缓存

在业务高峰前主动预热缓存。

```bash
# 预热常用资源
curl -X POST "https://cdn.api.com/preload" \
  -d "urls=https://cdn.example.com/js/app.js" \
  -d "urls=https://cdn.example.com/css/style.css"
```

---

### 7.3 减少回源请求

#### 方法一：边缘计算

在边缘节点执行逻辑，减少回源。

```javascript
// Cloudflare Worker 示例
addEventListener('fetch', event => {
  event.respondWith(handleRequest(event.request));
});

async function handleRequest(request) {
  const cacheKey = new Request(request.url, request);
  const cache = caches.default;
  
  // 检查缓存
  let response = await cache.match(cacheKey);
  if (response) {
    return response;
  }
  
  // 回源获取
  response = await fetch(request);
  
  // 缓存响应
  const headers = new Headers(response.headers);
  headers.set('Cache-Control', 'public, max-age=3600');
  
  const cachedResponse = new Response(response.body, {
    status: response.status,
    headers: headers
  });
  
  event.waitUntil(cache.put(cacheKey, cachedResponse.clone()));
  
  return cachedResponse;
}
```

---

#### 方法二：智能预取

预测用户行为，提前获取资源。

```html
<!-- 预取下一页资源 -->
<link rel="prefetch" href="/page2.html">

<!-- 预连接 CDN 域名 -->
<link rel="preconnect" href="https://cdn.example.com">

<!-- DNS 预解析 -->
<link rel="dns-prefetch" href="//cdn.example.com">
```

---

## 八、CDN 常见问题与解决方案

### 8.1 缓存更新不及时

**问题：** 更新了源站内容，但用户看到的还是旧内容。

**解决方案：**

```javascript
// 方案一：文件名加版本号/哈希
<script src="/js/app.v2.js"></script>
<script src="/js/app.a1b2c3.js"></script>

// 方案二：主动刷新 CDN 缓存
cdn.refresh('/js/app.js');

// 方案三：使用版本号参数
<script src="/js/app.js?v=2.0.0"></script>
// 注意：这种方式会影响缓存命中率
```

---

### 8.2 跨域问题

**问题：** CDN 资源跨域访问报错。

**解决方案：**

```nginx
# CDN 节点配置 CORS
location / {
    add_header Access-Control-Allow-Origin *;
    add_header Access-Control-Allow-Methods 'GET, OPTIONS';
    add_header Access-Control-Allow-Headers 'Origin, Content-Type, Accept';
    
    if ($request_method = 'OPTIONS') {
        return 204;
    }
}
```

---

### 8.3 HTTPS 证书问题

**问题：** CDN 节点需要配置 HTTPS 证书。

**解决方案：**

```
┌─────────────────────────────────────────────────────────────────┐
│                   HTTPS 证书配置                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  用户 ────HTTPS────▶ CDN 边缘节点 ────HTTP/HTTPS────▶ 源站       │
│                         │                                       │
│                    需要证书                                      │
│                                                                 │
│  方案一：上传自有证书到 CDN                                       │
│  方案二：使用 CDN 提供的免费证书（如 Cloudflare）                  │
│  方案三：使用 Let's Encrypt 自动签发                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

### 8.4 CDN 节点故障

**问题：** 某个 CDN 节点不可用。

**解决方案：**

```javascript
// 方案一：CDN 自动切换（健康检查）
// CDN 会自动检测节点健康状态，故障时切换到其他节点

// 方案二：多 CDN 策略
const cdnDomains = [
  'https://cdn1.example.com',
  'https://cdn2.example.com',
  'https://cdn3.example.com'
];

function loadResource(path) {
  return Promise.race(
    cdnDomains.map(domain => 
      fetch(`${domain}${path}`).catch(() => null)
    )
  ).then(response => {
    if (!response) throw new Error('All CDN failed');
    return response;
  });
}
```

---

## 九、CDN 与其他技术的对比

### 9.1 CDN vs 传统服务器

| 对比项 | CDN | 传统服务器 |
|--------|-----|-----------|
| 节点分布 | 全球分布，就近访问 | 单一位置，所有用户访问同一服务器 |
| 响应速度 | 快（就近访问） | 慢（距离远） |
| 负载能力 | 高（分布式） | 有限（单点） |
| 成本 | 按流量计费 | 固定成本 + 带宽成本 |
| 适用场景 | 静态资源、高并发 | 动态内容、数据库操作 |

---

### 9.2 CDN vs 对象存储

| 对比项 | CDN | 对象存储（OSS/S3） |
|--------|-----|-------------------|
| 主要功能 | 内容分发、加速 | 文件存储 |
| 节点分布 | 边缘节点多 | 区域节点少 |
| 访问速度 | 快（缓存） | 较慢（直接访问） |
| 存储成本 | 高（缓存多份） | 低（存储一份） |
| 典型用法 | 配合对象存储使用 | 作为 CDN 的源站 |

**最佳实践：对象存储 + CDN**

```
用户 → CDN 边缘节点 → 对象存储（源站）
```

---

## 十、总结

### 10.1 CDN 核心要点

| 要点 | 内容 |
|------|------|
| **定义** | 分布式内容分发网络，就近提供服务 |
| **核心组件** | 边缘节点、源站、DNS 系统、分发系统 |
| **工作原理** | DNS 解析 → 就近访问 → 缓存命中/回源 |
| **关键技术** | 智能调度、缓存机制、负载均衡、安全防护 |
| **应用场景** | 静态资源、动态内容、视频直播、文件下载 |

### 10.2 CDN 配置最佳实践

| 资源类型 | 缓存策略 | Cache-Control |
|---------|---------|---------------|
| JS/CSS（带哈希） | 1 年 | `public, max-age=31536000, immutable` |
| 图片 | 1 年 | `public, max-age=31536000` |
| HTML | 10 分钟 | `public, max-age=600, must-revalidate` |
| API | 不缓存 | `no-store` |

### 10.3 学习建议

1. **理解核心原理**：DNS 解析、缓存机制、回源流程
2. **掌握配置方法**：缓存策略、HTTPS 配置、跨域处理
3. **关注性能指标**：缓存命中率、TTFB、回源率
4. **实践优化技巧**：文件名哈希、预热缓存、多 CDN 策略
5. **了解安全防护**：DDoS 防护、WAF、HTTPS

CDN 是现代 Web 应用不可或缺的基础设施，掌握 CDN 原理和配置对于前端和运维工程师都至关重要！
