# CircleCI 配置文件语法

> 配置文件路径：`.circleci/config.yml`  
> 官方文档：https://circleci.com/docs/configuration-reference/

---

## 一、文件结构总览

```yaml
version: 2.1                      # CircleCI 版本（2.1 为最新）

orbs:                             # 可复用 Orb（可选）
  node: circleci/node@5.0.0

executors:                        # 执行器定义（可选）
  my-executor:
    docker:
      - image: cimg/node:20.0

commands:                         # 自定义命令（可选）
  my-command:
    steps:
      - run: echo "hello"

jobs:                             # 作业定义（必填）
  build:
    executor: my-executor
    steps:
      - checkout
      - run: npm test

workflows:                        # 工作流编排（必填）
  main:
    jobs:
      - build
```

---

## 二、执行器 (`executors`)

### Docker 执行器

```yaml
executors:
  node-executor:
    docker:
      - image: cimg/node:20.0
        auth:
          username: $DOCKERHUB_USER
          password: $DOCKERHUB_PASS
    resource_class: medium
    environment:
      NODE_ENV: test

  # 多容器
  integration-executor:
    docker:
      - image: cimg/node:20.0
      - image: cimg/postgres:15.0
        environment:
          POSTGRES_DB: test
        auth:
          username: $DOCKERHUB_USER
          password: $DOCKERHUB_PASS
```

### Machine 执行器

```yaml
executors:
  linux-executor:
    machine:
      image: ubuntu-2204:2024.01.1
    resource_class: medium
```

### macOS 执行器

```yaml
executors:
  mac-executor:
    macos:
      xcode: 15.2.0
    resource_class: macos.m1.medium.gen1
```

### Windows 执行器

```yaml
executors:
  win-executor:
    machine:
      image: windows-server-2022-gui:current
    resource_class: windows.medium
```

### 资源类 (`resource_class`)

| 类型 | 资源类 |
|------|--------|
| Docker | `small` / `medium` / `medium+` / `large` / `xlarge` / `2xlarge` / `2xlarge+` |
| Linux VM | `medium` / `large` / `xlarge` / `2xlarge` / `2xlarge+` |
| macOS | `macos.m1.medium.gen1` / `macos.m1.large.gen1` |
| GPU | `gpu.nvidia.small` / `gpu.nvidia.medium` |
| Arm | `arm.medium` / `arm.large` / `arm.xlarge` / `arm.2xlarge` |

---

## 三、命令 (`commands`)

### 自定义命令

```yaml
commands:
  install_deps:
    description: "Install npm dependencies with cache"
    parameters:
      cache_key:
        type: string
        default: "npm"
    steps:
      - restore_cache:
          keys:
            - << parameters.cache_key >>-{{ checksum "package-lock.json" }}
      - run: npm ci
      - save_cache:
          key: << parameters.cache_key >>-{{ checksum "package-lock.json" }}
          paths:
            - ~/.npm

  notify:
    description: "Send notification"
    parameters:
      status:
        type: string
      message:
        type: string
        default: ""
    steps:
      - run:
          command: echo "Status: << parameters.status >> << parameters.message >>"
          when: always
```

### 参数类型

| 类型 | 示例 |
|------|------|
| `string` | `default: "hello"` |
| `integer` | `default: 10` |
| `boolean` | `default: false` |
| `enum` | `enum: ["staging", "production"]` |
| `steps` | 传入步骤块 |
| `executor` | 传入执行器 |

---

## 四、作业 (`jobs`)

### 基础作业

```yaml
jobs:
  build:
    executor: node-executor
    parallelism: 4                # 并行度
    working_directory: ~/project
    
    environment:
      CI: true
    
    steps:
      - checkout
      - run: npm ci
      - run: npm test
```

### 参数化作业

```yaml
jobs:
  build:
    parameters:
      node-version:
        type: string
        default: "20"
      run-tests:
        type: boolean
        default: true
    
    docker:
      - image: cimg/node:<< parameters.node-version >>
    
    steps:
      - checkout
      - run: npm ci
      - when:
          condition: << parameters.run-tests >>
          steps:
            - run: npm test
```

---

## 五、步骤 (`steps`)

### 基础步骤

```yaml
steps:
  - checkout                     # 检出代码

  - run:                         # 执行命令
      name: "Install dependencies"
      command: npm ci
      working_directory: ./src
      shell: /bin/bash
      environment:
        NODE_ENV: test
      background: false          # 后台运行
      no_output_timeout: 30m     # 无输出超时

  - run:                         # 多行命令
      command: |
        echo "Starting build..."
        npm run build
        echo "Build complete"
      when: always               # on_success / always / on_fail
```

