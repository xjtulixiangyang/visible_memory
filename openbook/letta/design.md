# Letta 整体架构设计文档 (Design)

> 版本: 0.16.8 | 生成日期: 2026-06-07

## 1. 架构总览

Letta 采用**分层架构**设计，从上到下分为 API 层、服务层、Agent 层、基础设施层四个层次，各层之间通过明确的接口解耦。

```mermaid
graph TB
    subgraph "客户端层"
        CLI["CLI 客户端"]
        SDK_P["Python SDK"]
        SDK_TS["TypeScript SDK"]
        WS_CLIENT["WebSocket 客户端"]
    end

    subgraph "API 层"
        REST["REST API<br/>(FastAPI)"]
        WS["WebSocket API"]
        MW["中间件链<br/>CORS → Auth → Logging → RequestID"]
    end

    subgraph "服务层 (SyncServer)"
        AM["AgentManager"]
        MM["MessageManager"]
        BM["BlockManager"]
        PM["PassageManager"]
        TM["ToolExecutionManager"]
        FM["FileManager"]
        SM["Summarizer"]
        JM["JobManager"]
        GM["GroupManager"]
    end

    subgraph "Agent 层"
        AL["AgentLoop<br/>(工厂)"]
        LA["LettaAgent V2/V3"]
        SA["SleeptimeMultiAgent"]
        VA["VoiceAgent"]
        EA["EphemeralAgent"]
    end

    subgraph "LLM 抽象层"
        LCF["LLMClient<br/>(工厂)"]
        OAI["OpenAIClient"]
        ANT["AnthropicClient"]
        GGL["GoogleAIClient"]
        BDK["BedrockClient"]
        OTH["其他 Provider..."]
    end

    subgraph "基础设施层"
        DB[("PostgreSQL<br/>+ pgvector")]
        TP[("Turbopuffer<br/>向量存储")]
        REDIS[("Redis<br/>缓存/流")]
        GIT["Git (GCS)<br/>Block 版本化"]
        OTEL["OpenTelemetry<br/>追踪/指标"]
        SANDBOX["Docker/E2B<br/>沙箱"]
    end

    CLI --> REST
    SDK_P --> REST
    SDK_TS --> REST
    WS_CLIENT --> WS

    REST --> MW --> AM & MM & BM & PM & TM & FM & SM & JM & GM
    WS --> AM

    AM --> AL
    AL --> LA & SA & VA & EA

    LA & SA --> LCF
    LCF --> OAI & ANT & GGL & BDK & OTH

    AM & MM & BM & PM --> DB
    PM --> TP
    TM --> SANDBOX
    BM --> GIT
    REST --> REDIS

    LA & SA & OAI & ANT --> OTEL
```

## 2. 核心数据流

### 2.1 消息处理主流程

用户发送消息到 Agent 的完整数据流：

```mermaid
sequenceDiagram
    participant C as 客户端
    participant API as REST API
    participant MW as 中间件
    participant SS as SyncServer
    participant AM as AgentManager
    participant AL as AgentLoop
    participant Agent as LettaAgent
    participant LLM as LLMClient
    participant TM as ToolExecutionManager
    participant DB as PostgreSQL

    C->>API: POST /v1/agents/{id}/messages
    API->>MW: 认证 + 日志 + RequestID
    MW->>SS: send_message()
    SS->>AM: 获取 AgentState
    AM->>DB: 查询 Agent + Blocks + Tools
    DB-->>AM: AgentState
    AM-->>SS: AgentState

    SS->>AL: AgentLoop.load(agent_state)
    AL-->>SS: LettaAgent 实例

    loop Agent Step Loop
        SS->>Agent: step() / step_stream()
        Agent->>LLM: 发送 LLM 请求
        LLM-->>Agent: LLM 响应 (含 ToolCall)

        alt 有工具调用
            Agent->>TM: 执行工具
            TM-->>Agent: ToolExecutionResult
            Agent->>DB: 持久化消息
        else 无工具调用
            Agent->>DB: 持久化消息
        end

        alt 上下文窗口溢出
            Agent->>Agent: 触发 Summarizer 压缩
        end
    end

    Agent-->>SS: LettaResponse
    SS-->>API: 响应
    API-->>C: HTTP Response / SSE Stream
```

