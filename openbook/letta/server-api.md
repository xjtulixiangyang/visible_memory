# Letta Server/API 层模块设计文档

## 1. 模块概述

Letta Server/API 层是整个 Letta 系统的对外接口层，负责接收外部请求、协调内部服务、管理 Agent 生命周期并返回响应。其核心职责包括：

- **请求路由与分发**：基于 FastAPI 框架，将 HTTP/WebSocket 请求路由到对应的业务处理器
- **Agent 生命周期管理**：创建、更新、删除、导出/导入 Agent 实例
- **消息处理**：同步/异步/流式三种消息处理模式，支持 SSE 流式推送
- **工具管理**：自定义工具的 CRUD、MCP 服务器集成与 OAuth 流程
- **数据源管理**：文件上传、解析、向量化与归档存储
- **认证与安全**：密码中间件、API Key 验证、请求级日志追踪
- **数据库连接池**：基于 SQLAlchemy AsyncEngine 的 PostgreSQL 异步连接管理

### 核心组件一览

| 组件 | 文件路径 | 职责 |
|------|---------|------|
| SyncServer | `server/server.py` | 业务逻辑核心，协调所有 Manager |
| FastAPI App | `rest_api/app.py` | 应用入口、中间件注册、异常处理 |
| StreamingServerInterface | `rest_api/interface.py` | 流式响应的 chunk 处理与缓冲 |
| StreamingResponseWithStatusCode | `rest_api/streaming_response.py` | 支持 SSE + 动态状态码的流式响应 |
| WebSocketServer | `ws_api/server.py` | WebSocket 独立服务 |
| DatabaseRegistry | `server/db.py` | 数据库连接池与会话管理 |

---

## 2. API 架构

### 2.1 整体架构图

```mermaid
graph TB
    subgraph Client["客户端"]
        SDK[Letta SDK]
        CLI[CLI 工具]
        ADE[ADE 控制台]
        WS_CLIENT[WebSocket 客户端]
    end

    subgraph Server["Letta Server"]
        subgraph REST["REST API 层 (FastAPI)"]
            MW["中间件栈<br/>CORS → Password → Logging → RequestId"]
            ROUTER["路由层 (v1 Routers)"]
            DEP["依赖注入<br/>get_letta_server / get_headers"]
        end

        subgraph WS["WebSocket API 层"]
            WS_SERVER["WebSocketServer"]
            WS_PROTO["协议层 (protocol.py)"]
        end

        subgraph Core["核心服务层"]
            SYNC["SyncServer"]
            IFACE["StreamingServerInterface"]
        end

        subgraph Managers["Manager 层"]
            AM[AgentManager]
            TM[ToolManager]
            SM[SourceManager]
            MM[MessageManager]
            BM[BlockManager]
            PM[PassageManager]
            RM[RunManager]
            MCP[MCPManager]
            OM[其他 Manager...]
        end

        subgraph Data["数据层"]
            DB["PostgreSQL<br/>(SQLAlchemy AsyncEngine)"]
            REDIS["Redis<br/>(Run 状态追踪)"]
            TPUF["Turbopuffer / Pinecone<br/>(向量存储)"]
        end
    end

    SDK -->|HTTP/REST| MW
    CLI -->|HTTP/REST| MW
    ADE -->|HTTP/REST| MW
    WS_CLIENT -->|WebSocket| WS_SERVER

    MW --> ROUTER
    ROUTER --> DEP
    DEP --> SYNC
    WS_SERVER --> WS_PROTO
    WS_PROTO --> SYNC

    SYNC --> IFACE
    SYNC --> AM
    SYNC --> TM
    SYNC --> SM
    SYNC --> MM
    SYNC --> BM
    SYNC --> PM
    SYNC --> RM
    SYNC --> MCP

    AM --> DB
    MM --> DB
    TM --> DB
    PM --> TPUF
    RM --> REDIS
```

### 2.2 REST API 路由结构

