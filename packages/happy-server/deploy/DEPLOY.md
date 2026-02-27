# Happy 一体化 Docker Compose 部署说明

## 环境要求

- Docker Engine + Docker Compose v2
- 服务器开放端口：`3000`（API）、`8080`（Web）、`9000/9001`（MinIO）

## 目录约定

本文档中的 `docker compose` 命令默认在 `packages/happy-server/deploy` 目录执行。

## 重要说明

- `http://<服务器IP>:3000/` 是后端 API 根路径，返回 `Welcome to Happy Server!` 属于正常现象。
- 浏览器访问 API 时出现 `GET /favicon.ico 404` 是正常行为，不影响业务请求。
- 完整 Web 界面请访问 `http://<服务器IP>:8080/`（由 `happy-webapp` 服务提供）。

---

## 1. 准备环境变量

先复制示例文件：

```bash
cd packages/happy-server/deploy
cp .env.example .env
```

至少修改 `.env` 里的：

- `HANDY_MASTER_SECRET`
- `HAPPY_SERVER_URL`（浏览器访问 Web 时前端请求的 API 地址）
- `API_PUBLIC_URL`、`S3_PUBLIC_URL`（建议改成公网可访问地址）

---

## 2. 启动整套服务（后端 + Web）

```bash
cd packages/happy-server/deploy
docker compose up -d --build
```

服务包含：

- postgres
- redis
- minio
- happy-server（API，宿主机默认 `3000`）
- happy-webapp（Web 页面，宿主机默认 `8080`）

### 可选：手动打包镜像（保留原有习惯）

如果你希望先显式打包，再启动 compose，可在仓库根目录执行：

```bash
docker build -f Dockerfile.server -t happy-server:latest .
docker build -f Dockerfile.webapp -t happy-webapp:latest --build-arg HAPPY_SERVER_URL=http://<服务器IP>:3000 .
```

然后回到部署目录启动（可不加 `--build`）：

```bash
cd packages/happy-server/deploy
docker compose up -d
```

---

## 3. 初始化 MinIO Bucket（首次必做）

```bash
docker run --rm --network container:happy-minio --entrypoint /bin/sh minio/mc -c "mc alias set local http://localhost:9000 minioadmin minioadmin && mc mb -p local/happy || true && mc anonymous set download local/happy/public"
```

如果你在 `.env` 改了 MinIO 账号或 Bucket，请把上面命令中的 `minioadmin/minioadmin/happy` 同步改掉。

---

## 4. 执行数据库迁移

```bash
docker exec -w /repo/packages/happy-server happy-server npx prisma migrate deploy
```

---

## 5. 连通性检查

```bash
# API 健康检查
curl http://<服务器IP>:3000/
```

浏览器访问：

- Web 页面：`http://<服务器IP>:8080/`
- MinIO 控制台：`http://<服务器IP>:9001/`

---

## 更新部署

```bash
# 重建并更新整套服务
docker compose up -d --build

# 仅更新应用层（后端+web，不动数据库/缓存）
docker compose up -d --build --no-deps happy-server happy-webapp

# 若包含数据库变更，执行迁移
docker exec -w /repo/packages/happy-server happy-server npx prisma migrate deploy
```

---

## 常用运维命令

```bash
# 查看服务状态
docker compose ps

# 查看后端日志
docker compose logs -f happy-server

# 查看 Web 日志
docker compose logs -f happy-webapp

# 查看所有日志
docker compose logs -f

# 停止服务
docker compose down

# 停止并清理卷（慎用）
docker compose down -v
```
