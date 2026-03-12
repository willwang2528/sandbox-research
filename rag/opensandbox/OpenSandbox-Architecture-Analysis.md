# OpenSandbox 架构深度分析

> 作者：sandboxrosy | 日期：2026-03-12  
> 来源：GitHub alibaba/OpenSandbox 源码分析

---

## 1. 项目概览

**OpenSandbox** 是阿里巴巴开源的通用沙箱平台，专为 AI 应用场景设计。它提供多语言 SDK、标准化沙箱协议和灵活的运行时实现，支持 Coding Agents、GUI Agents、Agent Evaluation、AI Code Execution 和 RL Training 等场景。

| 特性 | 说明 |
|------|------|
| **开源协议** | Apache 2.0 |
| **语言支持** | Python, Java/Kotlin, TypeScript/JavaScript, C#/.NET, Go (Roadmap) |
| **运行时** | Docker (生产就绪), Kubernetes (生产就绪) |
| **隔离方案** | 原生容器、gVisor、Kata Containers、Firecracker microVM |

---

## 2. 四层架构设计

OpenSandbox 采用清晰的四层架构，从上到下依次为：

```
┌─────────────────────────────────────────────────────────────┐
│                    SDKs Layer                                │
│    Python | Java/Kotlin | TypeScript | C#/.NET | Go       │
├─────────────────────────────────────────────────────────────┤
│                    Specs Layer                               │
│    Sandbox Lifecycle API | Sandbox Execution API            │
├─────────────────────────────────────────────────────────────┤
│                   Runtime Layer                              │
│         Docker Runtime | Kubernetes Runtime                 │
├─────────────────────────────────────────────────────────────┤
│                Sandbox Instances Layer                       │
│    Base Container + execd Daemon + Entrypoint Process        │
└─────────────────────────────────────────────────────────────┘
```

### 2.1 SDKs 层

SDK 层提供开发者友好的高级抽象，封装了与沙箱交互的所有操作：

| 组件 | 功能 |
|------|------|
| **Sandbox** | 沙箱生命周期管理：创建、监控、续期、销毁 |
| **Filesystem** | 文件系统操作：CRUD、批量上传下载、搜索、权限管理 |
| **Commands** | 命令执行：前台/后台执行、SSE 流式输出、进程控制 |
| **CodeInterpreter** | 代码解释器：多语言有状态执行、Jupyter 内核集成 |

**关键特性：**
- 异步/同步 API 支持
- 自动状态轮询
- 资源配额管理 (CPU, Memory, GPU)
- TTL 自动过期与续期

### 2.2 Specs 层

协议层定义了 SDK 与运行时之间的契约：

#### Sandbox Lifecycle Spec (`specs/sandbox-lifecycle.yml`)

| 操作 | 端点 | 说明 |
|------|------|------|
| Create | `POST /sandboxes` | 从容器镜像创建沙箱 |
| List | `GET /sandboxes` | 列出沙箱（支持过滤和分页） |
| Get | `GET /sandboxes/{id}` | 获取沙箱详情 |
| Delete | `DELETE /sandboxes/{id}` | 销毁沙箱 |
| Pause | `POST /sandboxes/{id}/pause` | 暂停沙箱 |
| Resume | `POST /sandboxes/{id}/resume` | 恢复沙箱 |
| Renew | `POST /sandboxes/{id}/renew-expiration` | 续期 TTL |
| Endpoint | `GET /sandboxes/{id}/endpoints/{port}` | 获取端口访问地址 |

#### Sandbox Execution Spec (`specs/execd-api.yaml`)

| 类别 | 端点 | 说明 |
|------|------|------|
| Health | `GET /ping` | 健康检查 |
| Code | `POST /code` | 执行代码（SSE 流式） |
| Command | `POST /command` | 执行 Shell 命令 |
| Files | `POST /files/upload` | 上传文件 |
| Files | `GET /files/download` | 下载文件 |
| Metrics | `GET /metrics` | 获取系统指标 |

### 2.3 Runtime 层

运行时层实现了 Lifecycle Spec，负责沙箱容器的编排和管理：

#### Docker Runtime

```toml
[runtime]
type = "docker"
execd_image = "opensandbox/execd:v1.0.6"

[docker]
network_mode = "host"  # 或 "bridge"
```

**特性：**
- 直接 Docker API 集成
- 两种网络模式：Host（共享主机网络）和 Bridge（隔离网络）
- 容器生命周期管理
- 资源配额强制执行
- 私有镜像仓库认证

