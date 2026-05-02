# Docker

Docker 是一个容器化平台，将应用及其所有依赖打包成镜像，在任何环境中以相同方式运行，解决"在我机器上能跑"的问题。

**核心概念：**

| 概念 | 说明 |
|------|------|
| **镜像（Image）** | 只读模板，包含运行应用所需的一切（代码、运行时、库、配置） |
| **容器（Container）** | 镜像的运行实例，是隔离的进程 |
| **Dockerfile** | 构建镜像的脚本，描述每一层的操作 |
| **Registry** | 镜像仓库，如 Docker Hub、阿里云镜像仓库 |
| **Volume** | 数据卷，持久化容器数据 |
| **Network** | 容器网络，控制容器间通信 |
| **Compose** | 多容器编排工具，用 YAML 描述服务依赖 |

---

## 安装

### macOS / Windows

下载 [Docker Desktop](https://www.docker.com/products/docker-desktop/)，安装后启动即可。

### Linux（Ubuntu）

```shell
# 一键安装
curl -fsSL https://get.docker.com | sh

# 将当前用户加入 docker 组（免 sudo）
sudo usermod -aG docker $USER
newgrp docker

# 启动服务
sudo systemctl enable docker
sudo systemctl start docker

# 验证
docker --version
docker compose version
```

### 配置镜像加速（国内）

```json
// /etc/docker/daemon.json（Linux）
// ~/.docker/daemon.json（macOS Docker Desktop）
{
  "registry-mirrors": [
    "https://mirror.ccs.tencentyun.com",
    "https://docker.mirrors.ustc.edu.cn"
  ]
}
```

```shell
sudo systemctl daemon-reload
sudo systemctl restart docker
```

---

## 常用命令

### 镜像管理

```shell
# 搜索镜像
docker search nginx

# 拉取镜像（默认 latest 标签）
docker pull nginx
docker pull nginx:1.25-alpine
docker pull node:20-alpine

# 查看本地镜像
docker images
docker image ls

# 删除镜像
docker rmi nginx:latest
docker image rm nginx:1.25-alpine

# 删除所有未使用的镜像
docker image prune -a

# 查看镜像详情
docker inspect nginx:latest

# 镜像打标签
docker tag my-app:latest my-app:1.0.0
docker tag my-app:latest registry.cn-hangzhou.aliyuncs.com/myns/my-app:1.0.0

# 推送镜像到仓库
docker login registry.cn-hangzhou.aliyuncs.com
docker push registry.cn-hangzhou.aliyuncs.com/myns/my-app:1.0.0
```

### 容器生命周期

```shell
# 运行容器
docker run nginx                          # 前台运行
docker run -d nginx                       # 后台运行（detach）
docker run -d --name my-nginx nginx       # 指定容器名
docker run -d -p 8080:80 nginx            # 端口映射（宿主机:容器）
docker run -d -p 127.0.0.1:8080:80 nginx  # 只绑定本地回环

# 运行并进入交互模式
docker run -it ubuntu bash
docker run -it --rm ubuntu bash           # --rm：退出后自动删除容器

# 查看运行中的容器
docker ps
docker ps -a                # 包含停止的容器
docker ps --format "table {{.ID}}\t{{.Names}}\t{{.Status}}\t{{.Ports}}"

# 启动/停止/重启
docker start my-nginx
docker stop my-nginx        # 优雅停止（SIGTERM，等待 10s）
docker stop -t 30 my-nginx  # 等待 30s 后强制杀死
docker kill my-nginx        # 强制停止（SIGKILL）
docker restart my-nginx

# 删除容器
docker rm my-nginx           # 删除已停止的容器
docker rm -f my-nginx        # 强制删除（运行中也可以）
docker container prune       # 删除所有已停止的容器

# 进入运行中的容器
docker exec -it my-nginx bash
docker exec -it my-nginx sh   # Alpine 镜像只有 sh

# 以 root 身份进入
docker exec -it --user root my-nginx bash
```

### 日志与监控

```shell
# 查看日志
docker logs my-nginx
docker logs -f my-nginx         # 持续跟踪（类似 tail -f）
docker logs --tail 100 my-nginx # 最后 100 行
docker logs --since 1h my-nginx # 最近 1 小时的日志
docker logs -f --since 2024-01-01T00:00:00 my-nginx

# 查看容器资源占用
docker stats
docker stats my-nginx --no-stream  # 只显示一次

# 查看容器进程
docker top my-nginx

# 查看容器详细信息
docker inspect my-nginx

# 查看容器端口映射
docker port my-nginx
```

### 文件传输

```shell
# 宿主机 → 容器
docker cp ./config.json my-nginx:/etc/nginx/config.json

# 容器 → 宿主机
docker cp my-nginx:/etc/nginx/nginx.conf ./nginx.conf
```

---

## Dockerfile

Dockerfile 是构建镜像的脚本，每条指令创建一个新的镜像层。

### 常用指令

```dockerfile
# 基础镜像（必须是第一条指令）
FROM node:20-alpine

# 指定构建阶段名称（多阶段构建）
FROM node:20-alpine AS builder

# 设置工作目录（后续指令都在此目录执行）
WORKDIR /app

# 复制文件（宿主机 → 镜像）
COPY package*.json ./
COPY . .
COPY --from=builder /app/dist ./dist   # 从其他构建阶段复制

# 运行命令（构建时执行，每条 RUN 创建一层）
RUN npm install
RUN npm run build

# 合并为一条 RUN（减少层数，节省体积）
RUN apt-get update && \
    apt-get install -y curl wget && \
    rm -rf /var/lib/apt/lists/*

# 设置环境变量
ENV NODE_ENV=production
ENV PORT=3000

# 声明构建参数（docker build --build-arg 传入）
ARG APP_VERSION=1.0.0
RUN echo "构建版本：$APP_VERSION"

# 暴露端口（文档性质，不实际映射端口）
EXPOSE 3000

# 挂载点（声明数据卷）
VOLUME ["/app/data", "/app/logs"]

# 设置容器启动时的用户
USER node

# 设置启动命令（两种形式）
# CMD：可被 docker run 末尾的命令覆盖
CMD ["node", "server.js"]           # exec 形式（推荐）
CMD node server.js                   # shell 形式

# ENTRYPOINT：不会被覆盖，用于固定入口
ENTRYPOINT ["docker-entrypoint.sh"]
# 组合使用：ENTRYPOINT 是可执行文件，CMD 是默认参数
ENTRYPOINT ["nginx"]
CMD ["-g", "daemon off;"]

# 健康检查
HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
    CMD curl -f http://localhost:3000/health || exit 1

# 设置构建时执行（不影响镜像层）
LABEL maintainer="dev@example.com"
LABEL version="1.0.0"
```

### Node.js 应用示例

```dockerfile
# Dockerfile
FROM node:20-alpine

WORKDIR /app

# 先复制依赖文件（利用缓存层）
COPY package*.json ./
RUN npm ci --only=production

# 再复制源码
COPY . .

# 非 root 用户运行（安全）
USER node

EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=3s \
    CMD wget -qO- http://localhost:3000/health || exit 1

CMD ["node", "src/index.js"]
```

### 多阶段构建（Multi-stage Build）

多阶段构建可以显著减小最终镜像体积：

```dockerfile
# 阶段 1：构建（包含完整开发工具）
FROM node:20-alpine AS builder

WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# 阶段 2：运行时（只包含生产需要的文件）
FROM node:20-alpine AS runner

WORKDIR /app

ENV NODE_ENV=production

# 只从构建阶段复制必要文件
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/package.json ./

USER node
EXPOSE 3000
CMD ["node", "dist/index.js"]
```

### Next.js 生产镜像

```dockerfile
FROM node:20-alpine AS base

# 阶段 1：安装依赖
FROM base AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci

# 阶段 2：构建
FROM base AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
ENV NEXT_TELEMETRY_DISABLED=1
RUN npm run build

# 阶段 3：运行时（最终镜像）
FROM base AS runner
WORKDIR /app

ENV NODE_ENV=production
ENV NEXT_TELEMETRY_DISABLED=1

RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nextjs

COPY --from=builder /app/public ./public
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static

USER nextjs
EXPOSE 3000
ENV PORT=3000
ENV HOSTNAME="0.0.0.0"

CMD ["node", "server.js"]
```

### Go 应用（极致精简）

```dockerfile
# 阶段 1：编译
FROM golang:1.23-alpine AS builder

WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-w -s" -o server ./cmd/server

# 阶段 2：scratch 空镜像（仅包含二进制文件）
FROM scratch

COPY --from=builder /app/server /server
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/

EXPOSE 8080
ENTRYPOINT ["/server"]
```

### .dockerignore

```text
# .dockerignore
node_modules
.git
.gitignore
*.md
dist
.env
.env.*
*.log
coverage
.DS_Store
__pycache__
.pytest_cache
*.pyc
```

---

## 数据卷（Volume）

容器删除后数据默认丢失，使用 Volume 持久化数据：

```shell
# 创建命名卷
docker volume create mydata

# 查看卷列表
docker volume ls

# 查看卷详情
docker volume inspect mydata

# 删除卷
docker volume rm mydata
docker volume prune  # 删除所有未使用的卷

# 挂载命名卷（推荐）
docker run -d \
    --name postgres \
    -v pgdata:/var/lib/postgresql/data \
    postgres:16-alpine

# 挂载绑定挂载（Bind Mount，宿主机目录）
docker run -d \
    --name my-app \
    -v $(pwd)/src:/app/src \    # 开发时用，代码实时同步
    -v $(pwd)/config:/app/config:ro \  # :ro 只读
    my-app-image

# tmpfs 挂载（内存，容器停止后消失）
docker run -d --tmpfs /tmp:rw,size=64m my-app
```

---

## 网络（Network）

```shell
# 查看网络列表
docker network ls

# 创建自定义网络
docker network create mynet
docker network create --driver bridge --subnet 172.20.0.0/16 mynet

# 连接/断开容器到网络
docker network connect mynet my-app
docker network disconnect mynet my-app

# 删除网络
docker network rm mynet
docker network prune   # 删除所有未使用的网络

# 同一网络的容器可以用容器名互相访问
docker run -d --name redis --network mynet redis:7-alpine
docker run -d --name my-app --network mynet \
    -e REDIS_HOST=redis \    # 直接用容器名作为主机名
    my-app-image
```

**Docker 内置网络类型：**

| 类型 | 说明 |
|------|------|
| `bridge` | 默认，同一 bridge 网络的容器可互通 |
| `host` | 直接使用宿主机网络（Linux 限定，性能高） |
| `none` | 无网络，完全隔离 |
| `overlay` | 跨主机通信（Swarm 使用） |

---

## Docker Compose

Docker Compose 用于编排多容器应用，用一个 YAML 文件描述所有服务。

### 基础结构

```yaml
# docker-compose.yml
services:
  # 服务名（容器间用此名互相访问）
  app:
    build: .                         # 使用当前目录 Dockerfile 构建
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
      - DATABASE_URL=postgresql://postgres:password@db:5432/mydb
    depends_on:
      db:
        condition: service_healthy   # 等待 db 健康检查通过
      redis:
        condition: service_started
    volumes:
      - ./logs:/app/logs
    restart: unless-stopped
    networks:
      - backend

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
      POSTGRES_DB: mydb
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql  # 初始化脚本
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
    ports:
      - "5432:5432"   # 开发时暴露，生产环境可去掉
    networks:
      - backend

  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes --requirepass mypassword
    volumes:
      - redisdata:/data
    ports:
      - "6379:6379"
    networks:
      - backend

  nginx:
    image: nginx:1.25-alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/certs:/etc/nginx/certs:ro
    depends_on:
      - app
    networks:
      - backend

volumes:
  pgdata:
  redisdata:

networks:
  backend:
    driver: bridge
```

### Compose 常用命令

```shell
# 启动所有服务（后台运行）
docker compose up -d

# 启动并重新构建镜像
docker compose up -d --build

# 只启动指定服务
docker compose up -d app redis

# 查看运行状态
docker compose ps

# 查看日志
docker compose logs -f
docker compose logs -f app       # 指定服务
docker compose logs --tail 50 app

# 停止服务（不删除容器和卷）
docker compose stop

# 停止并删除容器、网络
docker compose down

# 停止并删除容器、网络、镜像、数据卷
docker compose down -v --rmi all

# 重启某个服务
docker compose restart app

# 扩缩容（需要服务无状态）
docker compose up -d --scale app=3

# 进入容器
docker compose exec app sh
docker compose exec db psql -U postgres

# 执行一次性命令（不复用已有容器）
docker compose run --rm app npm run migrate

# 查看服务配置（合并后的完整配置）
docker compose config

# 拉取最新镜像
docker compose pull
```

### 多环境配置

```yaml
# docker-compose.yml（基础配置）
services:
  app:
    image: my-app
    environment:
      - NODE_ENV=production
```

```yaml
# docker-compose.override.yml（本地开发自动合并）
services:
  app:
    build: .
    volumes:
      - .:/app
      - /app/node_modules
    environment:
      - NODE_ENV=development
    command: npm run dev
    ports:
      - "3000:3000"
```

```yaml
# docker-compose.prod.yml（生产环境）
services:
  app:
    image: registry.example.com/my-app:${VERSION}
    deploy:
      replicas: 3
      resources:
        limits:
          cpus: "0.5"
          memory: 512M
```

```shell
# 开发环境（自动合并 override）
docker compose up -d

# 生产环境
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

### 使用 .env 文件

```shell
# .env（Compose 自动加载）
POSTGRES_USER=postgres
POSTGRES_PASSWORD=secret123
POSTGRES_DB=myapp
APP_PORT=3000
VERSION=1.2.0
```

```yaml
# docker-compose.yml 中引用
services:
  app:
    ports:
      - "${APP_PORT}:3000"
  db:
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
```

---

## 常见应用部署示例

### PostgreSQL

```yaml
services:
  postgres:
    image: postgres:16-alpine
    container_name: postgres
    restart: unless-stopped
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: mydb
      PGDATA: /var/lib/postgresql/data/pgdata
    volumes:
      - pgdata:/var/lib/postgresql/data
    ports:
      - "127.0.0.1:5432:5432"   # 只绑定本地，不暴露到公网
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres -d mydb"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s

volumes:
  pgdata:
```

### Redis

```yaml
services:
  redis:
    image: redis:7-alpine
    container_name: redis
    restart: unless-stopped
    command: >
      redis-server
      --requirepass ${REDIS_PASSWORD}
      --appendonly yes
      --maxmemory 256mb
      --maxmemory-policy allkeys-lru
    volumes:
      - redisdata:/data
    ports:
      - "127.0.0.1:6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "${REDIS_PASSWORD}", "ping"]
      interval: 10s
      timeout: 5s
      retries: 3

volumes:
  redisdata:
```

### Nginx 反向代理

```nginx
# nginx/nginx.conf
upstream app {
    server app:3000;
}

server {
    listen 80;
    server_name example.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name example.com;

    ssl_certificate     /etc/nginx/certs/fullchain.pem;
    ssl_certificate_key /etc/nginx/certs/privkey.pem;

    location / {
        proxy_pass http://app;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
    }

    location /static/ {
        alias /app/public/;
        expires 30d;
        add_header Cache-Control "public, immutable";
    }
}
```

### Traefik（自动化反向代理 + HTTPS）

```yaml
# docker-compose.yml
services:
  traefik:
    image: traefik:v3.0
    restart: unless-stopped
    command:
      - "--api.insecure=true"
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"
      - "--certificatesresolvers.letsencrypt.acme.httpchallenge=true"
      - "--certificatesresolvers.letsencrypt.acme.httpchallenge.entrypoint=web"
      - "--certificatesresolvers.letsencrypt.acme.email=admin@example.com"
      - "--certificatesresolvers.letsencrypt.acme.storage=/letsencrypt/acme.json"
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock:ro"
      - "letsencrypt:/letsencrypt"

  app:
    image: my-app:latest
    restart: unless-stopped
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.app.rule=Host(`example.com`)"
      - "traefik.http.routers.app.entrypoints=websecure"
      - "traefik.http.routers.app.tls.certresolver=letsencrypt"
      - "traefik.http.services.app.loadbalancer.server.port=3000"
      # 自动 HTTP → HTTPS 重定向
      - "traefik.http.routers.app-http.rule=Host(`example.com`)"
      - "traefik.http.routers.app-http.entrypoints=web"
      - "traefik.http.routers.app-http.middlewares=https-redirect"
      - "traefik.http.middlewares.https-redirect.redirectscheme.scheme=https"

volumes:
  letsencrypt:
```

---

## 构建优化

### 充分利用缓存层

Dockerfile 中，**变化频繁的指令放后面**，让不变的层尽量缓存：

```dockerfile
# ✗ 差：每次代码改动都重新安装依赖
COPY . .
RUN npm install

# ✓ 好：依赖文件不变时，npm install 直接用缓存
COPY package*.json ./
RUN npm install
COPY . .
```

### 使用 BuildKit

```shell
# 开启 BuildKit（Docker 18.09+，已默认开启）
DOCKER_BUILDKIT=1 docker build .

# 并行构建多阶段
docker buildx build --platform linux/amd64,linux/arm64 -t my-app:latest --push .
```

### 精简镜像体积

```dockerfile
# 选择 Alpine 或 slim 变体
FROM node:20-alpine    # ~50MB vs node:20 的 ~1GB
FROM python:3.12-slim  # ~130MB vs python:3.12 的 ~900MB

# Alpine 安装包后清理缓存
RUN apk add --no-cache curl wget

# Debian/Ubuntu 清理 apt 缓存
RUN apt-get update && \
    apt-get install -y --no-install-recommends curl && \
    rm -rf /var/lib/apt/lists/*

# 不安装开发依赖
RUN npm ci --only=production
RUN pip install --no-cache-dir -r requirements.txt
```

---

## 资源限制

```shell
# 运行时限制资源
docker run -d \
    --memory 512m \           # 内存上限
    --memory-swap 512m \      # 禁用 swap（与 memory 相同）
    --cpus 0.5 \              # CPU 配额（50% 的一个核）
    --cpu-shares 512 \        # CPU 相对权重（默认 1024）
    my-app
```

```yaml
# Compose 中限制资源
services:
  app:
    deploy:
      resources:
        limits:
          cpus: "0.50"
          memory: 512M
        reservations:
          cpus: "0.25"
          memory: 256M
```

---

## 安全实践

```dockerfile
# 1. 使用非 root 用户运行
RUN addgroup --system --gid 1001 appgroup && \
    adduser  --system --uid 1001 --ingroup appgroup appuser
USER appuser

# 2. 使用确定版本标签，不用 latest
FROM node:20.11.1-alpine3.19

# 3. 只复制必要文件（配合 .dockerignore）
COPY --chown=appuser:appgroup src/ ./src/

# 4. 不在镜像中存储密钥（运行时通过环境变量传入）
# ✗  ENV DATABASE_PASSWORD=secret
# ✓  运行时传入：docker run -e DATABASE_PASSWORD=$SECRET

# 5. 只读文件系统
# docker run --read-only -v /tmp:/tmp my-app
```

---

## 清理系统

```shell
# 查看磁盘使用
docker system df
docker system df -v  # 详细

# 清理所有未使用资源（镜像、容器、网络、构建缓存）
docker system prune

# 包含未使用的数据卷
docker system prune -a --volumes

# 只清理构建缓存
docker builder prune

# 按需清理
docker container prune   # 已停止的容器
docker image prune -a    # 未使用的镜像
docker volume prune      # 未挂载的卷
docker network prune     # 未使用的网络
```

---

## 调试技巧

```shell
# 进入容器排查问题
docker exec -it <container> sh

# 查看容器文件系统变更
docker diff <container>

# 将容器当前状态保存为新镜像（调试用）
docker commit <container> debug-image:v1

# 查看镜像分层和每层大小
docker history my-app:latest

# 实时事件流
docker events

# 导出/导入镜像（离线传输）
docker save my-app:latest | gzip > my-app.tar.gz
docker load < my-app.tar.gz

# 导出容器文件系统
docker export my-container > container.tar

# 在构建中临时调试某一层
docker build --target builder -t debug .
docker run -it debug sh
```

---

## 常用命令速查

```shell
# ── 镜像 ──────────────────────────────
docker pull <image>                   # 拉取
docker push <image>                   # 推送
docker build -t <name>:<tag> .        # 构建
docker images                         # 列出
docker rmi <image>                    # 删除
docker image prune -a                 # 清理未使用

# ── 容器 ──────────────────────────────
docker run -d -p <h>:<c> --name <n> <image>  # 运行
docker ps / docker ps -a              # 列出
docker start / stop / restart <c>    # 启停
docker rm -f <c>                      # 删除
docker exec -it <c> sh                # 进入
docker logs -f <c>                    # 日志
docker stats                          # 监控
docker inspect <c>                    # 详情
docker cp <src> <c>:<dst>             # 复制

# ── Compose ───────────────────────────
docker compose up -d [--build]        # 启动
docker compose down [-v]              # 停止+清理
docker compose ps                     # 状态
docker compose logs -f [service]      # 日志
docker compose exec <svc> sh          # 进入
docker compose restart <svc>          # 重启
docker compose pull                   # 拉取最新

# ── 清理 ──────────────────────────────
docker system prune -a --volumes      # 全清
docker system df                      # 磁盘占用
```
