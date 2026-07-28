---
title: IndexedDB 详解
categories: 
- 浏览器
tags:
- 浏览器
- 存储
- IndexedDB
- 数据库
---

## 一、IndexedDB 是什么，为什么需要它

IndexedDB 是浏览器提供的一个**客户端 NoSQL 数据库**，可以存储大量结构化数据，支持索引、事务和游标查询。它是目前浏览器端功能最强大的存储方案。<!--more-->

### 浏览器存储方案对比

| 特性 | Cookie | localStorage | sessionStorage | IndexedDB |
|------|--------|-------------|----------------|-----------|
| 容量 | 4KB | 5-10MB | 5-10MB | 通常 ≥ 250MB |
| 数据类型 | 字符串 | 字符串 | 字符串 | **结构化数据、Blob、文件** |
| 同步/异步 | 同步 | 同步 | 同步 | **异步**（不阻塞主线程） |
| 索引 | 无 | 无 | 无 | **支持**（按字段高速查询） |
| 事务 | 无 | 无 | 无 | **支持**（ACID） |
| 查询能力 | 键值对 | 键值对 | 键值对 | **游标、范围查询、排序** |

### 何时使用 IndexedDB

- 需要存储大量数据（如离线地图瓦片、用户生成的内容）
- 需要按多个字段查询和排序（如邮件列表按时间+发件人过滤）
- 需要离线存储完整的应用数据（配合 Service Worker 实现 PWA）
- 需要存储二进制数据（图片、音频文件）
- 需要使用事务保证数据一致性

## 二、核心概念

### 数据库（Database）

一个域名可以创建多个数据库。每个数据库有唯一的名称，且版本号递增。

### 对象存储空间（Object Store）

相当于关系数据库中的**表**。每个 Object Store 存储一类对象，对象是键值对形式：

```
ObjectStore: "users"
  { id: 1, name: "Alice", email: "alice@example.com" }
  { id: 2, name: "Bob",   email: "bob@example.com" }
```

### 索引（Index）

为 Object Store 中的某个字段建立索引，以实现按该字段快速查询和排序。

### 事务（Transaction）

所有读写操作必须在事务中执行。事务具有 ACID 特性：

- **Atomicity（原子性）**：事务内的操作要么全部成功，要么全部回滚
- **Consistency（一致性）**：事务前后数据状态保持一致
- **Isolation（隔离性）**：事务之间互不干扰
- **Durability（持久性）**：事务提交后数据不会丢失

### 游标（Cursor）

用于遍历 Object Store 或 Index 中的多条记录，可以控制遍历方向和范围。

## 三、基础操作

### 打开数据库

```js
const request = indexedDB.open('MyApp', 1);   // 数据库名，版本号

request.onupgradeneeded = (event) => {
  // 数据库首次创建或版本号升级时触发
  const db = event.target.result;
  // 在此创建 Object Store 和 Index
};

request.onsuccess = (event) => {
  const db = event.target.result;
  console.log('数据库打开成功', db);
};

request.onerror = (event) => {
  console.error('数据库打开失败', event.target.error);
};
```

`indexedDB.open` 返回一个 IDBRequest 对象，通过事件监听获取结果：

| 事件 | 触发时机 |
|------|---------|
| `upgradeneeded` | 数据库不存在，或版本号大于当前版本 |
| `success` | 数据库打开成功 |
| `error` | 数据库打开失败 |
| `blocked` | 当前版本被其他页面占用，无法升级 |

### 创建 Object Store 和 Index

Object Store 和 Index **只能在 `upgradeneeded` 事件中创建**。

```js
request.onupgradeneeded = (event) => {
  const db = event.target.result;

  // 创建 Object Store，指定 keyPath（主键字段）
  const store = db.createObjectStore('users', {
    keyPath: 'id',
    autoIncrement: true,   // 是否自动递增
  });

  // 创建索引：按 name 查询，允许重复值
  store.createIndex('name', 'name', { unique: false });

  // 创建索引：按 email 查询，不允许重复
  store.createIndex('email', 'email', { unique: true });

  // 索引可以按多个字段组合
  store.createIndex('age', 'age', { unique: false });
};
```

### 新增数据

