---
title: AST 详解
date: 2026-07-15 16:30:00
categories:
  - 前端工程化
  - 编译原理
tags:
  - AST
  - Babel
  - TypeScript
  - 编译原理
  - 代码转换
---

# AST 详解

## 【面试速答版】

### Q1: 什么是 AST？为什么构建工具需要它？

AST（Abstract Syntax Tree，抽象语法树）是**源代码语法结构的一种树形表示**。树的每个节点对应源代码中的一个语法结构（如变量声明、函数调用、if 语句）。

为什么需要 AST？因为**字符串操作无法理解代码含义**。看这个例子：

```javascript
const message = 'hello' + 'world';
```

用字符串替换：`message.replace('hello', '你好')` 能得到 `'你好world'`。但如果你想让 `const` 变成 `var`，字符串正则替换很容易误伤：

```javascript
const result = 'const x = 1'; // 字符串里的 const 不该被替换
// 正则替换 /const/ → /var/ 会误改这里
```

而 AST 让工具能够**精确理解代码结构**——知道哪里是变量声明、哪里是字符串字面量、哪里是注释。所以 Babel、TypeScript、eslint、Prettier 这类工具都基于 AST 工作。

整个过程分三步：
1. **解析（Parse）**：源码 → AST（词法分析 + 语法分析）
2. **转换（Transform）**：遍历 AST，增删改节点
3. **生成（Generate）**：AST → 目标代码

### Q2: Babel 和 TypeScript Compiler 都使用 AST，它们的用法有什么不同？

**相同点**：两者都基于 AST 做代码转换，都遵循 parse → transform → generate 的流程。

**核心差异**：

| 维度 | Babel | TypeScript Compiler |
|:---|:---|:---|
| **AST 规范** | 自定义的 Babylon/Babel AST | TypeScript AST（ts.Node） |
| **解析器** | @babel/parser（前身 Babylon） | tsc 内置解析器 |
| **类型信息** | ❌ AST 无类型信息 | ✅ AST 节点携带类型（ts.Type） |
| **插件机制** | 丰富的 visitor 插件模式 | 无传统插件系统，提供 Transformer API |
| **典型用途** | 语法转译、polyfill、代码转换 | 类型检查、语言服务（IDE 补全/跳转） |
| **输出** | 转换后的源码 | 类型检查结果 + 可选转译输出 |

**关键区别**：Babel 的 AST 只包含「代码结构」，不包含「类型信息」。所以在 `tsc --noEmit` 能报出类型错误的地方，Babel 可能直接放行。但 Babel 的插件生态极其丰富，可以做各种自定义转换。

### Q3: AST 的三个阶段（Parse / Transform / Generate）分别做了什么？

**Parse（解析）**：将源码字符串转为 AST。

包含两个子阶段：
- **词法分析（Lexical Analysis）**：源码 → Token 流。逐个字符读取，识别出关键字、标识符、运算符、字符串等 token。
  ```javascript
  const a = 1;
  // 词法分析结果：[
  //   { type: 'Keyword', value: 'const' },
  //   { type: 'Identifier', value: 'a' },
  //   { type: 'Punctuator', value: '=' },
  //   { type: 'Numeric', value: '1' },
  //   { type: 'Punctuator', value: ';' },
  // ]
  ```
- **语法分析（Syntactic Analysis）**：Token 流 → AST。根据语言的语法规则（如 JS 的 ECMAScript 规范），将 token 组合成有层级的语法结构。

**Transform（转换）**：遍历 AST，通过 visitor 模式访问节点并修改。

```javascript
// Babel 插件示例：将所有 const 声明改为 var
module.exports = function () {
  return {
    visitor: {
      VariableDeclaration(path) {
        if (path.node.kind === 'const') {
          path.node.kind = 'var';
        }
      },
    },
  };
};
```

**Generate（生成）**：将修改后的 AST 重新转为代码字符串。这个阶段还会处理格式化（缩进、换行）、sourcemap 生成等工作。

---

## 【深入理解版】

### 1. AST 要解决什么问题？（从 0 讲起）

#### 1.1 字符串操作的天花板

假设你接到了一个任务：把项目中所有 `var` 声明改为 `let`。

你可能会想：「这还不简单？全局正则替换 `var` → `let`」

```javascript
// 要处理的代码
const code = `
var x = 1;
var y = "var is a string";
var z = variableNameContainsVar;
`;
```

你用 `code.replace(/var /g, 'let ')` 试试看：

```javascript
// 结果：
let x = 1;
let y = "let is a string";   // ✅ 正确，但注意字符串内容也变了
let z = variableNameContainslet; // ❌ 变量名被破坏
```

问题出现了：
1. 字符串里的 `"var"` 被误改成了 `"let"`
2. 变量名 `variableNameContainsVar` 中的 `Var` 被替换（由于正则加了空格才没命中，但如果变量名是 `varName` 呢？）

你改进正则：`/\bvar\b/g`：

```javascript
code.replace(/\bvar\b/g, 'let');
// 对于 var x = 1; 没问题
// 但对于 var x = variableVar;  —— 
// variableVar 中的 var 不是独立单词，\b 边界会正确识别
// 但如果遇到 var x = typeof var; 呢？typeof 后面的 var 是关键字吗？
```

你开始意识到：**仅靠字符串操作，你永远无法 100% 确定某个 `var` 是关键字还是标识符的一部分**。因为：
- `var` 在源代码中有三种角色：变量声明关键字、标识符的一部分、字符串内容
- 字符串操作没有「代码语境」的感知能力
- 语言的语法规则是递归嵌套的，正则表达式无法描述这种递归结构

**AST 的核心价值就在这里——它把「字符串」变成了「有语义的结构化数据」**。

#### 1.2 从字符串到 AST 的思维转变

把代码看作字符串 VS 把代码看作 AST：

```javascript
// 源代码
function add(a, b) {
  return a + b;
}
```

**作为字符串**：
```
"function add(a, b) {\n  return a + b;\n}"
// 索引 0-8 → "function "
// 索引 9-11 → "add"
// 索引 12 → "("
// ... 只有线性顺序，没有结构信息
```

**作为 AST**：

```
Program
├── FunctionDeclaration (name: "add")
│   ├── params
│   │   ├── Identifier (name: "a")
│   │   └── Identifier (name: "b")
│   └── body
│       └── BlockStatement
│           └── ReturnStatement
│               └── BinaryExpression (operator: "+")
│                   ├── Identifier (name: "a")
│                   └── Identifier (name: "b")
```

AST 的结构告诉你：
- `function add(a, b) { ... }` 是一个函数声明，不是函数调用
- `a` 和 `b` 是参数，不是全局变量
- `a + b` 是一个二元表达式，不是字符串拼接
- `return` 是函数内的返回语句

**有了 AST，工具就可以精确地回答这些问题**：
- 这个函数有哪些参数？→ 查 `FunctionDeclaration.params`
- 这个函数内部调用了哪些其他函数？→ 查 `body` 中的 `CallExpression`
- 这个变量在哪里声明、在哪里被引用？→ 查 `Identifier` 的定义和引用位置

### 2. AST 的核心原理与执行过程

#### 2.1 整体流程

```
源码 (Source Code)
    │
    ▼
┌─────────────────────────────────────────────────┐
│                   Parse                         │
│  ┌──────────────┐    ┌──────────────────────┐    │
│  │  词法分析     │ →  │      语法分析        │    │
│  │ (Lexer/扫描器) │    │ (Parser/解析器)       │    │
│  │ Source → Tokens│    │ Tokens → AST         │    │
│  └──────────────┘    └──────────────────────┘    │
└─────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────┐
│                  Transform                      │
│  遍历 AST，应用插件/转换器修改节点               │
│  (Babel: visitor, TS: Transformer)              │
└─────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────┐
│                  Generate                       │
│  AST → 代码字符串 + Sourcemap                   │
└─────────────────────────────────────────────────┘
    │
    ▼
目标代码
```

#### 2.2 词法分析（Lexical Analysis）— 源码切碎成 Token

词法分析器（也叫扫描器、Lexer）的任务是：**逐个字符读取源码，将其分组为有意义的词素（lexeme），输出 token 流**。

让我们手动模拟一个极简的词法分析器如何处理 `const a = 1;`：

