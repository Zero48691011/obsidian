# Istio 服务网格详解

---

## 一、官方定义

> **Istio** 是一个**开源的服务网格（Service Mesh）**，它透明地注入到现有的分布式应用中，提供**流量管理、安全、可观测性**等功能，而不需要修改应用代码。
>
> —— [istio.io](https://istio.io/)

拆开来看：

| 关键词 | 含义 |
|--------|------|
| **开源** | 由 Google、IBM、Lyft 联合发起，现由 CNCF 孵化 |
| **服务网格** | 在服务之间插入一层**网络代理**，接管所有通信 |
| **透明注入** | 不需要改代码，Sidecar 代理自动「贴」到每个 Pod 旁边 |
| **流量管理** | 灰度发布、A/B 测试、超时重试、熔断限流 |
| **安全** | 服务间 mTLS 加密通信、细粒度访问控制 |
| **可观测性** | 自动收集指标、链路追踪、访问日志 |

### 名字由来

```
Istio = 希腊语 ἱστίον（帆）

寓意：像帆一样，让微服务这艘船在云原生的大海上航行得更稳、更远。
      K8s 是「舵手」(Kubernetes)，Istio 是「帆」—— 一个管航向，一个借风力。
```

---

## 二、通俗解释

### 一句话版本

> **Istio 就是微服务的「智能网络层」** — 它在每个服务旁边放一个代理，所有流量都经过代理，于是你可以在**不改代码**的情况下实现灰度发布、限流、加密、监控。

### 类比：公寓楼的管家

想象你住在一栋公寓楼里，每个房间（服务）之间要互相送东西：

| 场景 | 没有 Istio | 有了 Istio |
|------|-----------|-----------|
| **送快递** | 自己去敲邻居门（直连） | 管家统一收发，登记谁送了谁收了 |
| **访客管理** | 谁都能进（无安全） | 管家检查身份证（mTLS） |
| **电梯调度** | 大家都挤 1 号电梯 | 管家分流：A 走 1 号，B 走 2 号（流量路由） |
| **监控** | 不知道谁在干嘛 | 管家记录所有进出（可观测性） |
| **装修期间** | 整层楼停用 | 管家先开新房间，好了再切过去（灰度发布） |

### 核心问题：没有 Istio 会怎样？

```
场景：你有一个微服务架构，20+ 个服务互相调用

没有 Istio：
  1. 灰度发布 → 改代码，引入 Spring Cloud / Netflix OSS
  2. 服务间加密 → 每个服务各自配置 TLS 证书
  3. 链路追踪 → 每个服务手动集成 Jaeger/Zipkin SDK
  4. 限流熔断 → 每个语言/框架单独实现（Java 用 Sentinel，Go 用 Hystrix-go...）
  5. 流量监控 → 各服务日志格式不统一，拼不出来全局视图
  6. 金丝雀发布 → 改 Nginx/Ingress 规则，手动切流量

有了 Istio：
  1. 灰度发布 → kubectl apply 一个 VirtualService，按权重分流
  2. 服务间加密 → 一行配置开启 mTLS，证书自动轮换
  3. 链路追踪 → Sidecar 自动注入 trace header，零代码
  4. 限流熔断 → DestinationRule 里配，不区分语言
  5. 流量监控 → Kiali 面板，一眼看全局调用拓扑
  6. 金丝雀发布 → VirtualService 把 5% 流量切到新版本
```

---

## 三、Istio 架构

### 3.1 整体架构（Sidecar 模式）

```
┌──────────────────────────────────────────────────────────────┐
│                        K8s 集群                               │
│                                                              │
│  ┌────────────────── 控制平面 (Control Plane) ──────────────┐ │
│  │  istiod (单体)                                           │ │
│  │  ├─ Pilot      → 流量规则下发、服务发现                    │ │
│  │  ├─ Citadel    → 证书管理、mTLS                          │ │
│  │  └─ Galley     → 配置校验、转发                           │ │
│  └──────────────────────────────────────────────────────────┘ │
│                          │ xDS 协议                            │
│                          ▼                                    │
│  ┌────────────────── 数据平面 (Data Plane) ─────────────────┐ │
│  │                                                          │ │
│  │  ┌──────────┐    ┌──────────┐    ┌──────────┐           │ │
│  │  │  Pod A   │    │  Pod B   │    │  Pod C   │           │ │
│  │  │ ┌──────┐ │    │ ┌──────┐ │    │ ┌──────┐ │           │ │
│  │  │ │ app  │ │    │ │ app  │ │    │ │ app  │ │           │ │
│  │  │ │ 容器 │ │    │ │ 容器 │ │    │ │ 容器 │ │           │ │
│  │  │ └──┬───┘ │    │ └──┬───┘ │    │ └──┬───┘ │           │ │
│  │  │    │     │    │    │     │    │    │     │           │ │
│  │  │ ┌──┴───┐ │    │ ┌──┴───┐ │    │ ┌──┴───┐ │           │ │
│  │  │ │Envoy │◄┼────┼─┤Envoy ├─┼────┼─┤Envoy │ │           │ │
│  │  │ │代理  │ │    │ │代理  │ │    │ │代理  │ │           │ │
│  │  │ └──────┘ │    │ └──────┘ │    │ └──────┘ │           │ │
│  │  └──────────┘    └──────────┘    └──────────┘           │ │
│  │      ↑ 所有流量都经过 Envoy Sidecar ↑                    │ │
│  └──────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────┘
```

### 3.2 核心组件

| 组件 | 所属平面 | 作用 |
|------|---------|------|
| **Envoy** | 数据平面 | 高性能 C++ 代理，拦截所有进出 Pod 的流量，执行路由/限流/加密 |
| **istiod** | 控制平面 | 单体进程，合并了 Pilot + Citadel + Galley，负责下发配置、签发证书 |
| **Ingress Gateway** | 入口 | 集群入口流量网关，代替传统 Nginx Ingress |
| **Egress Gateway** | 出口 | 管控集群对外部服务的访问 |

### 3.3 数据流示例

```
用户请求 → Ingress Gateway → Envoy(A) → app-A → Envoy(A) → Envoy(B) → app-B
                                                              ↑
                                                         mTLS 加密通信
```

每跳都是 Envoy 代理在转发，Istio 控制面只负责「告诉每个 Envoy 该怎么转发」，不经过实际流量。

---

## 四、核心 CRD（自定义资源）

Istio 通过 K8s CRD 扩展了 API，以下是最核心的 5 个：

### 4.1 VirtualService（虚拟服务）— 流量路由规则

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: reviews-route
spec:
  hosts:
  - reviews           # 对哪个服务的流量生效
  http:
  - match:
    - headers:
        end-user:
          exact: jason   # 特定用户
    route:
    - destination:
        host: reviews
        subset: v2        # 路由到 v2 版本
  - route:                # 默认路由
    - destination:
        host: reviews
        subset: v1
      weight: 90          # 90% 流量到 v1
    - destination:
        host: reviews
        subset: v2
      weight: 10          # 10% 流量到 v2（金丝雀发布）
```

### 4.2 DestinationRule（目标规则）— 流量策略

```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: reviews-destination
spec:
  host: reviews
  subsets:                    # 定义版本子集
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100   # 最大连接数
    loadBalancer:
      simple: ROUND_ROBIN     # 负载均衡策略
    outlierDetection:         # 熔断（异常检测）
      consecutiveErrors: 5
      interval: 30s
      baseEjectionTime: 60s
```

### 4.3 Gateway（网关）— 入口流量

```yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: bookinfo-gateway
spec:
  selector:
    istio: ingressgateway   # 绑定到 Ingress Gateway Pod
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "bookinfo.example.com"
```

### 4.4 PeerAuthentication（对等认证）— mTLS

```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: istio-system
spec:
  mtls:
    mode: STRICT    # 强制所有服务间通信使用 mTLS
```

### 4.5 AuthorizationPolicy（授权策略）— 访问控制

```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: deny-all
  namespace: default
spec:
  {}     # 空规则 = 拒绝所有（默认是允许所有）
```

### 5 个 CRD 的关系

```
Gateway (入口)  →  VirtualService (路由规则)  →  DestinationRule (策略)
                        ↑                              │
                        │    ServiceEntry (外部服务)    │
                        └──────────────────────────────┘

PeerAuthentication (mTLS)  +  AuthorizationPolicy (权限) = 安全层
```

---

## 五、核心功能详解

### 5.1 流量管理

| 功能 | 说明 | 实现方式 |
|------|------|---------|
| **金丝雀发布** | 新版本只接 5% 流量，慢慢加 | VirtualService 的 weight |
| **A/B 测试** | 按 Header/Cookie 分流 | VirtualService 的 match |
| **超时** | 请求超时直接返回，不雪崩 | VirtualService 的 timeout |
| **重试** | 失败自动重试 N 次 | VirtualService 的 retries |
| **熔断** | 下游挂了，快速失败 | DestinationRule 的 outlierDetection |
| **限流** | 限制每秒请求数 | EnvoyFilter 或 RateLimit 服务 |
| **故障注入** | 故意注入延迟/错误，测试容错 | VirtualService 的 fault |

### 5.2 安全

```
默认零信任模型：
  ┌──────────────────────────────────────┐
  │  服务 A  →  Envoy(A)  ──mTLS──▶  Envoy(B)  →  服务 B   │
  │              │                        │                │
  │              ├─ 证书自动签发           ├─ 证书自动验证    │
  │              ├─ 加密通信               ├─ 身份认证       │
  │              └─ 授权检查               └─ 授权检查       │
  └──────────────────────────────────────┘
```

| 能力 | 说明 |
|------|------|
| **mTLS** | 双向 TLS，服务间通信加密 + 身份互认 |
| **证书管理** | Citadel 自动签发、轮换证书，无需人工干预 |
| **RBAC** | 基于 AuthorizationPolicy，精确到 HTTP 方法 + URL 路径 |
| **JWT 认证** | 支持 JWT Token 验证（RequestAuthentication） |

### 5.3 可观测性

| 维度 | 工具 | 内容 |
|------|------|------|
| **指标 (Metrics)** | Prometheus + Grafana | QPS、延迟、错误率，Sidecar 自动采集 |
| **链路追踪 (Tracing)** | Jaeger / Zipkin | 请求在服务间的完整链路，自动注入 trace header |
| **网格可视化** | Kiali | 服务拓扑图、流量动画、健康状态 |
| **访问日志** | Envoy 日志 | 每个请求的详细信息（来源、目标、耗时、状态码） |

---

## 六、Istio 不是什么

| 误解 | 实际情况 |
|------|----------|
| 「Istio 是 API 网关」 | ⚠️ 部分正确。Istio 有 Ingress Gateway，但它的核心是**服务间（东西向）流量管理**，而 API 网关侧重**南北向**（外部到内部） |
| 「Istio 替代 K8s Service」 | ❌ 不是替代，是增强。K8s Service 做基础负载均衡，Istio 在此基础上加智能路由 |
| 「Istio 必须用 K8s」 | ⚠️ 主要运行在 K8s 上，但也支持虚拟机（VM）接入网格 |
| 「Istio 就是 Envoy」 | ❌ Envoy 是数据面代理，Istio 是控制面 + 数据面 + 生态的完整方案 |
| 「小项目也该用 Istio」 | ❌ 服务 < 5 个的项目，Istio 的复杂度远超收益，K8s Service + Ingress 就够 |

---

## 七、与其他方案对比

| 维度 | Istio | Linkerd | Consul Connect | 无网格（K8s 原生） |
|------|-------|---------|---------------|-------------------|
| **数据面代理** | Envoy (C++) | linkerd2-proxy (Rust) | Envoy 或内置 | 无 |
| **资源消耗** | 较高（每 Pod ~100MB） | 较低（每 Pod ~10MB） | 中等 | 无额外消耗 |
| **功能丰富度** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ | ⭐ |
| **复杂度** | 高 | 低 | 中 | 无 |
| **mTLS** | ✅ 自动 | ✅ 默认开启 | ✅ | ❌ 需手动 |
| **流量路由** | ✅ 极强（权重/Header/超时/重试） | ✅ 基本 | ✅ 基本 | ❌ 只有基础轮询 |
| **可观测性** | ✅ 完整（Prometheus + Jaeger + Kiali） | ✅ 自带 Dashboard | ✅ | ❌ 分散 |
| **适用场景** | 大型微服务，需要精细化流量管理 | 中小型，追求简单低消耗 | 已用 Consul 的 HashiCorp 生态 | 简单应用，服务少 |

---

## 八、Quick Start（最小化安装）

```bash
# 1. 下载 Istio
curl -L https://istio.io/downloadIstio | sh -
cd istio-*/

# 2. 安装（使用默认 profile）
istioctl install --set profile=demo -y

# 3. 启用 Sidecar 自动注入
kubectl label namespace default istio-injection=enabled

# 4. 部署示例应用
kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml

# 5. 暴露入口
kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml

# 6. 访问
export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway \
  -o jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}')
