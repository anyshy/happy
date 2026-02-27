# Happy Server 部署说明

## 环境要求

- Docker Engine + Docker Compose v2
- 服务器开放端口：`3000`（API）、`9000/9001`（MinIO）

## 目录约定

本文档中的 `docker compose` 命令默认在 `packages/happy-server/deploy` 目录执行。

## 服务说明

| 服务 | 镜像 | 端口 | 说明 |
|------|------|------|------|
| postgres | postgres:16 | 5432 | 主数据库 |
| redis | redis:7 | 6379 | 缓存 / 事件总线 |
| minio | minio/minio | 9000/9001 | 对象存储（图片等文件） |
| happy-server | happy-server:latest | 3000 -> 3005 | 应用服务（宿主机 3000 映射容器 3005） |

---

## 首次部署

### 1. 构建镜像（项目根目录）

```bash
docker build -f Dockerfile.server -t happy-server:latest .
```

### 2. 启动所有服务

```bash
cd packages/happy-server/deploy
docker compose up -d
```

### 3. 初始化 MinIO Bucket（首次必做）

```bash
docker run --rm --network container:happy-minio --entrypoint /bin/sh minio/mc -c "mc alias set local http://localhost:9000 minioadmin minioadmin && mc mb -p local/happy || true && mc anonymous set download local/happy/public"
```

### 4. 执行数据库迁移

```bash
docker exec -w /repo/packages/happy-server happy-server npx prisma migrate deploy
```

### 5. 重启应用

```bash
docker compose restart happy-server
```

---

## 更新部署

```bash
# 1. 在项目根目录重新构建镜像
docker build -f Dockerfile.server -t happy-server:latest .

# 2. 回到部署目录，仅重建应用容器
cd packages/happy-server/deploy
docker compose up -d --no-deps happy-server

# 3. 若包含数据库变更，执行迁移
docker exec -w /repo/packages/happy-server happy-server npx prisma migrate deploy
```

---

## 环境变量说明

| 变量 | 默认值 | 说明 |
|------|--------|------|
| NODE_ENV | production | 运行环境 |
| PORT | 3005 | 容器内监听端口（由 compose 映射到宿主机 3000） |
| HANDY_MASTER_SECRET | 示例值（必须修改） | 认证/加密主密钥 |
| DATABASE_URL | postgresql://postgres:postgres@postgres:5432/happy-server | PostgreSQL 连接串 |
| REDIS_URL | redis://redis:6379 | Redis 连接串 |
| S3_HOST | minio | MinIO 主机名（容器内用服务名） |
| S3_PORT | 9000 | MinIO 端口 |
| S3_USE_SSL | false | 是否启用 SSL |
| S3_ACCESS_KEY | minioadmin | MinIO 访问密钥 |
| S3_SECRET_KEY | minioadmin | MinIO 密钥 |
| S3_BUCKET | happy | Bucket 名称 |
| S3_PUBLIC_URL | http://localhost:9000/happy | 文件公开访问地址（生产环境应改为公网可访问地址） |
| SEED | 示例值 | 兼容旧配置，当前版本通常不使用 |

> 生产环境请至少修改：`POSTGRES_PASSWORD`、`HANDY_MASTER_SECRET`、`S3_ACCESS_KEY`、`S3_SECRET_KEY`。

---

## 常用运维命令

```bash
# 查看所有服务状态
docker compose ps

# 查看应用日志
docker compose logs -f happy-server

# 查看所有服务日志
docker compose logs -f

# 停止所有服务
docker compose down

# 停止并清除数据（慎用）
docker compose down -v
```

---

## MinIO 控制台

浏览器访问 `http://<服务器IP>:9001`，默认账号：

- 用户名：`minioadmin`
- 密码：`minioadmin`
