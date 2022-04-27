---
title: JSON 相关
categories: 
- JavaScript
tags:
- JavaScript
- JSON
---

![image-20220224220101127](https://cdn.jsdelivr.net/gh/baimohui/FigureBed/img/20220224220101.png)

<!--more-->

```js
JSON.stringify(123) // '123'
JSON.stringify('123') // '"123"'
JSON.stringify(true) // 'true'
JSON.stringify(NaN) // null
JSON.stringify(Infinity) // null
JSON.stringify(function(){}) // undefined
JSON.stringify({ x: [10, undefined, function(){}, Symbol('')] }) // '{"x":[10,null,null,null]}'
```