### 2.2 记忆读写数据流

```mermaid
sequenceDiagram
    participant Agent as LettaAgent
    participant BM as BlockManager
    participant PM as PassageManager
    participant MM as MessageManager
    participant DB as PostgreSQL
    participant TP as Turbopuffer
    participant GIT as Git (GCS)

    Note over Agent: Core Memory 写入
    Agent->>BM: update_block(label, value)
    BM->>DB: UPDATE block SET value=...
    BM->>GIT: git commit (如果 Git 版本化启用)
    BM-->>Agent: 更新后的 Block

    Note over Agent: Archival Memory 写入
    Agent->>PM: insert_passage(text, embedding)
    PM->>DB: INSERT passage
    PM->>TP: upsert vector
    PM-->>Agent: Passage ID

    Note over Agent: Archival Memory 检索
    Agent->>PM: search_passages(query, limit)
    PM->>TP: semantic_search(query_embedding)
    TP-->>PM: 排序后的 Passage IDs
    PM->>DB: SELECT passages WHERE id IN (...)
    PM-->>Agent: Passage 列表

    Note over Agent: Recall Memory 检索
    Agent->>MM: get_messages(agent_id, filters)
    MM->>DB: SELECT messages WHERE agent_id=...
    MM-->>Agent: Message 列表
```

## 3. 核心架构模式

### 3.1 分层架构

```mermaid
graph LR
    subgraph "API 层"
        direction TB
        R["Routers<br/>路由定义"]
        D["Dependencies<br/>依赖注入"]
    end

    subgraph "服务层"
        direction TB
        S["SyncServer<br/>统一入口"]
        M["Managers<br/>业务逻辑"]
    end

    subgraph "Agent 层"
        direction TB
        F["AgentLoop<br/>工厂"]
        A["Agent 实现<br/>执行循环"]
    end

    subgraph "数据层"
        direction TB
        O["ORM Models<br/>数据模型"]
        P["Pydantic Schemas<br/>API 契约"]
    end

    R --> D --> S --> M --> F --> A
    M --> O
    O -.->|映射| P
```

**关键约束**:
- API 层只做参数校验和路由，不含业务逻辑
- 服务层 (Manager) 是业务逻辑的唯一归属
- Agent 层通过 Manager 访问数据，不直接操作 ORM
- ORM Model 与 Pydantic Schema 严格分离，通过 `to_pydantic()` / `to_orm()` 转换

### 3.2 工厂模式

项目中大量使用工厂模式实现解耦：

| 工厂 | 位置 | 用途 |
|------|------|------|
| `AgentLoop.load()` | agents/agent_loop.py | 根据 AgentType 创建对应 Agent 实例 |
| `LLMClient.create()` | llm_api/llm_client.py | 根据 ProviderType 创建对应 LLM 客户端 |
| `create_token_counter()` | services/context_window_calculator/ | 根据模型创建 Token 计数器 |

### 3.3 适配器模式

LLM 层使用适配器将不同 Provider 的请求/响应格式统一：

```mermaid
graph LR
    A[Agent] -->|统一接口| B[LLMClientBase]
    B -->|适配| C[OpenAI 格式]
    B -->|适配| D[Anthropic 格式]
    B -->|适配| E[Google 格式]
    B -->|适配| F[Bedrock 格式]

    C --> G[OpenAI API]
    D --> H[Anthropic API]
    E --> I[Google AI API]
    F --> J[Bedrock API]

    style B fill:#e1f5fe
```

所有 Provider 的响应最终归一化为 `OpenAI ChatCompletionResponse` 格式，作为系统内部的"通用语言"。

### 3.4 观察者/事件模式

流式响应使用事件驱动模式：

```mermaid
graph TB
    LLM[LLM Stream] --> SI[StreamingInterface]
    SI --> |on_chunk| SSE[SSE Response]
    SI --> |on_chunk| WS[WebSocket]
    SI --> |on_chunk| CLI[CLI Output]

    style SI fill:#fff3e0
```

## 4. 核心子系统交互

### 4.1 Agent 执行循环

