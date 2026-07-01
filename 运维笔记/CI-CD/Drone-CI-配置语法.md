# Drone CI 配置文件语法

> 配置文件路径：`.drone.yml` 或 `.drone.yaml`  
> 官方文档：https://docs.drone.io/

---

## 一、文件结构总览

```yaml
---
kind: pipeline
type: docker
name: default

platform:
  os: linux
  arch: amd64

trigger:
  branch:
    - main

steps:
  - name: build
    image: node:20
    commands:
      - npm ci
      - npm run build
```

---

## 二、管道类型 (`kind`)

| 类型 | 说明 |
|------|------|
| `pipeline` | 常规 CI/CD 管道 |
| `secret` | 加密密钥 |
| `signature` | 资源签名 |

---

## 三、运行类型 (`type`)

| 类型 | 说明 |
|------|------|
| `docker` | Docker 容器中运行（最常用） |
| `exec` | 直接在主机上执行 |
| `ssh` | 通过 SSH 远程执行 |
| `kubernetes` | 在 Kubernetes Pod 中运行 |
| `macstadium` | macOS 虚拟机 |

### Docker 管道

```yaml
---
kind: pipeline
type: docker
name: default

steps:
  - name: test
    image: node:20
    commands:
      - npm test
```

### Exec 管道

```yaml
---
kind: pipeline
type: exec
name: default

platform:
  os: linux
  arch: amd64

steps:
  - name: build
    commands:
      - make build
      - make test
```

### SSH 管道

```yaml
---
kind: pipeline
type: ssh
name: deploy

server:
  host: prod.example.com
  user: deploy
  ssh_key:
    from_secret: ssh_key

steps:
  - name: deploy
    commands:
      - cd /opt/app && docker-compose up -d
```

---

## 四、步骤 (`steps`)

### 基础步骤

```yaml
steps:
  - name: build
    image: node:20-alpine
    pull: always
    commands:
      - npm ci
      - npm run build
    environment:
      NODE_ENV: production
      API_URL: https://api.example.com
    when:
      branch:
        - main
```

### 拉取策略

```yaml
steps:
  - name: a
    image: alpine:latest
    pull: always
  - name: b
    image: my-image:latest
    pull: never
  - name: c
    image: node:20
    pull: if-not-exists
```

### 失败处理

```yaml
steps:
  - name: a
    image: node:20
    failure: ignore
    commands:
      - npm run flaky-test
  - name: b
    image: node:20
    failure: always
    commands:
      - npm run critical-check
```

### 后台服务

```yaml
steps:
  - name: start-database
    image: postgres:15
    detach: true
    environment:
      POSTGRES_DB: test
      POSTGRES_USER: test
      POSTGRES_PASSWORD: test

  - name: test
    image: node:20
    environment:
      DATABASE_URL: postgres://test:***@start-database:5432/test
    commands:
      - npm test
```

---

## 五、插件 (`plugins`)

### 使用插件

```yaml
steps:
  - name: publish
    image: plugins/docker
    settings:
      repo: myorg/myapp
      tags:
        - latest
        - ${DRONE_COMMIT_SHA}
      username:
        from_secret: docker_username
      password:
        from_secret: docker_password
```

### 常用插件

| 插件 | 用途 |
|------|------|
| `plugins/docker` | 构建和推送 Docker 镜像 |
| `plugins/github-release` | 创建 GitHub Release |
| `plugins/s3` | 上传到 AWS S3 |
| `plugins/ecr` | 推送到 AWS ECR |
| `plugins/slack` | 发送 Slack 通知 |
| `plugins/webhook` | 发送 Webhook |

### Docker 插件示例

```yaml
steps:
  - name: docker-build
    image: plugins/docker
    settings:
      registry: registry.example.com
      repo: registry.example.com/myapp
      tags:
        - latest
        - ${DRONE_COMMIT_SHA:0:8}
      dockerfile: Dockerfile
      build_args:
        - NODE_ENV=production
      username:
        from_secret: docker_username
      password:
        from_secret: docker_password
```

### Slack 插件示例

```yaml
steps:
  - name: notify
    image: plugins/slack
    settings:
      webhook:
        from_secret: slack_webhook
      channel: deployments
      username: drone
      template: |
        {{#success build.status}}
          Build {{build.number}} succeeded.
        {{else}}
          Build {{build.number}} failed.
        {{/success}}
```

