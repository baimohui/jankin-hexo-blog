---
title: 前端 CI/CD 详解
date: 2026-07-16 15:00:00
categories:
  - 前端工程化
  - CI/CD
tags:
  - CI/CD
  - GitHub Actions
  - GitLab CI
  - 自动化部署
  - 前端工程化
---

## 一、CI/CD 是什么，为什么前端需要它

CI/CD（Continuous Integration / Continuous Deployment）即持续集成/持续部署，是一套自动化构建、测试、部署的流水线实践。<!--more-->

### 没有 CI/CD 时前端项目的痛点

- 每次合并代码后，手动在本地 `npm run build` → 手动上传到服务器 → 手动刷新 CDN 缓存
- 多人协作时，总有代码没跑 lint 或类型检查就被合并到主干
- 测试环境、预发布环境、生产环境的构建产物不一致
- 上线前才发现某次提交破坏了构建

CI/CD 将上述流程全部自动化：**代码推送 → 自动检查 → 自动构建 → 自动部署**。

## 二、核心概念

### CI（持续集成）

每次代码变更（push / PR）时，自动触发流水线，完成以下检查：

```
代码推送 → 安装依赖 → Lint 检查 → 类型检查 → 单元测试 → 构建验证
```

CI 的核心理念是**尽早发现问题**——在代码合并到主干之前就发现 lint 错误、类型错误、测试失败或构建异常。

### CD（持续部署/持续交付）

CI 通过之后，自动将产物部署到目标环境：

- **Continuous Delivery**：构建产物准备好，但需要手动确认发布（适合大版本上线）
- **Continuous Deployment**：CI 通过后自动部署到生产（适合高频迭代）

### Pipeline（流水线）

Pipeline 是 CI/CD 的核心抽象，将流程拆解为多个阶段（stage），每个阶段包含一个或多个任务（job）：

```
.gitlab-ci.yml
stages:
  - install     ← 安装依赖
  - lint        ← 代码检查
  - test        ← 单元测试
  - build       ← 构建打包
  - deploy      ← 部署
```

## 三、前端 CI/CD 典型流程

### 完整流水线示例

```
Push / PR
  │
  ├── install       缓存 node_modules，安装依赖
  ├── lint          ESLint + Prettier 检查
  ├── type-check    TypeScript 编译检查
  ├── test          单元测试 + 测试覆盖率
  ├── build         生产构建
  ├── e2e           可选：Playwright / Cypress 端到端测试
  ├── preview       可选：自动生成 Preview 预览环境（PR 场景）
  └── deploy        部署到 dev / staging / production
```

### 分支策略与流水线设计

| 分支 | 触发事件 | 流水线内容 | 目标环境 |
|------|----------|------------|----------|
| `feature/*` | Push | lint + type-check + test + build | 无 |
| PR → `main` | PR 创建/更新 | lint + type-check + test + build + preview | Preview |
| `main` | Merge | lint + type-check + test + build + deploy | Staging |
| `release/*` | Push tag | build + deploy | Production |

## 四、GitHub Actions 实战

### 基础结构

```yaml
name: Frontend CI/CD

on:
  push:
    branches: [main, 'release/*']
  pull_request:
    branches: [main]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'pnpm'
      - run: pnpm install
      - run: pnpm lint

  test:
    runs-on: ubuntu-latest
    needs: lint          # lint 通过后才执行
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'pnpm'
      - run: pnpm install
      - run: pnpm test -- --coverage
      - uses: actions/upload-artifact@v4
        with:
          name: coverage
          path: coverage/

  build:
    runs-on: ubuntu-latest
    needs: test
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'pnpm'
      - run: pnpm install
      - run: pnpm build
      - uses: actions/upload-pages-artifact@v3
        with:
          path: dist/
```

### 多阶段依赖控制

`needs` 控制 job 之间的依赖关系。并行执行 lint 和 type-check，都通过后再执行 test：

