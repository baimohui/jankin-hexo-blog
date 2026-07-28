---
title: TypeScript 实用类型（Utility Types）
categories: 
- TypeScript
tags:
- TypeScript
- Utility Types
- Pick
- Omit
- Record
- Partial
---

## 一、操作对象类型

### Partial<T>

将 T 的所有属性变为可选。

```ts
interface User {
  name: string;
  age: number;
  email: string;
}

// 实现
type MyPartial<T> = { [P in keyof T]?: T[P] };

type PartialUser = Partial<User>;
// { name?: string; age?: number; email?: string }

// 应用：更新操作
function updateUser(id: number, fields: Partial<User>) {
  // 只传需要修改的字段
}
updateUser(1, { name: 'Alice' });  // ✅ 只需传 name
```

### Required<T>

将 T 的所有属性变为必选（去掉 `?`）。

```ts
// 实现
type MyRequired<T> = { [P in keyof T]-?: T[P] };

interface Config {
  debug?: boolean;
  logLevel?: string;
}

type StrictConfig = Required<Config>;
// { debug: boolean; logLevel: string }
```

### Readonly<T>

将 T 的所有属性变为只读。

```ts
// 实现
type MyReadonly<T> = { readonly [P in keyof T]: T[P] };

type ReadonlyUser = Readonly<User>;
// { readonly name: string; readonly age: number; readonly email: string }
```

### Pick<T, K>

从 T 中选取一组属性 K 组成新类型。

```ts
// 实现
type MyPick<T, K extends keyof T> = { [P in K]: T[P] };

type UserName = Pick<User, 'name' | 'email'>;
// { name: string; email: string }

// 应用：接口返回的敏感字段过滤
type PublicUser = Pick<User, 'name'>;
```

### Omit<T, K>

从 T 中排除一组属性 K，保留剩余属性。

```ts
// 实现
type MyOmit<T, K extends keyof T> = { [P in Exclude<keyof T, K>]: T[P] };

type UserWithoutPassword = Omit<User, 'password' | 'secret'>;
// { name: string; age: number; email: string }
```

### Record<K, T>

以 K 为键、T 为值构造一个对象类型。

```ts
// 实现
type MyRecord<K extends keyof any, T> = { [P in K]: T };

type Page = 'home' | 'about' | 'contact';

type PageInfo = Record<Page, { title: string; url: string }>;
// {
//   home: { title: string; url: string };
//   about: { title: string; url: string };
//   contact: { title: string; url: string };
// }

const pages: PageInfo = {
  home: { title: '首页', url: '/' },
  about: { title: '关于', url: '/about' },
  contact: { title: '联系', url: '/contact' },
};

// 应用：枚举映射
const StatusMap: Record<number, string> = {
  0: 'pending',
  1: 'active',
  2: 'disabled',
};
```

## 二、操作联合类型

### Exclude<T, U>

从 T 中排除可赋值给 U 的类型。

```ts
// 实现
type MyExclude<T, U> = T extends U ? never : T;

type T0 = Exclude<'a' | 'b' | 'c', 'a'>;            // 'b' | 'c'
type T1 = Exclude<string | number | (() => void), Function>;  // string | number
```

### Extract<T, U>

从 T 中提取可赋值给 U 的类型。

```ts
// 实现
type MyExtract<T, U> = T extends U ? T : never;

type T2 = Extract<'a' | 'b' | 'c', 'a' | 'f'>;     // 'a'
type T3 = Extract<string | number, number>;          // number
```

### NonNullable<T>

从 T 中排除 `null` 和 `undefined`。

```ts
// 实现
type MyNonNullable<T> = T extends null | undefined ? never : T;

type T4 = NonNullable<string | number | null | undefined>;  // string | number
```

## 三、操作函数类型

### ReturnType<T>

获取函数类型的返回值类型。

```ts
// 实现
type MyReturnType<T extends (...args: any) => any>
  = T extends (...args: any) => infer R ? R : never;

type Fn = () => { a: number; b: string };
type T5 = ReturnType<Fn>;    // { a: number; b: string }

// 实际应用：获取 API 函数的返回值类型
function fetchUser() {
  return Promise.resolve({ id: 1, name: 'Alice' });
}

type FetchUserReturn = ReturnType<typeof fetchUser>;
// Promise<{ id: number; name: string }>

// 结合 Awaited 获取真正的返回值
type UserData = Awaited<ReturnType<typeof fetchUser>>;
// { id: number; name: string }
```

### Parameters<T>

