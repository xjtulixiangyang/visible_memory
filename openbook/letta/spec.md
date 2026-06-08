# Letta 项目规格说明书 (Specification)

> 版本: 0.16.8 | 生成日期: 2026-06-07

## 1. 项目概述

### 1.1 项目名称

**Letta** (formerly MemGPT) — 构建具备高级记忆能力、可自我学习与改进的 AI Agent 基础设施

### 1.2 项目定位

Letta 是一个开源的 AI Agent 框架，核心特性是让 LLM Agent 拥有持久化的多层记忆系统（Core Memory / Recall Memory / Archival Memory），使 Agent 能够跨会话保留和检索信息，实现持续学习与自我改进。

### 1.3 核心价值主张

| 能力 | 描述 |
|------|------|
| **持久记忆** | 三层记忆架构，Agent 可读写核心记忆、检索历史对话、搜索归档知识 |
| **多模型支持** | 支持 20+ LLM Provider（OpenAI、Anthropic、Google、Bedrock、Azure 等） |
| **多 Agent 协作** | Sleeptime、Supervisor、RoundRobin、Dynamic 四种多 Agent 模式 |
| **工具系统** | 内置记忆/文件/搜索工具 + MCP 协议集成外部工具 |
| **流式响应** | SSE + WebSocket 双通道实时流式输出 |
| **可观测性** | OpenTelemetry 集成，全链路追踪与指标采集 |

## 2. 技术栈

### 2.1 运行时依赖

| 类别 | 技术 | 版本 |
|------|------|------|
| 语言 | Python | >=3.11, <3.14 |
| Web 框架 | FastAPI + Uvicorn | - |
| ORM | SQLAlchemy 2.0 (async) | >=2.0.41 |
| 数据验证 | Pydantic v2 | >=2.10.6 |
| 数据库迁移 | Alembic | >=1.13.3 |
| LLM SDK | OpenAI / Anthropic / Google GenAI | - |
| 向量存储 | Turbopuffer / pgvector | - |
| 可观测性 | OpenTelemetry | 1.30.0 |
| MCP | mcp[cli] / fastmcp | >=1.9.4 / >=2.12.5 |
| 任务调度 | APScheduler | >=3.11.0 |
| 序列化 | Marshmallow + orjson | - |

### 2.2 数据库

- **主数据库**: PostgreSQL（生产）/ SQLite（开发）
- **向量引擎**: Turbopuffer（云端）/ pgvector（自托管）
- **缓存**: Redis（流管理、数据源连接器）

## 3. 系统功能规格

### 3.1 Agent 管理

| 功能 | 描述 |
|------|------|
| 创建 Agent | 支持指定 LLM 配置、记忆块、工具列表、系统提示 |
| 更新 Agent | 修改 Agent 状态、配置、工具绑定 |
| 删除 Agent | 软删除，保留历史数据 |
| Agent 类型 | letta_v1_agent / letta_v2_agent / sleeptime_agent / voice_convo_agent |
| 多 Agent 组 | 支持创建 Agent 组，配置协作模式 |

### 3.2 记忆系统

| 功能 | 描述 |
|------|------|
| Core Memory | Agent 的"工作记忆"，由 Block 组成，直接注入上下文窗口 |
| Recall Memory | 历史对话记录，支持时间/关键词/语义检索 |
| Archival Memory | 外部知识库，通过 Passage 存储和语义检索 |
| Block 管理 | CRUD + Checkpoint/Undo/Redo + Git 版本化 |
| 上下文压缩 | 自动摘要、滑动窗口、消息截断三级降级策略 |

### 3.3 消息处理

| 功能 | 描述 |
|------|------|
| 同步消息 | 发送消息并等待完整响应 |
| 流式消息 | SSE 实时推送 Agent 思考和工具调用过程 |
| 异步消息 | 提交消息后通过 Job 轮询结果 |
| OpenAI 兼容 | 支持 /v1/chat/completions 接口 |

### 3.4 工具系统

| 功能 | 描述 |
|------|------|
| 内置工具 | 记忆读写、归档搜索、消息发送 |
| 自定义工具 | Python 函数注册，自动生成 JSON Schema |
| MCP 工具 | 通过 MCP 协议连接外部工具服务器 |
| 工具规则 | 声明式规则控制工具调用顺序和权限 |
| 沙箱执行 | Docker/E2B 沙箱隔离执行自定义代码 |

### 3.5 LLM 集成

| 功能 | 描述 |
|------|------|
| 多 Provider | OpenAI、Anthropic、Google、Azure、Bedrock、Together、Groq 等 |
| 流式/非流式 | 统一接口支持阻塞和流式两种调用模式 |
| BYOK | 用户自带 API Key，支持 Letta 托管和自托管两种密钥管理 |
| 批处理 | 支持 OpenAI Batch API 异步批量推理 |
| Token 追踪 | 完整的 Token 使用量统计和费用追踪 |

### 3.6 数据源

