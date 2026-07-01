# Jenkinsfile 配置文件语法

> 配置文件路径：`Jenkinsfile`（项目根目录）  
> 官方文档：https://www.jenkins.io/doc/book/pipeline/

---

## 一、两种语法风格

| 风格 | 特点 | 适用场景 |
|------|------|----------|
| **Declarative Pipeline** (推荐) | 结构化、固定的 `pipeline { }` 块 | 大多数场景 |
| **Scripted Pipeline** | 灵活、Groovy 脚本风格 | 复杂逻辑、动态生成 |

---

## 二、Declarative Pipeline 结构

```groovy
pipeline {
    agent any                              // 运行节点（必填）

    environment {                          // 环境变量
        APP_NAME = 'my-app'
    }

    options {                              // 全局选项
        timeout(time: 1, unit: 'HOURS')
    }

    parameters {                           // 参数化构建
        string(name: 'VERSION', defaultValue: '1.0.0')
    }

    triggers {                             // 触发器
        cron('H 9 * * 1-5')
    }

    stages {                               // 阶段（必填）
        stage('Build') {
            steps {
                echo 'Building...'
            }
        }
    }

    post {                                 // 后置操作
        always {
            echo 'Cleanup...'
        }
    }
}
```

---

## 三、运行节点 (`agent`)

### 基础用法

```groovy
pipeline {
    agent any                              // 任意可用节点
    // agent none                          // 全局无 agent，每个 stage 自行指定
    // agent { label 'linux && docker' }   // 标签匹配
    // agent { node { label 'linux' } }     // 等同于上面
}
```

### Docker 代理

```groovy
pipeline {
    agent {
        docker {
            image 'node:20-alpine'
            label 'docker-host'
            args '-v /tmp:/tmp'
        }
    }
}
```

### Stage 级代理

```groovy
pipeline {
    agent none
    stages {
        stage('Build') {
            agent { label 'linux' }
            steps { ... }
        }
        stage('Test') {
            agent { docker { image 'node:20' } }
            steps { ... }
        }
    }
}
```

---

## 四、环境变量 (`environment`)

### 全局环境变量

```groovy
pipeline {
    environment {
        NODE_ENV = 'production'
        DOCKER_REGISTRY = 'registry.example.com'
    }
}
```

### Stage 级环境变量

```groovy
stage('Deploy') {
    environment {
        DEPLOY_TARGET = 'staging'
    }
    steps { ... }
}
```

### 动态环境变量

```groovy
environment {
    BUILD_TIME = sh(script: 'date +%s', returnStdout: true).trim()
}
```

### 引用凭证

```groovy
environment {
    DOCKER_USER = credentials('docker-hub-username')
    DOCKER_PASS = credentials('docker-hub-password')
}
```

---

## 五、参数化构建 (`parameters`)

```groovy
pipeline {
    parameters {
        string(name: 'VERSION', defaultValue: '1.0.0', description: '版本号')
        text(name: 'CHANGELOG', defaultValue: '', description: '更新日志')
        booleanParam(name: 'RUN_TESTS', defaultValue: true, description: '是否运行测试')
        choice(name: 'ENVIRONMENT', choices: ['staging', 'production'], description: '部署环境')
        password(name: 'API_KEY', defaultValue: '', description: 'API 密钥')
        file(name: 'CONFIG', description: '上传配置文件')
    }
}
```

---

## 六、选项 (`options`)

```groovy
pipeline {
    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))     // 保留构建数
        disableConcurrentBuilds()                           // 禁止并发
        skipDefaultCheckout()                               // 跳过默认 checkout
        timeout(time: 2, unit: 'HOURS')                     // 超时
        timestamps()                                        // 日志时间戳
        ansiColor('xterm')                                  // 彩色输出
        retry(3)                                            // 重试次数
    }
}
```

---

## 七、触发器 (`triggers`)

```groovy
pipeline {
    triggers {
        cron('H 9 * * 1-5')                    // 定时：工作日 9:00
        // cron('@midnight')                    // 每天午夜
        pollSCM('H/15 * * * *')                 // 轮询 SCM：每 15 分钟
        upstream(upstreamProjects: 'job-a', threshold: hudson.model.Result.SUCCESS)
    }
}
```

---

## 八、阶段 (`stages` / `stage`)

### 基础阶段

```groovy
stages {
    stage('Build') {
        steps {
            sh 'make build'
        }
    }
}
```

### 并行阶段

