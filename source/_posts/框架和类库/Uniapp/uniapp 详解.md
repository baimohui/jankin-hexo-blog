---
title: Uniapp 详解
categories: 
- uniapp
tags:
- uniapp
- 跨平台
- Vue
- 小程序
- APP
---

# Uniapp 详解

## 【面试速答版】

### Q1: 什么是 uni-app？它解决了什么问题？

uni-app 是一个使用 **Vue.js** 开发所有前端应用的**跨平台框架**，由 DCloud 团队开发。开发者编写一套代码，可发布到 iOS、Android、Web（H5），以及各种小程序（微信/支付宝/百度/字节/QQ/京东/快手/飞书/小红书等）。

它解决的核心问题是**多端重复开发**。在 uni-app 出现之前，一个产品要同时覆盖微信小程序、App、H5，通常需要 3 个独立团队维护 3 套代码，或使用各平台自研的 DSL（如微信小程序的 WXML/WXSS）。这不仅开发成本高，而且多端功能对齐困难、迭代节奏不一致。

uni-app 的核心思路是：**以 Vue.js 作为统一的 DSL，在编译时将同一套代码编译为各端的目标代码**。开发者只需关注业务逻辑，框架负责处理平台差异。

### Q2: uni-app 的跨平台原理是什么？

uni-app 采用**编译时 + 运行时**结合的跨平台方案：

**编译时（静态转换）**：
- uni-app 将 `.vue` 单文件组件（template/script/style）编译为各端的目标代码
- 对于小程序端，`<template>` 编译为对应平台的模板（微信的 WXML、支付宝的 AXML 等），`<style>` 编译为对应的样式语言（WXSS/ACSS 等）
- 对于 App 端，编译为 Vue 组件后通过 WebView 或 Weex 引擎渲染
- 对于 H5 端，直接输出标准的 Vue.js 应用

**运行时（动态适配）**：
- uni-app 提供了统一的 API 层，如 `uni.request()`、`uni.navigateTo()`、`uni.getStorage()` 等
- 这些 API 在**编译时**会根据 target 平台替换为对应的平台实现（微信的 `wx.request()`、支付宝的 `my.httpRequest()`、H5 的 `fetch()`）
- 框架内置了条件编译机制，允许开发者针对特定平台编写差异化代码

**注意**：uni-app 不是「一次编写，到处不调优」的工具，它是「一次编写，大部分一致，少部分按平台优化」。实际上有价值的跨平台项目，通常都会用到条件编译来处理平台差异。

### Q3: uni-app 与 React Native / Flutter / Taro 等竞品对比有什么优劣势？

| 维度 | uni-app | React Native | Flutter | Taro |
|:---|:---|:---|:---|:---|
| **基础语言** | Vue.js | React | Dart | React（类 React 语法） |
| **小程序支持** | ✅ 全平台（约 20+） | ❌ 无原生支持 | ❌ 无原生支持 | ✅ 多平台 |
| **App 性能** | ⚠️ WebView/Weex（中等） | ✅ JSI 桥接（接近原生） | ✅ 自渲染引擎（接近原生） | ⚠️ 依赖 Taro 的 RN 桥接 |
| **H5 支持** | ✅ 原生支持 | ⚠️ 需 react-native-web | ❌ 需第三方适配 | ✅ 原生支持 |
| **学习成本** | ★★（会 Vue 就会） | ★★★（需理解 RN 桥接） | ★★★★（Dart + 新框架） | ★★（会 React 就会） |
| **生态成熟度** | ★★★★（国内） | ★★★★★（国际） | ★★★★（增长快） | ★★★（中等） |
| **热更新** | ✅ 小程序/H5 原生支持 | ⚠️ 需 CodePush | ⚠️ 需三方方案 | ✅ 同 uni-app |

**选型建议**：
- 目标以**国内小程序为主、兼有 App/H5** → **uni-app**（小程序支持最全）
- 目标以**国际 App 为主**、团队熟悉 React → **React Native**
- 对**性能要求极高**（如动画、游戏）→ **Flutter**
- 团队熟悉 React、但需要小程序支持 → **Taro**

---

## 【深入理解版】

### 1. uni-app 要解决什么问题？

#### 1.1 多端开发之痛：一个产品，三套代码

假设你是一家创业公司的前端负责人，产品需要同时覆盖三个渠道：

```
微信小程序 → 用户基数大，社交裂变
iOS/Android App → 体验好，留存高
H5 网页 → SEO、分享、桌面入口
```

没有跨平台框架的时代，典型的项目结构是这样的：

```
project/
├── weapp/              # 微信小程序团队（3人）
│   ├── pages/
│   ├── app.wxss
│   └── app.json
├── app/                # App 团队（3人）
│   ├── src/            # iOS（Swift）
│   └── src-android/    # Android（Kotlin/Java）
└── h5/                 # H5 团队（2人）
    ├── src/
    └── package.json
```

**问题 1：技术栈不同，人员难复用**

- 小程序团队：Vue/React + WXML + WXSS
- App 团队：Swift + Java/Kotlin
- H5 团队：Vue/React

三端用三种完全不同的技术栈，前端工程师不能在团队间流动。同一个业务需求，需要三端同时排期开发，一个端延迟，整个 feature 延期。

**问题 2：功能对齐成本高**

假设你设计了一个「购物车页面」：

- **H5 版本**：Vue Router 导航，localStorage 存购物车数据
- **微信小程序版本**：`wx.navigateTo` 跳转，`wx.setStorageSync` 存数据
- **App 版本**：用 Coordinator 模式跳转，SQLite 或 SharedPreferences 存数据

每次 UI 改版或交互变更，都需要三端分别修改。上线后发现三端的购物车逻辑不一致的概率极高。

**问题 3：BUG 需要在三端分别修复**

```
H5 发现了一个「库存不足提示不显示」的 BUG
→ H5 前端修复、测试、上线
→ 同步修改微信小程序版本
→ 同步修改 App 版本
→ 同样的 BUG，修了三遍
```

#### 1.2 现有跨平台方案各自的局限

在 uni-app 之前，业界已有的跨平台方案：

**方案 1：纯 H5 套壳（Cordova/PhoneGap）**

把 H5 页面用 WebView 包装成 App。问题是：
- 性能差，长列表滚动卡顿
- 无法调用原生能力（或调用方式别扭）
- 离线能力弱

**方案 2：React Native / Weex**

用 JS 写 UI，通过 Bridge 调用原生组件。问题是：
- 不支持小程序（当时小程序正在爆发）
- 技术栈锁定 React（RN）或 Vue（Weex）
- 桥接通信有性能瓶颈，复杂动画吃力

**方案 3：微信小程序原生**

- 只能在一个平台运行，其他端要重写
- WXML/WXSS 是微信自定义 DSL，不通用
- 组件化能力弱，开发体验远不如 Vue/React

**uni-app 的切入点是：既然前端开发者最熟悉 Vue.js，为什么不能把 Vue 编译成各端能运行的代码？**

于是 uni-app 的核心思路确定了：
1. **用 Vue 作为统一的 UI 描述语言**
2. **编译时根据目标平台输出不同的代码**
3. **提供统一的 API 层抹平平台差异**

