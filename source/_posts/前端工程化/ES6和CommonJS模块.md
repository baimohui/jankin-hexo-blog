---
title: ES6 Module 和 CommonJS 模块详解
categories:
- 模块打包
tags:
- ES6
- CommonJS
- 模块系统
---

理解 ES6 Module（ESM）和 CommonJS（CJS）这两种模块系统的区别，是掌握前端工程化的基础。本文将从多个维度深入剖析两者的差异，帮助你彻底理解它们。<!-- more -->

---

## 一、什么是模块系统？

### 为什么需要模块？

在没有模块系统的时代，JavaScript 开发面临以下问题：

```html
<script src="utils.js"></script>
<script src="component.js"></script>
<script src="app.js"></script>
```

**问题：**
- 全局变量污染：所有变量都在 window 上
- 依赖关系不清晰：必须手动按顺序引入
- 难以维护：代码多了之后，不知道谁依赖谁
- 无法复用：代码难以在不同项目间复用

**模块系统解决的问题：**
- 每个模块有独立作用域，不会污染全局
- 明确的依赖声明
- 支持代码复用
- 更好的代码组织

---

## 二、两种模块系统简介

### CommonJS（CJS）

**诞生背景：**
- 2009 年由 Node.js 采用
- 主要用于服务端（Node.js）
- 同步加载，适合服务端（文件都在本地）

**特点：**
- 使用 `require()` 导入
- 使用 `module.exports` 或 `exports` 导出
- 运行时加载
- 值的拷贝

---

### ES6 Module (ESM)

**诞生背景：**
- 2015 年 ES6（ES2015）正式引入
- JavaScript 语言层面的模块系统
- 浏览器和 Node.js 都支持

**特点：**
- 使用 `import` 导入
- 使用 `export` 导出
- 编译时加载（静态分析）
- 值的引用

---

## 三、核心区别对比

| 维度 | CommonJS | ES6 Module |
|------|-----------|------------|
| **语法** | `require()` / `module.exports` | `import` / `export` |
| **加载时机** | 运行时加载 | 编译时加载（静态） |
| **加载方式** | 同步 | 异步（浏览器） |
| **导出方式** | 值的拷贝 | 值的引用（动态绑定） |
| **适用环境** | Node.js | 浏览器 + Node.js |
| **静态分析** | 不支持 | 支持（Tree Shaking） |
| **提升** | 不提升 | import 会提升到顶部 |

---

## 四、语法详解

### 1. CommonJS 语法

#### 导出

**方式一：module.exports**
```js
// utils.js
const add = (a, b) => a + b;
const multiply = (a, b) => a * b;

module.exports = {
  add,
  multiply
};
```

**方式二：exports（简写）**
```js
// utils.js
exports.add = (a, b) => a + b;
exports.multiply = (a, b) => a * b;
```

**注意：** `exports` 是 `module.exports` 的引用，不要直接赋值给 `exports`

```js
// ❌ 错误写法
exports = { add: () => {} };  // 这样不会生效

// ✅ 正确写法
module.exports = { add: () => {} };
```

#### 导入

```js
// main.js
const utils = require('./utils');
console.log(utils.add(1, 2));        // 3
console.log(utils.multiply(3, 4));   // 12

// 解构导入
const { add, multiply } = require('./utils');
```

#### 动态导入（任何地方都可以）

```js
// require 可以在任意位置调用
if (Math.random() > 0.5) {
  const moduleA = require('./moduleA');
} else {
  const moduleB = require('./moduleB');
}
```

---

### 2. ES6 Module 语法

#### 导出

**命名导出（Named Exports）**
```js
// utils.js
export const add = (a, b) => a + b;
export const multiply = (a, b) => a * b;

// 或者集中导出
const add = (a, b) => a + b;
const multiply = (a, b) => a * b;
export { add, multiply };
```

**默认导出（Default Export）**
```js
// Calculator.js
export default class Calculator {
  add(a, b) {
    return a + b;
  }
}
```

