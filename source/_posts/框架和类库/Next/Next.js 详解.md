---
title: Next.js 详解
categories: 
- 框架和类库
tags:
- Next
- React
- SSR
- 服务端渲染
- SEO
---

## 一、Next.js 是什么

Next.js 是一个基于 React 的全栈 Web 应用框架，由 Vercel 开发和维护。它提供了服务端渲染（SSR）、静态生成（SSG）、API 路由、文件系统路由等开箱即用的能力，是目前 React 生态中最流行的生产级框架。<!--more-->

### 为什么需要 Next.js

- **SEO 优化**：React SPA 的 HTML 只有一个空根节点，搜索引擎无法抓取内容
- **首屏性能**：服务端生成完整 HTML，用户无需等待 JS 加载即可看到内容
- **全栈能力**：同一项目中编写前端页面和后端 API，无需单独搭建服务器
- **开发体验**：文件路由、自动代码分割、HMR 等

### Next.js vs Nuxt.js

| 维度 | Next.js | Nuxt.js |
|------|---------|---------|
| 底层 | React | Vue 3 |
| 路由 | 文件系统路由（App Router / Pages Router） | 文件系统路由 |
| 渲染 | SSR、SSG、ISR、SPA | SSR、SSG、SPA |
| API 路由 | 内置（`/api/*`） | 内置（`/api/*`） |
| 部署 | Vercel（原生）、自托管 | Node 服务器、静态托管 |
| 社区生态 | 最大 | 较小 |

## 二、快速开始

```bash
npx create-next-app@latest my-app --typescript
cd my-app
npm run dev
```

### 项目结构（App Router）

```
app/
├── layout.tsx          # 根布局
├── page.tsx            # /
├── about/
│   └── page.tsx        # /about
├── blog/
│   ├── [slug]/
│   │   └── page.tsx    # /blog/:slug
│   └── page.tsx        # /blog
├── api/
│   └── users/
│       └── route.ts    # /api/users
├── loading.tsx         # 加载 UI
├── error.tsx           # 错误 UI
├── not-found.tsx       # 404 UI
├── globals.css         # 全局样式
├── page.module.css     # CSS Module
```

## 三、路由系统

Next.js 提供两种路由方式：**App Router**（推荐，React Server Components）和 **Pages Router**（传统方式）。

### App Router（推荐）

文件系统自动映射路由：

```text
app/page.tsx            →  /
app/about/page.tsx      →  /about
app/blog/[slug]/page.tsx→  /blog/:slug
app/blog/[...catchAll]/page.tsx → /blog/a/b/c（通配）
```

### 页面组件

```tsx
// app/page.tsx
export default function Home() {
  return <h1>Hello Next.js</h1>;
}
```

### 布局（Layout）

Layout 在页面切换时保持状态，不会重新渲染：

```tsx
// app/layout.tsx
export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="zh-CN">
      <body>
        <header>导航栏</header>
        <main>{children}</main>
        <footer>底部</footer>
      </body>
    </html>
  );
}
```

### 嵌套布局

```tsx
// app/blog/layout.tsx（仅 blog 下的页面使用）
export default function BlogLayout({ children }: { children: React.ReactNode }) {
  return (
    <div className="blog-container">
      <aside>侧边栏</aside>
      <article>{children}</article>
    </div>
  );
}
```

### 链接与导航

```tsx
import Link from 'next/link';
import { useRouter } from 'next/navigation';

export default function Nav() {
  const router = useRouter();

  return (
    <nav>
      <Link href="/about">关于</Link>
      <Link href={`/blog/${slug}`}>文章详情</Link>
      <button onClick={() => router.push('/login')}>跳转</button>
    </nav>
  );
}
```

## 四、渲染策略

### 服务端组件（Server Components，默认）

App Router 中的组件**默认是服务端组件**——它们在服务器上渲染，不会发送 JS 到客户端：

```tsx
// app/page.tsx（服务端组件）
export default async function Home() {
  const data = await fetch('https://api.example.com/data'); // 只会在服务端执行
  const posts = await data.json();

  return (
    <ul>
      {posts.map(p => <li key={p.id}>{p.title}</li>)}
    </ul>
  );
}
```

### 客户端组件（Client Components）

需要交互（useState、useEffect、onClick）时，在文件顶部声明 `'use client'`：

```tsx
'use client';

import { useState } from 'react';

export default function Counter() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
}
```

### 静态生成（SSG）

```tsx
// app/page.tsx
export const dynamic = 'force-static';
// 构建时生成静态 HTML，适用于博客、文档等不常变的内容
```

