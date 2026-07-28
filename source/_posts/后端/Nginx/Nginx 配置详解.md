---
title: Nginx 配置详解
categories: 
- 后端
tags:
- Nginx
- 反向代理
- 负载均衡
- 服务器
- 部署
---

## 一、Nginx 是什么

Nginx 是一个高性能的 HTTP 和反向代理服务器，以其高并发、低内存占用和丰富的功能特性成为目前最流行的 Web 服务器之一。常用于静态资源托管、反向代理、负载均衡、HTTPS 配置和 API 网关。<!--more-->

### Nginx vs Apache

| 特性 | Nginx | Apache |
|------|-------|--------|
| 并发模型 | 事件驱动（异步非阻塞） | 进程/线程驱动 |
| 内存占用 | 低（1 万连接约 2.5MB） | 高（每个连接一个进程） |
| 静态文件 | 极快 | 快 |
| 动态内容 | 通过代理转发（不原生支持） | 内置模块（mod_php） |
| 配置 | 简洁 | 复杂 |

## 二、安装与基本命令

```bash
# Ubuntu / Debian
sudo apt update
sudo apt install nginx -y

# CentOS / RHEL
sudo dnf install nginx -y

# macOS
brew install nginx
```

### 常用命令

```bash
nginx -t                    # 测试配置文件语法
sudo systemctl start nginx  # 启动
sudo systemctl stop nginx   # 停止
sudo systemctl restart nginx# 重启
sudo systemctl reload nginx # 重载配置（不停机）
sudo systemctl enable nginx # 开机自启
sudo systemctl status nginx # 查看状态

nginx -s reload             # 另一种重载方式
nginx -s quit               # 优雅退出
```

## 三、配置文件结构

```
/etc/nginx/
├── nginx.conf              # 主配置文件
├── sites-available/        # 可用站点配置
├── sites-enabled/          # 已启用的站点配置（软链接）
├── conf.d/                 # 额外配置片段
└── modules-enabled/        # 已启用的模块
```

### nginx.conf 基本结构

```nginx
user www-data;                     # 运行用户
worker_processes auto;             # 工作进程数（通常设为 CPU 核心数）
pid /run/nginx.pid;

events {
  worker_connections 1024;         # 每个进程最大连接数
  multi_accept on;
  use epoll;                       # Linux 高效 I/O 模型
}

http {
  include /etc/nginx/mime.types;
  default_type application/octet-stream;

  # 基础优化
  sendfile on;
  tcp_nopush on;
  tcp_nodelay on;
  keepalive_timeout 65;

  # Gzip 压缩
  gzip on;
  gzip_types text/plain text/css application/json application/javascript;

  # 日志格式
  access_log /var/log/nginx/access.log;
  error_log /var/log/nginx/error.log;

  # 站点配置
  include /etc/nginx/conf.d/*.conf;
  include /etc/nginx/sites-enabled/*;
}
```

## 四、静态文件服务

```nginx
server {
  listen 80;
  server_name example.com;
  root /var/www/html;

  index index.html;

  location / {
    try_files $uri $uri/ /index.html;  # SPA 降级
  }

  # 静态资源缓存
  location ~* \.(js|css|png|jpg|svg|ico)$ {
    expires 30d;
    add_header Cache-Control "public, immutable";
  }
}
```

## 五、反向代理

```nginx
server {
  listen 80;
  server_name api.example.com;

  location / {
    proxy_pass http://localhost:3000;                # 转发到 Node.js
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;

    # WebSocket 支持
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
  }
}
```

### 常见代理目标

| 后端 | 端口 |
|------|------|
| Node.js (Express/Koa) | 3000 |
| Nuxt.js | 3000 |
| Next.js | 3000 |
| Python (Django/Flask) | 8000 |
| Java (Spring Boot) | 8080 |
| PHP-FPM | Unix Socket |

## 六、HTTPS 配置

```nginx
server {
  listen 443 ssl http2;
  server_name example.com;

  ssl_certificate     /etc/ssl/certs/example.com.pem;
  ssl_certificate_key /etc/ssl/private/example.com.key;

  # 安全增强
  ssl_protocols TLSv1.2 TLSv1.3;
  ssl_ciphers HIGH:!aNULL:!MD5;
  ssl_prefer_server_ciphers on;
  ssl_session_cache shared:SSL:10m;
  ssl_session_timeout 10m;

  location / {
    proxy_pass http://localhost:3000;
  }
}

# HTTP → HTTPS 重定向
server {
  listen 80;
  server_name example.com;
  return 301 https://$host$request_uri;
}
```

