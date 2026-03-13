# OpenSandbox 使用示例详解

> **作者**：sandboxrosy  
> **日期**：2026-03-13  
> **来源**：OpenSandbox 官方 examples 目录 + 实践总结

---

## 目录

1. [环境准备](#1-环境准备)
2. [官方示例运行指南](#2-官方示例运行指南)
3. [运行完整 Python 项目](#3-运行完整-python-项目)
4. [高级场景](#4-高级场景)
5. [常见问题与排查](#5-常见问题与排查)

---

## 1. 环境准备

### 1.1 系统要求

| 依赖 | 版本要求 | 说明 |
|------|----------|------|
| **Docker** | 20.10+ | 本地运行沙箱必需 |
| **Python** | 3.10+ | SDK 和 Server 运行环境 |
| **uv** | 最新版 | 推荐的 Python 包管理器 |

### 1.2 安装 OpenSandbox Server

```bash
# 安装 Server
uv pip install opensandbox-server

# 初始化配置文件
opensandbox-server init-config ~/.sandbox.toml --example docker

# 启动服务（前台运行）
opensandbox-server
```

**配置文件说明 (`~/.sandbox.toml`)：**

```toml
[server]
host = "0.0.0.0"
port = 8080

[runtime]
type = "docker"  # 或 "kubernetes"

[docker]
# 安全配置
drop_capabilities = ["AUDIT_WRITE", "MKNOD", "NET_ADMIN"]
no_new_privileges = true
pids_limit = 512

# 资源限制
default_cpu_limit = "1"
default_memory_limit = "512Mi"
default_timeout = "30m"
```

### 1.3 安装 SDK

```bash
# 安装基础 SDK
uv pip install opensandbox

# 安装代码解释器 SDK
uv pip install opensandbox-code-interpreter

# 或从源码安装最新版
git clone https://github.com/alibaba/OpenSandbox.git
cd OpenSandbox
uv pip install -e sdks/sandbox/python
```

### 1.4 环境变量配置

```bash
# 基础配置
export SANDBOX_DOMAIN="localhost:8080"
export SANDBOX_API_KEY=""  # 如果 Server 配置了认证

# 可选：指定镜像
export SANDBOX_IMAGE="sandbox-registry.cn-zhangjiakou.cr.aliyuncs.com/opensandbox/code-interpreter:v1.0.1"
```

---

## 2. 官方示例运行指南

OpenSandbox 官方提供了丰富的示例，位于 `examples/` 目录：

```
examples/
├── code-interpreter/     # 代码解释器示例 ⭐ 入门推荐
├── host-volume-mount/    # 主机目录挂载示例
├── rl-training/          # 强化学习训练示例
├── aio-sandbox/          # All-in-One 沙箱示例
├── chrome/               # Chromium 浏览器自动化
├── playwright/           # Playwright 爬虫示例
├── vscode/               # VS Code Web 示例
├── desktop/              # VNC 桌面环境示例
├── claude-code/          # Claude Code 集成
├── gemini-cli/           # Gemini CLI 集成
├── codex-cli/            # OpenAI Codex CLI 集成
├── langgraph/            # LangGraph 工作流集成
├── google-adk/           # Google ADK 集成
└── ...
```

### 2.1 Code Interpreter 示例（入门推荐）

**功能**：在沙箱中执行多语言代码（Python/Java/Go/TypeScript）

**运行步骤**：

```bash
# 1. 克隆仓库
git clone https://github.com/alibaba/OpenSandbox.git
cd OpenSandbox

# 2. 安装依赖
uv pip install opensandbox opensandbox-code-interpreter

# 3. 拉取镜像
docker pull sandbox-registry.cn-zhangjiakou.cr.aliyuncs.com/opensandbox/code-interpreter:v1.0.1

# 4. 运行示例
uv run python examples/code-interpreter/main.py
```

**预期输出**：

```
=== Python example ===
[Python stdout] Hello from Python!
[Python result] {'py': '3.14.2', 'sum': 4}

=== Java example ===
[Java stdout] Hello from Java!
[Java result] 5

=== Go example ===
[Go stdout] Hello from Go!
3 + 4 = 7

=== TypeScript example ===
[TypeScript stdout] Hello from TypeScript!
sum = 6
```

### 2.2 Host Volume Mount 示例

**功能**：将主机目录挂载到沙箱中，实现文件共享

**运行步骤**：

```bash
# 1. 创建共享目录
mkdir -p /tmp/opensandbox-data
echo "hello-from-host" > /tmp/opensandbox-data/marker.txt

# 2. 配置服务器允许挂载路径
# 编辑 ~/.sandbox.toml，添加：
# [storage]
# allowed_host_paths = ["/tmp/opensandbox-data"]

# 3. 重启 Server
# Ctrl+C 停止，然后重新运行
opensandbox-server

# 4. 运行示例
HOST_VOLUME_PATH=/tmp/opensandbox-data uv run python examples/host-volume-mount/main.py
```

**三种挂载模式**：

| 模式 | 说明 | 使用场景 |
|------|------|----------|
| **Read-Write** | 读写挂载 | 沙箱输出文件到主机 |
| **Read-Only** | 只读挂载 | 共享数据集/配置 |
| **SubPath** | 子目录挂载 | 只挂载特定目录 |

### 2.3 RL Training 示例

**功能**：在沙箱中运行强化学习训练

**运行步骤**：

```bash
# 1. 安装依赖
uv pip install opensandbox

# 2. 运行训练
uv run python examples/rl-training/main.py

# 可选：调整训练步数
RL_TIMESTEPS=10000 uv run python examples/rl-training/main.py
```

**预期输出**：

```json
{
  "timesteps": 5000,
  "mean_reward": 150.5,
  "std_reward": 30.2,
  "checkpoint_path": "checkpoints/cartpole_dqn.zip"
}
```

### 2.4 浏览器自动化示例（Chrome/Playwright）

**Chrome 示例**：

```bash
# 拉取镜像
docker pull sandbox-registry.cn-zhangjiakou.cr.aliyuncs.com/opensandbox/chrome:v1.0.1

# 运行
uv run python examples/chrome/main.py
```

**Playwright 示例**：

```bash
uv run python examples/playwright/main.py
```

### 2.5 VS Code Web 示例

**功能**：在沙箱中启动 VS Code Web IDE

```bash
# 拉取镜像
docker pull sandbox-registry.cn-zhangjiakou.cr.aliyuncs.com/opensandbox/vscode:v1.0.1

# 运行
uv run python examples/vscode/main.py

# 访问 VS Code Web
# 控制台会输出访问地址，如：
# VS Code available at: http://localhost:xxxx
```

---

## 3. 运行完整 Python 项目

### 3.1 场景说明

假设你有一个 Python 项目，包含：

```
my-project/
├── main.py           # 入口文件
├── requirements.txt  # 依赖列表
├── config.json       # 配置文件
├── src/
│   ├── __init__.py
│   ├── utils.py
│   └── model.py
└── data/
    └── input.csv
```

**目标**：在沙箱中运行这个项目，并获取输出结果。

### 3.2 方法一：逐文件上传

**适用场景**：小型项目，文件数量少

```python
# run_project_v1.py

import asyncio
from pathlib import Path
from datetime import timedelta
from opensandbox import Sandbox
from opensandbox.config import ConnectionConfig

async def main():
    # 1. 创建沙箱
    sandbox = await Sandbox.create(
        "sandbox-registry.cn-zhangjiakou.cr.aliyuncs.com/opensandbox/code-interpreter:v1.0.1",
        timeout=timedelta(minutes=10),
    )

    async with sandbox:
        # 2. 上传项目文件
        project_dir = Path("my-project")
        
        # 逐个上传文件
        files_to_upload = [
            ("main.py", project_dir / "main.py"),
            ("requirements.txt", project_dir / "requirements.txt"),
            ("config.json", project_dir / "config.json"),
            ("src/__init__.py", project_dir / "src" / "__init__.py"),
            ("src/utils.py", project_dir / "src" / "utils.py"),
            ("src/model.py", project_dir / "src" / "model.py"),
            ("data/input.csv", project_dir / "data" / "input.csv"),
        ]
        
        for dest_path, src_path in files_to_upload:
            content = src_path.read_bytes()
            await sandbox.files.write_file(dest_path, content)
            print(f"Uploaded: {dest_path}")

        # 3. 安装依赖
        print("Installing dependencies...")
        install_result = await sandbox.commands.run(
            "pip install -r requirements.txt"
        )
        for msg in install_result.logs.stdout:
            print(f"[pip] {msg.text}")

        # 4. 运行项目
        print("Running main.py...")
        run_result = await sandbox.commands.run("python main.py")
        
        # 打印输出
        for msg in run_result.logs.stdout:
            print(f"[stdout] {msg.text}")
        for msg in run_result.logs.stderr:
            print(f"[stderr] {msg.text}")

        # 5. 获取输出文件（如果有）
        try:
            output = await sandbox.files.read_file("output/result.json")
            print(f"Output: {output}")
        except Exception as e:
            print(f"No output file: {e}")

    # 自动清理

if __name__ == "__main__":
    asyncio.run(main())
```

### 3.3 方法二：批量文件上传（推荐）

**适用场景**：中大型项目，需要上传大量文件

```python
# run_project_v2.py

import asyncio
from pathlib import Path
from datetime import timedelta
from opensandbox import Sandbox
from opensandbox.models import WriteEntry
from opensandbox.config import ConnectionConfig

async def upload_directory(sandbox: Sandbox, local_dir: Path, remote_dir: str = ""):
    """递归上传整个目录"""
    entries = []
    
    for file_path in local_dir.rglob("*"):
        if file_path.is_file():
            # 计算远程路径
            relative_path = file_path.relative_to(local_dir)
            remote_path = f"{remote_dir}/{relative_path}" if remote_dir else str(relative_path)
            
            # 添加到上传列表
            entries.append(WriteEntry(
                path=remote_path,
                data=file_path.read_bytes(),
                mode=0o644
            ))
    
    # 批量上传
    print(f"Uploading {len(entries)} files...")
    await sandbox.files.write_files(entries)
    print("Upload complete!")

async def main():
    config = ConnectionConfig(
        domain="localhost:8080",
        request_timeout=timedelta(minutes=10),
    )

    sandbox = await Sandbox.create(
        "sandbox-registry.cn-zhangjiakou.cr.aliyuncs.com/opensandbox/code-interpreter:v1.0.1",
        connection_config=config,
        timeout=timedelta(minutes=15),
    )

    async with sandbox:
        # 1. 上传整个项目目录
        project_dir = Path("my-project")
        await upload_directory(sandbox, project_dir)

        # 2. 安装依赖
        print("Installing dependencies...")
        install_result = await sandbox.commands.run(
            "cd /workspace && pip install -r requirements.txt"
        )
        for msg in install_result.logs.stdout:
            print(f"[pip] {msg.text}")

        # 3. 运行项目
        print("Running project...")
        run_result = await sandbox.commands.run(
            "cd /workspace && python main.py"
        )
        
        for msg in run_result.logs.stdout:
            print(f"[stdout] {msg.text}")
        for msg in run_result.logs.stderr:
            print(f"[stderr] {msg.text}")

        # 4. 获取结果文件
        try:
            result = await sandbox.files.read_file("workspace/output/result.json")
            print(f"Result: {result}")
        except Exception:
            pass

if __name__ == "__main__":
    asyncio.run(main())
```

### 3.4 方法三：Volume 挂载（最高效）

**适用场景**：本地开发、需要双向文件同步

```python
# run_project_v3.py

import asyncio
from pathlib import Path
from datetime import timedelta
from opensandbox import Sandbox
from opensandbox.models.sandboxes import Host, Volume
from opensandbox.config import ConnectionConfig

async def main():
    # 配置服务器允许挂载路径
    # ~/.sandbox.toml:
    # [storage]
    # allowed_host_paths = ["/Users/xxx/projects"]

    config = ConnectionConfig(domain="localhost:8080")
    
    project_path = Path("/Users/xxx/projects/my-project").resolve()

    sandbox = await Sandbox.create(
        "sandbox-registry.cn-zhangjiakou.cr.aliyuncs.com/opensandbox/code-interpreter:v1.0.1",
        connection_config=config,
        timeout=timedelta(minutes=15),
        # 挂载本地项目目录
        volumes=[
            Volume(
                name="project-code",
                host=Host(path=str(project_path)),
                mountPath="/workspace",
                readOnly=False,  # 可读写，沙箱输出会同步到本地
            ),
        ],
    )

    async with sandbox:
        # 1. 安装依赖
        print("Installing dependencies...")
        await sandbox.commands.run("cd /workspace && pip install -r requirements.txt")

        # 2. 运行项目
        print("Running project...")
        run_result = await sandbox.commands.run("cd /workspace && python main.py")
        
        for msg in run_result.logs.stdout:
            print(f"[stdout] {msg.text}")

        # 3. 结果已同步到本地
        # 无需手动下载，本地目录自动更新

if __name__ == "__main__":
    asyncio.run(main())
```

### 3.5 方法四：打包上传（适合远程部署）

**适用场景**：项目较大、网络传输优化

```bash
# 1. 打包项目
cd my-project
tar -czvf ../project.tar.gz .

# 2. 运行脚本
```

```python
# run_project_v4.py

import asyncio
import tarfile
import io
from pathlib import Path
from datetime import timedelta
from opensandbox import Sandbox
from opensandbox.config import ConnectionConfig

async def main():
    config = ConnectionConfig(domain="localhost:8080")
    
    # 读取打包文件
    tar_path = Path("project.tar.gz")
    tar_data = tar_path.read_bytes()

    sandbox = await Sandbox.create(
        "sandbox-registry.cn-zhangjiakou.cr.aliyuncs.com/opensandbox/code-interpreter:v1.0.1",
        connection_config=config,
        timeout=timedelta(minutes=15),
    )

    async with sandbox:
        # 1. 上传 tar 包
        await sandbox.files.write_file("/tmp/project.tar.gz", tar_data)
        print("Uploaded project.tar.gz")

        # 2. 解压
        await sandbox.commands.run(
            "mkdir -p /workspace && tar -xzf /tmp/project.tar.gz -C /workspace"
        )
        print("Extracted to /workspace")

        # 3. 安装依赖
        await sandbox.commands.run("cd /workspace && pip install -r requirements.txt")
        print("Dependencies installed")

        # 4. 运行
        result = await sandbox.commands.run("cd /workspace && python main.py")
        for msg in result.logs.stdout:
            print(f"[stdout] {msg.text}")

if __name__ == "__main__":
    asyncio.run(main())
```

### 3.6 完整示例：运行 Flask Web 项目

```python
# run_flask_project.py

import asyncio
from pathlib import Path
from datetime import timedelta
from opensandbox import Sandbox
from opensandbox.models import WriteEntry
from opensandbox.config import ConnectionConfig

# Flask 项目文件内容
APP_PY = """
from flask import Flask, jsonify
import os

app = Flask(__name__)

@app.route('/')
def hello():
    return jsonify({
        "message": "Hello from OpenSandbox!",
        "env": os.environ.get("APP_ENV", "development")
    })

@app.route('/health')
def health():
    return jsonify({"status": "healthy"})

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
"""

REQUIREMENTS_TXT = """
flask==3.0.0
gunicorn==21.2.0
"""

async def main():
    config = ConnectionConfig(
        domain="localhost:8080",
        request_timeout=timedelta(minutes=10),
    )

    sandbox = await Sandbox.create(
        "sandbox-registry.cn-zhangjiakou.cr.aliyuncs.com/opensandbox/code-interpreter:v1.0.1",
        connection_config=config,
        env={"APP_ENV": "sandbox"},
        timeout=timedelta(minutes=10),
    )

    async with sandbox:
        try:
            # 1. 上传项目文件
            print("Uploading files...")
            await sandbox.files.write_files([
                WriteEntry(path="app/app.py", data=APP_PY.encode()),
                WriteEntry(path="app/requirements.txt", data=REQUIREMENTS_TXT.encode()),
            ])

            # 2. 安装依赖
            print("Installing Flask...")
            install_result = await sandbox.commands.run(
                "cd app && pip install -r requirements.txt"
            )
            for msg in install_result.logs.stdout:
                if "Successfully" in msg.text or "flask" in msg.text.lower():
                    print(f"  {msg.text}")

            # 3. 后台启动 Flask（不阻塞）
            print("Starting Flask server...")
            await sandbox.commands.run_bg(
                "cd app && python app.py"
            )

            # 4. 等待服务启动
            await asyncio.sleep(3)

            # 5. 获取端点地址
            endpoint_result = await sandbox._client.get(
                f"/v1/sandboxes/{sandbox.id}/endpoints/5000"
            )
            flask_url = endpoint_result.json()["endpoint"]
            print(f"Flask running at: {flask_url}")

            # 6. 测试 API
            import httpx
            async with httpx.AsyncClient() as client:
                response = await client.get(f"{flask_url}/")
                print(f"Response: {response.json()}")

                health = await client.get(f"{flask_url}/health")
                print(f"Health: {health.json()}")

            # 7. 保持运行一段时间（演示）
            print("Server running... Press Ctrl+C to stop")
            await asyncio.sleep(30)

        finally:
            await sandbox.kill()

if __name__ == "__main__":
    asyncio.run(main())
```

---

## 4. 高级场景

### 4.1 使用自定义 Docker 镜像

```dockerfile
# Dockerfile
FROM python:3.11-slim

WORKDIR /app

# 预装依赖
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# 复制代码
COPY . .

CMD ["python", "main.py"]
```

```bash
# 构建并推送
docker build -t my-registry/my-app:v1.0 .
docker push my-registry/my-app:v1.0
```

```python
# 使用自定义镜像
sandbox = await Sandbox.create(
    "my-registry/my-app:v1.0",
    timeout=timedelta(minutes=10),
)
```

### 4.2 GPU 支持

```python
# 需要 Kubernetes Runtime + GPU 节点
sandbox = await Sandbox.create(
    "pytorch/pytorch:2.0-cuda11.7",
    resources={"gpu": 1},
    timeout=timedelta(hours=1),
)

async with sandbox:
    # 运行 GPU 训练
    result = await sandbox.commands.run("python train.py --device cuda")
```

### 4.3 多沙箱协作

```python
import asyncio
from opensandbox import Sandbox

async def worker(sandbox_id: int, task: str):
    sandbox = await Sandbox.create("python:3.11")
    async with sandbox:
        result = await sandbox.commands.run(task)
        print(f"Worker {sandbox_id}: {result.logs.stdout[0].text}")
        return result

async def main():
    # 并行创建多个沙箱执行任务
    tasks = [
        worker(1, "echo 'Task 1'"),
        worker(2, "echo 'Task 2'"),
        worker(3, "echo 'Task 3'"),
    ]
    results = await asyncio.gather(*tasks)
    print(f"Completed {len(results)} tasks")

asyncio.run(main())
```

### 4.4 使用 Kubernetes 池化

```yaml
# pool-config.yaml
apiVersion: sandbox.opensandbox.io/v1alpha1
kind: Pool
metadata:
  name: python-pool
spec:
  template:
    spec:
      containers:
        - name: sandbox
          image: sandbox-registry.cn-zhangjiakou.cr.aliyuncs.com/opensandbox/code-interpreter:v1.0.1
  capacitySpec:
    poolMin: 5
    poolMax: 50
    bufferMin: 3
    bufferMax: 10
```

```bash
# 创建池
kubectl apply -f pool-config.yaml
```

```python
# 从池中获取沙箱（毫秒级）
from opensandbox import Sandbox

# 配置使用 K8s pool
sandbox = await Sandbox.create(
    "opensandbox/code-interpreter:v1.0.1",
    pool="python-pool",  # 从指定池获取
)
```

---

## 5. 常见问题与排查

### 5.1 Docker 连接失败

**错误**：`FileNotFoundError: [Errno 2] No such file or directory`

**解决**：

```bash
# 检查 Docker 是否运行
docker version

# macOS Colima 用户
colima start
export DOCKER_HOST="unix://${HOME}/.colima/default/docker.sock"
```

### 5.2 沙箱创建超时

**错误**：`TimeoutError: Sandbox xxx not ready within 60s`

**解决**：

```python
# 增加超时时间
sandbox = await Sandbox.create(
    image,
    timeout=timedelta(minutes=5),  # 更长超时
)

# 或使用预热池
# 从池中获取 < 100ms
```

### 5.3 文件上传失败

**错误**：`Failed to write file`

**解决**：

```python
# 确保路径存在
await sandbox.commands.run("mkdir -p /workspace/data")

# 然后上传
await sandbox.files.write_file("/workspace/data/input.csv", data)
```

### 5.4 Volume 挂载失败

**错误**：`Host path not allowed`

**解决**：

```toml
# 编辑 ~/.sandbox.toml
[storage]
allowed_host_paths = ["/tmp", "/Users/xxx/projects"]
```

### 5.5 网络访问问题

**问题**：沙箱无法访问外网

**解决**：

```python
# 检查网络配置
result = await sandbox.commands.run("curl -I https://google.com")
print(result.logs.stdout)

# 如果使用 Egress 控制，确保规则允许
```

### 5.6 内存不足

**错误**：`OOMKilled`

**解决**：

```python
# 增加内存限制
sandbox = await Sandbox.create(
    image,
    resources={"memory": "2Gi", "cpu": "2"},
)
```

---

## 附录

### A. 快速命令参考

```bash
# 启动 Server
opensandbox-server

# 拉取镜像
docker pull sandbox-registry.cn-zhangjiakou.cr.aliyuncs.com/opensandbox/code-interpreter:v1.0.1

# 运行示例
uv run python examples/code-interpreter/main.py

# 查看日志
docker logs <container-id>
```

### B. SDK 常用方法速查

```python
# 创建沙箱
sandbox = await Sandbox.create(image, timeout=timedelta(minutes=10))

# 执行命令
result = await sandbox.commands.run("ls -la")

# 后台执行
await sandbox.commands.run_bg("python server.py")

# 上传文件
await sandbox.files.write_file("path/to/file", data)

# 批量上传
await sandbox.files.write_files([WriteEntry(path="a.py", data=code)])

# 读取文件
content = await sandbox.files.read_file("path/to/file")

# 搜索文件
files = await sandbox.files.search("*.py")

# 销毁沙箱
await sandbox.kill()
```

### C. 环境变量速查

| 变量 | 默认值 | 说明 |
|------|--------|------|
| `SANDBOX_DOMAIN` | `localhost:8080` | Server 地址 |
| `SANDBOX_API_KEY` | `""` | API 认证密钥 |
| `SANDBOX_IMAGE` | 官方镜像 | 默认镜像 |
| `HOST_VOLUME_PATH` | 临时目录 | 主机挂载路径 |

---

> **关于本文档**  
> 本文档基于 OpenSandbox 官方示例和实践经验编写，涵盖从入门到高级的各种使用场景。  
> 作者：sandboxrosy | 日期：2026-03-13