# Python 入门介绍

## 概述

Python 由 Guido van Rossum 于 1991 年发布，是一种**解释型、动态类型、面向对象**的高级编程语言。设计哲学强调代码可读性和简洁语法，被誉为"可执行的伪代码"。

---

## 一、核心特点

| 特点 | 说明 |
|------|------|
| **解释型** | 不需要编译，写完直接运行 |
| **动态类型** | 变量不需要声明类型，运行时自动推断 |
| **简洁优雅** | 用缩进定义代码块，括号党退散 |
| **胶水语言** | 轻松调用 C/C++ 库，几乎所有语言都能和 Python 交互 |
| **生态丰富** | 40 万+ PyPI 包，覆盖所有领域 |
| **跨平台** | Windows / macOS / Linux 都能跑 |

---

## 二、Python 的世界在哪里

```
Python 擅长的领域：
├── 人工智能 / 机器学习
│   ├── PyTorch / TensorFlow    ← 深度学习框架
│   ├── scikit-learn            ← 传统 ML
│   ├── HuggingFace             ← 大模型/NLP
│   └── LangChain / LlamaIndex  ← AI 应用开发
│
├── 数据科学
│   ├── NumPy / Pandas          ← 数据处理
│   ├── Matplotlib / Seaborn    ← 可视化
│   └── Jupyter                 ← 交互式分析
│
├── 后端开发
│   ├── Django / FastAPI / Flask ← Web 框架
│   └── Celery                   ← 异步任务队列
│
├── 自动化 & DevOps
│   ├── 运维脚本
│   ├── Ansible（Python 写的）
│   └── SaltStack
│
├── 爬虫
│   ├── Scrapy / BeautifulSoup
│   └── requests + selenium
│
└── 其他
    ├── 科学计算（SciPy、SymPy）
    ├── 渗透测试（Python 是黑客标配）
    └── 教育（全球入门编程首选语言）
```

**一句话：Python 是瑞士军刀，除了操作系统内核和 3D 游戏引擎，什么都能干。**

---

## 三、语法速览

### Hello World

```python
print("Hello, 世界")
```

### 变量与类型

```python
# 变量直接赋值，不需要声明类型
name = "Python"
age = 33
pi = 3.14
is_cool = True

# 基本类型
# int、float、bool、str、None

# 复合类型
nums = [1, 2, 3]           # 列表（动态数组）
point = (10, 20)            # 元组（不可变）
user = {"name": "张三", "age": 25}  # 字典（映射）
tags = {"python", "go"}     # 集合（去重）
```

### 函数

```python
def greet(name, greeting="你好"):
    """向某人打招呼"""
    return f"{greeting}，{name}！"

print(greet("小明"))                      # 你好，小明！
print(greet("小明", greeting="早上好"))    # 早上好，小明！

# 多返回值
def divide(a, b):
    if b == 0:
        return None, "除数不能为零"
    return a / b, None

result, error = divide(10, 2)
```

### 条件与循环

```python
# if-elif-else
score = 85
if score >= 90:
    grade = "A"
elif score >= 80:
    grade = "B"
else:
    grade = "C"

# for 循环
for i in range(5):          # 0, 1, 2, 3, 4
    print(i)

for key, value in user.items():
    print(f"{key}: {value}")

# 列表推导式（Python 的特色）
squares = [x**2 for x in range(10)]        # [0, 1, 4, 9, ..., 81]
evens = [x for x in range(20) if x % 2 == 0]
```

### 类与对象

```python
class Server:
    def __init__(self, host, port=80):
        self.host = host
        self.port = port

    def address(self):
        return f"{self.host}:{self.port}"

    def __str__(self):
        return f"Server({self.address()})"

s = Server("192.168.1.1", 8080)
print(s.address())  # 192.168.1.1:8080
print(s)            # Server(192.168.1.1:8080)
```

### 异常处理

```python
try:
    result = 10 / 0
except ZeroDivisionError as e:
    print(f"出错了：{e}")
else:
    print("没有异常时执行")
finally:
    print("无论如何都会执行")

# 自定义异常
class ConfigError(Exception):
    pass

raise ConfigError("配置文件格式错误")
```

### 装饰器（Python 独有特色）

```python
import time

def timer(func):
    """测量函数执行时间的装饰器"""
    def wrapper(*args, **kwargs):
        start = time.time()
        result = func(*args, **kwargs)
        print(f"{func.__name__} 耗时 {time.time() - start:.2f}s")
        return result
    return wrapper

@timer
def slow_function():
    time.sleep(1)
    return "done"

slow_function()  # slow_function 耗时 1.00s
```

### 上下文管理器（with 语句）

```python
# 不用手动关闭文件
with open("data.txt", "r") as f:
    content = f.read()
# 自动关闭，即使发生异常

# 数据库连接
with sqlite3.connect("db.sqlite") as conn:
    conn.execute("SELECT * FROM users")
```

### 并发

```python
# asyncio — 异步 IO（适合 IO 密集型）
import asyncio

async def fetch(url):
    # 模拟网络请求
    await asyncio.sleep(1)
    return f"data from {url}"

async def main():
    # 并发执行多个请求
    results = await asyncio.gather(
        fetch("url1"),
        fetch("url2"),
        fetch("url3"),
    )
    print(results)

asyncio.run(main())

# 多线程 — 适合 IO 密集型
from concurrent.futures import ThreadPoolExecutor

with ThreadPoolExecutor(max_workers=5) as executor:
    results = executor.map(fetch, urls)

# 多进程 — 适合 CPU 密集型
from multiprocessing import Pool

with Pool(processes=4) as pool:
    results = pool.map(heavy_computation, data)
```

