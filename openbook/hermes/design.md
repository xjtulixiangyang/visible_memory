# Hermes Agent 记忆模块 — 设计文档 (Design)

> 版本: 基于 Hermes Agent 官方文档分析

---

## 一、三层知识层总架构

```mermaid
flowchart TB
    U[用户请求]
    AG[AIAgent]
    PB[Prompt Builder]
    LOOP[run_conversation loop]
    MODEL[LLM]

    subgraph KNOW["Hermes 知识层"]
        subgraph SM["会话记忆 (Session Memory)"]
            SDB[SessionDB / state.db]
            SSEARCH[session_search]
            FTS[SQLite FTS5]
        end

        subgraph CM["跨会话记忆 (Cross-Session)"]
            MSTORE[MemoryStore<br/>MEMORY.md / USER.md]
            MTOOL[memory tool]
            MMGR[MemoryManager]
            MPROV[External Memory<br/>Honcho/Mem0/...]
        end

        subgraph PM["程序性记忆 (Procedural/Skills)"]
            SKINDEX[build_skills_system_prompt]
            SVIEW[skill_view]
            SMANAGE[skill_manage]
            SKSTORE[~/.hermes/skills/]
            HUB[skills_hub]
        end
    end

    U --> AG
    AG --> PB
    PB --> SKINDEX
    PB --> MSTORE
    AG --> LOOP
    LOOP --> MODEL

    AG --> SDB
    SDB --> FTS
    AG --> SSEARCH
    SSEARCH --> SDB
```

---

## 二、会话记忆：SessionDB 设计

### 2.1 模块职责

```mermaid
classDiagram
    class SessionDB {
        -db_path: str
        -conn: sqlite3.Connection
        +ensure_session(session_id, title) Session
        +create_session(session_id, title) Session
        +append_message(session_id, role, content, metadata) Message
        +get_messages_as_conversation(session_id) list
        +get_session(session_id) Session
        +list_sessions(limit, offset) list
        +search_sessions(query, limit) list
        +update_session(session_id, updates) Session
        +delete_session(session_id) bool
        +get_statistics() dict
    }

    class Message {
        +id: str
        +session_id: str
        +role: str (user/assistant/tool)
        +content: str
        +tool_name: str
        +tool_call_id: str
        +token_count: int
        +created_at: datetime
    }

    class Session {
        +id: str
        +title: str
        +status: str (active/completed/archived)
        +message_count: int
        +tool_call_count: int
        +total_tokens: int
        +total_cost: float
        +created_at: datetime
        +updated_at: datetime
    }

    SessionDB "1" --> "*" Session
    SessionDB "1" --> "*" Message
    Session "1" --> "*" Message
```

### 2.2 写入与搜索流程

```mermaid
flowchart TD
    subgraph Write["写入路径"]
        W1[run_agent 主循环]
        W2[_flush_messages_to_session_db]
        W3[游标: _last_flushed_db_idx]
        W4[append_message]
        W5[messages 表]
        W6[messages_fts 自动索引]
        W7[sessions 聚合更新]
    end

    subgraph Search["搜索路径"]
        S1[session_search 调用]
        S2[FTS5 全文搜索]
        S3[按 session 聚合]
        S4[辅助 LLM 聚焦摘要]
        S5[结构化召回结果]
    end

    W1 -->|每轮| W2
    W2 --> W3
    W3 --> W4
    W4 --> W5
    W5 --> W6
    W5 --> W7

    S1 --> S2
    S2 --> S3
    S3 --> S4
    S4 --> S5
```

---

## 三、跨会话记忆：MemoryStore 设计

### 3.1 内建记忆架构

```mermaid
classDiagram
    class MemoryStore {
        -memories_dir: str
        -memory_file: str (MEMORY.md)
        -user_file: str (USER.md)
        +read_memory() str
        +write_memory(content) bool
        +read_user_profile() str
        +write_user_profile(content) bool
        +search(query) list
        +get_summary() str
    }

    class MemoryManager {
        -provider: MemoryProvider
        +get_context() str
        +add_memory(content, metadata) bool
        +search_memories(query) list
        +prefetch_context() str
        +sync() bool
    }

    class MemoryProvider {
        <<abstract>>
        +add(content, metadata) bool
        +search(query, limit) list
        +delete(id) bool
        +get_context(session_id) str
    }

    class HonchoProvider {
        +add() bool
        +search() list
        +delete() bool
        +get_context() str
    }

    class Mem0Provider {
        +add() bool
        +search() list
        +delete() bool
        +get_context() str
    }

    MemoryManager o-- MemoryProvider
    MemoryProvider <|.. HonchoProvider
    MemoryProvider <|.. Mem0Provider
```

### 3.2 记忆注入时机

```mermaid
sequenceDiagram
    participant Agent as AIAgent
    participant PB as PromptBuilder
    participant Store as MemoryStore
    participant Mgr as MemoryManager
    participant Provider as External Provider

    Note over Agent: 会话启动

    Agent->>PB: build_prompt()
    PB->>Store: read_memory() + read_user_profile()
    Store-->>PB: MEMORY.md + USER.md 内容

    PB->>Mgr: get_context()
    Mgr->>Provider: prefetch_context()
    Provider-->>Mgr: 相关记忆
    Mgr-->>PB: 外部记忆上下文

    PB-->>Agent: 完整系统提示 (含记忆)

    Note over Agent: 会话进行中

    Agent->>Store: memory tool (读/写)
    Store-->>Agent: 结果

    Agent->>Mgr: memory tool (外部)
    Mgr->>Provider: add / search
    Provider-->>Mgr: 结果
    Mgr-->>Agent: 结果

    Note over Agent: 回合后

    Agent->>Mgr: sync()
    Mgr->>Provider: 同步更新
```

