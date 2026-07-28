---
title: Linux 系统与常用命令详解
categories: 
- 操作系统
tags:
- Linux
- 命令行
- Shell
- 服务器
---

## 一、Linux 是什么

Linux 是一个开源的类 Unix 操作系统内核，由 Linus Torvalds 于 1991 年创建。它广泛应用于服务器、嵌入式设备、超级计算机和移动端（Android），是后端开发和 DevOps 的核心环境。<!--more-->

### 前端工程师为什么需要 Linux

- **服务器环境**：生产环境 90%+ 运行在 Linux 上
- **CI/CD 流水线**：GitHub Actions、GitLab Runner 默认 Linux 环境
- **命令行工具**：npm/node、git、Docker 在 Linux 上表现最佳
- **SSH 运维**：排查线上问题、查看日志、部署代码

## 二、目录结构

```
/                 根目录
├── bin/           系统命令（ls、cp、mv）
├── sbin/          系统管理命令（reboot、fdisk）
├── etc/           配置文件（nginx.conf、hosts）
├── var/           可变数据（日志 /var/log、缓存）
├── usr/           用户程序（/usr/local 常用来安装软件）
├── home/          用户主目录（/home/ubuntu）
├── root/          root 用户主目录
├── tmp/           临时文件（重启后清除）
├── opt/           可选软件包
├── proc/          进程信息（虚拟文件系统）
└── dev/           设备文件
```

## 三、文件操作

### 查看与导航

```bash
pwd                              # 当前路径
ls                               # 列出文件
ls -l                            # 详细列表（权限、大小、时间）
ls -a                            # 包含隐藏文件（.开头）
ls -lh                           # 可读大小（KB/MB）
ls -lt                           # 按时间排序
ls -la /var/log                  # 查看指定目录

cd /home/ubuntu                  # 切换到指定目录
cd ~                             # 切换到 home
cd -                             # 切换到上一个目录
```

### 创建与删除

```bash
touch file.txt                   # 创建空文件
mkdir dir                        # 创建目录
mkdir -p a/b/c                   # 递归创建目录

cp file.txt /tmp/                # 复制文件
cp -r dir/ /tmp/                 # 递归复制目录
cp file.txt file_backup.txt      # 同目录复制

mv file.txt /tmp/                # 移动文件
mv old.txt new.txt               # 重命名文件

rm file.txt                      # 删除文件
rm -rf dir/                      # 递归强制删除目录（谨慎！）
```

### 查看文件内容

```bash
cat file.txt                     # 显示全部内容
less file.txt                    # 分页查看（q 退出，/搜索，n 下一个）
head -20 file.txt                # 前 20 行
tail -50 file.txt                # 后 50 行
tail -f app.log                  # 实时跟踪日志（Ctrl+C 退出）

wc -l file.txt                   # 行数
wc -w file.txt                   # 单词数
```

### 搜索

```bash
# grep 搜索文件内容
grep "error" app.log             # 搜索包含 error 的行
grep -i "error" app.log          # 忽略大小写
grep -r "TODO" src/              # 递归搜索目录
grep -n "error" app.log          # 显示行号
grep -c "error" app.log          # 统计匹配行数

# find 搜索文件
find . -name "*.js"              # 查找 .js 文件
find /var -size +100M            # 查找大于 100MB 的文件
find . -mtime -1                 # 最近 1 天修改的文件
```

### 解压缩

```bash
tar -czvf archive.tar.gz dir/    # 压缩目录
tar -xzvf archive.tar.gz         # 解压 .tar.gz
unzip file.zip                   # 解压 .zip
zip -r archive.zip dir/          # 压缩为 .zip
```

## 四、文件权限

### 权限结构

```bash
ls -l                            # 查看权限
-rw-r--r--  1  root  root  1024  Jan 1 12:00  file.txt
│└┬┘└┬┘└┬┘
│ │  │  └ 其他用户权限（r-- = 可读）
│ │  └─── 同组用户权限（r-- = 可读）
│ └────── 所有者权限（rw- = 可读写）
└──────── 文件类型（- 普通文件，d 目录，l 符号链接）
```

### 修改权限（chmod）

```bash
chmod 755 script.sh              # rwxr-xr-x（所有者读写执行，其他读执行）
chmod +x script.sh               # 添加执行权限
chmod -R 644 public/             # 递归设置目录下所有文件
```

常见权限值：

