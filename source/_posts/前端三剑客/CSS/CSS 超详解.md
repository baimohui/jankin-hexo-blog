---
title: CSS 核心
categories: 
- CSS
tags:
- CSS
---

## （一）CSS 规则结构

选择器和声明块组成了**CSS 规则集**（CSS ruleset），常简称为 CSS 规则。<!--more-->

```css
span {
    color: red;
    text-align: center;
}
```

CSS 中的**注释**：

```css
/* 单行注释 */
/*
    多行
    注释
*/
```

在 CSS 文件中，只有注释、CSS 规则集和 @规则能够被浏览器识别。

### @规则

@规则用于在样式表中描述元数据或特殊行为。以下是每个 @规则的使用示例：

#### @charset

定义样式表字符编码，**必须是样式表的第一个元素，且不能用在 `<style>` 块中**。

```css
@charset "UTF-8";
```

#### @import

引入外部样式表。必须放在 @charset 之后、其他规则之前。

```css
@import url('theme.css');
@import url('typography.css') screen and (min-width: 768px);  /* 带媒体条件 */
@import url('print.css') print;

/* 支持 @supports 条件（CSS Conditional Rules Level 5） */
@import url('grid.css') supports(display: grid);
```

`<link>` 与 `@import` 的区别：

- `<link>` 是 HTML 标签，可加载 CSS 和其他资源；`@import` 是 CSS 语法，只能导入 CSS
- `<link>` 在页面加载时同时加载；`@import` 在页面加载完成后才加载
- `<link>` 可通过 JS 动态操作 DOM 引入

#### @media

媒体查询，满足条件时应用内部样式。

```css
@media (max-width: 768px) {
  .sidebar { display: none; }
}

@media (prefers-color-scheme: dark) {
  body { background: #1a1a2e; color: #eee; }
}

@media (hover: hover) {
  .card:hover { transform: translateY(-4px); }
}

@media (width >= 1024px) and (width <= 1440px) {
  .container { padding: 2rem 4rem; }
}
```

#### @container

容器查询，根据父容器的尺寸而非视口来应用样式。

```css
.card-wrapper {
  container-type: inline-size;
  container-name: card;
}

@container card (min-width: 400px) {
  .card { display: flex; flex-direction: row; }
  .card-title { font-size: 1.5rem; }
}

@container card (max-width: 399px) {
  .card { display: flex; flex-direction: column; }
  .card-title { font-size: 1.25rem; }
}

/* 样式查询：检测容器是否具有某些样式能力 */
@container card style(--theme: dark) {
  .card { background: #333; color: #eee; }
}
```

#### @layer

声明层叠层，控制不同来源样式的优先级顺序。越后定义的层优先级越高。

```css
/* 1. 先声明层的顺序 */
@layer reset, base, components, utilities;

/* 2. 在各层中定义样式 */
@layer reset {
  * { margin: 0; padding: 0; box-sizing: border-box; }
}

@layer base {
  body { font-family: system-ui, sans-serif; line-height: 1.6; }
}

@layer components {
  .card { padding: 1.5rem; border-radius: 8px; box-shadow: 0 2px 8px rgba(0,0,0,.1); }
}

@layer utilities {
  .mt-4 { margin-top: 1rem; }
  .text-center { text-align: center; }
}

/* 3. 未指定层的样式优先级高于所有 @layer */
.button { background: #409eff; }  /* 覆盖 components 层中的 .button */

/* 4. 匿名层 */
@layer {
  .fallback { display: block; }
}

/* 5. 嵌套层 */
@layer components {
  @layer card {
    .card { padding: 1rem; }
  }
  @layer list {
    .list { display: flex; gap: 8px; }
  }
}
```

#### @font-face

定义自定义字体，供 `font-family` 使用。

