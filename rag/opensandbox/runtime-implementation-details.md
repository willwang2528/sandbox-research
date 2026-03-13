# OpenSandbox Runtime 层实现详解

> **作者**：sandboxrosy  
> **日期**：2026-03-13  
> **来源**：OpenSandbox Server 源码分析  
> **阅读时间**：约 30 分钟

---

## 引言

Runtime 层是 OpenSandbox 架构的核心，负责沙箱的生命周期管理。本文将深入分析 **Docker Runtime** 和 **Kubernetes Runtime** 两种实现的技术细节。

---

## 目录

1. [Runtime 层架构总览](#1-runtime-层架构总览)
2. [抽象接口设计](#2-抽象接口设计)
3. [Docker Runtime 实现](#3-docker-runtime-实现)
4. [Kubernetes Runtime 实现](#4-kubernetes-runtime-实现)
5. [关键流程对比](#5-关键流程对比)
6. [总结](#6-总结)

---

## 1. Runtime 层架构总览

### 1.1 层次结构

```
┌─────────────────────────────────────────────────────────────┐
│                    Runtime 层架构                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  API Layer (FastAPI)                                        │
│  ├─ /v1/sandboxes 路由                                      │
│  └─ 调用 SandboxService 接口                                │
│                                                             │
│  Service Layer                                              │
│  ┌─────────────────────────────────────────────────────┐   │
│  │           SandboxService (抽象接口)                  │   │
│  │  ├─ create_sandbox()                                 │   │
│  │  ├─ get_sandbox()                                    │   │
│  │  ├─ list_sandboxes()                                 │   │
│  │  ├─ delete_sandbox()                                 │   │
│  │  └─ renew_expiration()                               │   │
│  └───────────────────┬─────────────────────────────────┘   │
│                      │                                      │
│          ┌───────────┴───────────┐                         │
│          ▼                       ▼                         │
│  ┌─────────────────┐     ┌─────────────────────┐           │
│  │ DockerRuntime   │     │ KubernetesRuntime   │           │
│  │                 │     │                     │           │
│  │ DockerSandbox   │     │ ├─ BatchSandbox     │           │
│  │ Service         │     │ │   Provider        │           │
│  │                 │     │ ├─ AgentSandbox     │           │
│  │ docker-py       │     │ │   Provider        │           │
│  │ SDK             │     │ └─ K8s Client       │           │
│  └─────────────────┘     └─────────────────────┘           │
│                                                             │
│  Infrastructure Layer                                       │
│  ├─ Docker Engine API                                       │
│  └─ Kubernetes API Server                                   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 设计原则

**工厂模式创建服务实例**

```python
# factory.py
def create_sandbox_service(service_type: str, config: AppConfig) -> SandboxService:
    implementations = {
        "docker": DockerSandboxService,
        "kubernetes": KubernetesSandboxService,
    }
    return implementations[service_type](config=config)
```

这种设计允许：
- 运行时根据配置选择实现
- 新增 Runtime 实现无需修改 API 层
- 便于单元测试和 Mock

---

## 2. 抽象接口设计

### 2.1 SandboxService 接口

```python
# sandbox_service.py
class SandboxService(ABC):
    """沙箱服务抽象基类"""
    
    @abstractmethod
    def create_sandbox(self, request: CreateSandboxRequest) -> CreateSandboxResponse:
        """创建沙箱"""
        pass
    
    @abstractmethod
    def get_sandbox(self, sandbox_id: str) -> Sandbox:
        """获取沙箱状态"""
        pass
    
    @abstractmethod
    def list_sandboxes(self, request: ListSandboxesRequest) -> ListSandboxesResponse:
        """列出沙箱"""
        pass
    
    @abstractmethod
    def delete_sandbox(self, sandbox_id: str) -> bool:
        """删除沙箱"""
        pass
    
    @abstractmethod
    def renew_expiration(self, request: RenewSandboxExpirationRequest) -> RenewSandboxExpirationResponse:
        """延长过期时间"""
        pass
```

### 2.2 数据模型

**创建请求模型：**

```
CreateSandboxRequest
├─ image: str                    # 容器镜像
├─ entrypoint: List[str]         # 入口命令
├─ env: Dict[str, str]           # 环境变量
├─ resources: ResourceSpec       # 资源限制
├─ timeout: timedelta            # 超时时间
├─ volumes: List[Volume]         # 存储卷
├─ network_policy: NetworkPolicy # 网络策略
└─ metadata: Dict[str, str]      # 元数据标签
```

**沙箱状态模型：**

```
Sandbox
├─ sandbox_id: str               # 唯一标识
├─ status: SandboxStatus         # 状态信息
│   ├─ state: enum               # Pending/Running/Paused/Terminated
│   ├─ reason: str               # 状态原因
│   └─ message: str              # 状态消息
├─ created_at: datetime          # 创建时间
├─ expires_at: datetime          # 过期时间
├─ endpoints: List[Endpoint]     # 端点列表
└─ metadata: Dict[str, str]      # 元数据
```

---

## 3. Docker Runtime 实现

### 3.1 初始化流程

```
DockerSandboxService.__init__()
    │
    ├─→ 1. 验证配置
    │       runtime.type == "docker"
    │
    ├─→ 2. 初始化 Docker 客户端
    │       docker.from_env()
    │       └─ 读取 DOCKER_HOST 环境变量
    │
    ├─→ 3. 恢复已有沙箱
    │       _restore_existing_sandboxes()
    │       └─ 重建过期计时器
    │
    └─→ 4. 初始化安全运行时解析器
            SecureRuntimeResolver(config)
```

**关键代码逻辑：**

```python
def __init__(self, config: AppConfig):
    # 验证配置
    if config.runtime.type != "docker":
        raise ValueError("DockerSandboxService requires runtime.type = 'docker'")
    
    # 初始化 Docker 客户端
    self.docker_client = docker.from_env(timeout=180)
    
    # 恢复已有沙箱的过期计时器
    self._restore_existing_sandboxes()
    
    # 初始化安全运行时
    self.resolver = SecureRuntimeResolver(config)
    self.docker_runtime = self.resolver.get_docker_runtime()
```

### 3.2 创建沙箱流程

```
create_sandbox(request)
    │
    ├─→ 1. 参数验证
    │       ├─ ensure_entrypoint()
    │       ├─ ensure_future_expiration()
    │       ├─ ensure_volumes_valid()
    │       └─ ensure_metadata_labels()
    │
    ├─→ 2. 生成沙箱 ID
    │       sandbox_id = str(uuid4())
    │
    ├─→ 3. 拉取用户镜像
    │       docker_client.images.pull(request.image)
    │
    ├─→ 4. 准备 execd 注入
    │       ├─ 拉取 execd 镜像
    │       ├─ 创建临时容器
    │       ├─ 提取 execd 二进制
    │       └─ 缓存到内存
    │
    ├─→ 5. 构建容器配置
    │       ├─ 资源限制 (CPU/Memory)
    │       ├─ 环境变量注入
    │       ├─ execd 挂载卷
    │       └─ 网络模式配置
    │
    ├─→ 6. 创建并启动容器
    │       ├─ container.create()
    │       └─ container.start()
    │
    ├─→ 7. 配置过期计时器
    │       _schedule_expiration(sandbox_id, expires_at)
    │
    └─→ 8. 返回响应
            CreateSandboxResponse(sandbox_id, status="pending")
```

### 3.3 execd 注入机制

execd 通过**动态注入**方式进入容器，无需修改用户镜像：

```
execd 注入流程：
    │
    ├─→ 1. 拉取 execd 镜像
    │       opensandbox/execd:v1.0.6
    │
    ├─→ 2. 创建临时容器并启动
    │       docker.create(image=execd_image)
    │       docker.start()
    │
    ├─→ 3. 从容器提取 execd 二进制
    │       container.get_archive("/execd")
    │       └─ 返回 tar 流
    │
    ├─→ 4. 缓存到内存（避免重复提取）
    │       self._execd_archive_cache = data
    │
    └─→ 5. 创建沙箱时挂载
            volumes = ["/tmp/execd:/opt/opensandbox/execd:ro"]
```

**挂载配置示例：**

```python
container_config = {
    "Image": request.image,
    "Entrypoint": ["/opt/opensandbox/bootstrap.sh"],
    "Env": [f"{k}={v}" for k, v in env.items()],
    "HostConfig": {
        "Binds": [
            f"{execd_path}:/opt/opensandbox/execd:ro",
            f"{bootstrap_path}:/opt/opensandbox/bootstrap.sh:ro",
        ],
        "PortBindings": {"44772/tcp": [{"HostPort": "0"}]},
    }
}
```

### 3.4 过期管理机制

Docker Runtime 使用 **Python Timer** 实现自动过期：

```
过期管理流程：
    │
    ├─→ 创建时
    │       _schedule_expiration(sandbox_id, expires_at)
    │       └─ Timer(delay, _expire_sandbox).start()
    │
    ├─→ 续期时
    │       renew_expiration()
    │       └─ 取消旧 Timer，创建新 Timer
    │
    └─→ 过期触发
            _expire_sandbox(sandbox_id)
            ├─ container.kill()
            ├─ container.remove(force=True)
            └─ _cleanup_egress_sidecar()
```

**关键代码逻辑：**

```python
def _schedule_expiration(self, sandbox_id: str, expires_at: datetime):
    delay = max(0, (expires_at - datetime.now(timezone.utc)).total_seconds())
    timer = Timer(delay, self._expire_sandbox, args=(sandbox_id,))
    timer.daemon = True
    timer.start()
    self._expiration_timers[sandbox_id] = timer

def _expire_sandbox(self, sandbox_id: str):
    container = self._get_container_by_sandbox_id(sandbox_id)
    container.kill()
    container.remove(force=True)
    self._remove_expiration_tracking(sandbox_id)
```

### 3.5 网络模式

Docker Runtime 支持两种网络模式：

| 模式 | 配置 | 特点 | 适用场景 |
|------|------|------|----------|
| **Host** | `network_mode: host` | 容器共享主机网络 | 单实例开发 |
| **Bridge** | `network_mode: bridge` | 容器独立网络命名空间 | 需要端口隔离 |

```
Host 模式：
    容器端口直接映射到主机
    无需端口映射配置
    多实例会端口冲突

Bridge 模式：
    容器有独立 IP
    需要端口映射
    支持多实例
```

---

## 4. Kubernetes Runtime 实现

### 4.1 架构设计

Kubernetes Runtime 采用**分层架构**，支持多种 Workload Provider：

```
KubernetesSandboxService
    │
    ├─→ K8sClient (Kubernetes API 封装)
    │
    └─→ WorkloadProvider (工作负载抽象)
            │
            ├─→ BatchSandboxProvider (推荐)
            │       └─ 使用 BatchSandbox CRD
            │       └─ 支持批量创建、池化
            │
            └─→ AgentSandboxProvider
                    └─ 使用 SIG agent-sandbox CRD
```

### 4.2 初始化流程

```
KubernetesSandboxService.__init__()
    │
    ├─→ 1. 验证配置
    │       runtime.type == "kubernetes"
    │       kubernetes config 存在
    │
    ├─→ 2. 初始化 Kubernetes 客户端
    │       K8sClient(config)
    │       └─ 加载 kubeconfig 或 InCluster 配置
    │
    ├─→ 3. 创建 Workload Provider
    │       create_workload_provider(provider_type)
    │       └─ BatchSandboxProvider 或 AgentSandboxProvider
    │
    └─→ 4. 初始化 Informer (可选)
            WorkloadInformer
            └─ 监听资源变化，缓存状态
```

### 4.3 创建沙箱流程

```
create_sandbox(request)
    │
    ├─→ 1. 参数验证
    │       同 Docker Runtime
    │
    ├─→ 2. 生成沙箱 ID
    │
    ├─→ 3. 构建 Workload 规格
    │       ├─ 容器镜像
    │       ├─ 资源限制
    │       ├─ 环境变量
    │       ├─ execd initContainer
    │       └─ 安全上下文
    │
    ├─→ 4. 创建 Kubernetes 资源
    │       workload_provider.create_workload(spec)
    │       └─ 创建 BatchSandbox / Pod
    │
    ├─→ 5. 等待就绪
    │       _wait_for_sandbox_ready(sandbox_id)
    │       └─ 轮询直到 Pod Running
    │
    └─→ 6. 返回响应
```

### 4.4 BatchSandbox Provider

BatchSandbox 是 OpenSandbox 自定义的 CRD，专为**批量沙箱场景**优化：

**CRD 定义：**

```yaml
apiVersion: sandbox.opensandbox.io/v1alpha1
kind: BatchSandbox
metadata:
  name: sandbox-550e8400-e29b-41d4
  labels:
    opensandbox.io/sandbox-id: 550e8400-e29b-41d4
spec:
  template:
    spec:
      initContainers:
        - name: execd-installer
          image: opensandbox/execd:v1.0.6
          # 注入 execd 二进制
      containers:
        - name: sandbox
          image: python:3.11
          command: ["/opt/opensandbox/bootstrap.sh"]
          resources:
            limits:
              cpu: "1"
              memory: "512Mi"
```

**创建流程：**

```python
def create_workload(self, sandbox_id: str, spec: dict) -> dict:
    # 构建 BatchSandbox CR
    body = {
        "apiVersion": f"{self.group}/{self.version}",
        "kind": "BatchSandbox",
        "metadata": {
            "name": f"sandbox-{sandbox_id}",
            "labels": labels,
        },
        "spec": self._build_spec(spec),
    }
    
    # 调用 Kubernetes API 创建
    return self.custom_api.create_namespaced_custom_object(
        group=self.group,
        version=self.version,
        namespace=self.namespace,
        plural=self.plural,
        body=body,
    )
```

### 4.5 池化机制 (Pool)

Pool 是 Kubernetes Runtime 的核心特性，实现**预热沙箱**：

```
Pool 架构：
    │
    ├─→ Pool CRD
    │       ├─ template: Pod 模板
    │       └─ capacitySpec: 容量配置
    │
    ├─→ Pool Controller
    │       ├─ 维护最小池容量
    │       ├─ 监听资源变化
    │       └─ 自动补充预热实例
    │
    └─→ 获取沙箱
            从池中分配 → 立即可用 (< 100ms)
```

**Pool 配置示例：**

```yaml
apiVersion: sandbox.opensandbox.io/v1alpha1
kind: Pool
metadata:
  name: python-code-interpreter
spec:
  template:
    spec:
      containers:
        - name: sandbox
          image: opensandbox/code-interpreter:v1.0.1
  capacitySpec:
    poolMin: 10      # 最小预热数量
    poolMax: 100     # 最大实例数
    bufferMin: 5     # 最小缓冲
    bufferMax: 20    # 最大缓冲
```

**从 Pool 获取沙箱：**

```python
# 创建请求指定 poolRef
request = CreateSandboxRequest(
    image="",  # Pool 模式忽略
    extensions={"poolRef": "python-code-interpreter"},
    entrypoint=["python", "main.py"],
    env={"KEY": "value"},
)

# BatchSandboxProvider 检测到 poolRef 后
# 直接从预热的 Pool 中分配沙箱
# 获取延迟 < 100ms
```

### 4.6 安全容器支持

Kubernetes Runtime 支持多种安全容器运行时：

```
安全容器配置：
    │
    ├─→ runtimeClass 配置
    │       spec.runtimeClassName: kata-containers
    │
    └─→ SecureRuntimeResolver 解析
            ├─ Docker: --runtime=kata
            └─ K8s: runtimeClassName
```

**支持的运行时：**

| 运行时 | runtimeClass | 隔离级别 |
|--------|--------------|----------|
| **gVisor** | `gvisor` | 系统调用拦截 |
| **Kata Containers** | `kata-containers` | 硬件虚拟化 |
| **Firecracker** | `firecracker` | 微虚拟机 |

### 4.7 Informer 机制

Informer 是 Kubernetes 的核心设计模式，用于高效监听资源变化：

```
WorkloadInformer
    │
    ├─→ List-Watch 机制
    │       ├─ List: 获取全量资源
    │       └─ Watch: 监听增量变化
    │
    ├─→ 本地缓存
    │       store: Dict[sandbox_id, workload]
    │
    └─→ 事件处理
            ├─ OnAdd: 资源创建
            ├─ OnUpdate: 资源更新
            └─ OnDelete: 资源删除
```

**优势：**
- 减少对 API Server 的请求压力
- 快速响应状态查询（从缓存读取）
- 实时感知资源变化

---

## 5. 关键流程对比

### 5.1 创建流程对比

| 步骤 | Docker Runtime | Kubernetes Runtime |
|------|----------------|-------------------|
| **镜像拉取** | `docker pull` | Pod ImagePullPolicy |
| **execd 注入** | 卷挂载 | initContainer |
| **资源创建** | `docker.create()` | 创建 CRD 资源 |
| **状态等待** | 轮询容器状态 | 轮询 Pod 状态 |
| **端口暴露** | PortBindings | Service/Ingress |

### 5.2 性能对比

| 指标 | Docker Runtime | Kubernetes Runtime |
|------|----------------|-------------------|
| **冷启动** | 2-5 秒 | 5-15 秒 |
| **预热池** | 不支持 | < 100ms |
| **批量创建 (100)** | ~76 秒 | ~0.92 秒 |
| **水平扩展** | 有限 | 原生支持 |

### 5.3 适用场景

| 场景 | 推荐方案 | 原因 |
|------|----------|------|
| **本地开发** | Docker Runtime | 简单、无需 K8s 集群 |
| **CI/CD** | Docker Runtime | 快速启动、资源隔离 |
| **生产环境** | Kubernetes Runtime | 高可用、自动扩缩容 |
| **AI Agent 服务** | Kubernetes Runtime + Pool | 毫秒级响应 |
| **多租户** | Kubernetes Runtime + 安全容器 | 强隔离 |

---

## 6. 总结

### 6.1 架构亮点

| 设计点 | 实现方式 | 收益 |
|--------|----------|------|
| **抽象接口** | `SandboxService` 基类 | Runtime 可替换 |
| **工厂模式** | `create_sandbox_service()` | 配置驱动创建 |
| **execd 注入** | 卷挂载 / initContainer | 用户镜像零侵入 |
| **过期管理** | Timer / TTL 注解 | 自动清理资源 |
| **池化预热** | Pool CRD | 亚秒级响应 |

### 6.2 扩展点

```
新增 Runtime 实现：
    │
    ├─→ 1. 实现 SandboxService 接口
    │
    ├─→ 2. 注册到 factory.py
    │       implementations["containerd"] = ContainerdSandboxService
    │
    └─→ 3. 配置 runtime.type
            [runtime]
            type = "containerd"
```

### 6.3 最佳实践

**开发环境：**
```toml
[runtime]
type = "docker"

[docker]
network_mode = "host"
```

**生产环境：**
```toml
[runtime]
type = "kubernetes"

[kubernetes]
namespace = "opensandbox"
workload_provider = "batchsandbox"

[security]
runtime_class = "kata-containers"
```

---

> **关于本文档**  
> 本文档基于 OpenSandbox Server 源码分析，聚焦 Runtime 层的实现细节。  
> 作者：sandboxrosy | 日期：2026-03-13