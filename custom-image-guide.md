# OpenSandbox 自定义镜像指南

> 不使用官方镜像，而是使用自己构建的 Docker 镜像

---

## 核心原理

OpenSandbox 的镜像设计采用 **两层结构**：

```
┌─────────────────────────────────────────┐
│  用户自定义镜像                          │
│  (你的业务代码 + 依赖)                   │
├─────────────────────────────────────────┤
│  OpenSandbox Base 镜像                   │
│  (多语言运行时 + Jupyter + execd 注入点)   │
└─────────────────────────────────────────┘
```

**关键点**：
1. OpenSandbox 通过 **execd 注入** 机制工作，无需修改用户镜像
2. 用户镜像只需兼容 OpenSandbox 的 **entrypoint 协议**
3. 代码层面只需修改 `SANDBOX_IMAGE` 环境变量

---

## 示例 1：Code Interpreter 自定义镜像

### 1.1 官方镜像结构分析

官方 code-interpreter 镜像的构建流程：

```
Dockerfile_base (基础层)
    ↓
  - Ubuntu 24.04
  - Java 8/11/17/21
  - Python 3.10-3.14 (uv 管理)
  - Node.js 18/20/22
  - Go 1.23-1.25
  - Maven 3.9.2
    ↓
Dockerfile (完整镜像)
    ↓
  - ipykernel (Python Jupyter 内核)
  - IJava (Java Jupyter 内核)
  - tslab (TypeScript Jupyter 内核)
  - gonb (Go Jupyter 内核)
  - bash_kernel (Bash Jupyter 内核)
  - Jupyter Server
    ↓
  最终镜像：opensandbox/code-interpreter:v1.0.1
```

### 1.2 自定义镜像方案

#### 方案 A：基于官方 base 镜像扩展（推荐）

**适用场景**：需要官方多语言支持，但想添加自己的依赖

**Dockerfile 示例**：

```dockerfile
# 基于官方 base 镜像
FROM opensandbox/code-interpreter-base:latest

# 添加你的自定义依赖
RUN apt-get update && apt-get install -y --no-install-recommends \
    ffmpeg \
    libopencv-dev \
    && rm -rf /var/lib/apt/lists/*

# 安装 Python 自定义包
RUN python3 -m pip install --break-system-packages \
    pandas \
    numpy \
    matplotlib \
    opencv-python \
    torch \
    transformers

# 安装自定义 Jupyter 内核配置
COPY requirements.txt /workspace/requirements.txt
RUN pip install -r /workspace/requirements.txt

# 复制官方 entrypoint 脚本（关键！）
COPY --from=opensandbox/code-interpreter:latest \
    /opt/opensandbox/code-interpreter.sh \
    /opt/opensandbox/code-interpreter.sh
COPY --from=opensandbox/code-interpreter:latest \
    /root/.jupyter/jupyter_notebook_config.py \
    /root/.jupyter/jupyter_notebook_config.py

RUN chmod +x /opt/opensandbox/code-interpreter.sh

# 设置环境变量（与官方镜像一致）
ENV JUPYTER_HOST=http://127.0.0.1:44771 \
    JUPYTER_PORT=44771 \
    JUPYTER_TOKEN=opensandboxcodeinterpreterjupyter \
    PYTHON_VERSION=3.14 \
    NODE_VERSION=22 \
    GO_VERSION=1.25 \
    JAVA_VERSION=21

WORKDIR /workspace

# 使用官方 entrypoint（关键！）
ENTRYPOINT ["/opt/opensandbox/code-interpreter.sh"]
```

**构建命令**：

```bash
docker build -t my-custom-code-interpreter:latest .
```

**使用方式**（修改代码）：

