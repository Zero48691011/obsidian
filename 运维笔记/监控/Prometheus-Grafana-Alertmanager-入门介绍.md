# Prometheus + Grafana + Alertmanager 入门介绍

## 概述

这三个组件是云原生监控领域最流行的开源组合，通常被称为 **PGA 监控栈**。它们各司其职，配合使用可以覆盖从数据采集、可视化展示到告警通知的完整监控链路。

```
┌──────────────┐     ┌──────────────┐     ┌───────────────┐
│  Prometheus  │────▶│   Grafana    │     │ Alertmanager  │
│  (采集+存储) │     │  (可视化)    │     │  (告警通知)    │
└──────┬───────┘     └──────────────┘     └───────▲───────┘
       │                                         │
       │  拉取指标                                │  告警推送
       ▼                                         │
┌──────────────┐                        ┌────────────────┐
│  Exporters   │                        │  Prometheus    │
│  (被监控目标) │                        │  (告警规则触发) │
└──────────────┘                        └────────────────┘
```

---

## 一、Prometheus — 监控数据采集与存储

### 是什么

Prometheus 是一个**开源的系统监控和告警工具包**，由 SoundCloud 开发，2016 年加入 CNCF（云原生计算基金会），是继 Kubernetes 之后第二个从 CNCF 毕业的项目。

### 核心功能

| 功能 | 说明 |
|------|------|
| **指标采集** | 通过 HTTP Pull 模型定期从目标拉取指标数据 |
| **时序数据库** | 内置 TSDB，高效存储带时间戳的指标数据 |
| **PromQL** | 强大的查询语言，支持聚合、计算、过滤 |
| **告警规则** | 基于 PromQL 定义告警条件，触发后推送给 Alertmanager |
| **服务发现** | 自动发现 Kubernetes、Consul、EC2 等环境中的监控目标 |

### 核心概念

- **Metric（指标）**：一个带标签的时序数据，如 `http_requests_total{method="GET", status="200"}`
- **Label（标签）**：键值对，用于区分和筛选指标，如 `instance="10.0.0.1:9090"`
- **Sample（样本）**：某个时间点的指标值，由时间戳 + 数值组成
- **Exporter**：运行在被监控主机上的代理，暴露指标给 Prometheus 拉取（如 Node Exporter 暴露 CPU、内存、磁盘等系统指标）

### 数据模型（四种指标类型）

| 类型 | 说明 | 示例 |
|------|------|------|
| **Counter** | 只增不减的计数器 | 请求总数、错误次数 |
| **Gauge** | 可增可减的瞬时值 | CPU 使用率、内存占用 |
| **Histogram** | 对观测值分组统计（分桶） | 请求延迟分布（0~10ms, 10~50ms, ...） |
| **Summary** | 类似 Histogram，但在客户端计算分位数 | P50/P90/P99 延迟 |

### 工作方式

Prometheus 采用 **Pull 模型** —— 主动去目标拉取指标，而不是等目标推送。这是它与很多传统监控工具（如 Nagios、Zabbix）最大的区别。

```
Prometheus Server ──── HTTP GET /metrics ────▶ Node Exporter (被监控机器)
                        ◀──── 纯文本指标数据 ────
```

---

## 二、Grafana — 可视化仪表盘

### 是什么

Grafana 是一个**开源的数据可视化和监控仪表盘平台**。它不存储数据，而是连接各种数据源（Prometheus、InfluxDB、MySQL、Elasticsearch 等），将数据以图表、表格、热力图等形式展示出来。

### 核心功能

| 功能 | 说明 |
|------|------|
| **多数据源** | 支持 Prometheus、Graphite、InfluxDB、MySQL、PostgreSQL、Elasticsearch 等数十种数据源 |
| **仪表盘** | 拖拽式面板布局，支持时序图、柱状图、饼图、表格、热力图、状态面板等 |
| **变量与模板** | 动态切换主机、服务、时间范围等维度，一套仪表盘适配多场景 |
| **告警** | 内置告警引擎（可替代或补充 Alertmanager，但通常仍用 Alertmanager） |
| **权限管理** | 多组织、多用户、RBAC 权限控制 |
| **插件生态** | 丰富的社区仪表盘和面板插件 |

### 与 Prometheus 的关系

Grafana 本身不采集也不存储数据，它通过 PromQL 查询 Prometheus，将结果渲染成图表：

```
Grafana ──── PromQL 查询 ────▶ Prometheus ──── 返回时序数据 ────▶ 渲染图表
```

### 常用仪表盘

Grafana 社区有大量开箱即用的仪表盘模板，导入即可使用：

- **Node Exporter Full**（ID: 1860）：Linux 主机全方位监控
- **Kubernetes Cluster**（ID: 315）：K8s 集群概览
- **1 Panel Prometheus**（ID: 2）：Prometheus 自身监控

---

## 三、Alertmanager — 告警管理与通知

### 是什么

Alertmanager 是 Prometheus 生态中的**告警管理组件**，负责接收 Prometheus Server 推送的告警，然后进行**分组、抑制、静默、路由**，最终通过邮件、Slack、钉钉、Webhook 等渠道发送通知。

### 为什么要有 Alertmanager