### FastAPI 示例（现代 Python Web 框架）

```python
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()

class Item(BaseModel):
    name: str
    price: float

@app.get("/")
def hello():
    return {"message": "Hello, World!"}

@app.post("/items")
def create_item(item: Item):
    return {"name": item.name, "price": item.price}
```

---

## 四、Python vs 其他语言

| 特性 | Python | Go | Java | JavaScript |
|------|:--:|:--:|:--:|:--:|
| 类型系统 | 动态 | 静态 | 静态 | 动态 |
| 执行速度 | 慢 | 快 | 快 | 中等 |
| 开发速度 | 最快 | 快 | 慢 | 快 |
| 并发模型 | asyncio/多线程 | goroutine | 线程 | 事件循环 |
| GIL | 有 | 无 | 无 | 无 |
| 学习曲线 | ⭐ | ⭐⭐ | ⭐⭐⭐ | ⭐⭐ |
| 适合场景 | AI/脚本/原型 | 系统/云原生 | 企业应用 | 前端/全栈 |
| 包管理 | pip/uv/poetry | go mod | maven/gradle | npm/yarn |

> **GIL（全局解释器锁）**：Python 的痛。多线程无法利用多核 CPU（需要多进程绕过）。但 Python 3.13 开始支持禁用 GIL（实验性）。

---

## 五、常用命令

```bash
# 安装 Python（macOS 自带，但建议用 pyenv 管理版本）
brew install pyenv
pyenv install 3.12
pyenv global 3.12

# 运行脚本
python3 script.py

# 交互式 REPL
python3

# 安装包
pip install requests
pip install -r requirements.txt

# 虚拟环境
python3 -m venv venv
source venv/bin/activate

# 现代化包管理（推荐用 uv，更快）
uv pip install requests
uv sync                    # 安装所有依赖

# 格式化
black *.py                 # 代码格式化
ruff check *.py            # 代码检查

# 测试
pytest test_*.py
pytest -v                  # 详细输出
```

---

## 六、常用框架和库

| 类别 | 推荐 | 说明 |
|------|------|------|
| **Web 框架** | FastAPI / Django / Flask | FastAPI 是新宠，Django 是全栈，Flask 是轻量 |
| **ORM** | SQLAlchemy / Django ORM | 数据库操作 |
| **异步** | asyncio / httpx / aiohttp | 异步 HTTP 请求 |
| **数据科学** | NumPy / Pandas / Polars | 数据处理三件套 |
| **机器学习** | PyTorch / scikit-learn | 深度学习 + 传统 ML |
| **可视化** | Matplotlib / Plotly / Seaborn | 图表 |
| **爬虫** | Scrapy / BeautifulSoup / playwright | 网页抓取 |
| **CLI 工具** | Click / Typer | 命令行工具 |
| **测试** | pytest | 测试框架标配 |
| **配置** | pydantic / pydantic-settings | 类型安全配置 |
| **任务队列** | Celery / arq | 异步任务 |

---

## 七、Python 的哲学

> **"There should be one—and preferably only one—obvious way to do it."**
> **"应该有一种——最好是唯一一种——显而易见的做法。"**

在 Python 交互环境中输入 `import this` 可以看到完整的「Python 之禅」：

```
Beautiful is better than ugly.        优美胜于丑陋
Explicit is better than implicit.     显式胜于隐式
Simple is better than complex.        简单胜于复杂
Complex is better than complicated.   复杂胜于繁琐
Readability counts.                   可读性很重要
...
```

**核心原则：**
1. **缩进即语法** → 强迫代码整洁
2. **鸭子类型** → "如果它走路像鸭子，叫起来像鸭子，那它就是鸭子"
3. **一切皆对象** → 函数、类、模块都是对象，可以传递和赋值
4. **"Batteries included"** → 标准库自带 HTTP 服务器、JSON、SQLite、正则、日志、单元测试等

---

## 八、学习路径

```
1. 基础语法（1~2 天）
   ├── 变量、类型、运算符
   ├── 条件、循环、列表推导式
   └── 函数、参数、作用域
      │
      ▼
2. 数据结构（2~3 天）
   ├── 列表、元组、字典、集合
   ├── 字符串操作
   └── 文件读写（with 语句）
      │
      ▼
3. 面向对象（2~3 天）
   ├── 类、继承、多态
   ├── 魔术方法（__str__, __init__ 等）
   └── 装饰器、上下文管理器
      │
      ▼
4. 标准库（3~5 天）
   ├── os / sys / pathlib（文件系统）
   ├── json / csv / xml（数据格式）
   ├── datetime / collections / itertools
   └── logging（日志）
      │
      ▼
5. 实战项目（1~2 周）
   ├── 写一个 RESTful API（FastAPI）
   ├── 写一个爬虫
   └── 写一个 CLI 工具
      │
      ▼
6. 进阶（按需）
   ├── 异步编程（asyncio）
   ├── 并发编程（多线程/多进程）
   ├── 性能优化（Cython/Numba）
   └── AI/ML（PyTorch/Numpy）
```

---

## 九、推荐资源

| 资源 | 说明 |
|------|------|
| [Python 官方教程](https://docs.python.org/zh-cn/3/tutorial/) | 官方中文教程，入门首选 |
| [Real Python](https://realpython.com/) | 高质量 Python 教程 |
| 《流畅的 Python》 | 进阶必读，深入 Python 语言特性 |
| [Python 3 Module of the Week](https://pymotw.com/3/) | 标准库详解 |

---

*文档创建时间：2026-07-02*