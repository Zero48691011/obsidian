# MySQL 慢日志占满磁盘导致数据库异常问题分析报告

---

## 一、故障概述

### 1.1 故障时间

[待补充]

### 1.2 故障现象

- 服务器根分区磁盘使用率飙升至 **100%**，磁盘空间完全耗尽
- Docker 部署的 MySQL 容器进程异常退出，业务系统无法访问数据库
- 本地 MySQL 客户端连接报错，表现为连接握手失败、通讯中断
- 重启容器后可短暂恢复，但很快再次异常

### 1.3 故障影响

| 影响维度 | 详情 |
|----------|------|
| 业务影响 | 数据库读写完全中断，所有依赖 MySQL 的业务系统不可用 |
| 持续时间 | [待补充] |
| 数据安全 | 未造成数据丢失，但存在数据写入中断风险 |

---

## 二、根因分析

### 2.1 直接原因

MySQL 慢查询日志 `slow.log` **无自动清理机制**，在业务慢 SQL 堆积的叠加效应下，日志文件持续膨胀，最终占满服务器整块磁盘（根分区 100%），导致：

1. **磁盘写入阻塞**：操作系统无法为任何进程分配新的磁盘块
2. **MySQL 异常退出**：无法写入数据文件、binlog、redo log，进程被系统终止
3. **容器状态异常**：Docker 容器因底层存储不可写而进入不稳定状态

### 2.2 深层原因

```text
慢 SQL 未优化（根源）
    ↓
slow.log 快速增长
    ↓
无日志轮转 / 清理机制（放大因素）
    ↓
磁盘空间耗尽
    ↓
MySQL 进程不可用
```

**关键链路上的两个薄弱点：**

1. **慢 SQL 缺乏治理** — 业务侧未设置合理的 `long_query_time` 阈值，未启用 `log_queries_not_using_indexes` 的审慎过滤，导致低价值慢查询大量写入日志
2. **日志缺乏生命周期管理** — 既无 `logrotate` 切割归档，也无定时清理任务，日志成为"只写不删"的无限增长文件

### 2.3 连接异常的技术原因

磁盘满载时，MySQL 连接过程涉及多个磁盘 I/O 环节：

- **连接握手阶段**：读取 `mysql.user` 等系统表进行权限验证 → 磁盘不可写时文件句柄状态异常 → 握手失败（`ERROR 2013`）
- **通讯阶段**：SSL/TLS 握手涉及的临时文件写入失败 → 连接中断（`Lost connection to MySQL server`）
- **资源释放**：磁盘清理后文件系统恢复正常，连接资源得以重新分配

---

## 三、处置过程

| 步骤 | 操作 | 结果 |
|------|------|------|
| 1 | 登录服务器，`df -h` 确认根分区 100% 满载 | 定位问题范围 |
| 2 | `du -sh /* \| sort -rh \| head` 定位超大文件 | 发现 `slow.log` 异常膨胀 |
| 3 | `> /data/middle/mysql_logs/slow.log` 清空慢日志 | 磁盘降至 47%（释放约 50% 空间） |
| 4 | `docker-compose restart mysql` 重启容器 | 容器正常启动，端口监听恢复 |
| 5 | `mysql -h 127.0.0.1 -P 3306 -u root -p` 本地连接测试 | 登录访问正常 |
| 6 | 配置定时任务 + 分析慢 SQL 原因 | 防范再次发生 |

---

## 四、优化整改方案

### 4.1 日志管控（立即执行）

**方案一：定时清空（已配置）**

```bash
# 每周日凌晨 2 点清空慢日志
0 2 * * 7 echo > /data/middle/mysql_logs/slow.log
```

**方案二：logrotate 日志轮转（强烈推荐）**

```conf
# /etc/logrotate.d/mysql-slow
/data/middle/mysql_logs/slow.log {
    daily                    # 每天轮转
    rotate 7                 # 保留 7 天
    size 500M                # 或超过 500M 强制轮转
    compress                 # 压缩归档
    delaycompress            # 延迟一天压缩，确保 MySQL 写入完成
    missingok                # 文件不存在不报错
    notifempty               # 空文件不轮转
    postrotate
        docker exec mysql mysqladmin -u root -p'xxx' flush-logs 2>/dev/null || true
    endscript
}
```

### 4.2 慢查询治理（中期优化）

**MySQL 参数调优：**

```sql
-- 适度提高慢查询阈值（根据业务 SLO 设定，建议 0.5-2 秒）
SET GLOBAL long_query_time = 1;

-- 只记录使用全表扫描且未走索引的查询（减少日志量 60-80%）
SET GLOBAL log_queries_not_using_indexes = 0;

-- 限制慢日志总大小（MySQL 8.0+ 支持）
SET GLOBAL slow_query_log_max_len = 104857600; -- 100MB
```

**业务侧配合：**

1. 使用 `pt-query-digest` 或 `mysqldumpslow` 分析 Top-N 慢 SQL
2. 针对高频慢查询添加索引、优化 SQL 语句
3. 将慢 SQL 分析纳入 Code Review 流程

### 4.3 监控告警（长期建设）

| 监控项 | 阈值 | 告警方式 |
|--------|------|----------|
| 磁盘使用率 | > 80% | 即时告警（企业微信/钉钉/飞书） |
| slow.log 文件大小 | > 1GB | 预警通知 |
| MySQL 进程存活 | 进程退出 | 紧急告警 + 自动重启 |
| 慢查询数量突增 | 环比增长 > 200% | 预警通知 |

**轻量级监控脚本：**

```bash
#!/bin/bash
# /etc/cron.d/disk_monitor — 每 5 分钟检查磁盘
USAGE=$(df / | tail -1 | awk '{print $5}' | sed 's/%//')
SLOG_SIZE=$(stat -c%s /data/middle/mysql_logs/slow.log 2>/dev/null || echo 0)
SLOG_MB=$((SLOG_SIZE / 1048576))

[ "$USAGE" -gt 80 ] && echo "⚠️ 磁盘使用率 ${USAGE}% 超过阈值" | tee -a /var/log/disk_alert.log
[ "$SLOG_MB" -gt 1024 ] && echo "⚠️ slow.log 已超过 1GB (${SLOG_MB}MB)" | tee -a /var/log/disk_alert.log
```

---

## 五、总结

| 项 | 内容 |
|----|------|
| **故障根因** | 慢日志无清理机制 + 业务慢 SQL 堆积 → 磁盘耗尽 → MySQL 宕机 |
| **直接触发** | slow.log 无限增长至占满根分区 |
| **临时恢复** | 清空日志 + 重启容器 → 磁盘 47%，服务正常 |
| **短期防护** | 配置定时清空 + logrotate 切割 |
| **长期治理** | 慢 SQL 优化 + 参数调优 + 磁盘监控告警 |
| **教训** | 任何"只写不删"的日志文件都必须有生命周期管理；磁盘监控永远不能缺失 |

---

*文档编制：[待补充] &nbsp;|&nbsp; 审核：[待补充] &nbsp;|&nbsp; 日期：2026-05-26*