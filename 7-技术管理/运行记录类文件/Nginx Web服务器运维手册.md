<center>
# Nginx Web 服务器运维手册
</center>
> **适用版本**：Nginx 1.18.0+ on Ubuntu 20.04/22.04


## 一、手册说明

### 1.1 目的

规范 Ubuntu 环境下 Nginx Web 服务器的部署、配置、日常运维和故障处理操作，保障智慧管理平台前端访问及 API 网关的稳定性、安全性和高可用性。

### 1.2 适用范围

本手册适用于本项目所使用的所有 Nginx 实例（含开发、测试、生产环境）的运维工作，覆盖 Web 静态资源服务、反向代理、负载均衡、SSL 终止等场景。

### 1.3 核心文件路径

| 文件/目录 | 路径 | 说明 |
|-----------|------|------|
| 主配置文件 | `/etc/nginx/nginx.conf` | Nginx 主配置入口 |
| 站点配置目录 | `/etc/nginx/sites-available/` | 所有站点配置文件存放 |
| 站点启用目录 | `/etc/nginx/sites-enabled/` | 已启用的站点配置（软链接） |
| 模块配置目录 | `/etc/nginx/conf.d/` | 独立模块配置（如 SSL、限流） |
| 默认站点配置 | `/etc/nginx/sites-available/default` | 默认配置模板 |
| 访问日志 | `/var/log/nginx/access.log` | 所有请求访问记录 |
| 错误日志 | `/var/log/nginx/error.log` | Nginx 运行错误记录 |
| PID 文件 | `/var/run/nginx.pid` | Nginx 进程 ID |
| 静态资源目录 | `/var/www/html/` | 默认 Web 根目录 |


## 二、安装与部署

### 2.1 系统环境准备

```bash
# 更新系统包列表
sudo apt update && sudo apt upgrade -y

# 确认系统版本
lsb_release -a

# 查看可用 Nginx 版本
apt list -a nginx
```

### 2.2 安装 Nginx

```bash
# 方案一：使用 APT 安装（推荐，稳定版）
sudo apt install nginx -y

# 方案二：如需官方最新版，添加官方源
sudo add-apt-repository ppa:nginx/stable -y
sudo apt update
sudo apt install nginx -y
```

### 2.3 验证安装

```bash
# 查看 Nginx 版本
nginx -v

# 查看编译参数（含模块列表）
nginx -V

# 检查服务状态
sudo systemctl status nginx

# 访问测试（本地 curl 或浏览器打开服务器 IP）
curl -I http://localhost
```

### 2.4 服务管理

```bash
# 启动 Nginx
sudo systemctl start nginx

# 设置开机自启
sudo systemctl enable nginx

# 停止 Nginx
sudo systemctl stop nginx

# 重启 Nginx（完全停止再启动）
sudo systemctl restart nginx

# 重新加载配置（不中断服务，平滑生效）
sudo systemctl reload nginx

# 查看服务状态
sudo systemctl status nginx
```

### 2.5 防火墙配置

```bash
# 放行 HTTP（80）和 HTTPS（443）端口
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# 查看防火墙状态
sudo ufw status
```


## 三、配置规范

### 3.1 配置文件结构说明

Nginx 配置采用层级结构，建议按以下层次组织：

```
/etc/nginx/
├── nginx.conf              # 主配置（worker 进程、全局设置）
├── sites-available/        # 所有站点配置（域名站点）
│   ├── default             # 默认站点
│   └── app-api.conf        # 项目 API 反向代理
├── sites-enabled/          # 启用的站点（软链接到 sites-available）
│   ├── default -> ../sites-available/default
│   └── app-api.conf -> ../sites-available/app-api.conf
├── conf.d/                 # 全局片段配置
│   ├── gzip.conf           # Gzip 压缩配置
│   ├── security.conf       # 安全头配置
│   ├── rate-limit.conf     # 限流配置
│   └── upstream.conf       # 上游服务器配置
└── snippets/               # 可复用的配置片段
```

### 3.2 主配置：/etc/nginx/nginx.conf（生产环境基准）