### 条件执行

```yaml
steps:
  - when:                        # 条件块
      condition:
        equal: [main, << pipeline.git.branch >>]
      steps:
        - run: echo "On main branch"
  
  - unless:                      # 反向条件
      condition:
        equal: [staging, << pipeline.parameters.environment >>]
      steps:
        - run: echo "Not staging"
```

### 循环

```yaml
steps:
  - run:
      name: "Test all services"
      command: |
        for service in $(echo "<< parameters.services >>"); do
          npm test -- --service=$service
        done
```

---

## 六、缓存 (`restore_cache` / `save_cache`)

```yaml
steps:
  - restore_cache:
      name: "Restore npm cache"
      keys:
        - npm-v1-{{ checksum "package-lock.json" }}
        - npm-v1-                     # 回退键

  - run: npm ci

  - save_cache:
      name: "Save npm cache"
      key: npm-v1-{{ checksum "package-lock.json" }}
      paths:
        - ~/.npm
        - node_modules
```

---

## 七、工作区 (Workspace) 持久化

```yaml
jobs:
  build:
    steps:
      - checkout
      - run: npm run build
      - persist_to_workspace:       # 持久化
          root: .
          paths:
            - dist
            - package.json

  deploy:
    steps:
      - attach_workspace:            # 挂载
          at: /tmp/workspace
      - run: ls /tmp/workspace/dist
```

---

## 八、产物 (`store_artifacts` / `store_test_results`)

```yaml
steps:
  - run: npm test
  
  - store_test_results:              # 存储测试结果
      path: test-results

  - store_artifacts:                 # 存储产物
      path: dist
      destination: build-output
```

---

## 九、Orbs（可复用包）

### 引入 Orb

```yaml
version: 2.1

orbs:
  node: circleci/node@5.0.0
  docker: circleci/docker@2.0.0
  aws-s3: circleci/aws-s3@4.0.0
  slack: circleci/slack@4.0.0
```

### 使用 Orb 命令

```yaml
jobs:
  build:
    executor: node/default
    steps:
      - checkout
      - node/install-packages
      - node/run:
          script: build
```

### 使用 Orb 作业

```yaml
workflows:
  main:
    jobs:
      - node/test:
          version: "20.0"
          run-command: test
```

---

## 十、工作流 (`workflows`)

### 基础工作流

```yaml
workflows:
  version: 2
  main:
    jobs:
      - build
      - lint
      - test:
          requires:
            - build
      - deploy:
          requires:
            - lint
            - test
          filters:
            branches:
              only: main
```

### 条件触发

```yaml
workflows:
  version: 2
  main:
    triggers:
      - schedule:
          cron: "0 9 * * 1-5"
          filters:
            branches:
              only: main
    
    jobs:
      - build
      - deploy:
          requires:
            - build
          when: << pipeline.parameters.deploy_enabled >>
```

### 定时触发

```yaml
workflows:
  nightly:
    triggers:
      - schedule:
          cron: "0 0 * * *"
          filters:
            branches:
              only: main
    jobs:
      - e2e-tests
```

### 手动批准的继续/暂缓

```yaml
workflows:
  version: 2
  main:
    jobs:
      - build
      - test:
          requires:
            - build
      - hold:
          type: approval            # 手动批准
          requires:
            - test
      - deploy:
          requires:
            - hold
```

### 矩阵作业

```yaml
workflows:
  version: 2
  test:
    jobs:
      - test:
          matrix:
            parameters:
              node-version: ["16.0", "18.0", "20.0"]
              os: ["linux", "macos"]
            exclude:
              - node-version: "16.0"
                os: "macos"
```

### 扇出/扇入

```yaml
workflows:
  version: 2
  main:
    jobs:
      - build
      - lint:
          requires:
            - build
      - unit-test:
          requires:
            - build
      - integration-test:
          requires:
            - build
      - deploy:
          requires:
            - lint
            - unit-test
            - integration-test
```

---

## 十一、动态配置 (`setup: true`)

```yaml
# .circleci/config.yml
version: 2.1

setup: true                        # 启用动态配置

orbs:
  path-filtering: circleci/path-filtering@1.0.0

workflows:
  generate-config:
    jobs:
      - path-filtering/filter:
          base-revision: main
          config-path: .circleci/continue_config.yml
          mapping: |
            src/.* run-build-job true
            docs/.* run-docs-job true
```