```
输入字符流: 'c' 'o' 'n' 's' 't' ' ' 'a' ' ' '=' ' ' '1' ';'

第 1 步：读取 'c' → 进入「标识符/关键字」模式
         继续读 'o' 'n' 's' 't' → 发现后面是空格
         匹配到关键字 "const"
         输出 Token: { type: 'Keyword', value: 'const' }

第 2 步：跳过空格 ' '

第 3 步：读取 'a' → 进入「标识符/关键字」模式
         继续读，发现后面是空格，结束
         "a" 不是关键字，所以是标识符
         输出 Token: { type: 'Identifier', value: 'a' }

第 4 步：跳过空格 ' '

第 5 步：读取 '=' → 单字符的 Punctuator（标点符号）
         输出 Token: { type: 'Punctuator', value: '=' }

第 6 步：跳过空格 ' '

第 7 步：读取 '1' → 进入「数字」模式
         继续读，发现后面是 ';'，结束
         输出 Token: { type: 'Numeric', value: '1' }

第 8 步：读取 ';' → 单字符的 Punctuator
         输出 Token: { type: 'Punctuator', value: ';' }

最终 token 流：
[
  { type: 'Keyword',   value: 'const' },
  { type: 'Identifier', value: 'a'     },
  { type: 'Punctuator', value: '='     },
  { type: 'Numeric',    value: '1'     },
  { type: 'Punctuator', value: ';'     },
]
```

这个阶段**不关心语法是否正确**。比如 `const const const` 也能被正常分词为三个 `Keyword` token，语法错误是语法分析阶段的工作。

实际项目中比较复杂的词法分析场景包括：

- **模板字符串**：`hello ${name} world` — 需要跟踪嵌套的花括号状态
- **正则表达式字面量**：`/abc/g` — 需要区分除号 `/` 和正则起始符 `/`
- **注释**：`//` 行注释和 `/* */` 块注释 — 直接忽略，不产生 token
- **JSX**：`<Component prop={value} />` — 需要处理 HTML 风格的标签

```javascript
// 真实词法分析器需要考虑的边缘情况
const tricky1 = 0.5;       // 小数 → Numeric
const tricky2 = .5;        // 合法小数（省略整数部分）
const tricky3 = 1e10;      // 科学计数法
const tricky4 = 0xFF;      // 十六进制
const tricky5 = 0o77;      // 八进制（ES2015+）
const tricky6 = 0b1010;    // 二进制（ES2015+）
const tricky7 = 1_000_000; // 数字分隔符（ES2021+）
```

#### 2.3 语法分析（Syntactic Analysis）— Token 组装成 AST

语法分析器（通常直接称为 Parser）的任务是：**根据语言的语法规则（Grammar），将 token 流组织成有层级的 AST 结构**。

常见的实现方式有两种：
- **手写递归下降解析器（Recursive Descent Parser）**：为每种语法结构写一个解析函数。Babel、TypeScript、esbuild 都使用这种方式。
- **自动生成的解析器**：用 Parser Generator（如 Yacc、Antlr）从语法描述文件自动生成。性能不如手写，但开发速度快。

以 `const a = 1;` 为例，递归下降解析的过程：

```javascript
// 简化版递归下降解析器

function parseProgram(tokens) {
  const body = [];
  while (tokens.length > 0) {
    body.push(parseStatement(tokens));
  }
  return { type: 'Program', body };
}

function parseStatement(tokens) {
  if (tokens[0].value === 'const') {
    return parseVariableDeclaration(tokens);
  }
  if (tokens[0].value === 'function') {
    return parseFunctionDeclaration(tokens);
  }
  // ... 其他语句类型
  throw new Error('Unexpected token: ' + tokens[0].value);
}

function parseVariableDeclaration(tokens) {
  // 吃掉 'const'
  tokens.shift(); // 消费 Keyword('const')
  
  // 下一个必须是标识符
  const name = tokens.shift();
  if (name.type !== 'Identifier') {
    throw new Error('Expected identifier after const');
  }
  
  // 下一个必须是 '='
  const eq = tokens.shift();
  if (eq.value !== '=') {
    throw new Error('Expected = after identifier');
  }
  
  // 解析初始值（这里简化，只处理数字字面量）
  const init = tokens.shift();
  
  // 消费末尾的分号
  const semi = tokens.shift();
  
  return {
    type: 'VariableDeclaration',
    kind: 'const',
    declarations: [{
      type: 'VariableDeclarator',
      id: { type: 'Identifier', name: name.value },
      init: { type: 'NumericLiteral', value: init.value },
    }],
  };
}
```

递归下降解析器的「递归」体现在处理嵌套结构时。比如解析一个 if 语句：

```javascript
// 源代码
if (x > 0) { return x; }

// 解析过程
parseStatement(tokens)
  → 识别 if 关键字
  → parseIfStatement:
    → 解析条件：parseExpression → parseBinaryExpression
      → parsePrimary (Identifier "x")
      → 识别运算符 ">"
      → parsePrimary (Numeric 0)
    → 解析 then 分支：parseBlockStatement
      → parseStatement → parseReturnStatement
    → 递归调用！因为 if 可能有 else
    → 检查下一个 token 有没有 else
```

这里有两点值得注意：
1. **递归性**：语句内可以嵌套语句，表达式内可以嵌套表达式。解析器通过互相递归调用来处理这种嵌套。
2. **前瞻（Lookahead）**：解析器需要向前看一个或多个 token 来确定当前是什么语法结构。比如看到 `(` 可能是表达式分组，也可能是函数调用的参数列表，需要根据上下文判断。

#### 2.4 AST 节点结构

不同工具的 AST 节点结构略有不同，但核心思想一致。以 Babel 的 AST 为例：

```javascript
// Babel AST 的核心节点类型（部分）

// Program — 根节点
{
  type: 'Program',
  body: [/* 顶级语句 */],
  sourceType: 'module', // 或 'script'
}

// VariableDeclaration — 变量声明
{
  type: 'VariableDeclaration',
  kind: 'const',      // 'var' | 'let' | 'const'
  declarations: [
    {
      type: 'VariableDeclarator',
      id: { type: 'Identifier', name: 'x' },
      init: { type: 'NumericLiteral', value: 1 },
    },
  ],
}

// FunctionDeclaration — 函数声明
{
  type: 'FunctionDeclaration',
  id: { type: 'Identifier', name: 'add' },
  params: [
    { type: 'Identifier', name: 'a' },
    { type: 'Identifier', name: 'b' },
  ],
  body: {
    type: 'BlockStatement',
    body: [
      {
        type: 'ReturnStatement',
        argument: {
          type: 'BinaryExpression',
          operator: '+',
          left: { type: 'Identifier', name: 'a' },
          right: { type: 'Identifier', name: 'b' },
        },
      },
    ],
  },
}

// ArrowFunctionExpression — 箭头函数
{
  type: 'ArrowFunctionExpression',
  params: [{ type: 'Identifier', name: 'x' }],
  body: { type: 'NumericLiteral', value: 42 },
  expression: true,
}
```

Babel 的 AST 规范基于 **@babel/types** 包，所有节点类型都有对应的类型定义。TypeScript 的 AST 则基于 `ts.Node` 体系，区别在于 TypeScript 的节点还包括类型信息。

#### 2.5 核心机制：Visitor 模式

Visitor 模式是 AST 转换的核心。它的思想很简单：**你告诉工具「我对某类节点感兴趣」，工具遍历整棵 AST，遇到这类节点就调用你的回调函数**。

```javascript
// Babel 插件的 visitor 结构
module.exports = function () {
  return {
    visitor: {
      // 进入 VariableDeclaration 节点时调用
      VariableDeclaration(path) {
        // path.node 指向当前 AST 节点
        // path.parentPath 指向父节点
        // path.scope 包含作用域信息
      },
      
      // 也可以分别处理进入和退出
      FunctionDeclaration: {
        enter(path) {
          // 进入函数声明时
        },
        exit(path) {
          // 离开函数声明时
        },
      },
    },
  };
};
```

**为什么需要 visitor？**

手动遍历 AST 然后判断类型非常繁琐：

```javascript
// 手动遍历 — 不推荐
function traverse(node, callback) {
  if (node.type === 'VariableDeclaration') callback(node);
  if (node.body) node.body.forEach(n => traverse(n, callback));
  if (node.declarations) node.declarations.forEach(n => traverse(n, callback));
  if (node.init) traverse(node.init, callback);
  // ... 需要为每种节点类型写遍历逻辑
}
```

**visitor 模式如何工作？**

Babel 的 `traverse` 核心逻辑（简化）：

```javascript
// Babel traverse 简化版
function traverse(node, visitors, parent) {
  const enterFn = visitors[node.type];
  
  // 进入节点
  if (enterFn) {
    enterFn({ node, parent, scope: /* 计算作用域 */ });
  }
  
  // 根据节点类型遍历子节点
  switch (node.type) {
    case 'Program':
    case 'BlockStatement':
      node.body.forEach(child => traverse(child, visitors, node));
      break;
    case 'VariableDeclaration':
      node.declarations.forEach(child => traverse(child, visitors, node));
      break;
    case 'VariableDeclarator':
      traverse(node.id, visitors, node);
      if (node.init) traverse(node.init, visitors, node);
      break;
    // ... 数十种节点类型
  }
  
  // 退出节点
  const exitFn = visitors[node.type]?.exit;
  if (exitFn) {
    exitFn({ node, parent });
  }
}
```