```mermaid
graph LR
    subgraph API["/v1 (API_PREFIX)"]
        AGENTS["/agents"]
        MESSAGES["/messages"]
        TOOLS["/tools"]
        SOURCES["/sources"]
        AUTH["/auth"]
        OTHER["其他路由<br/>/blocks, /providers,<br/>/runs, /jobs, /identities..."]
    end

    subgraph AgentRoutes["/agents 路由组"]
        A_LIST["GET / → list_agents"]
        A_CREATE["POST / → create_agent"]
        A_GET["GET /{id} → retrieve_agent"]
        A_PATCH["PATCH /{id} → modify_agent"]
        A_DELETE["DELETE /{id} → delete_agent"]
        A_MSG["POST /{id}/messages → send_message"]
        A_MSG_STREAM["POST /{id}/messages/stream → send_message_streaming"]
        A_MSG_ASYNC["POST /{id}/messages/async → send_message_async"]
        A_MSG_CANCEL["POST /{id}/messages/cancel → cancel_message"]
        A_MEMORY["GET /{id}/core-memory → retrieve_memory"]
        A_ARCHIVAL["GET /{id}/archival-memory → list_passages"]
        A_TOOLS["GET /{id}/tools → list_tools"]
        A_EXPORT["GET /{id}/export → export_agent"]
        A_IMPORT["POST /import → import_agent"]
    end

    subgraph MessageRoutes["/messages 路由组"]
        M_LIST["GET / → list_all_messages"]
        M_SEARCH["POST /search → search_messages"]
        M_BATCH["POST /batches → create_batch"]
        M_GET["GET /{id} → retrieve_message"]
    end

    subgraph ToolRoutes["/tools 路由组"]
        T_LIST["GET / → list_tools"]
        T_CREATE["POST / → create_tool"]
        T_GET["GET /{id} → retrieve_tool"]
        T_RUN["POST /run → run_tool_from_source"]
        T_MCP["/mcp/servers → MCP 管理"]
        T_GEN["POST /generate-tool → AI 生成工具"]
    end

    subgraph SourceRoutes["/sources 路由组"]
        S_LIST["GET / → list_sources"]
        S_CREATE["POST / → create_source"]
        S_UPLOAD["POST /{id}/upload → upload_file"]
        S_FILES["GET /{id}/files → list_files"]
    end

    AGENTS --> AgentRoutes
    MESSAGES --> MessageRoutes
    TOOLS --> ToolRoutes
    SOURCES --> SourceRoutes
```

---

## 3. 请求处理流程

### 3.1 Agent 消息请求完整处理流程

以 `POST /v1/agents/{agent_id}/messages` 为例，展示从 HTTP 请求到响应的完整流程：

```mermaid
sequenceDiagram
    participant Client as 客户端
    participant CORS as CORSMiddleware
    participant PWD as CheckPasswordMiddleware
    participant LOG as LoggingMiddleware
    participant RID as RequestIdMiddleware
    participant Router as FastAPI Router
    participant Dep as 依赖注入
    participant Server as SyncServer
    participant UM as UserManager
    participant AM as AgentManager
    participant AL as AgentLoop
    participant LLM as LLM Provider
    participant MM as MessageManager
    participant DB as PostgreSQL

    Client->>CORS: POST /v1/agents/{id}/messages<br/>Body: LettaStreamingRequest
    CORS->>PWD: 传递请求
    PWD->>LOG: 验证密码/Token
    LOG->>RID: 提取日志上下文<br/>(actor_id, org_id, agent_id)
    RID->>Router: 设置 request_id contextvar

    Router->>Dep: 路由匹配 → send_message()
    Dep->>Server: get_letta_server() 注入 SyncServer
    Dep->>Dep: get_headers() 解析 HeaderParams

    Note over Dep: 判断 streaming 模式

    alt streaming=false (同步模式)
        Dep->>UM: get_actor_or_default_async()
        UM->>DB: 查询用户
        DB-->>UM: User 对象
        UM-->>Dep: actor

        Dep->>AM: get_agent_by_id_async()
        AM->>DB: 加载 AgentState + 关联数据
        DB-->>AM: AgentState
        AM-->>Dep: agent

        Dep->>AL: AgentLoop.load(agent_state, actor)
        Dep->>AL: agent_loop.step(messages, max_steps, ...)

        loop Agent Step 循环
            AL->>LLM: 发送 LLM 请求
            LLM-->>AL: 返回响应 (tool_calls / content)
            AL->>AM: 执行工具调用
            AM-->>AL: 工具返回结果
            AL->>MM: 持久化消息
            MM->>DB: INSERT messages
        end

        AL-->>Dep: LettaResponse
        Dep-->>Client: 200 OK JSON (LettaResponse)

    else streaming=true (SSE 流式模式)
        Dep->>Server: StreamingService.create_agent_stream()
        Server->>AM: 获取 AgentState
        Server->>AL: 创建 AgentLoop
        Server-->>Client: 200 OK SSE Stream

        loop Agent Step 循环
            AL->>LLM: 流式 LLM 请求
            LLM-->>AL: ChatCompletionChunk
            AL->>AL: StreamingServerInterface.process_chunk()
            AL-->>Client: SSE: data: {LettaMessage}
        end

        AL-->>Client: SSE: data: {MessageStreamStatus}
    end
```

