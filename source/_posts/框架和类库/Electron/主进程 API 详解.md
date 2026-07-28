---
title: Electron 主进程 API 详解
categories: 
- 框架和类库
tags:
- Electron
- 主进程
- BrowserWindow
- IPC
- Menu
- 原生API
---

## 一、BrowserWindow——窗口管理

BrowserWindow 是 Electron 中最核心的 API，负责创建和管理应用窗口。<!--more-->

### 创建窗口

```js
const { BrowserWindow } = require('electron');

const win = new BrowserWindow({
  width: 1024,
  height: 768,
  minWidth: 800,
  minHeight: 600,
  frame: true,                // 是否显示标题栏
  autoHideMenuBar: true,      // 自动隐藏菜单栏（Alt 显示）
  resizable: true,
  maximizable: true,
  fullscreenable: true,
  alwaysOnTop: false,         // 置顶
  transparent: false,         // 透明窗口
  titleBarStyle: 'hiddenInset', // macOS 隐藏标题栏但保留交通灯按钮
  webPreferences: {
    preload: path.join(__dirname, 'preload.js'),
    nodeIntegration: false,
    contextIsolation: true,
  },
});

win.loadFile('index.html');             // 加载本地文件
win.loadURL('https://example.com');     // 加载远程页面
```

### 窗口操作

```js
win.show();                     // 显示
win.hide();                     // 隐藏
win.minimize();                 // 最小化
win.maximize();                 // 最大化
win.unmaximize();               // 恢复
win.close();                    // 关闭
win.focus();                    // 聚焦

win.setSize(1280, 720);         // 设置大小
win.setMinimumSize(800, 600);   // 最小尺寸
win.setPosition(100, 100);      // 设置位置

win.isMaximized();              // 是否最大化
win.isFullScreen();             // 是否全屏

win.setFullScreen(true);        // 全屏
```

### 窗口事件

```js
win.on('closed', () => { /* 窗口关闭 */ });
win.on('focus', () => { /* 窗口获得焦点 */ });
win.on('blur', () => { /* 窗口失去焦点 */ });
win.on('maximize', () => {});
win.on('minimize', () => {});
win.on('resize', () => {});
win.on('move', () => {});
```

### 多窗口管理

```js
function createWindow(url) {
  const win = new BrowserWindow({ width: 800, height: 600 });
  win.loadURL(url);
  return win;
}

// 管理所有窗口
const windows = new Set();

function addWindow(url) {
  const win = createWindow(url);
  windows.add(win);
  win.on('closed', () => windows.delete(win));
}
```

## 二、IPC——进程间通信

Electron 提供多种 IPC 通信方式。

### 方式一：invoke / handle（请求-响应模式，推荐）

```js
// preload.js
contextBridge.exposeInMainWorld('api', {
  getUser: (id) => ipcRenderer.invoke('get-user', id),
});
```

```js
// main.js
ipcMain.handle('get-user', async (event, userId) => {
  const user = await db.findUser(userId);
  return user;
});
```

### 方式二：send / on（单向/双向消息）

```js
// 渲染进程 → 主进程
ipcRenderer.send('message', 'hello');

// 主进程接收
ipcMain.on('message', (event, arg) => {
  console.log(arg);  // 'hello'
  event.reply('reply', 'pong');  // 回复
});

// 渲染进程接收回复
ipcRenderer.on('reply', (event, arg) => {
  console.log(arg);  // 'pong'
});
```

### 方式三：主进程向渲染进程发送消息

```js
// main.js
win.webContents.send('update-available', { version: '2.0.0' });
```

```js
// preload.js
contextBridge.exposeInMainWorld('api', {
  onUpdate: (callback) => ipcRenderer.on('update-available', callback),
});
```

### 方式四：渲染进程之间通信

渲染进程不能直接通信，需要通过主进程转发：

```js
// 渲染进程 A
ipcRenderer.send('forward', { target: 'winB', data: 'hello' });

// 主进程
ipcMain.on('forward', (event, { target, data }) => {
  const targetWin = BrowserWindow.getAllWindows()
    .find(w => w.title === target);
  targetWin?.webContents.send('message', data);
});
```

## 三、Menu——原生菜单

### 应用菜单

```js
const { Menu, app } = require('electron');

const menu = Menu.buildFromTemplate([
  {
    label: '文件',
    submenu: [
      {
        label: '打开',
        accelerator: 'CmdOrCtrl+O',
        click: () => { /* 打开文件 */ },
      },
      { type: 'separator' },
      {
        label: '保存',
        accelerator: 'CmdOrCtrl+S',
        enabled: false,           // 禁用
        click: () => { /* 保存 */ },
      },
      { type: 'separator' },
      { role: 'quit' },           // 内置角色：退出
    ],
  },
  {
    label: '编辑',
    submenu: [
      { role: 'undo' },
      { role: 'redo' },
      { type: 'separator' },
      { role: 'cut' },
      { role: 'copy' },
      { role: 'paste' },
      { role: 'selectAll' },
    ],
  },
  {
    label: '视图',
    submenu: [
      { role: 'reload' },
      { role: 'toggleDevTools' },
      { type: 'separator' },
      { role: 'zoomIn' },
      { role: 'zoomOut' },
      { role: 'resetZoom' },
    ],
  },
]);

Menu.setApplicationMenu(menu);
```

### 上下文菜单（右键菜单）

