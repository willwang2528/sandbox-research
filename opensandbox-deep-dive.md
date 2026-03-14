# Alibaba OpenSandbox 深度解析

> 基于 MarkTechPost 报道，结合官方源码与文档的全面技术分析

---

## 摘要

Alibaba  released OpenSandbox，一个开源工具，旨在为 AI Agent 提供安全的隔离环境，用于代码执行、网页浏览和模型训练。该项目采用 Apache 2.0 许可证发布，目标是标准化 AI Agent 技术栈的"执行层"，提供跨多种编程语言和基础设施提供商的统一 API。

---

## 1. Agent 工作流中的技术缺口

### 1.1 问题背景

构建自主 Agent 通常涉及两个核心组件：

| 组件 | 描述 | 典型实现 |
|------|------|----------|
| **Brain（大脑）** | 大型语言模型 | LLM（如 Claude、Gemini、GPT） |
| **Tools（工具）** | 代码执行、网页访问、文件操作 | 需要安全隔离环境 |

**传统方案的痛点：**
- 开发者需手动配置 Docker 容器
- 管理复杂的网络隔离
- 依赖第三方收费 API 服务

### 1.2 OpenSandbox 的解决方案

OpenSandbox 提供标准化、安全的环境，使 Agent 能够：
- ✅ 执行任意代码
- ✅ 与图形界面交互
- ✅ 不危及主机系统完整性
- ✅ 通过单一 API 从本地开发平滑迁移到生产级部署

---

## 2. 架构设计

### 2.1 四层架构概览

