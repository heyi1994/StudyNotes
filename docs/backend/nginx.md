# Nginx

## 1. Nginx 简介

Nginx（读作 "engine-x"）是一款高性能的 HTTP 服务器、反向代理服务器及通用 TCP/UDP 代理，由俄罗斯工程师 Igor Sysoev 于 2004 年发布。

### 架构原理

```
传统服务器（Apache 等）：
  每个请求 → 创建/分配一个线程/进程处理
  高并发时：大量线程切换 → 内存和 CPU 开销巨大

Nginx 架构：
  Master 进程（1个）：读取配置、管理 Worker
  Worker 进程（N个，通常等于 CPU 核心数）：
    每个 Worker 用 epoll/kqueue 事件驱动
    单线程处理数千个并发连接（无线程切换开销）

  Master
    ├── Worker 1  (epoll: conn1, conn2, conn3...)
    ├── Worker 2  (epoll: conn4, conn5, conn6...)
    └── Worker N  (epoll: ...)
```

### Nginx vs Apache

| 维度       | Nginx                      | Apache                      |
| ---------- | -------------------------- | --------------------------- |
| 架构       | 事件驱动，异步非阻塞       | 进程/线程模型               |
| 高并发性能 | 极强（万级并发轻松应对）   | 较弱（线程开销大）          |
| 内存占用   | 极低                       | 较高                        |
| 动态内容   | 需配合 FastCGI/uWSGI       | 内置模块直接处理（mod_php） |
| 配置灵活性 | 较强（Location 精细控制）  | 支持 .htaccess 分目录配置   |
| 模块热加载 | 不支持（需重编译）         | 支持动态加载模块            |
| 适用场景   | 高并发、反向代理、静态资源 | 传统 Web 应用、共享主机     |

---

## 2. 安装与目录结构

### 安装

```bash
# Ubuntu / Debian
sudo apt update && sudo apt install nginx

# CentOS / RHEL
sudo yum install epel-release && sudo yum install nginx

# macOS（Homebrew）
brew install nginx

# 编译安装（自定义模块）
./configure --prefix=/etc/nginx \
            --with-http_ssl_module \
            --with-http_v2_module \
            --with-http_gzip_static_module \
            --with-stream \
            --with-stream_ssl_module
make && make install

# 验证安装
nginx -v          # 查看版本
nginx -V          # 查看版本及编译参数
```

### 目录结构

```
/etc/nginx/
├── nginx.conf              # 主配置文件
├── conf.d/                 # 子配置目录（include 进主配置）
│   ├── default.conf
│   └── myapp.conf
├── sites-available/        # Debian/Ubuntu 风格：可用站点
│   └── mysite
├── sites-enabled/          # 软链接到 sites-available 中已启用的站点
│   └── mysite -> ../sites-available/mysite
├── snippets/               # 可复用的配置片段
│   └── ssl-params.conf
├── mime.types              # 文件扩展名 → Content-Type 映射
├── fastcgi_params          # FastCGI 参数
├── uwsgi_params            # uWSGI 参数
└── modules-enabled/        # 动态模块

/var/log/nginx/
├── access.log              # 访问日志
└── error.log               # 错误日志

/var/www/html/              # 默认网站根目录（Ubuntu）
/usr/share/nginx/html/      # 默认网站根目录（CentOS）

/usr/lib/nginx/modules/     # 动态模块（.so 文件）
```

---

## 3. 核心配置语法

### 配置文件结构

```nginx
# 全局块（影响整个 Nginx 进程）
user              nginx;
worker_processes  auto;
error_log         /var/log/nginx/error.log warn;
pid               /var/run/nginx.pid;

# events 块（网络连接配置）
events {
    worker_connections 1024;   # 每个 Worker 最大并发连接数
    use epoll;                 # 事件模型（Linux 推荐 epoll）
    multi_accept on;           # 一次尽可能多地接受连接
}

# http 块（HTTP 服务配置）
http {
    include      mime.types;
    default_type application/octet-stream;

    # server 块（虚拟主机）
    server {
        listen      80;
        server_name example.com;

        # location 块（URL 匹配规则）
        location / {
            root  /var/www/html;
            index index.html;
        }

        location /api {
            proxy_pass http://backend:8080;
        }
    }
}

# stream 块（TCP/UDP 代理，需要 --with-stream 模块）
stream {
    server {
        listen     3306;
        proxy_pass mysql_backend;
    }
}
```

### 指令类型

```nginx
# 简单指令：名称 值;
worker_processes 4;
error_log /var/log/nginx/error.log;

# 块指令：名称 { ... }
events { }
http { }
server { }
location / { }

# include：引入其他配置文件
include /etc/nginx/conf.d/*.conf;
include /etc/nginx/sites-enabled/*;
```

### 变量

```nginx
# Nginx 内置变量（常用）
$host           # 请求的 Host 头（域名）
$uri            # 当前请求的 URI（不含查询串）
$args           # 查询字符串（? 后面的部分）
$request_uri    # 完整 URI（含查询串）
$request_method # 请求方法（GET/POST/...）
$remote_addr    # 客户端 IP
$server_port    # 服务器端口
$server_protocol# 协议版本（HTTP/1.1）
$scheme         # 协议（http/https）
$status         # 响应状态码
$body_bytes_sent# 响应体字节数
$http_referer   # Referer 头
$http_user_agent# User-Agent 头
$http_x_forwarded_for  # X-Forwarded-For 头
$time_local     # 本地时间
$request_time   # 请求处理耗时（秒）
$upstream_addr  # 上游服务器地址
$upstream_response_time # 上游响应时间
$ssl_protocol   # SSL 协议版本
$ssl_cipher     # SSL 加密算法

# 自定义变量
set $my_var "hello";
set $is_mobile 0;
```

### if 条件判断

```nginx
# 注意：Nginx 的 if 有很多"坑"，尽量用 map 或 try_files 替代
location / {
    # 正则匹配（~区分大小写，~*不区分）
    if ($http_user_agent ~* "Mobile") {
        set $is_mobile 1;
    }

    # 文件不存在
    if (!-f $request_filename) {
        return 404;
    }

    # 等值判断
    if ($request_method = POST) {
        return 405;
    }
}
```