当你在 Babel 插件中实现 `VariableDeclaration(path) { ... }` 时，Babel 会在每次**进入**一个 `VariableDeclaration` 节点时调用你的函数。通过 `path` 你可以：
- `path.node` — 访问当前节点
- `path.parentPath` — 访问父节点
- `path.replaceWith(newNode)` — 替换当前节点
- `path.remove()` — 删除当前节点
- `path.insertBefore(newNode)` — 在当前节点前插入
- `path.insertAfter(newNode)` — 在当前节点后插入
- `path.traverse(visitor)` — 在当前子树中继续遍历

### 3. Babel 中的 AST 使用方式

#### 3.1 Babel 的完整工作流

Babel 是一个 JavaScript 编译器，它的设计遵循**三阶段架构**：

```javascript
// Babel 的编译流程
const babel = require('@babel/core');

// 输入：源码字符串
const sourceCode = `
  const greet = (name) => {
    return \`Hello, \${name}\`;
  }
`;

// 输出：转换后的代码
const result = babel.transformSync(sourceCode, {
  presets: ['@babel/preset-env'],
  plugins: [
    // 可以传入自定义插件
    './my-custom-plugin',
  ],
  sourceMaps: true,
});

console.log(result.code);
// 根据 target 配置，箭头函数可能被转为普通函数：
// var greet = function(name) {
//   return "Hello, ".concat(name);
// };
```

**三步详解**：

```
第 1 步：@babel/parser（解析）
源代码 → 词法分析 → token 流 → 语法分析 → AST

第 2 步：@babel/traverse + plugins（转换）
AST → 深度优先遍历 → 执行每个 plugin 的 visitor → 修改后的 AST

第 3 步：@babel/generator（生成）
修改后的 AST → 代码字符串 + sourcemap
```

这三个功能被拆分为独立的 `@babel/parser`、`@babel/traverse`、`@babel/generator` 包，每个都可以单独使用。

#### 3.2 手动使用 Babel 操作 AST

**场景**：写一个工具，把所有 `console.log` 调用替换为自定义的 `logger.info`。

**步骤 1：解析生成 AST**

```javascript
const parser = require('@babel/parser');

const code = `
function handleClick() {
  console.log('Button clicked');
  console.log('Count:', count);
  const data = fetchData();
  console.log('Data loaded');
}
`;

const ast = parser.parse(code, {
  sourceType: 'module',
  plugins: ['typescript'], // 支持 TypeScript 语法
});

// 查看 AST 结构
console.log(JSON.stringify(ast, null, 2));
// Program.body[0] → FunctionDeclaration
//   .body.body[0] → ExpressionStatement
//     .expression → CallExpression
//       .callee → MemberExpression
//         .object → Identifier 'console'
//         .property → Identifier 'log'
```

**步骤 2：遍历并转换 AST**

```javascript
const traverse = require('@babel/traverse').default;
const t = require('@babel/types'); // 辅助创建/检查 AST 节点

traverse(ast, {
  // 访问所有 CallExpression（函数调用表达式）
  CallExpression(path) {
    const node = path.node;
    
    // 检查是否是 console.log(...) 调用
    if (
      t.isMemberExpression(node.callee) &&
      t.isIdentifier(node.callee.object, { name: 'console' }) &&
      t.isIdentifier(node.callee.property, { name: 'log' })
    ) {
      // 创建 logger.info(...) 调用
      const replacement = t.callExpression(
        t.memberExpression(
          t.identifier('logger'),
          t.identifier('info')
        ),
        node.arguments // 保留原参数
      );
      
      // 替换
      path.replaceWith(replacement);
    }
  },
});
```

**步骤 3：生成目标代码**

```javascript
const generate = require('@babel/generator').default;

const result = generate(ast, {
  retainLines: false,
  concise: false,
});

console.log(result.code);
// 输出：
// function handleClick() {
//   logger.info('Button clicked');
//   logger.info('Count:', count);
//   const data = fetchData();
//   logger.info('Data loaded');
// }
```

#### 3.3 @babel/types 的节点操作

`@babel/types` 提供了两个核心能力：**判断节点类型** 和 **创建新节点**。

**判断节点类型**：

```javascript
const t = require('@babel/types');

// 判断节点是否为特定类型
if (t.isIdentifier(node, { name: 'console' })) {
  // node 是名为 "console" 的 Identifier
}

if (t.isStringLiteral(node)) {
  // node 是字符串字面量
  const value = node.value; // 直接获取字符串值
}

// 更底层的检查
if (node.type === 'CallExpression') {
  // 等价于 t.isCallExpression(node)
}
```

**创建新节点**：

```javascript
// 创建标识符 identifier
t.identifier('myVariable')
// → { type: 'Identifier', name: 'myVariable' }

// 创建字符串字面量
t.stringLiteral('hello')
// → { type: 'StringLiteral', value: 'hello' }

// 创建数字字面量
t.numericLiteral(42)
// → { type: 'NumericLiteral', value: 42 }

// 创建布尔字面量
t.booleanLiteral(true)

// 创建 null 字面量
t.nullLiteral()

// 创建数组表达式
t.arrayExpression([t.numericLiteral(1), t.numericLiteral(2)])
// → { type: 'ArrayExpression', elements: [...] }

// 创建对象属性
t.objectExpression([
  t.objectProperty(
    t.identifier('name'),
    t.stringLiteral('Alice')
  ),
])

// 创建变量声明
t.variableDeclaration('const', [
  t.variableDeclarator(
    t.identifier('x'),
    t.numericLiteral(1)
  ),
])
// → {
//   type: 'VariableDeclaration',
//   kind: 'const',
//   declarations: [{
//     type: 'VariableDeclarator',
//     id: { type: 'Identifier', name: 'x' },
//     init: { type: 'NumericLiteral', value: 1 },
//   }],
// }

// 创建函数调用
t.callExpression(
  t.identifier('setTimeout'),
  [
    t.arrowFunctionExpression(
      [],
      t.callExpression(t.identifier('cb'), [])
    ),
    t.numericLiteral(1000),
  ]
)
// → setTimeout(() => { cb(); }, 1000)
```

#### 3.4 Babel 插件的完整实战

**需求**：写一个 Babel 插件，自动给 i18n 中未包裹的字符串字面量加上 `t()` 调用。

```javascript
// babel-plugin-auto-i18n.js
module.exports = function (api, options) {
  const { t } = api.types;

  return {
    name: 'auto-i18n',
    visitor: {
      // 处理 JSX 中的文本节点
      JSXText(path) {
        const text = path.node.value.trim();
        if (text && options.pattern?.test(text)) {
          path.replaceWith(
            t.jsxExpressionContainer(
              t.callExpression(t.identifier('t'), [
                t.stringLiteral(text),
              ])
            )
          );
        }
      },

      // 处理模板字符串中的纯文本部分
      TemplateLiteral(path) {
        const quasis = path.node.quasis;
        quasis.forEach((quasi, i) => {
          // 跳过表达式之间的部分（如 ${name} 前后的文本保留）
          const text = quasi.value.raw.trim();
          if (text && shouldTranslate(text)) {
            quasi.value.raw = `{{t("${text}")}}`;
          }
        });
      },

      // 处理普通字符串字面量（不处理已经作为 t() 参数的）
      StringLiteral(path) {
        // 跳过 t() 的参数
        if (
          t.isCallExpression(path.parent) &&
          t.isIdentifier(path.parent.callee, { name: 't' })
        ) {
          return;
        }
        
        // 跳过对象属性名
        if (t.isObjectProperty(path.parent) && path.parent.key === path.node) {
          return;
        }
        
        const text = path.node.value;
        if (text && shouldTranslate(text)) {
          path.replaceWith(
            t.callExpression(t.identifier('t'), [
              t.stringLiteral(text),
            ])
          );
        }
      },
    },
  };
};
```

使用时在 Babel 配置中注册：

```json
{
  "plugins": [
    ["./babel-plugin-auto-i18n", {
      "pattern": "/[\\u4e00-\\u9fa5]+/"
    }]
  ]
}
```

#### 3.5 Babel 的 Path 与 Scope

Babel 中的 `path` 对象比直接操作 AST 节点更强大，它提供了节点间的「导航」能力。

**Path 的核心能力**：

```javascript
// path 提供了节点在树中的上下文信息
traverse(ast, {
  Identifier(path) {
    // 路径相关
    path.node        // 当前 AST 节点
    path.parent      // 父节点
    path.parentPath   // 父路径
    path.container   // 节点所在的数组（如 body）
    path.key         // 节点在父对象中的键名
    path.listKey     // 如果节点在数组中，在数组中的位置
    
    // AST 操作
    path.replaceWith(newNode)     // 替换当前节点
    path.replaceWithMultiple(nodes) // 替换为多个节点
    path.remove()                 // 删除节点
    path.insertBefore(node)       // 在前方插入
    path.insertAfter(node)        // 在后方插入
    
    // 查找
    path.findParent(callback)     // 向上查找父节点
    path.find(callback)           // 向上查找（包括自身）
    path.get('body.0')            // 按路径获取子节点
    
    // 作用域信息
    path.scope                    // 当前作用域
    path.scope.bindings           // 当前作用域的所有绑定
    path.scope.references         // 当前作用域的所有引用
  },
});
```

**Scope（作用域）的用途**：

```javascript
// 在插件中处理变量重命名
traverse(ast, {
  VariableDeclaration(path) {
    const binding = path.scope.getBinding(path.node.declarations[0].id.name);
    
    if (binding) {
      // binding.path             → 声明该变量的节点路径
      // binding.referencePaths   → 所有引用该变量的路径
      // binding.constant         → 是否为常量
      // binding.constantViolations → 修改该变量的操作
      
      console.log(`变量 ${binding.identifier.name} 被引用了 ${binding.referencePaths.length} 次`);
      
      // 安全地重命名变量
      const newName = path.scope.generateUidIdentifier('newName');
      binding.rename(newName.name);
    }
  },
});
```

作用域信息让 Babel 能安全地进行变量重命名、检测变量是否被使用、判断是否有命名冲突等操作。这是**纯字符串操作无法做到的**。

### 4. TypeScript Compiler 中的 AST 使用方式

#### 4.1 TypeScript Compiler 的架构

TypeScript 的编译器（tsc）比 Babel 更复杂，因为它不只是做转译，还要做**类型检查**和**语言服务**（IDE 中的自动补全、跳转定义等）。

```
源代码 (.ts / .tsx)
    │
    ▼
┌──────────────────────────────────────┐
│            Scanner（扫描器）           │
│    词法分析：源码 → Token 流           │
│    (ts.createSourceFile 内部调用)     │
└──────────────────────────────────────┘
    │
    ▼
┌──────────────────────────────────────┐
│            Parser（解析器）            │
│    语法分析：Token → AST (SourceFile) │
│    Parser 在 parseSourceFile 中完成   │
└──────────────────────────────────────┘
    │
    ▼
┌──────────────────────────────────────┐
│       Binder（绑定器/符号表构建）       │
│   将声明与引用关联，建立 Symbol 表      │
│   每个 Symbol 包含类型、声明位置、      │
│   引用位置等信息                      │
└──────────────────────────────────────┘
    │
    ▼
┌──────────────────────────────────────┐
│      Checker（类型检查器）             │
│   基于 Symbol 表进行类型推导和检查      │
│   类型错误在这一步被发现               │
└──────────────────────────────────────┘
    │
    ▼
┌──────────────────────────────────────┐
│         Emitter / Transformer         │
│   代码生成：AST → 目标代码 (JS)       │
│   可注册自定义 Transformer 做转换     │
└──────────────────────────────────────┘
    │
    ▼
目标代码 (.js) + 声明文件 (.d.ts) + Sourcemap
```

和 Babel 的关键区别：**TypeScript 的 AST 贯穿整个编译流程，不只是解析和转换，类型检查也基于 AST 展开**。

#### 4.2 手动使用 TypeScript Compiler API

TypeScript 的所有编译功能都通过 `typescript` 包提供：

```typescript
import * as ts from 'typescript';

const sourceCode = `
interface Person {
  name: string;
  age: number;
}

function greet(person: Person): string {
  return \`Hello, \${person.name}\`;
}
`;

// 第 1 步：创建 SourceFile（AST）
const sourceFile = ts.createSourceFile(
  'example.ts',
  sourceCode,
  ts.ScriptTarget.Latest,
  true, // setParentPointers
  ts.ScriptKind.TS
);

// 第 2 步：遍历 AST
function visitNode(node: ts.Node, depth: number = 0) {
  const indent = '  '.repeat(depth);
  
  // ts.SyntaxKind 是 TS AST 中所有节点类型的枚举
  console.log(indent + ts.SyntaxKind[node.kind]);
  
  // 如果节点有类型信息（在执行类型检查后）
  // if (node.type) { ... }
  
  // 递归遍历子节点
  ts.forEachChild(node, child => visitNode(child, depth + 1));
}

visitNode(sourceFile);
// 输出：
// SourceFile
//   InterfaceDeclaration
//     Identifier (Person)
//     PropertySignature
//       Identifier (name)
//       StringKeyword
//     PropertySignature
//       Identifier (age)
//       NumberKeyword
//   FunctionDeclaration
//     Identifier (greet)
//     Identifier (person)
//     TypeReference
//       Identifier (Person)
//     Block
//       ReturnStatement
//         TemplateExpression
//           TemplateSpan
//             PropertyAccessExpression
//               Identifier (person)
//               Identifier (name)
```

**和 Babel 的区别已经显现了**：TypeScript 的 AST 节点类型通过 `ts.SyntaxKind` 枚举表示，而不是 Babel 的字符串 `node.type`。此外 `ts.forEachChild` 是 TypeScript 提供的标准遍历函数。

#### 4.3 TypeScript AST 节点的类型体系

TypeScript AST 的节点体系是一个**复杂的类继承结构**：

```
ts.Node（所有 AST 节点的基类）
├── ts.SourceFile（入口文件）
├── ts.Statement（语句）
│   ├── ts.VariableStatement
│   ├── ts.FunctionDeclaration
│   ├── ts.InterfaceDeclaration
│   ├── ts.ClassDeclaration
│   └── ...
├── ts.Expression（表达式）
│   ├── ts.Identifier
│   ├── ts.BinaryExpression
│   ├── ts.CallExpression
│   ├── ts.ArrowFunction
│   └── ...
├── ts.TypeNode（类型节点 — 纯 TS 概念，Babel 没有）
│   ├── ts.StringKeyword
│   ├── ts.NumberKeyword
│   ├── ts.TypeReference
│   ├── ts.ArrayTypeNode
│   ├── ts.UnionTypeNode
│   ├── ts.IntersectionTypeNode
│   └── ...
└── ...
```

**TypeScript AST 独有的特点**：

1. **类型节点是 AST 的一等公民**：在 Babel 中，`string` 只是一个 `Identifier` 节点。在 TypeScript AST 中，它是 `StringKeyword` 节点，带有完整的类型语义。

2. **节点携带位置信息**：每个节点都有 `pos`（起始位置）和 `end`（结束位置），指向源码字符串的索引。

```typescript
// 源代码
const x: number = 42;

// AST 节点
const varDecl = sourceFile.statements[0] as ts.VariableStatement;
const decl = varDecl.declarationList.declarations[0];
const typeNode = decl.type!; // ts.NumberKeyword

console.log(typeNode.pos);  // 9（':' 后面的空格不算，number 的 'n'）
console.log(typeNode.end);  // 15（number 后面的空格）
// sourceFile.text.substring(typeNode.pos, typeNode.end) → 'number'
```

3. **节点有 `parent` 指针**：每个节点都通过 `parent` 指向父节点，方便向上遍历（前提是 `setParentPointers: true`）。

#### 4.4 TypeScript 的 Transformer API

TypeScript 也提供了类似 Babel 的转换能力，通过 **Transformer** 机制。

```typescript
import * as ts from 'typescript';

// 自定义 Transformer：将所有 const 声明改为 let
function constToLetTransformer(/* context: ts.TransformationContext */): ts.TransformerFactory<ts.SourceFile> {
  // 返回一个 TransformerFactory
  return (context: ts.TransformationContext) => {
    // 返回一个 Transformer（SourceFile → SourceFile）
    return (sourceFile: ts.SourceFile) => {
      // 使用 visitor 模式遍历
      const visitor = (node: ts.Node): ts.Node => {
        // 检查是否是变量声明列表
        if (ts.isVariableStatement(node)) {
          const varStmt = node as ts.VariableStatement;
          if (varStmt.declarationList.flags & ts.NodeFlags.Const) {
            // 创建新的声明列表，移除 Const flag
            return ts.factory.updateVariableStatement(
              varStmt,
              varStmt.modifiers,
              ts.factory.createVariableDeclarationList(
                varStmt.declarationList.declarations,
                ts.NodeFlags.Let  // 改为 let
              )
            );
          }
        }
        return ts.visitEachChild(node, visitor, context);
      };
      
      return ts.visitEachChild(sourceFile, visitor, context);
    };
  };
}
```

**使用方式**：

```typescript
// 编译选项
const options: ts.CompileOptions = {
  target: ts.ScriptTarget.ES2020,
  module: ts.ModuleKind.ESNext,
};

// 创建程序并编译
const program = ts.createProgram(['input.ts'], options);
const sourceFile = program.getSourceFile('input.ts')!;

// 获取编译器输出（应用了我们自定义的 transformer）
const result = ts.transform(sourceFile, [constToLetTransformer()]);

// 打印结果
const printer = ts.createPrinter();
const output = printer.printFile(result.transformed[0]);
console.log(output);
```

**TypeScript 的 Transformer 和 Babel 插件的区别**：

| 方面 | Babel | TypeScript |
|:---|:---|:---|
| **注册方式** | 配置文件里的 plugins 数组 | createProgram 时传入 customTransformers |
| **节点创建** | @babel/types 的辅助函数 | ts.factory（v4.0+）或手动构造 |
| **节点检查** | t.isXXX(node) | ts.isXXX(node) 类型守卫 |
| **类型信息** | ❌ 无 | ✅ 可通过 TypeChecker 获取 |
| **复杂度** | 低，API 简洁 | 高，需要理解 ts.Node 体系 |
| **使用比例** | 极多（成千上万个 Babel 插件） | 极少（主要在框架工具中） |

#### 4.5 TypeScript 的类型检查器（TypeChecker）

这是 TypeScript AST 相比于 Babel AST 最强大的地方——**类型信息**。

```typescript
import * as ts from 'typescript';

function demonstrateTypeChecker() {
  const program = ts.createProgram(['app.ts'], {
    target: ts.ScriptTarget.ES2020,
    strict: true,
  });
  
  const sourceFile = program.getSourceFile('app.ts')!;
  
  // 获取类型检查器
  const checker = program.getTypeChecker();
  
  // 遍历 AST，获取每个标识符的类型信息
  function visitWithType(node: ts.Node) {
    if (ts.isIdentifier(node)) {
      // 获取该标识符的类型
      const symbol = checker.getSymbolAtLocation(node);
      if (symbol) {
        const type = checker.getTypeOfSymbolAtLocation(symbol, node);
        
        console.log(`标识符: ${node.text}`);
        console.log(`  类型: ${checker.typeToString(type)}`);
        
        // 检查类型是否可赋值给另一个类型
        // const isAssignable = checker.isTypeAssignableTo(type, targetType);
        
        // 获取属性的类型
        // const propType = checker.getTypeOfPropertyOfType(type, 'name');
      }
    }
    
    ts.forEachChild(node, visitWithType);
  }
  
  visitWithType(sourceFile);
}
// 假设 app.ts 中有一个变量 const name: string = 'Alice';
// 输出：
// 标识符: name
//   类型: string
```

**这在 Babel 里做不到**。Babel 的 AST 不包含类型信息，`t.isIdentifier` 只能检查节点类型，无法知道它的类型是 `string` 还是 `number`。

**TypeChecker 的常见用途**：

```typescript
// 1. 获取变量的声明类型
const symbol = checker.getSymbolAtLocation(node);
const declarations = symbol?.declarations;
// → 可以跳到变量声明的位置（"Go to Definition"）

// 2. 获取表达式的返回类型
const callExpr = node as ts.CallExpression;
const signature = checker.getResolvedSignature(callExpr);
const returnType = checker.getReturnTypeOfSignature(signature!);
// → IDE 中悬停显示函数返回值类型

// 3. 类型收窄检查
// 检查两个类型是否兼容
if (checker.isTypeAssignableTo(sourceType, targetType)) {
  // sourceType 可以赋值给 targetType
}

// 4. 获取类型的完整信息
const properties = checker.getPropertiesOfType(type);
// → 获取接口/类的所有成员
```

#### 4.6 TypeScript 的语言服务（Language Service）

TypeScript 的语言服务 API 是其 AST 能力的高阶应用，支撑了所有 IDE 中的 TS 功能：

```typescript
import * as ts from 'typescript';

// 创建语言服务
const servicesHost: ts.LanguageServiceHost = {
  getScriptFileNames: () => ['app.ts'],
  getScriptVersion: () => '1',
  getScriptSnapshot: (fileName) => {
    if (fileName === 'app.ts') {
      return ts.ScriptSnapshot.fromString(code);
    }
    return undefined;
  },
  getCurrentDirectory: () => '.',
  getCompilationSettings: () => ({ target: ts.ScriptTarget.ES2020 }),
  getDefaultLibFileName: (options) => ts.getDefaultLibFilePath(options),
  fileExists: (path) => path === 'app.ts',
  readFile: (path) => path === 'app.ts' ? code : undefined,
  readDirectory: () => [],
};

const services = ts.createLanguageService(servicesHost);

// 1. 自动补全
const completions = services.getCompletionsAtPosition('app.ts', 10);
console.log(completions?.entries.map(e => e.name));

// 2. 跳转到定义
const definition = services.getDefinitionAtPosition('app.ts', 15);
// → 返回变量声明的位置

// 3. 查找引用
const references = services.getReferencesAtPosition('app.ts', 15);
// → 所有引用该变量的位置

// 4. 悬停提示
const quickInfo = services.getQuickInfoAtPosition('app.ts', 15);
// → 显示类型信息

// 5. 错误诊断
const diagnostics = services.getSemanticDiagnostics('app.ts');
// → 编译错误列表
```

**语言服务的工作方式**：它会在内存中维护 AST、Symbol 表、类型信息，并提供查询接口。当你输入代码时，编辑器实时调用 Language Service 获取补全建议、错误提示等。这也是为什么 TypeScript 的 AST 设计必须比 Babel 更精细——它不只需要做一次性的编译，还需要支持高频的增量查询。

### 5. Babel AST vs TypeScript AST 的核心差异

#### 5.1 类型信息的层级

```typescript
// TypeScript AST
const source = 'const x: string = "hello";';

// TS AST 中 x 的类型节点是 string 关键字类型
const varDecl = sourceFile.statements[0]; // ts.VariableStatement
const declList = varDecl.declarationList;   // ts.VariableDeclarationList
const decl = declList.declarations[0];      // ts.VariableDeclaration
const type = decl.type;                     // ts.StringKeyword（非空！）
```

```javascript
// Babel AST
// 同样一行代码，Babel 的 AST 中没有类型节点
{
  type: 'VariableDeclaration',
  kind: 'const',
  declarations: [{
    type: 'VariableDeclarator',
    id: {
      type: 'Identifier',
      name: 'x',
      typeAnnotation: {
        type: 'TSTypeAnnotation',
        typeAnnotation: {
          type: 'TSStringKeyword',
        },
      },
    },
    init: {
      type: 'StringLiteral',
      value: 'hello',
    },
  }],
}
```

区别：
- **TypeScript AST**：`decl.type` 直接是 `StringKeyword` 节点，类型是 AST 的一等公民。
- **Babel AST**：类型注解被放在 `Identifier.typeAnnotation` 中，是可选的附加信息。Babel 本身不解析类型语义。

#### 5.2 节点创建 API

```javascript
// Babel：@babel/types 的函数式 API
const t = require('@babel/types');

t.identifier('x');             // Identifier
t.stringLiteral('hello');      // StringLiteral
t.arrowFunctionExpression(     // ArrowFunctionExpression
  [t.identifier('x')],
  t.identifier('x')
);
```

```typescript
// TypeScript：ts.factory 命名空间（TS v4.0+）
import * as ts from 'typescript';

ts.factory.createIdentifier('x');             // Identifier
ts.factory.createStringLiteral('hello');      // StringLiteral
ts.factory.createArrowFunction(                // ArrowFunction
  undefined,                                  // modifiers
  undefined,                                  // typeParameters
  [ts.factory.createParameterDeclaration(
    undefined, undefined, undefined,
    ts.factory.createIdentifier('x'),
    undefined, undefined
  )],
  undefined,                                  // type
  undefined,                                  // equalsGreaterThanToken
  ts.factory.createIdentifier('x')
);
```

TypeScript 的 API 相对冗长，因为参数更多（需要处理修饰符、类型参数等 TS 特有的语法）。Babel 的 API 则更简洁，因为不需要处理类型相关的参数。

#### 5.3 典型的应用场景对比

| 场景 | 选 Babel | 选 TypeScript |
|:---|:---|:---|
| 转译 ES2025 → ES2015 | ✅ 核心场景 | ❌ 不擅长 |
| 添加 polyfill | ✅ @babel/preset-env + core-js | ❌ 不支持 |
| 自定义代码转换（如国际化包裹） | ✅ 极佳 | ⚠️ 可以但麻烦 |
| TypeScript 类型检查 | ❌ 不支持 | ✅ 核心场景 |
| IDE 补全/跳转/重构 | ❌ 不支持 | ✅ Language Service |
| 代码压缩/混淆 | ❌ 不擅长 | ❌ 不擅长（用 esbuild/terser） |
| 静态分析（复杂度计算、依赖分析） | ✅ 可以 | ✅ 可以且有类型信息 |
| 创建代码生成工具（如脚手架） | ✅ API 简洁 | ⚠️ API 较复杂 |
| 移除类型注解 → JS | ⚠️ 可以（不检查类型） | ✅ 原生能力 |

### 6. 常见误区 & 实际项目中的坑

#### 6.1 误区：所有 AST 节点类型等价

**错误写法**：

```javascript
// 错误地认为所有 'if' 结构都是 IfStatement
traverse(ast, {
  IfStatement(path) {
    // 以为能处理所有条件分支
  },
});
```

**为什么错**：条件表达式 `三元运算符（ConditionalExpression）` 和 `if 语句（IfStatement）` 是不同的 AST 节点。

```javascript
// 这是 IfStatement
if (x > 0) { console.log('positive'); }

// 这也是 ConditionalExpression（三元运算符）
const result = x > 0 ? 'positive' : 'non-positive';

// 使用方式不同：
// IfStatement → 语句，不能作为表达式赋值
// ConditionalExpression → 表达式，可以放在赋值右侧

// 如果插件只处理 IfStatement，会漏掉所有三元运算符。
```

**正确做法**：

```javascript
// 如果需要处理所有条件逻辑
traverse(ast, {
  IfStatement(path) {
    // 处理 if 语句
  },
  ConditionalExpression(path) {
    // 处理三元运算符
  },
  SwitchStatement(path) {
    // 处理 switch 语句
  },
  LogicalExpression(path) {
    // 处理 && / || 短路逻辑
  },
  NullishCoalescingExpression(path) {
    // 处理 ?? 空值合并
  },
});
```

#### 6.2 误区：修改 AST 节点后忘记处理副作用

**错误写法**：

```javascript
traverse(ast, {
  Identifier(path) {
    // 把 console.log 改为 myLog
    if (path.node.name === 'console') {
      path.node.name = 'myConsole'; // 直接修改 AST 节点
    }
  },
});
```

**为什么会有问题**：

```javascript
// 源代码
const obj = {
  console: '这是一个属性名', // 这里的 console 是属性名，不是全局对象
};

console.log('test'); // 这里才是要替换的全局对象

// 如果直接替换所有名为 console 的 Identifier：
const obj = {
  myConsole: '这是一个属性名', // ❌ 属性名被误改了
};

myConsole.log('test'); // ✅ 全局对象被改了
```

**正确做法**：通过 `path` 的上下文判断是否需要修改。

```javascript
traverse(ast, {
  Identifier(path) {
    if (
      path.node.name === 'console' &&
      path.parent.type === 'MemberExpression' &&
      path.parent.object === path.node &&
      !path.isReferencedIdentifier() // 检查是否是引用（而非定义）
    ) {
      path.node.name = 'myConsole';
    }
  },
});
```

还可以结合 path 的作用域信息做更精确的判断：

```javascript
traverse(ast, {
  Identifier(path) {
    if (path.node.name !== 'console') return;
    
    // 检查该标识符是否有对应的绑定（声明）
    const binding = path.scope.getBinding(path.node.name);
    if (binding) {
      // 有绑定说明是局部变量，不是全局的 console
      // 比如：const console = { log: () => {} };
      return;
    }
    
    // 没有绑定说明是全局 console 对象
    // 安全替换
    path.node.name = 'myConsole';
  },
});
```

#### 6.3 性能陷阱：不要一次遍历 AST 多次

**错误写法**：

```javascript
// ❌ 每次调用 .transformSync 都会完整遍历一次 AST
function applyMultipleTransforms(code) {
  let result = code;
  
  // 第 1 次遍历：转换箭头函数
  result = babel.transformSync(result, {
    plugins: [arrowFunctionPlugin],
  }).code;
  
  // 第 2 次遍历：转换模板字符串
  result = babel.transformSync(result, {
    plugins: [templateLiteralPlugin],
  }).code;
  
  // 第 3 次遍历：添加 polyfill
  result = babel.transformSync(result, {
    presets: ['@babel/preset-env'],
  }).code;
  
  return result;
}
```

**为什么错**：每次 `transformSync` 都会完整执行 parse → traverse → generate 三个阶段。三次调用意味着 parse 三次、generate 三次、traverse 三次。

**正确做法**：将所有插件在一个 transform 调用中完成。

```javascript
// ✅ 一次遍历，多个插件
function applyMultipleTransforms(code) {
  return babel.transformSync(code, {
    plugins: [
      arrowFunctionPlugin,
      templateLiteralPlugin,
    ],
    presets: ['@babel/preset-env'],
  }).code;
  // 只 parse 一次，只 traverse 一次（但按顺序执行所有 visitor），
  // 只 generate 一次
}
```

**Babel 的 visitor 合并机制**：当传入多个插件时，Babel 会将所有插件的 visitor 合并成一个大的 visitor 对象，在同一次遍历中依次触发。这避免了多次遍历 AST 的开销。

#### 6.4 TypeScript 的 ts.factory 必须使用最新 API

**错误写法**（TypeScript < 4.0 的方式）：

```typescript
// ❌ 直接构造对象（旧方式）
const idNode: ts.Identifier = {
  kind: ts.SyntaxKind.Identifier,
  text: 'myVar',
  pos: -1,
  end: -1,
  parent: undefined,
  original: undefined,
};
// 这种方式容易遗漏必要属性，而且不同版本间属性可能变化
```

**正确做法**（TypeScript 4.0+）：

```typescript
// ✅ 使用 ts.factory
const idNode = ts.factory.createIdentifier('myVar');
const strNode = ts.factory.createStringLiteral('hello');
const callNode = ts.factory.createCallExpression(
  idNode,
  undefined,
  [strNode]
);
```

使用 `ts.factory` 有几点好处：
- **类型安全**：TypeScript 编译器会检查参数类型和数量
- **向前兼容**：工厂函数内部会处理版本间的节点结构变化
- **自动补全**：IDE 能给出参数提示

如果你需要处理旧版本 TypeScript（< 4.0），可以用 `ts.createIdentifier` 等旧函数。但 4.0+ 项目应统一使用 `ts.factory`。

### 7. 与相关知识的关联 & 对比

#### 7.1 AST 在代码工具链中的角色

```
源码
 │
 ├──→ Babel      → AST → 语法转译 + Polyfill → 兼容代码
 ├──→ TypeScript → AST → 类型检查 + 转译 → 类型安全的 JS
 ├──→ ESLint     → AST → 规则检查 → 错误/警告报告
 ├──→ Prettier   → AST → 格式化 → 格式化后的代码
 ├──→ esbuild    → AST → 转译 + 打包 + 压缩 → 优化后的代码
 ├──→ Webpack    → AST (通过 acorn) → 模块分析 → bundle
 ├──→ UglifyJS/Terser → AST → 压缩 → 更小的代码
 └──→ Istanbul   → AST → 代码插桩 → 覆盖率报告
```

虽然这些工具都基于 AST，但它们对 AST 的使用深度和方式不同：

| 工具 | 解析器 | 是否修改 AST | 转换能力 | 类型感知 |
|:---|:---|:---|:---|:---|
| Babel | @babel/parser | ✅ 深度修改 | 极强（插件生态） | ❌ |
| TypeScript | 内置 Parser | ✅ 修改 | 中（Transformer） | ✅ |
| ESLint | espree/@typescript-eslint | ❌ 只读 | ❌（只检查） | ⚠️ 部分 |
| Prettier | @babel/parser 等 | ❌ 只读 | ❌（只格式化） | ❌ |
| esbuild | 内置 Parser（Go） | ✅ 深度修改 | 中 | ❌ |

#### 7.2 各工具 AST 的节点差异

同一种语法结构在不同工具中的 AST 表示不尽相同：

```javascript
// 源代码：a + b
```

**Babel AST**：
```json
{
  "type": "BinaryExpression",
  "operator": "+",
  "left": { "type": "Identifier", "name": "a" },
  "right": { "type": "Identifier", "name": "b" }
}
```

**TypeScript AST**：
```json
{
  "kind": 209, // SyntaxKind.BinaryExpression
  "operatorToken": {
    "kind": 39, // SyntaxKind.PlusToken
    "pos": 2, "end": 3
  },
  "left": { "kind": 79, "text": "a" },
  "right": { "kind": 79, "text": "b" }
}
```

**ESLint (espree) AST**：
```json
{
  "type": "BinaryExpression",
  "operator": "+",
  "left": { "type": "Identifier", "name": "a" },
  "right": { "type": "Identifier", "name": "b" }
}
```

**esbuild AST**（Go 结构体，等价的 JSON 表示）：
```json
{
  "type": "BinaryExpr",
  "op": "2", // opcode 编号
  "left": { "type": "EIdentifier", "ref": { "bits": "..." } },
  "right": { "type": "EIdentifier", "ref": { "bits": "..." } }
}
```

注意点：
- ESLint 早期使用 SpiderMonkey 的 AST 规范（Mozilla Parser API），和 Babel AST 结构相似但有些许差异。
- TypeScript AST 使用枚举数值（`SyntaxKind.Identifier = 79`）而非字符串，对机器更高效但人类阅读不友好。
- esbuild 的 AST 用 Go 结构体表示，字段名和设计都和 JS 工具不同（如 `EIdentifier` 而不是 `Identifier`）。

### 8. 现代最佳实践（2025-2026）

#### 8.1 在项目中组合使用 Babel 和 TypeScript AST

**策略**：用 TypeScript 做类型检查，用 Babel 做代码转换。

```json
{
  "scripts": {
    "type-check": "tsc --noEmit",
    "build": "tsc --noEmit && babel src --out-dir dist --extensions .ts,.tsx"
  }
}
```

这样分工的原因是：
- `tsc --noEmit` 负责类型安全（类型检查是 TS 的核心优势）
- `babel src ...` 负责代码转换（插件生态丰富、polyfill、兼容性处理）
- Babel 的 `@babel/preset-typescript` 会剥离类型注解，但**不做类型检查**

#### 8.2 在自定义工具中使用 Babel 操作 AST

当你需要写一个自定义代码转换工具时，Babel 的 API 比 TypeScript 更友好：

```javascript
// transform-decorator.js
const parser = require('@babel/parser');
const traverse = require('@babel/traverse').default;
const t = require('@babel/types');
const generate = require('@babel/generator').default;

function transformDecorator(code) {
  const ast = parser.parse(code, {
    sourceType: 'module',
    plugins: ['decorators-legacy'],
  });
  
  let hasChanges = false;
  
  traverse(ast, {
    ClassDeclaration(path) {
      const decorators = path.node.decorators || [];
      
      if (decorators.length > 0) {
        // 将装饰器信息提取为静态属性
        const decoratorMeta = decorators.map(d => ({
          name: d.expression.name || d.expression.callee?.name,
          arguments: d.expression.arguments || [],
        }));
        
        path.node.decorators = []; // 移除装饰器
        
        // 添加一个静态属性来记录装饰器信息
        path.node.body.body.unshift(
          t.classProperty(
            t.identifier('__decorators__'),
            t.valueToNode(decoratorMeta),
            null, // typeAnnotation
            true  // static
          )
        );
        
        hasChanges = true;
      }
    },
  });
  
  if (!hasChanges) return code;
  
  const result = generate(ast, { retainLines: true });
  return result.code;
}

// 使用
const input = `
@log
@throttle(500)
class MyComponent {}
`;

console.log(transformDecorator(input));
// 输出：
// class MyComponent {
//   static __decorators__ = [
//     { name: "log", arguments: [] },
//     { name: "throttle", arguments: [500] },
//   ];
// }
```

#### 8.3 使用 TypeScript Language Service 构建 IDE 插件

TypeScript Language Service API 是构建代码分析工具的强大基础：

```typescript
import * as ts from 'typescript';

class CodeAnalyzer {
  private service: ts.LanguageService;
  
  constructor(rootFiles: string[], options: ts.CompileOptions) {
    const host = this.createHost(rootFiles, options);
    this.service = ts.createLanguageService(host);
  }
  
  /**
   * 找出项目中所有未使用的变量
   */
  findUnusedVariables(): Array<{ name: string; file: string; line: number }> {
    const result: Array<{ name: string; file: string; line: number }> = [];
    
    // 获取所有文件
    const files = this.service.getProgram()!.getSourceFiles();
    
    for (const file of files) {
      if (file.isDeclarationFile) continue;
      
      const diagnostics = this.service.getSemanticDiagnostics(file.fileName);
      
      for (const diag of diagnostics) {
        // TypeScript 编译器对未使用变量的错误码是 6133
        if (diag.code === 6133) {
          const pos = file.getLineAndCharacterOfPosition(diag.start!);
          const text = diag.messageText.toString();
          
          // 从错误消息中提取变量名
          // "'x' is declared but its value is never read."
          const match = text.match(/'([^']+)'/);
          if (match) {
            result.push({
              name: match[1],
              file: file.fileName,
              line: pos.line + 1,
            });
          }
        }
      }
    }
    
    return result;
  }
  
  /**
   * 获取某个函数的调用链
   */
  getCallChain(functionName: string): string[] {
    const calls: string[] = [];
    const program = this.service.getProgram()!;
    const checker = program.getTypeChecker();
    
    for (const file of program.getSourceFiles()) {
      if (file.isDeclarationFile) continue;
      
      this.findReferences(checker, file, functionName, calls);
    }
    
    return [...new Set(calls)];
  }
  
  private findReferences(
    checker: ts.TypeChecker,
    node: ts.Node,
    targetName: string,
    acc: string[]
  ) {
    if (ts.isCallExpression(node)) {
      const name = this.getCalledFunctionName(node);
      if (name) {
        if (name === targetName) {
          acc.push(`${name} (${node.getSourceFile().fileName}:${node.getStart()})`);
        }
        
        // 获取函数体内的函数调用
        const symbol = checker.getSymbolAtLocation(node.expression);
        if (symbol?.valueDeclaration) {
          ts.forEachChild(symbol.valueDeclaration, child => {
            this.findReferences(checker, child, targetName, acc);
          });
        }
      }
    }
    
    ts.forEachChild(node, child => {
      this.findReferences(checker, child, targetName, acc);
    });
  }
  
  private getCalledFunctionName(node: ts.CallExpression): string | undefined {
    const expr = node.expression;
    if (ts.isIdentifier(expr)) return expr.text;
    if (ts.isPropertyAccessExpression(expr)) return expr.name.text;
    return undefined;
  }
  
  private createHost(
    files: string[],
    options: ts.CompileOptions
  ): ts.LanguageServiceHost {
    const fileContents = new Map<string, string>();
    
    for (const file of files) {
      fileContents.set(file, ts.sys.readFile(file)!);
    }
    
    return {
      getScriptFileNames: () => files,
      getScriptVersion: () => '1',
      getScriptSnapshot: (name) => {
        const content = fileContents.get(name);
        return content
          ? ts.ScriptSnapshot.fromString(content)
          : undefined;
      },
      getCurrentDirectory: () => ts.sys.getCurrentDirectory(),
      getCompilationSettings: () => options,
      getDefaultLibFileName: (opts) => ts.getDefaultLibFilePath(opts),
      fileExists: (name) => fileContents.has(name),
      readFile: (name) => fileContents.get(name),
      readDirectory: ts.sys.readDirectory,
    };
  }
}

