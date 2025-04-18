
#### Docker 是什么
- **定义**：
  - Docker 是一个开源的容器化平台，用于开发、部署和运行应用程序，通过容器（Container）将应用及其依赖打包为一个轻量、可移植的单元。
- **本质**：
  - 基于 Linux 容器技术，提供标准化的环境隔离。

#### 有什么用
1. **环境一致性**：开发、测试、生产环境统一。
2. **快速部署**：镜像打包，一键运行。
3. **资源隔离**：多个容器共享主机资源，互不干扰。
4. **高效利用**：比虚拟机轻量，启动快。
5. **微服务支持**：便于拆分和扩展应用。

#### 底层实现原理
- **核心技术**：
  1. **Namespace**：隔离进程、用户、网络等。
  2. **Cgroup**：控制资源（如 CPU、内存）。
  3. **UnionFS**：分层文件系统，实现镜像复用。
- **运行时**：
  - 使用 `containerd` 或 `runc` 执行容器。

#### 核心点
- Docker 是容器化工具，简化部署，依赖 Linux 内核特性。

---

### 1. Docker 是什么
#### (1) 基本概念
- **容器**：
  - 一个运行时实例，包含应用、依赖和配置。
- **镜像**：
  - 只读模板，用于创建容器（如 Ubuntu + Nginx）。
- **Docker Engine**：
  - 核心组件，管理镜像和容器。
- **与虚拟机对比**：
  - 虚拟机：模拟完整 OS，包含 Hypervisor。
  - Docker：共享主机内核，仅隔离应用层。

#### 图示
```
虚拟机: [App + Libs + Guest OS] --> Hypervisor --> Host OS
Docker: [App + Libs] --> Docker Engine --> Host OS
```

---

### 2. Docker 有什么用
#### (1) 环境一致性
- 解决“在我机器上能跑”问题。
- 示例：开发用 Python 3.8，生产也保证一致。

#### (2) 快速部署
- 镜像打包一次，到处运行。
- 示例：`docker run -d nginx` 秒级启动。

#### (3) 资源隔离
- 每个容器有独立的文件系统、网络、进程。
- 示例：两个容器跑不同版本的 MySQL。

#### (4) 高效利用
- 无 Guest OS，开销小，启动快（秒级 vs 分钟级）。
- 示例：单机跑数十个容器。

#### (5) 微服务支持
- 拆分应用为多个容器，配合 Docker Compose/Kubernetes。
- 示例：前端、后端、数据库各一个容器。

---

### 3. 底层实现原理
#### (1) Namespace（命名空间）
- **作用**：
  - 隔离容器与主机及其他容器。
- **类型**：
  - **PID**：独立进程 ID。
  - **NET**：独立网络栈（如 IP）。
  - **MNT**：独立文件系统。
  - **UTS**：独立主机名。
  - **User**：独立用户权限。
- **示例**：
  - 容器内 `ps` 只看到自身进程。

#### (2) Cgroup（控制组）
- **作用**：
  - 限制和分配资源（CPU、内存、磁盘 I/O）。
- **实现**：
  - 通过文件系统（如 `/sys/fs/cgroup`）设置限额。
- **示例**：
```bash
docker run --memory="512m" --cpu-shares=512 nginx
```
  - 限制容器用 512MB 内存和相对 CPU 份额。

#### (3) UnionFS（联合文件系统）
- **作用**：
  - 分层存储镜像，复用底层文件。
- **实现**：
  - 如 AUFS、OverlayFS。
  - 镜像层只读，容器层可写（Copy-on-Write）。
- **示例**：
```
FROM ubuntu  # 基础层
RUN apt install nginx  # 添加层
```
  - 构建镜像分层保存。

#### (4) 运行时
- **containerd**：
  - 高层管理容器生命周期。
- **runc**：
  - 底层执行容器，符合 OCI 标准。
- **流程**：
  - Docker Daemon 调用 containerd → runc 创建容器。

#### 图示
```
[Docker CLI] --> [Docker Daemon] --> [containerd] --> [runc]
                            --> Namespace + Cgroup + UnionFS
```

---

### 4. 工作流程
1. **构建镜像**：
   - `docker build -t myapp .`（基于 Dockerfile）。
2. **运行容器**：
   - `docker run -d myapp`（创建并启动）。
3. **管理**：
   - `docker ps`、`docker stop`。

#### 示例 Dockerfile
```dockerfile
FROM openjdk:8
COPY app.jar /app.jar
CMD ["java", "-jar", "/app.jar"]
```

---

### 5. 优缺点
#### 优点
- 轻量、快速、一致。
- 易于分发和扩展。

#### 缺点
- 依赖 Linux 内核（Windows/Mac 用 VM）。
- 安全性低于 VM（共享内核）。

---

### 6. 延伸与面试角度
- **与 Kubernetes**：
  - Docker 提供容器，K8s 管理集群。
- **实际应用**：
  - CI/CD：Jenkins 构建镜像。
  - 微服务：每个服务一个容器。
- **面试点**：
  - 问“原理”时，提 Namespace 和 Cgroup。
  - 问“用途”时，提一致性和部署。

---

### 总结
Docker 是容器化平台，通过镜像和容器简化开发部署，利用 Namespace、Cgroup 和 UnionFS 实现隔离和高效。面试时，可提技术细节或画架构图，展示理解深度。