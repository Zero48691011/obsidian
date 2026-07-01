# Argo 项目：是什么、怎么部署、怎么使用

> Argo 是一套 Kubernetes 原生的开源工具集，用于工作流编排、GitOps 持续交付、渐进式发布和事件驱动自动化。  
> 官方文档：https://argo-cd.readthedocs.io / https://argoproj.github.io/workflows

---

## 一、Argo 项目概览

Argo 包含 4 个核心子项目：

| 项目 | 定位 | 一句话 |
|------|------|--------|
| **Argo Workflows** | 容器原生工作流引擎 | 在 K8s 上编排 DAG / 步骤流水线 |
| **Argo CD** | 声明式 GitOps CD 工具 | 让集群状态与 Git 仓库保持同步 |
| **Argo Rollouts** | 渐进式交付控制器 | 蓝绿、金丝雀发布替代 K8s Deployment |
| **Argo Events** | 事件驱动自动化 | 监听外部事件触发 Workflows |

---

## 二、Argo Workflows

### 2.1 是什么

Argo Workflows 是 Kubernetes 上的工作流引擎，每个步骤是一个容器，支持 DAG（有向无环图）、步骤、循环、条件分支。常用于 ML 流水线、CI/CD、数据处理。

### 2.2 部署

```bash
# 安装 Argo Workflows（需要 K8s 集群 + kubectl 已配置）
kubectl create namespace argo
kubectl apply -n argo -f https://github.com/argoproj/argo-workflows/releases/latest/download/quick-start-minimal.yaml

# 安装 CLI
brew install argo  # macOS
# 或
curl -sLO https://github.com/argoproj/argo-workflows/releases/latest/download/argo-linux-amd64.gz
gunzip argo-linux-amd64.gz && chmod +x argo-linux-amd64 && sudo mv argo-linux-amd64 /usr/local/bin/argo
```

### 2.3 配置访问

```bash
# 端口转发访问 UI
kubectl -n argo port-forward deployment/argo-server 2746:2746

# 或配置 Ingress / NodePort
# 认证：默认使用 SSO 或 Bearer Token
# 获取 Token（用于本地测试）
kubectl -n argo create token argo-server
```

### 2.4 使用

**Hello World 工作流：**

```yaml
# hello-world.yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: hello-world-
spec:
  entrypoint: whalesay
  templates:
    - name: whalesay
      container:
        image: docker/whalesay:latest
        command: [cowsay]
        args: ["Hello Argo!"]
```

```bash
argo submit hello-world.yaml
argo list          # 查看工作流
argo get @latest   # 查看最新工作流详情
argo logs @latest  # 查看日志
```

**DAG 工作流：**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: dag-demo-
spec:
  entrypoint: main
  templates:
    - name: main
      dag:
        tasks:
          - name: A
            template: echo
            arguments:
              parameters: [{name: msg, value: "Task A"}]
          - name: B
            dependencies: [A]
            template: echo
            arguments:
              parameters: [{name: msg, value: "Task B"}]
          - name: C
            dependencies: [A]
            template: echo
            arguments:
              parameters: [{name: msg, value: "Task C"}]
          - name: D
            dependencies: [B, C]
            template: echo
            arguments:
              parameters: [{name: msg, value: "Task D"}]

    - name: echo
      inputs:
        parameters:
          - name: msg
      container:
        image: alpine:latest
        command: [echo]
        args: ["{{inputs.parameters.msg}}"]
```

**步骤工作流（Steps）：**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: steps-
spec:
  entrypoint: main
  templates:
    - name: main
      steps:
        - - name: build
            template: build
        - - name: test
            template: test
        - - name: deploy
            template: deploy
            when: "{{steps.test.outputs.result}} == success"

    - name: build
      container:
        image: golang:1.21
        command: [sh, -c]
        args: ["echo building... && sleep 2"]

    - name: test
      container:
        image: golang:1.21
        command: [sh, -c]
        args: ["echo testing... && sleep 2"]

    - name: deploy
      container:
        image: alpine:latest
        command: [echo]
        args: ["deploying..."]
```

**参数和输出：**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: parameters-
spec:
  entrypoint: main
  arguments:
    parameters:
      - name: message
        value: "Hello Argo"
  templates:
    - name: main
      inputs:
        parameters:
          - name: message
      container:
        image: alpine:latest
        command: [echo]
        args: ["{{inputs.parameters.message}}"]
```

**CronWorkflow（定时任务）：**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: CronWorkflow
metadata:
  name: daily-backup
spec:
  schedule: "0 2 * * *"         # 每天凌晨 2 点
  concurrencyPolicy: Replace     # Allow / Forbid / Replace
  startingDeadlineSeconds: 300
  workflowSpec:
    entrypoint: backup
    templates:
      - name: backup
        container:
          image: alpine:latest
          command: [sh, -c]
          args: ["echo 'Running backup...'"]
```

