# 容器操作

---

## 1. 核心概念

| 概念 | 说明 |
|------|------|
| 镜像 (Image) | 只读模板，包含运行应用程序所需的代码、库、环境等 |
| 容器 (Container) | 镜像的运行实例，可以被启动、停止、删除 |
| 仓库 (Registry) | 存放镜像的地方（如 Docker Hub） |


## 2. 容器生命周期常用命令

### 2.1 启动与停止

```bash
# 启动容器
docker run [OPTIONS] IMAGE [COMMAND]

# 常用参数
-d      # 后台运行（Detach）
-p 80:80  # 端口映射（宿主机端口:容器端口）
--name <name>              # 给容器起个名字
-v /host/path:/container/path  # 数据卷挂载（持久化）

# 示例
docker run -d -p 8080:80 --name my-web nginx

# 停止容器
docker stop <container_id/name>

# 启动已停止的容器
docker start <container_id/name>

# 重启容器
docker restart <container_id/name>
```

### 2.2 查看容器状态

```bash
docker ps                    # 列出运行中的容器
docker ps -a                 # 列出所有容器（含已停止的）
docker inspect <name>        # 查看容器详细信息
docker logs -f <name>        # 查看日志（-f 持续追踪）
```

### 2.3 删除容器

```bash
docker rm <name>             # 删除已停止的容器
docker rm -f <name>          # 强制删除（运行中也可删）
docker container prune       # 清理所有已停止的容器
```

## 3. 进入与交互

```bash
# 进入正在运行的容器
docker exec -it <container_id/name> /bin/bash

# 参数说明
# -i：交互式操作
# -t：分配一个终端
# 无bash用sh
```

## 4. 镜像管理

```bash
docker pull <image_name>:<tag>    # 拉取镜像
docker images                      # 列出本地镜像
docker rmi <image_id/name>         # 删除镜像
docker build -t <new_name> .       # 构建镜像（Dockerfile 所在目录）
```

## 5. 实用操作技巧

### 5.1 数据卷 (Volumes) - 实现持久化

容器被删除后数据会丢失。使用 `-v` 参数将宿主机目录挂载到容器内：

```bash
docker run -v /my/data:/app/data nginx
# 即使容器销毁，数据依然保留在宿主机的 /my/data 中
```

### 5.2 资源限制

防止某个容器占用过多内存或 CPU：

```bash
docker run --memory=”512m” --cpus=”1.0” nginx
```

### 5.3 容器与宿主机拷贝文件

```bash
docker cp <container>:/path/to/file /host/path  # 容器 → 宿主机
docker cp /host/path <container>:/path/to/file  # 宿主机 → 容器
```

## 6. 问题排查”三板斧”

当容器起不来或服务不可用时，按此顺序排查：

| 步骤 | 命令 | 作用 |
|------|------|------|
| 看日志 (Logs) | `docker logs --tail 100 -f <name>` | 解决 90% 的问题 |
| 看状态 (Inspect) | `docker inspect <name>` | 查看挂载、IP、环境变量 |
| 看资源 (Stats) | `docker stats` | 查看是否 OOM 导致重启 |

---

## 常见问题

### Q: 容器自动退出了，如何排查？

按"三板斧"顺序排查：

1. **看日志**：`docker logs --tail 100 <name>`
2. **看状态**：`docker inspect <name>`，关注 ExitCode
3. **看资源**：`docker stats` 是否 OOM

常见退出码：
| 退出码 | 含义 |
|--------|------|
| 0 | 正常退出 |
| 1 | 应用报错退出 |
| 137 | 被 SIGKILL 杀死（内存超限 OOM） |
| 139 | 容器内部段错误 |
| 143 | 收到 SIGTERM 信号正常终止 |

### Q: docker run 和 docker start 区别？

| 命令 | 说明 |
|------|------|
| `docker run` | 创建新容器并启动 |
| `docker start` | 启动已存在的容器（不会创建） |

### Q: 容器内中文显示乱码？

```bash
docker run -e LANG=C.UTF-8 nginx
```

### Q: 如何查看容器 IP？

```bash
docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' <name>
```