### map 模块（if 的更好替代）

```nginx
http {
    # 根据 User-Agent 设置变量
    map $http_user_agent $device_type {
        default          "desktop";
        "~*Mobile"       "mobile";
        "~*iPad|Tablet"  "tablet";
    }

    # 根据 IP 段设置变量
    map $remote_addr $is_internal {
        default         0;
        "~^10\."        1;
        "~^192\.168\."  1;
        "~^172\.(1[6-9]|2[0-9]|3[01])\." 1;
    }

    server {
        location / {
            add_header X-Device $device_type;
            if ($is_internal = 0) {
                return 403;
            }
        }
    }
}
```

---

## 4. 静态文件服务

```nginx
server {
    listen      80;
    server_name static.example.com;

    root /var/www/static;   # 网站根目录

    # 首页文件（按顺序查找）
    index index.html index.htm;

    location / {
        # try_files：按顺序尝试，都不存在则返回 404
        try_files $uri $uri/ /index.html;
        # 含义：先找文件，再找目录（目录下的 index），再回退到 /index.html（SPA 路由）
    }

    # 图片、JS、CSS 等静态资源缓存
    location ~* \.(jpg|jpeg|png|gif|ico|svg|webp)$ {
        expires    30d;                        # 浏览器缓存 30 天
        add_header Cache-Control "public, immutable";
        add_header Vary Accept-Encoding;
        access_log off;                        # 静态资源不记录访问日志（减少 I/O）
    }

    location ~* \.(css|js)$ {
        expires    7d;
        add_header Cache-Control "public";
    }

    location ~* \.(woff|woff2|ttf|eot)$ {
        expires    365d;
        add_header Access-Control-Allow-Origin "*";  # 字体跨域
    }

    # 禁止访问隐藏文件（.git、.env 等）
    location ~ /\. {
        deny all;
        return 404;
    }

    # 目录浏览（谨慎开启）
    location /files/ {
        autoindex on;
        autoindex_exact_size off;   # 显示人类可读的文件大小
        autoindex_localtime  on;    # 显示本地时区时间
    }
}
```

### 别名（alias）

```nginx
# root：将 location 路径附加到 root 后
# alias：将 location 路径替换为 alias 路径

# root 示例
location /images/ {
    root /var/www;
    # 实际路径：/var/www/images/foo.jpg
}

# alias 示例（注意末尾斜杠）
location /images/ {
    alias /data/pics/;
    # 实际路径：/data/pics/foo.jpg（替换了 /images/）
}
```

---

## 5. 反向代理

### 基础反向代理

```nginx
server {
    listen 80;
    server_name api.example.com;

    location / {
        proxy_pass http://127.0.0.1:8080;  # 转发到后端

        # 必加的 Header，让后端知道真实客户端信息
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # 超时配置
        proxy_connect_timeout  10s;   # 连接上游超时
        proxy_send_timeout     60s;   # 发送请求超时
        proxy_read_timeout     60s;   # 读取响应超时

        # HTTP 版本（默认 1.0，建议改 1.1 以支持长连接）
        proxy_http_version 1.1;
        proxy_set_header   Connection "";

        # 缓冲区（避免后端响应慢时占用连接）
        proxy_buffering         on;
        proxy_buffer_size       4k;
        proxy_buffers           8 16k;
        proxy_busy_buffers_size 32k;
    }
}
```

### proxy_pass 路径拼接规则

```nginx
# 规则：proxy_pass 末尾有无 / 决定是否保留 location 前缀

# ❶ proxy_pass 不带路径（末尾无/）：完整转发 URI
location /api/ {
    proxy_pass http://backend;
    # 请求 /api/users → 转发 /api/users
}

# ❷ proxy_pass 带路径（末尾有/）：替换 location 前缀
location /api/ {
    proxy_pass http://backend/;
    # 请求 /api/users → 转发 /users（去掉了 /api 前缀）
}

# ❸ proxy_pass 带具体路径：替换 location 前缀
location /api/ {
    proxy_pass http://backend/v2/;
    # 请求 /api/users → 转发 /v2/users
}
```

### 转发到 Unix Socket

```nginx
# 比 TCP 更快（无网络栈开销），用于本机进程间通信
location / {
    proxy_pass http://unix:/var/run/app.sock;
}
```

---

## 6. 负载均衡

### upstream 块

```nginx
http {
    # 定义上游服务器组
    upstream backend {
        # 负载均衡算法（默认：轮询）
        server 10.0.0.1:8080;
        server 10.0.0.2:8080;
        server 10.0.0.3:8080;
    }

    server {
        location / {
            proxy_pass http://backend;
        }
    }
}
```

### 负载均衡算法

```nginx
# ---- 1. 轮询（Round Robin，默认）----
upstream backend_rr {
    server 10.0.0.1:8080;
    server 10.0.0.2:8080;
    server 10.0.0.3:8080;
}

# ---- 2. 加权轮询（按 weight 分配流量）----
upstream backend_weight {
    server 10.0.0.1:8080 weight=5;   # 分配 50% 流量（高配机器）
    server 10.0.0.2:8080 weight=3;   # 分配 30%
    server 10.0.0.3:8080 weight=2;   # 分配 20%
}

# ---- 3. IP Hash（同一 IP 固定到同一服务器，解决 Session 问题）----
upstream backend_iphash {
    ip_hash;
    server 10.0.0.1:8080;
    server 10.0.0.2:8080;
}

# ---- 4. least_conn（最少连接数，适合处理时长差异大的场景）----
upstream backend_lc {
    least_conn;
    server 10.0.0.1:8080;
    server 10.0.0.2:8080;
}

# ---- 5. 一致性哈希（按 URI 哈希，相同 URI 固定到同一服务器，利于缓存）----
upstream backend_hash {
    hash $request_uri consistent;
    server 10.0.0.1:8080;
    server 10.0.0.2:8080;
}
```

### 服务器参数详解

