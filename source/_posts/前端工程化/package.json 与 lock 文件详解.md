---
title: package.json 与 lock 文件详解
categories: 
- 前端工程化
tags:
- package.json
- npm
- lock
- 版本管理
- 依赖
---

## 一、package.json 是什么

`package.json` 是 Node.js 项目的清单文件，描述了项目的基本信息、依赖、脚本、配置等。它是所有 Node.js/npm 项目的核心入口。<!--more-->

```json
{
  "name": "my-app",
  "version": "1.0.0",
  "private": true,
  "description": "A sample project",
  "scripts": { ... },
  "dependencies": { ... },
  "devDependencies": { ... }
}
```

## 二、核心字段详解

### name 和 version

```json
{
  "name": "@scope/package-name",
  "version": "1.0.0"
}
```

- `name`：包名，发布到 npm 的唯一标识。可加 scope（`@org/pkg`）
- `version`：遵循 SemVer（语义化版本）

### 依赖声明

```json
{
  "dependencies": {
    "vue": "^3.4.0",
    "express": "~4.18.0",
    "lodash": "4.17.21"
  },
  "devDependencies": {
    "typescript": "^5.0.0",
    "vitest": "^1.0.0"
  },
  "peerDependencies": {
    "react": "^18.0.0"
  },
  "optionalDependencies": {
    "esbuild": "^0.20.0"
  },
  "bundledDependencies": ["my-lib"]
}
```

| 字段 | 用途 | 是否被打包 |
|------|------|-----------|
| `dependencies` | 运行时依赖（生产环境需要） | ✅ |
| `devDependencies` | 开发依赖（构建/测试/lint 工具） | ❌ |
| `peerDependencies` | 宿主依赖（插件声明需要的宿主版本，如 ESLint 插件） | ❌ 但不安装 |
| `optionalDependencies` | 可选依赖（安装失败不中断流程） | ❌ |
| `bundledDependencies` | 打包到发布包中的依赖 | ✅ 随包发布 |

`dependencies` 与 `devDependencies` 的区别在前端项目中主要影响 CI 安装速度——`npm install --production` 只安装 `dependencies`。

### 脚本

```json
{
  "scripts": {
    "dev": "vite",
    "build": "vue-tsc && vite build",
    "preview": "vite preview",
    "test": "vitest",
    "lint": "eslint . --fix",
    "typecheck": "tsc --noEmit",
    "prepare": "husky",
    "precommit": "lint-staged"
  }
}
```

npm 支持**钩子脚本**，在特定事件前后自动执行：

| 钩子 | 触发时机 |
|------|----------|
| `preinstall` | `npm install` 之前 |
| `postinstall` | `npm install` 之后 |
| `prepublishOnly` | `npm publish` 之前 |
| `prepare` | `npm install` 之后 + `npm publish` 之前（常用于 husky） |
| `preversion` | 版本号修改前 |
| `version` | 版本号修改后 |

```json
{
  "scripts": {
    "preversion": "npm test",
    "version": "git add -A .",
    "postversion": "git push && git push --tags"
  }
}
```

### 发布相关

```json
{
  "private": true,
  "main": "dist/index.js",
  "module": "dist/index.mjs",
  "types": "dist/index.d.ts",
  "exports": {
    ".": {
      "import": "./dist/index.mjs",
      "require": "./dist/index.js",
      "types": "./dist/index.d.ts"
    },
    "./styles.css": "./dist/styles.css"
  },
  "files": ["dist", "README.md"],
  "license": "MIT",
  "sideEffects": false
}
```

| 字段 | 说明 |
|------|------|
| `private: true` | 防止意外发布到 npm |
| `main` | CommonJS 入口（`require('pkg')` 加载的文件） |
| `module` | ES Module 入口（构建工具优先使用此字段） |
| `types` | TypeScript 类型声明入口 |
| `exports` | 子路径导出控制（**推荐**，优先级高于 main/module） |
| `files` | 发布时包含的文件（白名单） |
| `sideEffects` | Tree Shaking 标记：`false` 表示可安全摇树 |

`exports` 是 Node.js 12+ 引入的字段，支持条件导出和子路径控制。如果设置了 `exports`，它**会隐藏所有未显式声明的路径**：

```json
{
  "exports": {
    ".": "./index.js",
    "./utils": "./utils.js"
  }
}
// import 'pkg'           → OK
// import 'pkg/utils'     → OK
// import 'pkg/internal'  → ❌ 编译错误
```

### 配置字段

