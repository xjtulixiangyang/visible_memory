# OpenSquilla 架构设计文档 (Design Document)

> 版本: 0.3.1 | 生成日期: 2026-06-09

## 1. 架构总览

OpenSquilla 采用 **微内核 + 分层解耦 + 事件驱动** 的架构风格。核心是一个统一的 `TurnRunner` 对话循环，所有入口（Web UI、CLI、聊天通道）共享同一执行路径。

### 1.1 分层架构图

```mermaid
graph TB
    subgraph 入口层
        WEB[Web UI<br/>/control/]
        CLI[CLI<br/>chat/agent]
        CH[Channels<br/>Slack/Telegram/Discord/飞书/钉钉/企微/QQ/Matrix]
        CRON[Cron<br/>定时调度]
    end

    subgraph 网关层
        GW[Gateway<br/>Starlette ASGI :18791]
        WS[WebSocket<br/>协议 v3]
        RPC[RPC Registry<br/>100+ 方法]
        AUTH[Auth<br/>Token/Open/Proxy]
        MW[Middleware<br/>CORS/RateLimit/Security]
    end

    subgraph 调度层
        TR_RUNTIME[TaskRuntime<br/>公平并发调度]
        CH_DISPATCH[Channel Dispatch<br/>消息分发]
        ROUTING[Routing<br/>RouteEnvelope]
    end

    subgraph 引擎层
        TURN[TurnRunner<br/>对话循环核心]
        PIPE[Pipeline<br/>8步编排]
        STEPS[Steps<br/>路由/模型/技能/缓存/...]
        TOOLS[ToolRegistry<br/>工具调度]
        STREAM[Stream Wrappers<br/>流式响应]
    end

    subgraph 路由层
        ROUTER[SquillaRouter<br/>LightGBM + ONNX]
        CTRL[Controller<br/>T0-T3 / P0-P2]
    end

    subgraph 记忆层
        MEM_MGR[MemoryManager<br/>门面]
        STORE[LongTermMemoryStore<br/>SQLite]
        SYNC[SyncManager<br/>6种触发器]
        RETRIEVE[Retriever<br/>混合检索]
        EMBED[Embedding<br/>本地/远程/空]
    end

    subgraph 技能层
        SKILL_LOADER[SkillLoader<br/>六层扫描]
        SKILL_INJECT[SkillInjector<br/>提示注入]
        META[MetaOrchestrator<br/>DAG编排]
        MCP[MCP Client<br/>stdio/sse]
    end

    subgraph 身份层
        IDENTITY[Identity<br/>SOUL/IDENTITY/AGENTS]
        PROMPT[System Prompt<br/>Jinja2 模板]
        BOOTSTRAP[Bootstrap<br/>工作区引导]
    end

    subgraph 基础设施层
        DB[(SQLite<br/>WAL模式)]
        SANDBOX[Sandbox<br/>Standard/Strict/Locked]
        CONFIG[Config<br/>TOML + 环境变量]
        LOG[Logging<br/>structlog]
    end

    WEB & CLI & CH & CRON --> GW
    GW --> WS & RPC
    WS & RPC --> AUTH & MW
    RPC --> TR_RUNTIME & CH_DISPATCH
    TR_RUNTIME & CH_DISPATCH --> ROUTING
    ROUTING --> TURN
    TURN --> PIPE --> STEPS
    TURN --> TOOLS & STREAM
    STEPS --> ROUTER --> CTRL
    STEPS --> SKILL_LOADER --> SKILL_INJECT & META
    STEPS --> MCP
    TURN --> MEM_MGR --> STORE & SYNC & RETRIEVE & EMBED
    TURN --> IDENTITY --> PROMPT & BOOTSTRAP
    STORE --> DB
    TOOLS --> SANDBOX
    TURN --> CONFIG & LOG
```

### 1.2 架构原则

| 原则 | 体现 |
|---|---|
| **统一循环** | 所有入口共享同一 `TurnRunner`，工具调度、重试、决策日志行为一致 |
| **本地优先** | 路由分类和嵌入在设备端完成，用户提示不离开本地 |
| **渐进降级** | 路由不可用→默认模型；嵌入不可用→FTS-only；通道断开→状态机恢复 |
| **协议驱动** | Channel/ManagedChannel/InboundTransport 均为 Protocol，非抽象基类 |
| **事件驱动** | Session 事件通过 EventBridge 广播，支持断线重放 |
| **预算控制** | 上下文预算、技能注入预算、工作区文件预算 |

---

