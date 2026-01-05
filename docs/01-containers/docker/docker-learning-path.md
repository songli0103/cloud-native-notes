# 云原生程序员 Docker 知识体系指南

作为一个云原生（Cloud Native）程序员，掌握 Docker 不仅仅是会敲几个命令，更重要的是理解容器技术的底层原理、最佳实践以及如何与 Kubernetes、CI/CD 流水线高效协作。

本文档将 Docker 知识划分为五个阶段：**核心基础**、**镜像工程**、**运行时管理**、**云原生进阶**以及**排错与安全**。

---

## 📅 阶段一：核心概念与架构 (Foundations)

在动手之前，必须理解 Docker 是如何工作的，这对于后续理解 Kubernetes 的 Pod 机制至关重要。

### 1.1 核心术语
- **镜像 (Image)**: 只读的模板，包含运行应用所需的所有依赖（代码、库、环境变量、配置）。
- **容器 (Container)**: 镜像的运行实例，是一个隔离的进程。
- **仓库 (Repository/Registry)**: 存放镜像的地方（如 Docker Hub, Harbor, ACR/ECR）。

### 1.2 架构原理
- **Client-Server 架构**: `docker` CLI 与 `dockerd` 守护进程的通信机制。
- **底层技术 (关键)**:
    - **Namespace (命名空间)**: 实现资源隔离 (PID, NET, MNT, UTS, IPC, USER)。
    - **Cgroups (控制组)**: 实现资源限制 (CPU, Memory, IO)。
    - **UnionFS (联合文件系统)**: 镜像分层存储的原理 (Overlay2)。
- **OCI 标准**: 了解 Open Container Initiative。
    - **RunC**: 低层运行时。
    - **Containerd**: 高层运行时（Kubernetes 现阶段主要对接的对象）。

---

## 🛠️ 阶段二：镜像工程 (Image Engineering)

这是开发人员日常最主要的工作：编写高质量的 `Dockerfile`。

### 2.1 Dockerfile 核心指令
- `FROM`: 基础镜像选择 (Alpine vs Debian vs Scratch)。
- `RUN`: 执行命令构建层。
- `COPY` vs `ADD`: 文件复制的区别。
- `CMD` vs `ENTRYPOINT`: 容器启动命令的区别与 Shell/Exec 模式。
- `ENV` vs `ARG`: 构建时变量与运行时环境变量。
- `WORKDIR`: 切换工作目录。

### 2.2 最佳实践 (Cloud Native Essential)
- **多阶段构建 (Multi-stage Builds)**: 极其重要！分离构建环境和运行环境，大幅减小镜像体积。
- **构建缓存 (Layer Caching)**: 利用分层缓存加速构建（将 `package.json` 或 `go.mod` 的复制放在源码复制之前）。
- **最小权限原则**: 创建非 root 用户运行服务 (`USER appuser`)。
- **`.dockerignore`**: 排除无关文件（`.git`, `node_modules`）以减小上下文。
- **PID 1 问题**: 理解信号传递，使用 `tini` 等 init 进程管理僵尸进程。

---

## 🌐 阶段三：网络与存储 (Networking & Storage)

理解容器如何与外部世界交互，以及数据如何持久化。

### 3.1 容器网络
- **Bridge 模式**: 默认网络，NAT 转换，端口映射 (`-p`).
- **Host 模式**: 共享宿主机网络栈（高性能，但端口冲突风险）。
- **None 模式**: 无网络。
- **Container 模式**: 多个容器共享同一个 Network Namespace (Sidecar 模式的基础)。
- **DNS 解析**: Docker 内部如何通过服务名互相发现。

### 3.2 数据持久化
- **Volumes (数据卷)**: Docker 管理的存储区域，这是生产环境的首选。
- **Bind Mounts (绑定挂载)**: 挂载宿主机任意路径，开发环境常用。
- **Tmpfs**: 内存挂载，用于敏感数据或临时数据。

---

## 🧩 阶段四：编排与开发环境 (Orchestration)

虽然生产环境用 Kubernetes，但本地开发离不开 Docker Compose。

### 4.1 Docker Compose
- **`docker-compose.yaml`**: 编写多容器应用配置。
- **核心概念**: Services, Networks, Volumes。
- **常用命令**: `up -d`, `down`, `logs -f`, `exec`, `build`.
- **环境变量管理**: 使用 `.env` 文件。

---

## 🚀 阶段五：云原生进阶 (Cloud Native Advanced)

这部分知识将帮助你从 Docker 平滑过渡到 Kubernetes。

### 5.1 从 Docker 到 Kubernetes
- **CRI (Container Runtime Interface)**: 理解为什么 K8s 弃用了 Docker Shim，转而直接支持 Containerd/CRI-O。
- **Pod 的概念**: 理解 Docker 容器如何对应 K8s 中的 Container，以及 Pause 容器的作用。

### 5.2 生产环境安全
- **镜像扫描**: 使用 Trivy 或 Clair 扫描镜像漏洞。
- **镜像签名**: 保证镜像来源可信 (Docker Content Trust)。
- **Capabilities**: 细粒度控制容器权限 (如 `--cap-add`, `--cap-drop`)，避免使用 `--privileged`。

### 5.3 镜像仓库管理
- **Harbor**: 企业级私有仓库的搭建与配置。
- **镜像标签策略**: 避免使用 `latest`，使用语义化版本或 Git Commit SHA。

---

## 🔍 阶段六：排错与运维工具 (Troubleshooting)

当容器不工作时，你需要这些工具。

### 6.1 常用排错命令
- `docker logs`: 查看标准输出/错误日志。
- `docker inspect`: 查看容器/镜像的详细元数据（IP、挂载点、环境变量）。
- `docker stats`: 实时监控容器资源使用 (CPU/Mem)。
- `docker top`: 查看容器内运行的进程。
- `docker exec -it <id> /bin/sh`: 进入容器现场调试。

### 6.2 清理与维护
- `docker system prune`: 清理未使用的镜像、容器和网络。
- `docker system df`: 查看磁盘占用情况。

---

## 📚 学习资源推荐

1. **官方文档**: [docs.docker.com](https://docs.docker.com/) (最权威)
2. **实验室**: [Play with Docker](https://labs.play-with-docker.com/)
3. **书籍**: 《Docker 容器与容器云》
4. **进阶**: 深入阅读 CNCF Landscape 中关于 Container Runtime 的部分。

---

> **给云原生程序员的建议**：
> 不要只停留在“会用”层面。当你写 Dockerfile 时，要想着这个镜像未来会在 Kubernetes 集群中如何调度、如何扩缩容、如何配置探针（Liveness/Readiness Probe）。**Docker 是云原生的基石，基石稳，高楼才稳。**