```mermaid
stateDiagram-v2
    [*] --> Initialize: 加载 AgentState
    Initialize --> PrepareContext: 构建 System Prompt + Memory + Messages

    PrepareContext --> CallLLM: 发送请求
    CallLLM --> ParseResponse: 解析响应

    ParseResponse --> HasToolCall: 包含工具调用
    ParseResponse --> NoToolCall: 纯文本响应

    HasToolCall --> ExecuteTool: 执行工具
    ExecuteTool --> CheckRules: 检查工具规则
    CheckRules --> Continue: 规则允许继续
    CheckRules --> Terminal: 规则要求终止
    Continue --> ContextOverflow?

    NoToolCall --> Terminal: 发送消息工具 = 终止

    ContextOverflow?: --> Compress: 上下文溢出
    ContextOverflow?: --> PrepareContext: 上下文正常
    Compress --> PrepareContext: 压缩后重新构建

    Terminal --> PersistMessages: 持久化消息
    PersistMessages --> [*]
```

### 4.2 多 Agent 协作模式

```mermaid
graph TB
    subgraph "Sleeptime 模式"
        direction LR
        MA1["主 Agent<br/>(在线处理)"]
        SA1["睡眠 Agent<br/>(后台整理记忆)"]
        MA1 <-.->|共享 Block| SA1
    end

    subgraph "Supervisor 模式"
        direction LR
        SUP["监督者 Agent"]
        W1["Worker 1"]
        W2["Worker 2"]
        SUP -->|分配任务| W1
        SUP -->|分配任务| W2
        W1 -->|汇报结果| SUP
        W2 -->|汇报结果| SUP
    end

    subgraph "RoundRobin 模式"
        direction LR
        A1["Agent A"] --> A2["Agent B"]
        A2 --> A3["Agent C"]
        A3 --> A1
    end

    subgraph "Dynamic 模式"
        direction LR
        ORC["协调者"]
        ORC -->|动态选择| P1["Agent Pool"]
    end
```

### 4.3 上下文窗口管理

```mermaid
flowchart TD
    A[构建上下文] --> B{Token 数 > 阈值?}
    B -->|否| C[正常发送 LLM 请求]
    B -->|是| D{压缩模式?}

    D -->|static_buffer| E[滑动窗口截断<br/>保留最近 N 条消息]
    D -->|summarize| F[调用 Summarizer<br/>生成摘要替换旧消息]

    E --> G{Token 数仍超限?}
    F --> G

    G -->|是| H[消息截断<br/>保留 System + 最近消息]
    G -->|否| C

    H --> I{仍超限?}
    I -->|是| J[抛出 ContextWindowExceededError]
    I -->|否| C

    style D fill:#fff3e0
    style J fill:#ffebee
```

## 5. 数据模型关系

```mermaid
erDiagram
    Organization ||--o{ User : "拥有"
    User ||--o{ Agent : "创建"
    Agent ||--o{ Block : "绑定"
    Agent ||--o{ Tool : "绑定"
    Agent ||--o{ Message : "产生"
    Agent ||--o{ Conversation : "参与"
    Agent ||--o{ Source : "关联"
    Agent }o--|| Group : "属于"
    Group ||--o{ Agent : "包含"

    Block ||--o{ BlockHistory : "版本历史"
    Block }o--o{ Tag : "标签"

    Source ||--o{ Passage : "包含"
    Passage }o--o{ Tag : "标签"

    Conversation ||--o{ Message : "包含"

    Tool ||--o{ MCPServer : "来自"

    Agent ||--o{ Step : "执行步骤"
    Step ||--o{ StepMetrics : "指标"

    User ||--o{ Job : "提交"
    Job ||--o{ Message : "关联"
```

## 6. 部署架构