curl http://localhost:$INGRESS_PORT/productpage
```

---

## 九、总结

| 维度 | 一句话 |
|------|--------|
| **官方定义** | 开源服务网格，统一管理微服务间的流量、安全、可观测性 |
| **通俗定义** | 微服务的「智能网络层」，Sidecar 代理接管所有通信，无需改代码 |
| **核心价值** | 透明注入、流量管理、零信任安全、一键可观测 |
| **三大支柱** | 流量管理（VirtualService + DestinationRule）、安全（mTLS + AuthorizationPolicy）、可观测性（Kiali + Jaeger + Prometheus） |
| **适用场景** | 多语言微服务、需要灰度发布、需要服务间加密、需要统一流量治理 |
| **不适用场景** | 小型项目（< 5 个服务）、单体应用、对性能极度敏感的场景 |
| **与 K8s 关系** | K8s 管容器的「生老病死」，Istio 管服务间的「通信规则」— 互补，不是替代 |

---

## 十、生产实战：电商平台微服务案例

### 场景设定

某电商平台由 5 个微服务组成，部署在 K8s 集群中：

```
用户浏览器
    │
    ▼
┌──────────┐     ┌──────────────┐     ┌──────────────┐
│ frontend │────▶│ order-service│────▶│payment-service│
│  (前端)   │     │   (订单服务)  │     │   (支付服务)   │
└──────────┘     └──────┬───────┘     └──────────────┘
                        │
                        │         ┌──────────────────┐
                        ├────────▶│inventory-service │
                        │         │    (库存服务)      │
                        │         └──────────────────┘
                        │
                        │         ┌──────────────────┐
                        └────────▶│notification-svc  │
                                  │    (通知服务)      │
                                  └──────────────────┘