```nginx
# 运行用户
user www-data;

# Worker 进程数（建议设为 CPU 核心数）
worker_processes auto;

# PID 文件
pid /var/run/nginx.pid;

# 错误日志级别
error_log /var/log/nginx/error.log warn;

# 事件模型优化
events {
    worker_connections 4096;         # 单进程最大连接数
    use epoll;                       # Linux 高并发事件模型
    multi_accept on;                 # 一次接受所有连接
}

# HTTP 核心配置
http {
    # 基础文件类型
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    # 日志格式定义（含响应时间）
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for" '
                    'rt=$request_time uct="$upstream_connect_time" '
                    'uht="$upstream_header_time" urt="$upstream_response_time"';

    # 访问日志
    access_log /var/log/nginx/access.log main;

    # 高效文件传输（优化静态文件）
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;

    # 连接超时配置
    keepalive_timeout 65;
    client_header_timeout 30;
    client_body_timeout 30;
    send_timeout 30;

    # 请求体大小限制（根据业务调整，如文件上传）
    client_max_body_size 50M;

    # 开启 Gzip 压缩（减少传输体积）
    gzip on;
    gzip_vary on;
    gzip_comp_level 6;
    gzip_min_length 1024;
    gzip_types text/plain text/css text/xml text/javascript
               application/javascript application/json application/xml+rss;

    # 包含各站点配置
    include /etc/nginx/conf.d/*.conf;
    include /etc/nginx/sites-enabled/*;
}
```

### 3.3 站点配置模板

#### 3.3.1 静态站点配置

```nginx
# /etc/nginx/sites-available/static-site
server {
    listen 80;
    listen [::]:80;

    server_name static.yourdomain.com;
    root /var/www/html/static;

    index index.html index.htm;

    # 静态资源缓存（图片/JS/CSS 缓存 7 天）
    location ~* \.(jpg|jpeg|png|gif|ico|css|js|svg|woff|woff2)$ {
        expires 7d;
        add_header Cache-Control "public, immutable";
        try_files $uri $uri/ =404;
    }

    # HTML 文件不缓存
    location ~* \.(html|htm)$ {
        expires -1;
        add_header Cache-Control "no-store, no-cache, must-revalidate";
    }

    # 健康检查端点（负载均衡器使用）
    location /health {
        access_log off;
        return 200 "OK\n";
        add_header Content-Type text/plain;
    }
}
```

#### 3.3.2 反向代理配置（后端 API）

```nginx
# /etc/nginx/sites-available/api-gateway
upstream backend_api {
    # 负载均衡策略（默认轮询）
    # 如需保持会话：ip_hash;
    server 127.0.0.1:8080 max_fails=3 fail_timeout=30s;
    server 127.0.0.1:8081 max_fails=3 fail_timeout=30s;

    # 长连接保持
    keepalive 64;
}

server {
    listen 80;
    listen [::]:80;

    server_name api.yourdomain.com;

    # 安全头部
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;

    # 限流配置（引用全局限流）
    limit_req zone=api_limit burst=20 nodelay;

    location / {
        proxy_pass http://backend_api;
        proxy_http_version 1.1;
        proxy_set_header Connection "";

        # 透传真实客户端信息
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # 超时设置
        proxy_connect_timeout 30s;
        proxy_send_timeout 30s;
        proxy_read_timeout 30s;
        proxy_buffering off;          # 流式响应关闭缓冲
        proxy_cache_bypass $http_upgrade;
    }

    # 健康检查
    location /health {
        access_log off;
        return 200 "OK\n";
        add_header Content-Type text/plain;
    }
}
```

#### 3.3.3 HTTPS/SSL 配置（生产环境强制）

```nginx
# /etc/nginx/sites-available/app-https
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;

    server_name app.yourdomain.com;

    # SSL 证书路径（建议使用 Let's Encrypt）
    ssl_certificate /etc/letsencrypt/live/yourdomain/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/yourdomain/privkey.pem;

    # SSL 安全配置（A+ 级）
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers off;

    # HSTS（强制 HTTPS，有效期 6 个月）
    add_header Strict-Transport-Security "max-age=15768000; includeSubDomains; preload" always;

    # 其他安全头
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;

    # 代理后端（同 HTTP 配置）
    location / {
        proxy_pass http://backend_app;
        # ... 其他 proxy 配置同上
    }
}

# HTTP 自动跳转 HTTPS
server {
    listen 80;
    listen [::]:80;
    server_name app.yourdomain.com;
    return 301 https://$server_name$request_uri;
}
```

### 3.4 启用站点配置

```bash
# 创建软链接至 sites-enabled 以启用站点
sudo ln -s /etc/nginx/sites-available/app-api.conf /etc/nginx/sites-enabled/

# 测试配置文件语法
sudo nginx -t

# 重新加载使配置生效
sudo systemctl reload nginx
```


## 四、部署与管理

### 4.1 静态资源部署

```bash
# 将前端构建产物部署到 Web 根目录
sudo rsync -av --delete /local/build/ /var/www/html/static/

# 设置正确的权限（必须为 755/644）
sudo chown -R www-data:www-data /var/www/html/static
sudo find /var/www/html/static -type f -exec chmod 644 {} \;
sudo find /var/www/html/static -type d -exec chmod 755 {} \;
```

