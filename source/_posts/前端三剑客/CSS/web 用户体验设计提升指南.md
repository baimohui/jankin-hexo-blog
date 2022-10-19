---
title: Web 用户体验设计提升指南
categories: 
- 性能优化
tags:
- 性能优化
---



用户体验设计秉承着**以用户为中心的思想**，以用户需求为目标。设计过程注重以用户为中心，用户体验的概念从开发的最早期就开始进入整个流程，并贯穿始终。良好的用户体验设计，是产品每一个环节共同努力的结果。

除去一些很难一蹴而就的，本文将就**页面展示**、**交互细节**、**可访问性**三个方面入手，罗列一些在实际的开发过程中，积攒的一些有益的经验。通过本文，你将能收获到：

- 了解到一些小细节是如何影响用户体验的

- 了解到如何在尽量小的开发改动下，提升页面的用户体验

- 了解到一些优秀的交互设计细节

- 了解基本的无障碍功能及页面可访问性的含义

- 了解基本的提升页面可访问性的方法

  <!-- more -->

## （一）页面展示

### 1. 整体布局

对于大部分 PC 端的项目，首先需要考虑最外层的包裹。假设为 `.g-app-wrapper`。

```HTML
<div class="g-app-wrapper">
    <!-- 内部内容 -->
</div>
```

首先，对于 `.g-app-wrapper`，有两点是需要在项目开发前弄清楚的：

1. 项目是全屏布局还是定宽布局？
2. 对于全屏布局，需要适配的最小的宽度是多少？

对于定宽布局，就比较方便了，假设定宽为 `1200px`，那么：

```CSS
.g-app-wrapper {
    width: 1200px;
    margin: 0 auto;
}
```

利用 `margin: 0 auto` 实现布局的水平居中。在屏幕宽度大于 `1200px` 时，两侧留白，当然屏幕宽度小于 `1200px` 时，则出现滚动条，保证内部内容不乱。

![img](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3ca2418b6ef046d7b2635a2e10a4610c~tplv-k3u1fbpfcp-watermark.image)

对于现代布局，更多的是全屏布局。其实现在也更提倡这种布局，即使用可随用户设备的尺寸和能力而变化的自适应布局。

通常而言是左右两栏，左侧定宽，右侧自适应剩余宽度，当然，会有一个最小的宽度。那么，它的布局应该是这样：

```HTML
<div class="g-app-wrapper">
    <div class="g-sidebar"></div>
    <div class="g-main"></div>
</div>
```

```css
.g-app-wrapper {
    display: flex;
    min-width: 1200px;
}
.g-sidebar {
    flex-basis: 250px;
    margin-right: 10px;
}
.g-main {
    flex-grow: 1;
}
```

![img](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/865f09c168c04c808104b8540277a6f7~tplv-k3u1fbpfcp-watermark.image)

利用了 flex 布局下的 `flex-grow: 1`，让 `.main` 进行伸缩，占满剩余空间，利用 `min-width` 保证了整个容器的最小宽度。这是最基本的自适应布局。对于现代布局，我们应尽可能考虑更多的场景。做到：

<img src="https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4a1120cd519248cb8f706c63d28176be~tplv-k3u1fbpfcp-watermark.image" alt="img" style="zoom: 50%;" />

### 2. 底部 footer

页面存在一个 footer 页脚部分，如果整个页面的内容高度小于视窗的高度，则 footer 固定在视窗底部，如果整个页面的内容高度大于视窗的高度，则 footer 正常流排布（也就是需要滚动到底部才能看到 footer）。

![img](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4fe995f026044f63af807b601378294f~tplv-k3u1fbpfcp-watermark.image)

使用 flex 的`justify-content: space-between` 可以很好解决，同理使用 `margin-top: auto` 也非常容易完成：

```HTML
<div class="g-container">
    <div class="g-real-box">
        ...
    </div>
    <div class="g-footer"></div>
</div>
```

```css
.g-container {
    height: 100vh;
    display: flex;
    flex-direction: column;
}

.g-footer {
    margin-top: auto;
    flex-shrink: 0;
    height: 30px;
    background: deeppink;
}
```

### 3. 超长文本处理

对于所有接收后端接口字段的文本展示类的界面。都需要考虑全面（防御性编程：所有的外部数据都是不可信的），正常情况如下，是没有问题的。

<img src="https://cdn.jsdelivr.net/gh/baimohui/FigureBed/img/20220305164557.png" alt="image-20210328135439922" style="zoom:67%;" />

但如果文本是很长的一段内容，则可能发生如下情况：

<img src="https://cdn.jsdelivr.net/gh/baimohui/FigureBed/img/20220305164558.png" alt="image-20210328135506647" style="zoom:67%;" />

对于**单行文本**，使用单行省略：

```CSS
{
    width: 200px;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
}
```

<img src="https://cdn.jsdelivr.net/gh/baimohui/FigureBed/img/20220305164559.png" alt="image-20210328135637086" style="zoom:67%;" />

对于**多行文本**的超长省略，兼容性较好的做法如下：

```CSS
{
    width: 200px;
    overflow : hidden;
    text-overflow: ellipsis;
    display: -webkit-box;
    -webkit-line-clamp: 2;
    -webkit-box-orient: vertical;
}
```

<img src="https://cdn.jsdelivr.net/gh/baimohui/FigureBed/img/20220305164600.png" alt="image-20210328135810011" style="zoom:67%;" />

