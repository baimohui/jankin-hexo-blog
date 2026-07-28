---
title: Canvas 详解
categories: 
- 浏览器
tags:
- 浏览器
- Canvas
- 绘图
- 图形学
---

## 一、Canvas 是什么

Canvas（画布）是 HTML5 提供的一个通过 JavaScript 绘制图形的 API。它提供了一个 `<canvas>` 元素和一个 2D 渲染上下文（`CanvasRenderingContext2D`），允许开发者以编程方式绘制位图图形。<!--more-->

### Canvas vs DOM vs SVG

| 维度 | Canvas | DOM / CSS | SVG |
|------|--------|-----------|-----|
| 渲染方式 | 位图（像素） | 矢量（元素） | 矢量（元素） |
| 图形数量 | 数十万个无压力 | 数千个开始卡顿 | 数千个开始卡顿 |
| 交互 | 需手动检测点击位置 | 原生事件绑定 | 原生事件绑定 |
| 动画 | 按帧重绘全部或部分 | CSS transition/animation | 操作 DOM 属性 |
| 分辨率 | 受设备像素比影响（需适配） | 自动适配 | 自动适配 |
| 文本选择 | 不支持 | 支持 | 支持 |
| 使用场景 | 游戏、图表、图像处理、数据可视化 | 日常 UI 布局 | 图标、插画、可缩放图形 |

## 二、基础用法

### 获取上下文

```html
<canvas id="c" width="800" height="600"></canvas>
```

```js
const canvas = document.getElementById('c');
const ctx = canvas.getContext('2d');

// 检查支持
if (!ctx) {
  // 不支持 Canvas，降级处理
}
```

`width` / `height` 是画布的实际像素尺寸，**不是 CSS 尺寸**。CSS 尺寸只决定显示大小，不影响绘图精度。

### 坐标系

Canvas 坐标系原点在左上角，x 轴向右，y 轴向下：

```
(0,0) ──── x →
  │
  y
  ↓
```

## 三、绘图 API

### 矩形

```js
ctx.fillStyle = 'blue';
ctx.fillRect(10, 10, 100, 50);       // 填充矩形

ctx.strokeStyle = 'red';
ctx.lineWidth = 2;
ctx.strokeRect(10, 70, 100, 50);     // 描边矩形

ctx.clearRect(10, 10, 100, 50);      // 清除矩形区域（擦除）
```

### 路径

路径是 Canvas 最核心的绘图方式，所有非矩形的形状都通过路径绘制。

```js
// 三角形
ctx.beginPath();
ctx.moveTo(50, 50);       // 起点
ctx.lineTo(200, 50);      // 画线到
ctx.lineTo(125, 150);
ctx.closePath();           // 闭合路径
ctx.fillStyle = '#409eff';
ctx.fill();                // 填充
ctx.strokeStyle = '#333';
ctx.lineWidth = 2;
ctx.stroke();              // 描边

// 圆弧
ctx.beginPath();
ctx.arc(100, 100, 50, 0, Math.PI * 2);   // 圆心x, 圆心y, 半径, 起始角, 结束角
ctx.fillStyle = '#67c23a';
ctx.fill();

// 贝塞尔曲线
ctx.beginPath();
ctx.moveTo(20, 100);
ctx.quadraticCurveTo(80, 20, 140, 100);   // 二次贝塞尔：控制点, 终点
ctx.stroke();

ctx.beginPath();
ctx.moveTo(20, 150);
ctx.bezierCurveTo(60, 50, 120, 180, 180, 100);  // 三次贝塞尔：控制点1, 控制点2, 终点
ctx.stroke();
```

### 样式与颜色

```js
// 填充色
ctx.fillStyle = 'red';                    // 颜色名
ctx.fillStyle = '#f56c6c';                // 十六进制
ctx.fillStyle = 'rgba(64, 158, 255, .8)'; // RGBA
ctx.fillStyle = ctx.createLinearGradient(0, 0, 200, 0);  // 渐变

// 描边色
ctx.strokeStyle = '#333';
ctx.lineWidth = 2;
ctx.lineCap = 'round';    // 端点样式：butt/round/square
ctx.lineJoin = 'bevel';   // 连接点样式：miter/bevel/round

// 渐变
const gradient = ctx.createLinearGradient(0, 0, 200, 0);
gradient.addColorStop(0, '#409eff');
gradient.addColorStop(1, '#f56c6c');

const radialGrad = ctx.createRadialGradient(100, 100, 10, 100, 100, 50);
radialGrad.addColorStop(0, '#fff');
radialGrad.addColorStop(1, '#409eff');

// 阴影
ctx.shadowColor = 'rgba(0, 0, 0, .3)';
ctx.shadowBlur = 10;
ctx.shadowOffsetX = 2;
ctx.shadowOffsetY = 2;
```

### 文本

```js
ctx.font = 'bold 24px "Microsoft YaHei", sans-serif';
ctx.fillStyle = '#333';
ctx.textAlign = 'left';       // left/center/right
ctx.textBaseline = 'middle';  // top/middle/bottom
ctx.fillText('Hello Canvas', 100, 100);

// 描边文字
ctx.strokeStyle = '#409eff';
ctx.lineWidth = 1;
ctx.strokeText('Hello Canvas', 100, 150);

// 文本度量
const metrics = ctx.measureText('Hello Canvas');
console.log(metrics.width);    // 文本宽度
```

