---
title: TypeScript 泛型（Generics）
categories: 
- TypeScript
tags:
- TypeScript
- 泛型
- Generics
---

## 一、为什么需要泛型

```ts
// 不用泛型：只能针对单一类型
function identity(arg: number): number { return arg; }

// 或用 any 丢失类型信息
function identity(arg: any): any { return arg; }

// 用泛型：类型被保留，传入什么类型就返回什么类型
function identity<T>(arg: T): T {
  return arg;
}

const num = identity(42);       // type: 42（字面量类型）
const str = identity('hello');  // type: "hello"
```

泛型的核心价值：**在保证类型安全的前提下，编写可复用的组件**。不指定具体类型，而是由调用者传入类型参数。

## 二、泛型函数

```ts
// 多个类型参数
function swap<T, U>(pair: [T, U]): [U, T] {
  return [pair[1], pair[0]];
}

const swapped = swap([1, 'hello']);  // type: [string, number]

// 数组泛型
function first<T>(arr: T[]): T | undefined {
  return arr[0];
}

// 箭头函数泛型（JSX 中需注意语法）
const last = <T,>(arr: T[]): T | undefined => arr[arr.length - 1];
```

## 三、泛型接口与泛型类型

```ts
// 泛型接口
interface ResponseData<T> {
  code: number;
  message: string;
  data: T;
}

type User = { id: number; name: string };

const res: ResponseData<User> = {
  code: 200,
  message: 'ok',
  data: { id: 1, name: 'Alice' },
};

// 泛型类型别名
type Nullable<T> = T | null;
type Pair<T, U> = [T, U];
```

## 四、泛型约束（extends）

```ts
// ❌ 编译错误：.length 不一定存在
function logLength<T>(arg: T): T {
  console.log(arg.length);
  return arg;
}

// ✅ 约束 T 必须具有 length 属性
interface HasLength {
  length: number;
}

function logLength<T extends HasLength>(arg: T): T {
  console.log(arg.length);
  return arg;
}

logLength('hello');        // 5（string 有 length）
logLength([1, 2, 3]);     // 3（array 有 length）
logLength({ length: 10 });// 10（手动满足）

// keyof 约束
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}

const user = { name: 'Alice', age: 25 };
getProperty(user, 'name');    // type: string ✅
getProperty(user, 'age');     // type: number ✅
getProperty(user, 'email');   // ❌ 编译错误：email 不在 keyof 中
```

## 五、条件类型

```ts
// 基础条件类型
type IsString<T> = T extends string ? true : false;

type A = IsString<'hello'>;   // true
type B = IsString<number>;    // false

// 条件类型 + 泛型
type ExtractArrayType<T> = T extends (infer U)[] ? U : never;

type C = ExtractArrayType<string[]>;   // string
type D = ExtractArrayType<number[]>;   // number
type E = ExtractArrayType<number>;     // never
```

### infer 关键字

`infer` 用于在条件类型中**声明一个待推断的类型变量**：

```ts
// 提取 Promise 返回值类型
type Unwrap<T> = T extends Promise<infer R> ? R : T;

type F = Unwrap<Promise<string>>;  // string
type G = Unwrap<number>;           // number（非 Promise，直接返回 T）

// 提取函数返回值类型（类似 ReturnType）
type MyReturnType<T> = T extends (...args: any[]) => infer R ? R : never;

// 提取函数参数类型
type MyParameters<T> = T extends (...args: infer P) => any ? P : never;

// 递归提取嵌套 Promise
type DeepUnwrap<T> = T extends Promise<infer R> ? DeepUnwrap<R> : T;
type H = DeepUnwrap<Promise<Promise<Promise<string>>>>;  // string
```

### 分布式条件类型

当条件类型作用于**裸类型参数**时，会自动分发到联合类型的每个成员：

```ts
type ToArray<T> = T extends any ? T[] : never;

type I = ToArray<string | number>;
// 等价于 string[] | number[]（分发给每个成员）
// 不是 (string | number)[]！
```

如何阻止分发？用元组包裹：

```ts
type ToArrayNonDist<T> = [T] extends [any] ? T[] : never;
type J = ToArrayNonDist<string | number>;  // (string | number)[]
```

## 六、实用泛型模式

### 工厂函数

```ts
function createInstance<T>(ctor: new (...args: any[]) => T, ...args: any[]): T {
  return new ctor(...args);
}

class User {
  constructor(public name: string) {}
}

const u = createInstance(User, 'Alice');  // type: User
```

### 类型安全的 EventEmitter

```ts
interface EventMap {
  click: { x: number; y: number };
  focus: { element: HTMLElement };
  keydown: { key: string };
}

class TypedEmitter {
  private handlers = new Map<keyof EventMap, Function[]>();

  on<K extends keyof EventMap>(event: K, handler: (data: EventMap[K]) => void) {
    if (!this.handlers.has(event)) this.handlers.set(event, []);
    this.handlers.get(event)!.push(handler);
  }

  emit<K extends keyof EventMap>(event: K, data: EventMap[K]) {
    this.handlers.get(event)?.forEach(h => h(data));
  }
}

const emitter = new TypedEmitter();
emitter.on('click', (data) => console.log(data.x, data.y));  // type-safe
emitter.emit('click', { x: 10, y: 20 });
```

### 链式调用类型

```ts
class QueryBuilder<T extends Record<string, any>> {
  private conditions: string[] = [];

  where<K extends keyof T>(key: K, value: T[K]): this {
    this.conditions.push(`${String(key)} = ${value}`);
    return this;
  }

  build(): string {
    return this.conditions.join(' AND ');
  }
}

interface User {
  name: string;
  age: number;
}

const query = new QueryBuilder<User>()
  .where('name', 'Alice')   // ✅ type-safe
  .where('age', 25)         // ✅
  .build();
```

## 七、面试题

### Q1: `Extract` 和 `Exclude` 的实现

```ts
// Exclude<T, U>：从 T 中排除可赋值给 U 的类型
type MyExclude<T, U> = T extends U ? never : T;
type K = MyExclude<'a' | 'b' | 'c', 'a' | 'b'>;  // 'c'

// Extract<T, U>：从 T 中提取可赋值给 U 的类型
type MyExtract<T, U> = T extends U ? T : never;
type L = MyExtract<'a' | 'b' | 'c', 'a' | 'b'>;   // 'a' | 'b'
```

### Q2: 实现 `ReturnType`

```ts
type MyReturnType<T extends (...args: any) => any>
  = T extends (...args: any) => infer R ? R : never;

type M = MyReturnType<() => string>;  // string
```

### Q3: 实现 `Pick` 和 `Omit`

```ts
type MyPick<T, K extends keyof T> = {
  [P in K]: T[P];
};

type MyOmit<T, K extends keyof T> = {
  [P in Exclude<keyof T, K>]: T[P];
};
```

### Q4: 如何实现深度 Partial？

```ts
type DeepPartial<T> = {
  [P in keyof T]?: T[P] extends object ? DeepPartial<T[P]> : T[P];
};

interface Config {
  nested: {
    value: string;
    flag: boolean;
  };
}

type PartialConfig = DeepPartial<Config>;
// { nested?: { value?: string; flag?: boolean } }
```

### Q5: `infer` 实现数组元素类型

```ts
type ArrayItem<T> = T extends (infer U)[] ? U : never;

type N = ArrayItem<string[]>;     // string
type O = ArrayItem<number[][]>;   // number[]（只解一层）

// 深层解包
type DeepArrayItem<T> = T extends (infer U)[] ? DeepArrayItem<U> : T;
type P = DeepArrayItem<number[][][]>;  // number
```