### 增量静态生成（ISR）

在一定时间间隔后重新生成页面：

```tsx
export const revalidate = 60;   // 每 60 秒重新生成一次
```

## 五、数据获取

### 服务端 Fetch（推荐）

```tsx
// 服务端组件中直接使用 fetch（Next.js 扩展了原生 fetch）
async function getPost(slug: string) {
  const res = await fetch(`https://api.example.com/posts/${slug}`, {
    next: { revalidate: 60 },  // ISR：60 秒内复用缓存
  });
  return res.json();
}

export default async function Post({ params }: { params: { slug: string } }) {
  const post = await getPost(params.slug);
  return <article>{post.title}</article>;
}
```

### 客户端数据获取

```tsx
'use client';

import useSWR from 'swr';

export default function UserList() {
  const { data, error, isLoading } = useSWR('/api/users', fetcher);

  if (isLoading) return <div>加载中...</div>;
  return <div>{data?.map(u => <div key={u.id}>{u.name}</div>)}</div>;
}
```

## 六、API 路由

在 `app/api/` 目录中创建 `route.ts` 即可定义 API 端点：

```ts
// app/api/users/route.ts
import { NextResponse } from 'next/server';

export async function GET() {
  const users = await db.users.findMany();
  return NextResponse.json(users);
}

export async function POST(request: Request) {
  const body = await request.json();
  const user = await db.users.create({ data: body });
  return NextResponse.json(user, { status: 201 });
}
```

```ts
// app/api/users/[id]/route.ts
export async function GET(request: Request, { params }: { params: { id: string } }) {
  const user = await db.users.findUnique({ where: { id: params.id } });
  if (!user) return NextResponse.json({ error: 'Not found' }, { status: 404 });
  return NextResponse.json(user);
}
```

## 七、动态元数据（SEO）

```ts
// app/blog/[slug]/page.tsx
import type { Metadata } from 'next';

export async function generateMetadata({ params }: { params: { slug: string } }): Promise<Metadata> {
  const post = await getPost(params.slug);

  return {
    title: post.title,
    description: post.summary,
    openGraph: {
      title: post.title,
      images: [post.coverImage],
    },
  };
}
```

## 八、中间件

中间件在每次请求时执行，用于重定向、鉴权等：

```ts
// middleware.ts
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';

export function middleware(request: NextRequest) {
  const token = request.cookies.get('token');

  if (!token && request.nextUrl.pathname.startsWith('/dashboard')) {
    return NextResponse.redirect(new URL('/login', request.url));
  }

  return NextResponse.next();
}

export const config = {
  matcher: '/dashboard/:path*',  // 仅匹配 /dashboard 路径
};
```

## 九、样式方案

### CSS Module

```tsx
// app/page.tsx
import styles from './page.module.css';
export default function Page() {
  return <div className={styles.container}>样式隔离</div>;
}
```

### Tailwind CSS（内置支持）

```tsx
export default function Page() {
  return <div className="flex items-center justify-center min-h-screen bg-gray-100">Tailwind</div>;
}
```

## 十、部署

### Vercel（推荐）

```bash
git push
# Vercel 自动检测 Next.js，配置零
```

### 自托管

```bash
npm run build
node .next/standalone/server.js    # 需要在 next.config 中配置 output: 'standalone'
```

## 十一、常见问题

### Q1: 'use client' 和默认服务端组件怎么选

默认用服务端组件（性能更好、更安全）。只有需要交互（useState、事件监听、浏览器 API）时才加 `'use client'`。

### Q2: 图片优化

```tsx
import Image from 'next/image';

<Image
  src="/hero.jpg"
  alt="Hero"
  width={800}
  height={600}
  priority           // 首屏图片优先加载
  placeholder="blur" // 模糊占位
/>
```

### Q3: 如何连接数据库

在服务端组件或 API Route 中直接使用：

```ts
import { PrismaClient } from '@prisma/client';
const prisma = new PrismaClient();

export async function GET() {
  return NextResponse.json(await prisma.user.findMany());
}
```

### Q4: App Router 和 Pages Router 怎么选

新项目一律用 App Router（Next.js 13+）。Pages Router 仅在维护旧项目时使用。

## 十二、推荐学习路径

1. 理解 App Router 的文件系统路由和 Layout 嵌套
2. 掌握服务端组件与客户端组件的区别
3. 使用 `fetch` 在服务端组件中获取数据
4. 编写 API Route 实现后端逻辑
5. 配置动态 metadata 做 SEO
6. 学习中间件、ISR、图片优化
7. 部署到 Vercel
