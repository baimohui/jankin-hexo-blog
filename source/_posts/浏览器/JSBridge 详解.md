---
title: JSBridge 详解
categories: 
- 浏览器
tags:
- JSBridge
- Hybrid
- WebView
- 原生通信
- 跨端
---

## 一、为什么需要 JSBridge

JSBridge 是连接 **JavaScript（WebView）** 和 **原生（Native）** 的通信桥梁。它让 H5 页面能够调用原生能力（相机、GPS、蓝牙、支付），也让原生能够主动向 H5 发送事件（通知、状态变化）。<!--more-->

### Hybrid 应用的架构

```text
┌─────────────────────────────────────────┐
│             原生容器（iOS/Android）        │
│  ┌───────────────────────────────────┐  │
│  │          WebView                  │  │
│  │  ┌─────────────────────────────┐  │  │
│  │  │         H5 页面              │  │  │
│  │  │  JS ←→ JSBridge ←→ Native  │  │  │
│  │  └─────────────────────────────┘  │  │
│  └───────────────────────────────────┘  │
│                                         │
│  原生功能：相机 / GPS / 支付 / 蓝牙     │
└─────────────────────────────────────────┘
```

### 为什么不能直接用 AJAX

```text
H5 局限：
  ├── 无法访问系统硬件（相机、蓝牙、NFC）
  ├── 无法调用原生 SDK（微信支付、人脸识别）
  ├── 无法获取设备唯一标识
  ├── 无法在后台运行
  └── 无法发送系统通知

JSBridge 解决了"浏览器做不到"的事情。
```

## 二、JSBridge 的通信原理

### 2.1 JS → Native 的两种方式

#### 方式一：URL Scheme 拦截（最通用）

```js
// JS 端：通过 iframe/location 发起特殊 URL
function callNative(action, params, callback) {
  const id = Date.now();
  // 保存回调
  window._callbacks[id] = callback;

  // 创建隐藏 iframe 发起请求
  const iframe = document.createElement('iframe');
  iframe.style.display = 'none';
  iframe.src = `jsbridge://${action}?params=${encodeURIComponent(JSON.stringify(params))}&callback=${id}`;
  document.body.appendChild(iframe);
  setTimeout(() => document.body.removeChild(iframe), 100);
}
```

```java
// Android 端：拦截 shouldOverrideUrlLoading
webView.setWebViewClient(new WebViewClient() {
    @Override
    public boolean shouldOverrideUrlLoading(WebView view, String url) {
        if (url.startsWith("jsbridge://")) {
            // 解析 action 和 params
            // 执行原生操作
            // 通过 evaluateJavascript 回调结果
            return true;
        }
        return false;
    }
});
```

```objectivec
// iOS 端：拦截 decidePolicyForNavigationAction
- (void)webView:(WKWebView *)webView
    decidePolicyForNavigationAction:(WKNavigationAction *)navigationAction
                    decisionHandler:(void (^)(WKNavigationActionPolicy))decisionHandler {
    NSURL *url = navigationAction.request.URL;
    if ([url.scheme isEqualToString:@"jsbridge"]) {
        // 解析 action 和 params
        // 执行原生操作
        // 通过 evaluateJavaScript 回调结果
        decisionHandler(WKNavigationActionPolicyCancel);
        return;
    }
    decisionHandler(WKNavigationActionPolicyAllow);
}
```

**优点**：兼容性好（所有 WebView 都支持拦截请求）
**缺点**：
- URL 长度有限制（约 2KB），不适合传递大数据
- 用 iframe 发起请求的方式有性能开销
- 回调需要手动管理（通过 callbackId）

#### 方式二：JavaScript 注入（主流）

```java
// Android：通过 addJavascriptInterface 注入对象
webView.addJavascriptInterface(new Object() {
    @JavascriptInterface
    public String callNative(String action, String params) {
        // 执行原生操作
        return result;
    }
}, "NativeBridge");
```

```objectivec
// iOS WKWebView：通过 userContentController 注入
WKUserContentController *controller = webView.configuration.userContentController;
[controller addScriptMessageHandler:self name:@"NativeBridge"];

// JS 端
window.webkit.messageHandlers.NativeBridge.postMessage({
    action: "scan",
    params: { type: "qr" }
});
```

```js
// JS 端统一调用
const bridge = {
  call(action, params) {
    return new Promise((resolve, reject) => {
      const id = Date.now();
      window._callbacks[id] = { resolve, reject };

      // iOS
      if (window.webkit?.messageHandlers?.NativeBridge) {
        window.webkit.messageHandlers.NativeBridge.postMessage({ id, action, params });
        return;
      }

      // Android
      if (window.NativeBridge) {
        try {
          const result = JSON.parse(window.NativeBridge.callNative(action, JSON.stringify(params)));
          resolve(result);
        } catch (e) {
          reject(e);
        }
        return;
      }
    });
  },
};
```

```java
// Android 原生端回调
webView.evaluateJavascript("window._callbacks[" + id + "].resolve(" + result + ")", null);
```

**优点**：性能好、无长度限制、双向通信方便
**缺点**：Android `addJavascriptInterface` 在 4.2 之前有安全漏洞（需 `@JavascriptInterface` 注解）

### 2.2 Native → JS

```java
// Android
webView.evaluateJavascript("javascript:onNativeEvent(" + jsonData + ")", null);