**混合导出**
```js
// utils.js
export const add = (a, b) => a + b;
export const multiply = (a, b) => a * b;

const PI = 3.14159;
export default PI;
```

#### 导入

**命名导入**
```js
import { add, multiply } from './utils.js';
console.log(add(1, 2));  // 3
```

**重命名导入**
```js
import { add as sum, multiply as product } from './utils.js';
```

**默认导入**
```js
import PI from './utils.js';
import Calculator from './Calculator.js';
```

**命名 + 默认混合导入**
```js
import PI, { add, multiply } from './utils.js';
```

**整体导入**
```js
import * as utils from './utils.js';
console.log(utils.add(1, 2));
console.log(utils.default);  // 默认导出
```

#### 动态导入（import()）

```js
// 返回 Promise
button.onclick = async () => {
  const module = await import('./Chart.js');
  module.renderChart();
};

// 或者
import('./Chart.js')
  .then(module => {
    module.renderChart();
  });
```

---

## 五、关键特性深度解析

### 1. 运行时加载 vs 编译时加载

#### CommonJS：运行时加载

```js
// main.js
const utils = require('./utils');
```

**执行流程：**
1. 代码运行到这一行
2. 读取 `utils.js` 文件
3. 执行 `utils.js` 代码
4. 返回 `module.exports` 对象
5. 赋值给 `utils` 变量

**特点：**
- `require()` 是一个函数调用
- 只有运行时才知道导入了什么
- 可以动态判断导入哪个模块

---

#### ES6 Module：编译时加载（静态分析）

```js
// main.js
import { add } from './utils.js';
```

**执行流程：**
1. 编译时（代码运行前）扫描 `import` 语句
2. 分析依赖关系
3. 生成模块依赖图
4. 运行时执行代码

**特点：**
- `import` 是声明式的，不是函数
- 必须在模块顶部，不能在条件语句中
- 编译时就能确定依赖关系
- 支持 Tree Shaking（删除未使用代码）

**为什么 import 必须在顶部？**

```js
// ❌ 错误写法
if (Math.random() > 0.5) {
  import { add } from './utils.js';  // 不允许！
}

// ✅ 正确写法
import { add } from './utils.js';
if (Math.random() > 0.5) {
  console.log(add(1, 2));
}
```

原因：`import` 在编译时处理，此时条件语句还没执行。

---

### 2. 值的拷贝 vs 值的引用

#### CommonJS：值的拷贝

```js
// counter.js
let count = 0;

exports.getCount = () => count;
exports.increment = () => count++;
```

```js
// main.js
const { getCount, increment } = require('./counter');

console.log(getCount());  // 0
increment();
console.log(getCount());  // 1

// 再导入一次
const counter2 = require('./counter');
console.log(counter2.getCount());  // 1（从缓存读取，不是 0）
```

**注意：** CommonJS 导出的是**值的拷贝**

```js
// number.js
let num = 1;
module.exports = num;
num = 2;
```

```js
// main.js
const num = require('./number');
console.log(num);  // 1（拷贝的值，不会变）
```

---

#### ES6 Module：值的引用（动态绑定）

```js
// counter.js
export let count = 0;
export const increment = () => count++;
```

```js
// main.js
import { count, increment } from './counter.js';

console.log(count);  // 0
increment();
console.log(count);  // 1（自动更新！）
```

**ES6 Module 导出的是引用，不是拷贝**

```js
// number.js
export let num = 1;
setTimeout(() => num = 2, 500);
```

```js
// main.js
import { num } from './number.js';

console.log(num);  // 1
setTimeout(() => console.log(num), 600);  // 2（动态更新！）
```

---

### 3. 模块缓存

#### CommonJS 缓存机制

```js
// example.js
console.log('example.js 被执行了！');
module.exports = { say: 'hi' };
```

```js
// main.js
require('./example');  // 输出：example.js 被执行了！
require('./example');  // 没有输出（从缓存读取）
require('./example');  // 没有输出（从缓存读取）
```

