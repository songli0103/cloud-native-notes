# Docker Compose

## 概念

Docker Compose 用于定义和运行多容器应用，通过一个 YAML 文件描述所有服务，用一条命令启动/停止所有服务。

**解决的问题**：
- `docker run` 只能管理单个容器
- 多容器应用（如 Web + API + DB）用 `docker run` 管理很繁琐
- Docker Compose 用一个文件定义所有服务，一条命令即可启动全部

## 完整示例（逐行解释）

```yaml
# docker-compose.yml

# services：定义所有服务（容器）
services:
  # 服务名：web
  web:
    # image：从哪个镜像启动容器
    image: nginx:alpine

    # ports：端口映射（宿主机:容器）
    ports:
      - "80:80"

    # volumes：挂载存储
    # 宿主机 ./html 目录映射到容器内 /usr/share/nginx/html
    volumes:
      - ./html:/usr/share/nginx/html:ro

    # depends_on：依赖的服务（启动顺序）
    # web 会在 api 之后启动
    depends_on:
      - api

    # networks：加入的网络
    networks:
      - mynet

  api:
    # build：从当前目录的 Dockerfile 构建镜像
    build: ./api

    # ports：端口映射
    ports:
      - "3000:3000"

    # environment：环境变量
    environment:
      - NODE_ENV=production
      - DB_HOST=db

    # depends_on：依赖 db
    depends_on:
      - db

    # networks：加入的网络
    networks:
      - mynet

  db:
    # 官方镜像，直接启动
    image: mysql:8

    # environment：MySQL 环境变量
    environment:
      MYSQL_ROOT_PASSWORD: 123456
      MYSQL_DATABASE: myapp

    # volumes：持久化数据
    # db-data 是下面定义的顶层 volume
    volumes:
      - db-data:/var/lib/mysql

    # networks：加入的网络
    networks:
      - mynet

# volumes：定义顶层数据卷（供多个服务共享）
volumes:
  db-data:

# networks：定义网络
networks:
  mynet:
    # driver：网络驱动，bridge 是默认
    driver: bridge
```

## 常用命令

```bash
# docker-compose up：创建并启动所有服务
# -d：后台运行
docker-compose up -d

# docker-compose up --build：构建镜像后启动
docker-compose up --build

# docker-compose down：停止并删除所有容器、网络
docker-compose down

# docker-compose down -v：停止并删除（含数据卷）
docker-compose down -v

# docker-compose stop：停止服务（不删除）
docker-compose stop

# docker-compose start：启动已停止的服务
docker-compose start

# docker-compose restart：重启服务
docker-compose restart

# docker-compose ps：查看服务状态
docker-compose ps

# docker-compose logs：查看日志
# -f：实时跟踪
docker-compose logs -f web

# docker-compose exec：在服务容器中执行命令
docker-compose exec web bash

# docker-compose build：构建镜像
docker-compose build

# docker-compose config：验证并显示合并后的配置
docker-compose config

# docker-compose images：查看使用的镜像
docker-compose images
```

## 常用配置项

### services 下支持的字段

| 字段 | 说明 | 示例 |
|------|------|------|
| image | 镜像名称 | image: nginx:alpine |
| build | 构建上下文 | build: ./api |
| ports | 端口映射 | ports: ["80:80"] |
| volumes | 挂载卷 | volumes: ./html:/usr/share/nginx/html |
| environment | 环境变量 | environment: NODE_ENV=prod |
| env_file | 从文件加载环境变量 | env_file: .env |
| depends_on | 依赖服务 | depends_on: [db, redis] |
| networks | 加入的网络 | networks: [mynet] |
| restart | 重启策略 | restart: always |
| command | 覆盖默认命令 | command: npm start |
| entrypoint | 覆盖默认入口点 | entrypoint: ["python"] |
| healthcheck | 健康检查 | healthcheck: {...} |
| user | 运行用户 | user: appuser |

### restart 重启策略

```yaml
restart: no             # 不重启（默认）
restart: always         # 始终重启（容器退出就重启）
restart: on-failure      # 仅在非零退出码时重启
restart: unless-stopped  # 除非手动停止，否则始终重启
```

### depends_on 启动顺序

```yaml
services:
  web:
    depends_on:
      - db        # 等待 db 启动
      - redis     # 等待 redis 启动
```

**注意**：`depends_on` 只保证启动顺序，不保证服务已就绪（可用 `condition` 配合健康检查）。

### 健康检查

```yaml
services:
  db:
    image: mysql:8
    environment:
      MYSQL_ROOT_PASSWORD: 123456
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 10s      # 检查间隔
      timeout: 5s       # 超时时间
      retries: 5        # 重试次数
```

## 多环境配置

### override 文件（开发用）

```yaml
# docker-compose.yml（基础配置）
services:
  web:
    image: myapp
```

```yaml
# docker-compose.override.yml（开发覆盖，自动应用）
# 执行 docker-compose up 时会自动合并
services:
  web:
    build: ./dev
    volumes:
      - ./src:/app
    environment:
      DEBUG: "true"
```

```bash
# 开发环境：自动合并基础配置 + override
docker-compose up
```

### 生产配置分离

```yaml
# docker-compose.prod.yml
services:
  web:
    image: myapp:latest
    replicas: 3
```

```bash
# 生产环境：手动合并基础配置 + 生产配置
docker-compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

## depends_on 进阶用法

```yaml
services:
  web:
    depends_on:
      db:
        # 等待 db 服务健康检查通过
        condition: service_healthy
      redis:
        # 只等待 redis 启动，不等健康检查
        condition: service_started
```

## 命令行参数

```bash
# -f：指定 compose 文件（默认 docker-compose.yml）
docker-compose -f docker-compose.yml -f docker-compose.prod.yml up -d

# -p：指定项目名称（默认目录名）
docker-compose -p myproject up -d

# --scale：扩缩容（已废弃，v2+ 不支持）
docker-compose up -d --scale web=3
```

## 常见问题

**Q: docker-compose 和 docker run 区别？**

| 特性 | docker run | docker-compose |
|------|------------|----------------|
| 管理对象 | 单容器 | 多容器 |
| 配置方式 | 命令行参数 | YAML 文件 |
| 启动顺序 | 手动控制 | 自动控制 |
| 扩缩容 | 手动 | 部分支持 |

**Q: 修改代码后不生效？**

```bash
# 重新构建并启动
docker-compose up --build

# 如果挂载了代码目录（bind mount），不需要重新构建
# 代码变更会直接生效
```

**Q: 容器间无法通信？**

```bash
# 1. 确认在同一网络中
docker-compose exec web ping api

# 2. 检查 networks 配置
# 确保所有服务都在同一个 network 下
```

**Q: 如何查看某个服务日志？**

```bash
docker-compose logs web        # 查看 web 日志
docker-compose logs -f api    # 实时跟踪 api 日志
docker-compose logs --tail=50 db  # 只看最后 50 行
```

**Q: 服务启动了但连接失败？**

```bash
# depends_on 只保证启动顺序，不保证服务就绪
# 可能 db 进程启动了但 MySQL 还没准备好接受连接

# 解决方案：使用 healthcheck + condition: service_healthy
```

**Q: 如何进入容器调试？**

```bash
docker-compose exec web bash
docker-compose exec api sh
```