```js
const transaction = db.transaction(['users'], 'readwrite');  // 开启读写事务
const store = transaction.objectStore('users');

// 单条新增
const addRequest = store.add({ name: 'Alice', email: 'alice@example.com', age: 25 });

addRequest.onsuccess = () => console.log('新增成功');
addRequest.onerror = (event) => {
  if (event.target.error.name === 'ConstraintError') {
    console.error('主键或唯一索引冲突');
  }
};
```

`add` 与 `put` 的区别：
- `add`：主键重复时报错（`ConstraintError`）
- `put`：主键重复时**更新**已有记录

### 读取数据

```js
// 按主键查询
const getRequest = store.get(1);
getRequest.onsuccess = (event) => {
  console.log(event.target.result);   // { id: 1, name: "Alice", ... }
};

// 按索引查询
const nameIndex = store.index('name');
const indexRequest = nameIndex.get('Alice');
indexRequest.onsuccess = (event) => {
  console.log(event.target.result);
};

// 查询所有记录
const getAllRequest = store.getAll();
getAllRequest.onsuccess = (event) => {
  console.log(event.target.result);   // 数组
};
```

### 更新数据

```js
const putRequest = store.put({ id: 1, name: 'Alice', email: 'alice@new.com', age: 26 });
// put 会覆盖已存在的记录（根据 keyPath 匹配）
```

### 删除数据

```js
// 按主键删除
store.delete(1);

// 按范围删除
store.delete(IDBKeyRange.bound(10, 20));   // 删除主键 10~20 的记录
```

### 遍历数据（游标）

```js
const cursorRequest = store.openCursor();

cursorRequest.onsuccess = (event) => {
  const cursor = event.target.result;
  if (cursor) {
    console.log(cursor.key, cursor.value);   // 当前记录
    cursor.continue();                        // 移动到下一条
  } else {
    console.log('遍历完成');
  }
};
```

#### 游标方向

```js
// 默认：next（升序）
store.openCursor();                              // 升序
store.openCursor(null, 'next');                  // 升序（显式）
store.openCursor(null, 'prev');                  // 降序
store.openCursor(null, 'nextunique');            // 升序，跳过重复索引值
store.openCursor(null, 'prevunique');            // 降序，跳过重复索引值
```

#### 游标范围查询

```js
// 只取需要的范围，提高性能
const range = IDBKeyRange.bound(100, 200);       // 100 ≤ key ≤ 200
store.openCursor(range);

IDBKeyRange.lowerBound(100);                     // key ≥ 100
IDBKeyRange.upperBound(200);                     // key ≤ 200
IDBKeyRange.lowerBound(100, true);               // key > 100（排除边界）
IDBKeyRange.only(50);                            // key = 50
```

#### 使用索引游标排序

```js
const ageIndex = store.index('age');
ageIndex.openCursor(null, 'prev');               // 按 age 降序遍历
```

## 四、错误处理与事务

### 事务生命周期

```js
const tx = db.transaction(['users'], 'readwrite');

tx.oncomplete = () => console.log('事务完成');
tx.onerror = (event) => {
  console.error('事务失败', event.target.error);
  // 只有监听了 onerror 并调用 preventDefault，才能阻止事务回滚所有操作
  event.preventDefault();
};
tx.onabort = () => console.log('事务被中止');
```

**重要**：事务在事件循环中自动提交。一旦发起请求，事务会进入 `active` 状态；当所有请求完成且没有新请求排队时，事务自动提交。因此，**不要在事务中进行异步操作（如 `fetch`）**。

```js
// ❌ 错误：在事务中进行异步操作
const tx = db.transaction(['users'], 'readwrite');
fetch('/api/data').then(res => {
  tx.objectStore('users').add(res.data);   // 此时事务可能已提交
});

// ✅ 正确：先获取数据再开启事务
fetch('/api/data').then(res => {
  const tx = db.transaction(['users'], 'readwrite');
  tx.objectStore('users').add(res.data);
});
```

### 手动中止事务

```js
tx.abort();   // 显式中止，所有修改回滚
```

## 五、版本升级

IndexedDB 的版本号用于管理数据库结构变更。升级发生在 `onupgradeneeded` 事件中。

```js
// 将版本从 1 升级到 2
const request = indexedDB.open('MyApp', 2);

request.onupgradeneeded = (event) => {
  const db = event.target.result;
  const oldVersion = event.oldVersion;
  const newVersion = event.newVersion;

  if (oldVersion < 1) {
    // 首次创建，建表
    db.createObjectStore('users', { keyPath: 'id' });
  }

  if (oldVersion < 2) {
    // 版本 2：新增 orders 表
    const orderStore = db.createObjectStore('orders', { keyPath: 'id', autoIncrement: true });
    orderStore.createIndex('userId', 'userId', { unique: false });
  }
};
```