### 2. 核心原理与架构

#### 2.1 整体架构

```
开发者写的代码（.vue + .js + .css）
        │
        ▼
┌──────────────────────────────────────────┐
│              uni-app 编译器                │
│  （基于 Vue CLI / Vite 定制）              │
│                                          │
│  编译目标 ↓                               │
│   ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐  │
│   │ 微信  │ │ 支付宝 │ │ 头条  │ │  H5  │  │
│   │小程序 │ │ 小程序 │ │ 小程序 │ │      │  │
│   └──────┘ └──────┘ └──────┘ └──────┘  │
│   ┌──────┐ ┌──────┐ ┌──────┐           │
│   │ iOS  │ │Android│ │ 快应用│           │
│   │  App │ │  App  │ │      │           │
│   └──────┘ └──────┘ └──────┘           │
└──────────────────────────────────────────┘
        │
        ▼
各端运行时环境
（小程序引擎 / WebView / Weex / 原生）
```

架构的核心有三层：

**第一层：编译层（Build-time）**
- 读取 `.vue` 文件，解析 template/script/style
- 根据 `--target` 参数（如 `mp-weixin`、`app`、`h5`），生成对应平台的代码
- 插入条件编译标记、替换 API 调用

**第二层：运行时层（Runtime）**
- uni-app 运行时库（`uni-runtime`）提供框架层面的支持
- 包括 Vue 实例管理、生命周期映射、页面路由、事件系统
- 各端的运行时实现不同，但对上暴露一致的接口

**第三层：API 桥接层（Bridge）**
- 统一的 API 如 `uni.request()`、`uni.chooseImage()`、`uni.getLocation()`
- 编译时被替换为对应平台的实现
- 对于 App 端，通过 JS Bridge 调用原生能力

#### 2.2 编译过程详解

以编译到微信小程序为例，uni-app 的编译过程如下：

**输入**：开发者编写的 `pages/index/index.vue`

```vue
<template>
  <view class="container">
    <text class="title">{{ title }}</text>
    <button @tap="handleClick">点击</button>
  </view>
</template>

<script>
export default {
  data() {
    return { title: 'Hello uni-app' }
  },
  methods: {
    handleClick() {
      uni.showToast({ title: '点击了按钮' })
    }
  }
}
</script>

<style>
.container { padding: 20px; }
.title { font-size: 16px; color: #333; }
</style>
```

**编译后输出的微信小程序代码**：

**wxml**（模板）：

```xml
<view class="container">
  <text class="title">{{title}}</text>
  <button bindtap="handleClick">点击</button>
</view>
```

**js**（逻辑层）：

```javascript
Page({
  data() {
    return { title: 'Hello uni-app' }
  },
  handleClick() {
    wx.showToast({ title: '点击了按钮' })
  }
})
```

**wxss**（样式）：

```css
.container { padding: 20rpx; }
.title { font-size: 32rpx; color: #333; }
```

关键变化：
1. `<template>` 中的 `view`/`text`/`button` 保留（这些是 uni-app 的内置标签，对应小程序的 view/text/button）
2. `@tap` 事件绑定 → 微信的 `bindtap`
3. Vue 的 `data()` + `methods` → 微信的 `Page({...})` 配置对象
4. `uni.showToast(...)` → `wx.showToast(...)`
5. `px` → `rpx`（微信小程序的响应式单位，但保留原单位，开发者可自行使用 rpx）

如果是编译到 H5，同样的 `.vue` 文件会输出标准的 Vue 组件代码，用 `uni-app` 的运行时库在浏览器中运行；如果是编译到 App 端，则通过 Weex 引擎或 WebView 渲染。

#### 2.3 条件编译机制

条件编译是 uni-app 处理平台差异的核心机制。它允许在同一个文件中为不同平台写不同的代码。

```vue
<template>
  <view>
    <!-- #ifdef MP-WEIXIN -->
    <button open-type="getPhoneNumber" @getphonenumber="onGetPhoneNumber">
      微信获取手机号
    </button>
    <!-- #endif -->
    
    <!-- #ifdef H5 -->
    <input v-model="phone" placeholder="输入手机号">
    <button @tap="sendSmsCode">发送验证码</button>
    <!-- #endif -->
    
    <!-- #ifdef APP-PLUS -->
    <button @tap="getPhoneByApp">App 获取手机号</button>
    <!-- #endif -->
  </view>
</template>

<script>
export default {
  methods: {
    // #ifdef MP-WEIXIN
    onGetPhoneNumber(e) {
      // 微信小程序获取手机号逻辑
      uni.getPhoneNumber({
        provider: 'weixin',
        success: (res) => { /* ... */ }
      })
    },
    // #endif
    
    // #ifdef H5
    sendSmsCode() {
      // H5 发送短信验证码
    },
    // #endif
    
    // #ifdef APP-PLUS
    getPhoneByApp() {
      // App 端获取手机号
    },
    // #endif
  }
}
</script>

<style>
/* #ifdef MP-WEIXIN */
.button { background: #07c160; }
/* #endif */

/* #ifdef H5 */
.button { background: #1890ff; }
/* #endif */
</style>
```

**条件编译命令**：

| 命令 | 含义 |
|:---|:---|
| `#ifdef` | 如果目标平台在该列表中，则编译这段代码 |
| `#ifndef` | 如果目标平台不在该列表中，则编译这段代码 |
| `#endif` | 结束条件编译块 |

**平台标识**：

| 标识 | 平台 |
|:---|:---|
| `MP-WEIXIN` | 微信小程序 |
| `MP-ALIPAY` | 支付宝小程序 |
| `MP-BAIDU` | 百度小程序 |
| `MP-TOUTIAO` | 头条/抖音小程序 |
| `H5` | H5 网页 |
| `APP-PLUS` | App（iOS/Android） |
| `APP-VUE` | App Vue 模式 |
| `APP-NVUE` | App nvue 模式 |

条件编译在**编译时**处理，不参与运行。所以条件编译的代码不会增加其他平台的包体积。这是 uni-app 相比运行时动态判断（如 `if (process.env.PLATFORM === 'weixin')`）的优势——死代码在编译阶段就被移除了。

#### 2.4 运行时架构

**小程序端运行时**：

在小程序端，uni-app 的运行时相对轻量。因为编译过程已经把 Vue 模板转换成了小程序的 WXML，data/methods 转换成了 Page() 配置对象。运行时主要负责：

1. **生命周期映射**：把 Vue 的 `onMount` / `onUnmount` 映射为小程序的 `onLoad` / `onShow` / `onHide` / `onUnload`
2. **数据更新**：Vue 的响应式数据变更 → 触发 `this.setData()` 更新视图
3. **路由管理**：`uni.navigateTo()` / `uni.redirectTo()` 映射为小程序的路由 API
4. **组件系统**：将 uni-app 的内置组件（`view`、`text`、`scroll-view` 等）映射为小程序的原生组件