## 2. 核心流程设计

### 2.1 对话轮次完整流程

```mermaid
sequenceDiagram
    participant User as 用户
    participant Entry as 入口(Web/CLI/Channel)
    participant GW as Gateway
    participant Runtime as TaskRuntime
    participant Runner as TurnRunner
    participant Pipe as Pipeline
    participant Router as SquillaRouter
    participant Skills as SkillLoader
    participant Identity as Identity
    participant Memory as MemoryManager
    participant Provider as LLM Provider
    participant Tools as ToolRegistry
    participant Bridge as EventBridge

    User->>Entry: 发送消息
    Entry->>GW: 请求 (HTTP/WS/Channel)
    GW->>GW: 认证 + 权限检查
    GW->>Runtime: enqueue(RouteEnvelope)
    Runtime->>Runtime: 公平获取槽位
    Runtime->>Runner: run(message, session_key)

    Runner->>Identity: assemble_system_prompt()
    Identity-->>Runner: 系统提示 (可缓存基础)

    Runner->>Pipe: execute(TurnContext)
    Pipe->>Pipe: Step 1: inject_platform_hint
    Pipe->>Router: Step 2: apply_squilla_router
    Router-->>Pipe: RoutingDecision (tier, model, thinking_mode)
    Pipe->>Pipe: Step 3: resolve_model
    Pipe->>Skills: Step 4: apply_skills_filter
    Skills-->>Pipe: 注入 available_skills XML
    Pipe->>Pipe: Step 5-8: time/cache/meta/...

    Pipe-->>Runner: TurnContext (完整)

    Runner->>Memory: search(query) → 召回记忆
    Memory-->>Runner: 记忆片段
    Runner->>Runner: 追加易失性上下文

    loop 工具循环 (max N 轮)
        Runner->>Provider: chat(messages, tools, stream=True)
        Provider-->>Runner: 流式响应 (text_delta / tool_call)
        Runner->>Bridge: 推送事件

        alt 有工具调用
            Runner->>Tools: execute(tool_call)
            Tools-->>Runner: tool_result
        else 无工具调用
            Runner->>Runner: 退出循环
        end
    end

    Runner->>Runner: 计算 usage + pricing
    Runner->>Bridge: 推送最终事件
    Runner-->>Runtime: TurnOutcome
    Runtime-->>Entry: 流式/最终响应
    Entry-->>User: 回复
```

### 2.2 模型路由决策流程

```mermaid
flowchart TD
    A[用户消息] --> B{路由器启用?}
    B -- 否 --> Z[默认模型]
    B -- 是 --> C{图片附件?}
    C -- 是 --> D[图片路由层]
    C -- 否 --> E{RouterControlHold?}
    E -- 是 --> F[Hold 指定层]
    E -- 否 --> G[V4Phase3 分类]

    G --> H[特征提取<br/>BGE嵌入 + TF-IDF/SVD]
    H --> I[LightGBM + MLP<br/>双头分类]
    I --> J[概率向量<br/>R0-R3]

    J --> K[Controller 后处理]
    K --> K1[derive_thinking_mode<br/>T0-T3]
    K --> K2[derive_prompt_policy<br/>P0-P2]
    K1 & K2 --> K3[normalize_decisions<br/>禁止矛盾组合]

    K3 --> L[_finalize_decision]
    L --> L1[置信度保护<br/>low confidence → default]
    L --> L2[投诉升级<br/>检测不满词汇]
    L --> L3[KV缓存反降级<br/>30min窗口]
    L --> L4[大上下文地板<br/>25K+ → c2, 80K+ → c3]

    L1 & L2 & L3 & L4 --> M[_apply_controller]
    M --> N{rollout_phase}
    N -- observe --> O[仅记录元数据]
    N -- prompt_only --> P[注入 prompt hint]
    N -- full --> Q[注入 hint + thinking_level]
```

### 2.3 记忆检索流程

```mermaid
flowchart TD
    A[用户查询] --> B[MemoryRetriever.search]
    B --> C[触发 SyncManager.sync<br/>若有脏数据]
    C --> D{sqlite-vec 可用?}

    D -- 是 --> E[并行双路检索]
    E --> E1[向量搜索<br/>KNN + L2距离]
    E --> E2[FTS5 搜索<br/>BM25 + CJK分词]
    E1 & E2 --> F[加权融合<br/>0.7*vector + 0.3*text]

    D -- 否 --> G[纯 FTS5<br/>BM25 排序]

    F --> H[后处理增强]
    G --> H
    H --> H1[时间衰减<br/>半衰期30天]
    H --> H2[MMR 多样性重排<br/>lambda=0.7]
    H --> H3[源权重<br/>sessions ×0.92]

    H1 & H2 & H3 --> I[返回 MemorySearchResult]
```

