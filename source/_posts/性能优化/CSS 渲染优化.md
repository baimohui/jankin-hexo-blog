---
title: CSS 渲染优化
categories: 
- 性能优化
tags:
- 性能优化
- CSS
- 渲染
- content-visibility
- contain
- font-display
---

## 一、content-visibility——跳过屏幕外内容渲染

对于大量离屏内容，使用 `content-visibility: auto` 可大幅减少渲染时间。<!--more-->

```css
.card {
  content-visibility: auto;
  contain-intrinsic-size: 200px;  /* 保留占位空间 */
}
```

`content-visibility: auto` 将元素高度视为 0，浏览器会跳过其渲染，但可能导致滚动条异常。使用 `contain-intrinsic-size` 设置占位符尺寸来保持滚动条正常。

`content-visibility: hidden` 隐藏元素并保持渲染状态，再次显示时成本极低，优于 `display: none`。

## 二、contain——限制渲染范围

`contain` 使元素及其子元素独立于整个 DOM 树，浏览器只需对部分元素重绘、重排，不必针对整个页面：

- **`layout`**：内部布局不影响外部，反之亦然
- **`paint`**：子级不能在元素范围外显示
- **`size`**：元素盒子大小独立于其内容
- **`content`**：`contain: layout paint` 的简写
- **`strict`**：`contain: layout paint size` 的简写

```css
.item {
  contain: strict;
}
```

## 三、font-display——字体加载策略

`@font-face` 加载字体期间，默认浏览器会隐藏文字（FOIT）。`font-display` 控制此行为：

```css
@font-face {
  font-family: 'Open Sans Regular';
  src: url('fonts/OpenSans-Regular.woff2') format('woff2');
  font-display: swap;
}
```

| 值 | 阻塞时间 | 交换时间 | 行为 |
|----|----------|----------|------|
| `auto` | 浏览器决定 | 无限 | 默认行为 |
| `block` | 3s | 无限 | 短时隐藏，字体加载后切换 |
| `swap` | 0 | 无限 | 立即显示后备字体，加载后切换 |
| `fallback` | 100ms | 3s | 极短隐藏，后备字体显示，超时则不再切换 |
| `optional` | 100ms | 0 | 由浏览器决定是否使用自定义字体 |

## 四、减少渲染阻止时间

通过 `media` 属性将 CSS 拆分为多个文件，让非匹配的样式表不阻塞渲染：

```html
<link rel="stylesheet" href="styles.css" media="all" />
<link rel="stylesheet" href="sm.css" media="(min-width: 20em)" />
<link rel="stylesheet" href="lg.css" media="(min-width: 64em)" />
```

浏览器只会阻塞等待当前设备匹配的样式表，不匹配的样式表仍会下载但不会阻塞渲染。

## 五、避免 @import 嵌套

`@import` 是阻塞调用，嵌套使用时性能更差。使用多个 `<link>` 替代：

```html
<!-- 替代 @import url("windows.css") -->
<link rel="stylesheet" href="styles.css">
<link rel="stylesheet" href="windows.css">
```

`link` 与 `@import` 的区别：
- `<link>` 在页面加载时同时加载；`@import` 在页面加载完后才加载
- `<link>` 权重高于 `@import`
- `<link>` 可通过 JS 控制 DOM

## 六、CSS 自定义属性

```css
:root { --color: red; }
button { color: var(--color); }
```

注意：
- 尽量在局部作用域注册，避免在根元素上影响大量子代导致样式重新计算
- 使用 `setProperty` 修改自定义属性比内联更稳定
