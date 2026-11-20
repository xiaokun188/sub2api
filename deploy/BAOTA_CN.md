# 宝塔面板容器部署

本项目已在仓库根目录提供多阶段构建文件 `Dockerfile`。在宝塔面板的 Docker 镜像构建页面中，按下面的参数构建即可：

| 配置项 | 值 |
| --- | --- |
| 构建上下文 / 项目目录 | 仓库根目录（包含 `backend/`、`frontend/`、`docs/` 的目录） |
| Dockerfile | `Dockerfile` |
| 镜像名称 | `scarecorw`（使用编排配置时） |
| 镜像标签 | `sub2api` |
| 构建平台 | 与服务器架构一致；通常选 `linux/amd64`，ARM 服务器选 `linux/arm64` |

不能把 `deploy/` 单独设为构建上下文：Dockerfile 需要读取仓库根目录中的前后端源代码及法律文档。构建时需要访问 npm 和 Go 模块仓库。

## 创建并运行容器

Sub2API 应用运行在 Docker 容器中，PostgreSQL 和 Redis 使用服务器上已安装并运行的服务。可以在宝塔容器编排中导入 `deploy/docker-compose.baota.yml`，它会基于仓库根目录的 `Dockerfile` 构建应用镜像，只启动 Sub2API 应用容器。

若在宝塔容器页面手动创建应用容器，使用刚构建的镜像并设置以下参数：

- 镜像：`scarecorw:sub2api`（或你自行设置的镜像名称和标签）
- 容器端口：`8080`
- 主机端口：按需选择，例如 `8080`
- 重启策略：`unless-stopped`
- 数据卷：将宿主机持久化目录挂载到容器 `/app/data`
- 容器入口和启动命令：使用镜像默认值，不需要覆盖

至少配置以下环境变量。数据库和 Redis 地址必须是**容器内可访问的服务器地址**：Docker 容器中的 `127.0.0.1` 指向容器自身，不能用它连接服务器上的服务。Linux Docker 通常可通过 `host.docker.internal` 访问宿主机（本配置已添加 `host-gateway` 映射）；若面板或 Docker 版本不支持，也可填写宿主机的 Docker 网桥网关地址（常见为 `172.17.0.1`）或服务器内网 IP。

```dotenv
AUTO_SETUP=true
SERVER_HOST=0.0.0.0
SERVER_PORT=8080
SERVER_MODE=release
TZ=Asia/Shanghai

DATABASE_HOST=host.docker.internal
DATABASE_PORT=5432
DATABASE_USER=sub2api
DATABASE_PASSWORD=替换为强密码
DATABASE_DBNAME=sub2api
DATABASE_SSLMODE=disable

REDIS_HOST=host.docker.internal
REDIS_PORT=6379
REDIS_DB=0
REDIS_PASSWORD=如已设置则填写，否则留空

JWT_SECRET=替换为随机长密钥
ADMIN_EMAIL=管理员邮箱
ADMIN_PASSWORD=替换为强密码
```

确保 PostgreSQL 监听 Docker 网桥可到达的接口，并在 `pg_hba.conf` 中允许 Docker 网段连接；Redis 也需监听容器可达的宿主机接口。按服务器防火墙策略仅允许 Docker 网段访问这两个端口，不要将数据库或 Redis 端口公开到公网。若服务器上为 Redis 配置了密码或 ACL 用户，请同步填写 `REDIS_PASSWORD` 或 `REDIS_USERNAME`。

启动后可通过 `http://服务器地址:主机端口` 访问。健康检查地址为 `/health`。应用数据（包括自动生成的配置）保存在 `/app/data`，升级或重建容器时保留宿主机挂载目录。