### 3.2 异步消息处理流程

```mermaid
sequenceDiagram
    participant Client as 客户端
    participant API as send_message_async()
    participant RM as RunManager
    participant DB as PostgreSQL
    participant Redis as Redis
    participant Task as asyncio.Task<br/>(后台任务)
    participant AL as AgentLoop

    Client->>API: POST /v1/agents/{id}/messages/async
    API->>RM: create_run(background=True)
    RM->>DB: INSERT run (status=created)
    DB-->>RM: Run 对象
    RM-->>API: run

    API->>Redis: SET run_id for agent
    API->>Task: safe_create_shielded_task(<br/>_process_message_background())

    API-->>Client: 200 OK {Run} (立即返回)

    Note over Task: 后台异步执行
    Task->>AL: AgentLoop.step(messages)
    loop Agent Step
        Task->>Task: LLM 调用 + 工具执行
    end
    Task->>RM: update_run_by_id_async(<br/>status=completed/failed)
    RM->>DB: UPDATE run
```

---

## 4. 流式响应机制

### 4.1 SSE 流式推送架构

```mermaid
sequenceDiagram
    participant Client as 客户端
    participant API as send_message()
    participant SS as StreamingService
    participant IFACE as StreamingServerInterface
    participant AL as AgentLoop
    participant LLM as LLM Provider
    participant SR as StreamingResponseWithStatusCode

    Client->>API: POST messages (streaming=true)
    API->>SS: create_agent_stream()
    SS->>IFACE: 创建 StreamingServerInterface 实例
    SS->>AL: 创建 AgentLoop (注入 interface)

    SS->>SR: 创建 StreamingResponseWithStatusCode<br/>content=stream_generator()
    SR-->>Client: 200 OK Content-Type: text/event-stream

    Note over IFACE,SR: 流式数据推送循环

    loop 每个 Agent Step
        AL->>LLM: 流式 LLM 请求
        LLM-->>AL: ChatCompletionChunk (逐 chunk)

        AL->>IFACE: process_chunk(chunk, message_id, date)
        IFACE->>IFACE: _process_chunk_to_letta_style()

        alt 内容类型为 reasoning
            IFACE->>IFACE: 创建 ReasoningMessage
        else 内容类型为 tool_call
            IFACE->>IFACE: 解析 inner_thoughts + arguments
            IFACE->>IFACE: 创建 ToolCallMessage
        else 内容类型为 assistant (send_message)
            IFACE->>IFACE: 提取 message 字段
            IFACE->>IFACE: 创建 AssistantMessage
        end

        IFACE->>IFACE: _push_to_buffer(processed_chunk)
        IFACE->>IFACE: _event.set() 通知生成器

        IFACE-->>SR: generator yield chunk
        SR-->>Client: SSE: data: {LettaMessage JSON}\n\n
    end

    AL->>IFACE: step_yield() / step_complete()
    IFACE->>IFACE: _active = False
    IFACE->>IFACE: _event.set()

    IFACE-->>SR: generator 结束
    SR-->>Client: SSE: [流结束]
```

