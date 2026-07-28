---
title: Symbol 详解
categories: 
- JavaScript
tags:
- JavaScript
- ES6
- Symbol
- 元编程
---

## 一、Symbol 是什么，为什么需要它

Symbol 是 ES6 引入的第七种原始数据类型，表示**唯一且不可变**的值。它主要用于解决对象属性名冲突问题，以及定义对象的内部行为。<!--more-->

### 为什么需要 Symbol

在使用对象作为 Map 或混合多个模块时，属性名冲突是常见问题：

```js
// ❌ 属性名冲突
const user = {};
user.name = 'Alice';           // 业务代码
user.name = 'override';        // 另一个模块将同一个 key 覆盖了
```

Symbol 保证每次创建的值都是唯一的：

```js
const s1 = Symbol('name');
const s2 = Symbol('name');
s1 === s2;                     // false（即使描述相同，值也不同）
```

## 二、基本用法

### 创建 Symbol

```js
// 无描述的 Symbol
const s1 = Symbol();
typeof s1;                     // "symbol"

// 带描述的 Symbol（仅供调试用，不影响值）
const s2 = Symbol('unique');
s2.toString();                 // "Symbol(unique)"
String(s2);                    // "Symbol(unique)"
```

### Symbol 不是构造函数

```js
new Symbol();                  // TypeError: Symbol is not a constructor
```

Symbol 是原始类型，不能 `new`。如果需要创建包装对象，可以显式调用 `Object()`：

```js
const sym = Symbol('test');
const obj = Object(sym);
typeof obj;                    // "object"
```

### 唯一性

```js
Symbol() === Symbol();         // false
Symbol('id') === Symbol('id'); // false（即使描述相同）

// 唯一用途：作为对象的属性 key
const PROP = Symbol('private');
const obj = { [PROP]: 'secret' };
```

## 三、作为对象属性

Symbol 作为属性名时，最大的特点就是**不会与其他属性冲突**。

### 方括号语法

```js
const id = Symbol('id');
const user = {
  name: 'Alice',
  [id]: 12345,                // 计算属性名，Symbol 作为 key
};
```

### Object.defineProperty

```js
const key = Symbol('key');
Object.defineProperty(obj, key, {
  value: 'secret',
  writable: false,
});
```

### Symbol 属性不会被常规方法遍历

这是 Symbol 最重要的特性之一——隐藏内部私有属性：

```js
const token = Symbol('token');
const user = { name: 'Alice', [token]: 'abc123' };

// 以下方法都 ❌ 不会返回 Symbol 属性
Object.keys(user);            // ["name"]
Object.getOwnPropertyNames(user); // ["name"]
for (const key in user) {}    // 只遍历到 "name"
JSON.stringify(user);         // '{"name":"Alice"}'
```

### 获取 Symbol 属性的方法

```js
// ✅ 用以下方法才能获取 Symbol 属性
Object.getOwnPropertySymbols(user);  // [Symbol(token)]

// ✅ 同时获取所有属性（字符串 + Symbol）
Reflect.ownKeys(user);               // ["name", Symbol(token)]
```

**注意**：Symbol 属性只是**不参与常规遍历**，并非真正的私有。通过 `Object.getOwnPropertySymbols` 和 `Reflect.ownKeys` 仍然可以访问。

## 四、全局 Symbol 注册表

`Symbol.for()` 和 `Symbol.keyFor()` 使用全局注册表，可以在不同的代码模块间共享同一个 Symbol。

### Symbol.for（全局注册）

```js
// 从全局注册表获取（不存在则创建）
const uid = Symbol.for('app.user.id');
const sameUid = Symbol.for('app.user.id');

uid === sameUid;               // true（同一个 Symbol）

// 不同模块之间共享同一个 Symbol
// module-a.js
export const SYNC_KEY = Symbol.for('app.sync');

// module-b.js
import { SYNC_KEY } from './module-a';
const key = Symbol.for('app.sync');
key === SYNC_KEY;              // true
```

### Symbol.keyFor（查询注册名）

```js
const s = Symbol.for('foo');
Symbol.keyFor(s);              // "foo"

const s2 = Symbol('bar');      // 未注册
Symbol.keyFor(s2);             // undefined
```