// iOS WKWebView
[webView evaluateJavaScript:@"onNativeEvent(\(jsonData))" completionHandler:nil];
```

```js
// JS 端接收
window.onNativeEvent = function(data) {
  const { type, payload } = data;
  if (type === 'scanResult') {
    console.log('扫描结果：', payload);
  }
  if (type === 'networkChange') {
    console.log('网络状态变化：', payload);
  }
};
```

### 2.3 完整封装

```ts
// bridge.ts
type Callback = (result: any) => void;

class JSBridge {
  private callbacks: Map<number, { resolve: Function; reject: Function }> = new Map();
  private id = 0;

  // 统一调用原生
  call<T = any>(action: string, params?: Record<string, any>): Promise<T> {
    return new Promise((resolve, reject) => {
      const callId = ++this.id;
      this.callbacks.set(callId, { resolve, reject });

      const message = { id: callId, action, params: params || {} };

      // iOS WKWebView
      if (window.webkit?.messageHandlers?.NativeBridge) {
        window.webkit.messageHandlers.NativeBridge.postMessage(message);
        return;
      }

      // Android
      if (window.NativeBridge) {
        try {
          const result = JSON.parse(
            window.NativeBridge.callNative(action, JSON.stringify(params))
          );
          resolve(result);
        } catch (e) {
          reject(e);
        }
        return;
      }

      // URL Scheme 降级
      this._callViaURLScheme(action, params, callId);
    });
  }

  // 原生回调 JS
  onNativeCallback(callId: number, result: any) {
    const cb = this.callbacks.get(callId);
    if (cb) {
      cb.resolve(result);
      this.callbacks.delete(callId);
    }
  }

  // 原生主动推送事件
  onNativeEvent(event: string, data: any) {
    const handlers = this._eventHandlers.get(event);
    handlers?.forEach(h => h(data));
  }

  private _eventHandlers = new Map<string, Set<Function>>();
  on(event: string, handler: Function) {
    if (!this._eventHandlers.has(event)) {
      this._eventHandlers.set(event, new Set());
    }
    this._eventHandlers.get(event)!.add(handler);
  }
  off(event: string, handler: Function) {
    this._eventHandlers.get(event)?.delete(handler);
  }

  // URL Scheme 降级方案
  private _callViaURLScheme(action: string, params: Record<string, any>, callId: number) {
    const url = `jsbridge://${action}?params=${encodeURIComponent(JSON.stringify(params))}&cb=${callId}`;
    const iframe = document.createElement('iframe');
    iframe.style.display = 'none';
    iframe.src = url;
    document.body.appendChild(iframe);
    setTimeout(() => document.body.removeChild(iframe), 100);
  }
}

export const bridge = new JSBridge();
```

## 三、常见 JSBridge API

```ts
// 设备信息
const deviceInfo = await bridge.call('getDeviceInfo');
// { platform: 'ios', osVersion: '16.0', brand: 'iPhone' }

// 相机扫描
const result = await bridge.call('scanQRCode', { type: 'qr' });
// { code: 'https://example.com', type: 'QR_CODE' }

// 获取位置
const location = await bridge.call('getLocation');
// { latitude: 39.9, longitude: 116.4 }

// 本地存储
await bridge.call('setStorage', { key: 'token', value: 'xxx' });
const token = await bridge.call('getStorage', { key: 'token' });

// 分享
await bridge.call('share', {
  title: '分享标题',
  url: 'https://example.com',
  image: 'https://example.com/icon.png',
});

// 支付
const payResult = await bridge.call('pay', {
  orderId: '123456',
  amount: 99.9,
});

// 原生弹窗
await bridge.call('showToast', { message: '操作成功' });

// 获取网络状态
const network = await bridge.call('getNetworkType');
// { type: 'wifi' }
```

## 四、小程序中的 JSBridge

小程序的 JSBridge 对外不可见，由框架内部封装。但理解其原理有助于调试性能问题：

```text
小程序双线程通信：
  ┌──────────┐    setData (JSBridge)    ┌──────────┐
  │  逻辑层   │ ◄─────────────────────► │  渲染层   │
  │  JSCore   │      JSON 序列化         │  WebView  │
  └──────────┘                          └──────────┘

小程序原生能力调用：
  逻辑层 JS → wx API → JSBridge → Native SDK
  wx.requestPayment / wx.getLocation / wx.chooseImage
  都是通过内部 JSBridge 调用原生实现的。
```

```text
与 Hybrid H5 的差异：
  ├── 小程序 JSBridge 由框架统一管理，开发者不直接接触
  ├── 小程序的通信更高效（序列化优化、批量处理）
  ├── Hybrid H5 需要自行封装 JSBridge
  └── 小程序的 API 一致性更好（各平台差异由框架抹平）
