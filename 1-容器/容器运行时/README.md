# 容器运行时

## 概念分层

容器运行时是 Docker/K8s 等容器平台的核心，负责创建和管理容器。分层结构如下：

```
应用层
   ↓
Docker Client (docker CLI)
   ↓
Docker Engine (dockerd)
   ↓
Container Runtime (containerd)
   ↓
runc (OCI 规范实现)
   ↓
Linux 内核 (namespaces + cgroups)
```

**各层说明**：

| 层级 | 组件 | 职责 |
|------|------|------|
| CLI | docker CLI | 接收用户命令 |
| Engine | dockerd | 编排、网络、存储、镜像管理 |
| Runtime | containerd | 容器生命周期管理（创建/停止/删除） |
| Low-level | runc | 真正创建容器（fork 进程、设置 namespace） |
| Kernel | namespaces/cgroups | 资源隔离（进程、网络、文件系统等） |

## 常见容器运行时

| 运行时 | 说明 | 使用场景 |
|--------|------|----------|
| **runc** | OCI 规范的参考实现，containerd 底层调用它 | 底层工具，不直接使用 |
| **containerd** | 容器生命周期管理，K8s 默认运行时 | Docker、K8s |
| **cri-o** | K8s 专用实现，只支持 CRI 接口 | K8s（替代 containerd） |
| **gVisor** | Google 沙箱运行时，内核隔离更强 | 多租户、安全增强 |

## runc（底层运行时）

runc 是 OCI 运行时规范的参考实现，直接与 Linux 内核交互。

```bash
# runc 是底层工具，通常不直接使用
# containerd 会在创建容器时调用 runc

# 查看系统中的容器（由 runc 管理的）
runc list

# runc 需要完整的 rootfs 才能操作容器
# 格式：runc <command> <container-id>
runc start <container-id>     # 启动容器
runc kill <container-id>      # 杀死容器
runc delete <container-id>    # 删除容器
runc state <container-id>    # 查看容器状态
```

## containerd（容器运行时）

containerd 专注于容器生命周期管理，是 Docker 和 K8s 的核心组件。

### 架构

```
dockerd
   ├── 镜像管理（registry、layer）
   ├── 网络管理（libnetwork）
   └── containerd（IPC）
            ↓
        containerd-shim（每个容器一个）
            ↓
         runc
            ↓
      容器进程
```

**containerd-shim**：
- 位于 containerd 和 runc 之间
- 允许 containerd 退出而不影响容器运行
- 收集容器退出码

### 常用命令（ctr）

```bash
#ctr：containerd 的 CLI 工具

# 镜像操作
ctr images pull docker.io/library/nginx:alpine    # 拉取镜像
ctr images ls                                      # 列出镜像
ctr images rm nginx:alpine                        # 删除镜像

# 容器操作
ctr containers ls                                  # 列出容器
ctr containers create docker.io/library/nginx:alpine nginx1  # 创建容器
ctr containers start nginx1                        # 启动容器
ctr containers stop nginx1                         # 停止容器
ctr containers rm nginx1                          # 删除容器
```

## nerdctl（containerd CLI）

nerdctl 是 containerd 的命令行工具，兼容 Docker CLI，用法与 docker 完全一致。

```bash
# 安装（Ubuntu）
apt install nerdctl

# 常用命令（与 docker 命令相同）
nerdctl run -d nginx:alpine              # 运行容器
nerdctl ps                                # 查看容器
nerdctl logs nginx                        # 查看日志
nerdctl exec -it nginx bash              # 进入容器
nerdctl build -t myapp .                 # 构建镜像
nerdctl compose up -d                     # Compose 启动
nerdctl images                            # 查看镜像
nerdctl volume ls                         # 查看数据卷
```

## gVisor（沙箱运行时）

gVisor 是 Google 开发的沙箱运行时，比 runc 隔离更强。

### 与 runc 对比

| 特性 | runc | gVisor |
|------|------|--------|
| 隔离方式 | Linux namespace + cgroup | 用户态内核（Sentry） |
| 性能 | 接近原生 | 有额外开销 |
| 兼容性 | 所有 Linux 功能 | 部分限制 |
| 安全性 | 依赖内核 | 更强（用户态内核） |

### 使用方法

```bash
# 1. 安装 gVisor
# 从 https://github.com/google/gvisor/releases 下载
wget https://.../runsc
chmod +x runsc
mv runsc /usr/local/bin/

# 2. Docker 使用 gVisor 运行容器
docker run --runtime=runsc nginx:alpine

# 3. containerd 使用 gVisor
# 修改 /etc/containerd/config.toml
[plugins."io.containerd.grpc.v1.cri"]
  runtime.type = "io.containerd.runtime.v2"
  runtime.path = "/usr/local/bin/runsc"
```

## Docker 与 K8s 运行时关系

```
Docker (早期 K8s)
  └── dockershim（K8s 内置适配器）
            ↓
       containerd

K8s 1.24+（移除了 dockershim）
  ├── containerd（直接支持 CRI）
  └── cri-o（K8s 专用，轻量）
```

**为什么 K8s 1.24 移除 dockershim？**
- dockershim 是 K8s 内置的适配器，桥接 Docker 和 CRI
- K8s 统一使用 CRI（Container Runtime Interface）标准
- containerd/cri-o 直接实现 CRI，无需适配层

## 切换容器运行时（K8s）

### 从 dockershim 切换到 containerd

```bash
# 1. 安装 containerd
apt install containerd

# 2. 生成默认配置
containerd config default > /etc/containerd/config.toml

# 3. 修改配置，启用 CRI
# /etc/containerd/config.toml
[plugins."io.containerd.grpc.v1.cri"]
  sandbox_image = "registry.k8s.io/pause:3.10"

# 4. 重启 containerd
systemctl restart containerd

# 5. 修改 kubelet 配置
# /var/lib/kubelet/kubeadm-flags.env
KUBELET_EXTRA_ARGS=--container-runtime=remote --container-runtime-endpoint=unix:///run/containerd/containerd.sock

# 6. 重启 kubelet
systemctl restart kubelet
```

## 常见问题

**Q: containerd 和 Docker 区别？**

| 特性 | Docker | containerd |
|------|--------|------------|
| 定位 | 完整平台 | 运行时 |
| 镜像构建 | ✓ 支持 | ✗ 不支持 |
| 网络/存储 | ✓ 完整管理 | 基础管理 |
| API | REST API | gRPC API |
| 使用场景 | 开发/运维 | K8s 底层 |

**Q: 什么时候用 runc？**
- 一般不直接使用 runc
- containerd/cri-o 在幕后调用 runc
- 调试容器问题时可能需要

**Q: gVisor 适合什么场景？**
- 多租户环境（更强隔离）
- 安全性要求高的场景
- 性能要求不极端的场景

**Q: 如何查看当前使用的运行时？**
```bash
# Docker
docker info | grep "Runtime"

# K8s
kubectl get nodes -o wide
# 查看各节点的 runtime 版本
```
