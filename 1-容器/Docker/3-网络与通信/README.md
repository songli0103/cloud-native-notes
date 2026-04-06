# 网络与通信

## 网络模式详解

Docker 提供 5 种网络模式：

| 模式 | 说明 | 使用场景 |
|------|------|----------|
| bridge（默认） | 网桥模式，容器在独立网络 | 大多数场景 |
| host | 容器直接使用宿主机网络 | 性能要求高、端口不冲突 |
| container | 与另一个容器共享网络命名空间 | 特殊网络需求 |
| none | 禁用网络 | 完全隔离 |
| 自定义网络 | 桥接或 overlay | 容器互通 |

### 1. bridge（默认）

Docker 启动时会创建一个默认的 docker0 网桥，所有未指定网络的容器都会连接到这里。

```bash
# 查看默认网桥信息
docker network inspect bridge
```

**原理**：
```
宿主机
├── docker0 网桥 (172.17.0.1/16)
│   ├── vethxxx (容器 A 的 eth0)
│   └── vethyyy (容器 B 的 eth0)
└── 容器 A (172.17.0.2) ←→ 容器 B (172.17.0.3)
```

- 每个容器有独立的 eth0，通过 veth pair 连接到 docker0
- 容器访问外网通过 NAT（网络地址转换）
- 容器之间可以通过 IP 互访，但默认 bridge 网络不支持容器名解析

### 2. host

容器直接使用宿主机网络栈，绕过 Docker 的网络隔离。

```bash
docker run --network host nginx
```

**特点**：
- 容器和宿主机共享同一网络命名空间
- 性能最好（无 NAT 开销）
- 端口容易冲突（容器端口占用宿主机端口）

### 3. container

容器共享另一个容器的网络命名空间。

```bash
# 创建基础容器
docker run -d --name app-base nginx

# 新容器共享 app-base 的网络
docker run --network container:app-base nginx
```

**特点**：
- 新容器与 app-base 共享 IP 和端口
- 两个容器的网络完全一致

### 4. none

禁用容器所有网络功能。

```bash
docker run --network none nginx
```

**特点**：
- 容器有 loopback (127.0.0.1)，但没有其他网络接口
- 完全隔离，适用于不需要网络的场景

## 完整示例（逐行解释）

```bash
# docker network ls：列出所有网络
docker network ls

# docker network create：创建自定义网络
# --driver bridge：使用桥接驱动（默认）
docker network create --driver bridge mynet

# docker run --network：指定容器加入哪个网络
docker run -d --name web --network mynet nginx

# docker network inspect：查看网络详情
docker network inspect mynet

# -p 8080:80：端口映射
# 宿主机 8080 → 容器 80
docker run -d --name nginx -p 8080:80 nginx

# -p 127.0.0.1:8080:80：仅本地访问
docker run -d -p 127.0.0.1:8080:80 nginx

# -p 8080-8085:80-80：端口范围映射
docker run -d -p 8080-8085:80-80 nginx

# docker network connect：将运行中的容器加入网络
docker network connect mynet another-container

# docker network disconnect：断开容器的网络
docker network disconnect mynet another-container
```

## 端口映射

```bash
# 基本格式：-p 宿主机端口:容器端口
docker run -d -p 8080:80 nginx

# 多个端口
docker run -d -p 80:80 -p 443:443 nginx

# 仅本地访问（宿主机之外无法访问）
docker run -d -p 127.0.0.1:8080:80 nginx

# 随机端口（宿主机随机选一个端口）
docker run -d -p 80 nginx

# 端口范围
docker run -d -p 8080-8085:80-80 nginx

# 同时映射 UDP 端口
docker run -d -p 53:53 -p 53:53/udp nginx
```

## 自定义网络

推荐使用自定义网络而不是默认 bridge，因为：
- 自定义网络内置 DNS，容器间可以用名字互相访问
- 更好的网络隔离
- 可以创建网络插件

```bash
# 创建自定义网络
docker network create --driver bridge mynet

# 启动容器加入该网络
docker run -d --name web --network mynet nginx
docker run -d --name api --network mynet python app.py

# 容器间可以用名字互访（DNS 自动解析）
docker exec web ping api

# 默认 bridge 网络不支持名字互访，需要用 --link（已废弃）
```

## DNS 解析

**自定义网络**：
- 内置 DNS 服务器（127.0.0.11）
- 容器名自动解析为容器 IP
- 可以用容器名、别名访问

**默认 bridge 网络**：
- 不支持容器名解析
- 只能通过 IP 互访
- 可以用 `--link` 实现名字访问（已废弃）

```bash
# 在自定义网络中，容器间可以直接用名字访问
docker run -d --name db --network mynet mysql
docker run -d --name app --network mynet app

# app 容器内：
# ping db → 解析为 db 的 IP
# mysql -h db -u root -p → 连接到 db 容器
```

## 网络命令

```bash
# 列出所有网络
docker network ls

# 查看网络详情
docker network inspect bridge
docker network inspect mynet

# 创建网络
docker network create mynet
docker network create --driver bridge mynet

# 删除网络
docker network rm mynet

# 清理未使用的网络
docker network prune

# 将容器加入网络
docker network connect mynet container-name

# 将容器从网络断开
docker network disconnect mynet container-name
```

## 网络原理

```
容器访问外网：
容器 → veth → docker0 → iptables NAT → 物理网卡 → 外网

宿主机访问容器（端口映射）：
宿主机:8080 → iptables → 容器:80
```

**关键组件**：
- veth pair：虚拟网线，连接容器和主机
- docker0：Linux 网桥，转发数据包
- iptables：配置 NAT 和端口映射规则

## 常见问题

**Q: 容器无法访问外网？**

```bash
# 1. 检查宿主机是否能上网
ping 8.8.8.8

# 2. 检查 iptables 规则
iptables -t nat -L -n

# 3. 检查 DNS 是否生效
docker run --rm busybox ping -c 1 8.8.8.8

# 4. 重启 Docker
systemctl restart docker
```

**Q: 端口已占用？**

```bash
# 查看端口占用
netstat -tlnp | grep 8080
# 或
ss -tlnp | grep 8080

# 解决方案：
# 1. 停止占用该端口的进程
# 2. 或换一个端口映射
docker run -d -p 8081:80 nginx
```

**Q: 容器间无法互相访问？**

```bash
# 1. 确认在同一网络中
docker inspect container1 | grep Networks -A 5

# 2. 使用自定义网络（推荐）
docker network create mynet
docker run -d --name c1 --network mynet nginx
docker run -d --name c2 --network mynet nginx

# 3. 在 c1 中 ping c2
docker exec c1 ping c2
```

**Q: -p 和 EXPOSE 有什么区别？**
- `EXPOSE`：在 Dockerfile 中声明容器端口，仅文档作用
- `-p`：实际创建宿主机到容器的端口映射