```json
{
  "engines": {
    "node": ">=18.0.0",
    "pnpm": ">=8.0.0"
  },
  "packageManager": "pnpm@9.0.0",
  "type": "module",
  "browserslist": {
    "development": ["last 1 chrome version"],
    "production": ["> 0.2%", "not dead"]
  }
}
```

| 字段 | 说明 |
|------|------|
| `engines` | 声明项目需要的 Node/npm 版本（非强制，但安装时警告） |
| `packageManager` | 指定包管理器版本（Corepack 会强制使用） |
| `type` | `"module"` 表示项目默认使用 ES Module |
| `browserslist` | 浏览器兼容性目标（Autoprefixer、Babel 等工具共用） |
| `overrides` | 覆盖依赖的依赖版本 |
| `pnpm.overrides` | pnpm 特有的依赖覆盖 |
| `workspaces` | Monorepo 工作区配置 |

### overrides——强制版本覆盖

当某个间接依赖有安全漏洞时，用 `overrides` 强制指定版本：

```json
{
  "overrides": {
    "axios": "1.6.0",
    "loader-utils": "3.2.1",
    "minimist": {
      ".": "1.2.8",
      "deep-equal": "2.2.0"
    }
  }
}
```

### workspaces——Monorepo 工作区

```json
{
  "private": true,
  "workspaces": ["packages/*", "apps/*"]
}
```

## 三、SemVer 语义化版本

### 版本格式

```text
主版本号.次版本号.补丁号
  1   .   2    .   3

主版本号：不兼容的 API 修改（breaking change）
次版本号：向下兼容的新功能
补丁号：向下兼容的问题修复
```

### 预发布版本

```text
1.0.0-alpha.1
1.0.0-beta.2
1.0.0-rc.1

alpha < beta < rc < release
```

### 版本范围符号

| 写法 | 含义 | 示例 |
|------|------|------|
| `1.2.3` | 精确版本 | 只安装 1.2.3 |
| `^1.2.3` | **兼容大版本** | >=1.2.3 <2.0.0（最常用） |
| `~1.2.3` | **兼容小版本** | >=1.2.3 <1.3.0 |
| `>=1.2.3` | 大于等于 | |
| `*` | 任意版本 | |
| `1.x` 或 `1.*` | 主版本固定 | >=1.0.0 <2.0.0 |

**`^` 的详细行为**：

```text
^1.2.3  →  >=1.2.3 <2.0.0
^0.2.3  →  >=0.2.3 <0.3.0（0.x 时次版本号视为主版本）
^0.0.3  →  >=0.0.3 <0.0.4（0.0.x 时补丁号视为主版本）
```

### npm 的版本安装策略

```text
npm install vue          →  安装最新版，写入 ^x.y.z
npm install vue@3        →  安装 3.x 最新版，写入 ^3.x.z
npm install vue@3.4      →  安装 3.4.x 最新版，写入 ~3.4.z（或 ^3.4.0）
npm install vue@latest   →  安装 latest tag
npm install vue@next     →  安装 next tag（通常是 beta/rc）
npm install --save-exact →  写入精确版本（不带 ^/）
```

## 四、lock 文件

### 为什么需要 lock 文件

```text
没有 lock 文件：
  package.json 中声明 "vue": "^3.4.0"
  今天  npm install  →  安装 3.4.5
  明天  npm install  →  安装 3.5.0（3.5.0 已发布）
  → 同一个项目在不同时间安装的依赖版本不一致
  → "在我的机器上能跑"

有 lock 文件：
  package.json 中声明 "vue": "^3.4.0"
  lock 文件中锁定 vue@3.4.5（及其所有子依赖的精确版本）
  无论何时安装，始终是 3.4.5
```

### lock 文件的工作原理

lock 文件记录了**依赖树的完整快照**，包括：

```text
├── 直接依赖的精确版本
├── 间接依赖的精确版本
├── 每个包的 resolved URL（下载地址）
├── 每个包的 integrity hash（完整性校验）
├── 依赖的依赖关系（谁依赖谁）
└── 依赖的打包信息
```

### 不同包管理器的 lock 文件

| 包管理器 | lock 文件名 | 格式 |
|----------|------------|------|
| npm | `package-lock.json` | JSON |
| yarn | `yarn.lock` | 自定义文本格式 |
| pnpm | `pnpm-lock.yaml` | YAML |

```yaml
# pnpm-lock.yaml（可读性最好的 lock 格式）
packages:
  /vue/3.4.21:
    resolution: { integrity: sha512-... }
    engines: { node: ^18.0.0 }
    dependencies:
      '@vue/compiler-dom': 3.4.21
      '@vue/shared': 3.4.21
```