```yaml
jobs:
  lint:
    ...
  type-check:
    ...
  test:
    needs: [lint, type-check]   # 等待 lint 和 type-check 都完成
  build:
    needs: test
  deploy:
    needs: build
    if: github.ref == 'refs/heads/main'
```

### 自动部署到 GitHub Pages

```yaml
name: Deploy to GitHub Pages

on:
  push:
    branches: [main]

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: pages
  cancel-in-progress: true

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'pnpm'
      - run: pnpm install
      - run: pnpm build
      - uses: actions/upload-pages-artifact@v3
        with:
          path: dist/

  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - id: deployment
        uses: actions/deploy-pages@v4
```

### 使用 Secrets 管理敏感信息

```yaml
jobs:
  deploy:
    steps:
      - run: pnpm build
        env:
          VITE_API_BASE: ${{ secrets.VITE_API_BASE }}
          VITE_APP_ID: ${{ secrets.VITE_APP_ID }}
```

在 GitHub 仓库 Settings → Secrets and variables → Actions 中配置。

### 缓存优化

```yaml
- uses: actions/setup-node@v4
  with:
    node-version: 20
    cache: 'pnpm'          # 自动缓存 .pnpm-store 和 node_modules

# 或手动缓存
- uses: actions/cache@v4
  with:
    path: |
      **/node_modules
      ~/.pnpm-store
    key: ${{ runner.os }}-pnpm-${{ hashFiles('**/pnpm-lock.yaml') }}
    restore-keys: |
      ${{ runner.os }}-pnpm-
```

## 五、GitLab CI 实战

### .gitlab-ci.yml

```yaml
image: node:20-alpine

variables:
  PNPM_VERSION: '9'

cache:
  key: ${CI_COMMIT_REF_SLUG}
  paths:
    - node_modules/
    - .pnpm-store/

stages:
  - install
  - lint
  - test
  - build
  - deploy

install:
  stage: install
  script:
    - npm install -g pnpm@$PNPM_VERSION
    - pnpm install
  artifacts:
    paths:
      - node_modules/
    expire_in: 2 hours

lint:
  stage: lint
  script:
    - pnpm lint
  needs: [install]

test:
  stage: test
  script:
    - pnpm test -- --coverage
  coverage: '/All files[^|]*\|[^|]*\s+([\d\.]+)/'
  artifacts:
    paths:
      - coverage/
    expire_in: 30 days
  needs: [install]

build:
  stage: build
  script:
    - pnpm build
  artifacts:
    paths:
      - dist/
    expire_in: 2 hours
  needs: [install]

deploy:
  stage: deploy
  script:
    - apk add --no-cache curl
    - curl -X POST -F "token=$DEPLOY_TOKEN" -F "ref=main" $DEPLOY_TRIGGER_URL
  only:
    - main
  needs: [lint, test, build]
  environment:
    name: production
    url: https://example.com
```

### Review Apps（每 PR 自动生成预览环境）

```yaml
review:
  stage: deploy
  script:
    - pnpm build
    - cp -r dist/ /srv/nginx/review/$CI_MERGE_REQUEST_IID/
    - echo "部署完成，访问 https://review-$CI_MERGE_REQUEST_IID.example.com"
  environment:
    name: review/$CI_MERGE_REQUEST_IID
    url: https://review-$CI_MERGE_REQUEST_IID.example.com
    on_stop: stop-review
  only:
    - merge_requests

stop-review:
  stage: deploy
  script:
    - rm -rf /srv/nginx/review/$CI_MERGE_REQUEST_IID/
  environment:
    action: stop
  only:
    - merge_requests
  when: manual
```

## 六、部署方案选型

### 方案对比