```javascript
// 简化版的 uni-app 小程序运行时核心
class UniPage {
  constructor(vueComponent) {
    this.vm = vueComponent
    this.setupLifecycle()
    this.setupData()
  }
  
  setupLifecycle() {
    // Vue 的 mounted → 小程序 onShow
    const originalMounted = this.vm.mounted
    this.vm.mounted = function() {
      // 触发小程序的 onShow
      if (typeof this.onShow === 'function') this.onShow()
      if (originalMounted) originalMounted.call(this)
    }
  }
  
  setupData() {
    // 用 Vue 的响应式系统驱动 setData
    watchEffect(() => {
      const data = extractData(this.vm)
      this.setData(data) // 小程序原生的 setData
    })
  }
}
```

**App 端运行时**：

App 端有两种渲染模式：

1. **WebView 模式（默认）**：在 App 内嵌 WebView，运行编译后的 H5 页面。路由由 `plus.webview` 管理，与原生交互通过 JS Bridge（`plus` 对象）。优点是兼容性好，Web 前端技能可迁移；缺点是性能受限于 WebView。

2. **nvue（native Vue）模式**：基于 Weex 引擎，使用原生组件渲染。使用 `<view>` 等标签，但渲染由 Weex 的原生渲染引擎完成，性能接近原生。缺点是部分 CSS 语法不支持（如 `overflow: visible`），组件生态不如 WebView 模式丰富。

```javascript
// App 端的 JS Bridge 调用原生能力
// 调用相册
uni.chooseImage({
  count: 1,
  success: (res) => {
    // uni-app 编译后实际调用的是 plus.gallery.pick
    // plus.gallery.pick((result) => { ... })
  }
})

// uni-app 封装层的简化实现
function chooseImage(options) {
  if (process.env.PLATFORM === 'app') {
    // App 端使用 plus API
    plus.gallery.pick(
      (res) => options.success && options.success({ tempFilePaths: res.files }),
      options.fail,
      { filter: 'image' }
    )
  } else if (process.env.PLATFORM === 'h5') {
    // H5 端使用 File API
    const input = document.createElement('input')
    input.type = 'file'
    input.accept = 'image/*'
    input.onchange = (e) => { /* ... */ }
    input.click()
  }
  // 小程序端直接在编译时替换为 wx.chooseImage / my.chooseImage
}
```

#### 2.5 pages.json 配置体系

uni-app 使用 `pages.json` 统一管理页面路由和全局配置，替代了 H5 的 Vue Router 和小程序的 `app.json`：

```json
{
  "pages": [
    {
      "path": "pages/index/index",
      "style": {
        "navigationBarTitleText": "首页",
        "enablePullDownRefresh": true,
        "navigationBarBackgroundColor": "#007AFF"
      }
    },
    {
      "path": "pages/detail/detail",
      "style": {
        "navigationBarTitleText": "详情"
      }
    }
  ],
  "globalStyle": {
    "navigationBarTextStyle": "white",
    "navigationBarTitleText": "uni-app",
    "navigationBarBackgroundColor": "#007AFF",
    "backgroundColor": "#F8F8F8"
  },
  "tabBar": {
    "color": "#999",
    "selectedColor": "#007AFF",
    "list": [
      {
        "pagePath": "pages/index/index",
        "text": "首页",
        "iconPath": "static/tab/home.png",
        "selectedIconPath": "static/tab/home-active.png"
      },
      {
        "pagePath": "pages/cart/cart",
        "text": "购物车",
        "iconPath": "static/tab/cart.png",
        "selectedIconPath": "static/tab/cart-active.png"
      },
      {
        "pagePath": "pages/user/user",
        "text": "我的",
        "iconPath": "static/tab/user.png",
        "selectedIconPath": "static/tab/user-active.png"
      }
    ]
  },
  "condition": {
    "current": 0,
    "list": [
      {
        "name": "登录页",
        "path": "pages/login/login"
      }
    ]
  },
  "subPackages": [
    {
      "root": "packageA",
      "pages": [
        { "path": "pages/order/order", "style": { "navigationBarTitleText": "订单" } }
      ]
    }
  ]
}
```

**`pages.json` 的作用**：
- 跨平台页面路由配置，编译时被转换为各平台的路由配置
- 小程序端 → 转换为 `app.json` + 各页面各自的 `.json` 配置
- H5 端 → 转换为 Vue Router 的路由表（uni-app 在编译时自动生成）
- App 端 → 转换为原生路由表

**路由跳转方式**：

```javascript
// 通过 API 跳转（推荐）
uni.navigateTo({ url: '/pages/detail/detail?id=123' })
uni.switchTab({ url: '/pages/index/index' })
uni.redirectTo({ url: '/pages/login/login' })
uni.reLaunch({ url: '/pages/index/index' })

// 通过组件跳转
<navigator url="/pages/detail/detail?id=123">查看详情</navigator>
```

### 3. 实际应用场景与代码示例

#### 3.1 场景 1：从零搭建一个多端电商小程序

**Step 1：创建项目**

```bash
# 通过 HBuilderX 创建（官方 IDE）
# 文件 → 新建 → 项目 → uni-app → 默认模板

# 或通过 CLI 创建（推荐团队协作使用）
npx @dcloudio/uni-preset-vue#vue3 create uni-ecommerce
```

**Step 2：项目结构**

```
uni-ecommerce/
├── pages/
│   ├── index/           # 首页
│   │   ├── index.vue
│   │   └── components/  # 首页专属组件
│   ├── category/        # 分类页
│   ├── cart/            # 购物车
│   ├── user/            # 个人中心
│   └── detail/          # 商品详情
├── components/          # 全局公共组件
│   ├── product-card.vue
│   └── price-tag.vue
├── static/              # 静态资源
│   ├── images/
│   └── tab/
├── store/               # 状态管理（Vuex/Pinia）
│   └── index.js
├── api/                 # API 请求封装
│   ├── request.js       # uni.request 封装
│   └── product.js       # 商品相关接口
├── pages.json           # 页面路由配置
├── manifest.json        # 应用配置（App 图标、小程序 AppID 等）
├── uni.scss             # 全局样式变量
└── App.vue              # 应用入口组件
```

**Step 3：封装网络请求**

```javascript
// api/request.js
const BASE_URL = 'https://api.example.com'

// 请求拦截器
function requestInterceptor(config) {
  const token = uni.getStorageSync('token')
  if (token) {
    config.header = { ...config.header, Authorization: `Bearer ${token}` }
  }
  return config
}

// 响应拦截器
function responseInterceptor(response) {
  if (response.statusCode === 401) {
    // Token 过期，跳转登录
    uni.removeStorageSync('token')
    uni.navigateTo({ url: '/pages/login/login' })
    return Promise.reject(new Error('登录已过期'))
  }
  return response.data
}

// 封装请求方法
function request(config) {
  config = requestInterceptor(config)
  
  return new Promise((resolve, reject) => {
    uni.request({
      url: BASE_URL + config.url,
      method: config.method || 'GET',
      data: config.data,
      header: config.header,
      timeout: config.timeout || 10000,
      success: (res) => {
        try {
          const data = responseInterceptor(res)
          resolve(data)
        } catch (err) {
          reject(err)
        }
      },
      fail: (err) => {
        // 网络错误统一处理
        uni.showToast({ title: '网络异常', icon: 'none' })
        reject(err)
      }
    })
  })
}

export const get = (url, data) => request({ url, data })
export const post = (url, data) => request({ url, data, method: 'POST' })
```

