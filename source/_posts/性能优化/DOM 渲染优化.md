---
title: DOM 渲染优化
categories: 
- 性能优化
tags:
- 性能优化
- DOM
- 重排
- 重绘
- 事件优化
---

## 一、避免频繁读写 DOM

### 缓存 DOM 引用

```js
var dom = document.querySelector('element');
// 后续操作使用缓存的 dom 变量，不要重复查询
```

### 缓存读取的属性

```js
// ❌ 每次循环都读取 dom.style.left（触发回流）
setInterval(() => {
  dom.style.left = parseInt(dom.style.left) + 1 + 'px';
}, 16);

// ✅ 缓存变量，避免反复读取 DOM
let left = parseInt(dom.style.left);
setInterval(() => {
  left++;
  dom.style.left = left + 'px';
}, 16);
```

### 合并多次写操作

```js
// ❌ 每次循环都修改 innerHTML
for (let i = 0; i < 100; i++) {
  document.getElementById('text').innerHTML += 'text' + i;
}

// ✅ 合并为一次写操作
let html = '';
for (let i = 0; i < 100; i++) {
  html += 'text' + i;
}
document.getElementById('text').innerHTML = html;
```

### 使用 DocumentFragment

```js
const fragment = document.createDocumentFragment();
for (let i = 0; i < 100; i++) {
  const div = document.createElement('div');
  div.innerHTML = i;
  fragment.appendChild(div);
}
document.body.appendChild(fragment);
```

## 二、CSS 避免重排重绘

### 基本原则

- 使用 `className` 替代多次 `style` 操作
- 使用 `visibility:hidden` 替代 `display:none`（前者只触发重绘，后者触发重排）
- 避免使用 `table` 布局
- 对动画元素使用 `absolute` / `fixed`
- 对 resize 事件做防抖节流处理

### 开启 GPU 渲染动画

以下属性可以避免主线程频繁操作：

- `transform`：不触发 Layout，只触发 Composite
- `opacity`：不触发 Layout，只触发 Composite
- `will-change`：提前告知浏览器优化

#### transform

```css
/* ❌ 触发 Layout */
div { margin-left: 20px; transition: margin-left 1s; }

/* ✅ 仅触发 Composite */
div { transform: translateX(20px); transition: transform 1s; }
```

#### opacity 优化阴影动画

通过伪元素配合 `opacity` 过渡替代 `box-shadow` 过渡：

```css
div::before {
  content: '';
  position: absolute;
  box-shadow: 0 5px 15px rgba(0, 0, 0, 0.3);
  opacity: 0;
  transition: opacity 1s;
}
div:hover::before { opacity: 1; }
```

#### will-change

```css
.animate {
  will-change: transform, opacity;
}
```

使用原则：
- 不要应用到太多元素上
- 在变化前通过 JS 添加，变化完成后移除
- 给浏览器足够的时间做优化准备

## 三、JS 避免重排重绘

### 引起回流的属性

读取以下属性会强制触发回流：

```
offsetTop / offsetLeft / offsetWidth / offsetHeight
scrollTop / scrollLeft / scrollWidth / scrollHeight
clientTop / clientLeft / clientWidth / clientHeight
getBoundingClientRect() / getClientRects()
getComputedStyle()
```

### 读写分离

```js
// ❌ 每次修改都触发强制回流
dom.style.width = '100px';
console.log(dom.offsetWidth);
dom.style.height = '200px';
console.log(dom.offsetHeight);

// ✅ 集中读取，集中写入
const width = el.offsetWidth + 10;
const height = el.offsetHeight + 10;
el.style.width = width + 'px';
el.style.height = height + 'px';
```

### 使用 display:none 离线操作

对于需要多次操作的 DOM 元素，先设置为 `display:none`，操作完毕再显示：

```js
el.style.display = 'none';
// 进行大量 DOM 操作...
el.style.display = 'block';
```

### 使用 cloneNode

```js
const clone = el.cloneNode(true);
// 操作 clone...
el.parentNode.replaceChild(clone, el);
```

## 四、简化 DOM 结构

DOM 结构越深，内部元素变化引发的祖先重排越多。对于大量数据表格的展示，使用分页而非一次性渲染数万个 DOM。

## 五、DOM 事件优化

### 事件委托

```js
// 在父容器绑定一个事件，通过 target 分发
container.addEventListener('click', (e) => {
  const item = e.target.closest('.item');
  if (item) handleClick(item.dataset.id);
});
```

### 节流

```js
function throttle(func, wait) {
  let previous = 0;
  return function () {
    const now = +new Date();
    if (now - previous > wait) {
      func.apply(this, arguments);
      previous = now;
    }
  };
}
// 适用场景：scroll、resize
```

### 防抖

```js
function debounce(func, wait) {
  let timeout;
  return function () {
    clearTimeout(timeout);
    timeout = setTimeout(() => func.apply(this, arguments), wait);
  };
}
// 适用场景：输入搜索、窗口 resize 结束
```
