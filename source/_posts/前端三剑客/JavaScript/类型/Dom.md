---
title: JS 操作 dom 元素
categories: 
- JavaScript
tags:
- JavaScript
---

#### 0. 获取 dom 节点

```js
document.getElementById('id')
// 通过 ID 获取，上下文只能是 document，返回值只获取到一个元素，没有找到返回 null

document.getElementsByName('name')
// 通过 name 属性获取，上下文只能是 document，返回值是一个类数组，没有找到返回空数组

document.getElementByTagName('p')
var oDiv = document.getElementById('divId')
oDiv.getElementsByTagName('p')
// 通过标签获取，上下文可以是 document 和 dom 节点，返回值是一个类数组，没有找到返回空数组

document.getElementByClassName('class')
// 通过类名获取，上下文可以是 document 和 dom 节点，返回值是一个类数组，没有找到返回空数组，ie8 以下不兼容

document.querySelector('div.class')
// 通过选择器获取，上下文可以是 document 和 dom 节点，返回值只获取到一个元素

document.querySelectorAll('div.class')
// 通过选择器获取，上下文可以是 document 和 dom 节点，返回值是一个类数组

document.documentElement // 获取 html
document.body // 获取 body
document.body.contentEditable="true" // 文本可编辑
document.querySelector('video').playbackRate = 3; // 视频播放速度
```

<!-- more -->

#### 1. 获取 dom 位置