// 使用
const analyzer = new CodeAnalyzer(
  ['src/index.ts', 'src/utils.ts'],
  { target: ts.ScriptTarget.ES2020, strict: true }
);

console.log('未使用的变量:', analyzer.findUnusedVariables());
console.log('调用链:', analyzer.getCallChain('handleSubmit'));
```

#### 8.4 AST 操作中的性能优化

**针对大型文件的策略**：

```javascript
// 1. 尽早退出，避免深层次遍历
traverse(ast, {
  // ❌ 在深层递归中一直检查
  // 每次进入 Identifier 都检查，即使我们只关心顶层的 VariableDeclaration
  
  // ✅ 使用 path.findParent 或 path.get 限定范围后操作
  Identifier(path) {
    // 只处理在顶层作用域中的 Identifier
    if (path.scope.parent === null) {
      // 顶层标识符
    }
  },
});

// 2. 只遍历需要的子树
traverse(ast, {
  // ❌ 在每个函数内部深挖
  // FunctionDeclaration(path) { path.traverse({...}) }
  
  // ✅ 使用 path.stop() 在不需要时阻止继续遍历
  FunctionDeclaration(path) {
    // 忽略超过 100 行的函数
    if (path.node.end - path.node.start > 5000) {
      path.skip(); // 跳过该子树
    }
  },
  
  // 3. 多个变换合并一次遍历
  // ❌ 分别写多个插件 → 多次遍历
  // ✅ 合并到一个 visitor 中
  'FunctionDeclaration|VariableDeclaration|ClassDeclaration'(path) {
    // 处理所有声明类型
  },
});
```

### 9. 常见疑问解答（自问自答）

#### Q1: Babel 和 TypeScript 用了不同的解析器，它们的 AST 互转方便吗？

不方便。两个 AST 在设计哲学上就有差异：

1. **TypeScript AST 包含类型信息**：`const x: string = "hello"` 在 TS AST 中 `x` 的类型是 `StringKeyword` 节点；在 Babel AST 中是 `Identifier + TSTypeAnnotation` 嵌套。

2. **节点类型不同**：Babel 用字符串（如 `'Identifier'`），TypeScript 用枚举（如 `SyntaxKind.Identifier = 79`）。

3. **结构差异**：TypeScript 的 `BinaryExpression` 用 `operatorToken` 子节点表示运算符，Babel 直接用 `operator: "+"` 字符串属性。

**如果需要互转**，可以借助 `@babel/plugin-syntax-typescript` 和 `@babel/preset-typescript`。Babel 解析 TS 代码时，会保留类型注解在 AST 中，然后 Babel 插件可以处理这些类型节点。但这不是「Babel AST 转 TS AST」，而是 Babel 用自己的方式表示 TypeScript 语法。

#### Q2: 为什么 ESLint 也要用 AST？它也需要转换代码吗？

ESLint 使用 AST 的原因是：**只有理解了代码结构，才能准确检查代码规范**。

```javascript
// ESLint 规则举例：no-unused-vars（禁止未使用变量）
function example() {
  const x = 1; // 如果 x 未被使用，ESLint 报错
  const y = 2; // 如果 y 被使用了，不报错
  console.log(y);
}

