# 镜像构建

## Dockerfile 示例

```dockerfile
# ========== 构建阶段 ==========

# FROM：指定基础镜像，所有 Dockerfile 必须以 FROM 开头
# 基础镜像可以来自 Docker Hub 或私有仓库
FROM golang:1.22-alpine AS builder
# AS builder：给这个阶段起个名字，后续 COPY --from=builder 可以引用

# WORKDIR：设置工作目录，相当于 cd
# 如果目录不存在会自动创建
WORKDIR /app

# RUN：在构建镜像时执行的命令，会创建新的镜像层
# 适合安装依赖、编译代码等构建操作
RUN apk add --no-cache git
# apk add --no-cache：Alpine 包管理器，--no-cache 表示不保留下载的缓存文件

# COPY：将文件从构建上下文复制到镜像中
# 格式：COPY 源路径 目标路径
COPY . .
# . 是构建上下文目录（docker build 时的路径）
# /app 是目标路径（在镜像中的位置）

# 构建 Go 应用：CGO_ENABLED=0 静态编译，-ldflags="-s -w" 移除调试信息
RUN CGO_ENABLED=0 go build -ldflags="-s -w" -o myapp

# ========== 运行阶段 ==========

# FROM：开始新的阶段，基础镜像与构建阶段可以不同
FROM alpine:3.19

# RUN：创建非 root 用户（安全最佳实践）
# 容器内默认是 root，以非 root 用户运行可以减少安全风险
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
# addgroup -S：创建系统组（-S 表示 system）
# adduser -S appuser -G appgroup：创建用户并加入组

# COPY --from=xxx：从之前的构建阶段复制文件，而不是从构建上下文复制
# 这样最终镜像只包含运行所需的文件
COPY --from=builder /app/myapp /usr/local/bin/

# USER：设置后续命令以哪个用户执行
# 注意：USER 只影响 CMD/ENTRYPOINT/ RUN，不影响前面的 RUN
USER appuser

# EXPOSE：声明容器内进程监听的端口
# 这只是文档作用，告诉使用者应该映射哪个端口
# 不会实际打开端口或创建端口映射
EXPOSE 8080

# CMD：容器启动时执行的默认命令
# CMD 可以被 docker run 后的参数覆盖
# 格式必须是 exec 格式（JSON 数组），否则 PID 不为 1
CMD ["myapp"]
```

## 指令详解

### FROM

指定基础镜像，每个 Dockerfile 必须以 FROM 开头。

```dockerfile
# 官方镜像
FROM nginx:alpine
FROM python:3.11

# 私有仓库
FROM registry.example.com/myimage:tag

# 多阶段构建时给阶段起名
FROM golang:1.22 AS builder
```

### WORKDIR

设置工作目录，后续指令会在这个目录下执行。

```dockerfile
WORKDIR /app           # 创建并切换到 /app
WORKDIR /app/subdir    # 继续切换到 /app/subdir
WORKDIR ..             # 返回上一级
```

**注意**：
- 如果目录不存在会自动创建
- 等价于 `RUN mkdir -p /app && cd /app`

### COPY

从构建上下文复制文件到镜像中。

```dockerfile
COPY . .                           # 复制全部到当前目录
COPY ./src /app/src               # 指定源和目标
COPY package*.json /app/           # 支持通配符
COPY --chown=user:group ./src /app/src  # 复制并设置权限
```

### ADD

功能类似 COPY，但额外支持：
- 从 URL 下载文件
- 自动解压 tar 文件

```dockerfile
ADD https://example.com/file.tar.gz /app/
ADD ./files.tar.gz /app/           # 自动解压
```

**建议**： COPY 够用时优先使用 COPY，语义更清晰。

### RUN

在构建时执行命令，创建新的镜像层。

```dockerfile
# shell 格式（会调用 /bin/sh）
RUN echo "hello"

# exec 格式（直接执行，不调用 shell）
RUN ["apt-get", "install", "-y", "nginx"]

# 合并多个命令减少层数
RUN apt-get update && \
    apt-get install -y nginx curl && \
    apt-get clean
```

### CMD

容器启动时执行的默认命令，可被 `docker run` 参数覆盖。