---

## 十二、上下文和变量

### 上下文 (`contexts`)

```yaml
workflows:
  main:
    jobs:
      - deploy:
          context:
            - aws-credentials
            - slack-token
```

上下文在 CircleCI 项目设置中创建，包含环境变量组。

### 环境变量

```yaml
# 项目级
environment:
  APP_NAME: my-app

jobs:
  build:
    environment:
      NODE_ENV: test
```

### 管道参数

```yaml
version: 2.1

parameters:
  deploy_enabled:
    type: boolean
    default: false
  environment:
    type: string
    default: "staging"

workflows:
  main:
    when:
      condition: << pipeline.parameters.deploy_enabled >>
    jobs:
      - deploy:
          environment: << pipeline.parameters.environment >>
```

---

## 十三、常用内置变量

| 变量 | 说明 |
|------|------|
| `CIRCLE_SHA1` | 当前提交 SHA |
| `CIRCLE_BRANCH` | 分支名 |
| `CIRCLE_TAG` | Git 标签 |
| `CIRCLE_BUILD_NUM` | 构建编号 |
| `CIRCLE_JOB` | 当前作业名 |
| `CIRCLE_WORKFLOW_ID` | 工作流 ID |
| `CIRCLE_PROJECT_REPONAME` | 仓库名 |
| `CIRCLE_PROJECT_USERNAME` | 组织/用户名 |
| `CIRCLE_NODE_INDEX` | 并行索引 (0-based) |
| `CIRCLE_NODE_TOTAL` | 并行总数 |
| `CIRCLE_WORKING_DIRECTORY` | 工作目录 |
| `CI` | 总是 `true` |

---

## 十四、完整示例

```yaml
version: 2.1

orbs:
  node: circleci/node@5.0.0
  docker: circleci/docker@2.0.0
  slack: circleci/slack@4.0.0

parameters:
  run-deploy:
    type: boolean
    default: false

executors:
  node-exec:
    docker:
      - image: cimg/node:20.0
    resource_class: medium
    working_directory: ~/project

commands:
  setup:
    description: "Install deps and cache"
    steps:
      - restore_cache:
          keys:
            - deps-{{ checksum "package-lock.json" }}
      - run: npm ci
      - save_cache:
          key: deps-{{ checksum "package-lock.json" }}
          paths:
            - ~/.npm

jobs:
  lint:
    executor: node-exec
    steps:
      - checkout
      - setup
      - run: npm run lint

  test:
    executor: node-exec
    parallelism: 3
    steps:
      - checkout
      - setup
      - run:
          command: npm test -- --ci --shard=$CIRCLE_NODE_INDEX/$CIRCLE_NODE_TOTAL
      - store_test_results:
          path: test-results

  build:
    executor: node-exec
    steps:
      - checkout
      - setup
      - run: npm run build
      - persist_to_workspace:
          root: .
          paths:
            - dist
      - store_artifacts:
          path: dist

  docker-build:
    executor: docker/docker
    steps:
      - checkout
      - docker/check
      - docker/build:
          image: my-app
          tag: $CIRCLE_SHA1
      - docker/push:
          image: my-app
          tag: $CIRCLE_SHA1

  deploy:
    executor: node-exec
    steps:
      - attach_workspace:
          at: /tmp/workspace
      - run: |
          echo "Deploying to << pipeline.parameters.environment >>"
          ./deploy.sh

workflows:
  version: 2
  ci:
    jobs:
      - lint
      - test:
          matrix:
            parameters:
              node-version: ["18.0", "20.0", "22.0"]
      - build:
          requires:
            - lint
      - docker-build:
          requires:
            - test
            - build
          filters:
            branches:
              only: main
      - hold:
          type: approval
          requires:
            - docker-build
      - deploy:
          requires:
            - hold
          when: << pipeline.parameters.run-deploy >>
```

---

## 十五、常见问题

| 问题 | 解决方案 |
|------|----------|
| 缓存未命中 | 检查 `checksum` 路径，确保 `package-lock.json` 存在 |
| 工作区挂载失败 | 确保 `persist_to_workspace` 和 `attach_workspace` 路径一致 |
| 并行测试失败 | 使用 `circleci tests split` 分配测试文件 |
| 上下文不可用 | 检查项目设置中是否正确添加了上下文 |
| 作业超时 | 默认 5 小时，可联系支持调整；使用 `no_output_timeout` 控制单步 |