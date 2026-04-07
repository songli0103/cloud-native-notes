# 网络与通信

---

## 网络模式

| 模式 | 说明 | 使用场景 |
|------|------|----------|
| **bridge**（默认） | 网桥模式，容器在独立网络 | 大多数场景 |
| **host** | 容器直接使用宿主机网络 | 性能要求高、端口不冲突 |
| **container** | 与另一个容器共享网络命名空间 | 特殊网络需求 |
| **none** | 禁用网络 | 完全隔离 |

---

### bridge（默认）

Docker 启动时创建 docker0 网桥，所有未指定网络的容器连接到这里。

```bash
docker network inspect bridge
```

> **原理**
> ```
> 宿主机
> ├── docker0 网桥 (172.17.0.1/16)
> │   ├── vethxxx (容器 A 的 eth0)
> │   └── vethyyy (容器 B 的 eth0)
> └── 容器 A (172.17.0.x) ←→ 容器 B (172.17.0.x)
> ```
>
> - 每个容器有独立的 eth0，通过 veth pair 连接到 docker0
> - 容器访问外网通过 NAT
> - 默认 bridge 网络不支持容器名解析

---

### host

容器直接使用宿主机网络栈，绕过 Docker 网络隔离。

```bash
docker run --network host nginx
```

> - 性能最好（无 NAT 开销）
> - 端口容易冲突

---

### container

容器共享另一个容器的网络命名空间。

```bash
# 创建基础容器
docker run -d --name app-base nginx

# 新容器共享 app-base 的网络
docker run --network container:app-base nginx
```

> 新容器与 app-base 共享 IP 和端口

---

### none

禁用容器所有网络功能。

```bash
docker run --network none nginx
```

> 容器有 loopback (127.0.0.1)，但没有其他网络接口

---

## 完整示例

```bash
# =================== 网络操作 ===================
docker network ls                                    # 列出所有网络
docker network create --driver bridge mynet          # 创建自定义网络
docker network inspect mynet                         # 查看网络详情

# =================== 容器网络 ===================
docker run -d --name web --network mynet nginx      # 指定网络
docker network connect mynet another-container       # 将运行中的容器加入网络
docker network disconnect mynet another-container    # 断开容器的网络

# =================== 端口映射 ===================
docker run -d -p 8080:80 nginx                     # 宿主机 8080 → 容器 80
docker run -d -p 127.0.0.1:8080:80 nginx           # 仅本地访问
docker run -d -p 8080-8085:80-80 nginx             # 端口范围映射
```

---

## 端口映射

```bash
# 基本格式：-p 宿主机端口:容器端口
docker run -d -p 8080:80 nginx

# 多个端口
docker run -d -p 80:80 -p 443:443 nginx

# 仅本地访问
docker run -d -p 127.0.0.1:8080:80 nginx

# 随机端口
docker run -d -p 80 nginx

# 端口范围
docker run -d -p 8080-8085:80-80 nginx

# UDP 端口
docker run -d -p 53:53 -p 53:53/udp nginx
```

---

## 自定义网络

推荐使用自定义网络而不是默认 bridge。

> **优势**
> - 自定义网络内置 DNS，容器间可以用名字互相访问
> - 更好的网络隔离

```bash
docker network create --driver bridge mynet

# 启动容器加入该网络
docker run -d --name web --network mynet nginx
docker run -d --name api --network mynet python app.py

# 容器间可以用名字互访
docker exec web ping api
```

---

## DNS 解析

| 网络类型 | 容器名解析 |
|----------|-----------|
| 自定义网络 | ✓ 支持（内置 DNS） |
| 默认 bridge | ✗ 不支持（可用 --link，已废弃） |

```bash
# 在自定义网络中，容器间可以直接用名字访问
docker run -d --name db --network mynet mysql
docker run -d --name app --network mynet app

# app 容器内
# ping db → 解析为 db 的 IP
```

---

## 网络命令

```bash
# 列出所有网络
docker network ls

# 查看网络详情
docker network inspect bridge
docker network inspect mynet

# 创建/删除网络
docker network create mynet
docker network rm mynet

# 清理未使用的网络
docker network prune

# 连接/断开容器
docker network connect mynet container-name
docker network disconnect mynet container-name
```

---

## 网络原理

```
容器访问外网：
容器 → veth → docker0 → iptables NAT → 物理网卡 → 外网

宿主机访问容器（端口映射）：
宿主机:8080 → iptables → 容器:80
```

> **关键组件**
> - **veth pair**：虚拟网线，连接容器和主机
> - **docker0**：Linux 网桥，转发数据包
> - **iptables**：配置 NAT 和端口映射规则

---

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

---

**Q: 端口已占用？**

```bash
# 查看端口占用
netstat -tlnp | grep 8080
ss -tlnp | grep 8080

# 解决方案：换一个端口
docker run -d -p 8081:80 nginx
```

---

**Q: 容器间无法互相访问？**

```bash
# 1. 确认在同一网络中
docker inspect container1 | grep Networks -A 5

# 2. 使用自定义网络
docker network create mynet
docker run -d --name c1 --network mynet nginx
docker run -d --name c2 --network mynet nginx

# 3. 在 c1 中 ping c2
docker exec c1 ping c2
```

---

**Q: EXPOSE 和 -p 有什么区别？**

| 指令 | 说明 |
|------|------|
| EXPOSE | Dockerfile 中声明端口，仅文档作用 |
| -p | 实际创建宿主机到容器的端口映射 |
