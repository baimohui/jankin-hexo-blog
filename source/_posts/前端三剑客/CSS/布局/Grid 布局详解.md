---
title: CSS Grid 布局详解
categories: 
- CSS
tags:
- CSS
- Grid
- 布局
- 响应式布局
---

## 一、Grid 是什么，为什么需要它

CSS Grid（网格布局）是 CSS 中最强大的二维布局系统。它可以将页面划分为行和列组成的网格，精确控制元素在网格中的位置和尺寸。<!--more-->

### 一维 vs 二维布局

Flexbox 是一维布局——只能在**行**或**列**方向上排列元素。Grid 是二维布局——可以同时控制行和列。

```text
Flexbox 一维：                 Grid 二维：
┌────┬────┬────┐              ┌────┬────┬────┐
│    │    │    │              │    │    │    │
│ 一 │ 行 │ 排 │              │    │ 二 │    │
│    │    │    │              │    │ 维 │    │
├────┼────┼────┤              ├────┼────┼────┤
│    │    │    │              │    │ 排 │    │
│ 只 │ 能 │ 操 │              │    │ 列 │    │
│    │    │    │              │    │ 兼 │    │
└────┴────┴────┘              └────┴────┴────┘
```

### Grid vs Flexbox 选型

| 场景 | 推荐方案 |
|------|----------|
| 页面整体骨架（header/sidebar/main/footer） | **Grid** |
| 卡片网格、图片画廊 | **Grid** |
| 导航栏、按钮组、表单行 | Flexbox |
| 一列居中 + 另一列自适应 | Flexbox |
| 不知元素数量，需要自动折行 | Grid（auto-fill）或 Flexbox（flex-wrap） |

## 二、基础概念

### Grid 容器（Container）

设置了 `display: grid` 的元素成为网格容器，其直接子元素自动成为网格项。

```css
.container {
  display: grid;              /* 块级网格 */
  /* 或 */
  display: inline-grid;       /* 内联网格 */
}
```

### 网格线（Grid Line）

划分网格的线，编号从 1 开始。可以给网格线命名。

### 网格轨道（Grid Track）

两条相邻网格线之间的区域——即一行或一列。

### 网格单元（Grid Cell）

四条网格线围成的最小区域。

### 网格区域（Grid Area）

任意多个网格单元组成的矩形区域。

```
  │ 列线1 │ 列线2 │ 列线3 │ 列线4
──┼───────┼───────┼───────┼────
行线1 │       │       │       │
  │       │  单元  │       │
行线2 │       │       │       │
  │       ├───────┤       │
行线3 │       │  区域  │       │
  │       │       │       │
行线4 │       │       │       │
──┼───────┴───────┴───────┴────
```

## 三、定义网格

### grid-template-columns / grid-template-rows

```css
.container {
  display: grid;
  grid-template-columns: 100px 200px 1fr;   /* 三列：固定 100px、固定 200px、剩余空间 */
  grid-template-rows: 100px auto 100px;       /* 三行 */
  gap: 16px;                                  /* 行列间距 */
}
```

### fr 单位

`fr` 代表剩余空间的分数（fraction），是 Grid 中最常用的自适应单位：

```css
/* 三列等宽 */
grid-template-columns: 1fr 1fr 1fr;

/* 两列：侧边栏固定，主内容占剩余空间 */
grid-template-columns: 240px 1fr;

/* 混合使用 */
grid-template-columns: 100px 1fr 2fr;
/*     固定100px │ 占剩余1/3 │ 占剩余2/3 */
```

### repeat() 函数

```css
/* 重复 3 次 */
grid-template-columns: repeat(3, 1fr);   /* 等价于 1fr 1fr 1fr */

/* 重复固定+自适应模式 */
grid-template-columns: repeat(4, 100px 1fr);
/* 等价于：100px 1fr 100px 1fr 100px 1fr 100px 1fr */

/* 重复填充到容器填满（自动折行） */
grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));
```

### minmax() 函数

