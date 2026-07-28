---
title: Blob 对象详解
categories: 
- 浏览器
tags:
- 浏览器
- Blob
- File
- ArrayBuffer
- 二进制数据
---

## 一、Blob 是什么

Blob（Binary Large Object）是浏览器中表示**不可变的原始二进制数据**的对象。它可以存储任意类型的数据（图片、文件、文本、音视频等），是浏览器文件处理的核心接口。<!--more-->

### Blob 与相关类型的关系

```
字符串/String
    ↓ new TextEncoder()
ArrayBuffer      ← →   Uint8Array / Int8Array / ...
    ↓ new Blob()
Blob
    ↓ new File()    （继承自 Blob）
File              ← 用户上传 / File API
    ↓
Object URL / Base64 / FormData（上传）
```

## 二、创建 Blob

```js
// 从字符串创建
const blob = new Blob(['<h1>Hello</h1>'], { type: 'text/html' });

// 从 ArrayBuffer 创建
const buffer = new ArrayBuffer(8);
const blob2 = new Blob([buffer]);

// 从多个数据片段创建（Blob 会自动合并）
const parts = [
  '<html><head><title>Test</title></head><body>',
  '<h1>Hello</h1>',
  '</body></html>',
];
const htmlBlob = new Blob(parts, { type: 'text/html' });
```

### MIME 类型

```js
// 常见 MIME 类型
{ type: 'text/plain' }           // 纯文本
{ type: 'text/html' }            // HTML
{ type: 'text/css' }             // CSS
{ type: 'application/json' }     // JSON
{ type: 'application/pdf' }      // PDF
{ type: 'application/zip' }      // ZIP
{ type: 'image/png' }            // PNG 图片
{ type: 'image/jpeg' }           // JPEG 图片
{ type: 'image/webp' }           // WebP 图片
{ type: 'audio/mpeg' }           // MP3 音频
{ type: 'video/mp4' }            // MP4 视频
{ type: 'application/octet-stream' }  // 通用二进制
```

## 三、读取 Blob

### 方式一：转为文本

```js
const blob = new Blob(['Hello, 世界!'], { type: 'text/plain' });

// 使用 text() 方法（返回 Promise）
const text = await blob.text();
console.log(text);   // "Hello, 世界!"
```

### 方式二：转为 ArrayBuffer

```js
const arrayBuffer = await blob.arrayBuffer();
const uint8Array = new Uint8Array(arrayBuffer);
console.log(uint8Array);   // Uint8Array(13)
```

### 方式三：转为 Base64 Data URL

```js
const reader = new FileReader();

reader.onload = () => {
  console.log(reader.result);
  // "data:text/plain;base64,SGVsbG8sIOS4lueVjCE="
};

reader.readAsDataURL(blob);
```

### 方式四：转为 Object URL

```js
const url = URL.createObjectURL(blob);
// "blob:http://localhost:3000/550e8400-e29b-41d4-a716-446655440000"

// 在页面中使用
img.src = url;
iframe.src = url;
a.href = url;

// 使用完毕后必须释放
URL.revokeObjectURL(url);
```

### 方式五：转为 Stream

```js
const stream = blob.stream();
const reader = stream.getReader();

async function read() {
  const { done, value } = await reader.read();
  if (!done) {
    console.log(value);  // Uint8Array 数据块
    read();
  }
}
```

## 四、Blob 属性与方法

```js
const blob = new Blob(['Hello World'], { type: 'text/plain' });

blob.size;     // 11（字节数）
blob.type;     // "text/plain"

// 切片（大文件分段处理）
const chunk1 = blob.slice(0, 5);    // "Hello"
const chunk2 = blob.slice(6, 11);   // "World"

// 切片时可以指定新的 MIME 类型
const chunk = blob.slice(0, 5, 'text/plain');
```

## 五、File（继承自 Blob）

`File` 是 `Blob` 的子类，扩展了文件名和最后修改时间属性：