```groovy
stage('Test') {
    parallel {
        stage('Unit Tests') {
            steps {
                sh 'npm run test:unit'
            }
        }
        stage('Integration Tests') {
            steps {
                sh 'npm run test:integration'
            }
        }
    }
}
```

### 条件阶段 (`when`)

```groovy
stage('Deploy') {
    when {
        branch 'main'                            // 分支匹配
        // expression { return params.RUN_TESTS } // 表达式
        // environment name: 'DEPLOY', value: 'true' // 环境变量
        // allOf { ... }                         // 所有条件
        // anyOf { ... }                         // 任一条件
        // not { ... }                           // 取反
    }
    steps { ... }
}
```

### `when` 内置条件

| 条件 | 说明 |
|------|------|
| `branch` | 分支匹配 |
| `tag` | 标签匹配 |
| `expression` | Groovy 表达式 |
| `environment` | 环境变量匹配 |
| `allOf` | 所有条件满足 |
| `anyOf` | 任一条件满足 |
| `not` | 取反 |
| `equals` | 期望值相等 |
| `changeRequest` | PR/MR 触发 |
| `buildingTag` | 是否为标签构建 |

---

## 九、步骤 (`steps`)

### Shell 命令

```groovy
steps {
    sh 'echo "Hello"'
    sh '''
        echo "Multi-line"
        npm install
        npm test
    '''
    // 返回输出
    script {
        def version = sh(script: 'git describe --tags', returnStdout: true).trim()
    }
}
```

### 目录操作

```groovy
steps {
    dir('subproject') {
        sh 'make build'
    }
}
```

### 工作区清理

```groovy
steps {
    deleteDir()           // 删除整个工作区
    cleanWs()             // 更丰富的清理选项
}
```

### 文件操作

```groovy
steps {
    // 写入文件
    writeFile file: 'config.json', text: '{"key": "value"}'
    // 读取文件
    def config = readFile file: 'config.json'
    // 文件存在检查
    fileExists 'package.json'
}
```

### 错误处理

```groovy
steps {
    script {
        try {
            sh 'make deploy'
        } catch (Exception e) {
            echo "Deploy failed: ${e.message}"
            error("Deployment failed")
        }
    }
}
```

### 重试

```groovy
steps {
    retry(3) {
        sh 'curl -f https://api.example.com'
    }
}
```

### 超时设置

```groovy
steps {
    timeout(time: 5, unit: 'MINUTES') {
        sh 'npm test'
    }
}
```

---

## 十、后置操作 (`post`)

```groovy
pipeline {
    stages { ... }
    post {
        always {
            // 总是执行
            echo 'Cleaning up...'
            junit '**/test-results.xml'
        }
        success {
            // 成功时
            slackSend(color: 'good', message: 'Deploy succeeded!')
        }
        failure {
            // 失败时
            slackSend(color: 'danger', message: 'Deploy failed!')
        }
        unstable {
            // 不稳定时
            echo 'Tests were unstable'
        }
        changed {
            // 状态变化时
            echo 'Build status changed'
        }
        fixed {
            // 修复后
            echo 'Build fixed!'
        }
        regression {
            // 回归时
            echo 'Regressed!'
        }
        aborted {
            // 中止时
            echo 'Build was aborted'
        }
        unsuccessful {
            // 不成功（失败+中止）
            echo 'Not successful'
        }
        cleanup {
            // 总是最后执行（即使 aborted）
            cleanWs()
        }
    }
}
```

---

## 十一、工具和凭证

### 工具声明

```groovy
pipeline {
    tools {
        jdk 'JDK17'
        nodejs 'NodeJS 20'
        maven 'Maven 3.9'
    }
}
```

### 凭证使用

```groovy
pipeline {
    environment {
        // 用户名密码凭证
        GIT_CREDS = credentials('git-credentials')
        // 自动注入: ${GIT_CREDS_USR} / ${GIT_CREDS_PSW}

        // SSH 密钥凭证
        SSH_KEY = credentials('ssh-key')
        // 自动注入: ${SSH_KEY} / ${SSH_KEY_USR} / ${SSH_KEY_PSW} (passphrase)
    }
}
```

---

## 十二、共享库 (`@Library`)

```groovy
// 引入共享库
@Library('my-shared-lib@v1.0') _

pipeline {
    stages {
        stage('Deploy') {
            steps {
                // 调用共享库步骤
                deployToK8s(namespace: 'production')
            }
        }
    }
}
```

