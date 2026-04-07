# 镜像构建

---

## Dockerfile 学习模板

```dockerfile
# ==========================================
# 第一阶段：构建阶段 (Builder)
# ==========================================
FROM golang:1.22-alpine AS builder

# ARG: 仅构建时有效的变量（如国内镜像加速）
ARG GO_PROXY=https://goproxy.cn,direct

# ENV: 构建环境配置
ENV GOPROXY=$GO_PROXY \
    CGO_ENABLED=0

WORKDIR /build

# 优化技巧：先拷贝依赖描述文件，充分利用镜像层缓存
COPY go.mod go.sum ./
RUN go mod download

# 拷贝源代码并执行静态编译
COPY . .
RUN go build -ldflags="-s -w" -o /app/server .

# ==========================================
# 第二阶段：运行阶段 (Final)
# ==========================================
FROM alpine:3.19

# LABEL: 元数据标注
LABEL maintainer="admin@example.com"
LABEL version="1.0.0"

# ENV: 运行时环境变量（如时区）
ENV TZ=Asia/Shanghai \
    APP_MODE=production

# RUN: 安装基础运行库并清理缓存
RUN apk add --no-cache ca-certificates tzdata && \
    cp /usr/share/zoneinfo/$TZ /etc/localtime && \
    echo $TZ > /etc/timezone

WORKDIR /app

# ADD: 自动解压本地资源包到容器
ADD assets.tar.gz ./

# COPY --from: 从 builder 阶段提取编译产物（丢弃构建垃圾）
COPY --from=builder /app/server /app/server

# VOLUME: 声明持久化目录（挂载点）
VOLUME ["/app/data", "/app/logs"]

# EXPOSE: 声明监听端口（文档作用）
EXPOSE 8080

# USER: 安全加固，使用非 root 用户运行
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

# ENTRYPOINT & CMD: 设置启动程序与默认参数
ENTRYPOINT ["/app/server"]
CMD ["--port=8080"]
```

---

## 指令详解

### 基础类

**FROM**

指定基础镜像（必须是第一条指令）。建议使用轻量级镜像（如 alpine 或 slim 系列）。

```dockerfile
FROM golang:1.22-alpine
FROM python:3.11-slim
```

---

**WORKDIR**

设置工作目录。它会自动创建目录，并使后续命令都在此目录下执行。

> 规矩：始终使用绝对路径

```dockerfile
WORKDIR /app
WORKDIR /app/src
```

---

**ARG vs ENV**

| 指令 | 构建时有效 | 运行时有效 |
|------|-----------|-----------|
| ARG | ✓ | ✗ |
| ENV | ✓ | ✓ |

```dockerfile
# ARG 仅在镜像构建过程中有效
ARG GO_PROXY=https://goproxy.cn

# ENV 在镜像构建和容器运行中都有效
ENV APP_MODE=production
```

---

### 文件操作类

**COPY**

将本地文件复制到镜像中。首选方案。

```dockerfile
COPY . .                     # 复制全部
COPY ./src /app/src         # 指定源和目标
COPY package*.json /app/     # 支持通配符
```

---

**ADD**

增强版复制，支持：
- 从 URL 下载文件
- 自动解压本地压缩包（tar、gzip 等）

```dockerfile
ADD https://example.com/file.tar.gz /app/
ADD ./files.tar.gz /app/    # 自动解压
```

> 建议：COPY 够用时优先使用 COPY，语义更清晰

---

**VOLUME**

声明挂载点，提醒用户该目录需要持久化存储。

```dockerfile
VOLUME ["/app/data", "/app/logs"]
```

---

### 执行类

**RUN**

在构建镜像时执行命令（如安装软件）。每执行一次会创建一个新层。

```dockerfile
# shell 格式
RUN echo "hello"

# exec 格式
RUN ["apt-get", "install", "-y", "nginx"]

# 合并命令减少层数
RUN apt-get update && \
    apt-get install -y nginx && \
    apt-get clean
```

---

**ENTRYPOINT**

配置容器启动时运行的可执行文件。不可被覆盖，通常用于固定主程序。

```dockerfile
ENTRYPOINT ["/app/server"]
```

---

**CMD**

容器启动的默认参数，可以在 docker run 时被覆盖。

```dockerfile
CMD ["--port=8080"]
```

> **经典组合**：`ENTRYPOINT` 定义命令，`CMD` 定义默认参数

---

## Dockerfile 优化

### 规范 1：必须使用多阶段构建

> **原因**：构建环境（编译器、源代码、缓存文件）通常非常庞大，而运行环境只需要一个二进制文件

**做法**：在一个 Dockerfile 中使用多个 `FROM`，通过 `COPY --from=builder` 只提取产物

---

### 规范 2：利用镜像层缓存

> **原理**：Docker 会缓存每一个指令层。如果某一层的文件没变，后续构建会直接使用缓存

**做法**：先拷贝依赖定义文件（如 `package.json`、`go.mod`），安装依赖，最后再拷贝源代码。这样即使改了代码，也不需要重新下载庞大的依赖包。

```dockerfile
# 先拷贝依赖定义文件并安装
COPY package*.json ./
RUN npm ci

# 再拷贝源代码
COPY . .
```

---

### 规范 3：最小化镜像体积

**清理缓存**：使用包管理器安装软件后要清理缓存

```dockerfile
# apk
RUN apk add --no-cache ca-certificates

# apt
RUN apt-get update && \
    apt-get install -y nginx && \
    apt-get clean && rm -rf /var/lib/apt/lists/*
```

**减少层数**：使用 `&&` 连接多个相关的 RUN 命令

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

---

### 规范 4：安全性至上（Non-root）

> **原因**：默认情况下容器以 root 运行，一旦容器被攻破，宿主机面临风险

**做法**：在 Dockerfile 末尾通过 `USER` 指令切换到普通用户

```dockerfile
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser
```

---

### 规范 5：使用 .dockerignore

> **原因**：`COPY . .` 会把本地所有文件发给 Docker 引擎

**做法**：在项目根目录创建 `.dockerignore` 文件

```gitignore
.git
node_modules/
*.log
.env
```

---

## 构建命令

```bash
# 格式：docker build -f <文件名> -t <镜像名>:<标签> <上下文路径>
docker build -f Dockerfile.dev -t my-app:dev .

# 常用选项
-t  打标签
-f  指定 Dockerfile 路径
--build-arg  传递构建参数
--no-cache  跳过缓存
--target     从指定阶段构建（调试用）
```

---

## 常见问题

**Q: ADD 和 COPY 有什么区别？**
- COPY：纯复制，推荐使用
- ADD：支持 URL 下载 + tar 自动解压

---

**Q: CMD 和 ENTRYPOINT 有什么区别？**
- CMD：可以被 `docker run` 参数覆盖
- ENTRYPOINT：参数追加到后面

> 经典组合：`ENTRYPOINT` 定义命令，`CMD` 定义默认参数

---

**Q: 构建失败如何调试？**

```bash
# 查看详细错误
docker build -t myapp:debug . 2>&1

# 从指定阶段构建
docker build --target builder -t myapp:debug .

# 进入中间层容器
docker run -it myapp:debug /bin/sh
```
