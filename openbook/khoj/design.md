# Khoj 整体架构设计 (design.md)

## 1. 架构总览

Khoj 采用 **FastAPI + Django 混合架构**，FastAPI 作为主 Web 框架处理异步 API 请求，Django 作为 ORM 层管理数据持久化和 Admin 后台。两者通过 ASGI 协议在同一进程中运行。

```mermaid
graph TB
    subgraph "进程架构"
        MAIN[main.py<br/>入口点] --> FASTAPI[FastAPI App]
        MAIN --> DJANGO[Django App<br/>ORM + Admin]
        MAIN --> SCHEDULER[APScheduler<br/>定时任务]
    end

    subgraph "FastAPI 应用"
        MIDDLEWARE[中间件链<br/>Session → Auth → CORS → DB Cleanup]
        ROUTERS[路由模块<br/>Chat / Content / Agents / ...]
        MIDDLEWARE --> ROUTERS
    end

    subgraph "Django 应用"
        MODELS[数据模型<br/>KhojUser / Conversation / Entry / ...]
        ADMINS[Admin 后台<br/>Unfold]
        MIGRATIONS[数据库迁移]
    end

    FASTAPI --> MIDDLEWARE
    DJANGO --> MODELS

    ROUTERS --> MODELS
    SCHEDULER --> MODELS

    subgraph "外部依赖"
        PG[(PostgreSQL<br/>+ pgvector)]
        LLM_API[LLM APIs<br/>OpenAI / Anthropic / Google]
        SEARCH_API[搜索 APIs<br/>Serper / Exa / Google]
    end

    MODELS --> PG
    ROUTERS --> LLM_API
    ROUTERS --> SEARCH_API
```

---

## 2. 启动流程

```mermaid
sequenceDiagram
    participant CLI as CLI 参数
    participant MAIN as main.py
    participant DJANGO as Django
    participant CONFIG as configure.py
    participant INIT as initialization
    participant SCHED as APScheduler
    participant UVICORN as Uvicorn

    CLI->>MAIN: run()
    MAIN->>MAIN: set_state(args)
    MAIN->>DJANGO: django.setup()
    MAIN->>DJANGO: migrate --noinput
    MAIN->>DJANGO: collectstatic --noinput
    MAIN->>INIT: initialization()
    Note over INIT: 加载搜索模型、嵌入模型、创建默认 Agent

    MAIN->>SCHED: poll_task_scheduler()
    Note over SCHED: 每 60s 轮询 schedule

    MAIN->>SCHED: BackgroundScheduler.start()
    Note over SCHED: Process Lock 选举 Schedule Leader

    MAIN->>CONFIG: configure_routes(app)
    Note over CONFIG: 注册所有 API 路由

    MAIN->>CONFIG: configure_middleware(app)
    Note over CONFIG: Session → Auth → CORS → DB Cleanup

    MAIN->>UVICORN: uvicorn.run(app)
    Note over UVICORN: 启动 HTTP 服务
```

---

## 3. 请求处理流程

### 3.1 中间件链

```mermaid
flowchart TD
    REQ[HTTP Request] --> S1[SessionMiddleware<br/>会话管理]
    S1 --> S2[NextJsMiddleware<br/>/_next 路径重写]
    S2 --> S3[ServerErrorMiddleware<br/>5xx 错误处理]
    S3 --> S4[AuthenticationMiddleware<br/>用户认证]
    S4 --> S5[AsyncCloseConnectionsMiddleware<br/>数据库连接清理]
    S5 --> S6[SuppressClientDisconnectMiddleware<br/>客户端断开处理]
    S6 --> S7[HTTPSRedirectMiddleware<br/>HTTPS 重定向<br/>可选]

    S7 --> HANDLER[Route Handler]
    HANDLER --> RES[HTTP Response]
```

### 3.2 认证决策流程