```python
# examples/code-interpreter/main.py (自定义版本)
import os
from opensandbox import Sandbox
from opensandbox_code_interpreter import CodeInterpreter

async def main():
    # 关键修改：指定自定义镜像
    image = os.getenv(
        "SANDBOX_IMAGE",
        "my-custom-code-interpreter:latest"  # ← 改为你的镜像
    )
    
    domain = os.getenv("SANDBOX_DOMAIN", "localhost:8080")
    api_key = os.getenv("SANDBOX_API_KEY")
    
    async with Sandbox.create(
        image,
        connection_config=ConnectionConfig(
            domain=domain,
            api_key=api_key,
        ),
    ) as sandbox:
        interpreter = CodeInterpreter(sandbox)
        
        # 现在可以使用你预装的库
        result = await interpreter.run_python("""
import torch
import pandas as pd
from transformers import pipeline

print(f"PyTorch version: {torch.__version__}")
print(f"Pandas version: {pd.__version__}")

# 示例：使用 transformers
classifier = pipeline("sentiment-analysis")
result = classifier("OpenSandbox is awesome!")
print(result)
""")
        print(f"Result: {result}")

if __name__ == "__main__":
    import asyncio
    asyncio.run(main())
```

**环境变量设置**：

```bash
# .env 文件
SANDBOX_IMAGE=my-custom-code-interpreter:latest
SANDBOX_DOMAIN=localhost:8080
# SANDBOX_API_KEY=your-api-key (如果需要认证)
```

---

#### 方案 B：完全自定义镜像（从零构建）

**适用场景**：不需要多语言支持，只需特定运行时（如纯 Python 数据科学环境）

**Dockerfile 示例**：

```dockerfile
# 从 Ubuntu 基础镜像开始
FROM ubuntu:24.04

ENV DEBIAN_FRONTEND=noninteractive \
    LANG=C.UTF-8

# 1. 安装基础工具
RUN apt-get update && apt-get install -y --no-install-recommends \
    ca-certificates \
    curl \
    git \
    vim \
    python3 \
    python3-pip \
    python3-venv \
    && rm -rf /var/lib/apt/lists/*

# 2. 安装你的依赖
RUN pip3 install --break-system-packages \
    pandas \
    numpy \
    matplotlib \
    scikit-learn \
    jupyter \
    ipykernel

# 3. 创建 Jupyter 内核配置
RUN python3 -m ipykernel install --name python --display-name "Python"

# 4. 创建工作目录
WORKDIR /workspace

# 5. 关键：兼容 OpenSandbox execd 注入
# execd 会在容器启动时注入到 /opt/opensandbox/
# 我们需要确保 entrypoint 不冲突

# 创建启动脚本（兼容 execd）
RUN mkdir -p /opt/opensandbox
COPY <<'EOF' /opt/opensandbox/bootstrap.sh
#!/bin/bash
# OpenSandbox 兼容的启动脚本

# 如果 execd 存在，先启动 execd
if [ -f /opt/opensandbox/execd ]; then
    /opt/opensandbox/execd &
    sleep 1
fi

# 启动 Jupyter（如果配置了 JUPYTER_PORT）
if [ -n "$JUPYTER_PORT" ]; then
    jupyter notebook \
        --ip=127.0.0.1 \
        --port="$JUPYTER_PORT" \
        --allow-root \
        --no-browser \
        --NotebookApp.token="${JUPYTER_TOKEN:-opensandbox}" \
        >/opt/opensandbox/jupyter.log 2>&1 &
fi

# 保持容器运行
tail -f /dev/null
EOF

RUN chmod +x /opt/opensandbox/bootstrap.sh

# 设置环境变量
ENV JUPYTER_PORT=44771 \
    JUPYTER_TOKEN=opensandbox

# 使用自定义 entrypoint
ENTRYPOINT ["/opt/opensandbox/bootstrap.sh"]
```

**构建命令**：

```bash
docker build -t my-simple-python-sandbox:latest .
```

**使用代码**（修改 langgraph 示例）：

```python
# examples/langgraph/main.py (自定义镜像版本)
import os
from datetime import timedelta
from typing import TypedDict

from langgraph.graph import END, StateGraph
from opensandbox import Sandbox
from opensandbox.config import ConnectionConfig

class WorkflowState(TypedDict):
    sandbox: Sandbox | None
    run_output: str
    # ... 其他字段

async def create_sandbox(state: WorkflowState) -> WorkflowState:
    print("[create] Creating sandbox")
    domain = os.getenv("SANDBOX_DOMAIN", "localhost:8080")
    api_key = os.getenv("SANDBOX_API_KEY")
    
    # 关键修改：使用自定义镜像
    image = os.getenv(
        "SANDBOX_IMAGE",
        "my-simple-python-sandbox:latest"  # ← 自定义镜像
    )

    config = ConnectionConfig(
        domain=domain,
        api_key=api_key,
        request_timeout=timedelta(seconds=120),
    )

    sandbox = await Sandbox.create(
        image,
        connection_config=config,
        # 可选：自定义 entrypoint（如果镜像需要）
        # entrypoint=["/opt/opensandbox/bootstrap.sh"],
    )

    print(f"[create] Sandbox ready: {sandbox.id}")
    return {**state, "sandbox": sandbox}

# ... 其余代码保持不变
```