```

## 五、安全考虑

### 5.1 URL Scheme 劫持

```text
问题：外部 App 可以注册相同 scheme，拦截通信。
方案：
  ├── 使用私有 scheme（如 com.example.app://）
  ├── 添加 token 或签名验证来源
  └── 不使用敏感数据传输
```

### 5.2 注入接口的权限控制

```java
// Android：@JavascriptInterface 接口应做权限校验
@JavascriptInterface
public String callNative(String action, String params) {
    // 白名单校验：只允许特定域名调用
    String origin = mWebView.getUrl();
    if (!isAllowedOrigin(origin)) {
        return JSON.stringify({ code: -1, message: "Unauthorized" });
    }
    // 执行操作
}
```

### 5.3 XSS 注入防护

```js
// 所有从原生接收的数据都做转义
bridge.on('userInfo', (data) => {
  // ❌ 直接 innerHTML
  element.innerHTML = data.name;

  // ✅ 使用 textContent 或转义
  element.textContent = data.name;
});
```

## 六、JSBridge 的演进

```text
第一代：URL Scheme 拦截
  原理：通过 iframe.src 发起请求，原生拦截 shouldOverrideUrlLoading
  缺点：URL 长度限制、性能差

第二代：JavaScript 注入
  Android：addJavascriptInterface
  iOS：WKUserContentController
  优点：性能好、双向通信
  缺点：Android 4.2 之前有安全漏洞

第三代：DSBridge（现代方案）
  https://github.com/wendux/DSBridge-Android
  特点：
    ├── 支持同步/异步调用
    ├── 支持命名空间
    ├── 支持 TypeScript
    └── 统一 Android + iOS API

第四代：Flutter WebView + JavaScript Channel
  Flutter WebView 也内置了 JSBridge 机制，
  通过 JavaScriptChannel 实现 Dart ↔ JS 通信。
```

## 七、面试题

### Q1: JSBridge 的原理是什么

```text
JSBridge 的核心是实现 JS 与 Native 的双向通信。

JS → Native 的两种方式：
  1. URL Scheme 拦截：通过 iframe 发起特殊协议 URL，
     原生拦截 shouldOverrideUrlLoading
  2. JavaScript 注入：原生向 WebView 注入全局对象，
     JS 直接调用该对象的方法

Native → JS：
  通过 evaluateJavaScript 执行 JS 回调函数

完整流程：
  JS 调用 → 序列化参数 → 传给原生 → 原生执行 → 
  序列化结果 → evaluateJavaScript 回调 → JS 接收
```

### Q2: 为什么 Hybrid App 要用 JSBridge 而不是纯 H5

```text
纯 H5 的限制：
  1. 无法调用系统硬件（相机、蓝牙、NFC）
  2. 无法使用原生 SDK（微信支付、人脸识别）
  3. 性能不如原生（尤其长列表、复杂动画）
  4. 无法离线推送通知

JSBridge 让 H5 获得原生能力，同时保持 H5 的跨平台和热更新优势。
```

### Q3: 如何设计一个 JSBridge 的 API

```text
设计原则：
  1. 统一调用方式：所有 API 用 Promise 封装，保持 JS 侧一致
  2. 回调管理：用 callId 映射 Promise resolve/reject
  3. 参数序列化：统一 JSON 格式，原生端做校验
  4. 超时处理：每个调用设置超时，超时 reject
  5. 版本兼容：原生新增 API 时，JS 侧做降级

示例：
  bridge.call('scanQRCode', { type: 'qr' })
    .then(result => console.log(result.code))
    .catch(err => console.error(err))
```

### Q4: JSBridge 和 WebSocket 的区别

```text
JSBridge：WebView ←→ Native 的通信桥梁
  ├── 作用：让 H5 调用原生能力、原生推送事件给 H5
  ├── 实现：URL Scheme / JavaScript 注入
  └── 传输：同步/异步，数据量适中

WebSocket：客户端 ←→ 服务器的双向通信
  ├── 作用：实时数据传输、推送
  ├── 实现：TCP 协议
  └── 传输：全双工，数据量大

两者是互补关系——JSBridge 连接 WebView 和原生，
WebSocket 连接客户端和服务器。
```

### Q5: 如果原生端 JSBridge 还未注入，H5 调用会怎么样

```text
1. 如果原生注入晚于 H5 加载，会出现"调用时 NativeBridge 未定义"
2. 解决方案：
   - 在 H5 中做队列缓存，等 bridge 就绪后统一执行
   - 原生先注入 bridge，再加载 H5
   - H5 轮询检测 bridge 是否就绪

示例：
  function ensureBridge() {
    return new Promise((resolve) => {
      if (window.NativeBridge || window.webkit?.messageHandlers?.NativeBridge) {
        resolve();
        return;
      }
      const check = setInterval(() => {
        if (window.NativeBridge || window.webkit?.messageHandlers?.NativeBridge) {
          clearInterval(check);
          resolve();
        }
      }, 10);
    });
  }

  async function init() {
    await ensureBridge();
    const result = await bridge.call('getLocation');
  }
```
