# Docker Compose 配置详解

## 📋 概述

本文档详细解释了 Kortix 项目中 `backend/docker-compose.yml` 文件的每一行配置，帮助理解整个容器化部署架构。

该 Docker Compose 配置定义了 **3 个核心服务**：
- **api**: 后端 API 服务
- **worker**: 后台任务处理器
- **redis**: 缓存和消息队列

---

## 🏗️ 整体架构

```
┌─────────────────────────────────────────────────────────────┐
│                    Docker Compose 架构                      │
├─────────────────┬─────────────────┬─────────────────────────┤
│   API 服务       │   Worker 服务   │      Redis 服务          │
│                 │                 │                         │
│ • HTTP API      │ • 后台任务处理    │ • 缓存存储               │
│ • 端口 8000      │ • 异步队列       │ • 消息队列               │
│ • 健康检查       │ • 多进程/线程     │ • 数据持久化             │
└─────────────────┴─────────────────┴─────────────────────────┘
                           │
                    app-network (桥接网络)
```

---

## 📝 逐行配置解析

### 🔧 服务定义开始

```yaml
services:
```
**说明**: 定义 Docker Compose 中的所有服务容器

---

## 🌐 API 服务配置

### 基础配置

```yaml
  api:
```
**说明**: 定义名为 `api` 的服务，这是主要的后端 API 服务

```yaml
    image: ghcr.io/suna-ai/suna-backend:latest
```
**说明**:
- 指定使用的 Docker 镜像
- `ghcr.io`: GitHub Container Registry
- `suna-ai/suna-backend`: 镜像仓库名称
- `latest`: 使用最新版本标签

```yaml
    platform: linux/amd64
```
**说明**:
- 强制指定容器运行平台为 Linux AMD64 架构
- 确保在 Apple Silicon (M1/M2) Mac 上也使用 x86_64 架构
- 避免架构不兼容问题

```yaml
    build:
      context: .
      dockerfile: Dockerfile
```
**说明**:
- `context: .`: 构建上下文为当前目录（backend/）
- `dockerfile: Dockerfile`: 使用当前目录下的 Dockerfile 构建镜像
- 当本地没有镜像时，会从源码构建

### 网络和端口配置

```yaml
    ports:
      - "8000:8000"
```
**说明**:
- 端口映射：主机端口 8000 映射到容器端口 8000
- 外部可通过 `http://localhost:8000` 访问 API 服务

### 环境配置

```yaml
    env_file:
      - .env
```
**说明**:
- 从 `backend/.env` 文件加载环境变量
- 包含数据库连接、API 密钥等敏感配置

```yaml
    environment:
      - REDIS_HOST=redis
      - REDIS_PORT=6379
      - REDIS_PASSWORD=
      - LOG_LEVEL=INFO
```
**说明**:
- `REDIS_HOST=redis`: Redis 主机名，使用 Docker Compose 服务名
- `REDIS_PORT=6379`: Redis 默认端口
- `REDIS_PASSWORD=`: Redis 密码为空（内网环境）
- `LOG_LEVEL=INFO`: 日志级别设置为 INFO

### 数据卷配置

```yaml
    volumes:
      - .:/app
      - /app/.venv
      - ./logs:/app/logs
```
**说明**:
- `.:/app`: 将当前目录挂载到容器的 `/app` 目录（开发模式）
- `/app/.venv`: 匿名卷，避免本地 `.venv` 覆盖容器内的虚拟环境
- `./logs:/app/logs`: 将日志目录挂载到主机，便于查看日志

### 服务依赖和重启策略

```yaml
    restart: unless-stopped
```
**说明**:
- 容器重启策略：除非手动停止，否则总是重启
- 确保服务高可用性

```yaml
    depends_on:
      redis:
        condition: service_healthy
```
**说明**:
- 依赖 Redis 服务
- 等待 Redis 健康检查通过后才启动 API 服务
- 确保启动顺序正确

### 网络配置

```yaml
    networks:
      - app-network
```
**说明**:
- 加入名为 `app-network` 的自定义网络
- 允许服务间通过服务名进行通信

### 日志配置

```yaml
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
```
**说明**:
- 使用 JSON 文件日志驱动
- 单个日志文件最大 10MB
- 最多保留 3 个日志文件
- 防止日志文件无限增长