| 功能 | 描述 |
|------|------|
| 文件加载 | 支持 PDF、TXT、DOCX、PPTX 等格式 |
| 自动分块 | 按行/段落/Token 自动分块 |
| 向量嵌入 | 自动嵌入并存储到向量数据库 |
| 语义搜索 | 基于向量相似度的 Passage 检索 |

## 4. API 规格

### 4.1 REST API

基础路径: `/v1`

| 端点组 | 路径前缀 | 描述 |
|--------|----------|------|
| Agents | `/v1/agents` | Agent CRUD、消息发送 |
| Messages | `/v1/agents/{id}/messages` | 同步/流式/异步消息 |
| Tools | `/v1/tools` | 工具管理 |
| Sources | `/v1/sources` | 数据源管理 |
| Blocks | `/v1/blocks` | 记忆块管理 |
| Groups | `/v1/groups` | 多 Agent 组管理 |
| Jobs | `/v1/jobs` | 异步任务管理 |
| Conversations | `/v1/conversations` | 对话管理 |
| MCP Servers | `/v1/mcp-servers` | MCP 服务器管理 |
| Providers | `/v1/providers` | LLM Provider 管理 |
| Users | `/v1/users` | 用户管理 |
| Organizations | `/v1/organizations` | 组织管理 |
| Health | `/v1/health` | 健康检查 |
| Chat Completions | `/v1/chat/completions` | OpenAI 兼容接口 |

### 4.2 WebSocket API

路径: `/v1/ws`

支持实时双向通信，用于低延迟的 Agent 交互场景。

## 5. 非功能性规格

### 5.1 性能

- 支持并发 Agent 消息处理（基于 asyncio）
- 数据库连接池管理
- 流式响应 Keepalive 保活机制

### 5.2 可靠性

- 数据库死锁自动重试
- 乐观锁并发控制
- 上下文窗口溢出自动压缩
- Agent 执行异常优雅降级

### 5.3 安全

- API Key 认证
- 密码中间件保护
- 沙箱隔离执行用户代码
- MCP OAuth 认证支持
- 软删除数据保护

### 5.4 可观测性

- OpenTelemetry 全链路追踪
- ClickHouse Provider Trace 存储
- 自定义指标注册表
- 结构化日志（structlog）

## 6. 项目结构

```
letta/
├── agents/           # Agent 实现（BaseAgent, LettaAgent, VoiceAgent 等）
├── adapters/         # LLM 适配器层
├── cli/              # 命令行工具
├── client/           # 客户端工具
├── data_sources/     # 数据源连接器
├── functions/        # 工具/函数系统
│   ├── function_sets/ # 内置工具集
│   └── mcp_client/   # MCP 客户端
├── groups/           # 多 Agent 协作模式
├── helpers/          # 通用辅助工具
├── interfaces/       # 流式接口适配
├── jobs/             # 任务调度
├── llm_api/          # LLM API 客户端
├── local_llm/        # 本地 LLM 支持
├── model_specs/      # 模型规格
├── monitoring/       # 监控（看门狗、负载门控）
├── orm/              # SQLAlchemy ORM 模型
├── otel/             # OpenTelemetry 集成
├── personas/         # 预设人设
├── plugins/          # 插件系统
├── prompts/          # 系统提示模板
├── schemas/          # Pydantic 数据模型
│   ├── openai/       # OpenAI 兼容 Schema
│   └── providers/    # Provider Schema
├── serialize_schemas/ # 序列化 Schema（Marshmallow）
├── server/           # 服务器
│   ├── rest_api/     # REST API（FastAPI）
│   │   ├── auth/     # 认证
│   │   ├── middleware/ # 中间件
│   │   └── routers/  # 路由
│   └── ws_api/       # WebSocket API
└── services/         # 业务逻辑服务层
    ├── context_window_calculator/ # 上下文窗口计算
    ├── file_processor/  # 文件处理
    ├── helpers/         # 服务辅助
    ├── lettuce/         # Letta Cloud 客户端
    ├── llm_router/      # LLM 路由
    └── mcp/             # MCP 服务
```

## 7. 核心模块索引

| 模块 | 设计文档 | 描述 |
|------|----------|------|
| Agent 系统 | [agent-system.md](./agent-system.md) | Agent 生命周期、执行循环、多 Agent 协作 |
| LLM 抽象层 | [llm-abstraction.md](./llm-abstraction.md) | 多 Provider 路由、请求/响应归一化、流式处理 |
| Memory 记忆系统 | [memory-system.md](./memory-system.md) | 三层记忆架构、Block 管理、上下文压缩 |
| Server/API 层 | [server-api.md](./server-api.md) | REST/WebSocket API、流式响应、认证中间件 |
| ORM 持久化层 | [orm-persistence.md](./orm-persistence.md) | 数据模型、Schema 映射、多租户隔离 |
| Tool/Function 系统 | [tool-function.md](./tool-function.md) | 工具注册/执行、MCP 集成、规则系统 |