```js
// 方式一：用户上传
<input type="file" id="fileInput" multiple>
<script>
  document.getElementById('fileInput').addEventListener('change', (e) => {
    const files = e.target.files;
    // FileList，每个元素是 File 对象
    const file = files[0];
    console.log(file.name);          // "photo.jpg"
    console.log(file.size);          // 102400
    console.log(file.type);          // "image/jpeg"
    console.log(file.lastModified);  // 时间戳
  });
</script>

// 方式二：手动创建 File
const file = new File(
  ['{"name":"Alice"}'],
  'user.json',
  { type: 'application/json' }
);

// 方式三：拖拽（DataTransfer）
dropZone.addEventListener('drop', (e) => {
  const files = e.dataTransfer.files;
});
```

### Blob vs File

```js
// File 是 Blob 的子类，所以 File 拥有 Blob 的所有方法
file instanceof Blob;     // true
file.text();              // ✅ 继承自 Blob
file.slice();             // ✅ 继承自 Blob
```

## 六、Blob URL vs Data URL

```js
// Blob URL
const blob = new Blob(['<h1>Hello</h1>'], { type: 'text/html' });
const blobUrl = URL.createObjectURL(blob);
// "blob:http://localhost:3000/uuid"
// 约为 50 字节，指向浏览器内存中的 Blob

// Data URL
const dataUrl = 'data:text/html;base64,' + btoa('<h1>Hello</h1>');
// "data:text/html;base64,PGgxPkhlbGxvPC9oMT4="
// 体积约为原始数据的 1.37 倍
```

| 特性 | Blob URL | Data URL |
|------|----------|----------|
| 长度 | 固定（约 50 字节） | 随数据增长（比原数据大 37%） |
| 请求 | 不发起 HTTP 请求 | 不发起 HTTP 请求 |
| 内存 | 引用浏览器内存中的 Blob | 编码在 URL 字符串中 |
| 释放 | 需手动 `revokeObjectURL` | 随页面关闭自动释放 |
| 适用 | 大文件（图片、视频） | 小文件（图标、短文本） |

## 七、实际应用场景

### 场景一：前端生成文件并下载

```js
function downloadJSON(data, filename = 'data.json') {
  const blob = new Blob([JSON.stringify(data, null, 2)], {
    type: 'application/json',
  });
  const url = URL.createObjectURL(blob);

  const a = document.createElement('a');
  a.href = url;
  a.download = filename;
  a.click();

  URL.revokeObjectURL(url);
}

// 使用
downloadJSON({ id: 1, name: 'Alice' }, 'user.json');
```

### 场景二：图片压缩预览

```js
<input type="file" accept="image/*" id="upload">
<img id="preview">

<script>
  document.getElementById('upload').addEventListener('change', async (e) => {
    const file = e.target.files[0];
    const compressedBlob = await compressImage(file, 0.5);
    const url = URL.createObjectURL(compressedBlob);
    document.getElementById('preview').src = url;
  });

  function compressImage(file, quality = 0.7) {
    return new Promise((resolve) => {
      const img = new Image();
      img.onload = () => {
        const canvas = document.createElement('canvas');
        canvas.width = img.width / 2;
        canvas.height = img.height / 2;
        const ctx = canvas.getContext('2d');
        ctx.drawImage(img, 0, 0, canvas.width, canvas.height);
        canvas.toBlob((blob) => resolve(blob), file.type, quality);
      };
      img.src = URL.createObjectURL(file);
    });
  }
</script>
```

### 场景三：大文件分片上传

```js
async function uploadFile(file) {
  const chunkSize = 1024 * 1024;  // 1MB 一片
  const totalChunks = Math.ceil(file.size / chunkSize);

  for (let i = 0; i < totalChunks; i++) {
    const start = i * chunkSize;
    const end = Math.min(start + chunkSize, file.size);
    const chunk = file.slice(start, end);   // Blob.slice

    const formData = new FormData();
    formData.append('chunk', chunk, file.name);
    formData.append('index', i);
    formData.append('total', totalChunks);

    await fetch('/api/upload', { method: 'POST', body: formData });
  }
}
```

### 场景四：动态创建脚本/样式

```js
// 动态加载脚本（用于隔离作用域或动态代码）
const code = `
  self.onmessage = (e) => {
    const result = e.data.map(x => x * 2);
    self.postMessage(result);
  };