### 2.4 Meta-Skill DAG 编排流程

```mermaid
flowchart TD
    A[触发词匹配] --> B[MetaOrchestrator.iter_events]
    B --> C[scheduler.run_dag]
    C --> D[拓扑排序 steps]
    D --> E[并发执行无依赖步骤]

    E --> F{步骤类型 kind}
    F -->|agent| G[子 Agent + 工具循环]
    F -->|llm_classify| H[受限 LLM 分类]
    F -->|llm_chat| I[开放 LLM 对话]
    F -->|tool_call| J[直接工具调用]
    F -->|skill_exec| K[CLI 入口点]
    F -->|user_input| L[暂停等待用户]

    G & H & I & J & K --> M{成功?}
    L --> N{用户回复?}
    N -- 否 --> O[MetaPaused<br/>取消兄弟任务]
    N -- 是 --> M

    M -- 是 --> P[解锁下游]
    M -- 否 --> Q{on_failure?}
    Q -- 是 --> R[故障转移]
    Q -- 否 --> S[取消全部]

    P --> T{还有步骤?}
    T -- 是 --> E
    T -- 否 --> U[final_text_mode<br/>auto/raw/step:id]
```

---

## 3. 模块设计

### 3.1 Engine 模块

**职责**: 对话轮次执行的完整生命周期管理

```mermaid
graph TB
    subgraph Engine
        TR[TurnRunner] --> PL[Pipeline]
        PL --> S1[inject_platform_hint]
        PL --> S2[apply_squilla_router]
        PL --> S3[resolve_model]
        PL --> S4[apply_skills_filter]
        PL --> S5[inject_time_prefix]
        PL --> S6[apply_prompt_cache]
        PL --> S7[apply_meta_resolution]
        PL --> S8[reasoning_hint_observer]

        TR --> TC[TurnContext]
        TR --> TL[ToolLoop]
        TR --> SW[StreamWrappers]
        TR --> UC[UsageCalculator]

        TL --> TREG[ToolRegistry]
        TL --> SBX[Sandbox]
    end
```

**关键设计决策**:
- Pipeline 模式：步骤可插拔，新增步骤无需修改 TurnRunner
- 工具循环上限：防止无限工具调用
- 流式优先：TurnRunner 返回 `Stream[TurnEvent]`，支持实时推送

### 3.2 Gateway 模块

**职责**: ASGI 应用入口、RPC 分发、Session 管理

```mermaid
graph TB
    subgraph Gateway
        APP[create_gateway_app] --> MW_STACK[Middleware Stack]
        MW_STACK --> MW1[ErrorHandling]
        MW_STACK --> MW2[CORS]
        MW_STACK --> MW3[RateLimit]
        MW_STACK --> MW4[SecurityHeaders]
        MW_STACK --> MW5[Auth]

        APP --> WS_HANDLER[WebSocket Handler]
        APP --> HTTP_HANDLER[HTTP Routes]

        WS_HANDLER --> PROTO[Protocol Frames]
        PROTO --> RPC_REG[RpcRegistry]
        HTTP_HANDLER --> RPC_REG

        RPC_REG --> SCOPE[Scope Check]
        SCOPE --> HANDLERS[RPC Handlers]
    end
```

**关键设计决策**:
- 中间件管道：每个中间件独立关注点，可插拔
- RPC 注册表：启动后锁定，防止运行时动态添加
- Session epoch：乐观并发控制，防止过期帧

### 3.3 Channels 模块

**职责**: 多平台消息适配

```mermaid
graph TB
    subgraph Channels
        CM[ChannelManager] --> REG[Registry]
        REG --> SA[SlackAdapter]
        REG --> TA[TelegramAdapter]
        REG --> DA[DiscordAdapter]
        REG --> FA[FeishuAdapter]
        REG --> DTA[DingTalkAdapter]
        REG --> WCA[WeComAdapter]
        REG --> QA[QQAdapter]
        REG --> MA[MatrixAdapter]
        REG --> MSA[MSTeamsAdapter]

        CM --> DSP[Dispatch Loop]
        DSP --> SP[StreamPolicy]
        DSP --> AP[AccessPolicy]
        DSP --> AD[ArtifactDelivery]
    end
```