### Symbol.for vs Symbol 对比

| | `Symbol()` | `Symbol.for('key')` |
|---|------------|---------------------|
| 唯一性 | 每次创建都是唯一的 | 相同 key 返回同一个 |
| 存储位置 | 内存 | 全局注册表 |
| 跨模块共享 | ❌ | ✅ |
| 序列化 | 丢失 | 丢失（但可以用 key 恢复） |

## 五、内置 Symbol（Well-known Symbols）

JavaScript 内置了多个 Symbol 常量，用于**修改对象的底层行为**。它们以 `Symbol.xxx` 的形式存在。

### Symbol.iterator（可迭代协议）

```js
const range = {
  start: 1,
  end: 5,
  [Symbol.iterator]() {
    let current = this.start;
    return {
      next: () => ({
        value: current,
        done: current++ > this.end,
      }),
    };
  },
};

for (const n of range) {
  console.log(n);             // 1 2 3 4 5
}
```

### Symbol.toPrimitive（类型转换）

```js
const price = {
  value: 100,
  currency: 'CNY',
  [Symbol.toPrimitive](hint) {
    if (hint === 'number') return this.value;
    if (hint === 'string') return `${this.value}${this.currency}`;
    return this.value;
  },
};

+price;                        // 100（number hint）
String(price);                 // "100CNY"（string hint）
price + '';                    // "100"（default hint）
```

### Symbol.toStringTag（自定义 toString 标签）

```js
class Matrix {
  get [Symbol.toStringTag]() {
    return 'Matrix';
  }
}

Object.prototype.toString.call(new Matrix()); // "[object Matrix]"
```

### Symbol.hasInstance（自定义 instanceof）

```js
class Positive {
  static [Symbol.hasInstance](num) {
    return typeof num === 'number' && num > 0;
  }
}

1 instanceof Positive;          // true
-1 instanceof Positive;         // false
```

### Symbol.species（衍生对象构造函数）

```js
class MyArray extends Array {
  static get [Symbol.species]() { return Array; }
}

const arr = new MyArray(1, 2, 3);
const mapped = arr.map(x => x * 2);
mapped instanceof MyArray;     // false（使用 Array 而非 MyArray）
mapped instanceof Array;       // true
```

### 完整的内置 Symbol 列表

| Symbol | 作用 | 覆盖的方法 |
|--------|------|-----------|
| `Symbol.iterator` | 定义对象的默认迭代器 | `for...of`、`...` 展开、`Array.from` |
| `Symbol.asyncIterator` | 定义异步迭代器 | `for await...of` |
| `Symbol.toPrimitive` | 定义类型转换行为 | 隐式转换（`+`、`String()`、`Number()`） |
| `Symbol.toStringTag` | 定义 `Object.prototype.toString` 的描述 | `Object.prototype.toString` |
| `Symbol.hasInstance` | 定义 `instanceof` 的行为 | `instanceof` |
| `Symbol.species` | 定义衍生对象的构造函数 | `map`、`filter`、`slice` 等 |
| `Symbol.isConcatSpreadable` | 定义 `Array.prototype.concat` 是否展开 | `concat` |
| `Symbol.match` / `Symbol.replace` / `Symbol.search` / `Symbol.split` | 定义字符串匹配行为 | `String.prototype.match` / `replace` / `search` / `split` |
| `Symbol.unscopables` | 定义 `with` 语句中排除的属性 | `with` |

## 六、实际应用场景

### 场景一：定义枚举常量

```js
const Colors = {
  RED: Symbol('red'),
  GREEN: Symbol('green'),
  BLUE: Symbol('blue'),
};

function getColorName(color) {
  switch (color) {
    case Colors.RED: return '红色';
    case Colors.GREEN: return '绿色';
    case Colors.BLUE: return '蓝色';
    default: return '未知';
  }
}
```

相比字符串枚举，Symbol 枚举不会被外部值意外匹配：

```js
getColorName('red');        // 返回"未知"（不会错误匹配）
getColorName(Colors.RED);   // 返回"红色"
```

### 场景二：私有属性（约定式）