### 2.5 常用命令

```bash
argo submit workflow.yaml              # 提交工作流
argo submit --watch workflow.yaml      # 提交并等待完成
argo submit -p key=value workflow.yaml # 传入参数
argo list                              # 列出工作流
argo get @latest                       # 获取详情
argo logs @latest                      # 查看日志
argo delete @latest                    # 删除工作流
argo cron list                         # 列出定时任务
argo cron create cronworkflow.yaml     # 创建定时任务
```

---

## 三、Argo CD

### 3.1 是什么

Argo CD 是 GitOps 持续交付工具。你在 Git 仓库中声明应用的目标状态，Argo CD 自动将 K8s 集群同步到该状态。核心概念：**Git 是唯一真相来源**。

### 3.2 部署

```bash
# 创建命名空间
kubectl create namespace argocd

# 安装（非 HA 版本）
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 安装 CLI
brew install argocd  # macOS
# 或
curl -sSL -o argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 argocd /usr/local/bin/argocd
```

### 3.3 初次访问

```bash
# 端口转发
kubectl port-forward svc/argocd-server -n argocd 8080:443

# 获取初始密码
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

# 登录 CLI
argocd login localhost:8080 --username admin --password <password> --insecure

# 修改密码
argocd account update-password
```

### 3.4 使用

**创建应用（CLI）：**

```bash
# 通过 CLI 创建应用
argocd app create my-app \
  --repo https://github.com/myorg/my-app.git \
  --path manifests \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default \
  --sync-policy automated \
  --auto-prune

# 查看应用
argocd app list
argocd app get my-app

# 手动同步
argocd app sync my-app

# 回滚
argocd app rollback my-app
```

**通过 Application CR 创建：**

```yaml
# application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/my-app.git
    targetRevision: HEAD
    path: manifests
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true        # 自动删除 Git 中不存在的资源
      selfHeal: true     # 自动修复手动变更
    syncOptions:
      - CreateNamespace=true
```

```bash
kubectl apply -f application.yaml
```

### 3.5 App of Apps 模式（多应用管理）

```yaml
# app-of-apps.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: app-of-apps
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/argocd-apps.git
    targetRevision: HEAD
    path: apps
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

### 3.6 多集群管理

```bash
# 注册外部集群
argocd cluster add <context-name>

# 列出已注册集群
argocd cluster list
```

### 3.7 通知和 Webhook

```yaml
# 配置通知（通过 argocd-notifications）
# 在 argocd-notifications-cm ConfigMap 中配置
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
data:
  service.slack: |
    token: $slack-token
  trigger.on-sync-status-changed: |
    - when: app.status.operationState.phase in ['Succeeded', 'Failed']
      send: [slack-notification]
  template.slack-notification: |
    message: "App {{.app.metadata.name}} sync {{.app.status.operationState.phase}}"
```

---

## 四、Argo Rollouts

### 4.1 是什么

Argo Rollouts 是 Kubernetes 渐进式交付控制器，提供蓝绿部署和金丝雀发布，比原生 Deployment 更强大。

### 4.2 部署

```bash
kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml

# 安装 kubectl 插件
brew install argoproj/tap/kubectl-argo-rollouts  # macOS
# 或
curl -LO https://github.com/argoproj/argo-rollouts/releases/latest/download/kubectl-argo-rollouts-linux-amd64
chmod +x kubectl-argo-rollouts-linux-amd64
sudo mv kubectl-argo-rollouts-linux-amd64 /usr/local/bin/kubectl-argo-rollouts
```

### 4.3 使用

**蓝绿部署：**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: my-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: app
          image: myapp:v1
          ports:
            - containerPort: 8080
  strategy:
    blueGreen:
      activeService: my-app-active    # 生产流量
      previewService: my-app-preview  # 预览流量
      autoPromotionEnabled: false     # 需要手动 promote
```

```bash
# 更新镜像
kubectl argo rollouts set image my-app app=myapp:v2

# 查看状态
kubectl argo rollouts get rollout my-app --watch

# 手动 promote
kubectl argo rollouts promote my-app

# 中止
kubectl argo rollouts abort my-app
```

**金丝雀发布：**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: my-app
spec:
  replicas: 5
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: app
          image: myapp:v1
  strategy:
    canary:
      steps:
        - setWeight: 20           # 20% 流量到新版本
        - pause: {duration: 60s}  # 暂停 60s
        - setWeight: 50
        - pause: {duration: 60s}
        - setWeight: 100          # 全部流量到新版本
      analysis:                   # 集成分析
        templates:
          - templateName: success-rate
```

**金丝雀 + 自动分析：**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
spec:
  metrics:
    - name: success-rate
      interval: 30s
      successCondition: result[0] >= 0.95
      failureLimit: 3
      provider:
        prometheus:
          address: http://prometheus:9090
          query: |
            sum(rate(http_requests_total{status!~"5.."}[1m])) /
            sum(rate(http_requests_total[1m]))
```