**Step 4：页面开发（商品列表）**

```vue
<!-- pages/index/index.vue -->
<template>
  <view class="container">
    <!-- 搜索栏 -->
    <view class="search-bar">
      <uni-search-bar 
        placeholder="搜索商品" 
        @confirm="onSearch"
      />
    </view>
    
    <!-- 轮播图 -->
    <swiper 
      class="banner" 
      :indicator-dots="true" 
      :autoplay="true"
      indicator-color="rgba(255,255,255,0.5)"
      indicator-active-color="#fff"
    >
      <swiper-item 
        v-for="(item, index) in banners" 
        :key="index"
        @tap="goToBanner(item)"
      >
        <image :src="item.image" mode="widthFix" />
      </swiper-item>
    </swiper>
    
    <!-- 商品列表（无限滚动加载） -->
    <view class="product-list">
      <view 
        class="product-item" 
        v-for="(product, index) in products" 
        :key="product.id"
        @tap="goToDetail(product.id)"
      >
        <image :src="product.thumbnail" mode="aspectFill" />
        <view class="product-info">
          <text class="product-name">{{ product.name }}</text>
          <text class="product-price">¥{{ product.price }}</text>
          <text class="product-sold">已售 {{ product.soldCount }}</text>
        </view>
      </view>
    </view>
    
    <!-- 加载更多 -->
    <uni-load-more 
      :status="loadMoreStatus"
      :content-text="{
        contentdown: '上拉加载更多',
        contentrefresh: '加载中...',
        contentnomore: '没有更多了'
      }"
    />
  </view>
</template>

<script>
import { get } from '@/api/request'

export default {
  data() {
    return {
      banners: [],
      products: [],
      page: 1,
      pageSize: 10,
      hasMore: true,
      loadMoreStatus: 'more',
      keyword: ''
    }
  },
  
  onLoad() {
    this.loadBanners()
    this.loadProducts()
  },
  
  // 下拉刷新
  onPullDownRefresh() {
    this.page = 1
    this.hasMore = true
    this.products = []
    this.loadProducts().finally(() => {
      uni.stopPullDownRefresh()
    })
  },
  
  // 上拉加载更多
  onReachBottom() {
    if (this.hasMore) {
      this.loadMoreStatus = 'loading'
      this.page++
      this.loadProducts()
    }
  },
  
  methods: {
    async loadBanners() {
      try {
        const res = await get('/api/banners')
        this.banners = res.data
      } catch (err) {
        console.error('加载轮播图失败', err)
      }
    },
    
    async loadProducts() {
      try {
        const res = await get('/api/products', {
          page: this.page,
          pageSize: this.pageSize,
          keyword: this.keyword
        })
        
        if (res.data.length < this.pageSize) {
          this.hasMore = false
          this.loadMoreStatus = 'noMore'
        }
        
        this.products = [...this.products, ...res.data]
      } catch (err) {
        this.loadMoreStatus = 'more'
        console.error('加载商品失败', err)
      }
    },
    
    goToDetail(id) {
      uni.navigateTo({ url: `/pages/detail/detail?id=${id}` })
    },
    
    onSearch(e) {
      this.keyword = e.value
      this.page = 1
      this.products = []
      this.hasMore = true
      this.loadProducts()
    },
    
    goToBanner(banner) {
      if (banner.link) {
        uni.navigateTo({ url: banner.link })
      }
    }
  }
}
</script>

<style lang="scss">
.container {
  min-height: 100vh;
  background: #f5f5f5;
}

.search-bar {
  padding: 20rpx 30rpx;
  background: #fff;
}

.banner {
  height: 350rpx;
  
  image {
    width: 100%;
    height: 350rpx;
  }
}

.product-list {
  display: flex;
  flex-wrap: wrap;
  padding: 20rpx;
}

.product-item {
  width: 345rpx;
  margin-bottom: 20rpx;
  background: #fff;
  border-radius: 12rpx;
  overflow: hidden;
  
  &:nth-child(odd) {
    margin-right: 20rpx;
  }
  
  image {
    width: 345rpx;
    height: 345rpx;
  }
}

.product-info {
  padding: 16rpx;
}

.product-name {
  font-size: 26rpx;
  color: #333;
  display: -webkit-box;
  -webkit-line-clamp: 2;
  -webkit-box-orient: vertical;
  overflow: hidden;
}

.product-price {
  font-size: 32rpx;
  color: #ff6b6b;
  font-weight: bold;
}

.product-sold {
  font-size: 22rpx;
  color: #999;
}
</style>
```

**Step 5：编译发布**

```bash
# 编译到微信小程序
npm run dev:mp-weixin
# 用微信开发者工具打开 dist/dev/mp-weixin

# 编译到 H5
npm run dev:h5
# 浏览器访问 http://localhost:8080

# 编译到 App
npm run dev:app
# 用 HBuilderX 运行到真机或模拟器

# 生产构建
npm run build:mp-weixin    # 构建微信小程序
npm run build:h5           # 构建 H5
npm run build:app          # 构建 App
```

#### 3.2 场景 2：使用 uni-ui 组件库快速搭建页面

uni-app 官方提供了 **uni-ui** 组件库，包含 70+ 个跨平台组件：

```vue
<template>
  <view class="container">
    <!-- 导航栏 -->
    <uni-nav-bar 
      left-icon="back" 
      title="个人信息" 
      @clickLeft="goBack"
    />
    
    <!-- 表单 -->
    <uni-forms :modelValue="form" ref="form">
      <uni-forms-item label="姓名" name="name">
        <uni-easyinput 
          v-model="form.name" 
          placeholder="请输入姓名"
        />
      </uni-forms-item>
      
      <uni-forms-item label="性别" name="gender">
        <uni-data-checkbox 
          mode="button" 
          :localdata="genderOptions"
          v-model="form.gender"
        />
      </uni-forms-item>
      
      <uni-forms-item label="生日" name="birthday">
        <uni-datetime-picker 
          type="date" 
          v-model="form.birthday"
          :clear-icon="false"
        />
      </uni-forms-item>
      
      <uni-forms-item label="城市" name="city">
        <uni-data-picker 
          :localdata="cityData"
          v-model="form.city"
          placeholder="请选择城市"
        />
      </uni-forms-item>
      
      <uni-forms-item label="简介" name="bio">
        <uni-easyinput 
          type="textarea"
          v-model="form.bio" 
          placeholder="请填写简介"
          :maxlength="200"
        />
      </uni-forms-item>
    </uni-forms>
    
    <!-- 提交按钮 -->
    <button 
      type="primary" 
      class="submit-btn"
      @tap="submitForm"
    >
      保存
    </button>
    
    <!-- 列表导航 -->
    <uni-list>
      <uni-list-item 
        title="账号安全" 
        show-arrow
        @tap="goToSecurity"
      />
      <uni-list-item 
        title="收货地址" 
        show-arrow
        @tap="goToAddress"
      />
      <uni-list-item 
        title="关于我们" 
        show-arrow
        @tap="goToAbout"
      />
    </uni-list>
  </view>
</template>

<script>
export default {
  data() {
    return {
      form: {
        name: '',
        gender: 0,
        birthday: '',
        city: [],
        bio: ''
      },
      genderOptions: [
        { value: 0, text: '男' },
        { value: 1, text: '女' }
      ],
      cityData: [
        { text: '北京市', value: '110000' },
        { text: '上海市', value: '310000' },
        { text: '广东省', value: '440000', children: [
          { text: '广州市', value: '440100' },
          { text: '深圳市', value: '440300' }
        ]}
      ]
    }
  },
  
  methods: {
    async submitForm() {
      const valid = await this.$refs.form.validate()
      if (valid) {
        // 提交表单
        // ...
        uni.showToast({ title: '保存成功' })
      }
    },
    
    goBack() { uni.navigateBack() },
    goToSecurity() { uni.navigateTo({ url: '/pages/user/security' }) },
    goToAddress() { uni.navigateTo({ url: '/pages/user/address' }) },
    goToAbout() { uni.navigateTo({ url: '/pages/user/about' }) }
  }
}
</script>
```