获取函数类型的参数类型（元组）。

```ts
// 实现
type MyParameters<T extends (...args: any) => any>
  = T extends (...args: infer P) => any ? P : never;

type Fn2 = (name: string, age: number) => void;
type T6 = Parameters<Fn2>;   // [name: string, age: number]

type FirstParam<T> = T extends (arg: infer P, ...rest: any) => any ? P : never;
type T7 = FirstParam<Fn2>;   // string
```

### ConstructorParameters<T>

获取构造函数类型的参数类型。

```ts
type T8 = ConstructorParameters<typeof Date>;
// [value: string | number | Date]（取决于重载）

// 应用：工厂函数类型安全
function factory<T>(ctor: new (...args: any[]) => T, ...args: ConstructorParameters<typeof ctor>) {
  return new ctor(...args);
}
```

### InstanceType<T>

获取构造函数类型的实例类型。

```ts
class MyClass {
  constructor(public name: string) {}
}

type T9 = InstanceType<typeof MyClass>;  // MyClass
```

## 四、操作字符串类型

### Uppercase / Lowercase / Capitalize / Uncapitalize

```ts
type T10 = Uppercase<'hello'>;        // "HELLO"
type T11 = Lowercase<'HELLO'>;        // "hello"
type T12 = Capitalize<'hello'>;       // "Hello"
type T13 = Uncapitalize<'Hello'>;     // "hello"

// 应用：统一事件名格式
type EventName = 'click' | 'mouseenter' | 'scroll';
type Handlers = `on${Capitalize<EventName>}`;  // "onClick" | "onMouseenter" | "OnScroll"
// 修正：Uncapitalize 包裹
type CorrectHandler = `on${Capitalize<EventName>}`;
// 实际上用 Uncapitalize 保持首字母大写后的结果
```

## 五、高级实用类型

### Awaited<T

递归解包 Promise 类型（TS 4.5+）。

```ts
type T14 = Awaited<Promise<string>>;              // string
type T15 = Awaited<Promise<Promise<number>>>;     // number
type T16 = Awaited<Promise<string | number>>;     // string | number
```

### ThisParameterType / OmitThisParameter

```ts
function toHex(this: Number) {
  return this.toString(16);
}

type T17 = ThisParameterType<typeof toHex>;  // Number
type T18 = OmitThisParameter<typeof toHex>;  // () => string
```

## 六、面试题

### Q1: 如何让 Pick 支持嵌套路径？

```ts
type NestedPick<T, K extends string> =
  K extends `${infer Key}.${infer Rest}`
    ? Key extends keyof T
      ? { [P in Key]: NestedPick<T[Key], Rest> }
      : never
    : K extends keyof T
      ? { [P in K]: T[K] }
      : never;

type UserDeep = {
  profile: { name: string; age: number };
  settings: { theme: string };
};

type PickedUser = NestedPick<UserDeep, 'profile.name'>;
// { profile: { name: string } }
```

### Q2: 实现 `DeepReadonly`

```ts
type DeepReadonly<T> = {
  readonly [P in keyof T]: T[P] extends object
    ? T[P] extends Function
      ? T[P]
      : DeepReadonly<T[P]>
    : T[P];
};
```

### Q3: 从联合类型中排除函数类型

```ts
type ExcludeFunction<T> = T extends (...args: any) => any ? never : T;

type Mixed = string | number | (() => void) | (() => string);
type WithoutFn = ExcludeFunction<Mixed>;  // string | number
```

### Q4: `Pick` 和 `Omit` 的区别在面试中如何回答

```text
Pick<T, K>：从 T 中选取 K 属性集合组成新类型，K 必须为 T 的 key。
Omit<T, K>：从 T 中排除 K 属性集合，保留剩余属性。

两者互补：Omit<T, K> = Pick<T, Exclude<keyof T, K>>。

适用场景：
  Pick → 接口响应中只暴露部分字段给前端
  Omit → 排除敏感字段（密码、token）
```

### Q5: 实现一个类型安全的状态机

```ts
type StateMachine<S extends string, E extends string> = {
  initial: S;
  states: Record<S, {
    on: Partial<Record<E, S>>;
  }>;
};

const machine: StateMachine<'idle' | 'loading' | 'success', 'FETCH' | 'RETRY'> = {
  initial: 'idle',
  states: {
    idle: { on: { FETCH: 'loading' } },
    loading: { on: { FETCH: 'loading', RETRY: 'loading' } },
    success: { on: { FETCH: 'loading' } },
  },
};
```
