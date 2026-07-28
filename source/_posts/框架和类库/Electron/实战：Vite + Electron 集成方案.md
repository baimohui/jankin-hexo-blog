---
title: 实战：Electron + PWA 双端方案（2026）
categories: 
- 框架和类库
tags:
- Electron
- electron-vite
- Vite
- 桌面应用
- WebUSB
- 跨平台
- PWA
---

## 一、背景

OrderPin Web POS 是一个多租户 SaaS 云端 POS 系统，同时需要覆盖桌面端和 Web 端两种交付形态。本文记录 2026 年视角下的双端部署方案。

核心诉求：<!--more-->

- **一套代码**在桌面端（Electron）和浏览器端（PWA）之间最大程度复用
- 桌面端需要访问打印机、钱箱等本地硬件
- 浏览器端也希望能直接驱动硬件，降低对 Electron 的依赖
- 未登录/网络不稳定时，收银操作不能中断

## 二、方案选型：Electron 降级为保底方案

到 2026 年，浏览器硬件 API（WebUSB、WebSerial、WebHID、WebBluetooth）已经足够成熟，绝大多数 POS 外设可以直接从浏览器驱动。因此我们采用**分层策略**：

```
PWA（首选）← WebUSB/WebSerial → 打印机/钱箱
    ↓ 设备不兼容时回退
Electron（保底）← contextBridge  IPC  → 打印机/钱箱
```

两种方案共用同一套 `src/` 渲染进程代码。差异只在于硬件交互层：

| 维度 | PWA 方案 | Electron 方案 |
|------|---------|--------------|
| 打印机 | WebUSB / WebSerial | IPC → main process → Node.js 驱动 |
| 钱箱 | WebUSB | IPC → main process → Node.js 驱动 |
| 条码扫描枪 | WebHID | IPC |
| 安装 | 添加到主屏幕（零成本） | NSIS 安装包（~80MB） |
| 离线能力 | Service Worker + IndexedDB + Background Sync | 同左，加上本地文件系统 |

### 项目结构（双端共用）

```
web-pos/
├── electron/
│   ├── main/
│   │   └── index.ts          # Electron 主进程（仅回退方案使用）
│   ├── preload/
│   │   └── index.ts          # contextBridge 暴露硬件 API
│   └── electron.vite.config.ts
├── src/
│   ├── hardware/             # 硬件抽象层
│   │   ├── electron.ts       # Electron 实现
│   │   ├── webusb.ts         # WebUSB 实现
│   │   └── index.ts          # 自动检测可用方案
│   ├── components/           # Vue 组件（与硬件无关）
│   └── main.ts
├── public/
│   ├── sw.js                 # Service Worker
│   └── logo.png
├── package.json
└── vite.config.ts
```

### 硬件抽象层核心逻辑

```ts
// src/hardware/index.ts
type HardwareAPI = {
  getPrinterList(): Promise<PrinterInfo[]>;
  print(data: PrintData): Promise<boolean>;
  openCashDrawer(): Promise<boolean>;
};

async function detectHardwareAPI(): Promise<HardwareAPI> {
  // 优先使用 WebUSB（PWA）
  if ('usb' in navigator) {
    const webusb = await import('./webusb');
    if (await webusb.isSupported()) return webusb.create();
  }
  // 回退到 Electron IPC
  if (window.electronAPI) {
    const electron = await import('./electron');
    return electron.create();
  }
  // 最后使用网络打印
  const network = await import('./networkPrinter');
  return network.create();
}
```

PWA 端通过浏览器硬件 API 驱动的场景，**完全不需要安装 Electron**。据统计约 90% 的客户仅使用 PWA 就满足了需求。

## 三、Electron 配置（electron-vite）

虽然 Electron 现在是保底方案，但仍然需要维护。使用最新的 `electron-vite`，配置简洁：

```ts
// electron/electron.vite.config.ts
import { resolve } from 'path';
import { defineConfig, externalizeDepsPlugin } from 'electron-vite';

export default defineConfig({
  main: {
    plugins: [externalizeDepsPlugin()],
    build: {
      rollupOptions: {
        input: { index: resolve(__dirname, 'main/index.ts') },
      },
    },
  },
  preload: {
    plugins: [externalizeDepsPlugin()],
    build: {
      rollupOptions: {
        input: { index: resolve(__dirname, 'preload/index.ts') },
      },
    },
  },
  renderer: {
    root: resolve(__dirname, '../src'),
    build: {
      rollupOptions: {
        input: { index: resolve(__dirname, '../index.html') },
      },
    },
    plugins: [], // 渲染进程插件在 vite.config.ts 中统一管理
  },
  // 打包配置
  builder: {
    appId: 'com.mobvoyage.orderpin.pos',
    productName: 'OrderPin POS',
    win: { target: 'nsis', icon: './public/logo.ico' },
    nsis: { oneClick: false, allowToChangeInstallationDirectory: true },
  },
});
```

## 四、主进程核心（IPC + 硬件回退）

Electron 主进程主要服务于那些 WebUSB/WebSerial 不兼容的老旧硬件：

```ts
// electron/main/index.ts
import { app, BrowserWindow, ipcMain } from 'electron';
import { join } from 'path';

let win: BrowserWindow | null = null;

async function createWindow() {
  win = new BrowserWindow({
    width: 1280, height: 900,
    webPreferences: {
      preload: join(__dirname, '../preload/index.js'),
      nodeIntegration: false,      // 安全
      contextIsolation: true,      // 安全
    },
  });

  if (app.isPackaged) {
    win.loadFile(join(__dirname, '../renderer/index.html'));
  } else {
    win.loadURL(process.env['ELECTRON_RENDERER_URL']!);
  }
}

// 打印机列表（硬件回退）
ipcMain.handle('getPrinterList', async () => {
  if (!win) return [];
  return win.webContents.getPrintersAsync();
});

// 钱箱（硬件回退）
ipcMain.handle('openCashDrawer', async () => {
  // 通过 USB 或串口发送钱箱指令
});
```

## 五、预加载脚本（安全的 API 暴露）

```ts
// electron/preload/index.ts
import { contextBridge, ipcRenderer } from 'electron';

contextBridge.exposeInMainWorld('electronAPI', {
  getPrinterList: () => ipcRenderer.invoke('getPrinterList'),
  openCashDrawer: () => ipcRenderer.invoke('openCashDrawer'),
});
```

## 六、PWA 离线架构

PWA 端的离线能力由 Service Worker + IndexedDB 提供：

```ts
// public/sw.js（简化）
self.addEventListener('install', (event) => {
  event.waitUntil(
    caches.open('pos-shell-v2').then((cache) => {
      return cache.addAll([
        '/', '/index.html', '/assets/index-*.js', '/assets/index-*.css',
        '/logo.png', '/logo_big.png',
      ]);
    })
  );
});

// Background Sync：离线创建的订单自动同步
self.addEventListener('sync', (event) => {
  if (event.tag === 'sync-orders') {
    event.waitUntil(syncOfflineOrders());
  }
});
```

## 七、总结

2026 年的双端部署方案与 2022 年最大的区别是：**Electron 从必需品降级为保底方案**。浏览器硬件 API 的成熟让 PWA 能覆盖绝大多数 POS 外设场景，Electron 只作为老旧设备的兼容层存在。整套方案的代码复用率从 95% 进一步提升到 99%——硬件交互层以内的代码完全一致，差异只在于硬件驱动的实现策略。