#### 3.3 场景 3：使用 uni_modules 生态

uni_modules 是 uni-app 的插件生态，可以在项目中直接安装和使用社区贡献的插件：

```bash
# 在 HBuilderX 中：右键项目 → 使用 uni_modules 插件 → 选择插件安装
# 或手动从插件市场下载：
# https://ext.dcloud.net.cn/
```

安装后在项目中使用：

```vue
<template>
  <view>
    <!-- 使用 uni_modules 中的图表组件 -->
    <qiun-data-charts 
      type="column"
      :chartData="chartData"
      :opts="chartOpts"
    />
    
    <!-- 使用二维码生成组件 -->
    <tki-qrcode 
      :val="qrcodeValue"
      :size="200"
    />
  </view>
</template>

<script>
export default {
  data() {
    return {
      chartData: {
        categories: ['1月', '2月', '3月', '4月'],
        series: [{
          name: '销售额',
          data: [150, 230, 180, 290]
        }]
      },
      chartOpts: { color: ['#1890ff'] },
      qrcodeValue: 'https://example.com/product/123'
    }
  }
}
</script>
```

### 4. 常见误区 & 实际项目中的坑

#### 4.1 误区：以为条件编译是运行时判断

**错误写法**：

```javascript
// ❌ 在运行时判断平台
if (uni.getSystemInfoSync().platform === 'ios') {
  // iOS 特有逻辑
}
```

**问题**：这种写法确实能在运行时工作，但**不会参与编译时的代码裁剪**。所有平台的代码都被打包到产物中，只是根据条件选择性执行。这在某些场景下会导致：
- 包体积增大（所有平台的代码都在）
- 潜在的安全风险（其他平台的代码可以被反编译看到）
- 某些平台 API 在编译阶段被引用导致报错（如 `wx.login` 在 H5 中不存在）

**正确做法**：使用条件编译标记平台差异。

```javascript
// ✅ 编译时裁剪
// #ifdef APP-PLUS
const deviceId = plus.device.uuid // App 端使用 plus API
// #endif

// #ifdef H5
const deviceId = localStorage.getItem('deviceId') // H5 端使用 localStorage
// #endif

// #ifdef MP-WEIXIN
const deviceId = '' // 微信小程序无法获取设备唯一标识
// #endif
```

**条件编译和运行时判断的区别**：

```javascript
// 条件编译（编译期确定，死代码被移除）
// #ifdef H5
console.log('只在 H5 端执行')
// #endif

// 运行时判断（所有端都包含这段代码，只在运行时决定是否执行）
if (/* 判断当前平台 */) {
  console.log('所有端都携带这段代码')
}
```

**例外情况**：某些场景确实需要在运行时判断平台，比如根据平台动态展示 UI 风格。这时应该用条件编译包裹平台特有的逻辑，运行时只做样式调整。

#### 4.2 误区：使用浏览器/小程序独有的 API 不经过条件编译

**错误写法**：

```vue
<script>
export default {
  methods: {
    shareToWechat() {
      // ❌ 直接使用微信小程序 API，没有用条件编译
      wx.shareAppMessage({
        title: '分享标题',
        path: '/pages/index/index'
      })
      
      // ❌ 直接使用浏览器 API
      document.title = '新标题'
    }
  }
}
</script>
```

**为什么错**：
- 编译到 H5 时，`wx.shareAppMessage` 不存在，会报 `wx is not defined`
- 编译到微信小程序时，`document` 不存在，会报 `document is not defined`
- 你的代码在编译时不会报错（uni-app 无法检测到这些底层 API 的调用），但在运行时必崩

**正确做法**：

```javascript
methods: {
  shareToWechat() {
    // #ifdef MP-WEIXIN
    wx.shareAppMessage({
      title: '分享标题',
      path: '/pages/index/index'
    })
    // #endif
    
    // #ifdef H5
    // 使用 Web Share API 或自定义分享
    if (navigator.share) {
      navigator.share({ title: '分享标题', url: window.location.href })
    }
    // #endif
    
    // #ifdef APP-PLUS
    plus.share.sendWithSystem({ content: '分享内容', href: 'https://...' })
    // #endif
  }
}
```

**实操经验**：养成习惯——凡是看到 `wx.`、`my.`、`dd.`、`tt.`、`document.`、`window.`、`plus.` 等平台特有 API，立即用条件编译包裹。

#### 4.3 实际坑：组件命名与原生标签冲突

**错误写法**：

```vue
<template>
  <!-- 自定义组件命名为 <switch> → 和 uni-app 内置的 switch 组件冲突 -->
  <Switch :checked="isOn" @change="handleChange" />
</template>
```

**问题**：uni-app 内置了一些和小程序同名的组件，如 `view`、`text`、`image`、`scroll-view`、`swiper`、`switch`、`slider`、`picker` 等。如果你自定义的组件和这些名称相同，会导致**模板解析歧义**——编译器不知道该用内置组件还是你的自定义组件。

**正确做法**：

```vue
<template>
  <!-- 加上 uni- 前缀区分，或使用其他名称 -->
  <MySwitch :checked="isOn" @change="handleChange" />
  <!-- 或使用不同组件名 -->
  <ToggleSwitch :checked="isOn" @change="handleChange" />
</template>
```

**完整的内置标签列表**（避免用作自定义组件名）：

```
view, text, image, scroll-view, swiper, swiper-item,
movable-view, movable-area, cover-view, cover-image,
icon, textarea, input, slider, switch, picker,
picker-view, picker-view-column, navigator, functional-page-navigator,
button, checkbox, checkbox-group, radio, radio-group,
form, label, progress, rich-text,
editor, map, web-view, ad, page-meta,
canvas, camera, live-player, live-pusher,
video, audio, voip-room,
```

