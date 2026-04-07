# 容器操作与日志查看

---

## 完整示例

```bash
# =================== 基础操作 ===================
# docker run：创建并启动容器
docker run -d nginx:alpine                  # -d 后台运行
docker run -it ubuntu bash                  # -it 交互式终端

# 端口映射、命名
docker run -d --name mynginx -p 8080:80 nginx

# docker ps：查看容器
docker ps                      # 运行中的容器
docker ps -a                   # 所有容器（含已停止）
docker ps -q                   # 只显示 ID

# =================== 容器生命周期 ===================
# docker exec：在运行中的容器内执行命令
docker exec -it mynginx bash             # 进入容器
docker exec mynginx ls /app             # 执行单个命令

# docker logs：查看日志
docker logs -f --tail 100 mynginx       # 实时跟踪最后100行

# docker start/stop/restart
docker start mynginx
docker stop mynginx
docker restart mynginx

# docker rm：删除容器
docker rm mynginx              # 删除已停止的容器
docker rm -f mynginx           # 强制删除（运行中也可删）
```

---

## 常用命令

### docker run

创建并启动一个新容器。

```bash
# 基础用法
docker run nginx:alpine

# 后台运行
docker run -d nginx:alpine

# 交互式终端（进入容器）
docker run -it ubuntu bash

# 端口映射（宿主机端口:容器端口）
docker run -p 8080:80 nginx

# 指定容器名称
docker run --name myweb nginx

# 容器退出后自动删除
docker run --rm nginx

# 设置环境变量
docker run -e NODE_ENV=production nginx

# 挂载数据卷
docker run -v mydata:/data nginx
```

---

### docker ps

查看容器列表。

```bash
docker ps                              # 运行中的容器
docker ps -a                           # 所有容器
docker ps -q                           # 只显示 ID
docker ps --format "table {{.Names}}\t{{.Status}}"  # 格式化输出
```

---

### docker exec

在运行中的容器内执行命令。

```bash
# 进入容器的 bash
docker exec -it mynginx bash

# 执行单个命令
docker exec mynginx ls /app

# 以特定用户执行
docker exec -it -u root mynginx bash
```

---

### docker logs

查看容器日志。

```bash
docker logs mynginx                    # 查看全部日志
docker logs -f mynginx                # 实时跟踪（Ctrl+C 退出）
docker logs --tail 100 mynginx        # 只看最后 100 行
docker logs --since 10m mynginx       # 最近 10 分钟
docker logs -t mynginx                # 显示时间戳
```

---

### docker start / stop / restart

```bash
docker start mynginx     # 启动已停止的容器
docker stop mynginx      # 停止运行中的容器
docker restart mynginx   # 重启容器
```

---

### docker rm

删除容器。

```bash
docker rm mynginx              # 删除已停止的容器
docker rm -f mynginx           # 强制删除（运行中也可删）
docker container prune         # 删除所有已停止的容器
```

---

## 容器状态

| 状态 | 说明 |
|------|------|
| created | 已创建但未启动 |
| running | 运行中 |
| paused | 暂停（进程挂起） |
| restarting | 重启中 |
| exited | 已退出 |
| dead | 已死亡（异常状态） |

---

## 进入容器

| 方式 | 命令 | 特点 |
|------|------|------|
| **exec**（推荐） | `docker exec -it xxx bash` | exit 不会停止容器 |
| attach | `docker attach xxx` | exit 会停止容器 |

```bash
# 推荐：用 exec 进入容器
docker exec -it mynginx bash

# 不推荐：attach 进入主进程，exit 会 stop 容器
docker attach mynginx
```

---

## 容器内操作

```bash
# 复制文件（宿主机 ↔ 容器）
docker cp host.txt mynginx:/app/      # 宿主机 → 容器
docker cp mynginx:/app/log.txt ./      # 容器 → 宿主机

# 查看容器内进程
docker top mynginx

# 查看资源使用（实时）
docker stats mynginx

# 查看端口映射
docker port mynginx

# 重命名容器
docker rename old_name new_name

# 暂停/恢复容器
docker pause mynginx
docker unpause mynginx
```

---

## 常用参数

| 参数 | 说明 |
|------|------|
| `-d` | 后台运行 |
| `-it` | 交互式终端 |
| `-p 8080:80` | 端口映射（宿主机:容器） |
| `--name xxx` | 指定容器名称 |
| `-v /host:/container` | 挂载目录 |
| `--network xxx` | 指定网络 |
| `--restart always` | 开机自启 |
| `--rm` | 退出后自动删除 |
| `-e KEY=value` | 设置环境变量 |

---

## 常见问题

**Q: 容器自动退出了？**

```bash
# 1. 查看日志
docker logs mynginx

# 2. 查看退出码
docker inspect mynginx | grep -A 5 ExitCode

# 3. 常见退出码含义
退出码 0：正常退出
退出码 1：应用报错退出
退出码 137：被 SIGKILL 杀死（内存超限 OOM）
退出码 139：容器内部段错误
退出码 143：收到 SIGTERM 信号正常终止
```

---

**Q: 如何查看容器 IP？**

```bash
docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' mynginx
```

---

**Q: docker run 和 docker start 区别？**

| 命令 | 说明 |
|------|------|
| `docker run` | 创建新容器并启动 |
| `docker start` | 启动已存在的容器（不会创建） |

---

**Q: 容器内中文显示乱码？**

```bash
docker run -e LANG=C.UTF-8 nginx
```