### 4.2 StreamingServerInterface 内部状态机

```mermaid
stateDiagram-v2
    [*] --> Inactive: 初始化
    Inactive --> Active: stream_start()
    Active --> Active: process_chunk()<br/>→ _push_to_buffer()

    state Active {
        [*] --> AwaitingChunk
        AwaitingChunk --> ProcessingChunk: 收到 ChatCompletionChunk
        ProcessingChunk --> ReasoningPath: delta.content != null
        ProcessingChunk --> ToolCallPath: delta.tool_calls != null
        ProcessingChunk --> FinishPath: finish_reason != null

        ReasoningPath --> PushBuffer: 创建 ReasoningMessage
        ToolCallPath --> ParseInnerThoughts: inner_thoughts_in_kwargs=true
        ToolCallPath --> ParseAssistantMsg: use_assistant_message + send_message
        ToolCallPath --> DirectToolCall: 默认 ToolCallMessage

        ParseInnerThoughts --> PushBuffer: ReasoningMessage + 缓冲 args
        ParseAssistantMsg --> PushBuffer: AssistantMessage (提取 message 字段)
        DirectToolCall --> PushBuffer: ToolCallMessage
        PushBuffer --> AwaitingChunk: _event.set()
        FinishPath --> AwaitingChunk: return None
    }

    Active --> Active: step_complete()<br/>(multi_step=true 继续)
    Active --> Inactive: step_yield()<br/>(multi_step 结束)
    Active --> Inactive: stream_end()
    Inactive --> [*]
```

### 4.3 Keepalive 与取消机制

```mermaid
sequenceDiagram
    participant Client as 客户端
    participant Keepalive as add_keepalive_to_stream()
    participant Cancel as cancellation_aware_stream_wrapper()
    participant Stream as 原始 SSE 流
    participant RM as RunManager
    participant DB as PostgreSQL

    Note over Keepalive,Stream: Keepalive 包装层
    Stream->>Keepalive: 原始数据 chunk
    Keepalive->>Client: 转发原始数据

    Note over Keepalive: 30s 无数据时
    Keepalive->>Client: SSE: data: {LettaPing}<br/>(包含 seq_id, run_id)

    Note over Cancel,Stream: 取消感知包装层
    loop 每 0.5s 检查
        Cancel->>RM: get_run_by_id(run_id)
        RM->>DB: SELECT run
        DB-->>RM: Run 对象
        RM-->>Cancel: run.status

        alt run.status == cancelled
            Cancel->>Client: SSE: data: {"stop_reason": "cancelled"}
            Cancel->>Stream: athrow(RunCancelledException)
            Stream-->>Cancel: 优雅关闭
        else 正常运行
            Cancel->>Client: 继续转发 chunk
        end
    end
```

---

## 5. WebSocket 通信协议

### 5.1 WebSocket 通信流程

```mermaid
sequenceDiagram
    participant Client as WebSocket 客户端
    participant WS as WebSocketServer
    participant IFACE as SyncWebSocketInterface
    participant Server as SyncServer

    Client->>WS: WebSocket 连接
    WS->>IFACE: register_client(websocket)
    IFACE-->>WS: 注册成功

    Note over Client,WS: 命令消息
    Client->>WS: {"type": "command",<br/>"command": "create_agent",<br/>"config": {...}}
    WS->>Server: create_agent(config)
    Server-->>WS: Agent 创建结果
    WS-->>Client: {"type": "command_response",<br/>"status": "OK: Agent initialized"}

    Note over Client,WS: 用户消息
    Client->>WS: {"type": "user_message",<br/>"message": "Hello",<br/>"agent_id": "agent-xxx"}
    WS->>Client: {"type": "agent_response_start"}
    WS->>Server: user_message(agent_id, message)

    Note over Server: Agent 处理中...
    Server->>IFACE: internal_monologue(msg)
    IFACE->>Client: {"type": "agent_response",<br/>"message_type": "internal_monologue",<br/>"message": "..."}

    Server->>IFACE: assistant_message(msg)
    IFACE->>Client: {"type": "agent_response",<br/>"message_type": "assistant_message",<br/>"message": "..."}

    Server->>IFACE: function_message(msg)
    IFACE->>Client: {"type": "agent_response",<br/>"message_type": "function_message",<br/>"message": "..."}

    WS->>Client: {"type": "agent_response_end"}

    Note over Client,WS: 连接关闭
    Client->>WS: 连接关闭
    WS->>IFACE: unregister_client(websocket)
```