**判断方法**：在官方文档中查询组件名，如果该组件在文档中有专门页面，说明是内置组件。

#### 4.4 实际坑：nvue 模式下的 CSS 限制

nvue（native Vue）模式基于 Weex 渲染引擎，使用原生组件而非 WebView，因此 CSS 支持有限。

**不支持或不完全支持的 CSS 特性**：

```css
/* ❌ 不支持的选择器 */
* {}                 /* 通配符选择器 */
.parent > .child {}  /* 子选择器（仅支持单级） */
.parent .child {}    /* 后代选择器（仅支持单级） */
.class1.class2 {}    /* 多类名选择器 */
:before, :after {}   /* 伪元素 */
:first-child {}      /* 伪类 */
:last-child {}
:nth-child() {}

/* ❌ 不支持或有限制的属性 */
position: fixed;     /* 不推荐使用，位置可能不准确 */
overflow: visible;   /* 不支持，nvue 中 overflow 只能为 hidden/scroll */
background-image;    /* 不支持，使用 image 组件替代 */
box-shadow;          /* 不推荐，用 border 模拟 */
border-radius: 50%;  /* 百分比不支持，使用固定 px */
flex-direction: column-reverse; /* 不支持 */
order: -1;           /* 不支持 */
```

**正确做法**：

```vue
<!-- nvue 模式下的安全 CSS -->
<style>
/* ✅ 推荐的选择器和属性 */
.container {
  flex-direction: row;       /* nvue 默认是 column */
  justify-content: center;
  align-items: center;
  width: 750rpx;
  padding: 20rpx;
  background-color: #ffffff; /* 使用 background-color 而非 background */
  border-width: 1px;
  border-color: #e5e5e5;
  border-radius: 10px;       /* 固定 px 值 */
}

/* ✅ 只使用单级类名选择器 */
.title { }
.name { }
.price { }
</style>
```

**如何判断一个页面是否使用 nvue**？看文件后缀：
- `.vue` → WebView 模式（功能完整）
- `.nvue` → Weex 原生渲染（性能好，CSS 有限制）

**策略建议**：
- 大部分页面用 `.vue`（功能完整、开发效率高）
- 对性能敏感的页面（如长列表、聊天页、直播间）用 `.nvue`
- 在 `pages.json` 中通过 `style` 配置混合使用

```json
{
  "pages": [
    { "path": "pages/index/index", "style": {} },                    // 普通 .vue 页面
    { "path": "pages/chat/chat", "style": { "navigationStyle": "custom" } } // .nvue 页面
  ]
}
```

#### 4.5 实际坑：小程序包体积限制

微信小程序要求**主包 + 所有分包**不超过 20MB，单个分包/主包不超过 2MB。uni-app 项目很容易超出限制。

**常见原因**：
- 静态资源（图片、字体）放在 `static/` 中被直接打包
- 三方 UI 库全量引入
- `node_modules` 中的代码被错误打包

**优化策略**：

**1. 图片使用 CDN 而非本地引用**：

```vue
<!-- ❌ 本地图片 → 被打包进代码包 -->
<image src="/static/images/banner.png" />

<!-- ✅ CDN 图片 → 不占用包体积 -->
<image src="https://cdn.example.com/images/banner.png" />
```

**2. 合理使用分包**：

```json
// pages.json
{
  "pages": [
    { "path": "pages/index/index", "style": {} },
    { "path": "pages/login/login", "style": {} }
  ],
  "subPackages": [
    {
      "root": "package-order",
      "pages": [
        { "path": "pages/list/list" },
        { "path": "pages/detail/detail" }
      ]
    },
    {
      "root": "package-user",
      "pages": [
        { "path": "pages/setting/setting" },
        { "path": "pages/about/about" }
      ]
    }
  ],
  "preloadRule": {
    "pages/index/index": {
      "network": "all",
      "packages": ["package-order"] // 预加载订单分包
    }
  }
}
```

分包后，用户仅首次下载主包（通常 < 1MB），进入对应分包时再按需下载。`preloadRule` 可以预判用户行为提前加载。

**3. 使用 uni_modules 的按需引入**：

大多数 uni_modules 插件支持按需引入组件。只引入用到的组件，而不是全部：

```javascript
// ❌ 全量引入
import uni from '@dcloudio/uni-ui'

// ✅ 按需引入
import { UniNavBar, UniList, UniListItem } from '@dcloudio/uni-ui'
```

### 5. 与相关知识的关联 & 对比

#### 5.1 uni-app vs Taro

两者都是「编写一套代码，编译到多端」的框架，但核心差异在技术选型上：

| 维度 | uni-app | Taro |
|:---|:---|:---|
| **底层框架** | Vue.js（2/3） | React（兼容 Vue 3 实验性支持） |
| **编译方式** | Webpack/Vite 定制 + 模板替换 | Webpack + 自定义 Babel 插件 |
| **App 渲染** | WebView / Weex（nvue） | React Native |
| **小程序支持** | 最全（约 20+ 平台） | 多（约 10+ 平台） |
| **状态管理** | Vuex / Pinia | Redux / MobX |
| **组件库** | uni-ui（官方） | Taro UI（官方） |
| **CLI 工具** | HBuilderX / CLI | CLI 为主 |
| **社区生态** | 国内最大，DCloud 维护 | 京东开源，社区活跃 |

**选型差异**：
- 团队熟悉 **Vue** → 选 **uni-app**
- 团队熟悉 **React** → 选 **Taro**
- 目标平台主要是**微信 + 支付宝 + 头条**等小程序 → 两者都可以，看技术栈偏好
- 需要 App 端有**较好性能** → 选 Taro（基于 RN）或 uni-app 的 nvue 模式

#### 5.2 uni-app vs Flutter

| 维度 | uni-app | Flutter |
|:---|:---|:---|
| **语言** | JavaScript/TypeScript + Vue | Dart |
| **渲染引擎** | WebView / Weex | Skia 自绘引擎 |
| **性能** | 中等 | 高（接近原生） |
| **小程序** | ✅ 原生支持 | ❌ 无 |
| **H5** | ✅ 原生支持 | ⚠️ 需 flutter web（性能差） |
| **学习曲线** | 低（会 Vue 即可） | 高（Dart + 全新框架） |
| **UI 一致性** | 各端有差异 | 高度一致（自绘） |
| **热重载** | ✅ | ✅ 极快 |
| **国内大厂使用** | 腾讯、阿里、字节等大量使用 | 闲鱼、Google Ads 等 |

**什么时候该用 Flutter 而非 uni-app**：
1. **App 是主要目标**，小程序和 H5 只是辅助
2. **UI 一致性要求高**（Flutter 在所有平台绘制同样的像素）
3. **性能敏感型应用**（大量动画、60fps 流畅滚动、游戏化交互）
4. 团队愿意学习 Dart，且不依赖小程序生态

**什么时候该用 uni-app 而非 Flutter**：
1. **小程序是核心渠道**（uni-app 的小程序支持是 Flutter 完全无法替代的）
2. **团队主要是 Vue 开发者**，转型成本高
3. **快速 MVP 验证**，需要一周内上线多端
4. **项目中需要大量调用各端的原生 API**（定位、支付、蓝牙、NFC 等，uni-app 的封装更直接）