### 4.4 常用命令

```bash
kubectl argo rollouts get rollout my-app          # 查看状态
kubectl argo rollouts get rollout my-app --watch  # 实时监控
kubectl argo rollouts promote my-app              # 手动 promote
kubectl argo rollouts abort my-app                # 中止
kubectl argo rollouts retry rollout my-app        # 重试
kubectl argo rollouts dashboard                   # 启动 UI 仪表盘
```

---

## 五、Argo Events

### 5.1 是什么

Argo Events 是事件驱动自动化框架，监听外部事件（Webhook、S3、Kafka、GitHub 等），触发 Argo Workflows 或其他 K8s 资源。

### 5.2 部署

```bash
kubectl create namespace argo-events
kubectl apply -n argo-events -f https://raw.githubusercontent.com/argoproj/argo-events/stable/manifests/install.yaml
kubectl apply -n argo-events -f https://raw.githubusercontent.com/argoproj/argo-events/stable/manifests/install-validating-webhook.yaml
```

### 5.3 核心概念

| 组件 | 说明 |
|------|------|
| **EventSource** | 事件源定义（Webhook、S3、Kafka、GitHub 等） |
| **Sensor** | 监听事件 + 触发器（触发 Workflow / K8s 资源） |
| **EventBus** | 事件总线，EventSource → Sensor 的消息通道 |

### 5.4 使用

**Webhook → Workflow 示例：**

```yaml
# 1. 定义 EventBus
apiVersion: argoproj.io/v1alpha1
kind: EventBus
metadata:
  name: default
  namespace: argo-events
spec:
  nats:
    native:
      replicas: 3

---
# 2. 定义 EventSource（Webhook）
apiVersion: argoproj.io/v1alpha1
kind: EventSource
metadata:
  name: webhook
  namespace: argo-events
spec:
  service:
    ports:
      - port: 12000
        targetPort: 12000
  webhook:
    deploy:
      endpoint: /deploy
      method: POST
      port: "12000"

---
# 3. 定义 Sensor（触发 Workflow）
apiVersion: argoproj.io/v1alpha1
kind: Sensor
metadata:
  name: webhook-sensor
  namespace: argo-events
spec:
  dependencies:
    - name: deploy-hook
      eventSourceName: webhook
      eventName: deploy
  triggers:
    - template:
        name: deploy-trigger
        argoWorkflow:
          operation: submit
          source:
            resource:
              apiVersion: argoproj.io/v1alpha1
              kind: Workflow
              metadata:
                generateName: deploy-
              spec:
                entrypoint: deploy
                templates:
                  - name: deploy
                    container:
                      image: alpine:latest
                      command: [echo]
                      args: ["Deploy triggered by webhook!"]
```

---

## 六、Argo 项目对比总结

| 特性 | Workflows | CD | Rollouts | Events |
|------|-----------|----|----------|--------|
| 核心用途 | 工作流编排 | GitOps 持续交付 | 渐进式发布 | 事件驱动 |
| 声明式 | ✅ | ✅ | ✅ | ✅ |
| UI 仪表盘 | ✅ | ✅ | ✅ | ❌ |
| CLI | argo | argocd | kubectl plugin | ❌ |
| CRD 管理 | Workflow | Application | Rollout | EventSource/Sensor |
| 典型场景 | ML Pipeline, CI | 微服务 CD | 金丝雀/蓝绿 | Webhook 触发 |

---

## 七、生产环境建议

| 项目 | 建议 |
|------|------|
| **Argo CD** | 启用 SSO、RBAC、配置 Git Webhook 自动同步；使用 HA 部署 |
| **Argo Workflows** | 配置 artifact repository（S3/MinIO）；设置资源限制和 TTL |
| **Argo Rollouts** | 配合 Service Mesh（Istio/Linkerd）做流量管理；配置 Analysis 自动回滚 |
| **Argo Events** | 事件总线使用 NATS；配置 Sensor 的 error handling 和重试策略 |

---

## 八、快速上手路径

```
1. 先装 Argo CD → 理解 GitOps 同步模型
2. 再装 Argo Workflows → 理解容器原生工作流
3. 需要金丝雀发布 → 接入 Argo Rollouts
4. 需要外部事件触发 → 接入 Argo Events
```

---

## 九、常见问题

| 问题 | 解决方案 |
|------|----------|
| Argo CD 不同步 | 检查 `argocd app get` 的 sync status，查看 `argocd app manifests` 对比差异 |
| Workflow 卡在 Pending | 检查资源配额、镜像拉取、节点资源 |
| Rollout 不推进 | 检查 Analysis 指标、pause 条件是否满足 |
| EventSource 无法访问 | 检查 Service/Ingress 配置，确认端口映射正确 |
| 证书问题 | 配置 `--insecure` 或使用自有 CA 证书 |