### 版本锁定机制

```text
package.json 声明：vue: "^3.4.0"
lock 文件锁定：vue@3.4.21

npm install 流程：
  1. 检查 lock 文件是否存在
  2. 如果存在 → 读取 lock，安装 lock 中记录的精确版本
  3. 如果不存在 → 解析 package.json 中的范围，安装最新兼容版本，生成 lock
  
lock 文件更新时机：
  ├── 手动执行 npm install 新包
  ├── 手动执行 npm update
  ├── 修改 package.json 中的版本范围后 npm install
  └── 删除 node_modules 后重新安装，lock 不变
```

### lock 文件的冲突处理

```text
场景：团队多人同时修改 package.json（添加不同依赖）
      各自生成 package-lock.json，合并时产生冲突。

冲突本质：lock 文件记录了整个依赖树，即使只加了一个包，
          lock 文件中其他包的顺序/格式也可能不同。

解决方案：
  方案 A：用 `git merge` 冲突标记手动合并（痛苦）
  方案 B：只用一方 lock，另一方重新 npm install 生成
  方案 C（推荐）：使用 pnpm，pnpm-lock.yaml 的冲突远少于 package-lock.json
```

### lock 文件的常见问题

#### Q1: lock 文件应该提交到 Git 吗

```text
✅ 必须提交。
理由：
  ├── 保证团队安装一致的依赖
  ├── 保证 CI/CD 环境与本地一致
  ├── 便于回滚到历史版本时恢复当时的依赖
  └── 审查依赖变更（code review 时可以看到锁的变化）
```

#### Q2: lock 文件导致所有依赖变精确版本，怎么更新

```bash
# 更新单个包到符合 package.json 范围的最新版
npm install vue@latest

# 更新所有依赖到符合范围的最新版（并更新 lock）
npm update

# 完全重建 lock（谨慎使用）
rm -rf node_modules package-lock.json
npm install
```

#### Q3: 为什么 CI 上要用 `npm ci` 而不是 `npm install`

```text
npm install：
  ├── 检查 lock 文件 → 在 lock 范围内安装
  ├── 但如果 lock 与 package.json 不一致，会修改 lock
  └── 安装时间长

npm ci（CI 环境专用）：
  ├── 严格按照 lock 安装，lock 不存在或与 package.json 不一致则报错
  ├── 删除 node_modules 后全新安装
  ├── 安装速度比 npm install 快
  └── 保证 CI 与本地 100% 一致
```

#### Q4: lock 文件冲突如何解决

```bash
# 方案一：保留冲突的 lock，重新生成
git checkout --theirs package-lock.json   # 或 --ours
npm install                                # 重新生成

# 方案二：用 pnpm 减少冲突
# pnpm-lock.yaml 按包名排序，冲突概率远低于 npm
```

## 五、常见面试题

### Q1: `dependencies` 和 `devDependencies` 的区别

```text
dependencies：项目运行时需要的包（vue、react、axios）
devDependencies：只在开发/构建时需要的包（vite、eslint、typescript）

前端项目中区别不大，因为最终代码都被打包到一起。
但在 CI 中，npm install --production 只安装 dependencies，可加快安装速度。
在发布 npm 包时，用户安装你的包只会安装 dependencies。
```

### Q2: `^` 和 `~` 的区别

```text
^1.2.3：允许安装 >=1.2.3 <2.0.0，即大版本不变
~1.2.3：允许安装 >=1.2.3 <1.3.0，即次版本也不变

生产中建议用 ^，安全地获得 bugfix。直接使用 lock 文件保证一致性。
```

### Q3: lock 文件在 Monorepo 中如何处理

```text
Monorepo（pnpm workspace）：
  ├── 根目录一个 pnpm-lock.yaml，锁定所有 workspace 包的依赖
  ├── 安装速度更快（依赖提升）
  ├── 统一管理版本冲突

Monorepo（npm workspaces）：
  ├── 每个 workspace 可能有自己的 package-lock（npm）
  └── 根目录也有一个 package-lock
```

### Q4: 什么是幽灵依赖

```text
幽灵依赖（Phantom Dependency）：
  项目中可以直接 import 一个没有在 package.json 中声明的包，
  因为该包被其他依赖提升到了 node_modules 顶层。

危害：
  ├── 依赖隐式、不可追溯
  ├── 某天间接依赖更新/移除，项目直接报错
  └── 不同环境下 node_modules 结构不同，行为不一致

解决：pnpm 通过严格的依赖隔离解决了幽灵依赖。
```