```css
@font-face {
  font-family: 'Open Sans';
  src: url('opensans.woff2') format('woff2'),
       url('opensans.woff') format('woff');
  font-weight: 400;
  font-style: normal;
  font-display: swap;                          /* 避免 FOIT */
  unicode-range: U+0000-00FF, U+0131;          /* 只加载需要的字符 */
}

@font-face {
  font-family: 'Open Sans';
  src: url('opensans-bold.woff2') format('woff2');
  font-weight: 700;
}

body { font-family: 'Open Sans', sans-serif; }
```

#### @keyframes

定义动画关键帧，配合 `animation` 使用。

```css
@keyframes slide-in {
  from {
    transform: translateX(-100%);
    opacity: 0;
  }
  to {
    transform: translateX(0);
    opacity: 1;
  }
}

@keyframes bounce {
  0%, 100% {
    transform: translateY(0);
  }
  50% {
    transform: translateY(-20px);
  }
}

@keyframes loading {
  to {
    transform: rotate(360deg);
  }
}

.element {
  animation: slide-in 0.5s ease-out;
}

.spinner {
  animation: loading 0.8s linear infinite;
}
```

#### @supports

检测浏览器是否支持某个 CSS 特性，用于渐进增强。

```css
@supports (display: grid) {
  .container {
    display: grid;
    grid-template-columns: repeat(3, 1fr);
  }
}

@supports not (display: grid) {
  .container {
    display: flex;
    flex-wrap: wrap;
  }
}

@supports (display: grid) and (gap: 16px) {
  .container { gap: 16px; }
}

@supports (selector(:has(div))) {
  .card:has(.featured) { border-color: gold; }
}
```

#### @page

控制打印输出时的页面样式。

```css
@page {
  size: A4;
  margin: 2cm;
}

@page :first {
  margin-top: 4cm;
}

@page :left {
  margin-right: 3cm;
}

@page :right {
  margin-left: 3cm;
}

@media print {
  .nav, .sidebar { display: none; }
  body { font-size: 12pt; }
}
```

#### @property

注册类型化的 CSS 自定义属性（CSS Houdini 的一部分），让自定义属性具有类型约束、继承行为和初始值，并支持 transition 动画。

```css
@property --rotation {
  syntax: '<angle>';
  inherits: false;
  initial-value: 0deg;
}

@property --primary-color {
  syntax: '<color>';
  inherits: true;
  initial-value: #409eff;
}

@property --spacing {
  syntax: '<length>';
  inherits: true;
  initial-value: 0px;
}

.element {
  --rotation: 0deg;
  transform: rotate(var(--rotation));
  transition: --rotation 1s;
}

.element:hover {
  --rotation: 180deg;  /* 注册后的自定义属性支持 transition */
}
```

#### @scope

限定样式的作用范围（CSS Cascading and Inheritance Level 6）。

```css
/* 限定 .card 内部的样式，不影响嵌套更深的 .card */
@scope (.card) {
  :scope { padding: 1rem; border: 1px solid #ddd; }
  p { margin: 0.5rem 0; }
  a { color: var(--primary); }
}

/* 带作用范围限制 */
@scope (.article) to (.footer) {
  p { line-height: 1.8; }
  /* 只影响 .article 内部到 .footer 之前的 p */
}
```

## （二）选择器

### 基础选择器

| 选择器 | 示例 | 说明 |
|--------|------|------|
| 标签选择器 | `h1` | 匹配所有 h1 元素 |
| 类选择器 | `.checked` | 匹配 class 中包含 checked 的元素 |
| ID 选择器 | `#picker` | 匹配 id 为 picker 的元素 |
| 通配选择器 | `*` | 匹配所有元素 |
| 属性选择器 | `[type="text"]` | 匹配 type 为 text 的元素 |

### 属性选择器

```css
[attr]        /* 存在 attr 属性的元素 */
[attr=val]    /* attr 等于 val */
[attr*=val]   /* attr 包含 val */
[attr^=val]   /* attr 以 val 开头 */
[attr$=val]   /* attr 以 val 结尾 */
[attr~=val]   /* attr 包含完整单词 val */
[attr|=val]   /* attr 以 val- 开头 */
```