```

该集群已安装 Istio，所有 namespace 已开启 Sidecar 自动注入。

---

### 10.1 流量管理实战

#### 场景 A：金丝雀发布 — order-service 上线 v2 版本

**背景**：order-service 重构了订单查询逻辑，新版本 v2 需要逐步上线，避免一次性全量导致故障。

**Step 1：定义版本子集（DestinationRule）**

```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: order-service-dest
  namespace: prod
spec:
  host: order-service.prod.svc.cluster.local
  subsets:
  - name: v1
    labels:
      version: v1        # 当前稳定版本
  - name: v2
    labels:
      version: v2        # 新版本
```

**Step 2：配置流量权重（VirtualService）**

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: order-service-vs
  namespace: prod
spec:
  hosts:
  - order-service.prod.svc.cluster.local
  http:
  - match:
    - headers:
        x-canary:
          exact: "true"         # 内部测试人员带此 Header 直接打到 v2
    route:
    - destination:
        host: order-service.prod.svc.cluster.local
        subset: v2
  - route:                      # 普通用户流量
    - destination:
        host: order-service.prod.svc.cluster.local
        subset: v1
      weight: 95                # 95% → v1
    - destination:
        host: order-service.prod.svc.cluster.local
        subset: v2
      weight: 5                 # 5% → v2（金丝雀）
```

