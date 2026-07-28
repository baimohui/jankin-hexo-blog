---
title: SaaS 详解（前端视角）
categories: 
- 前端工程化
tags:
- SaaS
- 多租户
- 架构
- B2B
---

## 一、什么是 SaaS

SaaS（Software as a Service）是一种软件交付模式：软件由服务商托管，用户通过浏览器访问，按需付费，无需自行部署和维护服务器。<!--more-->

### 传统软件 vs SaaS

| 维度 | 传统软件 | SaaS |
|------|----------|------|
| 部署 | 客户自行安装部署 | 服务商托管，浏览器访问 |
| 更新 | 客户手动升级 | 服务商持续发布，用户无感知 |
| 付费 | 一次性买断 + 维护费 | 按年/按月订阅 |
| 运维 | 客户自建 IT 团队 | 服务商负责 |
| 定制 | 深度定制，独立分支 | 配置化，共享一套代码 |

### SaaS 的典型特征

```text
├── 多租户（Multi-tenant）：一套代码服务所有客户
├── 配置化（Configurable）：通过配置而非改代码实现个性化
├── 按需付费：按功能/用量/用户数收费
├── 自动扩容：云原生架构，弹性伸缩
└── 持续交付：每周/每日发布，用户无感升级
```

## 二、多租户架构

多租户是 SaaS 的核心——**一套代码服务多个客户（租户）**，隔离方式有三种：

| 模式 | 数据隔离 | 成本 | 复杂度 | 适用场景 |
|------|----------|------|--------|----------|
| **独立数据库** | 每个租户独立 DB | 高 | 低 | 金融、医疗等强隔离需求 |
| **共享数据库、独立 Schema** | 同一 DB，不同表前缀 | 中 | 中 | 中型 SaaS |
| **共享数据库、共享表** | 同一表，通过 `tenant_id` 区分 | 低 | 高 | 小型 SaaS，成本敏感 |

### 前端视角：多租户的处理

```ts
// 前端通过请求头标识租户
const API_BASE = import.meta.env.VITE_API_BASE;

// 登录时获取租户信息
async function login(tenantCode: string, username: string, password: string) {
  const res = await fetch(`${API_BASE}/auth/login`, {
    method: 'POST',
    headers: {
      'X-Tenant-Code': tenantCode,  // 租户标识
    },
    body: JSON.stringify({ username, password }),
  });
  // ...
}

// Axios 拦截器统一注入租户
api.interceptors.request.use(config => {
  config.headers['X-Tenant-Code'] = tenantStore.currentTenant?.code;
  return config;
});
```

## 三、前端在 SaaS 中的关键职责

### 1. 租户路由与自定义域名

```text
方案 A：子域名区分租户
  tenant-a.example.com
  tenant-b.example.com
  → 前端通过 window.location.hostname 获取租户

方案 B：路径区分租户
  example.com/tenant-a/dashboard
  example.com/tenant-b/dashboard
  → 前端路由第一级为租户参数

方案 C：自定义域名（白标）
  客户用自己的域名 crm.my-company.com 指向 SaaS 平台
  → 需要动态 SSL + 路由匹配
```

```ts
// 子域名方案：从 URL 解析租户
function getTenantFromHostname() {
  const parts = window.location.hostname.split('.');
  if (parts.length >= 3) {
    return parts[0];  // tenant-a.example.com → "tenant-a"
  }
  return 'default';
}
```

### 2. 多租户主题（白标）

每个租户可能有自己的品牌色、Logo、域名。前端需要支持**运行时主题切换**：

```ts
// 主题配置按租户加载
interface TenantTheme {
  primaryColor: string;
  logo: string;
  favicon: string;
  companyName: string;
}

const themeMap: Record<string, TenantTheme> = {
  'tenant-a': {
    primaryColor: '#409eff',
    logo: '/logos/tenant-a.png',
    favicon: '/favicons/tenant-a.ico',
    companyName: '公司 A',
  },
  'tenant-b': {
    primaryColor: '#67c23a',
    logo: '/logos/tenant-b.png',
    favicon: '/favicons/tenant-b.ico',
    companyName: '公司 B',
  },
};

// 运行时切换 CSS 变量（无需重新构建）
function applyTheme(theme: TenantTheme) {
  document.documentElement.style.setProperty('--primary', theme.primaryColor);
  document.getElementById('app-logo')!.src = theme.logo;
  document.getElementById('favicon')!.href = theme.favicon;
  document.title = theme.companyName;
}
```

### 3. 功能开关（Feature Flags）

不同套餐的用户看到不同的功能：

