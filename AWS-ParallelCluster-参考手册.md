# AWS ParallelCluster 参考手册

> 基于 AWS ParallelCluster 官方文档（v3）整理，简体中文版。最后更新：2026-06-15。

---

## 目录

1. [概述](#1-概述)
2. [架构与工作原理](#2-架构与工作原理)
3. [安装与配置](#3-安装与配置)
4. [集群配置文件详解](#4-集群配置文件详解)
5. [HeadNode 配置](#5-headnode-配置)
6. [Scheduling 调度器配置](#6-scheduling-调度器配置)
7. [SharedStorage 共享存储](#7-sharedstorage-共享存储)
8. [Image 镜像与操作系统](#8-image-镜像与操作系统)
9. [IAM 权限管理](#9-iam-权限管理)
10. [网络配置](#10-网络配置)
11. [CLI 命令参考](#11-cli-命令参考)
12. [Auto Scaling 弹性伸缩](#12-auto-scaling-弹性伸缩)
13. [监控与日志](#13-监控与日志)
14. [故障排查](#14-故障排查)
15. [最佳实践](#15-最佳实践)

---

## 1. 概述

### 什么是 AWS ParallelCluster？

AWS ParallelCluster 是 AWS 官方支持的开源集群管理工具，用于在 AWS 云中部署和管理高性能计算 (HPC) 集群。它自动配置所需的计算资源、调度器和共享文件系统。

**核心特性：**
- 支持 **Slurm** 和 **AWS Batch** 两种调度器
- 自动创建头节点、计算节点、共享存储
- 支持弹性伸缩（Auto Scaling）
- 开箱即用的 HPC 环境

**访问方式：**
| 方式 | 说明 |
|------|------|
| `pcluster` CLI | 命令行工具，主要交互方式 |
| AWS ParallelCluster API | 编程接口 |
| PCUI | Web 管理界面（v3.5.0+） |
| Python 库 API | 程序化调用（v3.5.0+） |
| CloudFormation | 自定义资源（v3.6.0+） |

**定价：** 使用 pcluster CLI 或 API 本身免费，只需为创建的 AWS 资源（EC2、EBS、FSx 等）付费。PCUI 基于无服务器架构，大多数情况在 AWS 免费套餐范围内。

---

## 2. 架构与工作原理

### 架构概览

```
┌─────────────────────────────────────────────────────┐
│                   AWS Cloud                          │
│  ┌───────────────────────────────────────────────┐  │
│  │               VPC / Subnet                     │  │
│  │  ┌──────────┐    ┌──────────────────────────┐ │  │
│  │  │ HeadNode │    │    Compute Fleet          │ │  │
│  │  │ (EC2)    │    │  ┌──────┐ ┌──────┐ ┌────┐│ │  │
│  │  │  + Slurm │───▶│  │ Node │ │ Node │ │... ││ │  │
│  │  │  + NFS   │    │  └──────┘ └──────┘ └────┘│ │  │
│  │  └────┬─────┘    └──────────────────────────┘ │  │
│  │       │                                        │  │
│  │  ┌────┴──────────────────────────────────┐    │  │
│  │  │          Shared Storage                │    │  │
│  │  │  EBS / EFS / FSx Lustre / FSx ONTAP   │    │  │
│  │  └───────────────────────────────────────┘    │  │
│  └───────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────┘
```

### 核心组件

| 组件 | 说明 |
|------|------|
| **HeadNode** | 头节点，运行调度器、共享存储挂载、管理服务 |
| **Compute Nodes** | 计算节点，由 Auto Scaling Group 管理，按需弹性伸缩 |
| **Shared Storage** | 共享存储，支持 EBS / EFS / FSx for Lustre / FSx for ONTAP / FSx for OpenZFS |
| **Scheduler** | 作业调度器，Slurm 或 AWS Batch |

### 生命周期

1. **pcluster create-cluster** → 创建 CloudFormation 堆栈
2. CloudFormation 创建 HeadNode + 初始资源
3. HeadNode 启动后配置调度器、挂载存储
4. 根据作业队列自动伸缩计算节点
5. **pcluster delete-cluster** → 删除所有资源

---

## 3. 安装与配置

### 前提条件

- AWS 账号及 IAM 用户（具有 AdministratorAccess 或 pcluster 所需权限）
- AWS CLI 已安装并配置（`aws configure`）
- Python 3.7+
- 已有 VPC、子网、SSH 密钥对

### 安装 CLI

```bash
# 方式一：pip 安装（推荐）
pip3 install aws-parallelcluster

# 方式二：使用 virtualenv
python3 -m venv pcluster-env
source pcluster-env/bin/activate
pip install aws-parallelcluster

# 验证安装
pcluster version
```

### 安装后配置

```bash
# 初始化配置（交互式）
pcluster configure

# 配置文件路径
~/.parallelcluster/config.yaml
```

### 配置 AWS 凭证

```bash
# 确保 AWS CLI 已配置
aws configure
# 输入：AWS Access Key ID、Secret Access Key、Region、Output Format
```

### 安装 PCUI（可选）

从 v3.5.0 开始提供 Web 管理界面，通过 CloudFormation 一键部署。

---

## 4. 集群配置文件详解

### 配置文件结构

```yaml
Region: us-east-1
Image:
  Os: alinux2          # 或 ubuntu2004, centos7, rhel8

HeadNode:
  InstanceType: c5.xlarge
  Networking:
    SubnetId: subnet-xxxxxxxx
  Ssh:
    KeyName: my-key-pair

Scheduling:
  Scheduler: slurm     # 或 awsbatch
  SlurmQueues:
    - Name: compute
      ComputeResources:
        - Name: small
          InstanceType: c5.xlarge
          MinCount: 0
          MaxCount: 10

SharedStorage:
  - Name: shared-data
    StorageType: Ebs
    MountDir: /shared
    EbsSettings:
      Size: 100
      VolumeType: gp3
```

### 核心配置段

| 配置段 | 必需 | 说明 |
|--------|------|------|
| `Region` | 是 | AWS 区域 |
| `Image` | 是 | 操作系统与 AMI |
| `HeadNode` | 是 | 头节点配置 |
| `Scheduling` | 是 | 调度器与计算节点 |
| `SharedStorage` | 否 | 共享存储 |
| `Iam` | 否 | IAM 权限 |
| `Monitoring` | 否 | 监控配置 |
| `Tags` | 否 | 标签 |
| `DirectoryService` | 否 | AD 集成 |
| `AdditionalPackages` | 否 | 额外软件包 |
| `Imds` | 否 | 实例元数据服务 |

---

## 5. HeadNode 配置

### 完整配置结构

```yaml
HeadNode:
  InstanceType: string           # 必需
  Networking:
    SubnetId: string             # 必需
    ElasticIp: string/boolean
    SecurityGroups:
      - string
    AdditionalSecurityGroups:
      - string
    Proxy:
      HttpProxyAddress: string
  DisableSimultaneousMultithreading: boolean
  Ssh:
    KeyName: string
    AllowedIps: string
  LocalStorage:
    RootVolume:
      Size: integer
      Encrypted: boolean
      VolumeType: string
      Iops: integer
      Throughput: integer
      DeleteOnTermination: boolean
    EphemeralVolume:
      MountDir: string
  Dcv:
    Enabled: boolean
    Port: integer
    AllowedIps: string
  CustomActions:
    OnNodeStart:
      Script: string
      Args: [string]
    OnNodeConfigured:
      Script: string
      Args: [string]
    OnNodeUpdated:
      Script: string
      Args: [string]
  Iam:
    InstanceRole: string
    InstanceProfile: string
    S3Access:
      - BucketName: string
        EnableWriteAccess: boolean
        KeyName: string
    AdditionalIamPolicies:
      - Policy: string
  Imds:
    Secured: boolean
  Image:
    CustomAmi: string
```

### 关键属性说明

| 属性 | 类型 | 说明 |
|------|------|------|
| `InstanceType` | 必填 | 头节点 EC2 实例类型，架构需与计算节点一致 |
| `SubnetId` | 必填 | 头节点所在的子网 |
| `ElasticIp` | 可选 | 是否分配弹性 IP，默认 true |
| `KeyName` | 可选 | SSH 密钥对名称 |
| `LocalStorage.RootVolume.Size` | 可选 | 根卷大小 (GiB) |
| `Dcv.Enabled` | 可选 | 是否启用 NICE DCV 远程桌面 |
| `CustomActions` | 可选 | 自定义脚本，在节点生命周期各阶段执行 |

### 自定义脚本生命周期

| 时机 | 说明 |
|------|------|
| `OnNodeStart` | 节点启动时，调度器启动前 |
| `OnNodeConfigured` | 节点配置完成后，调度器启动后 |
| `OnNodeUpdated` | 集群更新时 |

### 不支持的实例类型

- `hpc6id` 不能用作 HeadNode

---

## 6. Scheduling 调度器配置

AWS ParallelCluster 支持两种调度器：
- **Slurm**：传统 HPC 调度器，功能丰富，推荐用于大多数 HPC 场景
- **AWS Batch**：AWS 托管调度器，适合简单任务队列

### Slurm 配置结构

```yaml
Scheduling:
  Scheduler: slurm
  SlurmSettings:
    ScaledownIdletime: integer        # 闲置缩容时间（分钟），默认 10
    QueueUpdateStrategy: string       # DRAIN / TERMINATE
    EnableMemoryBasedScheduling: boolean
    CustomSlurmSettings: [dict]
    CustomSlurmSettingsIncludeFile: string
    Database:                         # Slurm 记账数据库
      Uri: string
      UserName: string
      PasswordSecretArn: string
    Dns:
      DisableManagedDns: boolean
      HostedZoneId: string
      UseEc2Hostnames: boolean
  SlurmQueues:
    - Name: string                    # 队列名称
      CapacityType: string            # ONDEMAND / SPOT
      AllocationStrategy: string      # lowest-price / capacity-optimized
      ComputeResources:
        - Name: string
          InstanceType: string        # 或 Instances: [{InstanceType: ...}]
          MinCount: integer
          MaxCount: integer
          SpotPrice: float
          DisableSimultaneousMultithreading: boolean
          SchedulableMemory: integer
          Efa:
            Enabled: boolean
            GdrSupport: boolean
          Gpu:
            Enabled: boolean
```

### AWS Batch 配置结构

```yaml
Scheduling:
  Scheduler: awsbatch
  AwsBatchQueues:
    - Name: string
      CapacityType: string            # ONDEMAND / SPOT
      Networking:
        SubnetIds: [string]
        AssignPublicIp: boolean
      ComputeResources:
        - Name: string
          InstanceTypes: [string]
          MinvCpus: integer
          DesiredvCpus: integer
          MaxvCpus: integer
          SpotBidPercentage: float
```

> **注意**：AWS Batch 仅支持 `alinux2` 操作系统和 `x86_64` 架构。

### 关键属性说明

| 属性 | 说明 |
|------|------|
| `ScaledownIdletime` | 节点空闲多久后缩容（分钟），默认 10 |
| `CapacityType` | `ONDEMAND`（按需）或 `SPOT`（竞价） |
| `AllocationStrategy` | `lowest-price`（最低价）或 `capacity-optimized`（容量优化） |
| `MinCount` / `MaxCount` | 该计算资源的最小/最大实例数 |
| `Efa.Enabled` | 是否启用 Elastic Fabric Adapter（高性能网络） |
| `GdrSupport` | 是否启用 GPUDirect RDMA |

### 更新策略

- **`Update policy: The compute fleet must be stopped`** — 需要先停止计算集群才能修改
- **`Update policy: This setting can be changed during an update`** — 可以在运行中修改
- **`Update policy: If this setting is changed, the update is not allowed`** — 不可修改，需重建集群

---

## 7. SharedStorage 共享存储

### 支持的存储类型

| 存储类型 | 配置名 | 说明 |
|----------|--------|------|
| Amazon EBS | `Ebs` | 块存储，单节点挂载（头节点） |
| Amazon EFS | `Efs` | 弹性文件系统，多节点共享 |
| FSx for Lustre | `FsxLustre` | 高性能并行文件系统 |
| FSx for ONTAP | `FsxOntap` | NetApp ONTAP 文件系统 |
| FSx for OpenZFS | `FsxOpenZfs` | OpenZFS 文件系统 |

### 外部存储 vs 托管存储

| 类型 | 说明 |
|------|------|
| **外部存储** | 已有卷/文件系统，通过 `VolumeId` / `FileSystemId` 引用，pcluster 不创建/删除 |
| **托管存储** | pcluster 创建和管理，通过 `DeletionPolicy` 控制删除行为 |

### EBS 配置示例

```yaml
SharedStorage:
  - Name: my-ebs
    StorageType: Ebs
    MountDir: /shared
    EbsSettings:
      VolumeType: gp3
      Size: 500
      Encrypted: true
      DeletionPolicy: Retain     # Delete / Retain / Snapshot
      Raid:
        Type: 0                  # RAID 类型
        NumberOfVolumes: 3       # 卷数量
```

### EFS 配置示例

```yaml
SharedStorage:
  - Name: my-efs
    StorageType: Efs
    MountDir: /efs
    EfsSettings:
      Encrypted: true
      PerformanceMode: maxIO     # generalPurpose / maxIO
      ThroughputMode: bursting   # bursting / provisioned
      ProvisionedThroughput: 100
      FileSystemId: fs-xxxxx     # 使用已有 EFS
```

### FSx for Lustre 配置示例

```yaml
SharedStorage:
  - Name: my-lustre
    StorageType: FsxLustre
    MountDir: /lustre
    FsxLustreSettings:
      StorageCapacity: 1200
      DeploymentType: PERSISTENT_2   # PERSISTENT_1 / PERSISTENT_2 / SCRATCH_1 / SCRATCH_2
      PerUnitStorageThroughput: 125  # 125 / 250 / 500 / 1000 MB/s/TiB
      ImportPath: s3://my-bucket/
      ExportPath: s3://my-bucket/export/
      DataCompressionType: LZ4
      DeletionPolicy: Delete
```

### EBS 卷类型 IOPS 参考

| 卷类型 | 默认 IOPS | IOPS 范围 | IOPS:Size 最大比 |
|--------|----------|----------|-----------------|
| `gp3` | 3000 | 3000-16000 | 500:1 |
| `io1` | 100 | 100-64000 | 50:1 |
| `io2` | 100 | 100-256000 | 500:1 |

---

## 8. Image 镜像与操作系统

### 配置

```yaml
Image:
  Os: alinux2          # 必填
  CustomAmi: ami-xxxxx # 可选
```

### 支持的操作系统

| 值 | 操作系统 | 说明 |
|----|---------|------|
| `alinux2` | Amazon Linux 2 | 默认推荐 |
| `ubuntu2004` | Ubuntu 20.04 | - |
| `ubuntu1804` | Ubuntu 18.04 | - |
| `centos7` | CentOS 7 | 部分区域不支持 |
| `rhel8` | Red Hat Enterprise Linux 8.7 | v3.6.0+，按需费用更高 |

### 自定义 AMI

- 可通过 `CustomAmi` 指定自定义 AMI ID
- 自定义 AMI 需要额外的权限（如加密快照需 KMS 权限）
- RHEL 自定义 AMI 需确保 `kernel-devel` 包版本与运行内核一致
- RHEL 8.2+ 才支持 FSx for Lustre
- RHEL 8.4+ 才支持 EFA

### 构建自定义 AMI

```bash
pcluster build-image --image-id my-custom-ami \
  --image-configuration image-config.yaml
```

---

## 9. IAM 权限管理

### 配置

```yaml
Iam:
  Roles:
    LambdaFunctionsRole: string    # Lambda 函数角色 ARN
  PermissionsBoundary: string      # 权限边界 ARN
  ResourcePrefix: string           # 资源前缀
```

### 关键属性

| 属性 | 说明 |
|------|------|
| `PermissionsBoundary` | 所有 pcluster 创建角色的权限边界，格式 `arn:aws:iam::${Account}:policy/${PolicyName}` |
| `LambdaFunctionsRole` | 覆盖 Lambda 函数的默认角色，格式 `arn:aws:iam::${Account}:role/${RoleName}` |
| `ResourcePrefix` | IAM 资源名称/路径前缀，最多 30 字符 |

### ResourcePrefix 示例

```yaml
# 名称前缀
Iam:
  ResourcePrefix: my-prefix

# 路径前缀
Iam:
  ResourcePrefix: /org/dept/team/project/user/

# 路径 + 名称
Iam:
  ResourcePrefix: /org/dept/team/project/user/my-prefix
```

### HeadNode 级别 IAM

```yaml
HeadNode:
  Iam:
    InstanceRole: string
    InstanceProfile: string
    S3Access:
      - BucketName: my-bucket
        EnableWriteAccess: true
        KeyName: optional/prefix
    AdditionalIamPolicies:
      - Policy: arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
```

---

## 10. 网络配置

### 网络需求

- 一个 VPC 及至少一个子网
- 头节点：需要公有子网（或 NAT 网关访问互联网）
- 计算节点：建议私有子网，通过 NAT 网关访问互联网
- 安全组：pcluster 自动创建，也可指定已有安全组

### 网络配置示例

```yaml
HeadNode:
  Networking:
    SubnetId: subnet-xxxxxxxx
    ElasticIp: true
    SecurityGroups:
      - sg-xxxxxxxx
    Proxy:
      HttpProxyAddress: http://proxy.example.com:8080

Scheduling:
  SlurmQueues:
    - Networking:
        SubnetIds:
          - subnet-yyyyyyyy
        AssignPublicIp: false
        PlacementGroup:
          Enabled: true
```

### 网络架构推荐

```
┌──────────────────────────────────┐
│  VPC                              │
│  ┌────────────┐  ┌────────────┐  │
│  │ 公有子网    │  │ 私有子网    │  │
│  │            │  │            │  │
│  │ HeadNode   │  │ Compute    │  │
│  │ + EIP      │  │ Nodes      │  │
│  │            │  │            │  │
│  └────────────┘  └─────┬──────┘  │
│                        │         │
│                   ┌────┴──────┐  │
│                   │ NAT Gateway│  │
│                   └───────────┘  │
└──────────────────────────────────┘
```

---

## 11. CLI 命令参考

### 命令总览

```bash
pcluster [command] [options]
```

### 集群管理

| 命令 | 说明 |
|------|------|
| `pcluster configure` | 交互式配置 |
| `pcluster create-cluster` | 创建集群 |
| `pcluster update-cluster` | 更新集群 |
| `pcluster delete-cluster` | 删除集群 |
| `pcluster describe-cluster` | 查看集群详情 |
| `pcluster list-clusters` | 列出所有集群 |
| `pcluster ssh` | SSH 登录头节点 |

### 计算集群管理

| 命令 | 说明 |
|------|------|
| `pcluster describe-compute-fleet` | 查看计算集群状态 |
| `pcluster update-compute-fleet` | 启动/停止计算集群 |
| `pcluster describe-cluster-instances` | 查看集群实例 |
| `pcluster delete-cluster-instances` | 删除指定实例 |

### 镜像管理

| 命令 | 说明 |
|------|------|
| `pcluster build-image` | 构建自定义 AMI |
| `pcluster describe-image` | 查看镜像详情 |
| `pcluster delete-image` | 删除镜像 |
| `pcluster list-images` | 列出自定义镜像 |
| `pcluster list-official-images` | 列出官方镜像 |

### 日志与诊断

| 命令 | 说明 |
|------|------|
| `pcluster export-cluster-logs` | 导出集群日志到 S3 |
| `pcluster export-image-logs` | 导出镜像构建日志 |
| `pcluster get-cluster-log-events` | 查看集群日志事件 |
| `pcluster get-cluster-stack-events` | 查看 CloudFormation 堆栈事件 |
| `pcluster list-cluster-log-streams` | 列出日志流 |

### 常用命令示例

```bash
# 创建集群
pcluster create-cluster --cluster-name my-cluster \
  --cluster-configuration config.yaml

# 查看集群状态
pcluster describe-cluster --cluster-name my-cluster

# SSH 登录头节点
pcluster ssh --cluster-name my-cluster

# 停止计算集群
pcluster update-compute-fleet --cluster-name my-cluster \
  --status STOP_REQUESTED

# 启动计算集群
pcluster update-compute-fleet --cluster-name my-cluster \
  --status START_REQUESTED

# 更新集群配置
pcluster update-cluster --cluster-name my-cluster \
  --cluster-configuration config.yaml

# 删除集群
pcluster delete-cluster --cluster-name my-cluster

# 导出日志
pcluster export-cluster-logs --cluster-name my-cluster \
  --bucket my-logs-bucket --output-file logs.tar.gz
```

### 日志位置

- CLI 日志：`~/.parallelcluster/pcluster.log.#`
- 集群日志：通过 `pcluster export-cluster-logs` 导出到 S3
- CloudFormation 日志：`pcluster get-cluster-stack-events`

---

## 12. Auto Scaling 弹性伸缩

### Slurm 弹性伸缩

Slurm 模式下，pcluster 通过以下机制实现弹性伸缩：

**扩容（Scale Up）：**
- `jobwatcher` 进程每分钟检查作业队列
- 根据待处理作业的节点需求自动增加计算节点
- 扩容上限由 `MaxCount` 限制

**缩容（Scale Down）：**
- `nodewatcher` 进程在每个计算节点上运行
- 节点空闲超过 `ScaledownIdletime`（默认 10 分钟）时终止
- 缩容需满足：无待处理作业 + 节点超过空闲时间

### AWS Batch 弹性伸缩

AWS Batch 自带弹性伸缩，通过 `MinvCpus` / `DesiredvCpus` / `MaxvCpus` 控制。

### 缩容配置

```yaml
Scheduling:
  SlurmSettings:
    ScaledownIdletime: 10        # 空闲缩容时间（分钟）
    QueueUpdateStrategy: DRAIN   # DRAIN / TERMINATE
```

### 静态集群

如需固定大小集群：
- 设置 `MinCount` = `MaxCount`
- 或者使用 v2 的 `maintain_initial_size: true`

---

## 13. 监控与日志

### 监控配置

```yaml
Monitoring:
  Logs:
    CloudWatch:
      Enabled: boolean
      RetentionInDays: integer
  Dashboards:
    CloudWatch:
      Enabled: boolean
  Alarms:
    Enabled: boolean
```

### CloudWatch 集成

- 集群日志可自动写入 CloudWatch Logs
- 可启用 CloudWatch Dashboard 监控集群
- 支持告警配置

### 关键日志路径

| 日志 | 路径 |
|------|------|
| HeadNode 配置日志 | `/var/log/cfn-init.log` |
| Slurm 日志 | `/var/log/slurmctld` |
| 计算节点日志 | `/var/log/parallelcluster/` |
| CloudWatch Agent | `/opt/aws/amazon-cloudwatch-agent/logs/` |

---

## 14. 故障排查

### 常见问题

#### 集群创建失败

```bash
# 查看 CloudFormation 堆栈事件
pcluster get-cluster-stack-events --cluster-name my-cluster

# 查看集群日志
pcluster get-cluster-log-events --cluster-name my-cluster \
  --log-stream-name head-node
```

#### 计算节点无法启动

1. 检查 `MaxCount` 是否 > 0
2. 检查子网是否有足够的 IP 地址
3. 检查 EC2 实例配额
4. 检查 IAM 权限

```bash
# 查看计算集群状态
pcluster describe-compute-fleet --cluster-name my-cluster
```

#### SSH 连接失败

1. 检查安全组是否允许 22 端口
2. 检查 `KeyName` 是否正确
3. 检查 `AllowedIps` 限制

#### 自定义 AMI 验证失败

```bash
# 查看镜像构建日志
pcluster get-image-log-events --image-id my-ami
```

#### 常见错误码

| 错误 | 原因 | 解决 |
|------|------|------|
| `CREATE_FAILED` | CloudFormation 创建失败 | 检查 IAM 权限、子网、配额 |
| `InsufficientInstanceCapacity` | 可用区容量不足 | 换实例类型或可用区 |
| `InstanceLimitExceeded` | 实例配额超限 | 申请配额提升 |
| `VolumeLimitExceeded` | EBS 卷数量超限 | 减少卷数量或申请配额 |

### 日志导出

```bash
# 导出所有集群日志到 S3
pcluster export-cluster-logs --cluster-name my-cluster \
  --bucket my-bucket --output-file logs.tar.gz

# 导出镜像构建日志
pcluster export-image-logs --image-id my-ami \
  --bucket my-bucket
```

---

## 15. 最佳实践

### 集群设计

1. **使用私有子网部署计算节点**，通过 NAT 网关访问互联网
2. **头节点置于公有子网**，便于 SSH 访问
3. **使用 Placement Group** 减少节点间延迟
4. **为不同类型的工作负载创建多个队列**

### 存储选择

| 场景 | 推荐存储 |
|------|---------|
| 低延迟、高 IOPS | FSx for Lustre |
| 通用共享存储 | EFS |
| 低成本块存储 | EBS gp3 |
| 与 S3 集成 | FSx for Lustre（自动同步 S3） |

### 成本优化

1. **使用 Spot 实例**：计算节点设为 `SPOT` 类型
2. **合理设置缩容时间**：`ScaledownIdletime` 不宜过长
3. **使用 EBS 快照** 备份数据，按需恢复
4. **设置 `DeletionPolicy: Delete`** 使集群删除时自动清理资源

### 安全

1. 启用 EBS 加密（默认 `true`）
2. 使用 `Imds.Secured: true` 限制 IMDS 访问
3. 通过 `AllowedIps` 限制 SSH 来源
4. 使用 IAM 最小权限原则
5. 定期更新 AMI 和软件包

### 升级路径

从 v2.x 迁移到 v3.x 需注意：
- 配置文件格式完全不同
- v3 使用 YAML 配置，v2 使用 INI 格式
- v3 支持多队列、多计算资源类型
- v3 不再支持 SGE 和 Torque 调度器

---

## 参考资源

- [AWS ParallelCluster 官方文档](https://docs.aws.amazon.com/parallelcluster/)
- [GitHub 仓库](https://github.com/aws/aws-parallelcluster)
- [AWS HPC 博客](https://aws.amazon.com/blogs/hpc/)
- [pcluster CLI 源码](https://github.com/aws/aws-parallelcluster-cli)

---

> 本文档基于 AWS ParallelCluster v3 官方文档整理，内容截至 2026-06-15。如有更新请以官方文档为准。