---

## 六、触发条件 (`trigger`)

### 分支触发

```yaml
trigger:
  branch:
    - main
    - develop
    - feature/*
```

### 事件触发

```yaml
trigger:
  event:
    - push
    - pull_request
    - tag
    - promote
    - rollback
    - cron
```

### 排除条件

```yaml
trigger:
  branch:
    exclude:
      - gh-pages
      - experimental/*
```

### 路径触发

```yaml
trigger:
  branch:
    - main
  paths:
    include:
      - src/**
      - package.json
    exclude:
      - docs/**
      - README.md
```

### Tag 触发

```yaml
trigger:
  event:
    - tag
  ref:
    - refs/tags/v*
```

### Pull Request 触发

```yaml
trigger:
  event:
    - pull_request
```

### 动作触发

```yaml
trigger:
  event:
    - promote
    - rollback
  target:
    - production
```

---

## 七、多管道 (`depends_on`)

```yaml
---
kind: pipeline
type: docker
name: lint

steps:
  - name: lint
    image: node:20
    commands:
      - npm run lint

---
kind: pipeline
type: docker
name: test

depends_on:
  - lint

steps:
  - name: test
    image: node:20
    commands:
      - npm test

---
kind: pipeline
type: docker
name: deploy

depends_on:
  - test

trigger:
  branch:
    - main

steps:
  - name: deploy
    image: alpine
    commands:
      - echo "Deploying..."
```

---

## 八、服务 (`services`) — 旧版语法

```yaml
---
kind: pipeline
type: docker
name: test

services:
  - name: database
    image: postgres:15
    environment:
      POSTGRES_DB: test
      POSTGRES_USER: test
      POSTGRES_PASSWORD: test
    ports:
      - 5432

steps:
  - name: test
    image: node:20
    environment:
      DATABASE_URL: postgres://test:***@database:5432/test
    commands:
      - npm test
```

---

## 九、卷挂载 (`volumes`)

### 临时卷

```yaml
steps:
  - name: build
    image: node:20
    volumes:
      - name: cache
        path: /root/.npm
    commands:
      - npm ci

volumes:
  - name: cache
    temporary: {}
```

### 主机路径卷

```yaml
volumes:
  - name: cache
    host:
      path: /var/lib/cache
```

### Docker 声明卷

```yaml
volumes:
  - name: data
    claim:
      name: drone-data
```

---

## 十、节点选择 (`node`)

```yaml
---
kind: pipeline
type: docker
name: gpu-build

node:
  arch: amd64
  os: linux
  gpu: true

steps:
  - name: train
    image: tensorflow/tensorflow:latest-gpu
    commands:
      - python train.py
```

---

## 十一、环境变量 (`environment`)

### 全局环境变量

```yaml
---
kind: pipeline
type: docker
name: default

environment:
  NODE_ENV: production
  REGISTRY: registry.example.com

steps:
  - name: build
    image: node:20
    commands:
      - echo $NODE_ENV
```

### 步骤级环境变量

```yaml
steps:
  - name: test
    image: node:20
    environment:
      CI: "true"
      DATABASE_URL: "postgres://localhost/test"
    commands:
      - npm test
```

### 从密钥注入

```yaml
steps:
  - name: deploy
    image: alpine
    environment:
      API_KEY:
        from_secret: api_key
    commands:
      - ./deploy.sh
```

---

## 十二、内置环境变量

| 变量 | 说明 |
|------|------|
| `DRONE_BUILD_NUMBER` | 构建编号 |
| `DRONE_COMMIT_SHA` | 提交 SHA |
| `DRONE_COMMIT_BRANCH` | 分支名 |
| `DRONE_COMMIT_TAG` | 标签名 |
| `DRONE_COMMIT_MESSAGE` | 提交信息 |
| `DRONE_COMMIT_AUTHOR` | 提交作者 |
| `DRONE_BUILD_EVENT` | 事件类型 |
| `DRONE_TAG` | Tag 名称 |
| `DRONE_REPO` | 仓库完整名 |
| `DRONE_REPO_OWNER` | 仓库所有者 |
| `DRONE_REPO_NAME` | 仓库名 |
| `DRONE_SOURCE_BRANCH` | PR 源分支 |
| `DRONE_TARGET_BRANCH` | PR 目标分支 |
| `DRONE_PULL_REQUEST` | PR 编号 |
| `DRONE_DEPLOY_TO` | 部署目标 |
| `DRONE_STAGE_NAME` | 阶段名 |
| `DRONE_STEP_NAME` | 步骤名 |
| `CI` | 总是 `true` |

