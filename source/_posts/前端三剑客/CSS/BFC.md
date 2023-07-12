---
title: BFC 详解
categories: 
- CSS
tags: 
- CSS
- 响应式布局
---

## BFC 到底是什么东西

`BFC`（`Block Formatting Context`/块级格式化上下文）可以看作是是⼀块独⽴的区域，让处于其内部的元素与外部的元素互相隔离。

`W3C`官方解释为：`BFC`决定了元素如何对其内容进行定位，以及与其它元素的关系和相互作用，当涉及到可视化布局时，`Block Formatting Context`提供了一个环境，`HTML`在这个环境中按照一定的规则进行布局。

<!--more-->

## 怎样触发 BFC

这里简单列举几个触发`BFC`使用的`CSS`属性

- 根元素，即 HTML 元素
- overflow 不为 visible
- position: absolute | fixed
- display: table-cell | inline-block | flex

## BFC 的规则

- `BFC` 就是页面中的一个隔离的独立容器，容器内部元素不会影响外部元素
- `BFC` 就是一个块级元素，块级元素会在垂直方向一个接一个的排列（两个相邻的块级元素之间的上下外边距会发生重叠，重叠后的外边距为两者中的较大值。创建新的 `BFC` 可以避免外边距重叠）
- 计算 `BFC` 的高度时，浮动元素也参与计算
- `BFC`  区域不会与浮动的容器发生重叠
- 每个元素的左 `margin` 值和容器的左 `border` 相接触

## BFC 解决了什么问题

### 1. 使用 Float 脱离文档流，高度塌陷

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>高度塌陷</title>
    <style>
        .box {
            margin: 100px;
            width: 100px;
            height: 100px;
            background: red;
            float: left;
        }
        .container {
            background: #000;
        }
    </style>
</head>
<body>
    <div class="container">
        <div class="box"></div>
        <div class="box"></div>
    </div>
</body>
</html>
```

**效果：**

<img src="https://cdn.jsdelivr.net/gh/baimohui/FigureBed/img/20220304003133.png" alt="image-20220304003133319" style="zoom:80%;" />

可以看到上面效果给`box`设置完`float`结果脱离文档流，使`container`高度没有被撑开，从而背景颜色没有颜色出来，解决此问题可以给`container`触发`BFC`，上面我们所说到的触发`BFC`属性都可以设置。

**修改代码**

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>高度塌陷</title>
    <style>
        .box {
            margin: 100px;
            width: 100px;
            height: 100px;
            background: red;
            float: left;
        }
        .container {
            background: #000;
            display: inline-block;
        }
    </style>
</head>
<body>
    <div class="container">
        <div class="box"></div>
        <div class="box"></div>
    </div>
</body>
</html>
```

**效果：**

![img](https://cdn.jsdelivr.net/gh/baimohui/FigureBed/img/20220304003111.webp)

### 2. 外边距重叠

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Document</title>
    <style>
        .box {
            margin: 10px;
            width: 100px;
            height: 100px;
            background: #000;
        }
    </style>
</head>
<body>
    <div class="container">
        <div class="box"></div>
        <div class="box"></div>
    </div>
</body>
</html>
```

**效果：**

![img](https://cdn.jsdelivr.net/gh/baimohui/FigureBed/img/20220304003107.webp)

可以看到上面我们为两个盒子的`margin`外边距设置的是`10px`，可结果显示两个盒子之间只有`10px`的距离，这就导致了`margin`塌陷问题。为了解决此问题可以使用`BFC`规则（为元素包裹一个 BFC 的盒子，使得 BFC 盒子的内部元素不受外面布局影响），或者简单粗暴地将一个设置`margin`，一个设置`padding`。

**修改代码**

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Document</title>
    <style>
        .box {
            margin: 10px;
            width: 100px;
            height: 100px;
            background: #000;
        }
        .box-wrapper {
          overflow: hidden;
        }
    </style>
</head>
<body>
    <div class="container">
      <div class="box"></div>
      <div class="box-wrapper"><div class="box"></div></div>
    </div>
</body>
</html>
```

**效果：**

![img](https://cdn.jsdelivr.net/gh/baimohui/FigureBed/img/20220304003100.webp)

### 3. 两栏布局

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>两栏布局</title>
    <style>
            div {
                 width: 200px;
                 height: 100px;
                 border: 1px solid red;
            }

    </style>
</head>
<body>
    <div style="float: left;">
        两栏布局两栏布局两栏布局两栏布局两栏布局两栏布局两栏布局两栏布局两栏布局两栏布局两栏布局两栏布局两栏布局
    </div>
    <div style="width: 300px;">
        我是蛙人，如有帮助请点个赞叭，如有帮助请点个赞叭，如有帮助请点个赞叭，如有帮助请点个赞叭，如有帮助请点个赞叭，如有帮助请点个赞叭
    </div>
</body>
</html>
```

**效果：**

![image-20220304003027435](https://cdn.jsdelivr.net/gh/baimohui/FigureBed/img/20220304003027.png)

可以看到上面元素，第二个`div`元素为`300px`宽度，但是被第一个`div`元素设置`Float`脱离文档流给覆盖上去了，解决此方法我们可以把第二个`div`元素设置为一个`BFC`。

**修改代码**

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>两栏布局</title>
    <style>
            div {
                 width: 200px;
                 height: 100px;
                 border: 1px solid red;
            }

    </style>
</head>
<body>
    <div style="float: left;">
        两栏布局两栏布局两栏布局两栏布局两栏布局两栏布局两栏布局两栏布局两栏布局两栏布局两栏布局两栏布局两栏布局
    </div>
    <div style="width: 300px;display:flex;">
        我是蛙人，如有帮助请点个赞叭，如有帮助请点个赞叭，如有帮助请点个赞叭，如有帮助请点个赞叭，如有帮助请点个赞叭，如有帮助请点个赞叭
    </div>
</body>
</html>
```

**效果：**

![image-20220304002958423](https://cdn.jsdelivr.net/gh/baimohui/FigureBed/img/20220304003038.png)