**不可逆操作**：`createObjectStore` 和 `deleteObjectStore` 是破坏性操作。升级前如果担心数据丢失，需要手动迁移数据：

```js
request.onupgradeneeded = (event) => {
  const db = event.target.result;

  // 读取旧数据
  const oldStore = event.target.transaction.objectStore('users');
  const allUsers = oldStore.getAll();

  allUsers.onsuccess = () => {
    // 删除旧表
    db.deleteObjectStore('users');

    // 用新结构重建
    const newStore = db.createObjectStore('users', { keyPath: 'id' });
    newStore.createIndex('name', 'name', { unique: false });

    // 写回数据
    allUsers.result.forEach(user => newStore.add(user));
  };
};
```

## 六、代码封装实践

`indexedDB` 的 API 基于事件回调，写起来比较繁琐。在实际项目中，通常用 Promise 封装。

### 封装 Database 类

```js
class DB {
  constructor(dbName, version, upgradeCallback) {
    return new Promise((resolve, reject) => {
      const request = indexedDB.open(dbName, version);

      request.onupgradeneeded = (event) => {
        upgradeCallback?.(event.target.result, event);
      };

      request.onsuccess = (event) => {
        this.db = event.target.result;
        resolve(this);
      };

      request.onerror = (event) => {
        reject(event.target.error);
      };
    });
  }

  // 通用事务方法
  async perform(storeName, mode, callback) {
    const tx = this.db.transaction(storeName, mode);
    const store = tx.objectStore(storeName);

    return new Promise((resolve, reject) => {
      callback(store, resolve, reject);

      tx.oncomplete = () => resolve;
      tx.onerror = (event) => reject(event.target.error);
    });
  }

  // 新增
  add(storeName, data) {
    return this.perform(storeName, 'readwrite', (store, resolve, reject) => {
      const request = store.add(data);
      request.onsuccess = () => resolve(request.result);
      request.onerror = () => reject(request.error);
    });
  }

  // 按主键查询
  get(storeName, key) {
    return this.perform(storeName, 'readonly', (store, resolve, reject) => {
      const request = store.get(key);
      request.onsuccess = () => resolve(request.result);
      request.onerror = () => reject(request.error);
    });
  }

  // 查询全部
  getAll(storeName) {
    return this.perform(storeName, 'readonly', (store, resolve, reject) => {
      const request = store.getAll();
      request.onsuccess = () => resolve(request.result);
      request.onerror = () => reject(request.error);
    });
  }

  // 删除
  delete(storeName, key) {
    return this.perform(storeName, 'readwrite', (store, resolve, reject) => {
      const request = store.delete(key);
      request.onsuccess = () => resolve();
      request.onerror = () => reject(request.error);
    });
  }
}
```

### 使用封装

```js
const db = await new DB('MyApp', 1, (db) => {
  db.createObjectStore('users', { keyPath: 'id', autoIncrement: true });
});

await db.add('users', { name: 'Alice', age: 25 });
const user = await db.get('users', 1);
const all = await db.getAll('users');
```

### 使用第三方库

如果不想自己封装，社区有成熟的库：

```bash
npm install idb
```

```js
import { openDB } from 'idb';

const db = await openDB('MyApp', 1, {
  upgrade(db) {
    db.createObjectStore('users', { keyPath: 'id' });
  },
});

// 类同步的 API
await db.add('users', { name: 'Alice' });
const user = await db.get('users', 1);
```

## 七、常见场景

### 场景一：离线缓存接口数据

```js
class CacheService {
  constructor() {
    this.dbPromise = openDB('ApiCache', 1, {
      upgrade(db) {
        const store = db.createObjectStore('responses', { keyPath: 'url' });
        store.createIndex('timestamp', 'timestamp', { unique: false });
      },
    });
  }

  async get(url) {
    const db = await this.dbPromise;
    const cached = await db.get('responses', url);

    if (cached && Date.now() - cached.timestamp < 5 * 60 * 1000) {
      return cached.data;   // 5 分钟内用缓存
    }

    const response = await fetch(url);
    const data = await response.json();

    await db.put('responses', { url, data, timestamp: Date.now() });
    return data;
  }
}

const cache = new CacheService();
const data = await cache.get('/api/todos');
```