#### Kubernetes Runtime

**核心特性：**
- **BatchSandbox** CRD：批量沙箱创建，吞吐量提升 80x+
- **Pool** CRD：资源池预热，毫秒级沙箱获取
- **Task Orchestration**：内置任务执行引擎
- 支持 gVisor、Kata Containers、Firecracker 等安全容器运行时

**性能对比（100 个沙箱）：**
| 方案 | 总耗时 |
|------|--------|
| SIG Agent-Sandbox (并发=1) | 76.35s |
| SIG Agent-Sandbox (并发=10) | 23.17s |
| SIG Agent-Sandbox (并发=50) | 33.85s |
| **BatchSandbox** | **0.92s** |

### 2.4 Sandbox Instances 层

每个沙箱实例包含三个组件：

```
┌────────────────────────────────────────────┐
│           Sandbox Instance                  │
├────────────────────────────────────────────┤
│  1. Base Container (用户镜像)               │
│     - ubuntu:22.04, python:3.11, etc.      │
├────────────────────────────────────────────┤
│  2. execd Daemon (注入的执行代理)           │
│     - HTTP API 服务                         │
│     - Jupyter 内核管理                      │
│     - 文件系统操作                          │
│     - 指标采集                              │
├────────────────────────────────────────────┤
│  3. Entrypoint Process (用户进程)          │
│     - 用户定义的启动命令                    │
└────────────────────────────────────────────┘
```

---

## 3. 核心组件详解

### 3.1 execd - 执行守护进程

**位置：** `components/execd/`

execd 是注入到每个沙箱容器中的高性能 Go 守护进程，基于 Beego 框架构建。

#### 核心职责

| 功能 | 实现方式 |
|------|----------|
| **代码执行** | Jupyter 内核会话管理，支持 Python/Java/JS/TS/Go/Bash |
| **命令执行** | 前台/后台 Shell 命令，SSE 流式输出 |
| **文件操作** | 完整的文件系统 CRUD API |
| **指标采集** | CPU、内存、运行时间监控 |

#### 技术栈

```
Go 1.24+ + Beego + Jupyter Protocol (WebSocket) + SSE
```

#### 包结构

| 路径 | 用途 |
|------|------|
| `pkg/flag/` | 配置和 CLI 参数 |
| `pkg/web/` | HTTP 层（控制器、模型、路由） |
| `pkg/runtime/` | 执行分发器 |
| `pkg/jupyter/` | Jupyter 内核客户端 |
| `pkg/util/` | 工具函数 |

#### 注入机制

```
容器启动
    ↓
修改 entrypoint → /opt/opensandbox/start.sh
    ↓
启动 Jupyter Server (端口 54321)
    ↓
启动 execd (端口 44772)
    ↓
执行用户 entrypoint
```

### 3.2 Egress Sidecar - 出口流量控制

**位置：** `components/egress/`

提供 FQDN 级别的出口流量控制，与沙箱应用容器共享网络命名空间。

#### 核心特性

| 特性 | 说明 |
|------|------|
| **FQDN 白名单** | 按域名控制出站流量（如 `api.github.com`） |
| **通配符支持** | 支持子域名通配（如 `*.pypi.org`） |
| **透明代理** | DNS 代理 + nftables，无需修改应用配置 |
| **动态 DNS** | 域名解析后自动将 IP 加入 nftables 规则 |

#### 架构层次

```
┌─────────────────────────────────────┐
│        Application Container         │
├─────────────────────────────────────┤
│  Layer 1: DNS Proxy (127.0.0.1:15353) │
│  - 过滤 DNS 查询                      │
│  - NXDOMAIN 返回被拒绝的域名          │
├─────────────────────────────────────┤
│  Layer 2: nftables (dns+nft 模式)     │
│  - IP 级别允许/拒绝                   │
│  - 动态 DNS IP 集合                   │
└─────────────────────────────────────┘
```

### 3.3 Ingress - 入口流量路由

**位置：** `components/ingress/`

HTTP/WebSocket 反向代理，将请求路由到沙箱实例。

#### 路由模式

| 模式 | 格式 | 示例 |
|------|------|------|
| **Header** | `OpenSandbox-Ingress-To: <sandbox-id>-<port>` | `curl -H "OpenSandbox-Ingress-To: my-sandbox-8080"` |
| **URI** | `/<sandbox-id>/<port>/<path>` | `curl https://ingress.io/my-sandbox/8080/api` |