---

## 十三、Scripted Pipeline 语法

```groovy
node('linux') {
    stage('Checkout') {
        checkout scm
    }

    stage('Build') {
        try {
            sh 'make build'
        } catch (Exception e) {
            currentBuild.result = 'FAILURE'
            error(e.message)
        }
    }

    stage('Test') {
        def result = sh(script: 'make test', returnStatus: true)
        if (result != 0) {
            unstable('Tests failed')
        }
    }

    stage('Deploy') {
        if (env.BRANCH_NAME == 'main') {
            sh 'make deploy'
        }
    }
}
```

### 声明式 vs 脚本式对比

| 特性 | 声明式 | 脚本式 |
|------|--------|--------|
| 入口 | `pipeline { }` | `node { }` |
| 结构 | 固定的 `stages/steps` | 灵活的 Groovy 代码 |
| 条件 | `when { }` 块 | `if/else` |
| 后置 | `post { }` 块 | `try/catch/finally` |
| 学习曲线 | 低 | 高 |
| 灵活性 | 受限 | 完全自由 |

---

## 十四、常用内置变量

| 变量 | 说明 |
|------|------|
| `env.BUILD_ID` | 构建 ID |
| `env.BUILD_NUMBER` | 构建编号 |
| `env.JOB_NAME` | 作业名称 |
| `env.BRANCH_NAME` | 分支名（Multibranch Pipeline） |
| `env.GIT_COMMIT` | Git 提交 SHA |
| `env.GIT_BRANCH` | Git 分支 |
| `env.WORKSPACE` | 工作区绝对路径 |
| `env.NODE_NAME` | 节点名称 |
| `currentBuild.result` | 构建结果（SUCCESS/FAILURE/UNSTABLE） |
| `currentBuild.currentResult` | 当前结果 |

---

## 十五、完整示例

```groovy
pipeline {
    agent {
        docker {
            image 'node:20-alpine'
            label 'docker'
        }
    }

    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        disableConcurrentBuilds()
        timeout(time: 1, unit: 'HOURS')
    }

    parameters {
        string(name: 'VERSION', defaultValue: 'latest', description: '镜像版本')
        choice(name: 'ENV', choices: ['staging', 'production'], description: '环境')
        booleanParam(name: 'RUN_TESTS', defaultValue: true, description: '运行测试')
    }

    environment {
        DOCKER_REGISTRY = 'registry.example.com'
        DOCKER_IMAGE = "${DOCKER_REGISTRY}/my-app:${params.VERSION}"
    }

    triggers {
        cron('H 9 * * 1-5')
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Install') {
            steps {
                sh 'npm ci'
            }
        }

        stage('Lint') {
            steps {
                sh 'npm run lint'
            }
        }

        stage('Test') {
            when {
                expression { return params.RUN_TESTS }
            }
            parallel {
                stage('Unit') {
                    steps {
                        sh 'npm run test:unit'
                    }
                }
                stage('Integration') {
                    steps {
                        sh 'npm run test:integration'
                    }
                }
            }
            post {
                always {
                    junit '**/test-results.xml'
                }
            }
        }

        stage('Build & Push') {
            when {
                branch 'main'
            }
            steps {
                sh """
                    docker build -t ${DOCKER_IMAGE} .
                    docker push ${DOCKER_IMAGE}
                """
            }
        }

        stage('Deploy') {
            when {
                branch 'main'
            }
            steps {
                script {
                    def env = params.ENV
                    sh "./deploy.sh ${env}"
                }
            }
        }
    }

    post {
        always {
            cleanWs()
        }
        success {
            slackSend(color: 'good', message: "Pipeline succeeded: ${env.JOB_NAME} #${env.BUILD_NUMBER}")
        }
        failure {
            slackSend(color: 'danger', message: "Pipeline failed: ${env.JOB_NAME} #${env.BUILD_NUMBER}")
        }
    }
}
```

---

## 十六、常见问题

| 问题 | 解决方案 |
|------|----------|
| 凭证不生效 | 检查 Jenkins 凭证 ID 是否匹配 |
| 并发构建冲突 | 添加 `disableConcurrentBuilds()` |
| 工作区残留 | 在 `post { cleanup }` 中添加 `cleanWs()` |
| Pipeline 不触发 | 检查 `triggers` 配置，确认 Webhook 已配置 |
| 变量作用域 | 声明式用 `environment`，脚本式用 `env.XXX` |