```js
const { Menu, ipcMain } = require('electron');

ipcMain.on('show-context-menu', (event) => {
  const template = [
    { label: '复制', role: 'copy' },
    { label: '粘贴', role: 'paste' },
    { type: 'separator' },
    { label: '删除', click: () => event.sender.send('delete-selected') },
  ];
  const menu = Menu.buildFromTemplate(template);
  menu.popup({ window: BrowserWindow.fromWebContents(event.sender) });
});
```

```js
// preload.js
contextBridge.exposeInMainWorld('api', {
  showMenu: () => ipcRenderer.send('show-context-menu'),
  onDelete: (cb) => ipcRenderer.on('delete-selected', cb),
});
```

### 内置角色（role）

内置角色简化了常用菜单项的实现：

| role | 作用 |
|------|------|
| `undo` / `redo` | 撤销/重做 |
| `cut` / `copy` / `paste` | 剪贴板操作 |
| `selectAll` | 全选 |
| `reload` | 重新加载 |
| `forceReload` | 强制重新加载 |
| `toggleDevTools` | 切换开发者工具 |
| `zoomIn` / `zoomOut` / `resetZoom` | 缩放控制 |
| `minimize` / `close` | 窗口操作 |
| `quit` | 退出应用 |
| `about` | 关于面板 |

## 四、Tray——系统托盘

```js
const { Tray, Menu, nativeImage } = require('electron');

const tray = new Tray('icon.png');   // 托盘图标（16x16 或 22x22）

const contextMenu = Menu.buildFromTemplate([
  { label: '显示窗口', click: () => mainWindow.show() },
  { label: '隐藏窗口', click: () => mainWindow.hide() },
  { type: 'separator' },
  { label: '退出', click: () => app.quit() },
]);

tray.setToolTip('My Electron App');
tray.setContextMenu(contextMenu);

// 点击托盘图标（macOS 表现不同）
tray.on('click', () => {
  mainWindow.isVisible() ? mainWindow.hide() : mainWindow.show();
});
```

## 五、Dialog——原生对话框

```js
const { dialog } = require('electron');

// 打开文件
const result = await dialog.showOpenDialog({
  title: '选择文件',
  defaultPath: '~/Downloads',
  filters: [
    { name: '图片', extensions: ['jpg', 'png', 'gif'] },
    { name: '所有文件', extensions: ['*'] },
  ],
  properties: ['openFile', 'multiSelections'],  // 多选
});
// result.canceled: boolean
// result.filePaths: string[]

// 保存文件
const saveResult = await dialog.showSaveDialog({
  title: '保存文件',
  defaultPath: 'untitled.txt',
  filters: [{ name: '文本文件', extensions: ['txt'] }],
});

// 消息提示
await dialog.showMessageBox({
  type: 'info',
  title: '提示',
  message: '操作成功完成',
  buttons: ['确定'],
  defaultId: 0,
});

// 错误提示
await dialog.showErrorBox('错误', '文件读取失败');
```

## 六、Notification——系统通知

```js
const { Notification } = require('electron');

// 主进程
new Notification({
  title: '更新完成',
  body: '应用已更新到最新版本',
  icon: 'icon.png',
}).show();
```

```js
// 渲染进程（HTML5 Notification 也可用）
new Notification('新消息', {
  body: '您收到一条新消息',
});
```

## 七、shell——文件管理

```js
const { shell } = require('electron');

// 在默认浏览器中打开 URL
shell.openExternal('https://example.com');

// 在文件管理器中显示文件
shell.showItemInFolder('/path/to/file.txt');

// 打开文件/目录（使用默认程序）
shell.openPath('/path/to/file.txt');

// 写入剪贴板
shell.writeClipboardText('Hello');
const text = shell.readClipboardText();
```

## 八、screen——屏幕信息

```js
const { screen } = require('electron');

const displays = screen.getAllDisplays();
const primaryDisplay = screen.getPrimaryDisplay();

// 主显示器尺寸
const { width, height } = primaryDisplay.workAreaSize;

// 鼠标所在显示器
const cursorPoint = screen.getCursorScreenPoint();
const currentDisplay = screen.getDisplayNearestPoint(cursorPoint);

// 可用屏幕区域（排除任务栏/Dock）
const workArea = primaryDisplay.workArea;
```

## 九、nativeTheme——主题检测

```js
const { nativeTheme } = require('electron');

// 当前系统主题
nativeTheme.themeSource;     // 'system' | 'light' | 'dark'
nativeTheme.shouldUseDarkColors;  // boolean

// 监听主题变化
nativeTheme.on('updated', () => {
  const isDark = nativeTheme.shouldUseDarkColors;
  // 通知渲染进程切换主题
  mainWindow.webContents.send('theme-changed', isDark);
});

// 强制设置主题
nativeTheme.themeSource = 'dark';
```

## 十、powerSaveBlocker——阻止休眠

```js
const { powerSaveBlocker } = require('electron');

const id = powerSaveBlocker.start('prevent-display-sleep');
// 或 'prevent-app-suspension'

// 检查是否在阻止状态
powerSaveBlocker.isStarted(id);

// 停止阻止
powerSaveBlocker.stop(id);
```

## 十一、常见问题

### Q1: 如何打开开发者工具

```js
// 在主进程中
win.webContents.openDevTools();

// 快捷键 Cmd+Option+I（macOS）/ Ctrl+Shift+I（Windows/Linux）
// 或在菜单中配置 { role: 'toggleDevTools' }
```

### Q2: 如何获取窗口当前的 URL

```js
const currentURL = win.webContents.getURL();
```

### Q3: 如何关闭某个特定窗口

```js
const targetWin = BrowserWindow.getAllWindows()
  .find(w => w.title === '设置');
targetWin?.close();
```