```css
/* 列宽最小 200px，最大 1fr（等分剩余空间） */
grid-template-columns: repeat(3, minmax(200px, 1fr));

/* 侧边栏最小 200px，最大 400px；主内容占剩余 */
grid-template-columns: minmax(200px, 400px) 1fr;
```

### auto-fill vs auto-fit

```css
/* auto-fill：保留空轨道 */
grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));

/* auto-fit：折叠空轨道（空轨道宽度为 0） */
grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
```

区别在于：当元素不足时，`auto-fill` 会留下空白轨道，`auto-fit` 会将空轨道折叠，让已有元素拉伸填充。

### 网格间距 gap

```css
.container {
  gap: 16px 24px;           /* row-gap column-gap */
  row-gap: 16px;             /* 行间距 */
  column-gap: 24px;          /* 列间距 */
}
```

## 四、放置网格项

### 1. 自动放置（默认）

子元素按顺序自动填充网格，从左到右、从上到下：

```css
.container {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
}
/* 子元素自动占据下一个可用单元格 */
```

### 2. 基于网格线的放置

使用 `grid-row` 和 `grid-column` 指定起止网格线编号：

```css
.item {
  grid-column: 1 / 3;       /* 从列线 1 到列线 3（跨越 2 列） */
  grid-row: 1 / 3;          /* 从行线 1 到行线 3（跨越 2 行） */
}

/* 简写：只写起始线，span 指定跨越数量 */
.item {
  grid-column: 1 / span 2;   /* 起始列线1，跨越2列 */
  grid-row: 1 / span 2;      /* 起始行线1，跨越2行 */
}

/* 从起始线到末尾 */
.item {
  grid-column: 2 / -1;       /* 从列线2到最后一列 */
}
```

### 3. 基于网格区域的放置

使用 `grid-template-areas` 定义命名区域，子元素通过 `grid-area` 指定归属：

```css
.layout {
  display: grid;
  grid-template-columns: 200px 1fr 200px;
  grid-template-rows: auto 1fr auto;
  grid-template-areas:
    "header header header"
    "sidebar main  aside"
    "footer footer footer";
  min-height: 100vh;
}

.header  { grid-area: header; }
.sidebar { grid-area: sidebar; }
.main    { grid-area: main; }
.aside   { grid-area: aside; }
.footer  { grid-area: footer; }
```

命名区域的规则：
- 每个单元格用名称标记，空格分隔
- 同一区域必须是矩形（不能是 L 形）
- 用 `.` 表示空单元格

```css
grid-template-areas:
  "header header header"
  "sidebar .      aside"    /* 中间列跳过 */
  "footer footer footer";
```

## 五、对齐

Grid 提供两层对齐控制：容器级别和项目级别。

### 容器级别

```css
.container {
  display: grid;
  grid-template-columns: 100px 100px;
  grid-template-rows: 100px 100px;

  /* 项目在行方向的对齐（水平） */
  justify-content: center;       /* start / end / center / stretch / space-around / space-between / space-evenly */

  /* 项目在列方向的对齐（垂直） */
  align-content: center;         /* 同上 */

  /* 项目在单元格内的水平对齐 */
  justify-items: stretch;        /* start / end / center / stretch */

  /* 项目在单元格内的垂直对齐 */
  align-items: stretch;          /* start / end / center / stretch */
}
```

### 项目级别

```css
.item {
  /* 覆盖容器级的 justify-items / align-items */
  justify-self: center;          /* start / end / center / stretch */
  align-self: end;               /* start / end / center / stretch */
}
```

## 六、隐式网格

当项目放置在已定义的行列之外时，Grid 会自动生成隐式网格轨道。

```css
.container {
  grid-template-columns: repeat(3, 1fr);   /* 只定义了3列 */
  grid-template-rows: repeat(2, 100px);    /* 只定义了2行 */
}

.item:nth-child(10) {
  grid-column: 4;                          /* 第4列不存在，自动创建 */
}
```

控制隐式轨道的大小：

