# Go 语言入门介绍

## 概述

Go（又称 Golang）是 Google 于 2009 年发布的开源编程语言，由 Robert Griesemer、Rob Pike 和 Ken Thompson（C 语言和 Unix 之父之一）设计。目标是解决 Google 内部大规模分布式系统开发中的痛点：编译慢、依赖管理混乱、并发编程复杂。

---

## 一、核心特点

| 特点 | 说明 |
|------|------|
| **编译型** | 编译成单一二进制文件，无需运行时依赖 |
| **静态类型** | 编译时检查类型，安全可靠 |
| **并发原生** | goroutine + channel，语言级并发支持 |
| **垃圾回收** | 自动内存管理，不需要手动 free |
| **简洁语法** | 只有 25 个关键字，一天可入门 |
| **跨平台** | 交叉编译，写一次到处跑 |
| **标准库强大** | 内置 HTTP 服务器、JSON、加密、测试等 |

---

## 二、Go 的世界在哪里

```
Go 擅长的领域：
├── 云原生基础设施
│   ├── Docker / containerd    ← 容器引擎
│   ├── Kubernetes              ← 容器编排
│   ├── Prometheus              ← 监控
│   ├── Terraform               ← IaC
│   └── Istio / Consul          ← 服务网格
│
├── 后端服务 / API
│   ├── 高并发 HTTP 服务
│   ├── gRPC 微服务
│   └── 实时通信（WebSocket）
│
├── CLI 工具
│   ├── gh (GitHub CLI)
│   ├── kubectl
│   └── 各种 DevOps 工具
│
└── 网络编程
    ├── 代理 / 网关
    ├── 消息队列
    └── 数据库中间件
```

**一句话：Go 是云原生时代的 C 语言。**

---

## 三、语法速览

### Hello World

```go
package main

import "fmt"

func main() {
    fmt.Println("Hello, 世界")
}
```

### 变量与类型

```go
// 变量声明
var name string = "Go"
age := 14                    // 短声明，自动推断类型

// 基本类型
var i int = 42
var f float64 = 3.14
var b bool = true
var s string = "hello"

// 复合类型
arr := [3]int{1, 2, 3}      // 数组（固定长度）
slice := []int{1, 2, 3}     // 切片（动态数组）
m := map[string]int{"a": 1} // 映射
```

### 函数

```go
// 普通函数
func add(a, b int) int {
    return a + b
}

// 多返回值（Go 的特色）
func divide(a, b int) (int, error) {
    if b == 0 {
        return 0, fmt.Errorf("division by zero")
    }
    return a / b, nil
}

// 使用
result, err := divide(10, 2)
if err != nil {
    // 处理错误
}
```

### 结构体与方法

```go
type Server struct {
    Host string
    Port int
}

// 给结构体绑定方法
func (s Server) Address() string {
    return fmt.Sprintf("%s:%d", s.Host, s.Port)
}
```

### 接口

```go
// 接口定义行为，而不是继承
type Writer interface {
    Write([]byte) (int, error)
}

// 任何实现了 Write 方法的类型都实现了 Writer 接口
// Go 的接口是隐式实现的（不需要显式声明 implements）
```

### 并发（Go 的灵魂）

```go
// goroutine — 轻量级线程，一个 Go 程序可以轻松跑上万个
go func() {
    fmt.Println("running in background")
}()

// channel — goroutine 之间通信
ch := make(chan string)

go func() {
    ch <- "hello from goroutine"  // 发送
}()

msg := <-ch  // 接收
fmt.Println(msg)

// select — 多路复用
select {
case msg := <-ch1:
    fmt.Println(msg)
case msg := <-ch2:
    fmt.Println(msg)
case <-time.After(1 * time.Second):
    fmt.Println("timeout")
}
```

### HTTP 服务器（标准库就够了）

```go
package main

import (
    "encoding/json"
    "net/http"
)

func main() {
    http.HandleFunc("/hello", func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("Content-Type", "application/json")
        json.NewEncoder(w).Encode(map[string]string{
            "message": "Hello, World!",
        })
    })

    http.ListenAndServe(":8080", nil)
}
```