```mermaid
graph TB
    subgraph "客户端"
        CLI_CLIENT["Letta Code CLI"]
        SDK_CLIENT["Python/TS SDK"]
        ADE["ADE Web UI"]
    end

    subgraph "Letta Server"
        API_SERVER["FastAPI Server<br/>:8283"]
        WS_SERVER["WebSocket Server<br/>:8283"]
        SCHEDULER["APScheduler<br/>任务调度"]
    end

    subgraph "数据存储"
        PG[("PostgreSQL<br/>+ pgvector")]
        TP[("Turbopuffer<br/>向量存储")]
        REDIS[("Redis<br/>缓存")]
        GCS[("GCS<br/>Git Block 存储")]
        CH[("ClickHouse<br/>Trace 存储")]
    end

    subgraph "外部服务"
        LLM_PROV["LLM Providers<br/>OpenAI/Anthropic/Google/..."]
        MCP_SRV["MCP Servers<br/>外部工具"]
        SANDBOX["Docker/E2B<br/>沙箱"]
    end

    CLI_CLIENT --> API_SERVER
    SDK_CLIENT --> API_SERVER
    ADE --> API_SERVER
    ADE --> WS_SERVER

    API_SERVER --> PG
    API_SERVER --> TP
    API_SERVER --> REDIS
    API_SERVER --> GCS
    API_SERVER --> CH
    API_SERVER --> LLM_PROV
    API_SERVER --> MCP_SRV
    API_SERVER --> SANDBOX
    SCHEDULER --> API_SERVER
```

## 7. 关键设计决策

| # | 决策 | 选择 | 理由 |
|---|------|------|------|
| 1 | 响应格式归一化 | 统一为 OpenAI ChatCompletionResponse | OpenAI 格式是事实标准，减少下游转换 |
| 2 | ORM/Schema 分离 | SQLAlchemy Model + Pydantic Schema 双层 | ORM 负责持久化，Schema 负责 API 契约，关注点分离 |
| 3 | Agent 工厂模式 | AgentLoop.load() 按 AgentType 创建 | 解耦 Agent 类型和实例化逻辑 |
| 4 | 流式接口抽象 | StreamingInterface 事件驱动 | 统一 SSE/WebSocket/CLI 多种输出通道 |
| 5 | 记忆三层架构 | Core/Recall/Archival | 模拟人类记忆系统，平衡延迟和容量 |
| 6 | 上下文压缩降级链 | 滑动窗口 → 摘要 → 截断 | 优雅降级，优先保证功能可用 |
| 7 | 多 Agent 共享 Block | Sleeptime Agent 通过 Block 传递记忆 | 简单高效，避免引入额外消息通道 |
| 8 | 工具规则声明式 | ToolRule 约束工具调用顺序 | 声明式比命令式更易理解和维护 |
| 9 | MCP 集成 | 双层（注册+运行时发现） | 支持动态工具发现，保持 Schema 一致性 |
| 10 | 软删除 | is_deleted 标记 | 保留审计追踪，支持数据恢复 |

## 8. 模块依赖关系

```mermaid
graph LR
    Server[Server/API] --> Services[Services]
    Server --> Agents[Agents]
    Agents --> Services
    Agents --> LLM[LLM API]
    Agents --> Functions[Functions/Tools]
    Services --> ORM[ORM]
    Services --> Schemas[Schemas]
    ORM --> Schemas
    LLM --> Schemas
    Functions --> Schemas
    Agents --> Groups[Groups]
    Groups --> Agents

    style Server fill:#e3f2fd
    style Agents fill:#e8f5e9
    style Services fill:#fff3e0
    style LLM fill:#fce4ec
    style ORM fill:#f3e5f5
```

## 9. 详细设计文档索引

| 模块 | 文档 | 核心内容 |
|------|------|----------|
| Agent 系统 | [agent-system.md](./agent-system.md) | 类继承体系、生命周期、执行循环、多 Agent 协作 |
| LLM 抽象层 | [llm-abstraction.md](./llm-abstraction.md) | Provider 路由、请求流程、流式响应、适配器模式 |
| Memory 记忆系统 | [memory-system.md](./memory-system.md) | 三层架构、Block 管理、上下文压缩、记忆检索 |
| Server/API 层 | [server-api.md](./server-api.md) | REST/WebSocket、流式推送、认证中间件、异常处理 |
| ORM 持久化层 | [orm-persistence.md](./orm-persistence.md) | ER 关系、Schema 映射、多租户、软删除 |
| Tool/Function 系统 | [tool-function.md](./tool-function.md) | 工具注册/执行、MCP 集成、规则系统、沙箱 |