#### Kubernetes 集成

- 支持 **BatchSandbox** CR（读取 `sandbox.opensandbox.io/endpoints` 注解）
- 支持 **AgentSandbox** CR（读取 `status.serviceFQDN`）

---

## 4. 沙箱生命周期状态机

```
          ┌──────────┐
          │ Pending  │ ← 创建中
          └────┬─────┘
               │ 创建完成
               ▼
          ┌──────────┐
          │ Running  │ ← 正常运行
          └────┬─────┘
               │
       ┌───────┼───────┐
       │       │       │
       ▼       │       ▼
  ┌─────────┐  │  ┌─────────┐
  │ Pausing │  │  │ Stopping│
  └────┬────┘  │  └────┬────┘
       │       │       │
       ▼       │       ▼
  ┌─────────┐  │  ┌───────────┐
  │ Paused  │──┘  │ Terminated│ ← 已销毁
  └─────────┘     └───────────┘
       │
       │ 恢复
       ▼
  ┌──────────┐
  │ Running  │
  └──────────┘
```

---

## 5. 通信流程

### 5.1 沙箱创建流程

```
用户/SDK
    │
    │ 1. POST /sandboxes (image, entrypoint, resources)
    ▼
Server (Lifecycle API)
    │
    │ 2. 拉取容器镜像
    │ 3. 注入 execd 二进制
    │ 4. 创建容器（修改 entrypoint）
    │ 5. 启动容器
    ▼
Sandbox Instance
    │
    │ 6. 启动 execd 守护进程
    │ 7. 启动 Jupyter Server
    │ 8. 执行用户 entrypoint
    ▼
Running (状态)
```

### 5.2 代码执行流程

```
用户/SDK
    │
    │ 1. 创建沙箱
    │ 2. 获取 execd 端点
    ▼
CodeInterpreter SDK
    │
    │ 3. POST /code/context (创建会话)
    │ 4. POST /code (执行代码)
    ▼
execd (Execution API)
    │
    │ 5. 路由到 Jupyter 运行时
    ▼
Jupyter Runtime
    │
    │ 6. WebSocket 连接 Jupyter Server
    │ 7. 发送 execute_request
    ▼
Jupyter Kernel (Python/Java/etc.)
    │
    │ 8. 执行代码
    │ 9. 流式输出事件
    ▼
execd
    │
    │ 10. 转换为 SSE 事件
    │ 11. 流式返回客户端
    ▼
用户/应用
```

---

## 6. 安全隔离机制

### 6.1 安全容器运行时

OpenSandbox 支持多种安全容器运行时：

| 运行时 | 隔离级别 | 特点 |
|--------|----------|------|
| **runc** (默认) | 进程级 | 性能最佳，隔离最弱 |
| **gVisor** | 用户态内核 | 拦截系统调用，中等开销 |
| **Kata Containers** | VM 级 | 强隔离，较高开销 |
| **Firecracker** | microVM | 轻量级 VM，启动快 |

### 6.2 Docker 安全加固

```toml
[docker]
# 删除危险能力
drop_capabilities = ["AUDIT_WRITE", "MKNOD", "NET_ADMIN", "NET_RAW", "SYS_ADMIN"]
# 禁止权限提升
no_new_privileges = true
# 限制进程数（防止 fork bomb）
pids_limit = 512
```

### 6.3 网络隔离

- **入口**：通过 Ingress 路由，支持认证
- **出口**：Egress Sidecar 实现 FQDN 级别控制
- **API 认证**：`OPEN-SANDBOX-API-KEY` 头部认证

---

## 7. 部署模式

### 7.1 本地开发（Docker）

```bash
# 安装
uv pip install opensandbox-server

# 初始化配置
opensandbox-server init-config ~/.sandbox.toml --example docker

# 启动服务
opensandbox-server
```

### 7.2 生产部署（Kubernetes）

```bash
# Helm 安装
helm install opensandbox-controller \
  https://github.com/alibaba/OpenSandbox/releases/download/helm/opensandbox-controller/0.1.0/opensandbox-controller-0.1.0.tgz \
  --namespace opensandbox-system \
  --create-namespace
```

### 7.3 预置环境镜像

| 镜像 | 用途 |
|------|------|
| `opensandbox/code-interpreter` | 代码解释器 |
| `opensandbox/chrome` | 浏览器自动化 |
| `opensandbox/desktop` | VNC 桌面环境 |
| `opensandbox/vscode` | VS Code Web |

