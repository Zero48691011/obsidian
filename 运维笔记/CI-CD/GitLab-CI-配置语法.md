# GitLab CI 配置文件语法

> 配置文件路径：`.gitlab-ci.yml`  
> 官方文档：https://docs.gitlab.com/ee/ci/yaml/

---

## 一、文件结构总览

```yaml
stages:                          # 阶段定义（必填）
  - build
  - test
  - deploy

variables:                       # 全局变量（可选）
  DOCKER_DRIVER: overlay2

before_script:                   # 每个作业前执行的命令
  - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD

job_name:                        # 作业名
  stage: build                   # 所属阶段
  script:                        # 执行的命令（必填）
    - echo "Building..."
  only:                          # 触发条件
    - main
```

---

## 二、阶段 (`stages`)

```yaml
stages:
  - build
  - test
  - deploy

# 自定义阶段执行顺序
build:
  stage: build
  script: make build

test:
  stage: test
  script: make test

deploy:
  stage: deploy
  script: make deploy
```

### 预定义阶段

| 阶段 | 说明 |
|------|------|
| `.pre` | 最先执行，总是在 pipeline 最开始 |
| `build` | 编译构建 |
| `test` | 测试 |
| `deploy` | 部署 |
| `.post` | 最后执行，总是在 pipeline 最末尾 |

---

## 三、变量 (`variables`)

### 全局变量

```yaml
variables:
  DOCKER_REGISTRY: registry.gitlab.com
  APP_NAME: my-app
```

### 作业级变量

```yaml
build:
  variables:
    BUILD_ENV: production
  script:
    - echo $BUILD_ENV
```

### 预定义变量

| 变量 | 说明 |
|------|------|
| `CI_COMMIT_SHA` | 提交 SHA |
| `CI_COMMIT_REF_NAME` | 分支名或标签名 |
| `CI_COMMIT_BRANCH` | 分支名 |
| `CI_COMMIT_TAG` | 标签名 |
| `CI_PROJECT_DIR` | 项目完整路径 |
| `CI_PIPELINE_ID` | Pipeline ID |
| `CI_JOB_ID` | 作业 ID |
| `CI_REGISTRY` | GitLab Registry 地址 |
| `CI_REGISTRY_IMAGE` | 项目镜像名 |
| `CI_DEFAULT_BRANCH` | 默认分支名 |

---

## 四、脚本 (`script` / `before_script` / `after_script`)

```yaml
job:
  before_script:
    - echo "Running before job..."
    - mkdir -p build
  
  script:
    - echo "Running main job..."
    - make build
  
  after_script:
    - echo "Running after job..."
    - rm -rf build
```

### 多行命令

```yaml
job:
  script:
    - |
      if [ "$CI_COMMIT_BRANCH" = "main" ]; then
        echo "Production build"
        make release
      else
        echo "Development build"
        make debug
      fi
```

---

## 五、触发条件

### `only` / `except`（基础控制）

```yaml
job:
  only:
    - main
    - develop
    - tags
    - merge_requests
  except:
    - feature/*
```

### `rules`（推荐使用）

```yaml
job:
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
      when: always
    - if: $CI_COMMIT_BRANCH == "develop"
      when: on_success
    - if: $CI_COMMIT_TAG
      when: manual
    - when: never
```

### `when` 选项

| 值 | 说明 |
|----|------|
| `on_success` | 前面阶段都成功时执行（默认） |
| `on_failure` | 前面阶段有失败时执行 |
| `always` | 总是执行 |
| `manual` | 手动触发 |
| `delayed` | 延迟执行 |
| `never` | 不执行 |

### 手动触发

```yaml
deploy:
  stage: deploy
  script: ./deploy.sh
  when: manual
  allow_failure: false         # 必须手动触发
```

### 延迟执行

```yaml
deploy:
  stage: deploy
  script: ./deploy.sh
  when: delayed
  start_in: 30 minutes
```

---

## 六、环境 (`environment`)

```yaml
deploy_to_production:
  stage: deploy
  script: ./deploy.sh
  environment:
    name: production
    url: https://myapp.example.com
    kubernetes:
      namespace: production
    on_stop: stop_production   # 对应停止环境的作业

stop_production:
  stage: deploy
  script: ./cleanup.sh
  environment:
    name: production
    action: stop
  when: manual
```

---

## 七、缓存和产物 (`cache` / `artifacts`)

### 缓存（跨作业共享）

```yaml
build:
  cache:
    key: $CI_COMMIT_REF_SLUG
    paths:
      - node_modules/
      - .cache/
    policy: pull-push          # pull / push / pull-push
```

### 缓存键策略

```yaml
cache:
  key: $CI_COMMIT_REF_SLUG            # 按分支
  # key: "$CI_JOB_NAME-$CI_COMMIT_REF_SLUG"  # 按作业+分支
  # key:                                  # 自定义
  #   files:
  #     - package-lock.json
  #     - Gemfile.lock
```

### 产物（向下游传递）