### 组合选择器

```css
A B      /* 后代选择器：A 内部的所有 B */
A > B    /* 子选择器：A 的直接子元素 B */
A + B    /* 相邻兄弟：紧接 A 后的第一个 B */
A ~ B    /* 普通兄弟：A 后的所有 B */
```

### 伪类

**条件伪类**：

```css
:not(selector)   /* 不匹配 selector 的元素 */
:is(selector)    /* 匹配 selector 列表中的任一（优先级取最高） */
:where(selector) /* 同 :is，但优先级始终为 0 */
:has(selector)   /* 包含 selector 子元素的父元素 */
```

`:is()` 和 `:where()` 的区别在于优先级处理——`:is()` 取参数中最高的优先级，`:where()` 优先级始终为 0。

```css
/* 用 :is 简化选择器 */
:is(header, main, footer) :is(h1, h2) { color: red; }
/* 等价于：header h1, header h2, main h1, main h2, footer h1, footer h2 */

/* :has 父级选择器（实用） */
.card:has(.featured) { border-color: gold; }        /* 包含 .featured 的卡片 */
form:has(:invalid) .submit-btn { opacity: 0.5; }    /* 表单有非法输入时禁用按钮 */
```

**行为伪类**：`:active`、`:hover`、`:focus`、`:focus-within`、`:target`

**表单伪类**：

```css
:checked     /* 选中的选项 */
:disabled    /* 禁用的元素 */
:required    /* 必填字段 */
:valid       /* 合法输入 */
:invalid     /* 非法输入 */
:in-range    /* 范围内的值 */
:out-of-range/* 范围外的值 */
:placeholder-shown /* 显示占位符时 */
:user-invalid/* 用户交互后的非法输入 */
```

**结构伪类**：

```css
:root               /* 根元素（html） */
:empty              /* 无子元素的元素 */
:nth-child(n)       /* 第 n 个子元素 */
:nth-last-child(n)  /* 倒数第 n 个 */
:first-child        /* 第一个子元素 */
:last-child         /* 最后一个子元素 */
:only-child         /* 唯一的子元素 */
:nth-of-type(n)     /* 第 n 个相同标签 */
:first-of-type      /* 第一个相同标签 */
:last-of-type       /* 最后一个相同标签 */
```

`:nth-child` 接受公式 `an + b`：

```css
li:nth-child(2n)     /* 偶数行 2, 4, 6... */
li:nth-child(2n+1)   /* 奇数行 1, 3, 5... */
li:nth-child(3n+1)   /* 1, 4, 7... */
li:nth-child(-n+3)   /* 前三行 */
```

### 伪元素

```css
::before         /* 在元素前插入内容（需 content） */
::after          /* 在元素后插入内容（需 content） */
::first-letter   /* 首字母 */
::first-line     /* 首行 */
::selection      /* 选中文本 */
::placeholder    /* 输入框占位符 */
::marker         /* 列表标记 */
::backdrop       /* 全屏/弹窗背景 */
```

### 选择器优先级

从高到低：

```text
10000  →  !important
01000  →  内联样式（style 属性）
00100  →  ID 选择器
00010  →  类、伪类、属性选择器
00001  →  元素、伪元素选择器
00000  →  通配 *、组合器、:where()
```

注意 `:not()`、`:is()`、`:where()` 的优先级差异——`:is()` 取参数中最高优先级，`:where()` 优先级固定为 0。

`!important` 应谨慎使用：
- 优先用优先级规则解决问题而非 `!important`
- 只在覆盖全局或第三方样式时使用
- 不要在全站 CSS 或插件中使用

`@layer` 可以控制不同来源的样式优先级，比 `!important` 更可控。

## （三）层叠与继承

### 层叠规则

当多个规则应用到同一元素时，按以下顺序决定最终值：