**上线过程**：观察 v2 的 5% 流量 → 无异常 → 改 weight 为 50:50 → 再观察 → 100% 切到 v2 → 下线 v1。

**没有 Istio 时**：需要改代码/用 Nginx 多层代理切流量，出问题只能手动回滚。

---

#### 场景 B：超时 + 重试 — payment-service 调用第三方支付网关

**背景**：payment-service 调用外部银行接口，偶尔超时，需要自动重试，但不能无限重试把下游打垮。

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: payment-service-vs
  namespace: prod
spec:
  hosts:
  - payment-service.prod.svc.cluster.local
  http:
  - route:
    - destination:
        host: payment-service.prod.svc.cluster.local
        subset: v1
    timeout: 5s          # 超过 5 秒直接返回超时，不让上游一直等
    retries:
      attempts: 2        # 最多重试 2 次
      perTryTimeout: 3s  # 每次重试超时 3 秒
      retryOn: 5xx,gateway-error,connect-failure  # 这些情况才重试
```

**效果**：用户下单时如果支付接口卡住，5 秒超时 + 最多 2 次重试，最坏情况 5 + 3 + 3 = 11 秒内一定有结果（成功或失败），前端不会一直转圈。

**没有 Istio 时**：每个语言各自实现重试逻辑（Java 用 Resilience4j，Go 手写 retry loop），参数散落各处，全局改超时时间要改 N 个服务。

---

#### 场景 C：熔断 — inventory-service 数据库挂了，快速失败

**背景**：库存服务依赖 MySQL，某天 MySQL 主库宕机，inventory-service 开始大量返回 500。如果不熔断，order-service 的请求会堆积，线程池耗尽，整个下单链路雪崩。

```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: inventory-service-dr
  namespace: prod