```yaml
build:
  artifacts:
    paths:
      - dist/
      - build/
    name: "$CI_JOB_NAME-$CI_COMMIT_REF_NAME"
    when: always              # on_success / always / on_failure
    expire_in: 7 days
    reports:                  # 特定报告类型
      junit: test-results.xml
      coverage_report:
        coverage_format: cobertura
        path: coverage/cobertura-coverage.xml
```

### 依赖产物

```yaml
test:
  dependencies:
    - build
  script:
    - ls dist/
```

---

## 八、服务容器 (`services`)

```yaml
test:
  services:
    - name: postgres:15
      alias: db
      variables:
        POSTGRES_DB: test_db
        POSTGRES_USER: test
        POSTGRES_PASSWORD: test
  variables:
    DATABASE_URL: postgres://test:test@db:5432/test_db
  script:
    - npm test
```

---

## 九、多作业继承

### `extends`（继承模板）

```yaml
.docker-build:
  image: docker:latest
  services:
    - docker:dind
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD

build:
  extends: .docker-build
  stage: build
  script:
    - docker build -t $CI_REGISTRY_IMAGE .
```

### `!reference`（引用标签）

```yaml
.setup:
  setup:
    - apt-get update
    - apt-get install -y curl

job:
  script:
    - !reference [.setup, setup]
    - echo "Do work"
```

### `include`（引入外部配置）

```yaml
include:
  - local: '/templates/.gitlab-ci-template.yml'
  - project: 'my-group/my-project'
    file: '/templates/.build.yml'
    ref: main
  - remote: 'https://gitlab.com/example/ci-templates/-/raw/master/Docker.yml'
  - template: 'Docker.gitlab-ci.yml'
```

---

## 十、镜像和标签 (`image` / `tags`)

### 指定镜像

```yaml
# 全局
image: node:20

# 作业级
build:
  image: node:20-alpine

# 多服务
build:
  image:
    name: docker:latest
    entrypoint: [""]          # 覆盖 entrypoint
```

### Runner 标签

```yaml
job:
  tags:
    - docker
    - linux
    - gpu
```

---

## 十一、并行和依赖

### 并行矩阵

```yaml
test:
  stage: test
  parallel:
    matrix:
      - NODE_VERSION: ['16', '18', '20']
      - OS: [ubuntu, alpine]
  image: node:$NODE_VERSION
  script:
    - node --version
    - npm test
```

### 作业依赖

```yaml
build:
  stage: build
  script: make build

test:
  stage: test
  needs: ["build"]            # 不等待整个阶段，只看 build
  script: make test

deploy:
  stage: deploy
  needs:
    - job: build
      artifacts: true
    - job: test
      optional: true          # 即使 test 失败也继续
  script: make deploy
```

---

## 十二、重试和超时

```yaml
job:
  retry:
    max: 2
    when:
      - runner_system_failure
      - stuck_or_timeout_failure
  timeout: 1h 30m
```

---

## 十三、触发管道

### 子管道

```yaml
trigger_job:
  trigger:
    include: path/to/child-pipeline.yml
  variables:
    ENV: production
```

### 跨项目管道

```yaml
trigger_job:
  trigger:
    project: my-group/my-project
    branch: main
    strategy: depend          # 等待子管道完成
```

---

## 十四、完整示例

```yaml
stages:
  - build
  - test
  - deploy

variables:
  DOCKER_REGISTRY: $CI_REGISTRY
  DOCKER_IMAGE: $CI_REGISTRY_IMAGE

default:
  image: node:20-alpine
  before_script:
    - npm ci

cache:
  key: $CI_COMMIT_REF_SLUG
  paths:
    - node_modules/
    - .cache/

lint:
  stage: build
  script:
    - npm run lint

build:
  stage: build
  artifacts:
    paths:
      - dist/
  script:
    - npm run build

test:
  stage: test
  dependencies:
    - build
  parallel:
    matrix:
      - NODE_VERSION: ['18', '20']
  image: node:$NODE_VERSION
  script:
    - npm test
  artifacts:
    reports:
      junit: test-results.xml

deploy-staging:
  stage: deploy
  script:
    - echo "Deploying to staging..."
  environment:
    name: staging
    url: https://staging.example.com
  rules:
    - if: $CI_COMMIT_BRANCH == "develop"

deploy-production:
  stage: deploy
  script:
    - echo "Deploying to production..."
  environment:
    name: production
    url: https://example.com
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
  when: manual
```

---

## 十五、常见问题

| 问题 | 解决方案 |
|------|----------|
| 密钥使用 | 在 Settings → CI/CD → Variables 中配置，使用 `$VARIABLE_NAME` |
| 缓存未命中 | 检查 `key` 是否匹配、`paths` 路径是否正确 |
| 产物过期 | 设置 `expire_in` 或手动下载 |
| Runner 不可用 | 检查 `tags` 是否匹配或共享 Runner 是否启用 |
| 管道不触发 | 检查 `rules`/`only`/`except` 配置 |