```text
1. 来源与重要性（从低到高）
   用户代理样式（浏览器默认）
   用户样式（用户自定义）
   作者样式（开发者）
   !important（顺序反转）
2. 优先级（选择器权重）
3. 出现顺序（后定义的覆盖先定义的）
```

### 继承

部分 CSS 属性会从父元素继承：

```css
/* 默认继承：font-size、color、font-family、line-height、text-align */
p { color: red; }
span { }  /* span 继承 p 的 color: red */

/* 默认不继承：width、height、margin、padding、border、background */
```

控制继承的关键字：

```css
.element {
  color: inherit;        /* 强制继承父元素 */
  color: initial;        /* 使用浏览器默认值 */
  color: unset;          /* 如果属性可继承则 inherit，否则 initial */
  color: revert;         /* 恢复到用户代理样式 */
}
```

### @layer

`@layer` 允许将样式分组到不同的"层"，后定义的层优先级更高：

```css
@layer reset, base, components, utilities;

@layer reset {
  * { margin: 0; padding: 0; }
}

@layer components {
  .card { padding: 1rem; border-radius: 8px; }
}

@layer utilities {
  .mt-4 { margin-top: 1rem; }
}
```

未被 `@layer` 包裹的样式优先级高于所有层。

## （四）盒模型

每个元素都可以看作一个盒子，由 4 部分组成：

```text
┌─────────────────────────────────┐
│          margin（外边距）        │
│  ┌───────────────────────────┐  │
│  │   border（边框）           │  │
│  │  ┌─────────────────────┐  │  │
│  │  │  padding（内边距）   │  │  │
│  │  │ ┌─────────────────┐ │  │  │
│  │  │ │    content       │ │  │  │
│  │  │ │    内容区域       │ │  │  │
│  │  │ └─────────────────┘ │  │  │
│  │  └─────────────────────┘  │  │
│  └───────────────────────────┘  │
└─────────────────────────────────┘
```

### 标准盒模型 vs IE 盒模型

```css
/* content-box（默认）：width = 内容宽度 */
.box { box-sizing: content-box; width: 200px; padding: 10px; border: 1px solid; }
/* 实际占用宽度 = 200 + 10 + 10 + 1 + 1 = 222px */

/* border-box：width = 内容 + padding + border */
.box { box-sizing: border-box; width: 200px; padding: 10px; border: 1px solid; }
/* 内容宽度 = 200 - 10 - 10 - 1 - 1 = 178px */
```

**推荐全局设置 `border-box`**：

```css
*, *::before, *::after {
  box-sizing: border-box;
}
```

### 外边距折叠

垂直方向相邻的外边距会合并为较大的那个：

```css
.top { margin-bottom: 30px; }
.bottom { margin-top: 20px; }
/* 两者之间的间距为 30px，而非 50px */
```

不会折叠的情况：`display: flex` / `grid` 的子元素、`overflow: hidden` 的容器、浮动元素、绝对定位元素。

## （五）display 与视觉格式化

| 值 | 行为 | 使用场景 |
|----|------|----------|
| `block` | 独占一行，可设宽高 | div、p、h1-h6 |
| `inline` | 同行排列，宽高由内容决定 | span、a、strong |
| `inline-block` | 同行排列，可设宽高 | 按钮、导航项 |
| `none` | 隐藏，不占空间 | 条件显示 |
| `flex` | 弹性盒子 | 一维布局 |
| `grid` | 网格布局 | 二维布局 |
| `table` | 表格布局 | 兼容旧场景 |

### 行内元素的空白间隙

```html
<!-- 换行导致间隙 -->
<span>A</span>
<span>B</span>
```

解决方案：

```css
/* 1. 父元素设置 font-size: 0，再为子元素单独设置 */
.parent { font-size: 0; }
.parent span { font-size: 16px; }

/* 2. 使用 flexbox */
.parent { display: flex; }
```

## （六）定位

