# 核心模块设计: Cross-Session Memory (跨会话记忆)

> 源码路径: `cross/`

## 1. 模块概述

Cross-Session Memory 实现了跨对话会话的记忆持久化与上下文注入，使 Agent 能够在不同会话间保持记忆连续性。核心能力：

1. **会话生命周期管理**: start → record → stop(finalize) → end
2. **自动记忆提取**: 会话结束时自动提取关键观察
3. **上下文注入**: 基于 token 预算的智能上下文组装
4. **语义搜索**: 跨所有会话的向量相似度检索

## 2. 模块架构

```mermaid
graph TB
    subgraph "编排层"
        ORC["CrossMemOrchestrator<br/>(统一入口)"]
    end

    subgraph "会话管理"
        SM["SessionManager<br/>(会话生命周期)"]
        SS["SessionState<br/>(会话状态)"]
    end

    subgraph "记忆处理"
        ME["MemoryExtractor<br/>(观察提取)"]
        MS["MemorySummarizer<br/>(摘要生成)"]
    end

    subgraph "存储层"
        DB["Database<br/>(SQLite)"]
        VS["VectorStore<br/>(LanceDB)"]
    end

    subgraph "检索层"
        CTX["ContextAssembler<br/>(上下文组装)"]
        SEARCH["SearchEngine<br/>(语义搜索)"]
    end

    subgraph "数据模型"
        SE["SessionEvent"]
        OB["Observation"]
        SU["Summary"]
        CME["CrossMemoryEntry"]
    end

    ORC --> SM & ME & MS & CTX & SEARCH
    SM --> SS
    SM --> DB
    ME --> DB
    MS --> DB
    ME --> VS
    MS --> VS
    CTX --> VS
    CTX --> DB
    SEARCH --> VS
    SEARCH --> DB

    DB --> SE & OB & SU
    VS --> CME
```

## 3. 会话生命周期

### 3.1 状态机

```mermaid
stateDiagram-v2
    [*] --> Active: start_session()
    Active --> Active: record_message()
    Active --> Active: record_tool_use()
    Active --> Finalizing: stop_session()
    Finalizing --> Finalized: 提取观察 + 生成摘要
    Finalized --> Ended: end_session()
    Ended --> [*]

    state Active {
        [*] --> Recording
        Recording --> Recording: 持续记录事件
    }

    state Finalizing {
        [*] --> Extracting
        Extracting --> Summarizing
        Summarizing --> Storing
        Storing --> [*]
    }
```

### 3.2 完整生命周期时序

```mermaid
sequenceDiagram
    participant Agent as LLM Agent
    participant ORC as CrossMemOrchestrator
    participant SM as SessionManager
    participant DB as Database
    participant ME as MemoryExtractor
    participant MS as MemorySummarizer
    participant VS as VectorStore

    Agent->>ORC: start_session(content_session_id, user_prompt)
    ORC->>SM: create_session()
    SM->>DB: INSERT session
    SM-->>ORC: {memory_session_id, context}

    Note over ORC: context 包含跨会话记忆

    loop 会话进行中
        Agent->>ORC: record_message(session_id, content, role)
        ORC->>SM: add_event(session_id, "message", data)
        SM->>DB: INSERT event

        Agent->>ORC: record_tool_use(session_id, tool, input, output)
        ORC->>SM: add_event(session_id, "tool_use", data)
        SM->>DB: INSERT event
    end

    Agent->>ORC: stop_session(session_id)
    ORC->>ME: extract_observations(session_id)
    ME->>DB: SELECT events WHERE session_id
    ME->>ME: LLM 提取关键观察
    ME->>DB: INSERT observations
    ME->>VS: add_entries(observations)

    ORC->>MS: generate_summary(session_id)
    MS->>DB: SELECT events + observations
    MS->>MS: LLM 生成摘要
    MS->>DB: INSERT summary
    MS->>VS: add_entry(summary)

    ORC-->>Agent: FinalizationReport

    Agent->>ORC: end_session(session_id)
    ORC->>SM: close_session(session_id)
    SM->>DB: UPDATE session SET status='ended'
```

## 4. 数据模型

### 4.1 SQLite 表结构

```mermaid
erDiagram
    SESSIONS {
        string session_id PK
        string content_session_id
        string user_prompt
        string status
        datetime started_at
        datetime ended_at
        int event_count
        int observation_count
        text summary
    }

    EVENTS {
        int id PK
        string session_id FK
        string event_type
        text content
        text metadata
        datetime timestamp
    }

    OBSERVATIONS {
        int id PK
        string session_id FK
        text content
        float importance
        text categories
        text related_entities
        datetime created_at
    }

    SUMMARIES {
        int id PK
        string session_id FK
        text content
        text key_topics
        text action_items
        datetime created_at
    }

    SESSIONS ||--o{ EVENTS : "has"
    SESSIONS ||--o{ OBSERVATIONS : "has"
    SESSIONS ||--o| SUMMARIES : "has"
```

### 4.2 CrossMemoryEntry (向量存储)

```mermaid
classDiagram
    class CrossMemoryEntry {
        +id: str
        +session_id: str
        +content: str
        +entry_type: str
        +importance: float
        +timestamp: str
        +categories: List~str~
        +related_entities: List~str~
        +vector: List~float~
    }
```

## 5. 上下文注入

### 5.1 ContextAssembler