| 权限 | 数字 | 说明 |
|------|------|------|
| `rwx` | 7 | 可读、可写、可执行（程序/脚本） |
| `rw-` | 6 | 可读可写（配置文件） |
| `r-x` | 5 | 可读可执行（公共程序） |
| `r--` | 4 | 只读 |

### 修改所有者（chown）

```bash
chown user:group file.txt        # 修改所有者和组
chown -R ubuntu:ubuntu /app/     # 递归修改
```

## 五、进程管理

```bash
ps aux                           # 查看所有进程
ps aux | grep nginx              # 搜索 nginx 进程
top                              # 实时进程监控（q 退出）
htop                             # 增强版 top（需安装）
kill 1234                        # 结束进程（PID）
kill -9 1234                     # 强制结束
kill -15 1234                    # 优雅结束（默认 SIGTERM）

# 后台运行
nohup node server.js &           # 后台运行，退出终端不停止
jobs                             # 查看后台任务
fg %1                            # 将后台任务 1 切到前台
```

## 六、网络相关

```bash
# 网络连接与端口
curl https://example.com         # HTTP 请求
curl -I https://example.com      # 只查看响应头
wget https://example.com/file    # 下载文件

ping -c 4 google.com             # Ping 测试（4 次）
ss -tlnp                         # 查看监听端口（替代 netstat）
ss -tunap                        # 查看所有连接

# DNS 解析
nslookup example.com             # DNS 查询
dig example.com                  # 更详细的 DNS 查询

# SSH 连接
ssh user@192.168.1.100           # 远程登录
ssh -i key.pem user@host         # 使用密钥登录
scp file.txt user@host:/tmp/     # 复制文件到远程
scp user@host:/remote/file ./    # 从远程复制到本地
```

## 七、磁盘与内存

```bash
df -h                            # 磁盘空间使用情况
du -sh /var/log/                 # 查看目录总大小
du -h --max-depth=1              # 查看当前目录下各子目录大小

free -h                          # 内存使用情况
uname -a                         # 系统内核信息
lscpu                            # CPU 信息
```

## 八、包管理

### Debian/Ubuntu（apt）

```bash
sudo apt update                  # 更新软件源
sudo apt upgrade                 # 升级所有已安装包
sudo apt install nginx           # 安装 nginx
sudo apt remove nginx            # 卸载
apt search keyword               # 搜索包

# 查看已安装
dpkg -l | grep nginx
dpkg -L nginx                    # 查看 nginx 安装了哪些文件
```

### CentOS/RHEL（yum/dnf）

```bash
sudo dnf update                  # 更新
sudo dnf install nginx           # 安装
sudo dnf remove nginx            # 卸载
```

### Node.js 相关（nvm）

```bash
nvm install 20                   # 安装 Node 20
nvm use 20                       # 切换到 Node 20
nvm ls                           # 查看已安装版本
```

## 九、Shell 技巧

### 管道与重定向

```bash
command1 | command2              # 管道：将 command1 的输出传给 command2

ls -la > output.txt              # 输出重定向（覆盖写入）
ls -la >> output.txt             # 输出重定向（追加写入）
cat < input.txt                  # 输入重定向

# 常见组合
ps aux | grep node               # 查找 Node 进程
cat access.log | grep "500" | wc -l  # 统计 500 错误数
ls -la | sort -k5 -n             # 按文件大小排序
```

### 文本处理三剑客

```bash
# grep（搜索）
grep "ERROR" app.log

# sed（替换）
sed 's/foo/bar/g' file.txt       # 将 foo 替换为 bar
sed -i 's/foo/bar/g' file.txt    # 原地替换

# awk（格式化输出）
awk '{print $1, $4}' access.log  # 打印第 1 和第 4 列
awk '{sum+=$NF} END {print sum}' # 对最后一列求和
awk '$9 == 500 {count++} END {print count}' access.log  # 统计 500 数
```

### 环境变量

```bash
echo $PATH                       # 查看 PATH
export MY_VAR="hello"            # 设置环境变量
echo $MY_VAR                     # hello

# 配置文件
~/.bashrc                        # bash 配置（每次打开终端加载）
~/.bash_profile                  # 登录 shell 加载
~/.zshrc                         # zsh 配置（macOS 默认）
```

### 别名

```bash
alias ll='ls -la'                # 设置别名
alias ..='cd ..'
unalias ll                       # 取消别名

# 写入 ~/.bashrc 可永久生效
echo "alias ll='ls -la'" >> ~/.bashrc
```

### 快捷键