| 值 | 参考系 | 特点 |
|----|--------|------|
| `static` | 无（默认） | 正常文档流 |
| `relative` | 自身原始位置 | 不脱离文档流 |
| `absolute` | 最近的定位祖先 | 脱离文档流 |
| `fixed` | 视口 | 脱离文档流，固定不动 |
| `sticky` | 最近滚动容器 + 视口 | 到达阈值前 relative，之后 fixed |

```css
.sticky-header {
  position: sticky;
  top: 0;           /* 滚动到顶部时固定 */
  background: white;
  z-index: 100;
}
```

`sticky` 生效条件：必须指定 `top`/`bottom`/`left`/`right` 至少一个值，且父容器不能 `overflow: hidden`。

### z-index 与层叠上下文

层叠上下文是一个独立的三维空间，内部元素的 z-index 只在该上下文中比较。

触发层叠上下文的条件：

```text
├── position: relative/absolute/fixed + z-index 非 auto
├── display: flex/grid + z-index 非 auto
├── opacity < 1
├── transform != none
├── filter != none
├── will-change
├── contain: paint
└── isolation: isolate（主动隔离）
```

`isolation: isolate` 可以手动创建层叠上下文，避免被外部元素影响。

## （七）布局

### Flexbox

```css
.container {
  display: flex;
  flex-direction: row;         /* row / column / row-reverse / column-reverse */
  flex-wrap: wrap;             /* nowrap / wrap / wrap-reverse */
  justify-content: center;     /* 主轴对齐 */
  align-items: center;         /* 交叉轴对齐 */
  align-content: center;       /* 多行对齐 */
  gap: 16px;                   /* 间距 */
}

.item {
  flex: 1;                     /* 简写：flex-grow flex-shrink flex-basis */
  align-self: center;          /* 单个项目对齐 */
  order: 1;                    /* 排序 */
}
```

`flex` 简写详解：

```css
flex: 1;               /* flex-grow: 1, flex-shrink: 1, flex-basis: 0% */
flex: auto;            /* flex-grow: 1, flex-shrink: 1, flex-basis: auto */
flex: none;            /* flex-grow: 0, flex-shrink: 0, flex-basis: auto */
```

### Grid

```css
.container {
  display: grid;
  grid-template-columns: repeat(3, 1fr);  /* 三列等宽 */
  grid-template-rows: auto 1fr auto;        /* 三行 */
  gap: 16px;
}

.item:nth-child(1) {
  grid-column: 1 / -1;                     /* 跨所有列 */
  grid-row: span 2;                        /* 跨 2 行 */
}

/* 命名区域 */
.layout {
  grid-template-areas:
    "header header header"
    "sidebar main main"
    "footer footer footer";
}
```

Grid 与 Flexbox 选型：

| 场景 | 推荐 |
|------|------|
| 一维排列（导航、按钮组） | Flexbox |
| 二维布局（页面骨架、卡片网格） | Grid |
| 内容宽度不确定，需要自动折行 | Grid（auto-fill）+ Flexbox（wrap） |

### 浮动

```css
.float-left {
  float: left;   /* left / right / none */
}

.clearfix::after {
  content: '';
  display: block;
  clear: both;
}
```

浮动最初用于文字环绕，后被用于布局。现在布局场景已被 Flexbox 和 Grid 替代，浮动仅用于**文字环绕**的场合。

### 多列布局

```css
.text-columns {
  column-count: 3;           /* 列数 */
  column-gap: 2rem;          /* 列间距 */
  column-rule: 1px solid #ddd;/* 列分隔线 */
  column-width: 300px;       /* 列最小宽度（与 count 互斥） */
}
```

## （八）尺寸与单位

### CSS 单位

**绝对单位**：

| 单位 | 说明 |
|------|------|
| `px` | 像素（1px = 1/96英寸） |

**相对单位**：

