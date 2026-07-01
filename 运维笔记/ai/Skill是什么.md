# Skill 是什么

## 一句话解释

**Skill（技能）** 是 Hermes Agent 中的**可复用知识模块**——它把完成某一类任务的最佳实践、步骤、命令和注意事项打包成一个文档，AI 在执行相关任务时自动加载并遵循。

可以理解为：**给 AI 写的「标准操作手册 (SOP)」**，每次遇到匹配的任务，AI 就会翻开对应的手册照着做。

---

## 为什么需要 Skill

### 没有 Skill 时

```
用户：帮我配置 GitHub CI
AI：（凭记忆和推理，可能遗漏步骤、用错命令、踩已知坑）
```

### 有 Skill 时

```
用户：帮我配置 GitHub CI
AI：检测到 "github" 任务 → 加载 github-pr-workflow skill
   → 按步骤 1-2-3 执行，跳过已知坑，用正确命令
```

### 核心价值

| 场景 | 没有 Skill | 有 Skill |
|------|-----------|----------|
| 复杂任务 | 可能遗漏步骤 | 按 SOP 逐条执行 |
| 踩过的坑 | 每次都重新踩 | 记录在 Skill 里，下次避开 |
| 环境差异 | 用通用方案 | 用针对你环境的方案 |
| 多人使用 | 经验无法传递 | Skill 共享，所有人受益 |

---

## Skill 的结构

一个 Skill 就是一个 Markdown 文件，包含：

```markdown
---
name: skill-name
description: 简短描述这个技能做什么
version: 1.0.0
---

# 技能标题

## 何时使用（触发条件）
- 描述什么情况下应该加载这个技能

## 前置条件
- 需要安装什么
- 需要什么权限

## 步骤
1. 第一步：具体命令
2. 第二步：验证方法
3. ...

## 常见坑
- 坑1：现象 + 原因 + 解决方案
- 坑2：...

## 验证
- 如何确认任务完成
```

---

## Skill 的类型

Hermes Agent 中有三类 Skill：

### 1. 内置 Skill（Bundled）

随 Hermes Agent 安装自带，覆盖通用场景。例如：

| Skill | 用途 |
|-------|------|
| `hermes-agent` | Hermes Agent 自身的配置和使用 |
| `native-mcp` | MCP 客户端配置 |
| `github-pr-workflow` | GitHub PR 创建和管理 |
| `systematic-debugging` | 系统化调试方法 |
| `test-driven-development` | TDD 开发流程 |

### 2. Hub Skill（社区/官方）

从远程 Skill Hub 安装，命令：

```bash
hermes skills search <关键词>    # 搜索
hermes skills install <id>      # 安装
hermes skills browse            # 浏览全部
```

### 3. Agent 自建 Skill（Agent-Created）

当你和 AI 协作完成一个复杂任务后，AI 会把成功经验保存为 Skill。下次遇到类似任务时自动加载。

```bash
# AI 会主动创建（或你手动触发）
hermes skills create <name>
```

---

## Skill 的生命周期

```
创建 → 使用 → 更新 → 可能过时 → 归档
  │       │       │
  │       │       └── 发现问题后立即修补（patch）
  │       └── 每次匹配任务时加载
  └── 复杂任务完成后创建
```

Hermes 的 **Curator（管理员）** 后台进程会自动：
- 追踪 Skill 使用频率
- 标记长时间未使用的 Skill 为「过时」
- 归档过时 Skill（不删除，可恢复）
- 定期备份

---

## 如何管理 Skill

### 查看

```bash
hermes skills list          # 列出已安装
hermes skills search <关键词>  # 搜索 Hub
hermes skills inspect <id>  # 预览某个 Skill
```

### 会话中加载

```
/skill <name>               # 手动加载到当前对话
hermes -s <name>            # 启动时预加载
```

### 配置

```bash
hermes skills config        # 按平台启用/禁用
hermes skills check         # 检查更新
hermes skills update        # 更新所有
```

---

## Skill 的实际例子

### 例子：`github-pr-workflow`

这是一个做 GitHub PR 的 Skill，包含：

1. **触发条件**：用户提到「提交 PR」「创建 pull request」「合并代码」
2. **步骤**：
   - `git checkout -b feature/xxx`
   - 修改代码 + 测试
   - `git add` + `git commit`
   - `gh pr create --title "..." --body "..."`
   - 等待 CI 通过
   - `gh pr merge --squash`
3. **常见坑**：
   - 忘记先 pull 最新代码 → 合并冲突
   - PR 标题不规范 → 用 `feat:` / `fix:` 前缀
4. **验证**：`gh pr status` 确认已合并

### 例子：`ssh-server-manage`

管理远程 Linux 服务器的 Skill，包含：

1. 健康检查命令：`uptime`、`df -h`、`free -m`
2. 诊断步骤：先看负载 → 再看磁盘 → 再看内存 → 最后看日志
3. 你的服务器信息：IP、端口、密钥路径

---

## Skill vs 其他概念

| 概念 | 是什么 | 区别 |
|------|--------|------|
| **Skill** | 可复用的操作流程 | 告诉 AI 「怎么做」 |
| **Memory** | 持久化的事实记忆 | 告诉 AI 「是什么」（用户偏好、环境信息） |
| **MCP Server** | 外部工具接入 | 告诉 AI 「能调什么工具」 |
| **Plugin** | Python 代码扩展 | 改变 Hermes 本身的行为 |

简单记忆：
- **Skill** = 操作手册（怎么做）
- **Memory** = 备忘录（是什么）
- **MCP** = 外设接口（连什么）
- **Plugin** = 系统插件（改什么）

---

## 总结

| 问题 | 答案 |
|------|------|
| Skill 是什么？ | AI 可加载的、可复用的任务操作手册 |
| 解决了什么？ | 让 AI 不再凭记忆做复杂任务，按 SOP 规范执行 |
| 谁创建？ | 官方内置、社区贡献、你和 AI 协作共创 |
| 存在哪？ | `~/.hermes/skills/` 目录 |
| 怎么生效？ | 匹配到任务时自动加载，或手动 `/skill 名称` |
| 会过期吗？ | 会，Curator 自动管理生命周期 |