| 方案 | 适用场景 | 成本 | 复杂度 |
|------|----------|------|--------|
| GitHub Pages | 文档站、个人项目 | 免费 | 低 |
| Vercel / Netlify | SPA、SSG 框架 | 免费额度 | 低 |
| OSS + CDN（阿里云/S3） | 中大型生产项目 | 按量计费 | 中 |
| Docker + K8s | 微前端、SSR | 较高 | 高 |
| 自建服务器（Nginx） | 定制化需求 | 固定成本 | 高 |

### Vercel 自动部署（GitHub 集成）

Vercel 与 GitHub 集成后，每次 PR 自动生成 Preview URL，合并到 main 后自动部署生产：

```yaml
# vercel.json
{
  "framework": "vite",
  "buildCommand": "pnpm build",
  "outputDirectory": "dist",
  "installCommand": "pnpm install"
}
```

### OSS + CDN 部署脚本

```yaml
deploy-oss:
  stage: deploy
  script:
    - pnpm build
    # 阿里云 OSS 上传
    - ossutil cp -r -f dist/ oss://my-bucket/$CI_COMMIT_REF_NAME/
    # CDN 刷新
    - aliyun cdn RefreshObjectCaches --ObjectPath "https://cdn.example.com/"
  only:
    - main
```

### Docker 部署（SSR 场景）

```dockerfile
# Dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package.json pnpm-lock.yaml ./
RUN npm install -g pnpm && pnpm install
COPY . .
RUN pnpm build

FROM node:20-alpine AS runner
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/server ./server
EXPOSE 3000
CMD ["node", "server/index.js"]
```

```yaml
deploy-docker:
  stage: deploy
  script:
    - docker build -t registry.example.com/my-app:$CI_COMMIT_TAG .
    - docker push registry.example.com/my-app:$CI_COMMIT_TAG
    - kubectl set image deployment/my-app app=registry.example.com/my-app:$CI_COMMIT_TAG
  only:
    - tags
```

## 七、前端 CI/CD 最佳实践

### 1. 缓存管理

依赖安装是整个流水线中最耗时的环节，合理利用缓存可以大幅提速：

| 包管理器 | 缓存路径 | 缓存 key 策略 |
|----------|----------|---------------|
| npm | `~/.npm` | `npm-lock.yaml` hash |
| pnpm | `~/.pnpm-store` | `pnpm-lock.yaml` hash |
| yarn | `~/.yarn/cache` | `yarn.lock` hash |

配置示例：

```yaml
- uses: actions/cache@v4
  with:
    path: |
      **/node_modules
      ~/.pnpm-store
    key: ${{ runner.os }}-pnpm-${{ hashFiles('**/pnpm-lock.yaml') }}
    restore-keys: |
      ${{ runner.os }}-pnpm-
```

> 关于 `restore-keys`：如果 lock 文件发生变化，key 匹配失败时会尝试 `restore-keys`，至少恢复上一次的缓存，避免完全重新下载。

### 2. 并行与依赖关系

将不互相依赖的任务并行执行：

```yaml
jobs:
  lint:       ...
  typecheck:  ...
  test:       ...
  # lint, typecheck, test 互不依赖，可以并行
  build:
    needs: [lint, typecheck, test]   # 都通过后再构建
```

### 3. 流水线速度优化

- **分层 job**：install 作为独立 stage，产物通过 artifacts 传递给后续 job
- **仅变更时触发**：只修改了 README 时不需要跑完整流水线

```yaml
on:
  push:
    paths-ignore:
      - '**.md'
      - 'docs/**'
```

- **条件执行**：某些 job 只在特定分支运行

```yaml
deploy-prod:
  if: github.ref == 'refs/heads/main'
```

### 4. 安全性

- **绝不将密钥硬编码**：使用 CI/CD 平台的 Secrets 管理功能
- **最小权限原则**：CI token 只授予必要的仓库和操作权限
- **避免 CI 中使用 `npm publish --access public` 类似的危险操作**

```yaml
# ❌ 危险：直接写在 yaml 中
- run: curl -H "Authorization: Bearer my-secret-token" ...

# ✅ 正确：使用 Secrets
- run: curl -H "Authorization: Bearer ${{ secrets.DEPLOY_TOKEN }}" ...
```