// 用 AST 实现这个规则：
// 1. 在 VariableDeclarator 中记录变量声明
// 2. 在 Identifier 中记录变量引用
// 3. 如果某个变量只有声明没有引用，报错
```

ESLint **不修改代码**（除了 `--fix` 时），它只用 AST 做**只读分析**。所以 ESLint 的 AST 只要足够「读」就行，不需要像 Babel 那样提供丰富的节点修改 API。

ESLint 最早使用 **espree** 解析器（基于 acorn），现在也支持 `@typescript-eslint/parser` 来解析 TypeScript。后者的 AST 包含了类型信息，使得 ESLint 可以编写类型感知的规则（如 `@typescript-eslint/no-floating-promises`）。

#### Q3: AST 和 CST（具体语法树）有什么区别？

CST（Concrete Syntax Tree，具体语法树）也被称为 **Parse Tree**，和 AST 的区别在于：

| 维度 | CST（具体语法树） | AST（抽象语法树） |
|:---|:---|:---|
| **包含信息** | 所有语法细节，包括括号、分号、关键字等 | 只保留「有用」的信息，省略标点等 |
| **节点数量** | 多（每个 token 一个节点） | 少（有语义的节点） |
| **是否适合转换** | ❌ 太冗余 | ✅ 简洁，便于操作 |
| **典型用例** | 编译器前端、语法高亮器 | 代码转换、lint、格式化 |

用代码对比：

```javascript
// 源代码
const x = (1 + 2) * 3;