```css
.container {
  grid-auto-rows: 100px;                    /* 隐式行高 */
  grid-auto-columns: 1fr;                   /* 隐式列宽 */
  grid-auto-flow: row;                      /* 自动放置方向 */
  /* row（默认）：从左到右填充 → 再换行 */
  /* column：从上到下填充 → 再换列 */
  /* dense：紧凑填充（可能会打乱顺序） */
  grid-auto-flow: row dense;                /* 紧凑模式 */
}
```

## 七、响应式布局模式

### 模式一：自动折行网格

```css
.grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(280px, 1fr));
  gap: 16px;
}
```

效果：每列最小 280px，容器宽时列数增加，窄时列数减少，无需媒体查询。

### 模式二：断点切换

```css
.grid {
  display: grid;
  grid-template-columns: 1fr;
  gap: 16px;
}

@media (min-width: 640px) {
  .grid { grid-template-columns: repeat(2, 1fr); }
}

@media (min-width: 1024px) {
  .grid { grid-template-columns: repeat(3, 1fr); }
}
```

### 模式三：经典圣杯布局

```css
.layout {
  display: grid;
  grid-template-columns: 1fr;
  grid-template-rows: auto 1fr auto;
  min-height: 100vh;
}

@media (min-width: 768px) {
  .layout {
    grid-template-columns: 200px 1fr 200px;
    grid-template-rows: auto 1fr auto;
    grid-template-areas:
      "header header header"
      "sidebar main aside"
      "footer footer footer";
  }
}
```

### 模式四：Dashboard 面板

```css
.dashboard {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  gap: 16px;
}

.dashboard .full-width {
  grid-column: 1 / -1;           /* 横跨所有列 */
}

.dashboard .double {
  grid-column: span 2;           /* 跨越 2 列 */
}

.dashboard .tall {
  grid-row: span 2;              /* 跨越 2 行 */
}
```

## 八、Grid 与 Flexbox 组合使用

Grid 负责页面级**宏观布局**，Flexbox 负责组件级**微观排列**：

```css
/* Grid：页面骨架 */
.page {
  display: grid;
  grid-template-columns: 250px 1fr;
  grid-template-rows: 60px 1fr;
  grid-template-areas:
    "header header"
    "sidebar main";
}

/* Flexbox：组件内部排列 */
.toolbar {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.card-footer {
  display: flex;
  gap: 8px;
  justify-content: flex-end;
}
```

## 九、Grid 调试

Chrome DevTools 提供了强大的 Grid 调试功能：

1. 选中设置了 `display: grid` 的元素
2. DevTools 的 Elements 面板中会出现 `grid` 徽章
3. 点击徽章可以**高亮显示网格线**、**显示网格编号**、**查看网格轨道大小**

## 十、常见问题

### Q1: 老旧浏览器的兼容性

CSS Grid 在 Chrome 57+、Firefox 52+、Safari 10.1+、Edge 16+ 中得到支持。如需兼容 IE，可以使用 Autoprefixer 添加 `-ms-` 前缀，但 IE 仅支持旧版 Grid 规范，功能受限。

### Q2: gap 在 Flexbox 中能用吗

```css
/* gap 最初是 Grid 的属性，现在 Flexbox 也支持 */
.flex-container {
  display: flex;
  gap: 16px;      /* 所有现代浏览器已支持 */
}
```

### Q3: 内容超出网格怎么办

```css
.item {
  overflow: auto;      /* 滚动 */
  /* 或 */
  word-break: break-all;  /* 断词 */
}
```

## 十一、推荐学习路径

1. 理解 Grid 的二维本质和核心概念（容器、项目、网格线、轨道）
2. 掌握 `grid-template-columns` 和 `repeat()`/`minmax()`/`fr`
3. 练习网格放置：`grid-column`、`grid-row`、`grid-area`
4. 用 `grid-template-areas` 搭建页面骨架
5. 结合 `auto-fill`/`auto-fit` 和 `minmax` 做响应式布局
6. 与 Flexbox 组合使用：Grid 管宏观、Flexbox 管微观