### 场景二：图片文件存储

IndexedDB 可以直接存储 `Blob` 对象，适合缓存图片。

```js
async function cacheImage(url) {
  const db = await openDB('ImageCache', 1, {
    upgrade(db) {
      db.createObjectStore('images', { keyPath: 'url' });
    },
  });

  // 查缓存
  const cached = await db.get('images', url);
  if (cached) return URL.createObjectURL(cached.blob);

  // 下载并缓存
  const response = await fetch(url);
  const blob = await response.blob();
  await db.put('images', { url, blob });
  return URL.createObjectURL(blob);
}
```

### 场景三：全文搜索（配合索引）

```js
const db = await openDB('Notes', 1, {
  upgrade(db) {
    const store = db.createObjectStore('notes', { keyPath: 'id', autoIncrement: true });
    store.createIndex('title', 'title', { unique: false });
    store.createIndex('createdAt', 'createdAt', { unique: false });
  },
});

// 插入笔记
await db.add('notes', { title: '学习 IndexedDB', content: '...', createdAt: Date.now() });

// 按标题前缀搜索
const index = db.transaction('notes').store.index('title');
const range = IDBKeyRange.bound('学习', '学习\uFFFF'); // 前缀匹配
const results = await index.getAll(range);

// 按时间降序
const cursor = await index.openCursor(null, 'prev');
while (cursor) {
  console.log(cursor.value);
  cursor.continue();
}
```

## 八、注意事项

### 1. 异步与非阻塞

IndexedDB 的所有操作都是异步的，不会阻塞主线程。但这意味着：

- 不要用同步思维写循环嵌套
- 事务会在微任务之后自动提交
- Promise 封装时需注意事务的 active 状态

### 2. 存储限制

- 每个域名下可创建多个数据库
- 单个数据库的大小通常没有硬性限制，但浏览器会提示用户授权（尤其是超过 50MB 时）
- 用户可以通过浏览器设置清除站点数据，IndexedDB 会被一起清除

### 3. 跨域限制

IndexedDB **遵循同源策略**。不同协议（http vs https）、不同端口、不同子域名之间不能互相访问。

### 4. 浏览器兼容性

所有现代浏览器（Chrome、Firefox、Safari 10+、Edge 12+）均支持 IndexedDB。IE 10 支持但实现有 bug，不再需要关心。

### 5. 调试

Chrome DevTools → Application → IndexedDB，可以查看和编辑数据库内容：

- 查看所有 Object Store 和数据
- 手动删除记录
- 清空整个数据库

## 九、常见问题

### Q1: IndexedDB 与 Web Storage 怎么选

简单轻量的键值存储用 `localStorage` ；需要存储大量结构化数据、需要索引和事务时用 IndexedDB。如待办事项列表用 localStorage 即可，邮件客户端用 IndexedDB。

### Q2: 页面关闭后 IndexedDB 数据还在吗

数据持久保存在磁盘上，关闭页面、关闭浏览器、重启电脑都不会丢失。只有用户手动清除站点数据时才会被删除。

### Q3: `IDBRequest` 的 `onsuccess` 和 `transaction.oncomplete` 有什么区别

`request.onsuccess` 表示单个操作完成，`transaction.oncomplete` 表示事务内所有操作都已完成。如果需要知道所有操作都完成后再执行逻辑，用后者。

### Q4: IndexedDB 可以被 Service Worker 访问吗

可以。在 Service Worker 中同样可以使用 `indexedDB.open`，这是离线缓存架构的关键——SW 负责拦截请求，IndexedDB 负责存储数据。

### Q5: `getAll` 会把所有数据加载到内存吗

是的，`getAll` 返回所有匹配记录到内存中。如果数据量极大（数十万条），应使用游标逐条处理以避免内存溢出。

## 十、推荐学习路径

1. 理解 IndexedDB 的定位：与 Cookie、localStorage、Cache API 的对比
2. 掌握增删改查和游标遍历（本文）
3. 封装 Promise 版本或使用 `idb` 库，消除回调嵌套
4. 结合 Service Worker 实现完整的 PWA 离线方案
5. 对大型数据集使用游标分页和索引优化，避免性能问题