---

## 示例 2：LangGraph + 自定义镜像

### 2.1 场景说明

假设你想为 LangGraph Agent 创建一个预装特定工具的镜像（如网页爬虫 + 数据分析）

### 2.2 构建自定义镜像

**Dockerfile**：

```dockerfile
FROM opensandbox/code-interpreter-base:latest

# 安装网页爬虫工具
RUN apt-get update && apt-get install -y --no-install-recommends \
    chromium \
    chromium-driver \
    && rm -rf /var/lib/apt/lists/*

# 安装 Python 依赖
RUN pip3 install --break-system-packages \
    playwright \
    beautifulsoup4 \
    requests \
    selenium \
    pandas \
    numpy

# 安装 Playwright 浏览器
RUN playwright install chromium

# 复制官方 entrypoint（保持兼容）
COPY --from=opensandbox/code-interpreter:latest \
    /opt/opensandbox/code-interpreter.sh \
    /opt/opensandbox/code-interpreter.sh
COPY --from=opensandbox/code-interpreter:latest \
    /root/.jupyter/jupyter_notebook_config.py \
    /root/.jupyter/jupyter_notebook_config.py

RUN chmod +x /opt/opensandbox/code-interpreter.sh

ENV JUPYTER_HOST=http://127.0.0.1:44771 \
    JUPYTER_PORT=44771 \
    JUPYTER_TOKEN=opensandboxcodeinterpreterjupyter \
    PYTHON_VERSION=3.14

WORKDIR /workspace

ENTRYPOINT ["/opt/opensandbox/code-interpreter.sh"]
```

**构建**：

```bash
docker build -t my-langgraph-sandbox:latest .
```

### 2.3 修改 LangGraph 代码