| 单位 | 相对于 |
|------|--------|
| `%` | 父元素对应属性 |
| `em` | 当前元素的 `font-size` |
| `rem` | 根元素 `font-size`（16px） |
| `vw` | 视口宽度 1% |
| `vh` | 视口高度 1% |
| `vmin` | `vw` 和 `vh` 中较小值 |
| `vmax` | `vw` 和 `vh` 中较大值 |
| `lvh` / `svh` / `dvh` | 大/小/动态视口高度（解决移动端地址栏问题） |
| `cqw` / `cqh` | 容器查询单位 |
| `ch` | 数字 0 的宽度 |
| `ex` | 字母 x 的高度 |

### CSS 函数

```css
/* calc：混合单位计算 */
width: calc(100% - 40px);
height: calc(100vh - 60px);
grid-template-columns: calc(100% - 300px) 300px;

/* min / max：取最小/最大值 */
width: min(100%, 1200px);         /* 不超过 1200px */
font-size: max(16px, 2vw);        /* 不小于 16px */

/* clamp：限制范围 */
font-size: clamp(16px, 2vw, 32px);/* 16px ~ 32px 之间按 2vw 缩放 */

/* var：使用 CSS 变量 */
color: var(--primary, #409eff);   /* 第二个参数为默认值 */
```

## （九）响应式设计

### 媒体查询

```css
/* 断点示例 */
@media (max-width: 640px)  { /* 手机 */ }
@media (min-width: 768px)  { /* 平板竖屏 */ }
@media (min-width: 1024px) { /* 桌面 */ }
@media (min-width: 1280px) { /* 大屏 */ }

/* 逻辑操作符 */
@media (min-width: 768px) and (orientation: portrait) { }
@media (prefers-color-scheme: dark) { }              /* 暗黑模式 */
@media (prefers-reduced-motion: reduce) { }           /* 减少动画 */
@media (hover: hover) { }                             /* 支持悬停 */
@media (pointer: coarse) { }                          /* 触摸设备 */
```

### 容器查询

容器查询允许子元素根据**父容器**的大小响应，而非视口：

```css
.card-container {
  container-type: inline-size;
  container-name: card;
}

@container card (min-width: 400px) {
  .card { display: flex; }
}

@container card (max-width: 399px) {
  .card { display: block; }
}
```

容器查询让组件级别的响应式成为可能，同一个组件在不同尺寸的容器中自动适配。

## （十）视觉效果

### 过渡

```css
.element {
  transition: property duration timing-function delay;
  /* 简写示例 */
  transition: all 0.3s ease;
  transition: transform 0.2s, opacity 0.3s ease-in 0.1s;
}

/* 缓动函数 */
linear        /* 匀速 */
ease          /* 慢快慢 */
ease-in       /* 慢到快 */
ease-out      /* 快到慢 */
ease-in-out   /* 慢快慢 */
cubic-bezier(0.68, -0.55, 0.27, 1.55) /* 弹跳效果 */
steps(3)      /* 步进（逐帧动画） */
```

### 变换

```css
.element {
  /* 2D */
  transform: translate(10px, 20px); /* 位移 */
  transform: rotate(45deg);         /* 旋转 */
  transform: scale(1.5);            /* 缩放 */
  transform: skew(10deg);           /* 倾斜 */

  /* 3D */
  transform: perspective(500px) translateZ(100px);
  transform: rotateX(45deg) rotateY(30deg);

  /* 变换基点 */
  transform-origin: center;
  transform-origin: top left;
}
```

多个变换按**从右到左**执行：

```css
transform: translateX(100px) rotate(45deg);
/* 先旋转 45°，再平移 100px（沿自身旋转后的方向） */
```

### 动画

```css
@keyframes slide-in {
  from {
    transform: translateX(-100%);
    opacity: 0;
  }
  to {
    transform: translateX(0);
    opacity: 1;
  }
}

@keyframes bounce {
  0%, 100% { transform: translateY(0); }
  50% { transform: translateY(-20px); }
}

.element {
  animation: slide-in 0.5s ease-out;
  animation: bounce 1s ease-in-out infinite;
  animation-delay: 0.2s;
  animation-fill-mode: backwards; /* 动画开始前应用 from 样式 */
}
```