// CST 会保留括号节点：
// Program
//   VariableDeclaration (const)
//     Identifier (x)
//     Initializer: BinaryExpression (*)
//       Left: BinaryExpression (+)
//         Left: NumericLiteral (1)
//         Operator: +
//         Right: NumericLiteral (2)
//       Operator: *
//       Right: NumericLiteral (3)
// 注意：括号在 CST 中是一个节点！但在 AST 中，
// 括号只是控制优先级，不产生新的语义节点。

// AST（括号被省略，因为运算顺序已经被树形结构决定了）：
// Program
//   VariableDeclaration (const)
//     Identifier (x)
//     Initializer: BinaryExpression (*)
//       Left: BinaryExpression (+)
//         Left: NumericLiteral (1)
//         Right: NumericLiteral (2)
//       Right: NumericLiteral (3)
```

**为什么 AST 能省略括号？** 因为 AST 的树形结构本身已经表达了运算优先级：`*` 的左右子节点中，`(+)` 先计算，所以不需要多余的括号节点。

大多数前端工具（Babel、TypeScript、ESLint、Prettier）都使用 AST。真正的 CST 在 JS 生态中比较少见，但在一些对格式保留要求严格的场景（如代码美化工具）中会被使用。

#### Q4: 写一个 Babel 插件和写一个 TypeScript Transformer，哪个门槛更高？

**Babel 插件门槛更低**，原因：

1. **文档和社区资源更丰富**：Babel 官网有完整的插件开发指南，社区有海量示例。
2. **API 设计更人性化**：`@babel/types` 的 `t.isXXX` 和 `t.xxx` 比 TS 的 `ts.isXXX` 和 `ts.factory.xxx` 更简洁。
3. **作用域管理内置**：Babel 的 `path.scope` 自动维护了作用域信息，TypeScript 的 Transformer 需要自己处理。
4. **调试更容易**：Babel 插件可以在 Chrome DevTools 中调试，TypeScript Transformer 的调试体验相对较差。

**TypeScript Transformer 门槛更高但能力更强**：

1. **类型信息加持**：可以基于类型做精确的代码转换（如只转换接受特定类型参数的函数调用）。
2. **原生 TS 支持**：不需要额外的 preset，原生处理 TypeScript 的所有语法。
3. **与语言服务集成**：可以访问 `LanguageService` 做更复杂的分析。

**学习建议**：先写 Babel 插件理解 AST 操作，再切入 TypeScript Transformer。这两个 API 的思路是相通的。

#### Q5: 在浏览器中也能解析 AST 吗？

可以。Babel 和 TypeScript 都提供了浏览器版本。

```html
<!-- 浏览器中解析 AST -->
<script src="https://unpkg.com/@babel/standalone/babel.min.js"></script>
<script>
  const code = 'const x = 1;';
  const ast = Babel.parse(code);
  console.log(ast.program.body[0].kind); // 'const'
