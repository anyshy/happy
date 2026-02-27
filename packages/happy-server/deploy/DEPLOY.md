# Happy 一体化 Docker Compose 部署说明

## 环境要求

- Docker Engine + Docker Compose v2
- 服务器只需开放 **80 端口**（所有流量通过 nginx 统一入口）

## 目录约定

本文档中的 `docker compose` 命令默认在 `packages/happy-server/deploy` 目录执行。

## 架构说明

所有服务通过 nginx 反向代理统一对外，内部服务不暴露公网端口：

```text
公网 80 端口 (nginx)
    ├── /        → happy-webapp（前端静态资源）
    ├── /api/    → happy-server:3005（后端 API）
    └── /s3/     → minio:9000（文件存储）
```

---

## 1. 准备环境变量

```bash
cd packages/happy-server/deploy
cp .env.example .env
```

只需修改两项：

```bash
# .env
HOST_IP=136.111.18.31        # 改成你的服务器公网 IP 或域名
HANDY_MASTER_SECRET=xxx      # 改成一个强随机字符串
```

其余 URL（`HAPPY_SERVER_URL`、`API_PUBLIC_URL`、`S3_PUBLIC_URL`）会自动基于 `HOST_IP` 拼接，无需手动设置。

---

## 2. 启动整套服务

```bash
cd packages/happy-server/deploy
docker compose up -d --build
```

服务包含：

- postgres（内网）
- redis（内网）
- minio（内网）
- happy-server（内网，API）
- happy-webapp（内网，前端）
- nginx（对外，80 端口）

---

## 3. 初始化 MinIO Bucket（首次必做）

```bash
docker run --rm --network container:happy-minio --entrypoint /bin/sh minio/mc -c "mc alias set local http://localhost:9000 minioadmin minioadmin && mc mb -p local/happy || true && mc anonymous set download local/happy/public"
```

如果在 `.env` 改了 MinIO 账号或 Bucket，请同步修改上面命令中的 `minioadmin/minioadmin/happy`。

---

## 4. 执行数据库迁移

```bash
docker exec -w /repo/packages/happy-server happy-server npx prisma migrate deploy
```

---

## 5. 连通性检查

```bash
# API 健康检查
curl http://<服务器IP>/api/
```

浏览器访问：

- Web 页面：`http://<服务器IP>/`

---

## 更新部署

```bash
# 重建并更新整套服务
docker compose up -d --build

# 仅更新应用层（不动数据库/缓存）
docker compose up -d --build --no-deps happy-server happy-webapp nginx

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

# 查看 nginx 日志
docker compose logs -f nginx

# 查看所有日志
docker compose logs -f

# 停止服务
docker compose down

# 停止并清理卷（慎用）
docker compose down -v
```
