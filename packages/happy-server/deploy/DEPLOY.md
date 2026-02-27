# Happy Server 部署说明

## 环境要求

- Docker & Docker Compose
- 服务器开放端口：3000（API）、9000/9001（MinIO）

## 服务说明

| 服务 | 镜像 | 端口 | 说明 |
|------|------|------|------|
| postgres | postgres:16 | 5432 | 主数据库 |
| redis | redis:7 | 6379 | 缓存 / 事件总线 |
| minio | minio/minio | 9000/9001 | 对象存储（图片等文件） |
| happy-server | happy-server:latest | 3000 | 应用服务 |

---

## 首次部署

### 1. 构建镜像

在项目根目录执行：

```bash
docker build -t happy-server:latest .
```

### 2. 启动所有服务

```bash
cd deploy
docker compose up -d
```

### 3. 初始化 MinIO Bucket

首次启动后需要手动创建 bucket：

```bash
docker exec happy-minio mc alias set local http://localhost:9000 minioadmin minioadmin
docker exec happy-minio mc mb local/happy
docker exec happy-minio mc anonymous set public local/happy/public
```

### 4. 重启 happy-server

bucket 创建完成后重启应用，确保 `loadFiles()` 检查通过：

```bash
docker compose restart happy-server
```

### 5. 执行数据库迁移

```bash
docker exec happy-server yarn migrate
```

---

## 更新部署

```bash
# 1. 重新构建镜像
docker build -t happy-server:latest .

# 2. 重启应用（数据库和 MinIO 不需要重启）
docker compose up -d --no-deps happy-server
```

---

## 环境变量说明

| 变量 | 默认值 | 说明 |
|------|--------|------|
| NODE_ENV | production | 运行环境 |
| PORT | 3005 | 内部监听端口 |
| SEED | yansyhyn | 随机种子，用于生成唯一 key |
| DATABASE_URL | - | PostgreSQL 连接串 |
| REDIS_URL | - | Redis 连接串 |
| S3_HOST | minio | MinIO 主机名（容器内用服务名） |
| S3_PORT | 9000 | MinIO 端口 |
| S3_USE_SSL | false | 是否启用 SSL |
| S3_ACCESS_KEY | minioadmin | MinIO 访问密钥 |
| S3_SECRET_KEY | minioadmin | MinIO 密钥 |
| S3_BUCKET | happy | Bucket 名称 |
| S3_PUBLIC_URL | http://localhost:9000/happy | 文件公开访问地址 |

> 生产环境建议修改 `SEED`、`S3_ACCESS_KEY`、`S3_SECRET_KEY`、`POSTGRES_PASSWORD` 为强密码。

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

浏览器访问 `http://<服务器IP>:9001`，用以下账号登录：

- 用户名：`minioadmin`
- 密码：`minioadmin`
