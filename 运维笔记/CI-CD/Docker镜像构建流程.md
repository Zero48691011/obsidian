# Docker 镜像构建流程详解

## 概述

Docker 镜像构建是将应用代码 + 运行环境打包成可移植镜像的过程。理解 Standard 流程，才能理解为什么需要 Kaniko、BuildKit 等变体。

---

## 一、整体流程

```
开发者提交代码
      │
      ▼
┌─────────────────────────────────────────────────────┐
│                   CI/CD Pipeline                     │
│                                                      │
│  1. git clone 代码                                   │
│  2. docker build -t myapp:v1.0 .                     │
│  3. docker push myapp:v1.0                           │
│  4. kubectl set image deploy/myapp myapp=v1.0        │
│                                                      │
└─────────────────────────────────────────────────────┘
```

---

## 二、`docker build` 内部发生了什么

```
Dockerfile 逐行处理：

FROM ubuntu:22.04          ──→ 拉取基础镜像（如已缓存则跳过）
RUN apt-get update          ──→ 启动临时容器 → 执行命令 → 对比文件系统快照 → 生成新层
RUN apt-get install -y nginx
COPY . /app                 ──→ 复制文件 → 生成新层
RUN pip install -r requirements.txt
CMD ["nginx", "-g", "daemon off;"]  ──→ 写入镜像元数据

最终产出：分层镜像 tar 包
```

### 关键机制：分层存储

```
镜像层结构：
┌─────────────────────────────┐
│  Layer 5: CMD + ENTRYPOINT  │  ← 元数据层（不占空间）
├─────────────────────────────┤
│  Layer 4: pip install 的结果 │  ← 只有新增/修改的文件
├─────────────────────────────┤
│  Layer 3: COPY . /app       │
├─────────────────────────────┤
│  Layer 2: apt-get install   │
├─────────────────────────────┤
│  Layer 1: apt-get update    │
├─────────────────────────────┤
│  Layer 0: ubuntu:22.04      │  ← 基础镜像层
└─────────────────────────────┘
```

- 每层只存储**相对于上一层的文件变化**
- 层是只读的，容器运行时在最上层加一个**可写层**
- **拉取/推送时，相同的层只需传输一次**（分层复用）

---

## 三、三种构建方式对比

### 3.1 标准 Docker（最常用）

```
┌──────────────┐      ┌──────────────────┐      ┌──────────────┐
│ Docker CLI   │─────▶│  Docker Daemon    │─────▶│  Registry    │
│ (docker build)│      │  (dockerd)        │      │  (Docker Hub)│
└──────────────┘      │                    │      └──────────────┘
                      │  1. 解析 Dockerfile│
                      │  2. 拉取基础镜像    │
                      │  3. 逐层构建        │
                      │  4. 打包推送        │
                      └────────────────────┘

前提条件：必须运行 dockerd（Docker Daemon）
权限要求：需要 root 或 docker 组成员
```

**`.gitlab-ci.yml` 示例：**

```yaml
build:
  stage: build
  image: docker:latest
  services:
    - docker:dind          # Docker-in-Docker，提供 Daemon
  script:
    - docker build -t myapp:$CI_COMMIT_SHA .
    - docker push myapp:$CI_COMMIT_SHA
```

### 3.2 Kaniko（无 Daemon）

```
┌──────────────┐      ┌──────────────────┐      ┌──────────────┐
│ Kaniko       │─────▶│  用户空间执行      │─────▶│  Registry    │
│ Executor     │      │  (不需要 dockerd)  │      │              │
└──────────────┘      │                    │      └──────────────┘
                      │  1. 自己解析 Dockerfile│
                      │  2. 自己拉取基础镜像   │
                      │  3. 文件系统快照对比    │
                      │  4. 自己打包推送       │
                      └────────────────────┘

前提条件：不需要 dockerd
权限要求：不需要 root（容器内运行即可）
```

### 3.3 BuildKit（Docker 的下一代构建引擎）

```
┌──────────────┐      ┌──────────────────┐      ┌──────────────┐
│ buildctl     │─────▶│  buildkitd       │─────▶│  Registry    │
│ (BuildKit    │      │  (BuildKit 守护进程)│      │              │
│  CLI)        │      │                    │      └──────────────┘
└──────────────┘      │  1. 并发构建多阶段   │
                      │  2. 智能缓存         │
                      │  3. 构建秘钥管理     │
                      └────────────────────┘

前提条件：需要 buildkitd
优势：速度快、支持 rootless、缓存更智能
```

---

## 四、构建流程的三种架构模式

### 模式 1：Docker-in-Docker (DinD)

```
CI Job Pod
├── docker:latest 容器
│   └── docker build → docker.sock → dind 容器
├── docker:dind 容器（Daemon）
│   └── 实际执行构建
└── 问题：需要 privileged 模式，安全风险
```

### 模式 2：挂载 docker.sock

```
CI Job Pod
└── docker:latest 容器
    └── docker build → /var/run/docker.sock → 宿主机 dockerd

问题：共享宿主机 Daemon，安全隔离差
```

### 模式 3：Kaniko（无 Daemon）

```
CI Job Pod
└── kaniko-executor 容器
    └── 用户空间构建 → 直接推送到 Registry

优势：不需要 Daemon，不需要 privileged，安全
```

---

## 五、标准构建 vs 弹性构建

```
标准构建（固定资源）：
┌─────────────┐
│  Runner VM  │  ← 固定的 2 台机器，常驻运行
│  (4C/8Gi)   │
└─────────────┘
问题：闲时浪费资源，忙时构建排队

弹性构建（ASK + ECI）：
┌──┐ ┌──┐ ┌──┐ ┌──┐ ┌──┐  ← 按需创建，构建完自动销毁
└──┘ └──┘ └──┘ └──┘ └──┘
优势：不用不花钱，忙时自动扩容
```

---

## 六、构建优化技巧

| 技巧 | 作用 | 示例 |
|------|------|------|
| **多阶段构建** | 分离构建环境和运行环境，镜像更小 | `FROM golang AS builder` → `FROM alpine` |
| **.dockerignore** | 排除不需要的文件，减少上下文 | `node_modules/`、`.git/` |
| **层顺序优化** | 不常变的放前面，利用缓存 | 先 `COPY requirements.txt`，再 `RUN pip install` |
| **合并 RUN** | 减少层数 | `RUN apt update && apt install -y pkg && rm -rf /var/lib/apt/lists/*` |
| **基础镜像选型** | 用小镜像 | `alpine` 替代 `ubuntu`，`distroless` 替代完整 OS |
| **缓存挂载** | BuildKit 特性，包管理器缓存复用 | `RUN --mount=type=cache,target=/var/cache/apt` |

---

## 七、一句话总结

| 阶段 | 做什么 |
|------|------|
| **构建** | 解析 Dockerfile → 逐行执行 → 生成分层镜像 |
| **推送** | 镜像 + 层 → 上传到 Registry |
| **部署** | k8s 拉取镜像 → 启动容器 |

> **标准流程 = Docker CLI 调 Daemon 构建**。那个海外弹性方案之所以用 Kaniko，是因为 ECI 里跑不了 Docker Daemon，Kaniko 在用户空间自己搞定一切。

---

*文档创建时间：2026-07-02*