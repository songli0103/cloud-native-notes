# 存储卷挂载实操

---

## 三种存储方式

| 类型 | 说明 | 存储位置 | 使用场景 |
|------|------|----------|----------|
| **Volume** | Docker 管理的数据卷 | `/var/lib/docker/volumes/` | 生产环境持久化 |
| **Bind Mount** | 挂载宿主机目录 | 宿主机任意位置 | 开发环境代码挂载 |
| **tmpfs** | 内存文件系统 | 内存中 | 临时敏感数据 |

---

## 完整示例

```bash
# =================== Volume ===================
docker volume create mydata                    # 创建数据卷
docker volume ls                              # 列出数据卷
docker volume inspect mydata                  # 查看详情（Mountpoint）
docker run -d -v mydata:/app/data nginx      # 挂载数据卷
docker volume rm mydata                       # 删除数据卷
docker volume prune                           # 删除未使用的数据卷

# =================== Bind Mount ===================
docker run -d -v /host/path:/container/path nginx        # 挂载宿主机目录
docker run -d -v $(pwd)/html:/usr/share/nginx/html nginx # $(pwd) 当前目录
docker run -d -v /host/config:/etc/nginx/conf:ro nginx  # :ro 只读

# =================== tmpfs ===================
docker run -d --tmpfs /app/tmpfs nginx                    # 挂载到内存
docker run -d --tmpfs /app/tmpfs:rw,noexec,size=100m nginx # 带选项
```

---

## Volume（数据卷）

Docker 管理的存储，存放在 `/var/lib/docker/volumes/` 下。

```bash
# 创建数据卷
docker volume create mydata

# 查看数据卷列表
docker volume ls

# 查看数据卷详情
docker volume inspect mydata
# {
#     "Mountpoint": "/var/lib/docker/volumes/mydata/_data",
#     "Name": "mydata"
# }

# 启动容器并挂载数据卷
docker run -d -v mydata:/var/lib/mysql mysql:8

# 启动容器时自动创建数据卷
docker run -d -v mysql-data:/var/lib/mysql mysql:8

# 删除数据卷
docker volume rm mydata
docker volume prune      # 删除未使用的所有数据卷
```

> **特点**
> - 由 Docker 管理，备份迁移方便
> - 多个容器可以共享同一个数据卷
> - 删除容器后数据卷仍然保留

---

## Bind Mount（绑定挂载）

直接挂载宿主机的目录到容器。

```bash
# 基本用法：-v 宿主机绝对路径:容器内路径
docker run -d -v /home/user/data:/app/data nginx

# 使用 $(pwd) 挂载当前目录
docker run -d -p 8080:80 -v $(pwd)/html:/usr/share/nginx/html nginx

# 只读挂载（容器内无法修改）
docker run -d -v $(pwd)/config:/etc/nginx/conf:ro nginx

# 挂载单个文件
docker run -d -v $(pwd)/nginx.conf:/etc/nginx/nginx.conf:ro nginx
```

> **特点**
> - 容器和宿主机可以同时访问同一份数据
> - 适合开发时代码热更新

---

## tmpfs（内存挂载）

将数据存储在内存中，容器重启后数据丢失。

```bash
# 挂载到内存
docker run -d --tmpfs /app/tmpfs nginx

# 指定挂载选项
# rw：可读写（默认）
# ro：只读
# noexec：不可执行
# nosuid：不允许设置特权
# size：最大大小（如 100m）
docker run -d --tmpfs /app/tmpfs:rw,noexec,nosuid,size=100m nginx
```

> **特点**
> - 数据存储在内存中，读写速度极快
> - 容器重启后数据丢失
> - 适合存储临时敏感数据（如密码、密钥）

---

## 实战示例

### MySQL 数据持久化

```bash
docker run -d \
  --name mysql \
  -v mysql-data:/var/lib/mysql \
  -e MYSQL_ROOT_PASSWORD=123456 \
  mysql:8
```

> - `-v mysql-data:/var/lib/mysql`：数据持久化
> - 生产环境请使用强密码

---

### Nginx 配置+静态文件

```bash
docker run -d \
  --name nginx \
  -p 80:80 \
  -v /host/nginx.conf:/etc/nginx/nginx.conf:ro \
  -v /host/html:/usr/share/nginx/html:ro \
  nginx
```

> - `:ro`：只读挂载，容器内无法修改配置

---

### Redis 持久化配置

```bash
docker run -d \
  --name redis \
  -v redis-data:/data \
  -v $(pwd)/redis.conf:/etc/redis/redis.conf:ro \
  redis redis-server /etc/redis/redis.conf
```

---

## 挂载选项

```bash
# :ro 只读
docker run -d -v /host:/container:ro nginx

# :rw 读写（默认）
docker run -d -v /host:/container:rw nginx
```

---

## 权限问题

容器内进程通常以 root 运行，可能与宿主机文件权限冲突。

```bash
# 问题：容器内用户无法写入宿主机目录
docker run -d -v /host/data:/app/data nginx

# 解决方案 1：容器内使用 root
docker run -d -v /host/data:/app/data --user root nginx

# 解决方案 2：修改宿主机目录权限
sudo chown -R 1000:1000 /host/data

# 解决方案 3：创建数据卷时指定权限
docker volume create --opt type=none --opt o=uid:1000,g=1000 mydata
```

---

## 三种方式对比

| 特性 | Volume | Bind Mount | tmpfs |
|------|--------|------------|-------|
| 存储位置 | Docker 管理 | 宿主机任意位置 | 内存 |
| 持久化 | ✓ | ✓ | ✗ |
| 多容器共享 | ✓ | ✓ | ✗ |
| 性能 | 高 | 高 | 最高 |
| 代码/配置挂载 | 一般 | ✓ | ✗ |
| 安全性 | 高 | 中 | 高 |

---

## 常见问题

**Q: 挂载后容器内目录是空的？**

```bash
# 1. 确认宿主机目录存在且有内容
ls -la /host/path

# 2. 检查路径是否正确（必须用绝对路径）
# 错误：-v html:/app/html
# 正确：-v $(pwd)/html:/app/html
```

---

**Q: 容器写入数据后，宿主机看不到？**

```bash
# Bind Mount：数据直接在宿主机目录，应该可见
ls -la /host/path

# Volume：数据在 Docker 管理目录
docker volume inspect mydata
# 查看 Mountpoint 路径
```

---

**Q: 如何查看 volume 在宿主机的位置？**

```bash
docker volume inspect mydata
# 返回结果中 Mountpoint 即为宿主机存储路径
```

---

**Q: 删除容器后数据会丢失吗？**

| 类型 | 数据丢失？ |
|------|-----------|
| Bind Mount | ✗ 不会（挂载的是宿主机目录） |
| Volume | ✗ 不会（删除容器后卷仍然存在） |
| tmpfs | ✓ 会（数据在内存中） |