---

## 四、技能记忆：Skill 系统设计

### 4.1 技能目录结构

```
~/.hermes/skills/
├── skill-name-1/
│   ├── SKILL.md          # 技能定义（必需）
│   ├── references/       # 参考文档
│   ├── templates/        # 代码模板
│   ├── scripts/          # 可执行脚本
│   └── assets/           # 静态资源
├── skill-name-2/
│   └── ...
```

### 4.2 技能注入策略

```mermaid
flowchart LR
    subgraph Startup["启动时"]
        A[扫描 skill 目录]
        B[构建技能索引<br/>名称 + 描述 + 版本]
        C[注入系统提示]
    end

    subgraph OnDemand["按需"]
        D[Agent 决定使用技能]
        E["skill_view 加载全文"]
        F[执行技能逻辑]
    end

    subgraph Creation["创建时"]
        G[Agent 完成工作]
        H["create_skill 沉淀方法"]
        I[写入 skill 目录]
    end

    A --> B --> C
    D --> E --> F
    G --> H --> I
```

---

## 五、上下文压缩引擎设计

### 5.1 五阶段压缩流程

```mermaid
stateDiagram-v2
    [*] --> Phase1: 触发压缩

    Phase1 --> Phase2: 预剪枝完成
    Phase1 --> [*]: 无可修剪

    Phase2 --> Phase3: 边界确定
    Phase2 --> [*]: 无需压缩

    Phase3 --> Phase4: 摘要生成成功
    Phase3 --> Phase3: 重试 (最多2次)
    Phase3 --> Phase5: 摘要失败 (丢弃中间段)

    Phase4 --> Phase5: 消息重组

    Phase5 --> [*]: 清理完成
```

### 5.2 头尾保护示意图

```
┌─────────────────────────────────────────────────────────┐
│  头保护 (protect_first_n = 3)                            │
│  ├─ system prompt                                        │
│  ├─ 第一条 user message                                  │
│  └─ 第一条 assistant response                            │
├─────────────────────────────────────────────────────────┤
│  中间段 (可压缩区域)                                      │
│  ┌─────────────────────────────────────────────────────┐ │
│  │  10轮工具调用序列                                    │ │
│  │  → LLM 摘要为结构化文本                              │ │
│  │  assistant: [摘要占位符]                              │ │
│  └─────────────────────────────────────────────────────┘ │
├─────────────────────────────────────────────────────────┤
│  尾保护 (protect_last_n = 20)                            │
│  ├─ 最近的 user message                                  │
│  ├─ 最近的 tool calls                                    │
│  └─ 最近的 assistant responses                           │
└─────────────────────────────────────────────────────────┘
```

### 5.3 反抖动机制

```mermaid
flowchart TD
    COMP[开始压缩] --> CHECK{上次摘要 vs 本次<br/>相似度 > 阈值?}
    CHECK -->|是| INC[增加无效计数]
    CHECK -->|否| RESET[重置计数]

    INC --> LIMIT{计数 > 上限?}
    LIMIT -->|是| STOP[停止压缩<br/>冷却 60s]
    LIMIT -->|否| CONTINUE[继续正常压缩]

    RESET --> CONTINUE

    STOP --> WAIT[等待冷却]
    WAIT -->|冷却结束| COMP
```

---

## 六、三层记忆的数据流汇总

```mermaid
flowchart TB
    subgraph Ingest["写入路径"]
        I1["用户/Agent 产生消息"]
        I2["_flush 持久化到 SessionDB"]
        I3["memory tool 写入 MEMORY.md"]
        I4["External Provider add"]
        I5["create_skill 沉淀技能"]
    end

    subgraph Index["索引路径"]
        J1["FTS5 自动索引消息"]
        J2["文件系统索引技能"]
    end

    subgraph Retrieve["检索路径"]
        K1["session 启动加载 MEMORY/USER.md"]
        K2["session_search FTS5 搜索"]
        K3["skill_list 索引注入"]
        K4["External prefetch_context"]
    end

    subgraph Compress["压缩路径"]
        L1["达到阈值触发"]
        L2["先 flush 到 SessionDB"]
        L3["五阶段压缩"]
        L4["替换消息列表"]
    end

    I1 --> I2
    I2 --> J1
    I3 --> MEM[(MEMORY.md)]
    I4 --> EXT[(External Store)]
    I5 --> SKI[(Skills 目录)]

    K1 --> MEM
    K2 --> J1
    K3 --> SKI
    K4 --> EXT

    L1 --> L2
    L2 --> L3
    L3 --> L4
```

---

## 七、关键设计决策

| 决策 | 选择 | 理由 |
|------|------|------|
| 记忆分层 | 三层独立系统 | 不同类型的知识有不同的生命周期和访问模式，混在一起会导致注入成本失控 |
| 会话搜索 | FTS5 + 辅助 LLM 摘要 | 全文搜索快但缺乏语义理解，辅助 LLM 补充摘要能力 |
| 跨会话记忆 | 内建文件 + 外部 Provider | 文件提供透明性和可读性，Provider 提供 AI 原生语义搜索能力 |
| 技能存储 | 目录 + SKILL.md | 兼容版本控制 (git)，支持复杂的配套文件 |
| 压缩模型 | 独立辅助模型 | 不阻塞主模型推理，便宜的模型完成摘要任务 |
| 头尾保护 | 永远不压缩 | 系统提示和近期上下文是决策的关键输入，不能丢失 |
| 反抖动 | 相似度检测 + 冷却 | 防止无效压缩无限循环浪费 token |