### 5.2 WebSocket 消息协议

```mermaid
graph TB
    subgraph ClientToServer["客户端 → 服务器"]
        CMD["command 消息<br/>{type: 'command',<br/>command: 'create_agent',<br/>config: {...}}"]
        USRMSG["user_message 消息<br/>{type: 'user_message',<br/>message: '...',<br/>agent_id: '...'}"]
    end

    subgraph ServerToClient["服务器 → 客户端"]
        CMDRESP["command_response<br/>{type: 'command_response',<br/>status: 'OK: ...'}"]
        RESPSTART["agent_response_start<br/>{type: 'agent_response_start'}"]
        RESPEND["agent_response_end<br/>{type: 'agent_response_end'}"]
        AGENTRESP["agent_response<br/>{type: 'agent_response',<br/>message_type: 'internal_monologue'<br/>| 'assistant_message'<br/>| 'function_message',<br/>message: '...'}"]
        RESPERR["agent_response_error<br/>{type: 'agent_response_error',<br/>message: '...'}"]
        SRVERR["server_error<br/>{type: 'server_error',<br/>message: '...'}"]
    end

    CMD --> CMDRESP
    USRMSG --> RESPSTART
    RESPSTART --> AGENTRESP
    AGENTRESP --> RESPEND
    USRMSG -.-> RESPERR
    CMD -.-> SRVERR
```

---

## 6. 认证与中间件

### 6.1 中间件处理链

```mermaid
flowchart TD
    REQ["HTTP 请求"] --> CORS["CORSMiddleware<br/>跨域资源共享<br/>allow_origins=settings.cors_origins"]

    CORS --> SECURE{secure 模式?}
    SECURE -->|是| PWD["CheckPasswordMiddleware<br/>验证 X-BARE-PASSWORD /<br/>Authorization Bearer"]
    SECURE -->|否| LOG
    PWD -->|验证通过| LOG
    PWD -->|验证失败| REJECT401["401 Unauthorized"]

    LOG["LoggingMiddleware<br/>1. 清除日志上下文<br/>2. 提取 actor_id, org_id, project_id<br/>3. 从 URL 解析 primitive IDs<br/>4. 设置日志上下文<br/>5. 异常时结构化日志"]

    LOG --> RID["RequestIdMiddleware<br/>(纯 ASGI 中间件)<br/>1. 提取 x-api-request-log-id<br/>2. 设置 request_id_var contextvar<br/>3. 存储 request.state.request_id"]

    RID --> ROUTER["FastAPI 路由处理"]

    ROUTER --> DEP["依赖注入层<br/>get_letta_server() → SyncServer<br/>get_headers() → HeaderParams"]

    DEP --> HANDLER["业务处理函数"]

    HANDLER --> RESP["HTTP 响应"]

    subgraph ExceptionHandling["异常处理链"]
        E400["400: LettaInvalidArgumentError<br/>LettaToolCreateError"]
        E404["404: NoResultFound<br/>LettaAgentNotFoundError"]
        E409["409: ConcurrentUpdateError<br/>ConversationBusyError<br/>DatabaseDeadlockError"]
        E429["429: LLMRateLimitError"]
        E500["500: AgentExportProcessingError"]
        E503["503: OperationalError<br/>DatabaseTimeoutError"]
    end

    HANDLER -.-> ExceptionHandling
    ExceptionHandling -.-> RESP
```

### 6.2 认证流程详解