```mermaid
flowchart TD
    REQ[请求] --> SESSION{Session 中<br/>有用户?}
    SESSION -->|是| SUB{已订阅?}
    SUB -->|是| PREMIUM[authenticated + premium]
    SUB -->|否| AUTH[authenticated]

    SESSION -->|否| BEARER{Bearer Token<br/>有效?}
    BEARER -->|是| SUB2{已订阅?}
    SUB2 -->|是| PREMIUM
    SUB2 -->|否| AUTH

    BEARER -->|否| WHATSAPP{WhatsApp<br/>Client?}
    WHATSAPP -->|是| SUB3{已订阅?}
    SUB3 -->|是| PREMIUM
    SUB3 -->|否| AUTH

    WHATSAPP -->|否| ANON{匿名模式?}
    ANON -->|是| PREMIUM
    ANON -->|否| UNAUTH[Unauthenticated]
```

---

## 4. 核心模块交互

### 4.1 对话请求完整流程

```mermaid
sequenceDiagram
    participant CLIENT as 客户端
    participant API as api_chat.py
    participant HELPER as helpers.py
    participant RESEARCH as research.py
    participant SEARCH as text_search.py
    participant ONLINE as online_search.py
    participant CODE as run_code.py
    participant OPERATOR as operator
    participant LLM as LLM Adapter
    participant DB as Database

    CLIENT->>API: POST /api/chat (q, n, d, stream)
    API->>HELPER: is_ready_to_chat()
    API->>HELPER: aget_data_sources_and_output_format()
    Note over HELPER: LLM 判断需要哪些数据源和输出模式

    alt 需要 Notes
        API->>HELPER: search_documents()
        HELPER->>SEARCH: query()
        SEARCH->>DB: EntryAdapters.search_with_embeddings()
        DB-->>SEARCH: 搜索结果
        SEARCH-->>HELPER: 搜索命中
    end

    alt 需要 Online
        API->>ONLINE: search_online()
        ONLINE-->>API: 在线搜索结果
    end

    alt 需要 Code
        API->>CODE: run_code()
        CODE-->>API: 代码执行结果
    end

    alt 需要 Operator
        API->>OPERATOR: operate_environment()
        OPERATOR-->>API: 操作结果
    end

    alt 需要 Research
        API->>RESEARCH: research()
        Note over RESEARCH: 多轮迭代: 选择工具 → 执行 → 汇总
        RESEARCH-->>API: 研究结果
    end

    API->>HELPER: agenerate_chat_response()
    HELPER->>LLM: 调用 LLM 生成回复
    LLM-->>HELPER: 流式响应

    HELPER-->>API: ResponseWithThought 流
    API->>DB: save_to_conversation_log()
    API-->>CLIENT: 流式/非流式响应
```

### 4.2 内容索引流程

```mermaid
sequenceDiagram
    participant CLIENT as 客户端
    participant API as api_content.py
    participant HELPER as helpers.py
    participant PROCESSOR as TextToEntries
    participant EMBED as EmbeddingsModel
    participant DB as Database

    CLIENT->>API: PUT /api/content (files)
    API->>API: 解析文件类型和内容
    API->>HELPER: configure_content(user, files, regenerate)

    loop 每种文件类型
        HELPER->>PROCESSOR: text_to_entries.process(files, user)
        PROCESSOR->>PROCESSOR: 解析文件内容
        PROCESSOR->>PROCESSOR: 拆分为条目 (entries)
        PROCESSOR->>DB: 删除旧条目
        PROCESSOR->>DB: 创建新条目
    end

    HELPER->>EMBED: 生成嵌入向量
    EMBED->>DB: 更新 Entry.embeddings

    DB-->>HELPER: 索引完成
    HELPER-->>API: 成功
    API-->>CLIENT: 200 OK
```

---

## 5. 模块依赖关系