```python
# examples/langgraph/main_custom_image.py
import os
from datetime import timedelta
from typing import TypedDict

from langchain_anthropic import ChatAnthropic
from langgraph.graph import END, StateGraph
from opensandbox import Sandbox
from opensandbox.config import ConnectionConfig

class WorkflowState(TypedDict):
    sandbox: Sandbox | None
    run_output: str
    summary: str
    last_error: str
    attempt: int
    max_attempts: int
    command: str
    fallback_command: str
    cleaned: bool

async def create_sandbox(state: WorkflowState) -> WorkflowState:
    print("[create] Creating sandbox with CUSTOM IMAGE")
    domain = os.getenv("SANDBOX_DOMAIN", "localhost:8080")
    api_key = os.getenv("SANDBOX_API_KEY")
    
    # ★★★ 关键修改：使用自定义镜像 ★★★
    image = os.getenv(
        "SANDBOX_IMAGE",
        "my-langgraph-sandbox:latest"  # ← 自定义镜像名称
    )
    
    print(f"[create] Using image: {image}")

    config = ConnectionConfig(
        domain=domain,
        api_key=api_key,
        request_timeout=timedelta(seconds=120),
    )

    sandbox = await Sandbox.create(
        image,
        connection_config=config,
        # 可选：添加自定义元数据
        metadata={
            "purpose": "langgraph-web-crawler",
            "owner": "will",
        },
    )

    print(f"[create] Sandbox ready: {sandbox.id}")
    return {**state, "sandbox": sandbox}

async def prepare_workspace(state: WorkflowState) -> WorkflowState:
    print("[prepare] Writing crawler script")
    sandbox = state["sandbox"]
    if sandbox is None:
        raise RuntimeError("Sandbox not initialized")

    # 写入自定义爬虫脚本
    await sandbox.files.write_file(
        "/workspace/crawler.py",
        """
import asyncio
from playwright.async_api import async_playwright

async def crawl(url):
    async with async_playwright() as p:
        browser = await p.chromium.launch()
        page = await browser.new_page()
        await page.goto(url)
        title = await page.title()
        content = await page.content()
        await browser.close()
        return {"title": title, "content_length": len(content)}

result = asyncio.run(crawl("https://example.com"))
print(result)
""",
    )
    
    # 写入配置文件
    await sandbox.files.write_file(
        "/workspace/config.json",
        '{"timeout": 30, "retry": 3}',
    )

    print("[prepare] Files written")
    return state

async def run_job(state: WorkflowState) -> WorkflowState:
    attempt = state["attempt"] + 1
    max_attempts = state["max_attempts"]
    
    # 使用自定义爬虫脚本
    command = state.get("command") or "python /workspace/crawler.py"
    
    print(f"[run] Executing job (attempt {attempt}/{max_attempts})")
    print(f"[run] Command: {command}")
    
    sandbox = state["sandbox"]
    if sandbox is None:
        raise RuntimeError("Sandbox not initialized")

    execution = await sandbox.commands.run(command)
    
    # 格式化输出
    stdout = "\\n".join(msg.text for msg in execution.logs.stdout)
    stderr = "\\n".join(msg.text for msg in execution.logs.stderr)
    
    if execution.error:
        stderr = "\\n".join([
            stderr,
            f"[error] {execution.error.name}: {execution.error.value}",
        ]).strip()
    
    run_output = stdout.strip()
    last_error = ""
    next_command = command

    if execution.error:
        last_error = f"{execution.error.name}: {execution.error.value}"
        if attempt < max_attempts:
            next_command = state.get("fallback_command", "python /workspace/crawler.py")
            print(f"[run] Failed, scheduling fallback: {next_command}")

    print(f"[run] Output: {run_output}")

    return {
        **state,
        "run_output": run_output,
        "last_error": last_error,
        "attempt": attempt,
        "command": next_command,
    }

def decide_next(state: WorkflowState) -> str:
    if state.get("last_error") and state["attempt"] < state["max_attempts"]:
        print("[decide] Retry with fallback command")
        return "run"
    print("[decide] Proceeding to cleanup")
    return "cleanup"

async def cleanup_sandbox(state: WorkflowState) -> WorkflowState:
    print("[cleanup] Cleaning up sandbox")
    sandbox = state.get("sandbox")
    if sandbox is not None:
        await sandbox.kill()
        await sandbox.close()
    print("[cleanup] Done")
    return {**state, "sandbox": None, "cleaned": True}

async def main() -> None:
    # 构建 LangGraph 工作流
    graph = StateGraph(WorkflowState)
    graph.add_node("create", create_sandbox)
    graph.add_node("prepare", prepare_workspace)
    graph.add_node("run", run_job)
    graph.add_node("cleanup", cleanup_sandbox)
    
    graph.set_entry_point("create")
    graph.add_edge("create", "prepare")
    graph.add_edge("prepare", "run")
    graph.add_conditional_edges(
        "run",
        decide_next,
        {"run": "run", "cleanup": "cleanup"},
    )
    graph.add_edge("cleanup", END)
    
    app = graph.compile()

    initial_state = {
        "sandbox": None,
        "run_output": "",
        "summary": "",
        "last_error": "",
        "attempt": 0,
        "max_attempts": 2,
        "command": "python /workspace/crawler.py",
        "fallback_command": "python /workspace/crawler.py",
        "cleaned": False,
    }

    state = initial_state
    try:
        async for update in app.astream(initial_state, stream_mode="values"):
            state = update
    finally:
        if not state.get("cleaned"):
            sandbox = state.get("sandbox")
            if sandbox is not None:
                await sandbox.kill()
                await sandbox.close()

    print(f"\\n=== Final Output ===")
    print(f"Run output: {state['run_output']}")

if __name__ == "__main__":
    import asyncio
    asyncio.run(main())
```

### 2.4 运行测试

```bash
# 设置环境变量
export SANDBOX_IMAGE=my-langgraph-sandbox:latest
export SANDBOX_DOMAIN=localhost:8080
export ANTHROPIC_API_KEY=sk-ant-xxx

# 启动 OpenSandbox 服务器（如果还没启动）
opensandbox-server

# 运行自定义示例
python examples/langgraph/main_custom_image.py
```