### 图片

```js
const img = new Image();
img.src = 'image.png';
img.onload = () => {
  // 绘制原图
  ctx.drawImage(img, 0, 0);

  // 缩放绘制
  ctx.drawImage(img, 0, 0, 200, 150);

  // 裁剪绘制：原图裁剪区域 (sx, sy, sw, sh) → 目标位置 (dx, dy, dw, dh)
  ctx.drawImage(img, 50, 50, 100, 100, 0, 0, 200, 200);
};
```

### 像素操作

```js
// 获取像素数据
const imageData = ctx.getImageData(0, 0, 100, 100);
// imageData.data 是 Uint8ClampedArray，按 RGBA 顺序排列

// 遍历像素（灰度滤镜）
const data = imageData.data;
for (let i = 0; i < data.length; i += 4) {
  const gray = data[i] * 0.299 + data[i + 1] * 0.587 + data[i + 2] * 0.114;
  data[i] = gray;           // R
  data[i + 1] = gray;       // G
  data[i + 2] = gray;       // B
  // data[i + 3] = alpha;   // A 保持不变
}

// 写回像素
ctx.putImageData(imageData, 0, 0);

// 创建空白像素数据
const emptyData = ctx.createImageData(100, 100);
```

## 四、变换与状态管理

### 基本变换

```js
ctx.translate(100, 100);    // 平移原点
ctx.rotate(Math.PI / 4);    // 旋转（弧度）
ctx.scale(2, 0.5);          // 缩放

// 绘制：所有坐标相对于变换后的坐标系
ctx.fillRect(0, 0, 50, 50);
```

### save / restore（状态栈）

`save` 保存当前状态（变换矩阵、样式、阴影等）到栈中，`restore` 恢复上一个状态：

```js
ctx.save();                            // 保存初始状态

ctx.translate(100, 100);
ctx.fillStyle = 'red';
ctx.fillRect(0, 0, 50, 50);

ctx.restore();                         // 恢复：translate 和 fillStyle 都还原了

ctx.fillRect(0, 0, 50, 50);            // 不受上面变换影响
```

推荐的绘画模式：

```js
function drawStar(ctx, cx, cy, r) {
  ctx.save();
  ctx.translate(cx, cy);
  // ... 相对 (0,0) 绘制
  ctx.restore();
}
```

## 五、动画

Canvas 动画的核心是 `requestAnimationFrame` 循环：

```js
function animate() {
  // 1. 清空画布
  ctx.clearRect(0, 0, canvas.width, canvas.height);

  // 2. 更新状态（位置、角度等）
  x += vx;
  y += vy;

  // 3. 绘制当前帧
  ctx.beginPath();
  ctx.arc(x, y, 20, 0, Math.PI * 2);
  ctx.fill();

  // 4. 请求下一帧
  requestAnimationFrame(animate);
}

animate();
```

### 帧率控制

```js
let lastTime = 0;
const FPS = 60;
const interval = 1000 / FPS;

function animate(timestamp) {
  const delta = timestamp - lastTime;
  if (delta >= interval) {
    lastTime = timestamp - (delta % interval);
    update();   // 更新逻辑
    render();   // 渲染
  }
  requestAnimationFrame(animate);
}
```

## 六、Canvas 与设备像素比

Canvas 默认的 1px 等于 1 个 CSS 像素。在高 DPR 屏幕上会模糊。解决方法是将画布的像素尺寸乘以 `devicePixelRatio`：

```js
function setupCanvas(canvas, width, height) {
  const dpr = window.devicePixelRatio || 1;
  canvas.width = width * dpr;
  canvas.height = height * dpr;
  canvas.style.width = width + 'px';
  canvas.style.height = height + 'px';

  const ctx = canvas.getContext('2d');
  ctx.scale(dpr, dpr);     // 缩放上下文，使绘图坐标与 CSS 像素对齐
  return ctx;
}

// 使用
const ctx = setupCanvas(canvas, 400, 300);
ctx.fillRect(0, 0, 100, 50);   // 在 Retina 屏幕上会清晰显示
```

## 七、性能优化

### 1. 减少绘制调用

```js
// ❌ 逐个绘制大量图形
for (const item of data) {
  ctx.beginPath();
  ctx.arc(item.x, item.y, 2, 0, Math.PI * 2);
  ctx.fill();
}

// ✅ 批量设置样式，减少状态切换
ctx.fillStyle = 'blue';
ctx.beginPath();
for (const item of data) {
  ctx.moveTo(item.x + 2, item.y);
  ctx.arc(item.x, item.y, 2, 0, Math.PI * 2);
}
ctx.fill();
```

### 2. 只更新变化区域

```js
// ❌ 全屏清除然后重绘所有
function render() {
  ctx.clearRect(0, 0, canvas.width, canvas.height);
  drawAll();
}

// ✅ 只清除变化区域
function render() {
  // 只清除并重绘上一个位置和当前位置
  clearPreviousPosition();
  drawCurrentPosition();
}
```

