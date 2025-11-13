## 基础服务（Postgres / Redis / MinIO / Docker Registry / NGINX）

使用 Docker Compose 一键启动常用基础服务，采用 host 网络以降低网络开销并提升性能（Linux 服务器环境）。

### 目录结构
```
.
├─ docker-compose.yml
├─ env.template           # 复制为 .env 后可自定义变量（可选）
├─ nginx/
│  ├─ nginx.conf          # 统一入口：对外监听 80
│  ├─ html/               # 静态占位页
│  └─ logs/               # Nginx 日志（挂载）
└─ data/
   ├─ postgres/           # PG 数据
   ├─ redis/              # Redis 数据
   ├─ minio/              # MinIO 数据
   └─ registry/           # Docker Registry 数据
```

### 端口约定（host 网络）
- Postgres: 5432
- Redis: 6379
- MinIO: 9000 (S3 API), 9001 (Console)
- Docker Registry: 5000
- NGINX: 80（对外统一入口）
  - `/` 反向代理到 `127.0.0.1:3001`（前端）
  - `/api/` 反向代理到 `127.0.0.1:8001`（后端）

说明：请在同一台服务器上部署或运行你的前端服务（监听 3001）和后端服务（监听 8001），NGINX 将统一在 80 端口对外提供访问。

### 环境要求
- Linux 服务器，Docker 与 Docker Compose 已安装
- host 网络模式仅在 Linux 上受支持（Windows/Mac 上的 Docker Desktop 不支持）

### 快速开始
1) 拉取仓库
```bash
git clone <your_repo_url> template-chat-docker
cd template-chat-docker
```

2) 可选：复制变量模板并按需修改
```bash
cp env.template .env
# 修改 .env 中的账号密码等
```
如不提供 `.env`，`docker-compose.yml` 会使用内置默认值：
- POSTGRES_USER=appuser
- POSTGRES_PASSWORD=appsecret
- POSTGRES_DB=appdb
- MINIO_ROOT_USER=minioadmin
- MINIO_ROOT_PASSWORD=minioadmin123

3) 启动
```bash
docker compose up -d
```

4) 验证
- NGINX: `curl http://localhost/healthz` 返回 `ok`
- Postgres: `psql -h 127.0.0.1 -U appuser -d appdb -p 5432`
- Redis: `redis-cli -h 127.0.0.1 -p 6379 ping`
- MinIO: `http://<server-ip>:9001`（控制台），`http://<server-ip>:9000`（S3 API）
- Registry: `curl http://127.0.0.1:5000/v2/`

### 常见问题
- 端口冲突：host 网络直接占用宿主机端口，请确保 80/5000/5432/6379/9000/9001 未被占用。
- Windows/Mac：由于 host 网络限制，建议在 Linux 服务器上运行。
- NGINX 未转发：请确认你的前端在 3001、后端在 8001 监听且可访问。