```mermaid
flowchart LR
    REQ["请求"] --> CHECK_SECURE{LETTA_SERVER_SECURE<br/>或 --secure 参数?}

    CHECK_SECURE -->|否| PASS["直接通过"]
    CHECK_SECURE -->|是| EXTRACT["提取认证信息"]

    EXTRACT --> CHECK_PATH{是否健康检查路径?<br/>/v1/health, /v1/ready}

    CHECK_PATH -->|是| PASS
    CHECK_PATH -->|否| CHECK_AUTH{验证方式}

    CHECK_AUTH -->|X-BARE-PASSWORD| VERIFY_PWD["比对 password"]
    CHECK_AUTH -->|Authorization: Bearer| VERIFY_TOKEN["比对 password /<br/>api_key_to_user()"]

    VERIFY_PWD -->|匹配| PASS
    VERIFY_TOKEN -->|匹配| PASS
    VERIFY_PWD -->|不匹配| REJECT["401 Unauthorized"]
    VERIFY_TOKEN -->|不匹配| REJECT

    PASS --> NEXT["进入下一中间件"]
```

---

## 7. 关键设计决策分析

### 7.1 单例 SyncServer + 依赖注入模式

**决策**：全局创建一个 `SyncServer` 实例，通过 FastAPI 的 `Depends(get_letta_server)` 注入到每个路由处理函数。

**理由**：
- SyncServer 持有所有 Manager 实例（AgentManager、ToolManager 等），Manager 本身是无状态的，共享实例可避免重复初始化
- 启动时自动同步 Provider 模型列表、创建默认用户/组织，确保系统就绪
- 依赖注入使测试时可以替换为 Mock 对象

**权衡**：
- 全局单例在多 Worker 模式下每个 Worker 各持一份，需要通过数据库保证一致性
- SyncServer 构造函数较重（初始化 20+ Manager），启动时间较长

### 7.2 三种消息处理模式

**决策**：提供同步（`/messages`）、流式（`/messages` + `streaming=true`）、异步（`/messages/async`）三种模式。

| 模式 | 适用场景 | 响应方式 | 超时风险 |
|------|---------|---------|---------|
| 同步 | 短对话、调试 | 完整 JSON | 高（阻塞等待） |
| 流式 SSE | 实时交互、长对话 | 逐 chunk 推送 | 低（keepalive 保活） |
| 异步 | 批量处理、后台任务 | 立即返回 Run ID | 无（后台执行） |

**关键实现**：
- 流式模式通过 `StreamingServerInterface` 的 `_chunks` deque + `asyncio.Event` 实现生产者-消费者模式
- `StreamingResponseWithStatusCode` 扩展了 Starlette 的 `StreamingResponse`，支持动态状态码和 `asyncio.shield` 保护
- 异步模式使用 `safe_create_shielded_task` 创建受保护的后台任务，防止客户端断连导致任务取消

### 7.3 流式响应的 chunk 解析策略

**决策**：`StreamingServerInterface` 实现了复杂的 chunk 解析状态机，支持多种 LLM 输出格式。

**核心处理逻辑**：
1. **inner_thoughts_in_kwargs**：当 LLM 将内部思考放在工具调用的参数中时，通过 `JSONInnerThoughtsExtractor` 增量解析 JSON，将 inner_thoughts 提取为 `ReasoningMessage`，其余参数作为 `ToolCallMessage`
2. **use_assistant_message**：将 `send_message` 工具调用转换为 `AssistantMessage`，对前端更友好
3. **OptimisticJSONParser**：乐观解析不完整的 JSON，提前提取关键字段（如 message 内容），减少流式延迟

**权衡**：
- 解析逻辑复杂，存在多种分支组合，维护成本高
- 但这是实现低延迟流式体验的必要代价

### 7.4 中间件分层设计

**决策**：采用四层中间件，从外到内依次为 CORS → Password → Logging → RequestId。

**设计考量**：
- **CORS 最外层**：确保预检请求（OPTIONS）不被其他中间件拦截
- **Password 次外层**：在日志记录之前拦截未授权请求，减少无效日志
- **Logging 中间层**：使用 `BaseHTTPMiddleware`，通过 `update_log_context()` 设置结构化日志上下文
- **RequestId 最内层**：使用纯 ASGI 中间件而非 `BaseHTTPMiddleware`，因为后者在流式响应中无法正确传播 `contextvars`