spec:
  host: inventory-service.prod.svc.cluster.local
  trafficPolicy:
    outlierDetection:           # 异常检测（熔断）
      consecutive5xxErrors: 5   # 连续 5 次 5xx 错误
      interval: 10s             # 每 10 秒检测一次
      baseEjectionTime: 30s     # 熔断后 30 秒内不再发请求给它
      maxEjectionPercent: 50    # 最多熔断 50% 的 Pod
```

**效果**：inventory-service 连续 5 次 5xx 后，Istio 自动把它踢出负载均衡池。30 秒后试探性地放一个请求过去，如果恢复了就重新加入，没好就继续踢。

**没有 Istio 时**：需要每个调用方都集成熔断库（Hystrix / Sentinel / Resilience4j），Go/Python/Java 各写一套，参数还不统一。

---

### 10.2 安全实战

#### 场景 D：全局 mTLS — 所有服务间通信加密

**背景**：合规要求，所有微服务间通信必须加密，不能明文传输订单数据、用户信息。

```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: mtls-strict
  namespace: prod
spec:
  mtls:
    mode: STRICT        # 强制 mTLS
```

**效果**：一条命令，prod 命名空间内所有服务间通信自动升级为 mTLS 加密。Istio 自动签发证书、自动轮换（默认 24 小时），无需任何应用代码改动。

**Enovy 代理间通信示例**：

```
order-service Pod                       payment-service Pod
┌──────────────┐                       ┌──────────────┐
│  app 容器     │                       │  app 容器     │
│  (HTTP 明文)  │                       │  (HTTP 明文)  │
└──────┬───────┘                       └──────▲───────┘
       │ localhost:8080                       │ localhost:8080
┌──────┴───────┐                       ┌──────┴───────┐
│ Envoy Sidecar│ ◀══ mTLS 加密 ═══▶   │ Envoy Sidecar│
└──────────────┘                       └──────────────┘
       应用之间是明文                          应用之间是明文
       代理之间是密文                          代理之间是密文
```

**没有 Istio 时**：每个服务各自配置 TLS 证书，证书过期了没人知道，半夜服务突然调不通。

---

#### 场景 E：细粒度授权 — 只允许 frontend 调用 order-service 的创建订单接口

**背景**：安全团队要求：只有 frontend 能调用 order-service 的 `POST /api/orders`（创建订单），其他服务（如后台管理系统）不能直接调订单创建接口。

```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: order-service-authz
  namespace: prod
spec:
  selector:
    matchLabels:
      app: order-service
  action: ALLOW
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/prod/sa/frontend"]  # 只有 frontend 的 ServiceAccount
    to:
    - operation:
        methods: ["POST"]
        paths: ["/api/orders"]        # 精确到接口
    - operation:
        methods: ["GET"]
        paths: ["/api/orders/*"]      # 查询订单可以是 GET
  - from:
    - source:
        principals: ["cluster.local/ns/prod/sa/order-service"]  # 自己调自己
    to:
    - operation:
        methods: ["GET", "POST"]
        paths: ["/*"]