[Element.getBoundingClientRect() - Web API 接口参考 | MDN (mozilla.org)](https://developer.mozilla.org/zh-CN/docs/Web/API/Element/getBoundingClientRect)

`getBoundingClientRect()` 方法返回元素的大小及其相对于视口的位置。

<img src="https://s2.loli.net/2021/12/21/M3QHBTLyD7lkois.png" alt="image-20211221161318473" style="zoom: 50%;" />

![image-20211221161204962](https://s2.loli.net/2021/12/21/f8qvTVY1Zkh6oRM.png)

```js
// 判断元素是否处于视口内
function isElementInViewport (el, offset = 0) {
    const box = el.getBoundingClientRect(),
          top = (box.top >= 0),
          left = (box.left >= 0),
          bottom = (box.bottom <= (window.innerHeight || document.documentElement.clientHeight) + offset),
          right = (box.right <= (window.innerWidth || document.documentElement.clientWidth) + offset);
    return (top && left && bottom && right);
}
```

#### 2. 移动 dom

[Element.scrollIntoView() - Web API 接口参考 | MDN (mozilla.org)](https://developer.mozilla.org/zh-CN/docs/Web/API/Element/scrollIntoView)

滚动 dom 的父容器，使 dom 对用户可见，平滑移向中间。

`dom.scrollIntoView({ block: 'nearest', inline: 'center', behavior: 'smooth' })`

```js
element.scrollIntoView(); // 等同于 element.scrollIntoView(true)
element.scrollIntoView(alignToTop); // Boolean 型参数
element.scrollIntoView(scrollIntoViewOptions); // Object 型参数
```

`alignToTop` 参数

- 如果为`true`，元素的顶端将和其所在滚动区的可视区域的顶端对齐。相应的 `scrollIntoViewOptions: {block: "start", inline: "nearest"}`。这是这个参数的默认值。
- 如果为`false`，元素的底端将和其所在滚动区的可视区域的底端对齐。相应的`scrollIntoViewOptions: {block: "end", inline: "nearest"}`。

`scrollIntoViewOptions` 参数属性

- `behavior` 可选

  定义动画过渡效果， `"auto"`或 `"smooth"` 之一。默认为 `"auto"`。

- `block` 可选

  定义垂直方向的对齐， `"start"`, `"center"`, `"end"`, 或 `"nearest"`之一。默认为 `"start"`。

- `inline` 可选

  定义水平方向的对齐， `"start"`, `"center"`, `"end"`, 或 `"nearest"`之一。默认为 `"nearest"`。

#### 3. 获取 dom 相关节点

```js
var dom = document.getElementById("c-dom");
var parent = dom.parentNode; // 父节点
var chils = dom.childNodes; // 返回全部子节点，包含属性节点，文本节点。
var children = dom.children; // 只返回 HTML 元素直系子节点，不包括文本节点和属性节点。
var first = dom.firstChild; // 第一个子节点
var last = dom.lastChild; // 最后一个子节点 
var previous = dom.previousSibling; // 上一个兄弟节点
var next = dom.nextSbiling; // 下一个兄弟节点
```

#### 4. 增删 dom

```js
// 增删子元素
var box = document.getElementById("dadkillme");
box.parentNode.removeChild(box);
box.parentNode.appendChild(box);
// 删除自身
var box = document.getElementById("ikillmyself");
box.remove();
```

#### 5. 操作 class

##### ① html5（不兼容 ie10 以下：）

```js
var dom = document.getElementById('domid');
// 检测
if (dom.classList.contains('c-page')) console.log('包含 c-page 类');
// 添加
dom.classList.add('c-new');
// 删除
dom.classList.remove('c-old');
// 切换
dom.classList.toggle('c-toggle');
```

##### ② 兼容

```js
var dom = document.getElementById('domid');
var classList = dom.getAttribute('class');
// 检测
if (classList.indexOf('c-page') > -1 || dom.className.indexOf('c-page') > -1) {
    console.log('两种方法，任选其一');
}
// 添加
dom.setAttribute('class', classList.concat(' c-new'));
// 删除
dom.setAttribute('class', classList.replace('c-old', ''));
```

#### 6. 操作 style

##### ① 设置：`dom.style.<attribute>`

使用该方法读取到的只有元素内联的样式。如果一个元素为`<div style='color:red，left:40px'></div>`，那么 `ele.style.color` 就会返回 red，如果使用 `ele.style.left` 就会返回 40px。该方法可以同时设置或者获取某一个元素的样式。

##### ② 获取：`window.getComputedStyle()` 与 `getPropertyValue`

![image-20211221170542641](https://s2.loli.net/2021/12/21/utw78eRfTsD4JUo.png)

```js
getComputedStyle(document.getElementById('caseroul'),null).getPropertyValue('left');
// 可以读取所有的样式属性值，但不能通过该方法去设置样式属性值。
```

##### ③ 获取：currentStyle 与 getAttribute（兼容 ie）

`currentStyle` 获取的是一个元素的所有的样式属性值，这一点功能是与 `getComputedStyle()` 一样的，但是在获取某一个具体的属性的时候，可以结合 `getAttribute` 来实现。
和 `getComputedStyle` 方法不同的是，`currentStyle` 要获得属性名的话必须采用驼峰式的写法。也就是如果我需要获取 `font-size` 属性，那么传入的参数应该是 `fontSize`。因此在 IE 中要获得单个属性的值，就必须将属性名转为驼峰形式。

```js
// IE 下将 CSS 命名转换为驼峰表示法
// font-size --> fontSize
// 利用正则处理一下就可以了
function camelize(attr) {
    // /\-(\w)/g 正则内的 (\w) 是一个捕获，捕获的内容对应后面 function 的 letter
    // 意思是将 匹配到的 -x 结构的 x 转换为大写的 X (x 这里代表任意字母)
    return attr.replace(/\-(\w)/g, function(all, letter) {
        return letter.toUpperCase();
    });
}
// 使用 currentStyle.getAttribute 获取元素 element 的 style 属性样式
element.currentStyle.getAttribute(camelize(style));
```

#### 7. 面试题

① 将 A 元素拖拽并放置到 B 元素中，B 元素需要做哪项操作（）？

- **`event.preventDefault()`**
- `event.prevent()`
- `event.drag()`
- `event.drop()`

<!-- event.preventDefault() 用于阻止浏览器的默认行为，防止在拖拽过程中因为拖拽，浏览器所产生的页面跳转等不被希望的行为。-->

② 以下不支持冒泡的鼠标事件为（   ）？

- mouseover
- click
- **mouseleave**
- mousemove

<!-- 不支持冒泡的 UI 事件：load、unload、resize、abort、error
不支持冒泡的焦点事件：blur、focus
不支持冒泡的鼠标事件：mouseleave、mouseenter -->