**修改缓存的值：**

```js
// main.js
const example1 = require('./example');
example1.say = 'hello';

const example2 = require('./example');
console.log(example2.say);  // hello（缓存被修改了）
```

**删除缓存：**

```js
delete require.cache[require.resolve('./example')];
const example3 = require('./example');  // 重新执行
```

---

#### ES6 Module 缓存

ES6 Module 也有缓存，但机制不同：

```js
// example.js
console.log('example.js 被执行了！');
export const say = 'hi';
```

```js
// main.js
import './example.js';  // 输出：example.js 被执行了！
import './example.js';  // 没有输出（缓存）
```

---

## 六、循环依赖

### 什么是循环依赖？

```
A 依赖 B
  ↑     ↓
B 依赖 A
```

### CommonJS 中的循环依赖

```js
// a.js
console.log('a.js 开始执行');
exports.done = false;
const b = require('./b.js');
console.log('a.js 中 b.done =', b.done);
exports.done = true;
console.log('a.js 执行完毕');
```

```js
// b.js
console.log('b.js 开始执行');
exports.done = false;
const a = require('./a.js');
console.log('b.js 中 a.done =', a.done);
exports.done = true;
console.log('b.js 执行完毕');
```

```js
// main.js
console.log('main.js 开始执行');
const a = require('./a.js');
const b = require('./b.js');
console.log('main.js 中 a.done =', a.done);
console.log('main.js 中 b.done =', b.done);
```

**输出结果：**
```
main.js 开始执行
a.js 开始执行
b.js 开始执行
b.js 中 a.done = false
b.js 执行完毕
a.js 中 b.done = true
a.js 执行完毕
main.js 中 a.done = true
main.js 中 b.done = true
```

**CommonJS 循环依赖的特点：**
- 遇到循环依赖时，返回已导出的部分
- 未执行到的代码不会导出
- 可能拿到不完整的模块

---

### ES6 Module 中的循环依赖

```js
// a.js
console.log('a.js 开始执行');
export let done = false;
import { done as bDone } from './b.js';
console.log('a.js 中 bDone =', bDone);
done = true;
console.log('a.js 执行完毕');
```

```js
// b.js
console.log('b.js 开始执行');
export let done = false;
import { done as aDone } from './a.js';
console.log('b.js 中 aDone =', aDone);
done = true;
console.log('b.js 执行完毕');
```

**ES6 循环依赖的特点：**
- 建立引用，等待值填充
- 如果在值设置前访问，可能是 `undefined`
- 需要注意代码执行顺序

---

## 七、在浏览器和 Node.js 中使用

### 浏览器中使用 ES6 Module

```html
<!-- 必须添加 type="module" -->
<script type="module" src="main.js"></script>
```

```js
// main.js
import { add } from './utils.js';
console.log(add(1, 2));
```

**注意：**
- 必须通过服务器访问（不能用 file:// 协议）
- 默认是严格模式
- 顶级 `this` 是 `undefined`，不是 `window`

---

### Node.js 中使用 ES6 Module

Node.js 12+ 支持 ES6 Module，有两种方式：

**方式一：文件后缀改为 .mjs**

```js
// utils.mjs
export const add = (a, b) => a + b;
```

```js
// main.mjs
import { add } from './utils.mjs';
```

**方式二：package.json 中设置 type: "module"**

```json
{
  "type": "module"
}
```

```js
// utils.js（不需要 .mjs 后缀）
export const add = (a, b) => a + b;
```

**Node.js 中混用两种模块：**

```js
// ES6 Module 中导入 CommonJS
import cjsModule from './commonjs-module.cjs';

// CommonJS 中导入 ES6 Module（需要动态导入）
const esmModule = await import('./es-module.js');
```

---

## 八、Babel 如何转译 import

因为有些环境还不支持 ES6 Module，Babel 会把 `import` 转译成 `require`。