### 5. 多环境管理

```yaml
# 通过环境变量区分
deploy-dev:
  environment: dev
  script: pnpm build --mode development && deploy-to-oss dev

deploy-staging:
  environment: staging
  script: pnpm build --mode staging && deploy-to-oss staging
  only:
    - main

deploy-prod:
  environment: production
  script: pnpm build --mode production && deploy-to-oss production
  only:
    - tags
```

### 6. Monorepo 策略

对于 Monorepo 项目（pnpm workspace / Turborepo），需要识别变更范围，只构建受影响的部分：

```yaml
# 通过 Turborepo 的 --filter 只构建变更的包
- run: pnpm turbo run build --filter=[origin/main]
```

```yaml
# 或手动判断哪些包发生了变更
- uses: dorny/paths-filter@v3
  id: changes
  with:
    filters: |
      package-a:
        - 'packages/package-a/**'
      package-b:
        - 'packages/package-b/**'

- name: Build package-a
  if: steps.changes.outputs.package-a == 'true'
  run: pnpm --filter package-a build

- name: Build package-b
  if: steps.changes.outputs.package-b == 'true'
  run: pnpm --filter package-b build
```

### 7. 通知与告警

```yaml
on-failure:
  needs: [deploy]
  if: failure()
  steps:
    - name: Send DingTalk notification
      run: |
        curl -X POST ${{ secrets.DINGTALK_WEBHOOK }} \
          -H 'Content-Type: application/json' \
          -d '{"msgtype":"text","text":{"content":"⚠️ 部署失败：${{ github.repository }} ${{ github.ref_name }}"}}'

    - name: Send Slack notification
      uses: slackapi/slack-github-action@v2
      with:
        webhook: ${{ secrets.SLACK_WEBHOOK }}
        webhook-type: incoming-webhook
        payload: |
          {"text": "❌ 部署失败：${{ github.repository }} ${{ github.ref_name }}"}
```

## 八、常见问题

### Q1: CI 中安装依赖报错，但本地没问题

可能是 lock 文件未提交、Node 版本不一致或平台差异（Windows vs Linux）。**始终提交 lock 文件**到仓库，并使用与本地一致的 Node 版本镜像。

### Q2: 流水线跑太久，影响开发效率

- 启用依赖缓存
- 只运行必要的 job（路径过滤、条件触发）
- 合理设计并行度
- 考虑使用更快的 runner（如 GitHub 的 larger runner）

### Q3: Preview 环境怎么管理

- PR 创建时自动创建预览环境
- PR 关闭/合并时自动销毁（避免资源泄漏）
- 给 Preview URL 加上权限控制，避免未授权访问

### Q4: 前端项目是否需要 E2E 测试

需要，但建议作为单独 stage 在构建后进行，并且只在特定分支（如 main）或 PR 中运行，不需要在每次 push 都跑。Playwright / Cypress 均可集成到 CI 中。

### Q5: CD 和多环境策略如何设计

建议采用 **Trunk-based Development** 配合环境晋升：

```
feature branch → CI 验证
       ↓
Merge to main → 自动部署 Staging
       ↓
打 Tag / 手动触发 → 部署 Production
```

## 九、总结

前端 CI/CD 的核心价值在于：

1. **自动化**：从代码提交到上线全流程无人干预
2. **质量门禁**：Lint / TypeScript / Test / Build 层层把关
3. **快速反馈**：每次变更几分钟内得到验证结果
4. **可追溯**：每个构建产物对应唯一 commit，回滚一键完成

推荐学习路径：

- 从 GitHub Actions 入手（文档最完善、社区生态最好）
- 掌握 `.gitlab-ci.yml` 的多阶段配置
- 理解 Docker 部署和 OSS/CDN 部署的区别
- 阅读项目已有 CI 配置文件，对照本文理解每个字段的含义