### 4. 保护边界

对于一些动态内容，我们经常使用 `min/max-width` 或 `min/max-height` 对容器的高宽限度进行合理的控制。在使用它们的时候，也有一些细节需要考虑到。譬如经常会使用 `min-width` 控制按钮的最小宽度：

```CSS
.btn {
    min-width: 120px;
}
```

<img src="https://cdn.jsdelivr.net/gh/baimohui/FigureBed/img/20220305164601.png" alt="image-20210328140034364" style="zoom:67%;" />

当内容较少时是没问题的，但当内容比较长，就容易出现按钮过长的情况：

<img src="https://cdn.jsdelivr.net/gh/baimohui/FigureBed/img/20220305164602.png" alt="image-20210328140143826" style="zoom:67%;" />

这里就需要配合 padding 一起：

```CSS
.btn {
    min-width: 88px;
    padding: 0 16px
}
```

借用[Min and Max Width/Height in CSS](https://ishadeed.com/article/min-max-css/)中一张非常好的图，作为释义：

<img src="https://cdn.jsdelivr.net/gh/baimohui/FigureBed/img/20220305164603.png" alt="image-20210328140301515" style="zoom:80%;" />

### 3. 图片相关

有的时候和产品、设计会商定，只能使用固定尺寸大小的图片，我们的布局可能是这样：

<img src="https://cdn.jsdelivr.net/gh/baimohui/FigureBed/img/20220305164604.png" alt="image-20210328141631582" style="zoom:67%;" />

对应布局：

```HTML
<ul class="g-container">
    <li>
        <img src="http://placehold.it/150x100">
        <p>图片描述</p>
    </li>
</ul>
```

```css
ul li img {
    width: 150px;
}
```

假设后端接口出现一张非正常大小的图片，上述不加保护的布局就会出问题：

<img src="https://cdn.jsdelivr.net/gh/baimohui/FigureBed/img/20220305164605.png" alt="image-20210328141756916" style="zoom:67%;" />

我们总是建议**对图片同时设置高和宽**，避免因为图片尺寸错误带来的布局问题。同时，给 `<img>` 标签同时写上高宽，可以在图片未加载之前提前占住位置，避免图片从未加载状态到渲染完成状态高宽变化引起的重排问题。

但限制高宽也可能出现图片被拉伸变形的问题，导致效果不美观。

<img src="https://cdn.jsdelivr.net/gh/baimohui/FigureBed/img/20220305164606.png" alt="image-20210328142107080" style="zoom:67%;" />

这个时候可以**借助 `object-fit`**，它能够指定可替换元素的内容（也就是图片）该如何适应它的父容器的高宽。利用 `object-fit: cover`，使图片内容在保持其宽高比的同时填充元素的整个内容框。

```css
ul li img {
    width: 150px;
    height: 100px;
    object-fit: cover;
}
```

<img src="https://cdn.jsdelivr.net/gh/baimohui/FigureBed/img/20220305164607.png" alt="image-20210328142248442" style="zoom:67%;" />

`object-fit` 还有一个配套属性 `object-position`，它可以控制图片在其内容框中的位置。（类似于 `background-position`），m默认是 `object-position: 50% 50%`，如果你不希望以图片的居中部分展示，可以使用它去改变图片实际展示的 position 。

像是这样，`object-position: 100% 50%` 指明从底部开始展示图片。

```CSS
ul li img {
    width: 150px;
    height: 100px;
    object-fit: cover;
    object-position: 50% 100%;
}
```

<img src="https://cdn.jsdelivr.net/gh/baimohui/FigureBed/img/20220305164608.png" alt="image-20210328142533433" style="zoom:67%;" />

在移动端或者一些高清的 PC 屏幕（苹果的 MAC Book），屏幕的 dpr 可能大于 1。这种时候，我们可能还需要考虑利用多倍图去适配不同 dpr 的屏幕。

```css
<img 
        src = "photo.png" 
        sizes = “(min-width: 600px) 600px, 300px" 
        srcset = “photo@1x.png 300w,
                       photo@2x.png 600w,
                       photo@3x.png 1200w,
>
```

接下来还需要考虑，当图片链接挂了，应该如何处理。处理的方式有很多种。最好的处理方式，是我最近在张鑫旭老师的这篇文章中 -- [图片加载失败后CSS样式处理最佳实践](https://www.zhangxinxu.com/wordpress/2020/10/css-style-image-load-fail/) 看到的。这里简单讲下：

1. 利用图片加载失败，触发 `<img>` 元素的 `onerror` 事件，给加载失败的 `<img>` 元素新增一个样式类
2. 利用新增的样式类，配合 `<img>` 元素的伪元素，展示默认兜底图的同时，还能一起展示 `<img>` 元素的 `alt` 信息

```HTML
<img src="test.png" alt="图片描述" onerror="this.classList.add('error');">
```

```css
img.error {
    position: relative;
    display: inline-block;
}

img.error::before {
    content: "";
    /** 定位代码 **/
    background: url(error-default.png);
}

img.error::after {
    content: attr(alt);
    /** 定位代码 **/
}
```

我们利用伪元素 `before` ，加载默认错误兜底图，利用伪元素 `after`，展示图片的 `alt` 信息：

<img src="https://cdn.jsdelivr.net/gh/baimohui/FigureBed/img/20220305164609.png" alt="image-20210328143432662" style="zoom:67%;" />









