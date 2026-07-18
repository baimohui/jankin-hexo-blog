---
title: Mocha 单元测试详解
date: 2026-07-17 07:45:00
categories:
- 前端工程化
- 测试
tags:
- Mocha
- Chai
- Sinon
- 单元测试
- 前端工程化
---

## 一、为什么需要单元测试

单元测试是对代码中最小的可测试单元（通常是一个函数或一个方法）进行验证的自动化手段。

### 没有测试时的开发成本

- 修改一个工具函数后，需要手动打开页面反复点击验证
- 代码重构时完全依赖"感觉"来判断有没有改坏
- 代码 Review 只能看逻辑对不对，无法验证行为是否正确
- 新成员接手项目，不敢改任何代码

### 单元测试的价值

| 价值 | 说明 |
|------|------|
| **快速反馈** | 改完代码运行 `npm test`，几秒内知道有没有改坏 |
| **重构保护网** | 有测试覆盖的代码可以放心重构 |
| **活文档** | 测试代码就是最好的使用示例和使用说明 |
| **设计驱动** | 难以测试的设计往往意味着耦合过重，倒逼优化架构 |
| **质量门禁** | CI 流水线中拦截回归，防止有问题的代码合并 |

## 二、Mocha 生态概览

Mocha 是一个功能丰富的 JavaScript 测试框架，搭配 Chai（断言库）和 Sinon（Mock 库）组成前端测试的经典三件套。

### Mocha 与其他框架对比

| 特性 | Mocha + Chai + Sinon | Jest | Vitest |
|------|---------------------|------|--------|
| **断言库** | 需额外安装 Chai | 内置 | 内置 |
| **Mock/Stub** | 需额外安装 Sinon | 内置 | 内置 |
| **运行速度** | 一般 | 一般 | 快（基于 Vite） |
| **配置复杂度** | 需手动配置 | 零配置 | 零配置 |
| **浏览器测试** | 原生支持 | 需 jsdom | 需 jsdom |
| **社区生态** | 老牌成熟 | 最主流 | 新兴快速崛起 |
| **适用场景** | Node.js / 浏览器通用 | React 生态 | Vue / Vite 生态 |

**选择 Mocha 的场景**：
- 需要运行在浏览器环境中的测试（Mocha 原生支持浏览器中运行）
- 已有的老项目使用 Mocha，需要保持一致
- 希望自由组合断言库和 Mock 库，不受框架绑定
- 对 Node.js 模块进行测试

## 三、快速开始

```bash
# 安装 Mocha + Chai + Sinon
npm install --save-dev mocha chai sinon
```

```json
// package.json
{
  "scripts": {
    "test": "mocha 'src/**/*.test.js'"
  }
}
```

```js
// src/utils/math.js
export function add(a, b) { return a + b; }
export function multiply(a, b) { return a * b; }
```

```js
// src/utils/math.test.js
import { expect } from 'chai';
import { add, multiply } from './math.js';

describe('Math Utils', () => {
  it('should add two numbers', () => {
    expect(add(2, 3)).to.equal(5);
  });

  it('should multiply two numbers', () => {
    expect(multiply(3, 4)).to.equal(12);
  });
});
```

```bash
npm test
#   Math Utils
#     ✓ should add two numbers
#     ✓ should multiply two numbers
```

## 四、Mocha 核心 API

### describe / it

```js
describe('Calculator', () => {       // 测试套件，对测试用例分组
  it('should return 4 when adding 2+2', () => {  // 测试用例
    // ...
  });
});
```

describe 可以嵌套，用于建立层次化的测试结构：

```js
describe('User', () => {
  describe('#login()', () => {
    it('should succeed with valid credentials', () => {});
    it('should fail with invalid password', () => {});
  });

  describe('#logout()', () => {
    it('should clear session', () => {});
  });
});
```

### Hooks（生命周期）