Symbol 属性不会被常规方法遍历，适合存放内部状态：

```js
const _items = Symbol('items');
const _max = Symbol('maxSize');

class Stack {
  constructor(maxSize = Infinity) {
    this[_items] = [];
    this[_max] = maxSize;
  }

  push(item) {
    if (this[_items].length >= this[_max]) {
      throw new Error('Stack overflow');
    }
    this[_items].push(item);
  }

  pop() {
    return this[_items].pop();
  }

  get size() {
    return this[_items].length;
  }
}

const stack = new Stack(5);
stack.push(1);
Object.keys(stack);                // []（看不到 _items 和 _max）
Object.getOwnPropertySymbols(stack); // 仍然可以获取，只是增加访问成本
```

### 场景三：防止属性覆盖

在多人协作的代码库或第三方库的扩展中，Symbol 可以避免属性名冲突：

```js
// 第三方库的扩展点
const PLUGIN_HOOK = Symbol('hook');
const instance = {};

// 插件 A
instance[PLUGIN_HOOK] = () => console.log('Plugin A');

// 插件 B（不会覆盖）
instance[PLUGIN_HOOK] = () => console.log('Plugin B'); // 可以独立使用不同的 Symbol
```

### 场景四：定义类的内部行为（配合内置 Symbol）

```js
class Vector2D {
  constructor(x, y) {
    this.x = x;
    this.y = y;
  }

  // 使 Vector 可迭代
  *[Symbol.iterator]() {
    yield this.x;
    yield this.y;
  }

  // 控制类型转换
  [Symbol.toPrimitive](hint) {
    if (hint === 'string') return `(${this.x}, ${this.y})`;
    return Math.sqrt(this.x ** 2 + this.y ** 2);
  }

  // 自定义 toString 标签
  get [Symbol.toStringTag]() {
    return 'Vector2D';
  }
}

const v = new Vector2D(3, 4);
[...v];                        // [3, 4]
String(v);                     // "(3, 4)"
+v;                            // 5
Object.prototype.toString.call(v); // "[object Vector2D]"
```

## 七、Symbol 与序列化

Symbol 在序列化时会丢失：

```js
const key = Symbol('key');
const obj = { [key]: 'secret', name: 'Alice' };

JSON.stringify(obj);            // '{"name":"Alice"}'（Symbol 属性丢失）

// 解决方案：序列化时手动处理
JSON.stringify(obj, (key, value) => {
  if (typeof key === 'symbol') return undefined;
  return value;
});
```

### Symbol 的反序列化

```js
const KEY = Symbol.for('app.key');
const data = JSON.parse(rawJSON);

// 使用全局 Symbol 恢复
data[KEY] = decode(data.value);
```

## 八、常见问题

### Q1: Symbol 为什么不能 new

因为 Symbol 是原始类型（primitive type），不是对象。如果需要包装对象，使用 `Object(sym)`。

### Q2: Symbol 属性是真正的私有吗

不是。`Object.getOwnPropertySymbols()` 和 `Reflect.ownKeys()` 仍能获取。Symbol 只是"约定式私有"——它不会出现在常规遍历中，增加了意外访问的成本。真正的私有需要用 `#` 私有字段（ES2022）或 Closure。

### Q3: Symbol.for 的 key 冲突怎么办

使用带命名空间的前缀：

```js
Symbol.for('myapp.user.id');     // ✅ 项目前缀
Symbol.for('lodash.internal');   // ✅ 库前缀
```

### Q4: Symbol 可以作为 React key 吗

```js
// ❌ Symbol 不能作为 React key
items.map(item => <div key={Symbol()} />);
// Warning: Keys must be strings or numbers

// ✅ 使用字符串或数字
items.map((item, i) => <div key={item.id || i} />);
```

## 九、推荐学习路径

1. 掌握 Symbol 的创建和唯一性
2. 理解 Symbol 属性在遍历中的行为（`Object.keys` vs `getOwnPropertySymbols`）
3. 区分 `Symbol()` 和 `Symbol.for()` 的用途
4. 掌握常用内置 Symbol：`iterator`、`toPrimitive`、`toStringTag`
5. 在实际场景中实践：枚举、私有属性、元编程
