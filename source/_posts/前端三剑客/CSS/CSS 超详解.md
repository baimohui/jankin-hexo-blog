---
title: CSS 核心
categories: 
- CSS
tags:
- CSS
---

## （一）核心知识

### 1. @规则

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

CSS 规则是样式表的主体，但有时需要在样式表中包括其它一些信息，比如字符集，导入其它的外部样式表，字体等，这些需要专门用@语句表示。CSS 里包含了以下@规则：

[@namespace](https://developer.mozilla.org/zh-CN/docs/Web/CSS/@namespace) 

告诉 CSS 引擎必须考虑XML命名空间。

[@media](https://developer.mozilla.org/zh-CN/docs/Web/CSS/@media)

如果满足媒体查询的条件则条件规则组里的规则生效。

[@page](https://developer.mozilla.org/zh-CN/docs/Web/CSS/@page)

描述打印文档时布局的变化.

[@font-face](https://developer.mozilla.org/zh-CN/docs/Web/CSS/@font-face)

描述将下载的外部的字体。

[@keyframes](https://developer.mozilla.org/zh-CN/docs/Web/CSS/@keyframes)

描述 CSS 动画的关键帧。

[@document](https://developer.mozilla.org/zh-CN/docs/Web/CSS/@document)

如果文档样式表满足给定条件则条件规则组里的规则生效。 (推延至 CSS Level 4 规范)

[@charset](https://developer.mozilla.org/zh-CN/docs/Web/CSS/@charset) 

用于定义样式表使用的字符集。它必须是样式表中的第一个元素。如果有多个 `@charset` 被声明，只有第一个会被使用，而且不能在HTML元素或HTML页面的 `<style>` 元素内使用。

注意：值必须是双引号包裹，且和

```css
@charset "UTF-8";
```

平时写样式文件都没写 @charset 规则，那这个 CSS 文件到底是用的什么字符编码的呢？

某个样式表文件到底用的是什么字符编码，浏览器有一套识别顺序（优先级由高到低）：

- 文件开头的 [Byte order mark](https://en.wikipedia.org/wiki/Byte_order_mark) 字符值，不过一般编辑器并不能看到文件头里的 BOM 值；

- HTTP 响应头里的 `content-type` 字段包含的 `charset` 所指定的值，比如：

  ```
  Content-Type: text/css; charset=utf-8
  ```

- CSS 文件头里定义的 @charset 规则里指定的字符编码；

- `<link>` 标签里的 charset 属性，该条已在 HTML5 中废除；

- 默认是 `UTF-8`。

[@import](https://developer.mozilla.org/zh-CN/docs/Web/CSS/@import) 

用于告诉 CSS 引擎引入一个外部样式表。

link 和 @import 都能导入一个样式文件，它们有什么区别嘛？

- link 是 HTML 标签，除了能导入 CSS 外，还能导入别的资源，比如图片、脚本和字体等；而 @import 是 CSS 的语法，只能用来导入 CSS；
- link 导入的样式会在页面加载时同时加载，@import 导入的样式需等页面加载完成后再加载；
- link 没有兼容性问题，@import 不兼容 ie5 以下；
- link 可以通过 JS 操作 DOM 动态引入样式表改变样式，而@import不可以。

[@supports](https://developer.mozilla.org/zh-CN/docs/Web/CSS/@supports) 

用于查询特定的 CSS 是否生效，可以结合 not、and 和 or 操作符进行后续的操作。

```css
/* 如果支持自定义属性，则把 body 颜色设置为变量 varName 指定的颜色 */
@supports (--foo: green) {
    body {
        color: var(--varName);
    }
}
```

### 2. 选择器

CSS 选择器无疑是其核心之一，对于基础选择器以及一些常用伪类必须掌握。下面列出了常用的选择器。 想要获取更多选择器的用法可以看 [MDN CSS Selectors](https://developer.mozilla.org/zh-CN/docs/Web/CSS/CSS_Selectors)。

**基础选择器**

- 标签选择器：`h1`
- 类选择器：`.checked`
- ID 选择器：`#picker`
- 通配选择器：`*`

**属性选择器**

- `[attr]`：指定属性的元素；
- `[attr=val]`：属性等于指定值的元素；
- `[attr*=val]`：属性包含指定值的元素；
- `[attr^=val]`	：属性以指定值开头的元素；
- `[attr$=val]`：属性以指定值结尾的元素；
- `[attr~=val]`：属性包含指定值(完整单词)的元素(不推荐使用)；
- `[attr|=val]`：属性以指定值(完整单词)开头的元素(不推荐使用)；

**组合选择器**

- 相邻兄弟选择器：`A + B`
- 普通兄弟选择器：`A ~ B`
- 子选择器：`A > B`
- 后代选择器：`A B`

**伪类**

条件伪类

- `:lang()`：基于元素语言来匹配页面元素；
- `:dir()`：匹配特定文字书写方向的元素；
- `:has()`：匹配包含指定元素的元素；
- `:is()`：匹配指定选择器列表里的元素；
- `:not()`：用来匹配不符合一组选择器的元素；

行为伪类

- `:active`：鼠标激活的元素；
- `:hover`：鼠标悬浮的元素；
- `::selection`：鼠标选中的元素；

状态伪类

- `:target`：当前锚点的元素；
- `:link`：未访问的链接元素；
- `:visited`：已访问的链接元素；
- `:focus`：输入聚焦的表单元素；
- `:required`：输入必填的表单元素；
- `:valid`：输入合法的表单元素；
- `:invalid`：输入非法的表单元素；
- `:in-range`：输入范围以内的表单元素；
- `:out-of-range`：输入范围以外的表单元素；
- `:checked`：选项选中的表单元素；
- `:optional`：选项可选的表单元素；
- `:enabled`：事件启用的表单元素；
- `:disabled`：事件禁用的表单元素；
- `:read-only`：只读的表单元素；
- `:read-write`：可读可写的表单元素；
- `:blank`：输入为空的表单元素；
- `:current()`：浏览中的元素；
- `:past()`：已浏览的元素；
- `:future()`：未浏览的元素；

结构伪类

- `:root`：文档的根元素；
- `:empty`：无子元素的元素；
- `:first-letter`：元素的首字母；
- `:first-line`：元素的首行；
- `:nth-child(n)`：元素中指定顺序索引的元素；
- `:nth-last-child(n)`：元素中指定逆序索引的元素；；
- `:first-child	`：元素中为首的元素；
- `:last-child`：元素中为尾的元素；
- `:only-child`：父元素仅有该元素的元素；
- `:nth-of-type(n)	`：标签中指定顺序索引的标签；
- `:nth-last-of-type(n)`：标签中指定逆序索引的标签；
- `:first-of-type`：标签中为首的标签；
- `:last-of-type`：标签中为尾标签；
- `:only-of-type`：父元素仅有该标签的标签；

**伪元素**

- `::before`：在元素前插入内容；
- `::after`：在元素后插入内容；

### 3. 优先级

优先级就是分配给指定的 CSS 声明的一个权重，它由匹配的选择器中的每一种选择器类型的数值决定。为了记忆，可以把权重分成如下几个等级，数值越大的权重越高：

- 10000：!important；
- 01000：内联样式；
- 00100：ID 选择器；
- 00010：类选择器、伪类选择器、属性选择器；
- 00001：元素选择器、伪元素选择器；
- 00000：通配选择器、后代选择器、兄弟选择器；

可以看到内联样式（通过元素中 style 属性定义的样式）的优先级大于任何选择器；而给属性值加上 `!important` 又可以把优先级提至最高，就是因为它的优先级最高，所以需要谨慎使用它，以下有些使用注意事项：

- 一定要优先考虑使用样式规则的优先级来解决问题而不是 !important；
- 只有在需要覆盖全站或外部 CSS 的特定页面中使用 !important；
- 永远不要在你的插件中使用 !important；
- 永远不要在全站范围的 CSS 代码中使用 !important；

### 4. 盒模型

在 CSS 中任何元素都可以看成是一个盒子，而一个盒子是由 4 部分组成的：内容（content）、内边距（padding）、边框（border）和外边距（margin）。盒模型有 2 种：标准盒模型和 IE 盒模型，分别是由 W3C 和 IExplore 制定的标准。

如果给某个元素设置如下样式：

```css
.box {
    width: 200px;
    height: 200px;
    padding: 10px;
    border: 1px solid #eee;
    margin: 10px;
}
```

在W3C标准下，元素的width值即为盒模型中的content的宽度值，height值即为盒模型中的content的⾼度值。

所以 `.box` 元素内容的宽度就为 `200px`，而实际的宽度则是 `width` + `padding-left` + `padding-right` + `border-left-width` + `border-right-width` = 200 + 10 + 10 + 1 + 1 = 222。

<img src="https://cdn.jsdelivr.net/gh/baimohui/FigureBed/img/20211106174046.png" alt="image-20210422133343636" style="zoom:67%;" />

所以 `.box` 元素内容的宽度就为 `200px`，而实际的宽度则是 `width` + `padding-left` + `padding-right` + `border-left-width` + `border-right-width` = 200 + 10 + 10 + 1 + 1 = 222。

IE 盒模型：元素占据的宽度 = margin-left + width + margin-right

<img src="https://cdn.jsdelivr.net/gh/baimohui/FigureBed/img/20211106174047.png" alt="image-20210422133421452" style="zoom:67%;" />

`.box` 元素所占用的实际宽度为 `200px`，而内容的真实宽度则是 `width` - `padding-left` - `padding-right` - `border-left-width` - `border-right-width` = 200 - 10 - 10 - 1 - 1 = 178。

现在高版本的浏览器基本上默认都是使用标准盒模型，而像 IE6 这种老古董才是默认使用 IE 盒模型的。

在  CSS3 中新增了一个属性 `box-sizing`，允许开发者来指定盒子使用什么标准，它有 2 个值：

- `content-box`：标准盒模型；
- `border-box`：IE 盒模型；

### 5. 媒体查询

媒体查询是指针对不同的设备、特定的设备特征或者参数进行定制化的修改网站的样式。

可以通过给 `<link>` 加上 media 属性来指定该样式文件只能对什么设备生效，不指定的话默认是 all，即对所有设备都生效：

```html
<link rel="stylesheet" src="styles.css" media="screen" />
<link rel="stylesheet" src="styles.css" media="print" />
```

- all：适用于所有设备；
- print：适用于在打印预览模式下在屏幕上查看的分页材料和文档；
- screen：主要用于屏幕；
- speech：主要用于语音合成器。

就算资源不匹配media指定的设备类型，但是浏览器依然会加载它。

<img src="https://cdn.jsdelivr.net/gh/baimohui/FigureBed/img/20211106174048.png" alt="image-20210422135634976" style="zoom:67%;" />

除了通过 `<link>` 让指定设备生效外，还可以通过 `@media` 让 CSS 规则在特定的条件下才能生效。响应式页面就是使用了 @media 才让一个页面能够同时适配 PC、Pad 和手机端。

```css
@media (min-width: 1000px) {}
```

媒体查询支持逻辑操作符：

- and：查询条件都满足的时候才生效；
- not：查询条件取反；
- only：整个查询匹配的时候才生效，常用语兼容旧浏览器，使用时候必须指定媒体类型；
- 逗号或者 or：查询条件满足一项即可匹配；

媒体查询还支持[众多的媒体特性](https://developer.mozilla.org/zh-CN/docs/Web/Guide/CSS/Media_queries#媒体特性)，使得它可以写出很复杂的查询条件：

```css
/* 用户设备的最小高度为680px或为纵向模式的屏幕设备 */
@media (min-height: 680px), screen and (orientation: portrait) {}
```

### 6. 过渡

下表列出了所有 CSS 过渡属性：

| 属性                                                         | 描述                                |
| :----------------------------------------------------------- | :---------------------------------- |
| [transition](https://www.w3school.com.cn/cssref/pr_transition.asp) | 简写属性，同时设置下边四个过渡属性  |
| [transition-duration](https://www.w3school.com.cn/cssref/pr_transition-duration.asp) | 规定过渡效果要持续多少秒或毫秒      |
| [transition-delay](https://www.w3school.com.cn/cssref/pr_transition-delay.asp) | 规定过渡效果的延迟（以秒计）        |
| [transition-property](https://www.w3school.com.cn/cssref/pr_transition-property.asp) | 规定过渡效果所针对的 CSS 属性的名称 |
| [transition-timing-function](https://www.w3school.com.cn/cssref/pr_transition-timing-function.asp) | 规定过渡效果的速度曲线              |



### 7. vertical-align

vertical-align 属性常用于将相邻的文本与元素垂直对齐。[MDN](https://link.juejin.cn?target=https%3A%2F%2Fdeveloper.mozilla.org%2Fzh-CN%2Fdocs%2FWeb%2FCSS%2Fvertical-align) 中将其定义为一种简单的 CSS 属性，用来指定行内元素（inline）或表格单元格（table-cell）元素的垂直对齐方式。vertical-align 的取值分为 baseline、top、middle、bottom。

**baseline**

vertical-align 的默认值，意思为基线对齐。（字母 x 底部所在的位置即为基线，具体可参考 [字母’x’在CSS世界中的角色和故事](https://link.juejin.cn/?target=https%3A%2F%2Fwww.zhangxinxu.com%2Fwordpress%2F2015%2F06%2Fabout-letter-x-of-css%2F)）

![image-20210808215528288](https://cdn.jsdelivr.net/gh/baimohui/FigureBed/img/20211106174049.png)









## （二）常见需求

#### 

圣杯布局：https://www.jianshu.com/p/f9bcddb0e8b4