</script>
```

浏览器中操作 AST 的常见场景：
- **在线代码编辑器（如 CodeSandbox、StackBlitz）**：实时编译用户输入的代码
- **代码教学工具**：在浏览器中展示代码的 AST 结构（如 astexplorer.net）
- **低代码平台**：将可视化组件拖拽映射为代码生成

#### Q6: AST Explorer 是什么？怎么使用它？

[AST Explorer](https://astexplorer.net/) 是学习和调试 AST 最常用的工具。它允许你：

1. 在左侧输入代码
2. 在右侧即时看到 AST 树
3. 切换不同的解析器（@babel/parser、typescript、espree、acorn、esbuild 等）
4. 编写自定义转换脚本并实时查看效果

**学习建议**：
- 在 AST Explorer 中输入不同的代码片段，观察 AST 结构变化
- 对比同一段代码在不同解析器下的 AST 差异
- 在转换面板中编写 Babel 插件，实时验证效果

### 10. 推荐的学习路径

```
1. 在 AST Explorer (https://astexplorer.net) 中玩 30 分钟
   → 输入简单代码（变量声明、函数、箭头函数），观察 AST 结构
   → 切换 @babel/parser 和 typescript，对比差异

2. 阅读本文【面试速答版】➝ 建立整体认知

3. 用 Babel 写第一个插件
   → 创建名为 `babel-plugin-hello.js` 的文件
   → 实现一个简单的 visitor：把所有的 `foo` 标识符改为 `bar`
   → 用 @babel/core 的 transformSync 在 Node.js 中测试

4. 阅读 Babel 官方插件手册
   → https://github.com/jamiebuilds/babel-handbook
   → 重点：Plugin 章节、path API、scope API

5. 用 TypeScript Compiler API 做 AST 遍历
   → 使用 ts.createSourceFile 解析 TS 代码
   → 用 ts.forEachChild 遍历 AST
   → 尝试获取类型信息（ts.TypeChecker）

6. 在实际项目中找一个需要代码自动转换的场景
   比如：自动添加 PropTypes、自动包裹 console.log、自动注入 try-catch

7. 研究一个成熟 Babel 插件的源码
   推荐：@babel/plugin-transform-arrow-functions（它很简洁，大概 20 行）
```

**关联知识点索引**：
- ES6/ES2025 语法对应的 AST 节点 → [ES6和CommonJS模块](./ES6和CommonJS模块.md)
- esbuild 中的 AST 使用 → [esbuild 详解](./esbuild%20详解.md)
- Babel 在 Webpack 中的 loader 角色 → [Webpack 详解](./Webpack%20详解.md)
- Vite 使用 esbuild 做 AST 转换 → [Vite 详解](./Vite%20详解.md)

**关联知识点索引**：
- 前端工程化中的构建工具链 → [Vite 详解](./Vite%20详解.md)、[esbuild 详解](./esbuild%20详解.md)
- Lint 工具中的 AST 使用 → [eslint 配置](./eslint%20配置.md)
- 模块系统的编译原理 → [ES6和CommonJS模块](./ES6和CommonJS模块.md)