```js
describe('Database', () => {
  before(() => {
    // 整个 describe 块执行一次（初始化数据库连接）
  });

  after(() => {
    // 整个 describe 块结束后执行一次（关闭连接）
  });

  beforeEach(() => {
    // 每个 it 之前执行（清空表、插入测试数据）
  });

  afterEach(() => {
    // 每个 it 之后执行（清理数据）
  });

  it('should insert a record', () => {});
  it('should query a record', () => {});
});
```

钩子的执行顺序：

```
before
  beforeEach
    it #1
  afterEach
  beforeEach
    it #2
  afterEach
after
```

### 独占与跳过

```js
describe.only('Only this suite', () => {   // 只运行这个套件
  it.only('only this test', () => {});     // 只运行这个用例
  it('will be skipped', () => {});
});

describe.skip('Skip this suite', () => {   // 跳过整个套件
  it('will not run', () => {});
});
```

## 五、断言（Chai）

Chai 提供三种断言风格：`assert`、`expect`、`should`。推荐使用 `expect`，可读性最好。

### expect 风格

```js
import { expect } from 'chai';

// 相等性
expect(1 + 1).to.equal(2);
expect({ a: 1 }).to.deep.equal({ a: 1 });    // 深比较
expect('hello').to.not.equal('world');

// 布尔值
expect(true).to.be.true;
expect(false).to.be.false;
expect(null).to.be.null;
expect(undefined).to.be.undefined;

// 类型检查
expect('str').to.be.a('string');
expect(42).to.be.a('number');
expect([]).to.be.an('array');

// 包含关系
expect([1, 2, 3]).to.include(2);
expect('hello world').to.include('world');
expect({ name: 'Alice' }).to.have.property('name');

// 长度
expect([1, 2, 3]).to.have.lengthOf(3);
expect('abc').to.have.lengthOf(3);

// 数值比较
expect(10).to.be.greaterThan(5);
expect(5).to.be.lessThan(10);
expect(5).to.be.at.least(5);

// 正则
expect('hello@example.com').to.match(/^[\w.-]+@[\w.-]+\.\w+$/);

// 异常
expect(() => JSON.parse('invalid')).to.throw(SyntaxError);
expect(() => divide(1, 0)).to.throw('除数不能为零');
```

### assert 风格

```js
import { assert } from 'chai';

assert.strictEqual(1 + 1, 2);
assert.deepEqual({ a: 1 }, { a: 1 });
assert.isArray([1, 2, 3]);
assert.include([1, 2, 3], 2);
assert.lengthOf('abc', 3);
assert.throws(() => { throw new Error('err'); });
```

## 六、异步测试

### 回调风格

```js
it('should read file content', (done) => {
  fs.readFile('/tmp/test.txt', 'utf8', (err, data) => {
    if (err) return done(err);   // 测试失败
    expect(data).to.include('hello');
    done();                       // 通知 Mocha 异步操作完成
  });
});
```

### Promise 风格

```js
it('should fetch user data', () => {
  return fetch('/api/user/1')
    .then(res => res.json())
    .then(user => {
      expect(user).to.have.property('name');
    });
});
```

### async / await 风格（推荐）

```js
it('should create an order', async () => {
  const order = await orderService.create({ userId: 1, items: [] });
  expect(order.status).to.equal('pending');
});
```

**注意事项**：
- 使用 `done` 回调时，如果超时（默认 2000ms），Mocha 会报超时错误
- `async` 函数不需要 `done`，但必须 return Promise
- 不要在 `async` 和 `done` 混用——会导致不可预期的行为

```js
// ❌ 错误混用
it('wrong', async (done) => {
  // 同时使用了 async 和 done
});
```

### 超时设置

```js
// 全局超时
this.timeout(10000);    // 10 秒

// 用例级别
it('slow test', async function() {
  this.timeout(5000);
  await delay(3000);
});
```

## 七、Mock / Stub / Spy（Sinon）

### Spy（间谍）

Spy 包装一个函数，记录其调用次数、参数和返回值，但不改变其行为：