| 快捷键 | 作用 |
|--------|------|
| `Ctrl + C` | 终止当前命令 |
| `Ctrl + D` | 退出终端 |
| `Ctrl + Z` | 暂停命令（fg 恢复） |
| `Ctrl + R` | 搜索历史命令 |
| `Ctrl + A` | 光标移到行首 |
| `Ctrl + E` | 光标移到行尾 |
| `Ctrl + U` | 删除光标前的内容 |
| `Ctrl + K` | 删除光标后的内容 |
| `Ctrl + W` | 删除前一个单词 |
| `Tab` | 自动补全文件名/命令 |

## 十、Vim 基础

```bash
vim file.txt                     # 打开文件
```

| 模式 | 操作 | 说明 |
|------|------|------|
| 普通模式 | `i` | 进入插入模式 |
| 普通模式 | `dd` | 删除当前行 |
| 普通模式 | `yy` | 复制当前行 |
| 普通模式 | `p` | 粘贴 |
| 普通模式 | `u` | 撤销 |
| 普通模式 | `:w` | 保存 |
| 普通模式 | `:q` | 退出 |
| 普通模式 | `:wq` | 保存并退出 |
| 普通模式 | `:q!` | 不保存强制退出 |
| 普通模式 | `/keyword` | 搜索 keyword |
| 普通模式 | `n` / `N` | 下一个/上一个搜索结果 |

## 十一、实战场景

### 场景一：排查线上问题

```bash
# 1. 查看进程是否存活
ps aux | grep node

# 2. 查看端口监听
ss -tlnp | grep 3000

# 3. 查看实时日志
tail -f /var/log/app/error.log

# 4. 搜索错误关键字
grep "Uncaught Exception" /var/log/app/error.log

# 5. 查看资源占用
top -o %MEM                     # 按内存占用排序

# 6. 查看磁盘
df -h
```

### 场景二：部署前端项目

```bash
# 1. 连接服务器
ssh ubuntu@your-server

# 2. 安装 nginx
sudo apt update
sudo apt install nginx -y

# 3. 从 GitHub 拉取代码
git clone https://github.com/user/project.git
cd project

# 4. 安装依赖并构建
npm install
npm run build

# 5. 复制构建产物到 nginx 目录
sudo cp -r dist/* /var/www/html/

# 6. 重启 nginx
sudo systemctl restart nginx
```

### 场景三：日志分析

```bash
# 统计访问量前 10 的 IP
cat access.log | awk '{print $1}' | sort | uniq -c | sort -rn | head -10

# 统计 HTTP 状态码分布
cat access.log | awk '{print $9}' | sort | uniq -c | sort -rn

# 统计 5xx 错误
cat access.log | awk '$9 ~ /^5/ {print $0}' | wc -l

# 最近 5 分钟的日志
awk -v date="$(date -d '5 minutes ago' '+%d/%b/%Y:%H:%M')" '$4 > "[" date' access.log
```

## 十二、常见问题

### Q1: command not found

```bash
# PATH 中没有该命令
which nginx                      # 查找命令位置
export PATH=$PATH:/usr/local/bin # 临时添加到 PATH

# 或安装
sudo apt install nginx
```

### Q2: Permission denied

```bash
# 没有执行权限
chmod +x script.sh

# 没有写权限
sudo chown -R $USER /path

# 使用 sudo
sudo apt install nginx
```

### Q3: 端口被占用

```bash
ss -tlnp | grep 3000             # 查看 3000 端口谁在用
kill -9 <PID>                    # 结束占用进程
```

### Q4: 磁盘空间满了

```bash
df -h                            # 查看磁盘使用
du -sh /var/log/* | sort -rh | head -5  # 找到大文件
# 清理日志、Docker 镜像、node_modules 等
sudo journalctl --vacuum-time=7d # 清理 systemd 日志
```

### Q5: 如何查看历史命令

```bash
history                          # 查看所有历史
history | grep nginx             # 搜索历史命令
!!                               # 执行上一条命令
!$                               # 上一条命令的最后一个参数
!233                             # 执行 history 中的第 233 条
```

## 十三、推荐学习路径

1. 掌握文件和目录操作（`ls`、`cd`、`cp`、`mv`、`rm`）
2. 学会查看文件（`cat`、`less`、`tail`、`grep`）
3. 理解权限和进程管理（`chmod`、`ps`、`kill`）
4. 掌握管道和重定向（`|`、`>`、`>>`）
5. 学习包管理和 Vim 基本操作
6. 综合实践：部署项目、分析日志