```nginx
upstream backend {
    server 10.0.0.1:8080 weight=3 max_fails=3 fail_timeout=30s;
    #                     权重=3  最大失败次数  失败后暂停时间

    server 10.0.0.2:8080;

    # 备用服务器（主服务器全挂才启用）
    server 10.0.0.3:8080 backup;

    # 永久下线（保留配置但不使用）
    server 10.0.0.4:8080 down;

    # 长连接池（避免每次请求都建立 TCP 连接）
    keepalive 32;              # 保持 32 个空闲长连接
    keepalive_requests 1000;   # 单个长连接最多复用 1000 次请求
    keepalive_timeout  60s;    # 长连接空闲超时
}
```

### 健康检查（商业版 nginx_upstream_check_module / 开源替代）

```nginx
# 开源版：被动健康检查（通过 max_fails / fail_timeout）
upstream backend {
    server 10.0.0.1:8080 max_fails=3 fail_timeout=30s;
    # 30s 内失败 3 次 → 标记为不可用，30s 后重试
}

# nginx-upstream-check-module（第三方模块，需编译）
upstream backend {
    server 10.0.0.1:8080;
    server 10.0.0.2:8080;

    check interval=3000 rise=2 fall=3 timeout=1000 type=http;
    check_http_send "HEAD /health HTTP/1.0\r\n\r\n";
    check_http_expect_alive http_2xx;
}
```

---

## 7. HTTPS 与 SSL

### 基础 HTTPS 配置

```nginx
server {
    listen 443 ssl http2;         # 开启 SSL 和 HTTP/2
    server_name example.com;

    # 证书文件（Let's Encrypt 免费证书）
    ssl_certificate     /etc/nginx/ssl/example.com.crt;
    ssl_certificate_key /etc/nginx/ssl/example.com.key;

    # 只允许安全协议版本
    ssl_protocols TLSv1.2 TLSv1.3;

    # 加密套件（Mozilla 推荐的中级配置）
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers off;   # TLSv1.3 不需要服务端优先

    # SSL Session 缓存（减少握手次数）
    ssl_session_cache   shared:SSL:10m;
    ssl_session_timeout 1d;
    ssl_session_tickets off;    # 禁用 Session Ticket（前向安全）

    # OCSP Stapling（加速证书验证）
    ssl_stapling        on;
    ssl_stapling_verify on;
    resolver 8.8.8.8 8.8.4.4 valid=300s;

    # HSTS（强制浏览器使用 HTTPS，一年）
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;

    location / {
        root /var/www/html;
        index index.html;
    }
}

# HTTP → HTTPS 永久重定向
server {
    listen 80;
    server_name example.com www.example.com;
    return 301 https://example.com$request_uri;
}
```

### Let's Encrypt 自动证书（Certbot）

```bash
# 安装 Certbot
sudo apt install certbot python3-certbot-nginx

# 自动申请证书并配置 Nginx
sudo certbot --nginx -d example.com -d www.example.com

# 测试自动续期
sudo certbot renew --dry-run

# 手动续期
sudo certbot renew

# 查看证书有效期
sudo certbot certificates
```

### DH 参数（增强安全性）

```bash
# 生成 DH 参数（一次性，较慢）
sudo openssl dhparam -out /etc/nginx/ssl/dhparam.pem 2048
```

```nginx
ssl_dhparam /etc/nginx/ssl/dhparam.pem;
```

---

## 8. 虚拟主机

### 基于域名的虚拟主机

```nginx
# 站点 1
server {
    listen 80;
    server_name example.com www.example.com;
    root /var/www/example;
}

# 站点 2
server {
    listen 80;
    server_name blog.example.com;
    root /var/www/blog;
}

# 站点 3
server {
    listen 80;
    server_name shop.example.com;
    root /var/www/shop;
}

# 默认服务器（无匹配域名时使用）
server {
    listen 80 default_server;
    server_name _;        # _ 是无效域名，用来表示"兜底"
    return 444;           # 444：Nginx 特有，直接关闭连接（无响应）
}
```

### 基于端口的虚拟主机

```nginx
server {
    listen 8080;
    server_name localhost;
    root /var/www/app1;
}

server {
    listen 9090;
    server_name localhost;
    root /var/www/app2;
}
```

### server_name 匹配规则

```nginx
# 精确匹配（优先级最高）
server_name example.com;

# 通配符前缀
server_name *.example.com;

# 通配符后缀
server_name example.*;

# 正则（最低优先级，以 ~ 开头）
server_name ~^(www|api|static)\.example\.com$;

# 优先级：精确 > 前缀通配 > 后缀通配 > 正则 > default_server
```

---

## 9. Location 匹配规则

### 匹配语法与优先级

```nginx
server {
    # ❶ = 精确匹配（最高优先级）
    location = / {
        # 只匹配根路径 "/"，精确到字符
        return 200 "exact match";
    }

    # ❷ ^~ 前缀匹配（比正则优先，找到后不再尝试正则）
    location ^~ /static/ {
        root /var/www;
        # 请求 /static/a.js → 匹配此处，不再检查正则
    }

    # ❸ ~ 正则匹配（区分大小写）
    location ~ \.php$ {
        fastcgi_pass unix:/var/run/php-fpm.sock;
    }

    # ❹ ~* 正则匹配（不区分大小写）
    location ~* \.(jpg|jpeg|png|gif)$ {
        expires 30d;
    }

    # ❺ 普通前缀匹配（最低优先级，作为兜底）
    location / {
        try_files $uri $uri/ =404;
    }
}
```

### 优先级总结

```
1. =    精确匹配          ← 最高
2. ^~   前缀匹配（阻止正则）
3. ~    正则（区分大小写） ← 按配置顺序，先匹配先用
   ~*   正则（不区分大小写）
4. /xxx 普通前缀匹配      ← 最低（取最长匹配）
```

### 实际场景示例