### 4.2 配置变更流程

```bash
# 1. 编辑配置文件
sudo vim /etc/nginx/sites-available/app-api.conf

# 2. 测试语法（必须执行）
sudo nginx -t

# 3. 测试通过后重新加载（平滑生效）
sudo systemctl reload nginx

# 4. 如测试失败，检查错误位置
sudo nginx -t 2>&1 | grep "failed"
```


## 五、日志管理

### 5.1 日志查看与分析

```bash
# 实时查看访问日志
sudo tail -f /var/log/nginx/access.log

# 实时查看错误日志
sudo tail -f /var/log/nginx/error.log

# 查看最近 100 行错误日志
sudo tail -n 100 /var/log/nginx/error.log

# 统计今日独立 IP 访问量
cat /var/log/nginx/access.log | \
    grep "$(date +%d/%b/%Y)" | \
    awk '{print $1}' | sort -u | wc -l

# 查看 TOP 10 请求 URI
cat /var/log/nginx/access.log | \
    awk '{print $7}' | sort | uniq -c | sort -nr | head -10

# 查看 TOP 10 慢请求（响应时间 > 1s）
cat /var/log/nginx/access.log | \
    awk '{split($NF, rt, "="); if(rt[2] > 1.0) print $0}' | head -20
```

### 5.2 日志轮转（logrotate）

Nginx 日志默认由 `logrotate` 管理，配置位于 `/etc/logrotate.d/nginx`：

```
/var/log/nginx/*.log {
    daily
    missingok
    rotate 30
    compress
    delaycompress
    notifempty
    create 644 www-data www-data
    postrotate
        if [ -f /var/run/nginx.pid ]; then
            kill -USR1 `cat /var/run/nginx.pid`
        fi
    endscript
}
```

**手动触发轮转：**
```bash
sudo logrotate -f /etc/logrotate.d/nginx
```


## 六、监控与巡检

### 6.1 每日巡检清单

| 检查项 | 命令 | 正常标准 |
|--------|------|----------|
| 服务状态 | `sudo systemctl status nginx` | active (running) |
| 配置语法 | `sudo nginx -t` | syntax is ok |
| 进程检查 | `ps aux | grep nginx` | master + worker 进程存在 |
| 端口监听 | `sudo netstat -tulnp | grep nginx` | 80/443 正常监听 |
| 错误日志 | `sudo tail -n 50 /var/log/nginx/error.log` | 无 ERROR/EMERG |
| 访问响应 | `curl -I http://localhost -w "%{http_code}" -o /dev/null -s` | 返回 200 |
| 磁盘空间 | `df -h /var` | 剩余 > 20% |
| **巡检人** | | **巡检日期** | |

### 6.2 性能监控指标

```bash
# 查看当前连接状态
curl -s http://localhost/nginx_status | head -20

# 使用 ngxtop 实时监控（需安装：pip install ngxtop）
ngxtop

# 统计当前 Worker 进程数
ps aux | grep "nginx: worker" | grep -v grep | wc -l
```

### 6.3 第三方监控集成

| 监控方案 | 采集内容 | 部署方式 |
|----------|---------|----------|
| **Prometheus + nginx-prometheus-exporter** | 连接数/请求数/状态码分布 | 独立 exporter |
| **Zabbix** | 服务存活/端口/日志关键字 | 主动/被动 agent |
| **nginx-plus (商业版)** | 内置扩展指标 + Dashboard | 内置 |


## 七、性能优化

### 7.1 Worker 进程调优

```nginx
# /etc/nginx/nginx.conf

# Worker 数建议等于 CPU 核心数
worker_processes auto;

# CPU 亲和性绑定（auto 自动优化）
worker_cpu_affinity auto;

# 单个 Worker 最大连接数（建议 4096-8192）
events {
    worker_connections 4096;
    use epoll;
    multi_accept on;
}
```

### 7.2 静态资源缓存优化

```nginx
# 开启 sendfile（零拷贝，提升大文件传输效率）
sendfile on;
tcp_nopush on;

# 代理缓存（减少后端请求）
proxy_cache_path /var/cache/nginx levels=1:2 keys_zone=cache_zone:100m inactive=30m;
proxy_cache cache_zone;
proxy_cache_valid 200 304 30m;
proxy_cache_valid 404 1m;
```

### 7.3 连接数估算公式

```
最大连接数 ≈ worker_processes × worker_connections
```

- 例：`worker_processes=4`，`worker_connections=4096`
- 最大并发 ≈ 4 × 4096 = **16,384**（已包含浏览器连接 + 后端连接）

### 7.4 调优检查工具