#### 5.3 uni-app vs 原生小程序开发

| 维度 | uni-app | 原生小程序 |
|:---|:---|:---|
| **开发效率** | 高（一套代码多端） | 低（每端独立开发） |
| **调试体验** | HBuilderX / VS Code | 各平台开发者工具 |
| **性能** | 略低于原生（有中间层开销） | 最高 |
| **包体积** | 略大（包含运行时库） | 最小 |
| **第三方组件** | uni_modules + npm | 各平台市场 |
| **NPM 支持** | ✅ 完整 | ⚠️ 有限（需构建工具） |
| **TypeScript** | ✅ 支持 | ⚠️ 部分支持 |

**uni-app 比原生小程序多出来的运行时体积**：

| 端 | 额外体积 | 说明 |
|:---|:---|:---|
| 微信小程序 | ~50-80 KB | uni-app 运行时 + Vue 运行时 |
| H5 | ~80-120 KB | uni-app 运行时 + Vue 运行时 |
| App | ~2-5 MB | 集成 Weex 或 WebView 引擎 |

对于大部分项目，这几十 KB 的开销是可以接受的。但对于包体积极度敏感的场景（如小游戏、工具类小程序），原生开发依然是最优选择。

### 6. 现代最佳实践（2025-2026）

#### 6.1 使用 Vue 3 + Vite + TypeScript 开发 uni-app

uni-app 从 3.x 开始全面支持 Vue 3 + Vite + TypeScript：

```bash
# 创建 Vue 3 版本的项目
npx @dcloudio/uni-preset-vue#vue3 create my-uni-app

# 安装 TypeScript 类型声明
npm install -D @dcloudio/types
```

**tsconfig.json 配置**：

```json
{
  "compilerOptions": {
    "target": "ESNext",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "strict": true,
    "jsx": "preserve",
    "paths": {
      "@/*": ["./src/*"]
    },
    "types": [
      "@dcloudio/types",  // uni-app API 类型
      "@uni-helper/uni-ui-types" // uni-ui 组件类型（可选）
    ]
  },
  "include": ["src/**/*.ts", "src/**/*.d.ts", "src/**/*.vue"]
}
```

**TypeScript 组件示例**：

```vue
<script setup lang="ts">
import { ref, onMounted } from 'vue'

interface Product {
  id: number
  name: string
  price: number
  thumbnail: string
}

const products = ref<Product[]>([])
const loading = ref(false)

onMounted(async () => {
  loading.value = true
  try {
    const res = await uni.request({
      url: 'https://api.example.com/products',
      method: 'GET'
    })
    products.value = res.data as Product[]
  } catch (err) {
    uni.showToast({ title: '加载失败', icon: 'error' })
  } finally {
    loading.value = false
  }
})
</script>

<template>
  <view class="container">
    <view v-if="loading" class="loading">
      <uni-load-more status="loading" />
    </view>
    
    <view v-else class="product-list">
      <view 
        v-for="product in products" 
        :key="product.id"
        class="product-item"
        @tap="navigateTo(`/pages/detail/detail?id=${product.id}`)"
      >
        <image :src="product.thumbnail" mode="aspectFill" />
        <text class="name">{{ product.name }}</text>
        <text class="price">¥{{ product.price }}</text>
      </view>
    </view>
  </view>
</template>
```

#### 6.2 状态管理：使用 Pinia（Vue 3）

```javascript
// store/cart.js
import { defineStore } from 'pinia'

export const useCartStore = defineStore('cart', () => {
  const items = ref([])
  
  const totalCount = computed(() => 
    items.value.reduce((sum, item) => sum + item.count, 0)
  )
  
  const totalPrice = computed(() => 
    items.value.reduce((sum, item) => sum + item.price * item.count, 0)
  )
  
  function addItem(product) {
    const existing = items.value.find(item => item.id === product.id)
    if (existing) {
      existing.count++
    } else {
      items.value.push({ ...product, count: 1 })
    }
    saveToStorage()
  }
  
  function removeItem(productId) {
    items.value = items.value.filter(item => item.id !== productId)
    saveToStorage()
  }
  
  function saveToStorage() {
    uni.setStorageSync('cart_items', JSON.stringify(items.value))
  }
  
  function loadFromStorage() {
    const data = uni.getStorageSync('cart_items')
    if (data) {
      items.value = JSON.parse(data)
    }
  }
  
  return { items, totalCount, totalPrice, addItem, removeItem, loadFromStorage }
})
```

#### 6.3 组件通信策略

uni-app 中跨页面/跨组件的通信方案：

```javascript
// 1. uni.$on / uni.$emit（全局事件总线）
// 在页面 A 中监听
uni.$on('loginSuccess', (userInfo) => {
  console.log('用户登录成功', userInfo)
})

// 在页面 B 中触发
uni.$emit('loginSuccess', { name: 'Alice', id: 123 })

// 注意：要在页面卸载时取消监听，防止内存泄漏
onUnmounted(() => {
  uni.$off('loginSuccess')
})

// 2. getApp() 访问全局 App 实例
// App.vue
export default {
  globalData: {
    userInfo: null
  },
  onLaunch() {
    // 初始化
  }
}

// 任意页面中访问
const app = getApp()
console.log(app.globalData.userInfo)

// 3. Vuex/Pinia（推荐的全局状态方案）
// 适用于跨页面共享的复杂状态
```

#### 6.4 性能优化清单

```javascript
// 1. 长列表优化：使用虚拟列表
// 使用 uni-ui 的 uni-virtual-list 或社区组件
import VirtualList from '@/components/virtual-list'

// 2. 图片懒加载
<image 
  :src="item.thumbnail" 
  lazy-load   // 启用小程序图片懒加载
  mode="aspectFill"
/>

// 3. 减少 setData 频率
// ❌ 频繁 setData
this.count = this.count + 1 // 触发一次 setData
this.name = 'new name'       // 触发一次 setData

// ✅ 合并变更
this.$nextTick(() => {
  this.count = this.count + 1
  this.name = 'new name'
})

// 4. 使用分包 + 预加载
// pages.json 中配置 preloadRule

// 5. 合理使用 v-show 而非 v-if
// 高频切换用 v-show（只切换 display），低频用 v-if（条件渲染）

// 6. App 端启用 nvue 模式
// 对性能敏感的页面使用 .nvue 文件
```

### 7. 常见疑问解答（自问自答）

#### Q1: uni-app 的项目用 HBuilderX 还是 CLI 开发好？

| 场景 | 推荐方式 | 原因 |
|:---|:---|:---|
| 个人开发者/小团队 | HBuilderX | 开箱即用，内置 uni-app 支持 |
| 中大型团队（多人协作） | CLI（VS Code） | Git 集成好，可定制构建流程 |
| 需要自定义 Webpack/Vite 配置 | CLI | HBuilderX 的构建配置是黑盒 |