**关键设计决策**:
- Protocol 而非 ABC：适配器无需显式继承
- 能力声明式：`ChannelCapabilityProfile` 运行时查询
- 三种流式策略：适配不同平台能力

### 3.4 SquillaRouter 模块

**职责**: 本地模型路由分类

**关键设计决策**:
- 设备端分类：用户提示不离开本地
- 双头分类：LightGBM 主头 + MLP 辅助头
- 渐进式发布：observe → prompt_only → full
- 四重后处理：置信度保护 + 投诉升级 + 反降级 + 大上下文地板

### 3.5 Memory 模块

**职责**: 持久化记忆存储与检索

**关键设计决策**:
- 混合检索：向量(0.7) + FTS5(0.3) 加权融合
- 降级链：本地嵌入 → 远程嵌入 → FTS-only
- 事件驱动同步：6种触发器，避免全量扫描
- Dream 巩固：证据门控，防止低质量记忆提升

---

## 4. 并发与调度设计

### 4.1 TaskRuntime 公平调度

```mermaid
flowchart TD
    A[任务入队] --> B{全局信号量<br/>_max_concurrency}
    B -- 有槽位 --> C{per-agent 公平队列}
    B -- 无槽位 --> D[等待/拒绝<br/>PendingOverflowPolicy]
    C --> E[Round-Robin<br/>per-agent deque]
    E --> F{subagent 预留槽?}
    F -- 是 --> G[优先分配预留槽]
    F -- 否 --> H[正常分配]
    G & H --> I[acquire per-session 执行锁]
    I --> J[执行 Turn]
    J --> K[释放执行锁 + 全局槽位]
```

### 4.2 Session 锁层级

| 锁类型 | 粒度 | 用途 |
|---|---|---|
| 全局信号量 | 全局 | 限制最大并发 Turn 数 |
| per-agent RR deque | per-agent | 公平调度不同 agent |
| subagent 预留槽 | per-session | 防止子代理饿死 |
| per-session 执行锁 | per-session | 同一 session 串行执行 |
| per-session 写锁 | per-session | 写操作互斥 |

### 4.3 通道调度状态机

```mermaid
stateDiagram-v2
    [*] --> Running: start()
    Running --> Running: 正常处理消息
    Running --> Exhausted: 内层重试5次耗尽
    Exhausted --> Restarting: restart_count < 3
    Restarting --> Running: 30s 后重启
    Exhausted --> Dead: restart_count >= 3
    Dead --> Running: 管理员 restart_channel()
```

---

## 5. 安全设计

### 5.1 认证与授权

```mermaid
flowchart TD
    A[请求到达] --> B{auth mode}
    B -- token --> C[TokenScopeResolver<br/>验证 Token → Principal]
    B -- none --> D[OpenScopeResolver<br/>loopback → CLI_DEFAULT<br/>remote → REMOTE_OPERATOR]
    B -- trusted_proxy --> E[ProxyResolver<br/>X-Forwarded-For + Token]
    C & D & E --> F[Principal<br/>role + scopes]
    F --> G[authorize_call<br/>METHOD_SCOPES 查表]
    G --> H{scope 满足?}
    H -- 是 --> I[执行 handler]
    H -- 否 --> J[403 Forbidden]
```

### 5.2 沙箱分层

| 层级 | 策略 | 说明 |
|---|---|---|
| Standard | 默认 | 允许大部分操作，敏感操作需审批 |
| Strict | 受限 | 限制文件系统和网络访问 |
| Locked | 锁定 | 最小权限，仅允许白名单工具 |

### 5.3 注入防护

- 技能元数据 XML 转义，防止逃逸 `<available_skills>` 边界
- 工具结果 XML 转义，防止 prompt injection
- 工作区文件 `scan_for_injection` 扫描
- SSRF 防护：web fetch 限制目标

---

## 6. 数据流设计

### 6.1 消息处理主数据流

```mermaid
flowchart LR
    subgraph 入站
        A1[Web UI] --> B[Gateway]
        A2[CLI] --> B
        A3[Channel] --> B
        A4[Cron] --> B
    end

    subgraph 调度
        B --> C[RouteEnvelope]
        C --> D[TaskRuntime]
    end

    subgraph 执行
        D --> E[TurnRunner]
        E --> F[Pipeline]
        F --> G[LLM Provider]
        G --> H[Tool Loop]
        H --> G
    end

    subgraph 出站
        E --> I[EventBridge]
        I --> J1[WebSocket 推送]
        I --> J2[Channel 回复]
        I --> J3[Session Stream 记录]
    end
```