```ts
// 从后端获取当前租户的功能开关
interface FeatureFlags {
  advancedAnalytics: boolean;
  exportPDF: boolean;
  apiAccess: boolean;
  customDomain: boolean;
}

// 前端做条件渲染
function FeatureGate({ feature, children }: {
  feature: keyof FeatureFlags;
  children: React.ReactNode;
}) {
  const flags = useFeatureFlags();
  if (!flags[feature]) return null;
  return <>{children}</>;
}

// 使用
<FeatureGate feature="advancedAnalytics">
  <AnalyticsDashboard />
</FeatureGate>
```

### 4. 套餐与配额管理

```ts
interface Quota {
  maxUsers: number;
  maxStorageGB: number;
  apiCallsPerDay: number;
}

// 前端实时展示用量
function UsageBar({ used, quota }: { used: number; quota: number }) {
  const percent = (used / quota) * 100;
  return (
    <div>
      <div style={{ width: `${Math.min(percent, 100)}%` }} />
      <span>{used} / {quota}（quota === Infinity ? '不限' : quota）</span>
    </div>
  );
}
```

### 5. 订阅与支付

```ts
// 前端处理订阅状态
type SubscriptionStatus = 'active' | 'trialing' | 'past_due' | 'canceled' | 'unpaid';

interface Subscription {
  plan: 'starter' | 'business' | 'enterprise';
  status: SubscriptionStatus;
  currentPeriodEnd: number;
}

// 续费相关 UI 提示
function SubscriptionBanner({ sub }: { sub: Subscription }) {
  if (sub.status === 'past_due') {
    return <Alert type="warning">您的账单已逾期，请在 7 天内续费</Alert>;
  }
  if (sub.status === 'canceled') {
    return <Alert type="info">服务将在 {formatDate(sub.currentPeriodEnd)} 后停止</Alert>;
  }
  return null;
}
```

## 四、SaaS 前端的架构挑战

### 1. 隔离性

```text
同一套代码服务多个租户，需确保：
├── 数据隔离：A 租户的用户看不到 B 租户的数据
├── 路由隔离：/tenant-a/dashboard ≠ /tenant-b/dashboard
├── 存储隔离：localStorage 需加租户前缀
└── 缓存隔离：CDN 缓存需按租户区分
```

```ts
// localStorage 隔离
function getStorageKey(key: string) {
  return `${currentTenant.code}:${key}`;
}

localStorage.setItem(getStorageKey('theme'), 'dark');
localStorage.getItem(getStorageKey('theme'));
```

### 2. 可配置性

```text
├── 品牌配置：Logo、颜色、域名（运行时配置，非构建时）
├── 功能配置：不同套餐不同功能集合
├── 流程配置：审批流、工作流可自定义
└── 权限配置：RBAC 角色权限按租户自定义
```

### 3. 灰度发布

```ts
// 按租户 ID 做灰度
function isGrayRelease(tenantId: string, percent = 10): boolean {
  // 基于租户 ID 的哈希决定是否命中灰度
  const hash = hashCode(tenantId) % 100;
  return hash < percent;
}

// 灰度功能
const showNewFeature = isGrayRelease(tenantId, 20);
if (showNewFeature) {
  renderNewDashboard();
} else {
  renderOldDashboard();
}
```

### 4. 性能

```text
├── 首屏优化：SaaS 可能接入上百个租户的域名，CDN 缓存策略需合理
├── 代码分割：按套餐/功能拆分 JS，避免一次性加载全部代码
├── 数据预取：根据租户角色预加载常用数据
└── 骨架屏：多租户场景下等待时间长，骨架屏提升感知体验
```

## 五、SaaS 常见前端问题

### Q1: 租户切换时需要做什么

```text
├── 清除当前租户的本地缓存（localStorage、IndexedDB）
├── 重置全局状态（用户信息、权限、主题）
├── 跳转到新租户的首页
├── 重新请求新租户的配置
└── 如有 WebSocket，断开后重新连接
```

### Q2: 每个租户的构建产物是同一份吗

是。SaaS 的核心就是**一套代码服务所有客户**。差异化通过**运行时配置**实现，而非构建时分支。不同租户访问的是同一份静态资源。

### Q3: 如何做自定义域名

```text
1. 客户将自己的域名（如 crm.my-company.com）CNAME 到 SaaS 平台
2. 平台根据域名解析出租户
3. 平台自动申请 SSL 证书（Let's Encrypt）
4. 前端通过 hostname 匹配租户配置
```

### Q4: 前端缓存导致 A 租户看到 B 租户的数据

```text
原因：localStorage 未按租户隔离
解决：所有存储 key 加租户前缀（tenantId:key）

原因：Service Worker 缓存了跨租户的数据
解决：SW 缓存 key 包含租户标识

原因：CDN 缓存了租户特定的 API 响应
解决：API 响应头设置 Vary: X-Tenant-Code
```