**HBuilderX 的优势**：
- 内置 uni-app 编译、运行、调试（真机调试、模拟器调试）
- 可视化 `pages.json` 编辑器
- 一键打包、云打包
- 插件市场集成

**CLI 的优势**：
- 可以用 VS Code/WebStorm 等主流 IDE
- 完整的 `vue.config.js` / `vite.config.js` 自定义能力
- 更好的 Git 对比和 Code Review 体验
- CI/CD 集成方便

**建议**：项目初始化用 HBuilderX 快速创建模板，然后切换到 CLI 开发。或者直接用 CLI 创建项目。

#### Q2: uni-app 支持 npx/npm 安装的第三方库吗？

支持。uni-app CLI 项目完全兼容 npm 生态。但要注意：

**✅ 可以正常使用的库类型**：
- 纯逻辑库（`lodash`、`dayjs`、`axios`、`crypto-js`）
- 与框架无关的工具库
- Vue 插件（`vue-i18n`、`pinia`）

**⚠️ 需要谨慎的库类型**：
- **依赖 DOM/BOM API 的库**（如在 H5 中正常，但在小程序中运行时会报错）：
  ```javascript
  // 这类库在小程序中不可用
  import $ from 'jquery'      // ❌ 依赖 document.getElementById
  import { animate } from 'motion' // ❌ 依赖 requestAnimationFrame
  ```
  
- **依赖 Node.js 内置模块的库**（在小程序/H5 中不可用）：
  ```javascript
  import fs from 'fs'           // ❌ 小程序/H5 中没有 fs
  import crypto from 'crypto'   // ⚠️ 小程序有 polyfill，H5 有 Web Crypto API，但不一样
  ```

**解决方式**：
1. 在 `vue.config.js` 中配置 `transpileDependencies`（让 uni-app 编译这些库的源码）
2. 寻找替代库（如用 `crypto-js` 替代 `node:crypto`）
3. 对平台特有库使用条件编译

#### Q3: uni-app 怎么调用原生的蓝牙/摄像头/NFC 等能力？

uni-app 对主流原生能力都有封装：

```javascript
// 蓝牙
uni.openBluetoothAdapter({
  success: () => {
    uni.startBluetoothDevicesDiscovery({
      success: (res) => { console.log('发现设备', res.devices) }
    })
  }
})

// 扫码
uni.scanCode({
  onlyFromCamera: true,
  success: (res) => { console.log('扫码结果', res.result) }
})

// NFC（需在 manifest.json 中声明 NFC 权限）
uni.getNFCAdapter()

// 指纹/面容识别
uni.startSoterAuthentication({
  requestAuthModes: ['fingerPrint', 'facial'],
  success: (res) => { console.log('认证成功') }
})
```

如果想调用 uni-app 未封装的**原生 SDK**，有两种方式：

**方式 1：App 原生插件（推荐）**
- 在 DCloud 插件市场搜索现成的原生插件
- 或自行开发 App 原生插件（Android 用 Java/Kotlin，iOS 用 Swift/Objective-C）

```javascript
// 使用原生插件
const plugin = uni.requireNativePlugin('PluginName')
plugin.methodName({ key: 'value' }, (result) => {
  console.log(result)
})
```

**方式 2：条件编译直接调用各端 API**

```javascript
// #ifdef APP-PLUS
// 通过 plus 对象调用 5+ App 原生能力
plus.runtime.openURL('weixin://')
// #endif

// #ifdef MP-WEIXIN
// 微信小程序的权限 API
wx.authorize({ scope: 'scope.record' })
// #endif

// #ifdef H5
// Web API
navigator.mediaDevices.getUserMedia({ audio: true })
// #endif
```

#### Q4: uni-app 如何进行自动化测试？

uni-app 的测试方案因平台而异：

**H5 端**：标准的 Web 测试工具

```bash
npm install -D vitest @vue/test-utils
```

```javascript
// __tests__/product.test.js
import { mount } from '@vue/test-utils'
import ProductCard from '@/components/product-card.vue'

describe('ProductCard', () => {
  it('渲染商品名称', () => {
    const wrapper = mount(ProductCard, {
      props: { name: '测试商品', price: 99 }
    })
    expect(wrapper.text()).toContain('测试商品')
    expect(wrapper.text()).toContain('99')
  })
  
  it('点击触发事件', async () => {
    const wrapper = mount(ProductCard)
    await wrapper.trigger('tap')
    expect(wrapper.emitted('click')).toBeTruthy()
  })
})
```

**小程序端**：

```bash
# 使用小程序官方测试工具
npm install -D miniprogram-automator
```

```javascript
// test/wx.test.js
const automator = require('miniprogram-automator')

describe('Mini Program Test', () => {
  let miniProgram
  
  beforeAll(async () => {
    miniProgram = await automator.connect({
      projectPath: './dist/dev/mp-weixin'
    })
  })
  
  it('首页商品列表展示', async () => {
    const page = await miniProgram.navigateTo('/pages/index/index')
    await page.waitFor(500)
    const items = await page.$$('.product-item')
    expect(items.length).toBeGreaterThan(0)
  })
  
  afterAll(async () => {
    await miniProgram.close()
  })
})
```

**App 端**：
- 推荐使用各平台的原生测试框架
- iOS：XCTest（XCUITest）
- Android：Espresso / UI Automator

实际上，大部分 uni-app 项目的主要测试策略是**手动测试 + 小程序真机调试 + App 云真机**。自动化测试在跨平台项目中成本较高，通常只在核心业务路径上实施。

### 8. 参考资料 & 推荐学习路径

**官方资源**：
- [uni-app 官方文档](https://uniapp.dcloud.net.cn/) — 必读，全面覆盖 API 和组件
- [uni-app 插件市场](https://ext.dcloud.net.cn/) — 查找社区插件
- [uni-ui 组件库](https://uniapp.dcloud.net.cn/component/uniui/uni-ui.html) — 官方跨平台组件
- [HBuilderX 下载](https://www.dcloud.io/hbuilderx.html) — 推荐 IDE

**推荐学习路径**：

```
1. 阅读本文【面试速答版】 → 建立整体认知
2. 下载 HBuilderX，用默认模板创建第一个 uni-app 项目
3. 在 H5 端运行，查看效果：npm run dev:h5
4. 安装微信开发者工具，运行到小程序模拟器
5. 学习 pages.json 配置和路由跳转
6. 熟悉常用的 uni-api：uni.request / uni.navigateTo / uni.showToast / uni.setStorage
7. 学习条件编译，处理平台差异
8. 使用 uni-ui 组件库搭建一个简单的页面
9. 学习 uni_modules 插件的安装和使用
10. 尝试将一个现有的 Vue 项目迁移到 uni-app（理解迁移中的坑）
```

**关联知识点索引**：
- uni-app 的底层框架 → [Vue3 核心变化与 Composition API](../Vue3/Vue3%20核心变化与 Composition%20API.md)
- 跨页面的传参机制 → [跨页面传参](./跨页面传参.md)
- 状态管理方案 → [Pinia](../Vue3/Pinia.md)
- 条件编译底层原理 → [AST 详解](../../前端工程化/AST%20详解.md)（编译时代码转换）
