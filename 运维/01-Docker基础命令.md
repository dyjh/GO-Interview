# Docker 基础命令

## Docker 是什么

Docker 是容器化平台，用于把应用和依赖打包成镜像，并以容器形式运行。容器共享宿主机内核，比虚拟机更轻量，适合开发、测试、部署和微服务环境隔离。

核心概念：

- Image：镜像，只读模板。
- Container：容器，镜像运行后的实例。
- Dockerfile：构建镜像的描述文件。
- Registry：镜像仓库，如 Docker Hub、Harbor。
- Volume：数据卷，用于持久化数据。
- Network：容器网络。

## 查看信息

```bash
docker --version
docker info
docker version
```

## 镜像命令

```bash
docker pull nginx:latest
docker images
docker rmi nginx:latest
docker build -t myapp:1.0 .
docker tag myapp:1.0 registry.example.com/myapp:1.0
docker push registry.example.com/myapp:1.0
```

注意：

- 镜像 tag 不写时默认 `latest`，但生产环境不建议依赖 latest。
- 删除镜像前要确认没有容器引用。

## 容器命令

```bash
docker run -d --name web -p 8080:80 nginx
docker ps
docker ps -a
docker start web
docker stop web
docker restart web
docker rm web
```

常用参数：

- `-d`：后台运行。
- `-it`：交互式终端。
- `--name`：容器名称。
- `-p`：端口映射。
- `-v`：挂载数据卷或目录。
- `-e`：设置环境变量。
- `--restart`：重启策略。

## 进入容器和查看日志

```bash
docker exec -it web /bin/sh
docker logs web
docker logs -f --tail=100 web
```

注意：很多精简镜像没有 bash，只有 sh。

## 文件复制

```bash
docker cp web:/etc/nginx/nginx.conf ./nginx.conf
docker cp ./app.conf web:/etc/nginx/conf.d/app.conf
```

## 数据卷

```bash
docker volume create mysql-data
docker volume ls
docker run -d --name mysql -v mysql-data:/var/lib/mysql mysql:8
```

数据卷用于持久化数据库、上传文件等。不要把重要数据只放在容器可写层里。

## 网络

```bash
docker network ls
docker network create app-net
docker run -d --name web --network app-net nginx
```

同一个自定义 bridge 网络内，容器可通过容器名互相访问。

## 清理命令

谨慎使用清理命令：

```bash
docker container prune
docker image prune
docker system df
```

生产机器上执行 prune 前要确认不会删除仍需保留的资源。

## Dockerfile 基础

示例：

```dockerfile
FROM golang:1.22 AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o server ./cmd/server

FROM alpine:3.19
WORKDIR /app
COPY --from=builder /app/server /app/server
EXPOSE 8080
ENTRYPOINT ["/app/server"]
```

优化点：

- 使用多阶段构建减小镜像。
- 优先复制 `go.mod` 和 `go.sum` 利用构建缓存。
- 不把密钥写进镜像。
- 使用 `.dockerignore` 排除无用文件。
- 生产镜像尽量精简。

## Docker Compose

适合本地开发编排多个服务：

```yaml
services:
  app:
    build: .
    ports:
      - "8080:8080"
    depends_on:
      - redis
  redis:
    image: redis:7
```

命令：

```bash
docker compose up -d
docker compose ps
docker compose logs -f
docker compose down
```

## 容器运行常见坑

### 容器退出和重启策略

容器主进程退出，容器就会退出。后台服务不要只靠 `tail -f` 这类方式“保活”，应让真正的应用进程作为主进程运行。

常见重启策略：

```bash
docker run --restart=always ...
docker run --restart=unless-stopped ...
```

### 资源限制和 OOM

生产环境要关注 CPU、内存和文件句柄限制。容器被 OOM Kill 时，应用日志可能来不及完整写出，需要结合 `docker inspect`、宿主机日志和监控排查。

### 数据持久化

数据库、上传文件、状态数据应放到 volume 或外部存储，不要只放容器可写层。容器删除后，可写层数据会丢失。

## 面试追问

### 容器和虚拟机区别？

容器共享宿主机内核，启动快、资源占用低；虚拟机包含完整操作系统，隔离更强但更重。

### CMD 和 ENTRYPOINT 区别？

`ENTRYPOINT` 定义容器主命令，`CMD` 提供默认参数或默认命令。运行容器时传入参数会覆盖 CMD。

### 镜像为什么要分层？

镜像层可复用和缓存，提高构建、分发效率。Dockerfile 每条指令通常会生成一层，应合理组织顺序。

## 文章导航
上一篇：[02-大数据分页与查询优化](../系统设计/02-大数据分页与查询优化.md)
下一篇：[02-Nginx负载均衡](02-Nginx负载均衡.md)
