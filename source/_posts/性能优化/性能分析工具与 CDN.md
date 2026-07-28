---
title: 性能分析工具与 CDN
categories: 
- 性能优化
tags:
- 性能优化
- 分析工具
- CDN
- Network
- Performance
---

## 一、分析性能工具

### Chrome 开发者工具

#### Network 面板

从 Network 面板上可以看到请求资源 size、时长、数量、接口响应时长、瀑布图等信息。

##### 瀑布图

瀑布图展示了浏览器如何加载资源并渲染成网页。每一行是一次单独的浏览器请求。每一行的宽度代表浏览器发出请求并下载该资源所耗费的时间。

瀑布图颜色说明：

- **DNS Lookup** [深绿色]：DNS 查询，将域名转换成 IP 地址
- **Initial Connection** [橙色]：建立 TCP 连接
- **SSL/TLS Negotiation** [紫色]：建立安全连接的过程
- **Time To First Byte (TTFB)** [绿色]：从请求发送到收到第一个字节的时间
- **Downloading** [蓝色]：浏览器下载资源所用的时间

判断瀑布图是否健康的标准：

- 减小瀑布图的**宽度**：减少所有资源的加载时间
- 降低瀑布图的**高度**：减少请求数量
- 将"开始渲染"线**向左移**：优化资源请求顺序来加快渲染时间

##### 代码利用率（Coverage）

打开 Chrome 开发者工具 → Ctrl+Shift+P → 输入 Coverage → 点击 reload 按钮，可查看 JavaScript 的代码利用率。

#### Performance 面板

可分析 FCP/LCP 时间、请求并发情况、请求发起顺序、JavaScript 执行情况。

### webpack-bundle-analyzer

用于分析 bundle 包中所有模块及其 size：

```bash
npm install --save-dev webpack-bundle-analyzer
```

```js
const BundleAnalyzerPlugin = require('webpack-bundle-analyzer').BundleAnalyzerPlugin;
module.exports = {
  plugins: [new BundleAnalyzerPlugin()],
};
```

构建包完毕后自动弹出窗口展示模块大小分布，可排查出无用或过大的模块。

### Performance Navigation Timing

```js
const [entry] = performance.getEntriesByType('navigation');
console.table(entry.toJSON());
```

常用时间参数：

```
DNS 解析时间：domainLookupEnd - domainLookupStart
TCP 建立连接时间：connectEnd - connectStart
白屏时间：responseStart - navigationStart
DOM 渲染完成时间：domContentLoadedEventEnd - navigationStart
页面 onload 时间：loadEventEnd - navigationStart
```

## 二、CDN（内容分发网络）

### 概述

CDN 将内容分发至全国各地的加速节点，用户就近获取所需内容，避免网络拥堵、地域、运营商等因素带来的访问延迟问题。

传统网站请求过程：用户浏览器 → DNS 解析 → TCP 连接 → HTTP 请求 → 服务器响应

引入 CDN 后：

1. 用户请求经本地 DNS 解析后，将解析权交给 CDN 专用 DNS 服务器
2. CDN DNS 返回全局负载均衡设备 IP
3. 用户向全局负载均衡设备发起请求
4. 设备根据用户 IP 和请求 URL，选择最优的区域负载均衡设备
5. 区域负载均衡设备综合距离、内容、负载等因素，返回最优缓存服务器 IP
6. 用户向缓存服务器发起请求

### CDN 组成

- **中心节点**：CDN 网管中心和全局负载均衡 DNS 重定向解析系统，负责整个 CDN 网络的分发及管理
- **边缘节点**：由负载均衡设备、高速缓存服务器两部分组成

### CDN 核心技术

- **内容发布**：借助索引、缓存、流分裂、组播等技术，将内容投递到最近的服务点
- **内容存储**：包括内容源的存储和 Cache 节点中的存储
- **内容路由**：通过 DNS 重定向机制，在多个远程节点上均衡用户请求
- **内容管理**：通过内部和外部监控，测量端到端性能，保证网络处于最佳运行状态