---

## 十三、密钥管理 (`from_secret`)

在 Drone 管理界面或 CLI 添加密钥：

```bash
drone secret add \
  --repository myorg/myrepo \
  --name docker_password \
  --value my-password
```

在配置中使用：

```yaml
steps:
  - name: docker-push
    image: plugins/docker
    settings:
      username:
        from_secret: docker_username
      password:
        from_secret: docker_password
```

---

## 十四、克隆选项 (`clone`)

```yaml
---
kind: pipeline
type: docker
name: default

clone:
  depth: 50
  disable: false
  skip_verify: false
  tags: false
  retries: 3

steps:
  ...
```

---

## 十五、完整示例

```yaml
---
kind: pipeline
type: docker
name: lint

platform:
  os: linux
  arch: amd64

trigger:
  event:
    - push
    - pull_request

steps:
  - name: lint
    image: node:20-alpine
    commands:
      - npm ci
      - npm run lint

---
kind: pipeline
type: docker
name: test

depends_on:
  - lint

platform:
  os: linux
  arch: amd64

steps:
  - name: start-database
    image: postgres:15-alpine
    detach: true
    environment:
      POSTGRES_DB: test
      POSTGRES_USER: test
      POSTGRES_PASSWORD: test

  - name: test
    image: node:20-alpine
    environment:
      DATABASE_URL: postgres://test:***@start-database:5432/test
    commands:
      - npm ci
      - npm test

---
kind: pipeline
type: docker
name: build-and-push

depends_on:
  - test

trigger:
  branch:
    - main
  event:
    - push

volumes:
  - name: docker
    host:
      path: /var/run/docker.sock

steps:
  - name: build
    image: node:20-alpine
    commands:
      - npm ci
      - npm run build

  - name: docker
    image: plugins/docker
    volumes:
      - name: docker
        path: /var/run/docker.sock
    settings:
      registry: registry.example.com
      repo: registry.example.com/myapp
      tags:
        - latest
        - ${DRONE_COMMIT_SHA:0:8}
      username:
        from_secret: docker_username
      password:
        from_secret: docker_password

  - name: notify
    image: plugins/slack
    settings:
      webhook:
        from_secret: slack_webhook
      channel: deployments
    when:
      status:
        - success
        - failure

---
kind: pipeline
type: docker
name: deploy

depends_on:
  - build-and-push

trigger:
  event:
    - promote
  target:
    - production

steps:
  - name: deploy
    image: alpine/k8s:1.28.0
    environment:
      KUBE_CONFIG:
        from_secret: kube_config
    commands:
      - kubectl set image deployment/myapp app=myapp:${DRONE_COMMIT_SHA:0:8}
      - kubectl rollout status deployment/myapp
```

---

## 十六、YAML 锚点模板

```yaml
---
kind: pipeline
type: docker
name: default

definitions:
  steps:
    - &npm-install
      name: install
      image: node:20-alpine
      commands:
        - npm ci

steps:
  - *npm-install

  - <<: *npm-install
    name: install-prod
    commands:
      - npm ci --production

  - name: test
    image: node:20-alpine
    commands:
      - npm test
```

---

## 十七、常见问题

| 问题 | 解决方案 |
|------|----------|
| 密钥不可用 | 确认密钥已添加到仓库，检查 `from_secret` 名称拼写 |
| 步骤间数据共享 | 使用 `volumes` 临时卷或挂载 Docker socket |
| 后台服务连接失败 | 后台服务名即步骤 `name`，以此为 hostname 连接 |
| 管道不触发 | 检查 `trigger` 条件，确认 Webhook 已配置 |
| 自定义镜像拉取失败 | 配置私有仓库认证或设置 `pull: never` |