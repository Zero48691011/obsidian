# GitHub Actions 配置文件语法

> 配置文件路径：`.github/workflows/*.yml`  
> 官方文档：https://docs.github.com/en/actions

---

## 一、文件结构总览

```yaml
name: CI Pipeline                    # 工作流名称（可选）

on:                                 # 触发条件（必填）
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:                                # 全局环境变量（可选）
  NODE_VERSION: '18'

jobs:                               # 作业定义（必填）
  job_id:                           # 作业 ID
    name: Job Display Name          # 作业显示名称（可选）
    runs-on: ubuntu-latest          # 运行环境（必填）
    steps:                          # 步骤列表（必填）
      - name: Step Name
        uses: actions/checkout@v4   # 复用 Action
      - name: Run script
        run: echo "Hello"
```

---

## 二、触发条件 (`on`)

### 事件类型

| 事件 | 说明 |
|------|------|
| `push` | 代码推送 |
| `pull_request` | PR 创建/更新 |
| `schedule` | 定时触发 (cron) |
| `workflow_dispatch` | 手动触发 |
| `workflow_call` | 被其他工作流调用 |
| `release` | 发布事件 |
| `issues` / `issue_comment` | Issue 相关 |
| `pull_request_target` | PR 目标分支上下文 |

### 分支/标签过滤

```yaml
on:
  push:
    branches:
      - main
      - 'release/**'       # 通配符
    branches-ignore:
      - 'feature/*'
    tags:
      - 'v*'
    paths:                  # 路径过滤（仅指定路径变更时触发）
      - 'src/**'
    paths-ignore:
      - 'docs/**'
```

### 定时触发

```yaml
on:
  schedule:
    - cron: '0 9 * * 1-5'  # 工作日 9:00 UTC
```

### 手动触发 (带输入参数)

```yaml
on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Deploy target'
        required: true
        default: 'staging'
        type: choice
        options:
          - staging
          - production
```

---

## 三、全局环境变量 (`env`)

```yaml
env:
  DOCKER_REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}
```

在步骤中引用：
```yaml
- run: echo ${{ env.DOCKER_REGISTRY }}
```

---

## 四、作业 (`jobs`)

### 基础结构

```yaml
jobs:
  build:
    name: Build and Test
    runs-on: ubuntu-latest
    timeout-minutes: 30          # 超时时间（默认 360）
    
    permissions:                 # 作业级权限
      contents: read
      packages: write
    
    strategy:                    # 矩阵策略
      matrix:
        node-version: [16, 18, 20]
        os: [ubuntu-latest, macos-latest]
      fail-fast: false           # 一个失败不影响其他
    
    env:                         # 作业级环境变量
      NODE_ENV: test
    
    steps:
      - uses: actions/checkout@v4
      - run: npm test
```

### 运行环境 (`runs-on`)

| 值 | 说明 |
|----|------|
| `ubuntu-latest` | Ubuntu 22.04 (x64) |
| `ubuntu-24.04` | Ubuntu 24.04 |
| `macos-latest` | macOS 14 (ARM) |
| `macos-13` | macOS 13 (x64) |
| `windows-latest` | Windows Server 2022 |
| `self-hosted` | 自托管 Runner |
| `self-hosted, linux, gpu` | 标签匹配 |

### 作业依赖

```yaml
jobs:
  lint:
    runs-on: ubuntu-latest
    steps: [ ... ]
  
  test:
    needs: lint                  # 依赖 lint 完成
    runs-on: ubuntu-latest
    steps: [ ... ]
  
  deploy:
    needs: [lint, test]          # 多个依赖
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps: [ ... ]
```

### 条件执行 (`if`)

```yaml
steps:
  - name: Deploy to prod
    if: github.ref == 'refs/heads/main' && success()
    run: ./deploy.sh

  - name: Notify failure
    if: failure()
    run: ./notify.sh
```

常用条件函数：
| 函数 | 说明 |
|------|------|
| `success()` | 前面步骤都成功 |
| `failure()` | 前面有步骤失败 |
| `always()` | 总是执行 |
| `cancelled()` | 工作流被取消 |

---

## 五、步骤 (`steps`)

### 使用 Action

```yaml
steps:
  - uses: actions/checkout@v4        # 官方 Action
    with:
      fetch-depth: 0                  # 完整历史

  - uses: docker/login-action@v3
    with:
      registry: ghcr.io
      username: ${{ github.actor }}
      password: ${{ secrets.GITHUB_TOKEN }}
```