```dockerfile
# exec 格式（推荐）：直接执行命令，PID 为 1
CMD ["python", "app.py"]

# shell 格式：调用 /bin/sh 执行
CMD python app.py

# 覆盖 CMD
docker run 镜像 --help  # --help 会覆盖 CMD 中的命令
```

### ENTRYPOINT

容器入口点，`docker run` 的参数会追加到 ENTRYPOINT 后面。

```dockerfile
ENTRYPOINT ["python", "app.py"]
# docker run 镜像 --help  → 执行 python app.py --help
```

**与 CMD 的区别**：
```dockerfile
# CMD 可以被覆盖
CMD ["python", "app.py"]
# docker run 镜像 echo hi  → 执行 echo hi

# ENTRYPOINT 不可被覆盖（除非 --entrypoint）
ENTRYPOINT ["python", "app.py"]
# docker run 镜像 echo hi  → 执行 python app.py echo hi
```

### ENV

设置环境变量，在镜像和容器中都生效。

```dockerfile
ENV NODE_ENV=production
ENV PORT=8080

# 多个变量写在一行
ENV PORT=8080 VERSION=1.0
```

### ARG

构建参数，仅在构建时使用，不会出现在最终镜像中。

```dockerfile
ARG VERSION=1.0
RUN echo "Building version $VERSION"

# docker build --build-arg VERSION=2.0
```

### USER

设置后续指令和容器运行时的用户。

```dockerfile
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser
```

### EXPOSE

声明容器内进程监听的端口，仅起文档作用。

```dockerfile
EXPOSE 8080
EXPOSE 80 443
```

**注意**：
- 不会实际创建端口映射
- 需要配合 `docker run -p` 才能从宿主机访问

### VOLUME

声明数据持久化目录，挂载到 Docker 管理的主机目录。

```dockerfile
VOLUME /data
VOLUME ["/data", "/logs"]
```

### HEALTHCHECK

容器健康检查，告诉 Docker 如何判断容器是否正常运行。

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --retries=3 CMD curl -f http://localhost:8080/health || exit 1
```

## Dockerfile 优化

### 1. 减少层数

多个 RUN 指令会创建多层镜像，合并可以减小体积。

```dockerfile
# 不好：3 层
RUN apt-get update
RUN apt-get install -y nginx
RUN apt-get clean

# 好：1 层
RUN apt-get update && \
    apt-get install -y nginx && \
    apt-get clean
```

### 2. 利用构建缓存

Docker 按顺序执行指令，利用缓存可以加速重复构建。

```dockerfile
# 先复制依赖文件并安装，再复制代码
# 修改代码时，依赖安装层会被缓存
COPY package*.json ./
RUN npm ci --omit=dev
COPY . .
```

### 3. 使用轻量基础镜像

| 镜像 | 大小 |
|------|------|
| ubuntu:22.04 | ~77MB |
| alpine:3.19 | ~7MB |
| python:3.11 | ~1GB |
| python:3.11-alpine | ~50MB |

### 4. 非 root 用户

```dockerfile
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser
```

### 5. .dockerignore

排除不需要发送到 Docker daemon 的文件。

```gitignore
.git
node_modules/
*.log
.env
```

## 构建命令

```bash
# 基本构建
docker build -t myapp:latest .

# 多标签
docker build -t myapp:1.0 -t myapp:latest .

# 传递构建参数
docker build --build-arg VERSION=2.0 .

# 跳过缓存
docker build --no-cache .

# 指定 Dockerfile
docker build -f ./Dockerfile.prod .

# 从指定阶段构建（调试用）
docker build --target builder -t myapp:debug .

# 查看镜像层
docker history myapp:latest
```

## 常见问题

**Q: EXPOSE 和 -p 有什么区别？**
- EXPOSE：仅文档作用，不实际映射
- -p：创建实际端口映射

**Q: CMD vs ENTRYPOINT？**
- CMD：可以被覆盖
- ENTRYPOINT：参数追加到后面

**Q: ADD vs COPY？**
- COPY：纯复制，推荐使用
- ADD：支持 URL 和 tar 解压，COPY 够用时用 COPY

**Q: 构建失败怎么调试？**
```bash
# 查看错误
docker build -t myapp:debug . 2>&1

# 进入中间阶段容器
docker build --target builder -t myapp:debug .
docker run -it myapp:debug /bin/sh
```