Prometheus 本身只负责"发现异常并触发告警"，但告警怎么发、发给谁、怎么去重、怎么抑制——这些不归 Prometheus 管。Alertmanager 就是专门解决这些问题的。

### 核心功能

| 功能 | 说明 | 场景 |
|------|------|------|
| **分组（Grouping）** | 将相似告警合并成一条通知 | 10 台机器同时宕机 → 只发 1 条通知，而不是 10 条 |
| **抑制（Inhibition）** | 高优先级告警触发后，抑制相关的低优先级告警 | 整机宕机 → 自动抑制该机上所有「服务不可用」告警 |
| **静默（Silencing）** | 手动或定时屏蔽某类告警 | 计划内维护期间，临时屏蔽告警 |
| **路由（Routing）** | 根据告警标签分发到不同接收者 | 数据库告警 → DBA 团队；网络告警 → 网络团队 |
| **重复提醒** | 告警未恢复时，按间隔重复发送 | 每 4 小时提醒一次，直到问题解决 |

### 通知渠道

Alertmanager 支持通过 `receiver` 配置多种通知方式：

- **Email**（SMTP）
- **Slack** / **Mattermost**
- **钉钉** / **飞书** / **企业微信** Webhook
- **PagerDuty** / **OpsGenie**
- **Webhook**（自定义 HTTP 回调）
- **Telegram**

### 工作流程

```
Prometheus 告警规则触发
        │
        ▼
Alertmanager 接收告警
        │
        ├── 分组：合并相似告警
        ├── 抑制：过滤低优先级告警
        ├── 静默：检查是否在静默期
        │
        ▼
路由匹配 → 发送到对应 Receiver → 通知相关人员
```

---

## 四、三者协作全景

```
┌─────────────────────────────────────────────────────────────────┐
│                        监控目标                                  │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐        │
│  │  Node    │  │  MySQL   │  │  Nginx   │  │  自定义   │        │
│  │ Exporter │  │ Exporter │  │ Exporter │  │  Exporter │        │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘        │
│       │              │              │              │              │
│       └──────────────┼──────────────┼──────────────┘              │
│                      │   Pull 指标   │                            │
│                      ▼              ▼                            │
│              ┌──────────────────────────┐                        │
│              │      Prometheus          │                        │
│              │   (TSDB + PromQL + 告警)  │                        │
│              └──────┬──────────┬────────┘                        │
│                     │          │                                  │
│          PromQL 查询 │          │ 告警推送                         │
│                     ▼          ▼                                  │
│           ┌──────────────┐  ┌────────────────┐                  │
│           │   Grafana    │  │  Alertmanager  │                  │
│           │  (仪表盘)     │  │  (分组/路由/通知)│                  │
│           └──────────────┘  └───────┬────────┘                  │
│                                      │                           │
│                                      ▼                           │
│                          ┌─────────────────────┐                │
│                          │  邮件 / Slack / 钉钉  │                │
│                          │  飞书 / PagerDuty ... │                │
│                          └─────────────────────┘                │
└─────────────────────────────────────────────────────────────────┘
```

### 一句话总结

| 组件 | 一句话 |
|------|--------|
| **Prometheus** | 采集指标、存储数据、触发告警 |
| **Grafana** | 把 Prometheus 的数据变成漂亮的图表 |
| **Alertmanager** | 管理告警怎么发、发给谁、什么时候发 |

---

## 五、快速上手

### 最小化 Docker Compose 部署

```yaml
version: '3'
services:
  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - grafana_data:/var/lib/grafana

  alertmanager:
    image: prom/alertmanager:latest
    ports:
      - "9093:9093"
    volumes:
      - ./alertmanager.yml:/etc/alertmanager/alertmanager.yml

  node-exporter:
    image: prom/node-exporter:latest
    ports:
      - "9100:9100"

volumes:
  prometheus_data:
  grafana_data:
```

### 默认端口

| 组件 | 默认端口 | 访问地址 |
|------|----------|----------|
| Prometheus | 9090 | `http://localhost:9090` |
| Grafana | 3000 | `http://localhost:3000` |
| Alertmanager | 9093 | `http://localhost:9093` |
| Node Exporter | 9100 | `http://localhost:9100/metrics` |

---

## 六、常见问题

**Q: Prometheus 和 Grafana 有什么区别？**
> Prometheus 负责**存数据**，Grafana 负责**画图表**。Grafana 也可以连接 MySQL、InfluxDB 等其他数据源，不限于 Prometheus。

**Q: 为什么需要 Alertmanager？Prometheus 不能直接发告警吗？**
> Prometheus 可以触发告警，但缺乏告警管理能力（分组、抑制、静默）。不用 Alertmanager 的话，10 台机器同时挂掉你会收到 10 条告警，而不是 1 条。

**Q: 数据存多久？占用多少磁盘？**
> Prometheus 默认保留 15 天数据。每 10 万样本/秒大约占用 1.5 GB/天。实际取决于指标数量和采集频率。

**Q: 大规模场景怎么办？**
> 可以使用 Thanos 或 Cortex（VictoriaMetrics）做**长期存储**和**水平扩展**，或使用 Grafana Mimir 作为 Prometheus 的替代后端。

---

*文档创建时间：2026-07-02*