`;
const blob = new Blob([code], { type: 'application/javascript' });
const worker = new Worker(URL.createObjectURL(blob));

worker.postMessage([1, 2, 3]);
worker.onmessage = (e) => console.log(e.data);  // [2, 4, 6]

// 动态插入样式
const cssBlob = new Blob([`
  .dynamic-style { color: red; font-size: 20px; }
`], { type: 'text/css' });
const link = document.createElement('link');
link.rel = 'stylesheet';
link.href = URL.createObjectURL(cssBlob);
document.head.appendChild(link);
```

### 场景五：Canvas 导出为图片

```js
const canvas = document.getElementById('myCanvas');

// 方式一：toBlob（推荐）
canvas.toBlob((blob) => {
  const url = URL.createObjectURL(blob);
  // 下载或展示
}, 'image/png');

// 方式二：toDataURL → Blob
const dataUrl = canvas.toDataURL('image/jpeg', 0.8);
const byteString = atob(dataUrl.split(',')[1]);
const ab = new ArrayBuffer(byteString.length);
const ia = new Uint8Array(ab);
for (let i = 0; i < byteString.length; i++) {
  ia[i] = byteString.charCodeAt(i);
}
const blob = new Blob([ab], { type: 'image/jpeg' });
```

## 八、Blob 与 ArrayBuffer 的对比

| 特性 | Blob | ArrayBuffer |
|------|------|-------------|
| 可变性 | 不可变 | 可通过 TypedArray 修改 |
| 编码 | 固定编码（MIME 类型） | 纯二进制，无编码信息 |
| 读取 | `text()`、`arrayBuffer()`、`stream()` | 需通过 DataView 或 TypedArray |
| 用途 | 文件、上传、下载、URL | WebGL、音频处理、加密、WebSocket |

```js
// Blob → ArrayBuffer
const buffer = await blob.arrayBuffer();

// ArrayBuffer → Blob
const blob = new Blob([buffer], { type: 'application/octet-stream' });

// 使用 TypedArray 操作二进制数据
const uint8 = new Uint8Array([72, 101, 108, 108, 111]);  // "Hello"
const textBlob = new Blob([uint8], { type: 'text/plain' });
```

## 九、兼容性

所有现代浏览器均完整支持 Blob、File、FileReader、Blob URL。IE 10+ 支持基本功能但不支持 `blob.text()`、`blob.stream()` 等 Promise 方法。

| API | 支持情况 |
|-----|----------|
| `new Blob()` | 所有浏览器 |
| `File` | IE 10+ |
| `FileReader` | IE 10+ |
| `URL.createObjectURL` | IE 10+ |
| `blob.text()` / `blob.arrayBuffer()` | Chrome 76+, Safari 14.1+ |
| `blob.stream()` | Chrome 76+, Safari 14.1+ |

## 十、常见问题

### Q1: createObjectURL 什么时候需要 revoke

每次 `createObjectURL` 都会占用内存映射。如果不需要了应立即 `revokeObjectURL`。对于图片预览，在 `img.onload` 后即可 revoke：

```js
const url = URL.createObjectURL(file);
const img = new Image();
img.onload = () => {
  URL.revokeObjectURL(url);   // 图片加载完成后立即释放
};
img.src = url;
```

### Q2: blob.text() 和 FileReader 有什么区别

`blob.text()` 是异步方法，返回 Promise，更简洁。`FileReader` 基于事件回调，可读性较差。推荐使用 `blob.text()` 和 `blob.arrayBuffer()`。

### Q3: 如何检测一个对象是不是 Blob

```js
blob instanceof Blob;        // true
Blob.prototype.isPrototypeOf(blob);  // true
```

## 十一、推荐学习路径

1. 掌握 Blob 的创建和读取（`text`、`arrayBuffer`、`stream`）
2. 理解 Blob URL 和 Data URL 的区别
3. 结合 File 对象做文件上传预览
4. 学会 `Blob.slice` 实现大文件分片
5. 结合 Canvas 做图片压缩和导出