### 运行命令

```yaml
steps:
  - name: Install dependencies
    run: |
      npm ci
      npm run build
  
  - name: Run tests
    run: npm test
    working-directory: ./src          # 工作目录
    shell: bash                       # Shell 类型
    env:                              # 步骤级环境变量
      CI: true
```

### 条件步骤

```yaml
steps:
  - name: Deploy production
    if: github.ref == 'refs/heads/main'
    run: ./deploy-prod.sh
```

### 输出变量

```yaml
steps:
  - id: set-var
    run: echo "version=1.2.3" >> $GITHUB_OUTPUT
  
  - run: echo "Version is ${{ steps.set-var.outputs.version }}"
```

---

## 六、上下文 (`${{ }}`)

### 常用上下文

| 上下文 | 说明 |
|--------|------|
| `github` | 仓库/事件信息 |
| `env` | 环境变量 |
| `secrets` | 加密密钥 |
| `matrix` | 矩阵变量 |
| `steps` | 步骤输出 |
| `needs` | 依赖作业输出 |
| `inputs` | 工作流输入参数 |
| `vars` | 仓库/组织变量 |

### 常用 `github` 属性

| 属性 | 说明 |
|------|------|
| `github.ref` | 分支/标签 ref |
| `github.sha` | 提交 SHA |
| `github.event_name` | 触发事件名 |
| `github.actor` | 触发用户 |
| `github.repository` | 仓库名 (owner/repo) |
| `github.run_id` | 运行 ID |
| `github.workspace` | 工作目录路径 |

---

## 七、密钥和变量 (`secrets` / `vars`)

### 使用密钥

```yaml
steps:
  - run: curl -H "Authorization: Bearer ${{ secrets.API_TOKEN }}" https://api.example.com
```

### 使用变量

```yaml
env:
  ENVIRONMENT: ${{ vars.ENVIRONMENT }}
```

---

## 八、缓存和产物

### 缓存

```yaml
steps:
  - uses: actions/cache@v4
    with:
      path: |
        ~/.npm
        node_modules
      key: npm-${{ runner.os }}-${{ hashFiles('package-lock.json') }}
      restore-keys: npm-${{ runner.os }}-
```

### 上传产物

```yaml
steps:
  - uses: actions/upload-artifact@v4
    with:
      name: build-output
      path: dist/
```

### 下载产物

```yaml
steps:
  - uses: actions/download-artifact@v4
    with:
      name: build-output
```

---

## 九、服务容器 (`services`)

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: test
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    steps:
      - run: npm test
```

---

## 十、可复用工作流

### 定义

```yaml
# .github/workflows/reusable.yml
name: Reusable Workflow
on:
  workflow_call:
    inputs:
      node-version:
        required: true
        type: string
    secrets:
      NPM_TOKEN:
        required: true
    outputs:
      build-artifact:
        value: ${{ jobs.build.outputs.artifact }}

jobs:
  build:
    outputs:
      artifact: ${{ steps.upload.outputs.artifact-id }}
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
```

### 调用

```yaml
jobs:
  call-workflow:
    uses: ./.github/workflows/reusable.yml
    with:
      node-version: '18'
    secrets:
      NPM_TOKEN: ${{ secrets.NPM_TOKEN }}
```

---

## 十一、完整示例

```yaml
name: Node.js CI/CD

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  NODE_VERSION: '20'
  REGISTRY: ghcr.io

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
      - run: npm ci
      - run: npm run lint

  test:
    needs: lint
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node-version: [18, 20, 22]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'npm'
      - run: npm ci
      - run: npm test

  build-and-push:
    needs: test
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v4
      - uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - uses: docker/build-push-action@v5
        with:
          push: true
          tags: ${{ env.REGISTRY }}/${{ github.repository }}:latest
```

---

## 十二、常见问题

| 问题 | 解决方案 |
|------|----------|
| 密钥不暴露 | 使用 `${{ secrets.XXX }}`，不可直接 echo |
| 权限不足 | 在作业中添加 `permissions` 块 |
| 超时 | 设置 `timeout-minutes`，默认 360 分钟 |
| 矩阵过大 | 设置 `max-parallel` 限制并发 |
| 步骤间共享数据 | 使用 `$GITHUB_OUTPUT` 或 artifact |