### 6.2 事件重放数据流

```mermaid
sequenceDiagram
    participant Client as WS 客户端
    participant Stream as SessionStreamRegistry
    participant Bridge as EventBridge
    participant WS as WsConnection

    Note over Stream: per-session deque, max 500 events

    Bridge->>Stream: record(session_key, event, payload)
    Stream->>Stream: 分配 stream_seq
    Stream->>WS: send_event(event, seq)

    Note over Client: 断线重连
    Client->>Stream: subscribe(since_stream_seq=N)
    Stream->>Stream: replay(seq > N)
    Stream->>Stream: gap 检测 (lossy events)
    Stream-->>Client: 重放事件 + 当前 seq
```

---

## 7. 设计模式汇总

| 模式 | 应用位置 | 说明 |
|---|---|---|
| **Pipeline** | Engine Steps | 步骤可插拔编排 |
| **Strategy** | 路由策略 / 嵌入提供者 / 认证策略 / 流式策略 / 溢出策略 | 运行时切换行为 |
| **Registry** | RPC / Channel / Tool / Agent Task | 统一发现与构建 |
| **Observer/Pub-Sub** | EventBridge / SubscriptionManager | 解耦事件广播 |
| **Middleware Pipeline** | Gateway 中间件栈 | 横切关注点可插拔 |
| **Envelope** | RouteEnvelope | 统一不同来源元数据 |
| **Adapter** | Channel 适配器 | 平台差异适配 |
| **Protocol-Oriented** | Channel / ManagedChannel / InboundTransport | 接口契约 |
| **Facade** | MemoryManager / ServiceContainer | 简化复杂子系统 |
| **State Machine** | Channel 调度 / Session 生命周期 | 状态转换管理 |
| **Circuit Breaker** | FloodStrikeBackoff | 防止雪崩 |
| **Backpressure** | TaskQueueFullError / ChannelInFlightSet | 过载保护 |
| **Fair Scheduling** | TaskRuntime per-agent RR | 公平并发 |
| **Optimistic Concurrency** | Session epoch | 无锁并发控制 |
| **Lazy Load + Double-Check Lock** | 路由模型 / 嵌入模型 | 延迟加载 + 线程安全 |
| **Snapshot Cache** | Skills JSON 快照 | 加速冷启动 |
| **Layered Override** | Skills 六层 / Identity 多来源 | 渐进定制 |
| **DAG Scheduling** | Meta-Skill 步骤图 | 并行编排 |
| **Failover** | Meta-Skill on_failure | 自动故障转移 |
| **Pause/Resume** | Meta-Skill user_input | CAS 原子恢复 |
| **Evidence-Gated** | Dream 记忆巩固 | 质量门控 |
| **Budget Control** | 上下文 / 技能注入 / 工作区文件 | 资源限制 |

---

## 8. 部署架构

```mermaid
graph TB
    subgraph 用户设备
        subgraph OpenSquilla 进程
            UVICORN[Uvicorn<br/>ASGI Server]
            GW[Gateway<br/>:18791]
            ENGINE[Engine<br/>TurnRunner]
            ROUTER[SquillaRouter<br/>ONNX + LightGBM]
            MEM[Memory<br/>SQLite + Embeddings]
        end

        subgraph 数据目录
            CONFIG[config.toml]
            STATE[(SQLite DB)]
            WORKSPACE[Agent Workspace<br/>SOUL.md / MEMORY.md / ...]
        end
    end

    subgraph 外部服务
        LLM[LLM APIs]
        SEARCH[Search APIs]
        CHANNEL[Channel APIs]
        MCP_SVR[MCP Servers]
    end

    UVICORN --> GW
    GW --> ENGINE
    ENGINE --> ROUTER
    ENGINE --> MEM
    MEM --> STATE
    ENGINE --> LLM & SEARCH & CHANNEL & MCP_SVR
    GW --> CONFIG & WORKSPACE
```

---

## 附录：模块分析文档索引

| 文档 | 路径 |
|---|---|
| Engine 模块分析 | [engine.md](engine.md) |
| Gateway 模块分析 | [gateway.md](gateway.md) |
| Channels 模块分析 | [channels.md](channels.md) |
| Router & Memory 模块分析 | [router-memory.md](router-memory.md) |
| Skills, Identity & MCP 模块分析 | [skills-identity-mcp.md](skills-identity-mcp.md) |
| 项目规格说明 | [spec.md](spec.md) |