```
┌─────────────────────────────────────────────────────────────┐
│                    SDKs Layer                               │
│  (Python / Java / TypeScript / Go / C#)                     │
├─────────────────────────────────────────────────────────────┤
│                    Specs Layer                              │
│  (Sandbox Lifecycle API + Execution API - OpenAPI 规范)      │
├─────────────────────────────────────────────────────────────┤
│                    Runtime Layer                            │
│  (FastAPI Server + Docker/Kubernetes 运行时)                 │
├─────────────────────────────────────────────────────────────┤
│                 Sandbox Instances Layer                     │
│  (隔离容器 + execd 守护进程 + Jupyter 内核)                   │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 各层详细说明

#### 2.2.1 SDKs Layer（客户端库层）

**位置**: `sdks/`

提供多语言 SDK，封装与沙箱交互的高级抽象：

| 组件 | 功能 | 源码位置 |
|------|------|----------|
| **Sandbox** | 生命周期管理（创建/监控/销毁） | `sdks/sandbox/` |
| **Filesystem** | 文件 CRUD、批量操作、权限管理 | `sdks/sandbox/` |
| **Commands** |  shell 命令执行、前台/后台模式 | `sdks/sandbox/` |
| **CodeInterpreter** | 多语言有状态代码执行 | `sdks/code-interpreter/` |

**支持语言**:
- ✅ Python (已发布)
- ✅ Java/Kotlin (已发布)
- 🔄 TypeScript (路线图)
- 🔄 Go (路线图)
- 🔄 C# (路线图)

#### 2.2.2 Specs Layer（规范层）

**位置**: `specs/`

定义两个核心 OpenAPI 规范：

**1. Sandbox Lifecycle Spec** (`specs/sandbox-lifecycle.yml`)

| 操作 | 端点 | 描述 |
|------|------|------|
| Create | `POST /sandboxes` | 从容器镜像创建沙箱 |
| List | `GET /sandboxes` | 带过滤和分页的沙箱列表 |
| Get | `GET /sandboxes/{id}` | 获取沙箱详情和状态 |
| Delete | `DELETE /sandboxes/{id}` | 终止沙箱 |
| Pause | `POST /sandboxes/{id}/pause` | 暂停运行中的沙箱 |
| Resume | `POST /sandboxes/{id}/resume` | 恢复暂停的沙箱 |
| Renew | `POST /sandboxes/{id}/renew-expiration` | 延长 TTL |
| Endpoint | `GET /sandboxes/{id}/endpoints/{port}` | 获取端口公网 URL |

**2. Sandbox Execution Spec** (`specs/execd-api.yaml`)

定义 execd 守护进程实现的 API：

| 类别 | 端点 | 描述 |
|------|------|------|
| **Health** | `GET /ping` | 健康检查 |
| **Code Interpreting** | `POST /code/context` | 创建执行上下文 |
| | `POST /code` | 执行代码（流式输出） |
| | `DELETE /code` | 中断代码执行 |
| **Command Execution** | `POST /command` | 执行 shell 命令 |
| | `DELETE /command` | 中断命令 |
| **Filesystem** | `GET/POST/DELETE /files/*` | 文件操作（CRUD/搜索/权限） |
| **Metrics** | `GET /metrics` | 获取系统指标快照 |
| | `GET /metrics/watch` | 通过 SSE 流式推送指标 |

#### 2.2.3 Runtime Layer（运行时层）

**位置**: `server/`

基于 FastAPI 的服务，核心功能：

- **生命周期管理**: 创建、监控、暂停、恢复、终止沙箱
- **可插拔运行时**: Docker（生产就绪）、Kubernetes（生产就绪）
- **异步配置**: 后台创建以降低延迟
- **自动过期**: 可配置 TTL，支持续期
- **访问控制**: API Key 认证
- **可观测性**: 统一状态跟踪与转换日志

**Docker 运行时特性**:
- 直接 Docker API 集成
- 两种网络模式：
  - **Host Mode**: 容器共享主机网络（单实例）
  - **Bridge Mode**: 隔离网络 + HTTP 路由
- 容器生命周期管理
- 资源配额强制执行
- 私有仓库认证
- execd 注入的卷挂载
- 过期自动清理

**Kubernetes 运行时特性**:
- 基于 Operator 模式
- 支持 Pool（资源池）预热
- 自动扩缩容
- PVC 持久化存储
- 多租户隔离

#### 2.2.4 Sandbox Instances Layer（沙箱实例层）

每个隔离容器内注入的核心组件：

| 组件 | 描述 | 技术实现 |
|------|------|----------|
| **execd** | 高性能 Go 守护进程 | 命令执行、文件操作、指标采集 |
| **Jupyter Kernel** | 多语言代码解释器 | Python/Java/Go/TypeScript 有状态执行 |
| **bootstrap.sh** | 启动脚本 | execd 初始化与注册 |

---

## 3. 核心技术能力

### 3.1 环境无关性设计

OpenSandbox 设计为环境无关：
- **本地开发**: Docker 运行时
- **生产部署**: Kubernetes 运行时（分布式、生产级）

### 3.2 四种沙箱类型

| 类型 | 用途 | 示例镜像 |
|------|------|----------|
| **Coding Agents** | 代码编写、测试、调试 | `code-interpreter` |
| **GUI Agents** | 图形界面交互（VNC 桌面） | `desktop` |
| **Code Execution** | 高性能脚本执行 | `code-interpreter` |
| **RL Training** | 强化学习安全迭代训练 | `rl-training` |

### 3.3 代码执行能力详解

#### 3.3.1 Code Interpreter SDK

**源码示例**: `examples/code-interpreter/main.py`

```python
from opensandbox import Sandbox
from opensandbox_code_interpreter import CodeInterpreter

async def main():
    # 创建沙箱 + 代码解释器
    async with Sandbox() as sandbox:
        interpreter = CodeInterpreter(sandbox)
        
        # Python 示例
        result = await interpreter.run_python("print('Hello from Python!')\n3.14 + 0.002")
        print(f"Python result: {result}")
        
        # Java 示例
        result = await interpreter.run_java("""
            public class Main {
                public static void main(String[] args) {
                    System.out.println("Hello from Java!");
                    System.out.println("2 + 3 = " + (2 + 3));
                }
            }
        """)
        
        # Go 示例
        result = await interpreter.run_go("""
            package main
            import "fmt"
            func main() {
                fmt.Println("Hello from Go!")
                fmt.Printf("3 + 4 = %d\\n", 3 + 4)
            }
        """)
        
        # TypeScript 示例
        result = await interpreter.run_typescript("""
            const sum = 2 + 4;
            console.log("Hello from TypeScript!");
            console.log(`sum = ${sum}`);
        """)
```

**关键特性**:
- ✅ **多语言支持**: Python, Java, JavaScript, TypeScript, Go, Bash
- ✅ **会话管理**: 多次调用间变量持久化
- ✅ **Jupyter 集成**: 基于 Jupyter 内核协议
- ✅ **结果流式推送**: 通过 SSE 实时输出
- ✅ **错误处理**: 结构化错误响应（含堆栈跟踪）

**使用场景**:
- 交互式编程环境（如 Jupyter Notebook）
- AI 代码生成与执行
- 数据分析与可视化
- 教育编程平台

#### 3.3.2 预置镜像

官方提供多种场景镜像：

| 镜像 | 用途 | 拉取命令 |
|------|------|----------|
| `code-interpreter` | 代码解释器 | `docker pull opensandbox/code-interpreter:v1.0.1` |
| `chrome` | 浏览器自动化 | `docker pull opensandbox/chrome:v1.0.1` |
| `desktop` | 完整 VNC 桌面 | `docker pull opensandbox/desktop:v1.0.1` |
| `vscode` | 远程开发环境 | `docker pull opensandbox/vscode:v1.0.1` |

---

## 4. 整合与生态系统支持

### 4.1 模型接口集成

OpenSandbox 原生兼容主流 AI 模型接口：

| 集成 | 描述 | 示例位置 |
|------|------|----------|
| **Claude Code** | Anthropic 代码助手 | `examples/claude-code/` |
| **Gemini CLI** | Google Gemini 命令行 | `examples/gemini-cli/` |
| **OpenAI Codex** | OpenAI 代码模型 | `examples/codex-cli/` |
| **Kimi CLI** | 月之暗面 Kimi | `examples/kimi-cli/` |

### 4.2 编排框架集成

#### 4.2.1 LangGraph 集成

**源码示例**: `examples/langgraph/main.py`

LangGraph 是一个图驱动的 Agent 工作流框架。OpenSandbox 与之集成，实现显式状态机控制流：

```python
from langgraph.graph import StateGraph, END
from typing import TypedDict, Annotated
from opensandbox import Sandbox

# 定义状态
class AgentState(TypedDict):
    sandbox_id: str
    command: str
    output: str
    error: str
    retry_count: int

# 构建图
workflow = StateGraph(AgentState)

# 节点 1: 创建沙箱
async def create_sandbox(state):
    sandbox = await Sandbox.create()
    return {"sandbox_id": sandbox.id}

# 节点 2: 执行命令
async def run_command(state):
    try:
        # 执行代码
        result = await sandbox.commands.run(state["command"])
        return {"output": result.stdout}
    except Exception as e:
        return {"error": str(e)}

# 节点 3: 决策（失败时重试）
def should_retry(state):
    if state["error"] and state["retry_count"] < 3:
        return "retry"
    return "cleanup"

# 添加节点和边
workflow.add_node("create", create_sandbox)
workflow.add_node("run", run_command)
workflow.add_node("cleanup", cleanup_sandbox)

workflow.add_edge("create", "run")
workflow.add_conditional_edges("run", should_retry, {
    "retry": "run",
    "cleanup": "cleanup"
})
workflow.add_edge("cleanup", END)

# 运行图
app = workflow.compile()
result = await app.ainvoke({"command": "python3 script.py", "retry_count": 0})
```

**工作流**:
1. 创建沙箱
2. 准备环境（安装依赖）
3. 执行任务
4. 失败时决策重试（默认 `python` vs `python3` 回退）
5. 用 Claude 总结结果
6. 清理沙箱实例

#### 4.2.2 Google ADK 集成

**源码示例**: `examples/google-adk/main.py`

Google Agent Development Kit (ADK) 是 Google 的 Agent 开发框架：

```python
from google.adk import Agent, Runner
from opensandbox import Sandbox

# 定义 OpenSandbox 工具
@tool
def write_file(path: str, content: str):
    """在沙箱中写入文件"""
    await sandbox.filesystem.write_file(path, content)

@tool
def read_file(path: str) -> str:
    """从沙箱读取文件"""
    return await sandbox.filesystem.read_file(path)

@tool
def run_in_sandbox(command: str) -> str:
    """在沙箱中执行命令"""
    result = await sandbox.commands.run(command)
    return result.stdout

# 创建 ADK Agent
agent = Agent(
    name="sandbox-agent",
    model="gemini-2.5-flash",
    tools=[write_file, read_file, run_in_sandbox]
)

# 运行
runner = Runner(agent=agent)
async for event in runner.run_stream("在沙箱中创建一个 Python 脚本并执行它"):
    if event.tool_call:
        print(f"Tool call: {event.tool_call.name}")
```

**功能**:
- ADK Agent 驱动工具调用
- 工具在沙箱内执行
- 支持文件操作和命令执行

#### 4.2.3 其他编排框架

| 框架 | 示例位置 | 描述 |
|------|----------|------|
| **LangGraph** | `examples/langgraph/` | 图驱动工作流 |
| **Google ADK** | `examples/google-adk/` | Google Agent 开发套件 |
| **Playwright** | `examples/playwright/` | 浏览器自动化 |

### 4.3 自动化工具集成

| 工具 | 用途 | 示例 |
|------|------|------|
| **Chrome** | 浏览器自动化 | `examples/chrome/` |
| **Playwright** | 跨浏览器自动化 | `examples/playwright/` |
| **VNC** | 可视化监控与交互 | `examples/desktop/` |

**典型工作流示例**:
```
Agent 任务："爬取网站并训练线性回归模型"

1. Agent 使用 Playwright 导航网页
2. 下载数据到沙箱本地文件系统
3. 执行 Python 代码处理数据
4. 训练模型并输出结果
→ 全程在 OpenSandbox 隔离环境中完成
```

---

## 5. 部署与配置

### 5.1 本地开发部署

**三步快速启动**:

```bash
# 1. 安装服务器组件
pip install opensandbox-server

# 2. 生成配置文件
opensandbox-server init-config ~/.sandbox.toml --example docker

# 3. 启动服务器
opensandbox-server
```

**环境变量**:
| 变量 | 默认值 | 描述 |
|------|--------|------|
| `SANDBOX_DOMAIN` | `localhost:8080` | 沙箱服务地址 |
| `SANDBOX_API_KEY` | (可选) | API Key 认证 |
| `SANDBOX_IMAGE` | `opensandbox/code-interpreter:v1.0.1` | 使用的镜像 |

### 5.2 生产环境部署（Kubernetes）

**Pool 资源配置示例**:

```yaml
apiVersion: sandbox.opensandbox.io/v1alpha1
kind: Pool
metadata:
  name: pool-sample
  namespace: opensandbox
spec:
  template:
    spec:
      initContainers:
        - name: execd-installer
          image: sandbox-registry.cn-zhangjiakou.cr.aliyuncs.com/opensandbox/execd:v1.0.6
          command: ["/bin/sh", "-c"]
          args:
            - |
              cp ./execd /opt/opensandbox/bin/execd && 
              chmod +x /opt/opensandbox/bin/execd
      containers:
        - name: sandbox
          image: opensandbox/code-interpreter:v1.0.1
  capacitySpec:
    bufferMin: 1      # 最小缓冲池
    bufferMax: 3      # 最大缓冲池
    poolMin: 0        # 最小池大小
    poolMax: 5        # 最大池大小
```

**关键优势**:
- ✅ 资源池预热（降低冷启动延迟）
- ✅ 自动扩缩容
- ✅ 分布式调度
- ✅ PVC 持久化存储

### 5.3 配置选项

**~/.sandbox.toml 示例**:

```toml
[runtime]
type = "docker"  # 或 "kubernetes"

[network]
mode = "bridge"  # 或 "host"

[security]
api_key = "your-api-key"
allowed_domains = ["pypi.org", "github.com"]  # FQDN 级别出口控制

[resources]
cpu_limit = "2"
memory_limit = "4Gi"
ttl_seconds = 3600  # 1 小时自动过期
```

---

## 6. 扩展指南：自定义 SDK 与 Runtime

### 6.1 用户自定义 SDK 语言实现步骤

OpenSandbox 采用**协议优先（Protocol-First）**设计，所有交互由 OpenAPI 规范定义。这意味着任何语言只需遵循 Specs 层的 API 契约，即可实现兼容的 SDK。

#### 6.1.1 SDK 核心结构（以 Python 为例）

**源码位置**: `sdks/sandbox/python/src/opensandbox/`

```
opensandbox/
├── __init__.py          # 包入口，导出 Sandbox 主类
├── sandbox.py           # 核心 Sandbox 类（生命周期管理）
├── config/
│   └── connection.py    # 连接配置（ baseURL, API Key, timeout）
├── models/
│   └── sandboxes.py     # 数据模型（SandboxInfo, Volume, NetworkPolicy 等）
├── services/
│   ├── sandbox.py       # 沙箱服务接口（抽象基类）
│   ├── filesystem.py    # 文件系统服务接口
│   ├── command.py       # 命令执行服务接口
│   ├── health.py        # 健康检查服务接口
│   └── metrics.py       # 指标采集服务接口
├── adapters/
│   ├── factory.py       # 适配器工厂（创建各服务实例）
│   ├── sandboxes_adapter.py
│   ├── filesystem_adapter.py
│   ├── command_adapter.py
│   ├── health_adapter.py
│   └── metrics_adapter.py
└── exceptions/
    └── __init__.py      # 自定义异常类
```

#### 6.1.2 实现步骤

**Step 1: 定义连接配置**

```python
# config/connection.py
from dataclasses import dataclass
import httpx

@dataclass
class ConnectionConfig:
    base_url: str
    api_key: str | None = None
    timeout: int = 300
    
    def create_client(self) -> httpx.AsyncClient:
        headers = {"Authorization": f"Bearer {self.api_key}"} if self.api_key else {}
        return httpx.AsyncClient(
            base_url=self.base_url,
            headers=headers,
            timeout=self.timeout
        )
```

**Step 2: 实现数据模型**

```python
# models/sandboxes.py
from dataclasses import dataclass
from datetime import datetime
from typing import Optional, Dict, Any

@dataclass
class SandboxImageSpec:
    image: str
    auth: Optional[Dict[str, str]] = None

@dataclass
class SandboxInfo:
    id: str
    status: str  # "pending" | "running" | "paused" | "terminated"
    image: str
    created_at: datetime
    expires_at: Optional[datetime]
    endpoints: Dict[str, str]
    resources: Dict[str, str]
```

**Step 3: 实现服务适配器**

```python
# adapters/sandboxes_adapter.py
from opensandbox.config import ConnectionConfig
from opensandbox.models.sandboxes import SandboxInfo, SandboxImageSpec

class SandboxesAdapter:
    def __init__(self, config: ConnectionConfig):
        self.config = config
    
    async def create(self, image: SandboxImageSpec, **kwargs) -> SandboxInfo:
        async with self.config.create_client() as client:
            response = await client.post(
                "/sandboxes",
                json={
                    "image": {"image": image.image, "auth": image.auth},
                    **kwargs
                }
            )
            response.raise_for_status()
            data = response.json()
            return SandboxInfo(**data)
    
    async def get(self, sandbox_id: str) -> SandboxInfo:
        async with self.config.create_client() as client:
            response = await client.get(f"/sandboxes/{sandbox_id}")
            response.raise_for_status()
            return SandboxInfo(**response.json())
    
    async def delete(self, sandbox_id: str) -> None:
        async with self.config.create_client() as client:
            response = await client.delete(f"/sandboxes/{sandbox_id}")
            response.raise_for_status()
```

**Step 4: 实现主 Sandbox 类**

```python
# sandbox.py
from opensandbox.adapters.factory import AdapterFactory
from opensandbox.config import ConnectionConfig
from opensandbox.models.sandboxes import SandboxImageSpec, SandboxInfo

class Sandbox:
    def __init__(self, sandbox_id: str, factory: AdapterFactory, info: SandboxInfo):
        self.id = sandbox_id
        self._factory = factory
        self._info = info
    
    @property
    def files(self):
        return self._factory.create_filesystem_service(self._info.endpoints)
    
    @property
    def commands(self):
        return self._factory.create_command_service(self._info.endpoints)
    
    @classmethod
    async def create(
        cls,
        image: str | SandboxImageSpec,
        base_url: str = "http://localhost:8080",
        api_key: str | None = None,
        **kwargs
    ) -> "Sandbox":
        config = ConnectionConfig(base_url=base_url, api_key=api_key)
        factory = AdapterFactory(config)
        sandbox_service = factory.create_sandbox_service()
        
        if isinstance(image, str):
            image = SandboxImageSpec(image=image)
        
        info = await sandbox_service.create(image, **kwargs)
        
        # 等待沙箱就绪
        await cls._wait_until_ready(info.id, factory)
        
        return cls(info.id, factory, info)
    
    @staticmethod
    async def _wait_until_ready(sandbox_id: str, factory: AdapterFactory, timeout: int = 300):
        import asyncio
        start = asyncio.get_event_loop().time()
        while True:
            health = factory.create_health_service(...)
            if await health.check():
                break
            if asyncio.get_event_loop().time() - start > timeout:
                raise TimeoutError("Sandbox readiness timeout")
            await asyncio.sleep(1)
    
    async def kill(self) -> None:
        await self._factory.create_sandbox_service().delete(self.id)
    
    async def close(self) -> None:
        await self.kill()
```

**Step 5: 实现代码解释器（可选，高级功能）**

```python
# code-interpreter 子包
from opensandbox import Sandbox

class CodeInterpreter:
    def __init__(self, sandbox: Sandbox):
        self.sandbox = sandbox
        self.context_id = None
    
    async def create_context(self, language: str = "python"):
        result = await self.sandbox.commands.post(
            "/code/context",
            json={"language": language}
        )
        self.context_id = result["context_id"]
    
    async def run_python(self, code: str):
        if not self.context_id:
            await self.create_context("python")
        
        result = await self.sandbox.commands.post(
            "/code",
            json={"context_id": self.context_id, "code": code}
        )
        return result["output"]
```

#### 6.1.3 新增语言 SDK 检查清单

| 步骤 | 任务 | 验收标准 |
|------|------|----------|
| 1 | 阅读 OpenAPI 规范 | 理解 `/sandboxes` 和 `/execd` 所有端点 |
| 2 | 实现 HTTP 客户端封装 | 支持 async/await、自动重试、超时处理 |
| 3 | 定义数据模型 | 与 OpenAPI Schema 完全一致 |
| 4 | 实现五个核心服务 | Sandbox/Filesystem/Commands/Health/Metrics |
| 5 | 实现 Sandbox 主类 | 提供 `create()`, `kill()`, `files`, `commands` 属性 |
| 6 | （可选）CodeInterpreter | 支持多语言有状态执行 |
| 7 | 编写测试 | 覆盖核心流程，通过官方测试套件 |

**参考实现**:
- Python SDK: `sdks/sandbox/python/`
- Kotlin SDK: `sdks/sandbox/kotlin/`
- C# SDK: `sdks/sandbox/csharp/`
- JavaScript SDK: `sdks/sandbox/javascript/`

---

### 6.2 用户自定义 Runtime 实现步骤

OpenSandbox 的 Runtime 层采用**可插拔设计**，通过抽象接口 `SandboxService` 定义生命周期操作，具体实现由后端（Docker/Kubernetes）提供。

#### 6.2.1 Runtime 核心接口

**源码位置**: `server/src/services/sandbox_service.py`

```python
from abc import ABC, abstractmethod
from src.api.schema import (
    CreateSandboxRequest,
    CreateSandboxResponse,
    ListSandboxesRequest,
    ListSandboxesResponse,
    RenewSandboxExpirationRequest,
    RenewSandboxExpirationResponse,
    Sandbox,
)

class SandboxService(ABC):
    """沙箱生命周期操作的抽象接口"""
    
    @staticmethod
    def generate_sandbox_id() -> str:
        """生成唯一沙箱 ID（UUID4）"""
        return str(uuid4())
    
    @abstractmethod
    def create_sandbox(self, request: CreateSandboxRequest) -> CreateSandboxResponse:
        """从容器镜像创建沙箱"""
        pass
    
    @abstractmethod
    def list_sandboxes(self, request: ListSandboxesRequest) -> ListSandboxesResponse:
        """带过滤和分页的沙箱列表"""
        pass
    
    @abstractmethod
    def get_sandbox(self, sandbox_id: str) -> Sandbox:
        """获取沙箱详情"""
        pass
    
    @abstractmethod
    def delete_sandbox(self, sandbox_id: str) -> None:
        """终止沙箱"""
        pass
    
    @abstractmethod
    def pause_sandbox(self, sandbox_id: str) -> None:
        """暂停沙箱"""
        pass
    
    @abstractmethod
    def resume_sandbox(self, sandbox_id: str) -> None:
        """恢复沙箱"""
        pass
    
    @abstractmethod
    def renew_expiration(
        self, sandbox_id: str, request: RenewSandboxExpirationRequest
    ) -> RenewSandboxExpirationResponse:
        """延长沙箱 TTL"""
        pass
```

#### 6.2.2 实现步骤

**Step 1: 创建新的 Runtime 实现类**

以实现 **containerd** 运行时为例：

```python
# server/src/services/containerd.py
import containerd
from src.services.sandbox_service import SandboxService
from src.api.schema import (
    CreateSandboxRequest,
    CreateSandboxResponse,
    Sandbox,
    SandboxStatus,
)
from src.config import AppConfig
from datetime import datetime, timedelta

class ContainerdSandboxService(SandboxService):
    """containerd 运行时实现"""
    
    def __init__(self, config: AppConfig):
        self.config = config
        self.containerd_client = containerd.Client(
            config.containerd.address  # 如 "/run/containerd/containerd.sock"
        )
        self._sandboxes = {}  # 内存存储沙箱状态
    
    def create_sandbox(self, request: CreateSandboxRequest) -> CreateSandboxResponse:
        # 1. 生成沙箱 ID
        sandbox_id = self.generate_sandbox_id()
        
        # 2. 拉取镜像
        image = self.containerd_client.pull(request.image.image)
        
        # 3. 准备 execd 注入（参考 DockerSandboxService）
        execd_archive = self._prepare_execd_archive()
        
        # 4. 创建容器
        container = self.containerd_client.create_container(
            image=image,
            id=sandbox_id,
            spec=self._build_container_spec(request, execd_archive),
        )
        
        # 5. 启动容器
        task = container.new_task()
        task.start()
        
        # 6. 记录沙箱状态
        expires_at = datetime.now() + timedelta(seconds=request.expiration_seconds)
        self._sandboxes[sandbox_id] = {
            "container": container,
            "task": task,
            "expires_at": expires_at,
            "status": SandboxStatus.RUNNING,
        }
        
        # 7. 返回响应
        return CreateSandboxResponse(
            sandbox_id=sandbox_id,
            status=SandboxStatus.RUNNING,
            endpoints=self._get_endpoints(container),
        )
    
    def _build_container_spec(self, request: CreateSandboxRequest, execd_archive):
        """构建 containerd 容器规格"""
        from containerd.specs import Spec
        
        return Spec(
            image=request.image.image,
            command=request.entrypoint or ["/opt/opensandbox/bootstrap.sh"],
            env=[f"{k}={v}" for k, v in request.environment.items()],
            resources=self._build_resources(request.resources),
            mounts=[
                # 注入 execd
                {
                    "type": "bind",
                    "source": execd_archive,
                    "destination": "/opt/opensandbox",
                    "options": ["ro"],
                },
                # 挂载卷
                *[{"type": "bind", "source": v.host_path, "destination": v.container_path}
                  for v in request.volumes],
            ],
        )
    
    def list_sandboxes(self, request: ListSandboxesRequest):
        # 实现过滤和分页逻辑
        sandboxes = list(self._sandboxes.values())
        
        # 应用过滤器
        if request.status:
            sandboxes = [s for s in sandboxes if s["status"] == request.status]
        if request.image:
            sandboxes = [s for s in sandboxes if s["image"] == request.image]
        
        # 分页
        start = (request.page - 1) * request.page_size
        end = start + request.page_size
        
        return ListSandboxesResponse(
            sandboxes=[self._to_sandbox_info(s) for s in sandboxes[start:end]],
            pagination={"total": len(sandboxes), "page": request.page},
        )
    
    def get_sandbox(self, sandbox_id: str) -> Sandbox:
        if sandbox_id not in self._sandboxes:
            raise HTTPException(404, f"Sandbox {sandbox_id} not found")
        return self._to_sandbox_info(self._sandboxes[sandbox_id])
    
    def delete_sandbox(self, sandbox_id: str) -> None:
        if sandbox_id in self._sandboxes:
            data = self._sandboxes[sandbox_id]
            data["task"].kill()
            data["container"].delete()
            del self._sandboxes[sandbox_id]
    
    def pause_sandbox(self, sandbox_id: str) -> None:
        data = self._sandboxes[sandbox_id]
        data["task"].pause()
        data["status"] = SandboxStatus.PAUSED
    
    def resume_sandbox(self, sandbox_id: str) -> None:
        data = self._sandboxes[sandbox_id]
        data["task"].resume()
        data["status"] = SandboxStatus.RUNNING
    
    def renew_expiration(self, sandbox_id: str, request):
        from datetime import timedelta
        self._sandboxes[sandbox_id]["expires_at"] += timedelta(seconds=request.extra_seconds)
        return RenewSandboxExpirationResponse(
            sandbox_id=sandbox_id,
            expires_at=self._sandboxes[sandbox_id]["expires_at"],
        )
    
    def _to_sandbox_info(self, data: dict) -> Sandbox:
        """转换为标准 Sandbox 响应对象"""
        return Sandbox(
            id=data["id"],
            status=data["status"],
            image=data["image"],
            created_at=data["created_at"],
            expires_at=data["expires_at"],
        )
```

**Step 2: 注册到工厂**

**源码位置**: `server/src/services/factory.py`

```python
def create_sandbox_service(
    service_type: str | None = None,
    config: AppConfig | None = None,
) -> SandboxService:
    active_config = config or get_config()
    selected_type = (service_type or active_config.runtime.type).lower()
    
    # 注册表
    implementations: dict[str, type[SandboxService]] = {
        "docker": DockerSandboxService,
        "kubernetes": KubernetesSandboxService,
        "containerd": ContainerdSandboxService,  # ← 新增
        # 未来可扩展:
        # "podman": PodmanSandboxService,
        # "lxc": LxcSandboxService,
    }
    
    if selected_type not in implementations:
        raise ValueError(f"Unsupported runtime: {selected_type}")
    
    return implementations[selected_type](config=active_config)
```

**Step 3: 添加配置支持**

**源码位置**: `server/src/config.py`

```python
from pydantic import BaseModel

class ContainerdConfig(BaseModel):
    address: str = "/run/containerd/containerd.sock"
    namespace: str = "opensandbox"
    snapshotter: str = "overlayfs"

class AppConfig(BaseModel):
    runtime: RuntimeConfig
    docker: DockerConfig
    kubernetes: K8sConfig
    containerd: ContainerdConfig  # ← 新增
    # ...
```

**Step 4: 配置文件示例**

```toml
# ~/.sandbox.toml
[runtime]
type = "containerd"

[containerd]
address = "/run/containerd/containerd.sock"
namespace = "opensandbox"
snapshotter = "overlayfs"

[security]
api_key = "your-api-key"
```

#### 6.2.3 自定义 Runtime 关键考量

| 考量点 | 说明 | 实现建议 |
|--------|------|----------|
| **execd 注入** | 所有运行时必须注入 execd 守护进程 | 参考 `DockerSandboxService._prepare_execd_archive()` |
| **生命周期管理** | 实现完整的状态机（Pending→Running→Paused→Terminated） | 使用内存/数据库存储状态 |
| **资源隔离** | CPU/内存/GPU 配额限制 | 调用底层运行时的资源限制 API |
| **网络隔离** | 端口映射、出口控制 | 实现网络策略或与现有方案集成 |
| **过期清理** | TTL 到期自动终止 | 使用后台定时器或事件驱动 |
| **可观测性** | 日志、指标、追踪 | 集成 OpenTelemetry 或 Prometheus |

#### 6.2.4 可选扩展方向

| 运行时类型 | 适用场景 | 实现难度 |
|-----------|----------|----------|
| **Podman** | 无守护进程、rootless 场景 | ⭐⭐ |
| **LXC/LXD** | 系统容器、更高性能 | ⭐⭐⭐ |
| **Firecracker** | 微 VM、更高隔离级别 | ⭐⭐⭐⭐ |
| **gVisor** | 用户态内核、增强安全 | ⭐⭐⭐ |
| **Kata Containers** | VM 级隔离、兼容 OCI | ⭐⭐⭐⭐ |

---

### 6.3 整体架构对从业者的启发

OpenSandbox 的设计体现了多个架构最佳实践，对 AI 基础设施从业者具有重要参考价值。

#### 6.3.1 核心设计原则

**1. 协议优先（Protocol-First）**

```
Specs Layer (OpenAPI)
       ↓
   定义契约
       ↓
SDKs ←──→ Runtime
```

**启发**:
- 所有交互由 OpenAPI 规范定义，确保组件独立演进
- 支持多语言 SDK 并行开发，无需等待运行时实现
- 新运行时只需遵循 API 契约即可无缝集成

**参考实践**:
- Kubernetes: OpenAPI + CRD
- Stripe: API-First 设计
- OpenAI: 统一的 REST API 规范

**2. 关注点分离（Separation of Concerns）**

| 层 | 职责 | 技术栈 |
|---|------|--------|
| SDKs | 客户端抽象、开发者体验 | Python/Java/TS |
| Specs | 协议定义、文档 | OpenAPI YAML |
| Runtime | 编排、生命周期管理 | FastAPI + Docker/K8s |
| execd | 容器内操作代理 | Go + Jupyter |

**启发**:
- 每层可独立替换（如替换 Runtime 不影响 SDK）
- 团队可并行开发不同层
- 故障隔离（execd 崩溃不影响 Server）

**3. 可插拔架构（Pluggable Architecture）**

```python
# 工厂模式实现运行时切换
implementations = {
    "docker": DockerSandboxService,
    "kubernetes": KubernetesSandboxService,
    # 新增运行时只需添加一行
    "containerd": ContainerdSandboxService,
}
```

**启发**:
- 使用工厂模式 + 策略模式
- 配置驱动运行时选择（`runtime.type = "docker"`）
- 支持渐进式迁移（开发用 Docker，生产用 K8s）

**4. 状态机驱动生命周期**

```
Pending → Running → Pausing → Paused → Stopping → Terminated
   ↑          ↓                      ↑
   └──────────┴────── Renew ─────────┘
```

**启发**:
- 显式状态转换，便于调试和监控
- 支持暂停/恢复（节省资源）
- TTL 自动过期（防止资源泄漏）

#### 6.3.2 关键技术决策

| 决策 | 选择 | 理由 | 替代方案 |
|------|------|------|----------|
| **容器内代理** | execd (Go) | 高性能、单二进制、易注入 | Python 脚本、SSH |
| **代码执行** | Jupyter Kernel | 多语言、有状态、成熟生态 | 直接 exec、gRPC |
| **输出流式** | SSE (Server-Sent Events) | 简单、浏览器原生支持 | WebSocket、gRPC Streaming |
| **API 框架** | FastAPI | 异步、自动 OpenAPI 生成 | Flask、Django |
| **K8s 集成** | Operator 模式 | 声明式、自动扩缩容 | 直接 API 调用 |

#### 6.3.3 可复用的架构模式

**模式 1: 双 API 分离**

```
Lifecycle API (server)     Execution API (execd)
     ↓                           ↓
  创建/销毁沙箱              容器内操作
  列表/状态查询              代码/文件/命令
```

**适用场景**: 需要区分"管理平面"和"数据平面"的系统

**模式 2: 资源池预热（Pool）**

```yaml
capacitySpec:
  bufferMin: 1    # 最小缓冲
  bufferMax: 3    # 最大缓冲
  poolMin: 0      # 池下限
  poolMax: 5      # 池上限
```

**适用场景**: 降低冷启动延迟（如 Serverless、AI Agent）

**模式 3: 注入式侧车（Injected Sidecar）**

```
用户镜像 + execd 注入 → 完整沙箱
```

**适用场景**: 无需修改用户镜像即可增强功能

#### 6.3.4 生产环境参考点

| 领域 | OpenSandbox 实践 | 可借鉴点 |
|------|------------------|----------|
| **安全** | API Key 认证 + execd Token | 双层认证（管理/执行分离） |
| **网络** | FQDN 级别出口控制 | 细粒度网络策略 |
| **资源** | CPU/Memory/GPU 配额 | 防止资源耗尽 |
| **可观测** | 结构化状态转换日志 | 便于审计和调试 |
| **高可用** | K8s Operator + Pool | 自动故障转移 |

#### 6.3.5 对 AI 基础设施的启示

**1. 执行层标准化是趋势**

随着 AI Agent 普及，"执行层"（Execution Layer）将成为独立的基础设施层：
```
LLM (Brain) → Planning → Tools → Execution Layer (OpenSandbox)
```

**2. 隔离是刚需**

- AI 生成的代码不可信 → 需要沙箱隔离
- Agent 可能执行危险操作 → 需要网络/文件系统限制
- 多租户场景 → 需要资源隔离

**3. 有状态执行是关键差异点**

- 无状态：每次执行独立（如 AWS Lambda）
- 有状态：变量/文件跨调用持久化（如 Jupyter）
- OpenSandbox 选择有状态 → 更适合 AI 迭代开发

**4. 统一 API 降低集成成本**

- 一次集成，多运行时支持
- 开发者无需关心底层是 Docker 还是 K8s
- 类似 Kubernetes 的"声明式 API"理念

---

## 7. 关键要点总结

### 7.1 统一、语言无关的执行

| 特性 | 描述 |
|------|------|
| **一致 API** | 跨语言、跨运行时的统一交互模式 |
| **多语言 SDK** | Python/TypeScript/Java 已发布，C#/Go 路线图 |
| **有状态执行** | 变量跨多次调用持久化 |

### 6.2 基础设施灵活性（Docker & Kubernetes）

| 场景 | 运行时 | 优势 |
|------|--------|------|
| 本地开发 | Docker | 快速迭代、低延迟 |
| 生产部署 | Kubernetes | 大规模分布式调度、资源池化、自动扩缩容 |
| 迁移 | 统一 API | 消除"环境漂移"问题 |

### 6.3 广泛的生态系统集成

**已支持**:
- 🤖 模型接口：Claude Code, Gemini CLI, OpenAI Codex
- 🔄 编排框架：LangGraph, Google ADK
- 🌐 自动化工具：Chrome, Playwright
- 🖥️ 可视化：VNC 完整支持

### 6.4 消除"沙箱依赖"

| 对比项 | 托管服务 | OpenSandbox |
|--------|----------|-------------|
| 许可证 | 商业闭源 | Apache 2.0 开源 |
| 计费模式 | 按分钟收费 | 免费自托管 |
| 厂商锁定 | 是 | 否 |
| 可定制性 | 低 | 高 |

### 6.5 高保真交互（VNC & Web）

**超越简单脚本执行**:
- ✅ 完整 VNC 桌面支持
- ✅ 浏览器自动化
- ✅ 多模态任务（导航网页、使用桌面应用）
- ✅ "防爆"安全环境

---

## 8. 可扩展点与生产注意事项

### 8.1 可扩展方向

| 方向 | 描述 | 优先级 |
|------|------|--------|
| **自定义镜像** | 预装特定依赖的沙箱镜像 | ⭐⭐⭐ |
| **网络策略细化** | FQDN 级别出口控制扩展 | ⭐⭐⭐ |
| **资源配额动态调整** | 运行时 CPU/内存调整 | ⭐⭐ |
| **多租户隔离** | Kubernetes Namespace 隔离 | ⭐⭐ |
| **监控告警集成** | Prometheus + Grafana | ⭐⭐ |
| **审计日志** | 操作审计与合规 | ⭐⭐ |

### 8.2 生产环境注意事项

| 关注点 | 建议 |
|--------|------|
| **安全加固** | 启用 API Key 认证、限制出口域名、定期更新基础镜像 |
| **资源管理** | 配置合理的 TTL、使用 Pool 预热、设置资源配额 |
| **高可用** | Kubernetes 多副本部署、健康检查、自动故障转移 |
| **监控** | 集成 Prometheus 指标、设置告警阈值 |
| **备份** | PVC 持久化存储、定期快照 |

---

## 9. 参考资源

| 资源 | 链接 |
|------|------|
| **GitHub 仓库** | https://github.com/alibaba/OpenSandbox |
| **官方文档** | https://open-sandbox.ai |
| **MarkTechPost 报道** | https://www.marktechpost.com/2026/03/03/alibaba-releases-opensandbox/ |
| **Python SDK** | `pip install opensandbox` |
| **服务器** | `pip install opensandbox-server` |

---

*文档生成时间：2026-03-14*  
*基于 OpenSandbox v1.0.1 源码与官方文档*