```js
import sinon from 'sinon';

it('should call callback', () => {
  const callback = sinon.spy();
  [1, 2, 3].forEach(callback);

  expect(callback.calledThrice).to.be.true;
  expect(callback.firstCall.args[0]).to.equal(1);
});
```

```js
// 监听现有对象的方法
const spy = sinon.spy(console, 'log');
console.log('hello');
expect(spy.calledWith('hello')).to.be.true;
spy.restore();   // 恢复原始方法
```

### Stub（桩）

Stub 替换一个函数，完全控制其行为：

```js
it('should return mocked data', async () => {
  // 替代 api.getUser，不发送真实 HTTP 请求
  sinon.stub(api, 'getUser').resolves({ id: 1, name: 'Alice' });

  const result = await userService.getUser(1);

  expect(result.name).to.equal('Alice');
  expect(api.getUser.calledOnce).to.be.true;

  api.getUser.restore();
});
```

常见的 Stub 场景：

```js
// 返回固定值
stub.returns(42);

// 抛出异常
stub.throws(new Error('fail'));

// 返回不同的值（按调用顺序）
stub.onFirstCall().returns(1);
stub.onSecondCall().returns(2);

// 替换整个模块
const readStub = sinon.stub(fs, 'readFileSync');
readStub.withArgs('config.json').returns(JSON.stringify({ port: 3000 }));
```

### Mock（模拟对象）

Mock 在测试前预设期望，测试结束后验证期望是否全部满足：

```js
it('should call save exactly once', () => {
  const mock = sinon.mock(db);
  mock.expects('save').once().withArgs(sinon.match.object);

  controller.create({ name: 'test' });

  mock.verify();   // 验证 save 被调用了一次
  mock.restore();
});
```

### Fake Timers（假时钟）

用于测试 `setTimeout`、`setInterval`、`Date.now` 等时间相关逻辑：

```js
it('should debounce calls', () => {
  const clock = sinon.useFakeTimers();
  const callback = sinon.spy();

  const debounced = debounce(callback, 100);
  debounced(); debounced(); debounced();

  // 时间还没到，callback 不应被调用
  expect(callback.notCalled).to.be.true;

  // 快进 100ms
  clock.tick(100);
  expect(callback.calledOnce).to.be.true;

  clock.restore();
});
```

### Sinon 与 Mocha 的集成模式

```js
// 推荐：使用 sandbox 自动清理
describe('User Service', () => {
  let sandbox;

  beforeEach(() => {
    sandbox = sinon.createSandbox();
  });

  afterEach(() => {
    sandbox.restore();   // 一次 restore 所有 stub/spy/mock
  });

  it('should get user', async () => {
    sandbox.stub(api, 'getUser').resolves({ id: 1 });
    const user = await userService.getUser(1);
    expect(user.id).to.equal(1);
  });
});
```

## 八、浏览器测试

Mocha 可以直接在浏览器中运行，这是它相比 Jest 的独特优势：

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8">
  <title>Mocha Browser Test</title>
  <link rel="stylesheet" href="node_modules/mocha/mocha.css">
</head>
<body>
  <div id="mocha"></div>

  <script src="node_modules/mocha/mocha.js"></script>
  <script src="node_modules/chai/chai.js"></script>

  <script>
    mocha.setup('bdd');
    const expect = chai.expect;
  </script>

  <!-- 测试代码 -->
  <script>
    describe('DOM Tests', () => {
      it('should add element', () => {
        const div = document.createElement('div');
        div.className = 'box';
        document.body.appendChild(div);
        expect(document.querySelectorAll('.box').length).to.equal(1);
      });
    });
  </script>

  <script>
    mocha.run();
  </script>
</body>
</html>
```

浏览器测试的优势：
- 可以测试真实 DOM API（`getBoundingClientRect`、`IntersectionObserver` 等）
- 可以测试 Canvas、WebGL、Web Audio 等浏览器专用 API
- 可以验证布局和渲染行为（需配合截图对比）

## 九、测试报告与覆盖率

### 报告器（Reporter）

```bash
# 默认 spec 报告器
npx mocha --reporter spec

