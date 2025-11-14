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
  - `/v2/`（HTTP 80）自动 301 跳转到 HTTPS

### 使用自有域名与 HTTPS 推送镜像（template-chat.xyz）
本模板已内置私有 Registry（`registry:2`，监听 5000）和 NGINX 反代。为域名 `template-chat.xyz` 增加了 443 端口的 HTTPS 入口：

1) 放置证书（必须）
   - 将你的证书文件放入：`nginx/certs/template-chat.xyz/`
     - `fullchain.pem`
     - `privkey.pem`
   - 容器内将通过 `/etc/nginx/certs/template-chat.xyz/{fullchain.pem,privkey.pem}` 引用

2) 启动 / 重载
```bash
docker compose up -d
# 已在运行：
docker compose restart nginx
```

3) 验证可用性
```bash
curl -I https://template-chat.xyz/v2/
# 期待：HTTP/2 200 或 401（若启用 Basic Auth）
```

4) 客户端登录并推送镜像
```bash
# 登录（若未启用 Basic Auth，仅 TLS，login 也会成功但可省略）
docker login template-chat.xyz

# 构建并推送
docker build -t template-chat.xyz/your-namespace/your-app:1.0.0 .
docker push template-chat.xyz/your-namespace/your-app:1.0.0

# 查看仓库与标签
curl -s https://template-chat.xyz/v2/_catalog | jq .
curl -s https://template-chat.xyz/v2/your-namespace/your-app/tags/list | jq .
```

5)（可选）开启 Basic Auth 认证
- 生成 `htpasswd` 文件（在宿主机任意位置执行）：
```bash
# 如果没有 htpasswd，可安装 apache2-utils（Debian/Ubuntu）或 httpd-tools（CentOS/RHEL）
htpasswd -Bc nginx/registry.htpasswd <username>
```
- 编辑 `nginx/nginx.conf` 中 HTTPS server 下的 `/v2/`，取消两行注释：
  - `auth_basic "Private Docker Registry";`
  - `auth_basic_user_file /etc/nginx/registry.htpasswd;`
- 并在 `docker-compose.yml` 的 nginx 服务中增加挂载：
```yaml
    volumes:
      - ./nginx/registry.htpasswd:/etc/nginx/registry.htpasswd:ro
```
- 重启 NGINX：`docker compose restart nginx`
- 再次登录：`docker login template-chat.xyz`

6) 删除镜像与垃圾回收
- 本模板已启用 `REGISTRY_STORAGE_DELETE_ENABLED=true`
- 删除 manifest 后可按需停库执行 GC 以回收空间（请参考官方文档）

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