使用 Let's Encrypt 免费证书：

```bash
sudo apt install certbot python3-certbot-nginx
sudo certbot --nginx -d example.com
```

## 七、负载均衡

```nginx
upstream backend {
  # 负载均衡策略
  # 默认：轮询（round-robin）
  # least_conn：最少连接
  # ip_hash：IP 哈希（会话保持）

  server 192.168.1.10:3000 weight=3;   # weight 越大，权重越高
  server 192.168.1.11:3000 weight=2;
  server 192.168.1.12:3000 backup;     # 备用服务器
}

server {
  listen 80;
  server_name app.example.com;

  location / {
    proxy_pass http://backend;
    proxy_set_header Host $host;
  }
}
```

## 八、URL 重写与重定向

```nginx
server {
  # 301 永久重定向（SEO 迁移）
  location /old-page {
    return 301 /new-page;
  }

  # 302 临时重定向
  location /promo {
    return 302 https://another-site.com;
  }

  # rewrite 规则
  rewrite ^/articles/(\d+)$ /post?id=$1 permanent;

  # 禁止访问隐藏文件
  location ~ /\. {
    deny all;
  }

  # 指定文件类型拒绝
  location ~* \.(env|git|log|sql)$ {
    deny all;
  }
}
```

## 九、安全配置

```nginx
server {
  # 限制请求速率
  limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;
  location /api/ {
    limit_req zone=api burst=20 nodelay;
    proxy_pass http://localhost:3000;
  }

  # IP 白名单
  location /admin {
    allow 192.168.1.0/24;
    deny all;
  }

  # 限制请求体大小
  client_max_body_size 10m;

  # 隐藏 Nginx 版本号
  server_tokens off;

  # 安全头
  add_header X-Frame-Options "SAMEORIGIN" always;
  add_header X-Content-Type-Options "nosniff" always;
  add_header X-XSS-Protection "1; mode=block" always;
}
```

## 十、日志

```nginx
# 自定义日志格式
log_format json escape=json '{'
  '"time":"$time_iso8601",'
  '"remote_addr":"$remote_addr",'
  '"request":"$request",'
  '"status":$status,'
  '"body_bytes":$body_bytes_sent,'
  '"request_time":$request_time,'
  '"upstream_addr":"$upstream_addr",'
  '"http_referer":"$http_referer",'
  '"http_user_agent":"$http_user_agent"'
 '}';

access_log /var/log/nginx/access.log json;
error_log /var/log/nginx/error.log warn;
```

## 十一、性能调优

```nginx
# nginx.conf
worker_processes auto;
worker_rlimit_nofile 65535;

events {
  worker_connections 4096;
  use epoll;
  multi_accept on;
}

http {
  sendfile on;
  tcp_nopush on;
  tcp_nodelay on;

  # 缓冲
  proxy_buffering on;
  proxy_buffer_size 4k;
  proxy_buffers 8 4k;

  # 超时
  keepalive_timeout 65;
  proxy_connect_timeout 10;
  proxy_read_timeout 30;

  # 静态文件缓存
  open_file_cache max=2000 inactive=20s;
  open_file_cache_valid 60s;
  open_file_cache_min_uses 2;
}
```

## 十二、常见问题

### Q1: 配置修改后未生效

```bash
nginx -t              # 检查语法
sudo systemctl reload nginx  # 重载（不要用 restart）
```

### Q2: 403 Forbidden

```bash
# 原因：权限不足
# 确保 nginx 用户有目录读权限
sudo chown -R www-data:www-data /var/www/html
sudo chmod -R 755 /var/www/html
```

### Q3: 502 Bad Gateway

```bash
# 原因：上游服务器未启动
systemctl status node-server
# 或检查端口
ss -tlnp | grep 3000
```

### Q4: 反向代理获取不到真实客户端 IP

```nginx
# 在代理层设置
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
```

### Q5: 端口转发

```nginx
# 将 80 端口转发到内部端口
server {
  listen 80;
  server_name example.com;
  location / {
    proxy_pass http://localhost:3000;
  }
}
```

## 十三、推荐学习路径

1. 掌握静态文件服务和反向代理配置
2. 配置 HTTPS（Let's Encrypt）
3. 理解 location 匹配规则和 rewrite
4. 配置负载均衡和速率限制
5. 调优 worker 数量和缓存策略
6. 阅读 `/var/log/nginx/access.log` 排查问题