# JSON 格式
npx mocha --reporter json

# 进度条
npx mocha --reporter progress

# 生成 HTML 报告
npx mocha --reporter html > report.html

# 在 CI 中输出 JUnit 格式
npx mocha --reporter mocha-junit-reporter
```

### 测试覆盖率（c8 / istanbul）

```bash
npm install --save-dev @istanbuljs/nyc
```

```json
// package.json
{
  "scripts": {
    "test": "mocha 'src/**/*.test.js'",
    "coverage": "nyc npm test"
  },
  "nyc": {
    "reporter": ["text", "lcov", "html"],
    "include": ["src/**/*.js"],
    "exclude": ["src/**/*.test.js"]
  }
}
```

```bash
npm run coverage
# ---------------------|---------|---------|---------|---------|
# File                 | % Stmts | % Branch| % Funcs | % Lines |
# ---------------------|---------|---------|---------|---------|
# src/utils/math.js    |     100 |      75 |     100 |     100 |
# src/services/user.js |   89.47 |    87.5 |    87.5 |   89.47 |
# ---------------------|---------|---------|---------|---------|
```

覆盖率指标：

| 指标 | 含义 | 目标 |
|------|------|------|
| **Stmts** | 语句覆盖率，每行代码是否被执行 | ≥ 80% |
| **Branch** | 分支覆盖率，每个 `if/else`、`switch/case` 是否覆盖 | ≥ 75% |
| **Funcs** | 函数覆盖率，每个函数是否被调用 | ≥ 90% |
| **Lines** | 行覆盖率，同 Stmts 类似 | ≥ 80% |

## 十、实际项目配置

### 完整的 Mocha 配置

```json
// .mocharc.json
{
  "ui": "bdd",
  "reporter": "spec",
  "timeout": 5000,
  "recursive": true,
  "exit": true,
  "require": ["@babel/register"],
  "spec": ["src/**/*.test.js", "test/**/*.test.js"]
}
```

或使用 `mocha.config.js`：

```js
// mocha.config.js
module.exports = {
  ui: 'bdd',
  reporter: 'spec',
  timeout: 5000,
  recursive: true,
  exit: true,
  require: ['@babel/register'],
};
```

### 与 Babel / ES Modules 配合

```bash
npm install --save-dev @babel/register @babel/preset-env
```

```json
// .babelrc
{
  "presets": [
    ["@babel/preset-env", { "targets": { "node": "current" } }]
  ]
}
```

```json
// .mocharc.json
{
  "require": ["@babel/register"]
}
```

### 与 TypeScript 配合

```bash
npm install --save-dev tsx
```

```json
// .mocharc.json
{
  "require": ["tsx"],
  "extensions": ["ts", "tsx"]
}
```

```js
// src/user.test.ts
import { expect } from 'chai';

describe('TypeScript', () => {
  it('should work with TS', () => {
    const greet = (name: string) => `Hello ${name}`;
    expect(greet('World')).to.equal('Hello World');
  });
});
```

### CI 集成

```yaml
# .github/workflows/test.yml
name: Test

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'
      - run: npm install
      - run: npm test
      - run: npm run coverage
      - uses: actions/upload-artifact@v4
        with:
          name: coverage
          path: coverage/
```

## 十一、最佳实践

### 1. 测试结构（AAA 模式）

每个测试用例应遵循 Arrange-Act-Assert 模式：

```js
it('should calculate total price with discount', () => {
  // Arrange（准备）
  const items = [
    { price: 100, quantity: 2 },
    { price: 50, quantity: 1 },
  ];
  const discountRate = 0.9;

  // Act（执行）
  const total = calculateTotal(items, discountRate);

  // Assert（断言）
  expect(total).to.equal(225);   // (200 + 50) * 0.9
});
```

### 2. 避免测试内部实现

```js
// ❌ 不好的：测试内部实现细节
it('should call internal method', () => {
  const spy = sinon.spy(User.prototype, 'validate');
  const user = new User({ name: 'Alice' });
  user.save();
  expect(spy.calledOnce).to.be.true;
});