动画性能原则：只对 `transform` 和 `opacity` 做动画（仅触发 Composite，不触发 Layout/Paint）。

### 渐变

```css
/* 线性渐变 */
background: linear-gradient(to right, red, blue);
background: linear-gradient(45deg, #667eea, #764ba2);

/* 径向渐变 */
background: radial-gradient(circle at center, #ff0, #f00);

/* 锥形渐变 */
background: conic-gradient(from 0deg, red, yellow, green, blue, red);

/* 重复渐变 */
background: repeating-linear-gradient(
  45deg,
  #f0f0f0 0px,
  #f0f0f0 10px,
  #ddd 10px,
  #ddd 20px
);
```

### 阴影

```css
.box-shadow: 2px 2px 10px rgba(0,0,0,.2);  /* x偏移 y偏移 模糊半径 颜色 */
.box-shadow: inset 0 0 10px #000;           /* 内阴影 */
.box-shadow: 0 0 0 3px #ff0;               /* 模拟边框（不占布局） */
.box-shadow: 2px 2px 5px #000, -2px -2px 5px #ccc;  /* 多重阴影 */

text-shadow: 1px 1px 2px rgba(0,0,0,.5);   /* 文字阴影 */
```

## （十一）CSS 自定义属性（变量）

```css
:root {
  --primary: #409eff;
  --success: #67c23a;
  --font-base: 16px;
  --spacing: 1rem;
}

.button {
  background: var(--primary);
  font-size: var(--font-base, 14px);  /* 第二个参数为默认值 */
  padding: var(--spacing);
}

/* JS 修改 */
element.style.setProperty('--primary', '#f56c6c');
```

### @property 注册类型化自定义属性

```css
@property --rotation {
  syntax: '<angle>';
  inherits: false;
  initial-value: 0deg;
}

.element {
  transform: rotate(var(--rotation));
  transition: --rotation 1s;
}

.element:hover {
  --rotation: 180deg;
}
```

## （十二）BFC

**块级格式化上下文**（Block Formatting Context）是一个独立的渲染区域，内部元素的布局不会影响外部。

BFC 的触发条件：

```text
├── float 不为 none
├── overflow 不为 visible（常用 overflow: hidden）
├── display: flow-root（推荐，无副作用）
├── display: flex / grid / inline-block
├── position: absolute / fixed
└── contain: layout / paint
```

BFC 的应用：

```css
/* 1. 清除浮动 */
.clearfix { display: flow-root; }

/* 2. 防止外边距折叠 */
.container { display: flow-root; }

/* 3. 自适应两栏布局 */
.aside { float: left; width: 200px; }
.main { display: flow-root; }  /* 自动占满剩余宽度 */
```

推荐使用 `display: flow-root` 触发 BFC，它没有副作用。

## （十三）常见需求

### 水平居中

```css
/* 行内元素 */
.parent { text-align: center; }

/* 块级元素 */
.block { margin: 0 auto; width: fit-content; }

/* Flexbox */
.parent { display: flex; justify-content: center; }
```

### 垂直居中

```css
/* 单行文本 */
.parent { height: 100px; line-height: 100px; }

/* Flexbox（推荐） */
.parent { display: flex; align-items: center; }

/* Grid */
.parent { display: grid; align-items: center; }

/* 绝对定位 + transform */
.parent { position: relative; }
.child { position: absolute; top: 50%; transform: translateY(-50%); }
```

### 全屏居中

```css
.parent {
  display: grid;
  place-items: center;   /* 同时水平和垂直居中 */
  min-height: 100vh;
}
```

### 文字溢出省略

```css
/* 单行 */
.ellipsis {
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

/* 多行（第 n 行省略） */
.line-clamp-2 {
  display: -webkit-box;
  -webkit-line-clamp: 2;
  -webkit-box-orient: vertical;
  overflow: hidden;
}

/* 或使用 line-clamp 标准属性 */
.line-clamp-2 {
  line-clamp: 2;
}
```