```

**效果**：后台管理系统（admin-service）试图直接调 `POST /api/orders` → 被 Envoy 拦截，返回 403 Forbidden。**授权在 Sidecar 层完成，恶意请求连应用进程都碰不到**。

---

### 10.3 可观测性实战

#### 场景 F：Kiali 拓扑图 — 一眼定位故障

**背景**：运维收到告警「下单成功率下降 30%」，打开 Kiali Dashboard：

```
┌─────────────────────────────────────────────────────────┐
│  Kiali — Service Mesh 拓扑图                              │
│                                                         │
│       ┌──────────┐    HTTP/2.0     ┌──────────────┐    │
│       │ frontend │───────▶────────│order-service │    │
│       │  ✅ 正常  │  0.5% err      │   ⚠️ 黄色    │    │
│       └──────────┘                └──┬───┬───┬───┘    │
│                                      │   │   │         │
│                           ┌──────────┘   │   └──────┐  │
│                           ▼               ▼           ▼  │
│                  ┌──────────────┐ ┌──────────────┐ ┌───┐ │
│                  │payment-svc   │ │inventory-svc │ │...│ │
│                  │   ✅ 正常     │ │  ❌ 红色      │ │   │ │
│                  └──────────────┘ │ 错误率 85%    │ └───┘ │
│                                   └──────────────┘       │
│                                                         │
│  点击 inventory-service → 看到详细指标：                   │
│  QPS: 1,200 → 340 (骤降)                                │
│  P99 延迟: 50ms → 8,000ms (暴涨)                         │
│  错误率: 0.1% → 85%                                     │
│                                                         │
│  → 点击链路追踪，跳转到 Jaeger：                           │
│                                                         │
│  frontend → order-service → inventory-service            │
│  总耗时 8.2s                                              │
│    ├─ frontend: 120ms  ✅                                │
│    ├─ order-service: 150ms  ✅                           │
│    └─ inventory-service: 7,900ms  ❌ ← 瓶颈在这里！       │
│         └─ SQL: SELECT * FROM inventory WHERE ...        │
│            耗时 7.8s  ← MySQL 慢查询导致！                │
└─────────────────────────────────────────────────────────┘
```

**排查路径**：Kiali 看拓扑 → 发现 inventory-service 变红 → 点 Jaeger 看链路 → 发现是 MySQL 慢查询 → 找 DBA 优化索引 → 问题解决。

**没有 Istio 时**：接到告警 → 挨个服务看日志 → 猜是哪个环节慢 → 加日志重新部署 → 再看日志 → 可能半小时过去了还没定位到根因。

---

### 10.4 全部配置汇总

把上面的场景整合到一个目录结构里：

```
istio-config/
├── 01-destination-rules.yaml    # 所有服务的版本子集 + 熔断策略
├── 02-virtual-services.yaml     # 路由规则 + 超时重试 + 金丝雀权重
├── 03-gateway.yaml              # 入口网关
├── 04-peer-authentication.yaml  # mTLS 全局策略
├── 05-authorization-policies.yaml # 各服务的授权策略
└── 06-observability.yaml        # Kiali/Jaeger 配置（或用 istioctl dashboard）
```

**一键部署**：

```bash
kubectl apply -f istio-config/
```

**验证**：

```bash
# 查看金丝雀流量分布
kubectl exec -it deploy/frontend -c istio-proxy -- \
  pilot-agent request GET stats | grep order-service

# 查看 mTLS 状态
istioctl experimental authz check order-service-xxx

# 打开 Kiali 面板
istioctl dashboard kiali
```

---

### 10.5 这个案例覆盖了哪些 Istio 功能

| 场景 | 功能 | 用到的 CRD | 解决的问题 |
|------|------|-----------|-----------|
| A 金丝雀发布 | 流量管理 | VirtualService + DestinationRule | 新版本逐步上线，出问题秒级回滚 |
| B 超时重试 | 流量管理 | VirtualService | 下游慢不拖垮上游，自动重试提高成功率 |
| C 熔断 | 流量管理 | DestinationRule | 故障服务自动隔离，防止雪崩 |
| D 全局 mTLS | 安全 | PeerAuthentication | 零代码实现服务间加密通信 |
| E 细粒度授权 | 安全 | AuthorizationPolicy | 精确到 HTTP 方法 + 路径的访问控制 |
| F 可观测性 | 可观测性 | 内置集成 | Kiali 看拓扑 + Jaeger 看链路，分钟级定位故障 |

**一句话总结这个案例**：前端改一行代码都没改，5 个 YAML 文件就把一个裸奔的微服务集群变成了有金丝雀发布、自动熔断、mTLS 加密、权限控制、全局可观测的「企业级」架构。