---

## 8. 集成生态

### 8.1 AI Agent 集成

| 集成 | 说明 |
|------|------|
| **Claude Code** | 在沙箱中运行 Claude Code |
| **Gemini CLI** | 在沙箱中运行 Google Gemini CLI |
| **Codex CLI** | 在沙箱中运行 OpenAI Codex CLI |
| **Kimi CLI** | 月之暗面 Kimi CLI 集成 |
| **LangGraph** | 状态机工作流集成 |
| **Google ADK** | Agent Development Kit 集成 |

### 8.2 预置沙箱示例

| 示例 | 用途 |
|------|------|
| `code-interpreter` | 端到端代码解释器 |
| `chrome` | Chromium 浏览器自动化 |
| `playwright` | Playwright 测试框架 |
| `desktop` | 完整 VNC 桌面 |
| `vscode` | VS Code 远程开发 |
| `rl-training` | 强化学习训练环境 |

---

## 9. 设计亮点

### 9.1 协议优先设计

- 所有交互由 OpenAPI 规范定义
- 组件间清晰契约
- 支持多语言实现
- 允许自定义运行时实现

### 9.2 关注点分离

```
SDK → 客户端抽象
Specs → 协议定义
Runtime → 沙箱编排
execd → 沙箱内执行
```

### 9.3 可扩展性

- 可插拔运行时实现
- 自定义沙箱镜像
- 多语言 SDK
- 扩展 Jupyter 内核

### 9.4 可观测性

- 结构化状态转换日志
- 实时指标流
- 完整日志记录
- 健康检查端点

---

## 10. 建设性批判

### 10.1 优点

1. **架构清晰**：四层分离，职责明确
2. **协议开放**：OpenAPI 规范，可替换实现
3. **性能优异**：BatchSandbox 吞吐量提升 80x+
4. **生态丰富**：多语言 SDK、预置镜像、Agent 集成
5. **安全增强**：支持多种安全容器运行时

### 10.2 潜在问题

| 问题 | 说明 | 建议 |
|------|------|------|
| **K8s 暂停/恢复** | Kubernetes 运行时不支持 pause/resume | 文档已说明，待后续实现 |
| **execd 注入开销** | 每个沙箱都需要注入 execd 二进制 | 考虑预构建镜像减少注入时间 |
| **Egress 依赖特权** | 需要 `CAP_NET_ADMIN` 权限 | 生产环境需评估安全策略 |
| **Go SDK 待实现** | Roadmap 中尚未发布 | 可先用 Python/Java SDK |

### 10.3 生产环境建议

1. **资源规划**：预热资源池以减少冷启动延迟
2. **安全加固**：使用 gVisor 或 Kata Containers
3. **网络策略**：配置 Egress 白名单限制出站流量
4. **监控告警**：集成 Prometheus/Grafana 监控沙箱状态
5. **日志收集**：收集 execd 和 Server 日志用于问题排查

---

## 11. 与其他方案对比

| 特性 | OpenSandbox | E2B | Modal | Firecracker |
|------|-------------|-----|-------|-------------|
| **开源** | ✅ Apache 2.0 | ❌ 专有 | ❌ 专有 | ✅ Apache 2.0 |
| **自托管** | ✅ | ❌ | ❌ | ✅ |
| **多语言 SDK** | ✅ 5+ | Python/JS | Python | - |
| **K8s 原生** | ✅ | ❌ | ❌ | ✅ (配合其他) |
| **安全容器** | ✅ gVisor/Kata/FC | ✅ Firecracker | ✅ | ✅ |
| **代码解释器** | ✅ 内置 | ✅ | ✅ | 需自建 |
| **网络控制** | ✅ FQDN 级别 | 有限 | 有限 | 需自建 |

---

## 12. 总结

OpenSandbox 是一个设计精良、功能完备的沙箱平台，适合 AI Agent 代码执行、远程开发环境、自动化测试等场景。其四层架构设计清晰，协议优先的理念使其具有良好的可扩展性。通过 Docker 和 Kubernetes 双运行时支持，可以平滑地从开发环境迁移到生产环境。

对于需要自托管、多语言支持、细粒度网络控制的团队，OpenSandbox 是一个值得考虑的开源方案。

---

*文档生成于 2026-03-12 | sandboxrosy*