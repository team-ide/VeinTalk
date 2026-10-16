# 镜像与 Compose 详解（面向运维）

> 进阶篇：镜像自构建、compose 各服务说明、离线部署、数据备份与恢复。常规部署只需按 [单机部署](02-单机部署.md) 操作即可。

## 一、Compose 服务说明

`docker-compose.yml` 共 6 个服务，全部为 `network_mode: host`：

| 服务 | 镜像 | 说明 |
|------|------|------|
| mysql | mysql:8.4.11 | 数据库，初始化 `vein_talk` 库与业务账号；字符集 utf8mb4；带健康检查 |
| minio | minio/minio | 对象存储，9000 API / 9001 控制台；带健康检查 |
| redis | redis:7-alpine | 缓存（可选启用，`CACHE_MODE=redis`），AOF 持久化、密码访问 |
| livekit | livekit/livekit-server:v1.13.7 | 音视频服务，依赖 Redis；配置 `livekit.yaml`（端口、node_ip、keys） |
| nginx | nginx:1.27-alpine | 反向代理 / 静态资源 / TLS；依赖 im |
| im | vein-talk-aio:\<版本\>-\<架构\> | 聚合服务（server-all）+ 管理后台 / 公众平台 / 门户静态资源；启动前等待 MySQL、MinIO 就绪 |

- 服务间依赖通过 `depends_on` + `healthcheck` 保证启动顺序
- im 镜像内部：`/data/server/server`（聚合服务二进制）、`/data/server/conf`（配置）、`im-root` / `im-manage` / `im-platform`（静态资源）、`start-im.sh`（等待依赖后启动）

## 二、镜像获取与自构建

### 官方镜像离线包

`vein-talk-aio:<版本>-<架构>.tar.gz`，导入：

```bash
docker load -i vein-talk-aio:1.0.0-test.1-amd64.tar.gz
```

### 自构建（从发布文件）

构建脚本位于发布包内（对应 `deploy/aio` 与 `deploy/docker/im`）：

```bash
# 构建基础镜像（含运行环境，一次构建即可）
./build-docker-base.sh

# 构建应用镜像（amd64 / arm64）
./build-docker.sh          # Dockerfile-amd64 / Dockerfile-arm64
```

应用镜像构建过程：

1. 以基础镜像为底座（alpine 运行环境）
2. 拷贝 `conf`（服务端配置）、`server.sh`（启动/健康检查脚本）
3. 拷贝 `server-all`（聚合服务二进制）
4. 拷贝 `im-root` / `im-manage` / `im-platform`（各 Web 端构建产物）
5. 拷贝 `start-im.sh`（容器启动入口：等待 MySQL、MinIO 后启动服务）

### 导出镜像（离线分发）

```bash
docker save vein-talk-aio:1.0.0-test.1-amd64 | gzip > vein-talk-aio:1.0.0-test.1-amd64.tar.gz
```

## 三、离线部署

无外网环境时，除服务端镜像外还需准备中间件镜像（发布包 `images/` 目录提供导出脚本与导入脚本）：

- `export-*.sh`：在有网环境导出 mysql / minio / redis / livekit / nginx 镜像为 tar
- `import-*.sh`：在离线环境导入
- 导入后按 [单机部署](02-单机部署.md) 步骤操作即可（compose 中的中间件镜像标签需与导入的镜像一致）

## 四、数据备份与恢复

各组件提供导出/导入脚本（发布包 `docker/<组件>/` 目录）：

| 脚本 | 用途 |
|------|------|
| `mysql/export-mysql.sh` / `import-mysql.sh` | MySQL 数据库导出（mysqldump）/ 恢复 |
| `minio/export-minio.sh` / `import-minio.sh` | MinIO 对象数据导出（mc mirror）/ 恢复 |
| `redis/export-redis.sh` / `import-redis.sh` | Redis 数据导出（RDB）/ 恢复 |
| `nginx/export-nginx.sh` / `import-nginx.sh` | Nginx 配置与静态资源备份 / 恢复 |

备份建议：

- 定期备份 MySQL 与 MinIO 数据（`/data/vein-talk/mysql/data`、`/data/vein-talk/minio/data`）
- 备份前建议停止写入或使用脚本的快照能力，保证一致性

## 五、挂载目录与持久化

见 [单机部署 · 目录与挂载说明](02-单机部署.md#四目录与挂载说明)。所有数据均落在 `/data/vein-talk/`，迁移服务器时整目录拷贝 + 导入镜像即可完成迁移。

## 六、配置调整如何生效

| 改动 | 生效方式 |
|------|----------|
| `.env` 参数 | `docker compose up -d` 重建相关容器 |
| `nginx/conf.d/`、`nginx/ssl/` | `docker compose restart nginx`（配置挂载为只读） |
| `livekit/livekit.yaml` | `docker compose restart livekit` |
| `mysql/conf/my.cnf` | `docker compose restart mysql` |
| 门户 `app/`（data.json5）、`privacy.html` | 挂载覆盖镜像内置版本，修改后重启 im 或等待浏览器刷新 |