不需要 Nginx，不需要 Tomcat，**一个二进制文件就是完整的服务器**。

---

## 四、Go vs 其他语言

| 特性 | Go | Python | Java | Rust |
|------|:--:|:--:|:--:|:--:|
| 编译/解释 | 编译 | 解释 | 编译(JIT) | 编译 |
| 并发模型 | goroutine | asyncio | 线程 | async/线程 |
| 内存管理 | GC | GC | GC | 所有权 |
| 学习曲线 | ⭐⭐ | ⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ |
| 执行速度 | 快 | 慢 | 快 | 最快 |
| 开发速度 | 快 | 最快 | 慢 | 慢 |
| 二进制大小 | 大(~10MB) | — | — | 小 |
| 适合场景 | 系统/云原生 | 脚本/AI | 企业应用 | 系统/安全 |

---

## 五、常用命令

```bash
# 安装 Go（macOS）
brew install go

# 初始化模块
go mod init myproject

# 运行
go run main.go

# 编译
go build -o myapp main.go

# 交叉编译（在 macOS 上编译 Linux 二进制）
GOOS=linux GOARCH=amd64 go build -o myapp main.go

# 安装依赖
go get github.com/gin-gonic/gin

# 整理依赖
go mod tidy

# 测试
go test ./...

# 格式化
go fmt ./...

# 代码检查
go vet ./...
```

---

## 六、常用框架和库

| 类别 | 推荐 | 说明 |
|------|------|------|
| **Web 框架** | Gin / Echo / Fiber | 高性能 HTTP 框架 |
| **ORM** | GORM | 最流行的 Go ORM |
| **gRPC** | 标准库 + protobuf | 微服务通信首选 |
| **配置管理** | Viper | 读取 YAML/JSON/ENV 配置 |
| **日志** | Zap / Zerolog | 高性能结构化日志 |
| **CLI 工具** | Cobra | kubectl、gh 等都在用 |
| **测试** | 标准库 testing + testify | 单元测试 + 断言 |
| **消息队列** | RabbitMQ / Kafka Go SDK | — |
| **Redis** | go-redis | — |

---

## 七、Go 的哲学

> **"Less is more" — 少即是多**

1. **没有类继承**，用组合代替继承
2. **没有异常**，用多返回值处理错误
3. **没有泛型宏**（Go 1.18 才加入泛型，简单克制）
4. **没有运行时反射滥用**，编译时确定一切
5. **gofmt 强制统一格式**，团队代码风格天然一致（没有"我的代码格式更好"之争）

Go 的设计哲学是**限制你的表达方式，让代码更可读、更可维护**。这恰恰是大型团队协作最需要的。

---

## 八、学习路径

```
1. 基础语法（1~2 天）
   ├── 变量、类型、函数
   ├── 结构体、接口
   └── 错误处理
      │
      ▼
2. 并发编程（2~3 天）
   ├── goroutine + channel
   ├── select 多路复用
   └── sync 包（Mutex、WaitGroup）
      │
      ▼
3. 标准库（2~3 天）
   ├── net/http（写 HTTP 服务）
   ├── encoding/json
   ├── io / os（文件操作）
   └── testing（单元测试）
      │
      ▼
4. 实战项目（1~2 周）
   ├── 写一个 RESTful API 服务
   ├── 写一个 CLI 工具
   └── 写一个简单的代理/网关
      │
      ▼
5. 进阶
   ├── gRPC 微服务
   ├── Go 内存模型、pprof 性能分析
   └── 阅读 Kubernetes / Docker 源码
```

---

## 九、推荐资源

| 资源 | 说明 |
|------|------|
| [Go 官方教程](https://go.dev/tour/) | 交互式入门，必看 |
| [Effective Go](https://go.dev/doc/effective_go) | 最佳实践，Go 程序员必读 |
| [Go by Example](https://gobyexample.com/) | 代码示例大全 |
| 《Go 语言圣经》 | 经典入门书 |

---

*文档创建时间：2026-07-02*