---

## 关键要点总结

### 镜像生成（Dockerfile）

| 要素 | 官方做法 | 自定义做法 |
|------|----------|-----------|
| **基础镜像** | Ubuntu 24.04 + 多语言运行时 | 可基于官方 base 或从零开始 |
| **Jupyter 内核** | ipykernel + IJava + tslab + gonb | 按需安装 |
| **Entrypoint** | `/opt/opensandbox/code-interpreter.sh` | 必须兼容 execd 注入 |
| **环境变量** | `JUPYTER_PORT`, `JUPYTER_TOKEN` 等 | 保持一致 |
| **工作目录** | `/workspace` | 建议保持一致 |

### 代码修改

**只需修改一处**（99% 的场景）：

```python
# 原代码
image = os.getenv(
    "SANDBOX_IMAGE",
    "sandbox-registry.cn-zhangjiakou.cr.aliyuncs.com/opensandbox/code-interpreter:v1.0.1",
)

# 修改为
image = os.getenv(
    "SANDBOX_IMAGE",
    "my-custom-image:latest",  # ← 改这里
)
```

**可选修改**（高级场景）：

```python
# 自定义 entrypoint（如果镜像需要）
sandbox = await Sandbox.create(
    image,
    entrypoint=["/opt/opensandbox/bootstrap.sh"],
    # 自定义环境变量
    environment={
        "MY_CUSTOM_VAR": "value",
        "PYTHONPATH": "/workspace",
    },
    # 自定义元数据
    metadata={
        "purpose": "custom-task",
        "owner": "will",
    },
)
```

### execd 注入机制（核心原理）

OpenSandbox 的 **神奇之处** 在于 execd 注入：

```
1. 用户创建沙箱请求
   ↓
2. OpenSandbox Server 拉取用户镜像
   ↓
3. 将 execd 二进制文件注入到容器 /opt/opensandbox/
   ↓
4. 覆盖容器 entrypoint，先启动 execd
   ↓
5. execd 启动 Jupyter（如果配置了）
   ↓
6. 容器就绪，返回 SDK
```

**这意味着**：
- ✅ 用户镜像无需预装 execd
- ✅ 用户镜像无需修改代码
- ✅ 只需保持 entrypoint 兼容（或让 OpenSandbox 覆盖）

---

## 常见问题

### Q1: 我的镜像需要预装 execd 吗？

**不需要**。OpenSandbox Server 会在创建沙箱时自动注入 execd。

### Q2: 我的镜像必须使用 Jupyter 吗？

**不一定**。如果只需要命令执行，可以不装 Jupyter：

```dockerfile
FROM ubuntu:24.04
RUN apt-get update && apt-get install -y python3 python3-pip
# 不装 Jupyter，只用 sandbox.commands.run()
```

### Q3: 如何调试自定义镜像？

```bash
# 1. 本地测试镜像
docker run -it my-custom-image:latest bash

# 2. 手动测试 entrypoint
docker run -it \
  -e JUPYTER_PORT=44771 \
  -e JUPYTER_TOKEN=test \
  my-custom-image:latest

# 3. 查看 OpenSandbox Server 日志
opensandbox-server  # 终端会显示详细日志
```

### Q4: 如何推送镜像到私有仓库？

```bash
# 1. 登录仓库
docker login my-registry.com

# 2. 打标签
docker tag my-custom-image:latest \
  my-registry.com/opensandbox/custom:latest

# 3. 推送
docker push my-registry.com/opensandbox/custom:latest

# 4. 在代码中使用
image = "my-registry.com/opensandbox/custom:latest"
```

### Q5: 如何配置私有仓库认证？

**方式 1：配置文件**（~/.sandbox.toml）

```toml
[runtime.docker]
registry_auth = [
  { registry = "my-registry.com", username = "user", password = "pass" },
]
```

**方式 2：环境变量**

```bash
export DOCKER_REGISTRY_AUTH='{"my-registry.com": {"username": "user", "password": "pass"}}'
```

---

*基于 OpenSandbox v1.0.1 源码分析*