**关键细节**：`RequestIdMiddleware` 同时将 request_id 存入 `contextvars` 和 `request.state`，因为流式响应的生成器运行在不同的上下文中，`contextvars` 可能无法传播。

### 7.5 数据库连接池管理

**决策**：使用 SQLAlchemy AsyncEngine + asyncpg，模块级创建单例引擎。

**关键配置**：
- `pool_size` / `max_overflow`：控制连接池大小
- `statement_cache_size=0`：禁用 asyncpg 的预处理语句缓存，避免多 Worker 场景下的语句冲突
- `pool_pre_ping=True`：每次从池中取连接时验证连接有效性
- `DatabaseRegistry.async_session()` 实现了自动重试（3次）和 `CancelledError` 的特殊处理（显式 rollback 防止连接泄漏）

### 7.6 异常处理策略

**决策**：在 FastAPI App 层注册大量细粒度的异常处理器，将领域异常映射为标准 HTTP 状态码。

**分类**：

| HTTP 状态码 | 对应异常 | 语义 |
|------------|---------|------|
| 400 | `LettaInvalidArgumentError`, `ValueError` | 请求参数错误 |
| 404 | `NoResultFound`, `LettaAgentNotFoundError` | 资源不存在 |
| 409 | `ConcurrentUpdateError`, `ConversationBusyError`, `DatabaseDeadlockError` | 并发冲突 |
| 429 | `LLMRateLimitError` | 速率限制 |
| 503 | `OperationalError`, `DatabaseTimeoutError` | 服务暂时不可用 |
| 504 | `LLMTimeoutError` | LLM 请求超时 |

**特殊处理**：
- 数据库死锁（`DatabaseDeadlockError`）返回 409 + `Retry-After: 1` 头，提示客户端重试
- 客户端断连（`anyio.BrokenResourceError`）返回 499（非标准但广泛使用的状态码）
- `RequestValidationError` 对路径参数中的 UUID 格式错误提供友好的错误消息和示例 ID

### 7.7 WebSocket API 的定位

**决策**：WebSocket API 作为独立服务运行，与 REST API 分离。

**当前状态**：WebSocket API 功能较简单，仅支持 `create_agent` 和 `user_message` 两种操作，使用 `SyncWebSocketInterface` 进行同步式消息推送。

**与 REST API 的对比**：

| 维度 | REST API | WebSocket API |
|------|---------|--------------|
| 框架 | FastAPI + Uvicorn/Granian | websockets 库 |
| 端口 | 默认 8283 | 默认 8284 |
| 消息格式 | JSON Request/Response | JSON 文本帧 |
| 流式支持 | SSE (Server-Sent Events) | 原生双向流 |
| 认证 | Header-based | 无 |
| 功能完整度 | 完整 | 基础 |

### 7.8 运行时监控与可观测性

**决策**：集成多层可观测性工具。

- **OpenTelemetry**：分布式追踪（`setup_tracing`）+ 指标收集（`setup_metrics`）
- **Sentry**：异常上报（`sentry_sdk.capture_exception`）
- **Datadog**：APM 追踪 + LLMObs + Profiling（可选）
- **Event Loop Watchdog**：监控事件循环阻塞（15s 阈值）
- **DB Pool Monitoring**：SQLAlchemy 连接池状态监控
- **Readiness State**：通过 `readiness_state` 管理启动/关闭状态，支持优雅启停

### 7.9 版本兼容性策略

**决策**：通过 Header 检测 SDK 版本，动态调整响应格式。

- `is_1_0_sdk_version(headers)` 检测客户端 SDK 版本
- 1.0+ SDK：`include_relationships` 默认为空（不加载关联数据），`attach_tool` 返回 `None`
- 旧 SDK：默认加载所有关联数据，`attach_tool` 返回完整 `AgentState`
- 同时维护 `/v1/` 和 `/latest/` 前缀，`/latest/` 不出现在 OpenAPI 文档中