### 3. 离屏 Canvas

将静态内容绘制到离屏 Canvas 上，避免每帧重复绘制：

```js
// 创建离屏 Canvas
const offscreen = document.createElement('canvas');
const offCtx = offscreen.getContext('2d');
offscreen.width = 800;
offscreen.height = 600;

// 一次性绘制静态背景
drawBackground(offCtx);

// 动画循环中只需绘制动态内容 + 合成
function render() {
  ctx.clearRect(0, 0, canvas.width, canvas.height);
  ctx.drawImage(offscreen, 0, 0);    // 快速合成离屏画布
  drawDynamicContent(ctx);            // 绘制动态内容
}
```

### 4. 使用整数坐标

子像素渲染需要额外的抗锯齿计算，尽量使用整数坐标：

```js
// ❌ 浮点坐标
ctx.fillRect(10.3, 20.7, 100, 50);

// ✅ 整数坐标
ctx.fillRect(Math.round(x), Math.round(y), 100, 50);
```

## 八、实际应用示例

### 绘制柱状图

```js
function drawBarChart(ctx, data, { width, height, padding = 40 }) {
  const chartW = width - padding * 2;
  const chartH = height - padding * 2;
  const max = Math.max(...data.map(d => d.value));
  const barWidth = chartW / data.length * 0.6;
  const gap = chartW / data.length * 0.4;

  ctx.clearRect(0, 0, width, height);

  data.forEach((item, i) => {
    const x = padding + i * (barWidth + gap);
    const barH = (item.value / max) * chartH;
    const y = padding + chartH - barH;

    // 柱体
    ctx.fillStyle = item.color || '#409eff';
    ctx.fillRect(x, y, barWidth, barH);

    // 标签
    ctx.fillStyle = '#666';
    ctx.font = '12px sans-serif';
    ctx.textAlign = 'center';
    ctx.fillText(item.label, x + barWidth / 2, height - padding + 16);

    // 数值
    ctx.fillText(item.value, x + barWidth / 2, y - 6);
  });
}

drawBarChart(ctx, [
  { label: 'Mon', value: 120, color: '#409eff' },
  { label: 'Tue', value: 200, color: '#67c23a' },
  { label: 'Wed', value: 150, color: '#e6a23c' },
], { width: 400, height: 300 });
```

### 鼠标交互（拾取检测）

```js
const circles = [
  { x: 100, y: 100, r: 40, color: '#409eff' },
  { x: 200, y: 150, r: 30, color: '#f56c6c' },
];

function draw() {
  ctx.clearRect(0, 0, canvas.width, canvas.height);
  circles.forEach(c => {
    ctx.beginPath();
    ctx.arc(c.x, c.y, c.r, 0, Math.PI * 2);
    ctx.fillStyle = c.color;
    ctx.fill();
  });
}

// 拾取检测：根据鼠标位置判断点击了哪个图形
canvas.addEventListener('click', (e) => {
  const rect = canvas.getBoundingClientRect();
  const mx = e.clientX - rect.left;
  const my = e.clientY - rect.top;

  // 从后往前遍历（上层图形优先）
  for (let i = circles.length - 1; i >= 0; i--) {
    const c = circles[i];
    const dist = Math.hypot(mx - c.x, my - c.y);
    if (dist <= c.r) {
      console.log(`点击了圆形 ${i}`);
      break;
    }
  }
});

draw();
```

对于大量图形的拾取，可以用**离屏 Canvas 颜色索引法**：每个图形用唯一颜色绘制到离屏画布，鼠标位置读取像素颜色来识别图形。

## 九、常见问题

### Q1: Canvas 绘制出来的文字模糊

原因通常是 DPR 没有适配。参考第六节的 `setupCanvas` 方法。

### Q2: 动画中如何限帧

```js
let lastTime = 0;
function animate(time) {
  if (time - lastTime >= 16) {   // ~60fps
    lastTime = time;
    render();
  }
  requestAnimationFrame(animate);
}
```

### Q3: Canvas 内容如何导出为图片

```js
canvas.toBlob((blob) => {
  const url = URL.createObjectURL(blob);
  const a = document.createElement('a');
  a.href = url;
  a.download = 'chart.png';
  a.click();
}, 'image/png');
```

### Q4: Canvas 与 WebGL 怎么选

| 场景 | 推荐 |
|------|------|
| 2D 绘图、图表、图像处理 | Canvas 2D |
| 大量粒子系统 | Canvas 2D 或 WebGL |
| 3D 渲染、游戏 | WebGL / Three.js |
| 高性能 2D 游戏 | WebGL 2D |

## 十、推荐学习路径

1. 掌握基础绘图 API：矩形、路径、圆弧、文本、图片
2. 理解变换矩阵和 `save/restore` 状态管理
3. 用 `requestAnimationFrame` 做动画，理解帧率控制
4. 适配 DPR 解决 Retina 屏幕模糊
5. 学习离屏 Canvas 和像素操作
6. 结合实际场景（图表、小游戏、图像滤镜）练习