```mermaid
graph TB
    subgraph "路由层 (routers/)"
        R_API[api.py]
        R_CHAT[api_chat.py]
        R_CONTENT[api_content.py]
        R_AGENTS[api_agents.py]
        R_AUTO[api_automation.py]
        R_MODEL[api_model.py]
        R_MEM[api_memories.py]
        R_AUTH[auth.py]
        R_HELPERS[helpers.py]
        R_RESEARCH[research.py]
    end

    subgraph "处理器层 (processor/)"
        P_CONV[conversation/]
        P_CONTENT[content/]
        P_EMBED[embeddings.py]
        P_IMAGE[image/]
        P_SPEECH[speech/]
        P_TOOLS[tools/]
        P_OPERATOR[operator/]
    end

    subgraph "搜索层 (search_type/ + search_filter/)"
        S_SEARCH[text_search.py]
        S_FILTER[search_filter/]
    end

    subgraph "数据层 (database/)"
        D_MODELS[models/]
        D_ADAPTERS[adapters/]
    end

    subgraph "工具层 (utils/)"
        U_STATE[state.py]
        U_CONFIG[config.py]
        U_HELPERS[helpers.py]
        U_MODELS[models.py]
    end

    R_CHAT --> R_HELPERS
    R_CHAT --> R_RESEARCH
    R_CHAT --> P_CONV
    R_CHAT --> P_TOOLS
    R_CHAT --> P_OPERATOR
    R_CHAT --> P_IMAGE
    R_CHAT --> P_SPEECH

    R_CONTENT --> P_CONTENT
    R_CONTENT --> P_EMBED

    R_HELPERS --> S_SEARCH
    R_HELPERS --> P_CONV
    R_HELPERS --> P_EMBED

    R_RESEARCH --> P_TOOLS
    R_RESEARCH --> P_OPERATOR

    S_SEARCH --> P_EMBED
    S_SEARCH --> S_FILTER
    S_SEARCH --> D_ADAPTERS

    P_CONV --> D_ADAPTERS
    P_CONTENT --> D_ADAPTERS
    P_TOOLS --> D_ADAPTERS
    P_OPERATOR --> D_ADAPTERS

    D_ADAPTERS --> D_MODELS

    R_API --> U_STATE
    R_CHAT --> U_STATE
    P_EMBED --> U_STATE
    S_SEARCH --> U_STATE
```

---

## 6. 状态管理

### 6.1 全局状态 (state.py)

Khoj 使用模块级全局变量管理应用状态：

```mermaid
classDiagram
    class State {
        +embeddings_model: Dict~str, EmbeddingsModel~
        +cross_encoder_model: Dict~str, CrossEncoderModel~
        +openai_client: OpenAI
        +SearchType: Enum
        +scheduler: BackgroundScheduler
        +schedule_leader_process_lock: ProcessLock
        +anonymous_mode: bool
        +billing_enabled: bool
        +telemetry_disabled: bool
        +telemetry: List~dict~
        +khoj_version: str
        +verbose: int
        +host: str
        +port: int
        +log_file: Path
        +ssl_config: dict
        +device: str
        +cli_args: List~str~
    }
```

### 6.2 搜索模型初始化

```mermaid
flowchart TD
    START[initialize_server] --> CHECK{有有效 AI 模型 API?}
    CHECK -->|是| INIT_OPENAI[初始化 OpenAI 客户端]
    CHECK -->|否| SKIP[跳过]

    INIT_OPENAI --> GET_MODELS[get_or_create_search_models]
    GET_MODELS --> LOOP[遍历每个 SearchModelConfig]
    LOOP --> BI[创建 EmbeddingsModel<br/>Bi-Encoder]
    LOOP --> CROSS[创建 CrossEncoderModel<br/>Cross-Encoder]
    BI --> DONE[存入 state.embeddings_model]
    CROSS --> DONE2[存入 state.cross_encoder_model]

    DONE --> SEARCH_TYPES[configure_search_types]
    DONE2 --> SEARCH_TYPES
    SEARCH_TYPES --> DEFAULT_AGENT[setup_default_agent]
```

---

## 7. 定时任务