```nginx
server {
    listen 80;
    server_name example.com;
    root /var/www/html;

    # 根路径精确匹配，快速响应
    location = / {
        try_files /index.html =404;
    }

    # API 反向代理
    location /api/ {
        proxy_pass http://backend:8080/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    # 静态资源（阻止正则，直接用文件系统）
    location ^~ /assets/ {
        expires 365d;
        add_header Cache-Control "public, immutable";
    }

    # PHP 处理
    location ~ \.php$ {
        include        fastcgi_params;
        fastcgi_pass   unix:/var/run/php/php8.2-fpm.sock;
        fastcgi_index  index.php;
        fastcgi_param  SCRIPT_FILENAME $document_root$fastcgi_script_name;
    }

    # 禁止访问敏感文件
    location ~ /\.(git|env|htaccess|DS_Store) {
        deny all;
        return 404;
    }

    # SPA 前端（兜底）
    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

---

## 10. Rewrite 与重定向

### return 指令

```nginx
# 重定向
return 301 https://example.com$request_uri;   # 永久重定向
return 302 https://example.com$request_uri;   # 临时重定向

# 直接返回状态码和内容
return 200 "OK";
return 404 "Not Found";
return 403;

# Nginx 特有：关闭连接（不返回任何响应，用于拒绝恶意请求）
return 444;
```

### rewrite 指令

```nginx
# 语法：rewrite 正则 替换 [flag]
# flag:
#   last      ：重写后重新搜索 location（类似内部跳转）
#   break     ：重写后停止，在当前 location 继续处理
#   redirect  ：302 临时重定向
#   permanent ：301 永久重定向

# 去掉 URL 中的 index.php
rewrite ^/index\.php/(.*)$ /$1 permanent;

# 旧路径迁移到新路径
rewrite ^/old-blog/(.*)$ /blog/$1 permanent;

# 强制带 www
server {
    server_name example.com;
    return 301 $scheme://www.example.com$request_uri;
}

# 强制去掉 www
server {
    server_name www.example.com;
    return 301 $scheme://example.com$request_uri;
}

# 删除尾部斜杠（/about/ → /about）
rewrite ^/(.*)/$ /$1 permanent;
```

### 实际场景

```nginx
server {
    listen 80;
    server_name example.com;

    # 维护模式（屏蔽除特定 IP 外的所有请求）
    set $maintenance 0;
    if ($remote_addr != "1.2.3.4") {
        set $maintenance 1;
    }
    if ($maintenance = 1) {
        return 503;
    }

    # 移动端重定向
    location / {
        if ($http_user_agent ~* "Mobile|Android|iPhone") {
            return 302 https://m.example.com$request_uri;
        }
        root /var/www/html;
    }

    # 防盗链（Referer 校验）
    location ~* \.(jpg|png|gif|mp4)$ {
        valid_referers none blocked example.com *.example.com;
        if ($invalid_referer) {
            return 403;
        }
        root /var/www/media;
    }
}
```

---

## 11. 缓存

### 代理缓存（proxy_cache）

```nginx
http {
    # 定义缓存区域
    proxy_cache_path /var/cache/nginx
        levels=1:2              # 目录层级（减少单目录文件数）
        keys_zone=my_cache:10m  # 共享内存区域名称和大小（存储缓存键）
        max_size=10g            # 磁盘最大占用
        inactive=60m            # 60分钟无访问则删除
        use_temp_path=off;      # 直接写入缓存目录（避免额外拷贝）

    server {
        location / {
            proxy_pass http://backend;

            proxy_cache          my_cache;           # 使用哪个缓存区
            proxy_cache_key      "$scheme$host$uri$args"; # 缓存键
            proxy_cache_valid    200 302 10m;        # 200/302 响应缓存 10 分钟
            proxy_cache_valid    404      1m;        # 404 缓存 1 分钟
            proxy_cache_valid    any      5m;        # 其他状态缓存 5 分钟
            proxy_cache_use_stale error timeout updating; # 后端故障时返回旧缓存
            proxy_cache_lock     on;                 # 防缓存击穿（同一 key 只有一个请求穿透到后端）

            # 在响应头中显示缓存状态（调试用）
            add_header X-Cache-Status $upstream_cache_status;
            # $upstream_cache_status 可能的值：
            # HIT（命中）, MISS（未命中）, EXPIRED（过期）,
            # BYPASS（绕过）, STALE（陈旧）, REVALIDATED（重验证）
        }

        # 强制跳过缓存（管理后台、带 Cookie 的请求等）
        location /admin {
            proxy_pass  http://backend;
            proxy_cache_bypass $http_cookie;    # 有 Cookie 时不用缓存
            proxy_no_cache     $http_cookie;    # 有 Cookie 时不缓存
        }
    }
}
```

### 浏览器缓存控制

```nginx
# 不缓存 HTML（确保用户获取最新页面）
location ~* \.html$ {
    add_header Cache-Control "no-cache, no-store, must-revalidate";
    add_header Pragma        "no-cache";
    expires 0;
}

# 强缓存静态资源（文件名带 hash，可永久缓存）
location ~* \.(js|css)$ {
    expires 1y;
    add_header Cache-Control "public, immutable";
}

# 图片缓存 30 天
location ~* \.(jpg|jpeg|png|gif|webp|svg|ico)$ {
    expires 30d;
    add_header Cache-Control "public";
}
```

---

## 12. 限流与限速

### 请求频率限制（limit_req）

```nginx
http {
    # 定义限流区域
    # $binary_remote_addr：以客户端 IP 作为限流 key（binary 格式节省内存）
    # zone=req_limit:10m：共享内存区名称和大小（10MB 约存 16万个 IP 状态）
    # rate=10r/s：每秒最多 10 个请求
    limit_req_zone $binary_remote_addr zone=req_limit:10m rate=10r/s;

    # 也可以按接口限流
    limit_req_zone $binary_remote_addr zone=login_limit:10m rate=1r/s;

    # 全局限流（按 IP）
    limit_req_zone $binary_remote_addr zone=global:10m rate=100r/s;

    server {
        # 应用限流
        location /api/ {
            limit_req zone=req_limit burst=20 nodelay;
            # burst=20：允许突发 20 个请求（存入桶中排队）
            # nodelay：突发请求立即处理，不排队等待（超出 burst 才返回 503）
            proxy_pass http://backend;
        }

        # 登录接口严格限流
        location /api/login {
            limit_req zone=login_limit burst=5;
            # 无 nodelay：超过 rate 的请求进队列等待
            proxy_pass http://backend;
        }

        # 自定义限流错误页面
        limit_req_status 429;
        error_page 429 /rate_limit.html;
    }
}
```

### 并发连接数限制（limit_conn）

```nginx
http {
    # 限制同一 IP 的并发连接数
    limit_conn_zone $binary_remote_addr zone=conn_limit:10m;

    server {
        # 同一 IP 最多 10 个并发连接
        limit_conn conn_limit 10;

        # 下载接口限连
        location /download/ {
            limit_conn conn_limit 2;   # 每个 IP 最多 2 个并发下载
            limit_rate 500k;           # 每个连接限速 500KB/s
            limit_rate_after 1m;       # 前 1MB 不限速，之后开始限速
            root /var/www/files;
        }
    }
}
```

### 带宽限速（limit_rate）

```nginx
location /download/ {
    limit_rate_after 10m;    # 前 10MB 全速
    limit_rate       1m;     # 之后限速 1MB/s
}