**输入：**
```js
// es6.js
import { add } from './utils';
import * as utils from './utils';
import Calculator from './Calculator';
```

**Babel 转译后：**
```js
// 转译后
var _utils = require('./utils');
var _utils2 = _interopRequireWildcard(_utils);
var _Calculator = require('./Calculator');
var _Calculator2 = _interopRequireDefault(_Calculator);

function _interopRequireDefault(obj) {
  return obj && obj.__esModule ? obj : { default: obj };
}

function _interopRequireWildcard(obj) {
  if (obj && obj.__esModule) {
    return obj;
  }
  var newObj = {};
  if (obj != null) {
    for (var key in obj) {
      if (Object.prototype.hasOwnProperty.call(obj, key)) {
        newObj[key] = obj[key];
      }
    }
  }
  newObj.default = obj;
  return newObj;
}
```

---

## 九、最佳实践

### 1. 选择哪种模块系统？

| 场景 | 推荐 |
|------|------|
| 新项目（前端） | ES6 Module |
| Node.js 新项目 | ES6 Module |
| 老旧 Node.js 项目 | CommonJS |
| 需要 Tree Shaking | ES6 Module |

---

### 2. ES6 Module 最佳实践

**✅ 推荐做法：**

```js
// 导入放在文件顶部
import { useState } from 'react';
import axios from 'axios';

// 命名导出清晰
export const fetchUser = async (id) => {
  const res = await axios.get(`/users/${id}`);
  return res.data;
};

export const updateUser = async (id, data) => {
  const res = await axios.put(`/users/${id}`, data);
  return res.data;
};
```

**❌ 避免做法：**

```js
// 不要在条件中使用 import
if (something) {
  import('./module');  // 用 import() 替代
}

// 不要整体导入后只使用一部分
import * as _ from 'lodash';  // 改用 import debounce from 'lodash/debounce'
```

---

### 3. CommonJS 最佳实践

**✅ 推荐做法：**

```js
// 明确导出
const add = (a, b) => a + b;
const multiply = (a, b) => a * b;

module.exports = {
  add,
  multiply
};
```

**❌ 避免做法：**

```js
// 不要混用 exports 和 module.exports
exports.add = (a, b) => a + b;
module.exports = { multiply: (a, b) => a * b };  // 会覆盖上面的

// 不要直接赋值给 exports
exports = { add: (a, b) => a + b };  // 不生效
```

---

## 十、常见面试题

### Q1：CommonJS 和 ES6 Module 的区别？

**回答要点：**
1. 语法不同：`require/module.exports` vs `import/export`
2. 加载时机：运行时 vs 编译时
3. 导出方式：值拷贝 vs 值引用
4. 静态分析：CommonJS 不支持，ESM 支持 Tree Shaking
5. 适用环境：Node.js vs 浏览器 + Node.js

---

### Q2：为什么 ES6 Module 可以 Tree Shaking？

**回答：**
因为 ESM 是静态的，`import` 在编译时就能确定哪些导出被使用。而 CommonJS 的 `require()` 是运行时的，无法静态分析。

---

### Q3：import 会提升吗？

**回答：**
会，`import` 会提升到模块顶部，但建议还是写在顶部以便阅读。

---

### Q4：如何在浏览器中使用 ES6 Module？

**回答：**
在 script 标签添加 `type="module"`，并且通过服务器访问。

---

## 十一、总结

| 特性 | CommonJS | ES6 Module |
|------|-----------|------------|
| **推荐使用** | 老项目 | 新项目 |
| **语法** | 简单 | 更灵活 |
| **Tree Shaking** | ❌ | ✅ |
| **浏览器原生支持** | ❌ | ✅ |
| **动态导入** | ✅ 任意位置 | ✅ 用 import() |
| **循环依赖** | 返回已导出部分 | 引用绑定 |

**最终建议：**
- 新项目统一使用 **ES6 Module**
- 老项目保持 CommonJS 或逐步迁移
- 理解两者的差异，避免踩坑