```mermaid
flowchart TD
    subgraph "Schedule Leader 选举"
        LOCK[ProcessLock<br/>SCHEDULE_LEADER] --> CHECK{锁存在且有效?}
        CHECK -->|是| PAUSE[scheduler.start<br/>paused=True]
        CHECK -->|否| CREATE[创建新锁<br/>43200s 有效期]
        CREATE --> START[scheduler.start<br/>paused=True → resume]
    end

    subgraph "定时任务列表"
        T1[update_content_index_regularly<br/>每 22-25 小时]
        T2[upload_telemetry<br/>每 2 分钟]
        T3[delete_old_user_requests<br/>每 31 分钟]
        T4[wakeup_scheduler<br/>每 17 分钟]
    end

    subgraph "Leader 维持"
        T4 --> CHECK_LOCK{检查锁有效性}
        CHECK_LOCK -->|有效| WAKEUP[scheduler.wakeup]
        CHECK_LOCK -->|无效| REELECT[重新选举]
        REELECT --> NEW_LOCK[获取新锁]
        NEW_LOCK --> RESUME[scheduler.resume]
    end
```

---

## 8. 错误处理策略

### 8.1 LLM 调用回退

```mermaid
flowchart TD
    CALL[调用 LLM] --> RESULT{调用结果}
    RESULT -->|成功| RETURN[返回响应]
    RESULT -->|失败| CHECK{可重试错误?}
    CHECK -->|是| RETRY[重试 3 次<br/>指数退避]
    RETRY --> RESULT2{重试结果}
    RESULT2 -->|成功| RETURN
    RESULT2 -->|失败| FALLBACK[回退到下一个模型]
    CHECK -->|否| FALLBACK
    FALLBACK --> NEXT_MODEL[使用 ServerChatSettings<br/>优先级列表中的下一个模型]
    NEXT_MODEL --> CALL
```

### 8.2 可重试错误类型

| 提供商 | 可重试错误 |
|--------|-----------|
| OpenAI | RateLimitError, InternalServerError, APITimeoutError |
| Anthropic | RateLimitError, APIError |
| Google | 429, 500, 502, 503, 504 错误码 |
| 通用 | httpx.TimeoutException, httpx.NetworkError, ValueError |

---

## 9. 关键设计决策

### 9.1 FastAPI + Django 混合架构

**决策**: 使用 FastAPI 作为主 Web 框架，Django 仅作为 ORM 和 Admin 层。

**原因**:
- FastAPI 原生支持异步，适合 LLM 流式响应
- Django ORM 成熟稳定，Admin 后台开箱即用
- Django 的迁移系统可靠管理数据库 Schema 演化

**代价**:
- 需要手动管理数据库连接清理（AsyncCloseConnectionsMiddleware）
- 两个框架的中间件链独立运行

### 9.2 pgvector 向量搜索

**决策**: 使用 PostgreSQL + pgvector 而非独立向量数据库。

**原因**:
- 减少运维复杂度（单一数据库）
- 事务一致性（向量与业务数据在同一数据库）
- pgvector 对 Khoj 的数据规模足够高效

### 9.3 全局状态管理

**决策**: 使用模块级全局变量（state.py）管理应用状态。

**原因**:
- 嵌入模型加载耗时，需要全局缓存
- 搜索模型配置在运行时不变
- 简单直接，适合单进程部署

**代价**:
- 不适合多进程共享（需要 Process Lock 机制）
- 测试时需要手动重置状态

### 9.4 Process Lock 分布式调度

**决策**: 使用数据库 ProcessLock 实现分布式调度 Leader 选举。

**原因**:
- 避免多 Worker 重复执行定时任务
- 基于数据库实现，无需额外依赖（如 Redis）
- 自动过期机制防止死锁

---

## 10. 核心模块设计文档索引

| 模块 | 文档路径 | 核心内容 |
|------|----------|----------|
| Conversation | [conversation-module-design.md](./conversation-module-design.md) | 对话流程、多模型适配、流式响应、上下文管理、工具调用 |
| Search | [search-module-design.md](./search-module-design.md) | 双编码器架构、向量搜索、过滤器、结果排序 |
| Content | [content-module-design.md](./content-module-design.md) | 文档解析、TextToEntries 基类、增量索引 |
| Database & API | [database-api-module-design.md](./database-api-module-design.md) | 数据模型、适配器层、认证流程、速率限制、Agent 系统 |
| Tools & Operator | [tools-operator-module-design.md](./tools-operator-module-design.md) | 在线搜索、代码执行、MCP、Operator 架构、Research 模块 |