# 根据用户等级动态限速
map $cookie_plan $rate_limit {
    default  "500k";
    "vip"    "0";      # 0 表示不限速
    "pro"    "2m";
}

location /media/ {
    limit_rate $rate_limit;
    proxy_pass http://media_backend;
}
```

---

## 13. 访问控制与安全

### IP 访问控制

```nginx
location /admin/ {
    # 白名单（只允许特定 IP）
    allow 192.168.1.0/24;
    allow 10.0.0.0/8;
    allow 1.2.3.4;
    deny  all;          # 其他全部拒绝

    proxy_pass http://admin_backend;
}

# 黑名单（屏蔽特定 IP）
location / {
    deny  1.2.3.4;
    deny  5.6.7.0/24;
    allow all;
}
```

### HTTP Basic 认证

```bash
# 生成密码文件
sudo htpasswd -c /etc/nginx/.htpasswd admin
sudo htpasswd    /etc/nginx/.htpasswd user1   # 追加用户（不加 -c）
```

```nginx
location /private/ {
    auth_basic           "请输入凭据";
    auth_basic_user_file /etc/nginx/.htpasswd;
}
```

### 安全响应头

```nginx
server {
    # 防点击劫持（禁止在 iframe 中加载）
    add_header X-Frame-Options "SAMEORIGIN" always;

    # 禁止 MIME 类型嗅探
    add_header X-Content-Type-Options "nosniff" always;

    # XSS 保护（旧浏览器）
    add_header X-XSS-Protection "1; mode=block" always;

    # 内容安全策略（CSP）
    add_header Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline';" always;

    # 引用策略
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;

    # 权限策略（禁用不必要的浏览器功能）
    add_header Permissions-Policy "geolocation=(), microphone=(), camera=()" always;

    # HSTS（仅 HTTPS）
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

    # 隐藏 Nginx 版本号
    server_tokens off;
}
```

### 防 DDoS 基础配置

```nginx
http {
    # 限制请求体大小（防上传攻击）
    client_max_body_size    10m;
    client_body_timeout     12s;
    client_header_timeout   12s;
    send_timeout            10s;

    # 限制连接数和请求频率
    limit_conn_zone $binary_remote_addr zone=ddos_conn:10m;
    limit_req_zone  $binary_remote_addr zone=ddos_req:10m rate=20r/s;

    server {
        limit_conn ddos_conn 20;
        limit_req  zone=ddos_req burst=50 nodelay;

        # 屏蔽空 User-Agent（爬虫、扫描器）
        if ($http_user_agent = "") {
            return 444;
        }

        # 屏蔽常见扫描器 UA
        if ($http_user_agent ~* "sqlmap|nikto|nmap|masscan|zgrab") {
            return 444;
        }
    }
}
```

---

## 14. 日志

### 日志格式

```nginx
http {
    # 定义日志格式
    log_format main '$remote_addr - $remote_user [$time_local] '
                    '"$request" $status $body_bytes_sent '
                    '"$http_referer" "$http_user_agent"';

    # JSON 格式（便于 ELK/Loki 收集）
    log_format json_log escape=json
        '{'
            '"timestamp":"$time_iso8601",'
            '"remote_addr":"$remote_addr",'
            '"method":"$request_method",'
            '"uri":"$uri",'
            '"args":"$args",'
            '"status":$status,'
            '"bytes_sent":$body_bytes_sent,'
            '"request_time":$request_time,'
            '"upstream_addr":"$upstream_addr",'
            '"upstream_time":"$upstream_response_time",'
            '"http_referer":"$http_referer",'
            '"http_user_agent":"$http_user_agent",'
            '"request_id":"$request_id"'
        '}';

    # 全局访问日志
    access_log /var/log/nginx/access.log json_log buffer=32k flush=5s;
    # buffer：缓冲写入（减少磁盘 I/O）
    # flush：最迟 5 秒刷盘

    # 错误日志级别：debug | info | notice | warn | error | crit | alert | emerg
    error_log /var/log/nginx/error.log warn;

    server {
        # 单个站点日志
        access_log /var/log/nginx/example.access.log json_log;
        error_log  /var/log/nginx/example.error.log  warn;

        # 关闭特定路径的日志（减少噪音）
        location /health {
            access_log off;
        }

        # 只记录错误
        location /static/ {
            access_log /var/log/nginx/static.log combined if=$upstream_cache_status;
        }
    }
}
```

### 日志轮转（logrotate）

```bash
# /etc/logrotate.d/nginx
/var/log/nginx/*.log {
    daily                   # 每天轮转
    missingok               # 文件不存在不报错
    rotate 14               # 保留 14 天
    compress                # gzip 压缩
    delaycompress           # 延迟压缩（先保留昨天的未压缩，明天再压）
    notifempty              # 空文件不轮转
    create 0640 nginx adm   # 新日志文件权限
    sharedscripts
    postrotate
        # 轮转后发送 USR1 信号，Nginx 重新打开日志文件
        if [ -f /var/run/nginx.pid ]; then
            kill -USR1 `cat /var/run/nginx.pid`
        fi
    endscript
}
```

---

## 15. Gzip 压缩

```nginx
http {
    gzip              on;
    gzip_vary         on;           # 添加 Vary: Accept-Encoding 响应头
    gzip_proxied      any;          # 代理请求也压缩
    gzip_comp_level   6;            # 压缩级别 1-9（6 是速度与压缩率的平衡点）
    gzip_buffers      16 8k;
    gzip_http_version 1.1;
    gzip_min_length   256;          # 小于 256 字节不压缩（压缩反而变大）

    # 需要压缩的 MIME 类型
    gzip_types
        text/plain
        text/css
        text/xml
        text/javascript
        application/json
        application/javascript
        application/x-javascript
        application/xml
        application/xml+rss
        application/atom+xml
        image/svg+xml
        font/truetype
        font/opentype
        application/vnd.ms-fontobject;

    # 不压缩 IE6（IE6 的 gzip 有 bug）
    gzip_disable "MSIE [1-6]\.";
}
```

### 预压缩静态文件（gzip_static）

```bash
# 提前生成 .gz 文件（比动态压缩更快）
gzip -9 /var/www/html/app.js        # 生成 app.js.gz
gzip -9 /var/www/html/style.css     # 生成 style.css.gz
```

```nginx
location ~* \.(js|css)$ {
    gzip_static on;    # 优先使用预压缩的 .gz 文件
    expires 1y;
    add_header Cache-Control "public, immutable";
}
```

---

## 16. WebSocket 代理

```nginx
http {
    # 升级连接映射（必须放在 http 块）
    map $http_upgrade $connection_upgrade {
        default upgrade;
        ''      close;
    }

    server {
        listen 80;
        server_name ws.example.com;

        location /ws/ {
            proxy_pass http://websocket_backend;

            # WebSocket 必须的 Header
            proxy_http_version 1.1;
            proxy_set_header Upgrade    $http_upgrade;
            proxy_set_header Connection $connection_upgrade;

            # 传递真实 IP
            proxy_set_header Host       $host;
            proxy_set_header X-Real-IP  $remote_addr;

            # WebSocket 超时（心跳间隔 < 此值）
            proxy_read_timeout 3600s;   # 1 小时
            proxy_send_timeout 3600s;
        }
    }
}
```

---

## 17. Stream 模块（TCP/UDP 代理）

```nginx
# nginx.conf 顶层（与 http 块平级）
stream {
    # ---- MySQL 负载均衡 ----
    upstream mysql_pool {
        server 10.0.0.1:3306 weight=3;
        server 10.0.0.2:3306 weight=1;
    }

    server {
        listen 3306;
        proxy_pass    mysql_pool;
        proxy_timeout 600s;
    }

    # ---- Redis 代理 ----
    server {
        listen 6379;
        proxy_pass 10.0.0.10:6379;
    }

    # ---- TCP SSL 终止 ----
    server {
        listen 443 ssl;
        ssl_certificate     /etc/nginx/ssl/cert.pem;
        ssl_certificate_key /etc/nginx/ssl/key.pem;
        proxy_pass backend_servers;
    }

    # ---- UDP（DNS 代理）----
    server {
        listen 53 udp;
        proxy_pass 8.8.8.8:53;
        proxy_timeout 1s;
        proxy_responses 1;
    }
}
```

---

## 18. 性能调优

### Worker 进程调优

```nginx
# 通常设为 CPU 核心数，或 auto（自动检测）
worker_processes auto;

# 绑定 Worker 到指定 CPU 核心（减少 CPU 缓存失效）
worker_cpu_affinity auto;

# 单个 Worker 最大文件描述符数（需配合系统 ulimit）
worker_rlimit_nofile 65535;

events {
    # 每个 Worker 最大并发连接数
    # 总并发 = worker_processes × worker_connections
    worker_connections 4096;

    # 使用 epoll（Linux 最高效的 I/O 多路复用）
    use epoll;

    # 一次尽可能多接受新连接
    multi_accept on;
}
```

### 系统级调优

```bash
# /etc/sysctl.conf
# 扩大 TCP 连接队列
net.core.somaxconn = 65535
net.ipv4.tcp_max_syn_backlog = 65535

# 快速回收 TIME_WAIT 连接
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_fin_timeout = 15

# 扩大端口范围（反向代理需要大量临时端口）
net.ipv4.ip_local_port_range = 1024 65535

# 文件描述符限制
# /etc/security/limits.conf
nginx soft nofile 65535
nginx hard nofile 65535
```

### HTTP 连接优化

```nginx
http {
    # 开启 sendfile（零拷贝，内核直接将文件发到网卡，不经用户空间）
    sendfile    on;

    # 配合 sendfile 使用，将 TCP 包凑满再发送（减少包数量）
    tcp_nopush  on;

    # 禁用 Nagle 算法，减少延迟（适合小数据频繁发送，如 API）
    tcp_nodelay on;

    # Keep-Alive（复用 TCP 连接）
    keepalive_timeout  65s;      # 连接保持时长
    keepalive_requests 1000;     # 单连接最多处理请求数

    # 请求头缓冲区
    client_header_buffer_size 1k;
    large_client_header_buffers 4 8k;

    # 响应缓冲区
    output_buffers   2 32k;
    postpone_output  1460;

    # 隐藏版本号
    server_tokens off;

    # 哈希表大小（影响 server_name 查找速度）
    server_names_hash_bucket_size 64;
    server_names_hash_max_size    512;
}
```

### 上游连接池

```nginx
upstream backend {
    server 10.0.0.1:8080;
    server 10.0.0.2:8080;

    # 连接池：避免每次请求都建立 TCP 连接
    keepalive         64;       # 保持 64 个空闲连接
    keepalive_requests 10000;
    keepalive_timeout  60s;
}

server {
    location / {
        proxy_pass http://backend;
        proxy_http_version 1.1;
        proxy_set_header Connection "";  # 清除 Connection: close（启用长连接）
    }
}
```

---

## 19. 常用运维命令

```bash
# ---- 进程管理 ----
nginx                          # 启动
nginx -s stop                  # 立即停止（强制）
nginx -s quit                  # 优雅停止（等待处理中的请求完成）
nginx -s reload                # 热重载配置（不中断服务）
nginx -s reopen                # 重新打开日志文件（配合 logrotate）

# systemd 管理（推荐）
systemctl start   nginx
systemctl stop    nginx
systemctl reload  nginx        # 热重载
systemctl restart nginx        # 重启
systemctl status  nginx
systemctl enable  nginx        # 开机自启

# ---- 配置测试 ----
nginx -t                       # 测试配置文件语法
nginx -T                       # 测试并打印完整配置

# ---- 信号 ----
kill -HUP  $(cat /var/run/nginx.pid)   # 热重载
kill -USR1 $(cat /var/run/nginx.pid)   # 重新打开日志文件
kill -USR2 $(cat /var/run/nginx.pid)   # 热升级（新版本 binary 替换）
kill -WINCH $(cat /var/run/nginx.pid)  # 优雅停止 Worker

# ---- 查看状态（需开启 stub_status 模块）----
curl http://localhost/nginx_status
# Active connections: 291
# server accepts handled requests
#  16630948 16630948 31070465
# Reading: 6 Writing: 179 Waiting: 106

# ---- 日志分析 ----
# 实时查看访问日志
tail -f /var/log/nginx/access.log

# 统计 TOP 10 访问 IP
awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head -10

# 统计 TOP 10 访问 URI
awk '{print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head -10

# 统计响应状态码分布
awk '{print $9}' /var/log/nginx/access.log | sort | uniq -c | sort -rn

# 统计每分钟请求数
awk '{print $4}' /var/log/nginx/access.log | cut -c 1-17 | sort | uniq -c

# 查找耗时超过 3 秒的请求（JSON 格式日志）
cat /var/log/nginx/access.log | jq 'select(.request_time > 3)'
```

### stub_status 监控接口

```nginx
server {
    listen 8080;
    allow 127.0.0.1;
    deny  all;

    location /nginx_status {
        stub_status;
    }
}
```

---

## 20. 实战：微服务网关配置

这是一份完整的生产级 Nginx 配置，整合了前面所有知识点。

```nginx
# /etc/nginx/nginx.conf
user  nginx;
worker_processes  auto;
worker_rlimit_nofile 65535;

error_log /var/log/nginx/error.log warn;
pid       /var/run/nginx.pid;

events {
    worker_connections 4096;
    use epoll;
    multi_accept on;
}

http {
    include      mime.types;
    default_type application/octet-stream;

    # ---- 性能 ----
    sendfile        on;
    tcp_nopush      on;
    tcp_nodelay     on;
    keepalive_timeout  65;
    keepalive_requests 1000;
    server_tokens  off;

    # ---- 缓冲区 ----
    client_max_body_size        20m;
    client_body_buffer_size     128k;
    client_header_buffer_size   1k;
    large_client_header_buffers 4 8k;
    proxy_buffer_size           4k;
    proxy_buffers               8 16k;
    proxy_busy_buffers_size     32k;

    # ---- 超时 ----
    client_body_timeout   12s;
    client_header_timeout 12s;
    send_timeout          10s;

    # ---- 日志格式 ----
    log_format json escape=json
        '{"ts":"$time_iso8601","ip":"$remote_addr","method":"$request_method",'
        '"uri":"$uri","status":$status,"bytes":$body_bytes_sent,'
        '"rt":"$request_time","urt":"$upstream_response_time",'
        '"ua":"$http_user_agent","rid":"$request_id"}';

    access_log /var/log/nginx/access.log json buffer=32k flush=5s;

    # ---- Gzip ----
    gzip              on;
    gzip_vary         on;
    gzip_comp_level   6;
    gzip_min_length   256;
    gzip_types text/plain text/css application/json application/javascript
                text/xml application/xml image/svg+xml;

    # ---- 代理缓存 ----
    proxy_cache_path /var/cache/nginx levels=1:2
        keys_zone=api_cache:20m max_size=5g inactive=30m use_temp_path=off;

    # ---- 限流 ----
    limit_req_zone  $binary_remote_addr zone=global:20m    rate=200r/s;
    limit_req_zone  $binary_remote_addr zone=login:10m     rate=5r/m;
    limit_req_zone  $binary_remote_addr zone=api:20m       rate=50r/s;
    limit_conn_zone $binary_remote_addr zone=conn_limit:10m;
    limit_req_status  429;
    limit_conn_status 429;

    # ---- 上游服务 ----
    upstream user_service {
        least_conn;
        server user-service-1:8080 weight=3 max_fails=3 fail_timeout=30s;
        server user-service-2:8080 weight=3 max_fails=3 fail_timeout=30s;
        server user-service-3:8080 weight=1 backup;
        keepalive 64;
    }

    upstream order_service {
        least_conn;
        server order-service-1:8080 max_fails=3 fail_timeout=30s;
        server order-service-2:8080 max_fails=3 fail_timeout=30s;
        keepalive 32;
    }

    upstream pay_service {
        server pay-service-1:8080 max_fails=2 fail_timeout=60s;
        server pay-service-2:8080 max_fails=2 fail_timeout=60s;
        keepalive 16;
    }

    upstream static_service {
        server static-1:8080;
        server static-2:8080;
        keepalive 128;
    }

    # ---- WebSocket 升级 ----
    map $http_upgrade $connection_upgrade {
        default upgrade;
        ''      close;
    }

    # ---- HTTP → HTTPS 重定向 ----
    server {
        listen 80;
        server_name example.com api.example.com;
        return 301 https://$host$request_uri;
    }

    # ---- 主 HTTPS 服务器 ----
    server {
        listen 443 ssl http2;
        server_name example.com;

        # SSL
        ssl_certificate     /etc/nginx/ssl/example.com.crt;
        ssl_certificate_key /etc/nginx/ssl/example.com.key;
        ssl_protocols       TLSv1.2 TLSv1.3;
        ssl_ciphers         ECDHE-RSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-CHACHA20-POLY1305;
        ssl_session_cache   shared:SSL:10m;
        ssl_session_timeout 1d;
        ssl_stapling        on;
        ssl_stapling_verify on;

        # 安全头
        add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
        add_header X-Frame-Options           "SAMEORIGIN"                           always;
        add_header X-Content-Type-Options    "nosniff"                              always;
        add_header X-XSS-Protection          "1; mode=block"                        always;
        add_header X-Request-ID              $request_id                            always;

        # 全局限流
        limit_conn conn_limit 30;
        limit_req  zone=global burst=100 nodelay;

        # 健康检查
        location = /health {
            access_log off;
            return 200 '{"status":"ok"}';
            add_header Content-Type application/json;
        }

        # 前端静态资源
        location / {
            root /var/www/html;
            try_files $uri $uri/ /index.html;

            location ~* \.(js|css)$ {
                expires 1y;
                add_header Cache-Control "public, immutable";
                access_log off;
            }

            location ~* \.(jpg|jpeg|png|gif|webp|svg|ico|woff|woff2)$ {
                expires 30d;
                add_header Cache-Control "public";
                access_log off;
            }
        }

        # 用户服务 API
        location /api/v1/users {
            limit_req zone=api burst=30 nodelay;

            proxy_pass         http://user_service;
            proxy_http_version 1.1;
            proxy_set_header   Connection         "";
            proxy_set_header   Host               $host;
            proxy_set_header   X-Real-IP          $remote_addr;
            proxy_set_header   X-Forwarded-For    $proxy_add_x_forwarded_for;
            proxy_set_header   X-Forwarded-Proto  $scheme;
            proxy_set_header   X-Request-ID       $request_id;
            proxy_connect_timeout 5s;
            proxy_read_timeout    30s;

            # GET 请求缓存
            proxy_cache            api_cache;
            proxy_cache_methods    GET HEAD;
            proxy_cache_key        "$scheme$host$uri$args";
            proxy_cache_valid      200 5m;
            proxy_cache_bypass     $http_authorization;
            proxy_no_cache         $http_authorization;
            proxy_cache_use_stale  error timeout updating;
            proxy_cache_lock       on;
            add_header X-Cache-Status $upstream_cache_status;
        }

        # 登录接口严格限流
        location = /api/v1/auth/login {
            limit_req zone=login burst=3;

            proxy_pass         http://user_service;
            proxy_http_version 1.1;
            proxy_set_header   Connection        "";
            proxy_set_header   Host              $host;
            proxy_set_header   X-Real-IP         $remote_addr;
            proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;
            proxy_connect_timeout 5s;
            proxy_read_timeout    10s;
        }

        # 订单服务 API
        location /api/v1/orders {
            limit_req zone=api burst=20 nodelay;

            proxy_pass         http://order_service;
            proxy_http_version 1.1;
            proxy_set_header   Connection        "";
            proxy_set_header   Host              $host;
            proxy_set_header   X-Real-IP         $remote_addr;
            proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;
            proxy_set_header   X-Forwarded-Proto $scheme;
            proxy_set_header   X-Request-ID      $request_id;
            proxy_connect_timeout 5s;
            proxy_read_timeout    60s;   # 订单处理可能较慢
        }

        # 支付服务（严格超时控制）
        location /api/v1/pay {
            limit_req zone=api burst=10 nodelay;

            proxy_pass         http://pay_service;
            proxy_http_version 1.1;
            proxy_set_header   Connection        "";
            proxy_set_header   Host              $host;
            proxy_set_header   X-Real-IP         $remote_addr;
            proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;
            proxy_set_header   X-Forwarded-Proto $scheme;
            proxy_connect_timeout 3s;
            proxy_read_timeout    30s;
            client_max_body_size  1m;
        }

        # WebSocket
        location /ws/ {
            proxy_pass         http://user_service;
            proxy_http_version 1.1;
            proxy_set_header   Upgrade    $http_upgrade;
            proxy_set_header   Connection $connection_upgrade;
            proxy_set_header   Host       $host;
            proxy_set_header   X-Real-IP  $remote_addr;
            proxy_read_timeout 3600s;
            proxy_send_timeout 3600s;
        }

        # 禁止访问敏感路径
        location ~ /\.(git|env|htpasswd|DS_Store) {
            deny all;
            return 404;
        }

        # 自定义错误页
        error_page 429 /errors/429.json;
        error_page 502 503 504 /errors/5xx.json;

        location /errors/ {
            internal;
            root /var/www;
        }
    }

    # ---- API 子域 ----
    server {
        listen 443 ssl http2;
        server_name api.example.com;

        ssl_certificate     /etc/nginx/ssl/example.com.crt;
        ssl_certificate_key /etc/nginx/ssl/example.com.key;
        ssl_protocols       TLSv1.2 TLSv1.3;
        ssl_session_cache   shared:SSL:10m;

        location / {
            proxy_pass         http://user_service;
            proxy_http_version 1.1;
            proxy_set_header   Connection        "";
            proxy_set_header   Host              $host;
            proxy_set_header   X-Real-IP         $remote_addr;
            proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;
            proxy_set_header   X-Forwarded-Proto $scheme;
        }
    }

    # ---- 监控接口（内网访问）----
    server {
        listen 8080;
        allow 10.0.0.0/8;
        allow 127.0.0.1;
        deny  all;

        location /nginx_status {
            stub_status;
            access_log off;
        }

        location /metrics {
            proxy_pass http://prometheus_exporter:9113;
        }
    }
}
```

---

## 快速参考

```bash
# 常用操作速查
nginx -t                        # 检查配置
nginx -s reload                 # 热重载（不中断服务）
systemctl reload nginx          # 推荐方式

# 查看当前连接数
ss -s
netstat -an | grep :80 | wc -l

# 查看 Nginx 进程
ps aux | grep nginx

# 实时错误日志
tail -f /var/log/nginx/error.log

# 查找配置中的语法问题
nginx -T 2>&1 | grep -n "error\|warn"
```

```
配置加载优先级（从高到低）：
  location = /exact          # 精确匹配
  location ^~ /prefix        # 前缀阻止正则
  location ~ \.php$          # 正则（按顺序）
  location /prefix           # 最长前缀匹配
```