### 健康检查

```yaml
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/api/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
```
**说明**:
- `test`: 健康检查命令，访问健康检查端点
- `interval: 30s`: 每 30 秒检查一次
- `timeout: 10s`: 单次检查超时时间 10 秒
- `retries: 3`: 失败重试 3 次后标记为不健康
- `start_period: 40s`: 启动后 40 秒内不进行健康检查

---

## 🔄 Worker 服务配置

### 基础配置

```yaml
  worker:
```
**说明**: 定义名为 `worker` 的后台任务处理服务

```yaml
    image: ghcr.io/suna-ai/suna-backend:latest
    platform: linux/amd64
    build:
      context: .
      dockerfile: Dockerfile
```
**说明**:
- 与 API 服务使用相同的镜像和构建配置
- 代码复用，减少镜像大小

### 启动命令

```yaml
    command: uv run dramatiq --skip-logging --processes 4 --threads 4 run_agent_background
```
**说明**:
- 覆盖默认启动命令
- `uv run dramatiq`: 使用 uv 运行 Dramatiq 任务队列
- `--skip-logging`: 跳过 Dramatiq 内置日志配置
- `--processes 4`: 启动 4 个工作进程
- `--threads 4`: 每个进程 4 个线程
- `run_agent_background`: 执行的任务模块

### 环境和卷配置

```yaml
    env_file:
      - .env
    volumes:
      - .:/app
      - /app/.venv
      - ./worker-logs:/app/logs
```
**说明**:
- 与 API 服务类似的配置
- `./worker-logs:/app/logs`: 使用独立的日志目录，便于区分

### 依赖和网络

```yaml
    restart: unless-stopped
    depends_on:
      redis:
        condition: service_healthy
    networks:
      - app-network
```
**说明**:
- 与 API 服务相同的重启策略和网络配置
- 同样依赖 Redis 服务健康

### Worker 健康检查

```yaml
    healthcheck:
      test: ["CMD", "uv", "run", "worker_health.py"]
      timeout: 20s
      interval: 30s
      start_period: 40s
```
**说明**:
- 使用自定义健康检查脚本 `worker_health.py`
- `timeout: 20s`: 超时时间较长，适应后台任务特性
- 其他参数与 API 服务类似

---

## 🗄️ Redis 服务配置

### 基础配置

```yaml
  redis:
```
**说明**: 定义 Redis 缓存和消息队列服务

```yaml
    image: redis:8-alpine
```
**说明**:
- 使用 Redis 8.x 版本的 Alpine Linux 镜像
- Alpine 镜像体积小，安全性高

### 端口配置

```yaml
    ports:
      - "127.0.0.1:6379:6379"
```
**说明**:
- 仅绑定到本地回环地址 `127.0.0.1`
- 不允许外部网络直接访问 Redis
- 提高安全性

### 数据持久化

```yaml
    volumes:
      - redis_data:/data
      - ./services/docker/redis.conf:/usr/local/etc/redis/redis.conf:ro
```
**说明**:
- `redis_data:/data`: 使用命名卷持久化 Redis 数据
- `./services/docker/redis.conf`: 挂载自定义 Redis 配置文件
- `:ro`: 只读挂载，防止容器修改配置文件

### Redis 启动命令

```yaml
    command: redis-server /usr/local/etc/redis/redis.conf --appendonly yes --bind 0.0.0.0 --protected-mode no --maxmemory 8gb --maxmemory-policy allkeys-lru
```
**说明**:
- `redis-server /usr/local/etc/redis/redis.conf`: 使用自定义配置启动
- `--appendonly yes`: 启用 AOF 持久化
- `--bind 0.0.0.0`: 绑定所有网络接口（容器内安全）
- `--protected-mode no`: 关闭保护模式（内网环境）
- `--maxmemory 8gb`: 最大内存使用 8GB
- `--maxmemory-policy allkeys-lru`: 内存满时使用 LRU 算法淘汰键

### Redis 健康检查

```yaml
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 10s
```
**说明**:
- 使用 `redis-cli ping` 检查 Redis 可用性
- `interval: 10s`: 检查间隔较短，快速发现问题
- `retries: 5`: 重试次数较多，避免误报
- `start_period: 10s`: Redis 启动较快，等待时间短