```bash
# 压力测试（生产前建议执行）
ab -n 10000 -c 100 http://yourdomain.com/

# 查看 Nginx 当前连接状态
curl -s http://localhost/nginx_status
```


## 八、安全加固

### 8.1 隐藏版本号

```nginx
# /etc/nginx/nginx.conf
server_tokens off;   # 隐藏响应头中的版本号
```

### 8.2 限制请求方法

```nginx
location / {
    if ($request_method !~ ^(GET|HEAD|POST)$ ) {
        return 405;
    }
}
```

### 8.3 限制 IP 访问（白名单）

```nginx
location /admin/ {
    allow 192.168.1.0/24;
    allow 10.0.0.0/8;
    deny all;
}
```

### 8.4 防止 SQL 注入/XSS（基础过滤）

```nginx
location ~* \.(php|jsp|cgi|asp|aspx)$ {
    return 403;
}
```

### 8.5 SSL 证书自动更新（Let's Encrypt）

```bash
# 安装 Certbot
sudo apt install certbot python3-certbot-nginx -y

# 首次申请证书（自动配置 Nginx）
sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com

# 测试自动续期
sudo certbot renew --dry-run

# 证书自动续期（cron 每日检查）
# 系统已自动添加：/etc/cron.d/certbot
```


## 九、常见故障排查

### 9.1 服务无法启动

```bash
# 1. 查看服务状态
sudo systemctl status nginx

# 2. 查看错误日志
sudo tail -n 50 /var/log/nginx/error.log

# 3. 测试配置文件
sudo nginx -t

# 4. 常见原因：
# - 配置文件语法错误 → 修正后重试
# - 端口冲突（80/443 被占用）→ sudo netstat -tulnp | grep -E '80|443'
# - PID 文件残留 → sudo rm -f /var/run/nginx.pid
```

### 9.2 502 Bad Gateway（代理后端不通）

```bash
# 1. 检查后端服务是否运行
sudo systemctl status backend-service

# 2. 检查后端服务端口是否监听
sudo netstat -tulnp | grep 8080

# 3. 检查防火墙
sudo ufw status

# 4. 检查 Nginx 错误日志
sudo tail -n 50 /var/log/nginx/error.log | grep upstream
```

### 9.3 504 Gateway Timeout（后端响应超时）

```nginx
# 调整 proxy_read_timeout 参数（增加超时时间）
proxy_read_timeout 60s;
proxy_connect_timeout 60s;
proxy_send_timeout 60s;
```

### 9.4 413 Request Entity Too Large

```nginx
# 调整 client_max_body_size 参数（文件上传限制）
client_max_body_size 100M;   # 在 http/server/location 块中设置
```

### 9.5 磁盘空间不足

```bash
# 1. 查看磁盘使用
df -h

# 2. 查看日志文件大小
du -sh /var/log/nginx/*

# 3. 清理压缩旧日志
sudo logrotate -f /etc/logrotate.d/nginx

# 4. 清理缓存目录
sudo rm -rf /var/cache/nginx/*
```


## 十、附录

### 10.1 常用命令速查

| 用途 | 命令 |
|------|------|
| 安装 Nginx | `sudo apt install nginx -y` |
| 启动服务 | `sudo systemctl start nginx` |
| 停止服务 | `sudo systemctl stop nginx` |
| 重启服务 | `sudo systemctl restart nginx` |
| 重新加载配置 | `sudo systemctl reload nginx` |
| 查看状态 | `sudo systemctl status nginx` |
| 测试配置文件 | `sudo nginx -t` |
| 查看版本 + 编译参数 | `nginx -V` |
| 查看访问日志 | `sudo tail -f /var/log/nginx/access.log` |
| 查看错误日志 | `sudo tail -f /var/log/nginx/error.log` |
| 检查端口监听 | `sudo netstat -tulnp | grep nginx` |
| 启用站点 | `sudo ln -s /etc/nginx/sites-available/xxx /etc/nginx/sites-enabled/` |
| 禁用站点 | `sudo rm /etc/nginx/sites-enabled/xxx` |
| 证书自动续期 | `sudo certbot renew` |

### 10.2 配置文件检查清单（上线前必检）

- [ ] `nginx -t` 语法测试通过
- [ ] 站点配置文件已正确创建软链接到 `sites-enabled`
- [ ] server_name 域名正确
- [ ] 静态文件目录权限为 755/644
- [ ] 代理后端 upstream 地址正确且服务正常
- [ ] SSL 证书有效期 > 30 天
- [ ] HTTP 已配置跳转 HTTPS
- [ ] 安全头部（HSTS/X-Frame-Options）已添加
- [ ] 日志路径正确且有写入权限
- [ ] 防火墙 80/443 端口已放行