```mermaid
sequenceDiagram
    participant Agent as LLM Agent
    participant ORC as CrossMemOrchestrator
    participant CTX as ContextAssembler
    participant VS as VectorStore
    participant DB as Database

    Agent->>ORC: get_context_for_prompt(user_prompt)
    ORC->>CTX: assemble(user_prompt, token_budget)

    CTX->>VS: semantic_search(user_prompt, top_k)
    VS-->>CTX: 相关记忆条目

    CTX->>DB: get_recent_summaries(limit=3)
    DB-->>CTX: 最近会话摘要

    CTX->>CTX: 按重要性排序
    CTX->>CTX: Token 预算分配

    Note over CTX: 分配策略:<br/>1. 摘要 (30% 预算)<br/>2. 高相关性观察 (50% 预算)<br/>3. 补充观察 (20% 预算)

    CTX->>CTX: 格式化为可读文本
    CTX-->>ORC: context_string
    ORC-->>Agent: context_string
```

### 5.2 Token 预算分配

```mermaid
flowchart TB
    Budget["Token 预算<br/>(默认 2000)"] --> Alloc["预算分配"]

    Alloc -->|"30%"| Summaries["会话摘要<br/>(最近3个)"]
    Alloc -->|"50%"| Relevant["高相关性观察<br/>(向量搜索 Top-K)"]
    Alloc -->|"20%"| Supplement["补充观察<br/>(重要性排序)"]

    Summaries --> Format["格式化"]
    Relevant --> Format
    Supplement --> Format

    Format --> Check{"超出预算?"}
    Check -->|Yes| Truncate["截断低优先级内容"]
    Check -->|No| Output["输出上下文字符串"]

    Truncate --> Output
```

## 6. 语义搜索

### 6.1 搜索流程

```mermaid
sequenceDiagram
    participant User as 用户
    participant ORC as CrossMemOrchestrator
    participant SEARCH as SearchEngine
    participant VS as VectorStore
    participant DB as Database

    User->>ORC: search(query, top_k=5)
    ORC->>SEARCH: search(query, top_k)

    SEARCH->>VS: semantic_search(query, top_k * 2)
    VS-->>SEARCH: 候选记忆条目

    SEARCH->>DB: get_session_info(session_ids)
    DB-->>SEARCH: 会话元数据

    SEARCH->>SEARCH: 合并记忆 + 元数据
    SEARCH->>SEARCH: 去重 + 排序
    SEARCH-->>ORC: List[CrossMemoryEntry]
    ORC-->>User: 搜索结果
```

## 7. 记忆提取

### 7.1 观察提取流程

```mermaid
flowchart TB
    Events["会话事件列表"] --> Group["按事件类型分组"]
    Group --> Messages["用户/助手消息"]
    Group --> ToolUse["工具调用记录"]

    Messages --> LLM1["LLM 提取观察"]
    ToolUse --> LLM2["LLM 提取工具相关观察"]

    LLM1 --> Obs1["观察列表1"]
    LLM2 --> Obs2["观察列表2"]

    Obs1 & Obs2 --> Merge["合并去重"]
    Merge --> Score["重要性评分"]
    Score --> Store["存储到 DB + VS"]

    subgraph "观察结构"
        O["Observation"]
        O --> Content["content: 观察内容"]
        O --> Imp["importance: 0-1"]
        O --> Cat["categories: 分类标签"]
        O --> Ent["related_entities: 相关实体"]
    end

    Store --> O
```

## 8. 多租户隔离

```mermaid
flowchart TB
    Request["请求"] --> Auth{"Token 认证"}
    Auth -->|有效| Tenant["提取 tenant_id"]
    Auth -->|无效| Reject["401 拒绝"]

    Tenant --> Isolate["数据隔离"]
    Isolate --> DB_Q["SQLite: WHERE tenant_id = ?"]
    Isolate --> VS_Q["LanceDB: filter tenant_id = ?"]

    DB_Q --> Result["租户数据"]
    VS_Q --> Result
```

## 9. MCP 集成

### 9.1 MCP 工具映射

```mermaid
flowchart LR
    subgraph "MCP Tools"
        T1["memory_start_session"]
        T2["memory_record_message"]
        T3["memory_record_tool_use"]
        T4["memory_stop_session"]
        T5["memory_end_session"]
        T6["memory_search"]
        T7["memory_get_context"]
    end

    subgraph "CrossMemOrchestrator"
        M1["start_session()"]
        M2["record_message()"]
        M3["record_tool_use()"]
        M4["stop_session()"]
        M5["end_session()"]
        M6["search()"]
        M7["get_context_for_prompt()"]
    end

    T1 --> M1
    T2 --> M2
    T3 --> M3
    T4 --> M4
    T5 --> M5
    T6 --> M6
    T7 --> M7
```

### 9.2 MCP 请求处理时序

```mermaid
sequenceDiagram
    participant Client as MCP Client
    participant Server as HTTP Server
    participant Handler as MCP Handler
    participant Auth as Token Auth
    participant ORC as CrossMemOrchestrator

    Client->>Server: POST /mcp (JSON-RPC 2.0)
    Server->>Handler: handle_request(body)
    Handler->>Auth: validate_token(headers)
    Auth-->>Handler: tenant_id

    Handler->>Handler: 解析 method + params
    Handler->>ORC: 调用对应方法

    ORC-->>Handler: 结果
    Handler-->>Server: JSON-RPC Response
    Server-->>Client: HTTP Response
```

## 10. 配置

| 参数 | 默认值 | 描述 |
|:--|:--|:--|
| db_path | ./data/cross_memory.db | SQLite 数据库路径 |
| vector_db_path | ./data/cross_vectors | LanceDB 向量存储路径 |
| embedding_model | Qwen3-Embedding | 嵌入模型 |
| token_budget | 2000 | 上下文注入 token 预算 |
| max_recent_summaries | 3 | 最近摘要数量 |
| search_top_k | 5 | 搜索返回数量 |
| observation_importance_threshold | 0.3 | 观察重要性阈值 |
