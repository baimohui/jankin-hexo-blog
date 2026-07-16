---
title: 部署与运维入门 — Docker / CI-CD / Nginx
categories: 
- 后端
tags:
- 部署
- Docker
- CI/CD
- Nginx
- 运维
---

## 从开发到上线

作为前端，你平时开发时 `npm run dev` 就完事了，但上线部署到服务器需要理解全链路：

```text
本地开发
  ↓ git push
代码仓库（GitHub/GitLab）
  ↓ 触发 CI
CI/CD 流水线（自动化）
  ↓ 构建、测试
构建产物（镜像 / 静态文件）
  ↓ 部署
服务器（云服务器 / 容器平台）
  ↓
通过域名访问
```

## Docker — 让应用"打包带走"

### 为什么需要 Docker？

```text
没有 Docker 前：
"在我电脑上能跑啊！"
→ 环境不同（系统、Node 版本、数据库版本...）

有了 Docker：
"打包成一个镜像，到处都能跑"
→ 开发/测试/生产环境一致
```

### Docker 核心概念

```text
Dockerfile（菜谱）→ 构建 → Image（打包好的镜像）→ 运行 → Container（实际运行的实例）
```

```dockerfile
# 一个 Node.js 应用的 Dockerfile
FROM node:20-alpine           # 基于 Node 的轻量基础镜像
WORKDIR /app                  # 工作目录
COPY package.json ./          # 先复制依赖描述文件
RUN npm install               # 安装依赖（利用缓存优化）
COPY . .                      # 复制源码
EXPOSE 3000                   # 暴露端口
CMD ["node", "server.js"]     # 启动命令
```

### 前端视角的 Docker

```dockerfile
# 前端项目的 Dockerfile — 多阶段构建
# 阶段 1: 构建
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# 阶段 2: 用 Nginx 跑静态文件
FROM nginx:alpine
COPY --from=builder /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
```

**前端同学只需要知道：** 用 Docker 部署前端 = 先用 Node 镜像构建 → 把构建产物扔到 Nginx 镜像里。

## CI/CD — 自动化流水线

| 阶段 | 做什么 | 前端示例 |
|---|---|---|
| **CI（持续集成）** | 每次 push 自动跑测试、lint、构建 | `npm run lint` → `npm run test` → `npm run build` |
| **CD（持续部署）** | CI 通过后自动部署到服务器 | 构建产物上传 CDN / 部署到服务器 |

```yaml
# GitHub Actions 示例 — 前端项目
name: Deploy
on:
  push:
    branches: [main]
jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
      - run: npm ci && npm run build
      - uses: peaceiris/actions-gh-pages@v3  # 部署到 GitHub Pages
        with:
          publish_dir: ./dist
```

**基础理解：** CI/CD = 把"手动操作"变成"自动化"。不需要自己登录服务器、手动拖文件、手动重启——push 代码后一切自动完成。

## Nginx 反向代理

Nginx 在前端部署和全栈开发中非常常见。**反向代理**是它的核心用途之一：

```text
用户访问 example.com
  ↓ DNS → 指向服务器
  ↓
Nginx（监听 80/443 端口）
  ├─ 静态文件 → /var/www/html（前端）
  ├─ /api/*    → 转发到 Node.js 服务（后端）
  └─ WebSocket → 转发到应用服务器
```

```nginx
server {
    listen 80;
    server_name example.com;

    # 前端静态文件
    root /var/www/html;
    index index.html;

    # 前端路由（SPA 需要，否则刷新 404）
    location / {
        try_files $uri $uri/ /index.html;
    }

    # 反向代理到后端 API
    location /api/ {
        proxy_pass http://localhost:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

## Nginx 常见场景速查

| 场景 | 配置要点 |
|---|---|
| 部署 SPA | `try_files $uri $uri/ /index.html` 解决刷新 404 |
| 跨域 | `add_header Access-Control-Allow-Origin *` |
| HTTPS | `listen 443 ssl; ssl_certificate` + certbot 自动续期 |
| 负载均衡 | `upstream backend { server 192.168.1.1; server 192.168.1.2; }` |
| 限流 | `limit_req_zone` 防止恶意请求 |
| Gzip | `gzip on; gzip_types text/css application/javascript` |

## 后端运维入门清单

- [ ] 会起一台云服务器（阿里云/腾讯云/AWS）
- [ ] 会用 SSH 登录服务器
- [ ] 会装 Node.js、Nginx、MySQL
- [ ] 会写一个 Dockerfile 部署自己的应用
- [ ] 会配 GitHub Actions 自动部署
- [ ] 知道 HTTPS 和证书是什么、怎么配
- [ ] 知道基本的"看日志"命令（`journalctl`、`tail -f`）
