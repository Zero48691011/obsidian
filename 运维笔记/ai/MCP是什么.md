# MCP 是什么

## 一句话解释

**MCP（Model Context Protocol，模型上下文协议）** 是 Anthropic 提出的一套开放标准协议，让 AI 模型能够安全、标准化地连接外部工具和数据源。你可以把它理解为「AI 的 USB-C 接口」——不管什么工具，只要遵循 MCP 协议，AI 就能直接调用。

---

## 核心概念

### 传统方式 vs MCP

```
传统方式（每个工具单独接入）：
  AI ─── 自定义代码 ─── GitHub API
  AI ─── 自定义代码 ─── 数据库
  AI ─── 自定义代码 ─── 文件系统
  AI ─── 自定义代码 ─── Slack
  （每个集成都要写一套代码，M×N 问题）

MCP 方式（统一协议）：
  AI ─── MCP 客户端 ───┬─ MCP Server (GitHub)
                       ├─ MCP Server (数据库)
                       ├─ MCP Server (文件系统)
                       └─ MCP Server (Slack)
  （只需实现一次客户端，服务端遵循协议即可，M+N 问题）
```

### 三个角色

| 角色 | 说明 | 类比 |
|------|------|------|
| **MCP Host** | 运行 AI 的程序（如 Hermes Agent、Claude Desktop） | 电脑主机 |
| **MCP Client** | Host 内部的协议客户端，负责与服务端通信 | USB 控制器 |
| **MCP Server** | 提供具体能力的服务进程（如 GitHub 操作、文件读写） | USB 外设 |

---

## 两种传输方式

### 1. Stdio（标准输入输出）

最常见的方式。AI 程序启动一个 MCP Server 作为子进程，通过 stdin/stdout 通信。

```yaml
# 示例：通过 npx 启动 GitHub MCP Server
mcp_servers:
  github:
    command: "npx"
    args: ["-y", "@modelcontextprotocol/server-github"]
    env:
      GITHUB_PERSONAL_ACCESS_TOKEN: "ghp_xxx"
```

### 2. HTTP（StreamableHTTP）

远程或共享的 MCP Server，通过网络访问。

```yaml
mcp_servers:
  remote_api:
    url: "https://mcp.example.com/mcp"
    headers:
      Authorization: "Bearer sk-xxx"
```

---

## MCP Server 能做什么

MCP Server 可以暴露三种能力：

| 能力 | 说明 | 示例 |
|------|------|------|
| **Tools（工具）** | AI 可以调用的函数 | 读文件、发消息、查数据库 |
| **Resources（资源）** | 可读取的数据 | 文档内容、配置信息 |
| **Prompts（提示模板）** | 预定义的提示词 | 代码审查模板、翻译模板 |

---

## Hermes Agent 中的 MCP

Hermes Agent 内置了 MCP 客户端，配置方式：

### 1. 安装依赖

```bash
pip install mcp
```

### 2. 配置 Server

在 `~/.hermes/config.yaml` 中添加：

```yaml
mcp_servers:
  time:
    command: "uvx"
    args: ["mcp-server-time"]
```

### 3. 重启 Hermes

启动后会自动：
- 连接所有 MCP Server
- 发现每个 Server 提供的工具
- 注册为 `mcp_{server名}_{工具名}` 格式
- 注入到所有对话中，可直接调用

### 工具命名规则

```
mcp_{server_name}_{tool_name}
```

例如：
- Server `filesystem`，工具 `read_file` → `mcp_filesystem_read_file`
- Server `github`，工具 `list-issues` → `mcp_github_list_issues`

---

## 常见 MCP Server 示例

| Server | 用途 | 安装命令 |
|--------|------|----------|
| `mcp-server-time` | 获取当前时间 | `uvx mcp-server-time` |
| `server-filesystem` | 文件系统操作 | `npx @modelcontextprotocol/server-filesystem /path` |
| `server-github` | GitHub 操作 | `npx @modelcontextprotocol/server-github` |
| `server-postgres` | PostgreSQL 查询 | `npx @modelcontextprotocol/server-postgres` |
| `server-slack` | Slack 消息 | `npx @modelcontextprotocol/server-slack` |

---

## 安全性

- **环境变量过滤**：MCP 子进程默认只能访问 `PATH`、`HOME`、`USER` 等安全变量，不会泄露 API 密钥
- **需要显式授权**：敏感变量（如 `GITHUB_TOKEN`）必须在配置中明确声明才会传给 Server
- **凭证脱敏**：错误信息中的密钥类字符串会自动打码

---

## 总结

| 问题 | 答案 |
|------|------|
| MCP 是什么？ | AI 连接外部工具的统一协议标准 |
| 谁定义的？ | Anthropic，开源 |
| 解决什么问题？ | 避免每个工具都要单独写集成代码 |
| 怎么用？ | 配置 Server → 重启 → AI 自动发现工具 |
| 和 API 的区别？ | API 是具体接口，MCP 是协议层，MCP Server 内部可以用 API 实现 |