### 网络和重启

```yaml
    restart: unless-stopped
    networks:
      - app-network
```
**说明**:
- 与其他服务相同的重启策略和网络配置

### 日志配置

```yaml
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
```
**说明**:
- 与其他服务相同的日志轮转配置

---

## 🌐 网络配置

```yaml
networks:
  app-network:
    driver: bridge
```
**说明**:
- 定义名为 `app-network` 的自定义网络
- 使用桥接驱动（默认）
- 允许容器间通过服务名通信
- 提供网络隔离

---

## 💾 数据卷配置

```yaml
volumes:
  redis_data:
```
**说明**:
- 定义名为 `redis_data` 的命名卷
- 用于持久化 Redis 数据
- 数据在容器重启后保持不变

---

## 🔄 服务启动流程

### 启动顺序
```
1. 创建网络 app-network
2. 创建数据卷 redis_data
3. 启动 Redis 服务
4. 等待 Redis 健康检查通过
5. 并行启动 API 和 Worker 服务
6. 等待所有服务健康检查通过
```

### 服务间通信
```
API 服务 ←→ Redis 服务 (缓存、会话)
Worker 服务 ←→ Redis 服务 (任务队列)
外部请求 → API 服务 (HTTP API)
```

---

## 🛠️ 开发和生产环境差异

### 开发环境特性
- **代码热重载**: 通过卷挂载实现代码实时更新
- **日志可见**: 日志文件挂载到主机便于调试
- **端口暴露**: 直接暴露端口便于测试

### 生产环境建议
- **移除卷挂载**: 避免 `.:/app` 挂载，使用镜像内代码
- **环境变量**: 使用 Docker secrets 管理敏感信息
- **资源限制**: 添加 CPU 和内存限制
- **监控集成**: 集成 Prometheus、Grafana 等监控工具

---

## 🔧 常用操作命令

### 启动服务
```bash
# 启动所有服务
docker compose up -d

# 启动并重新构建
docker compose up -d --build

# 查看服务状态
docker compose ps
```

### 日志查看
```bash
# 查看所有服务日志
docker compose logs -f

# 查看特定服务日志
docker compose logs -f api
docker compose logs -f worker
docker compose logs -f redis
```

### 服务管理
```bash
# 停止服务
docker compose down

# 重启特定服务
docker compose restart api

# 扩展 Worker 服务
docker compose up -d --scale worker=3
```

### 健康检查
```bash
# 检查服务健康状态
docker compose ps --format "table {{.Name}}\t{{.Status}}\t{{.Ports}}"

# 手动执行健康检查
docker compose exec api curl -f http://localhost:8000/api/health
```

---

## 📊 资源使用分析

### 内存使用
- **Redis**: 最大 8GB（配置限制）
- **API 服务**: 约 200-500MB（取决于负载）
- **Worker 服务**: 约 300-800MB（4 进程 × 4 线程）
- **总计**: 约 8.5-9.3GB

### CPU 使用
- **Redis**: 单线程，CPU 使用较低
- **API 服务**: 取决于请求量
- **Worker 服务**: 4 进程并行处理，CPU 密集

### 磁盘使用
- **Redis 数据**: 取决于缓存大小
- **日志文件**: 每服务最大 30MB（10MB × 3 文件）
- **镜像大小**: 约 1-2GB

---

## 🔒 安全考虑

### 网络安全
- Redis 仅绑定本地回环地址
- 使用自定义网络隔离
- 不暴露不必要的端口

### 数据安全
- 敏感配置通过环境变量传递
- Redis 数据持久化到命名卷
- 日志轮转防止磁盘占满

### 运行时安全
- 使用非 root 用户运行（镜像内配置）
- 只读挂载配置文件
- 健康检查确保服务可用性

---

## 📚 相关文档

- [部署项目描述.md](./部署项目描述.md) - 服务部署详细说明
- [配置信息项总结.md](./配置信息项总结.md) - 环境变量配置
- [Docker 官方文档](https://docs.docker.com/compose/) - Docker Compose 参考
- [Redis 配置文档](https://redis.io/docs/management/config/) - Redis 配置参考

---

**这个 Docker Compose 配置为 Kortix 项目提供了完整的容器化部署方案，确保了服务的可靠性、可扩展性和可维护性！** 🚀