// ✅ 好的：测试外部行为
it('should reject invalid user name', () => {
  const user = new User({ name: '' });
  expect(() => user.save()).to.throw('用户名不能为空');
});
```

### 3. 每个 it 只测一件事

```js
// ❌ 一次测多个行为
it('should handle user operations', () => {
  const user = createUser();
  expect(user.name).to.equal('Alice');
  expect(user.save()).to.be.true;
  expect(user.delete()).to.be.true;
});

// ✅ 拆分为独立的测试
it('should create user with name');
it('should save user to database');
it('should delete user from database');
```

### 4. 使用工厂函数简化准备

```js
// test/factories.js
export function buildUser(overrides = {}) {
  return {
    id: 1,
    name: 'Alice',
    email: 'alice@example.com',
    role: 'user',
    ...overrides,
  };
}

// 测试中使用
it('should format user display name', () => {
  const user = buildUser({ name: 'Bob', role: 'admin' });
  expect(formatDisplay(user)).to.equal('Bob (admin)');
});
```

### 5. 测试命名规范

```
should [expected behavior] when [condition]
```

```js
// 清晰描述"什么条件下应该发生什么"
it('should return cached data when cache is valid');
it('should fetch fresh data when cache is expired');
it('should throw error when token is missing');
```

### 6. 测试金字塔

```text
        ╱╲
       ╱ E2E ╲          少量：关键用户流程，覆盖最广
      ╱────────╲
     ╱ Integration ╲    适量：模块间交互
    ╱────────────────╲
   ╱   Unit Tests     ╲  大量：函数级测试，快而精确
  ╱────────────────────╲
```

Mocha 主要负责金字塔的底层：**单元测试** 和 **集成测试**。

## 十二、常见问题

### Q1: Mocha 和 Jest 应该选哪个

- 新项目优先选 **Jest** 或 **Vitest**（零配置，内置断言/Mock）
- Node.js 工具库或浏览器测试优先选 **Mocha**（灵活、浏览器原生支持）
- 已有 Mocha 项目不需要迁移，Mocha 仍然被积极维护

### Q2: 异步测试总超时

- 检查 `done()` 是否被调用（或调用了多次）
- 检查 `async` 函数是否没有 return 或 await
- 检查网络请求是否有 Mock
- 适当调大 `timeout` 值

### Q3: 测试文件应该放在哪

两种主流策略：

```
# 策略一：集中管理（推荐给大多数项目）
test/
├── unit/
│   ├── utils.test.js
│   └── services.test.js
├── integration/
│   └── api.test.js
└── factories.js

# 策略二：就近放置（适合库/组件库）
src/
├── utils/
│   ├── math.js
│   └── math.test.js   # 测试和源码放一起
└── components/
    ├── Button.jsx
    └── Button.test.jsx
```

### Q4: 如何测试私有函数

如果函数没有导出，有几种方法：

- 通过公共方法间接测试（推荐）
- 使用 `rewire` 或 `proxyquire` 获取私有函数
- 如果确实需要直接测试，说明这个函数可能应该被提取到独立模块

### Q5: 覆盖率 100% 有必要吗

没必要。追求 100% 覆盖率会导致测试成本急剧上升。建议：
- 核心业务逻辑：≥ 90%
- 工具函数：≥ 80%
- UI 组件：按需覆盖关键状态
- 配置代码、类型定义：不需要覆盖

## 十三、推荐学习路径

1. 理解单元测试的理念和价值（本文）
2. 用 Mocha + Chai 写一个纯函数的测试（如 `add`、`formatDate`）
3. 引入 Sinon，Mock 网络请求和定时器
4. 配置覆盖率工具，理解哪些代码没有被覆盖
5. 集成到 CI 流水线，让每次提交自动运行测试
6. 阅读优秀开源项目的测试代码（如 Lodash、Express 的测试）
