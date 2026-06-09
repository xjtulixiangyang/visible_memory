# OpenClaw 记忆模块 — 设计文档 (Design)

> 版本: v2026.6 | 基于 OpenClaw 官方文档分析

---

## 一、系统总体架构

```mermaid
flowchart TB
    subgraph User["用户层"]
        CHAT[聊天消息]
        CMD["CLI 命令"]
        FILE[直接编辑文件]
    end

    subgraph Agent["Agent 层"]
        MAIN[主 Agent]
        AM[Active Memory<br/>子代理]
        ENGINE[上下文引擎<br/>Context Engine]
    end

    subgraph Memory["记忆核心层"]
        MC["memory-core 插件"]
        MW["memory-wiki 插件"]
        DREAM["Dreaming 系统"]
        FLUSH[自动 Flush<br/>压缩前保存]
    end

    subgraph Store["存储层"]
        MEM["MEMORY.md<br/>长期记忆"]
        DAILY["memory/YYYY-MM-DD.md<br/>每日笔记"]
        DREAMS["DREAMS.md<br/>梦境日记"]
        SQLITE["SQLite 索引<br/>FTS5 + 向量"]
        EXTERNAL["QMD / Honcho<br/>外部后端"]
    end

    subgraph Embedding["Embedding 层"]
        OPENAI[OpenAI]
        GEMINI[Gemini]
        LOCAL[Local GGUF]
        COPILOT[GitHub Copilot]
        VOYAGE[Voyage]
        MISTRAL[Mistral]
        BEDROCK[Bedrock]
        OLLAMA[Ollama]
    end

    CHAT --> MAIN
    MAIN -->|搜索| MC
    MC --> MEM & DAILY & SQLITE
    MAIN -->|会话启动| MEM

    AM -->|自动搜索注入| MAIN
    MAIN -->|压缩前| FLUSH
    FLUSH -->|保存上下文| DAILY

    DREAM -->|深度阶段| MEM
    DREAM -->|审查输出| DREAMS

    MW -->|编译| SQLITE
    MW -->|结构知识库| EXTERNAL

    SQLITE --> OPENAI & GEMINI & LOCAL & COPILOT & VOYAGE & MISTRAL & BEDROCK & OLLAMA
```

---

## 二、核心模块设计

### 2.1 嵌入 Provider 自动检测优先级

```mermaid
flowchart TD
    START[开始检测] --> L1{local.modelPath 配置?}
    L1 -->|是| LOCAL[Local GGUF]
    L1 -->|否| L2{Copilot Token?}
    L2 -->|是| COPILOT[GitHub Copilot]
    L2 -->|否| L3{OpenAI Key?}
    L3 -->|是| OPENAI[OpenAI]
    L3 -->|否| L4{Gemini Key?}
    L4 -->|是| GEMINI[Gemini]
    L4 -->|否| L5{Voyage Key?}
    L5 -->|是| VOYAGE[Voyage]
    L5 -->|否| L6{Mistral Key?}
    L6 -->|是| MISTRAL[Mistral]
    L6 -->|否| BEDROCK[AWS Bedrock<br/>Instance Role / SSO / AK]

    LOCAL --> DONE[选定 Provider]
    COPILOT --> DONE
    OPENAI --> DONE
    GEMINI --> DONE
    VOYAGE --> DONE
    MISTRAL --> DONE
    BEDROCK --> DONE
```

### 2.2 索引管道

```mermaid
sequenceDiagram
    participant Watch as 文件监听
    participant Index as 索引引擎
    participant Chunk as 分块器
    participant Embed as 嵌入服务
    participant DB as SQLite 数据库

    Note over Watch: 监控 MEMORY.md + memory/*.md

    Watch->>Index: 文件变更事件
    Index->>Index: 1.5s 防抖去重
    Index->>Index: 计算变更差异

    Index->>Chunk: 解析 Markdown
    Chunk->>Chunk: 分块 (~400 tokens, 80 重叠)
    Chunk->>DB: 存储文本块

    Index->>Embed: 批量嵌入
    Embed->>Embed: 调用 Provider API
    Embed-->>Index: vectors[]

    Index->>DB: 写入向量索引
    Index->>DB: 更新 FTS5 索引

    Note over DB: 自动合并变更

    Index-->>Watch: 索引完成
```

### 2.3 Active Memory 激活模型

Active Memory 采用**二门控模型**：

```mermaid
flowchart TD
    REQ[收到消息] --> G1{插件启用?}
    G1 -->|否| SKIP[不激活]
    G1 -->|是| G2{匹配 Agent ID?}
    G2 -->|否| SKIP
    G2 -->|是| G3{可交互持久会话?}
    G3 -->|否| SKIP
    G3 -->|是| AM[Active Memory 启动]

    AM --> Q[查询记忆]
    Q --> R[排序过滤]
    R --> I[注入上下文]
    I --> REPLY[主 Agent 回复]

    style AM fill:#e1f5fe
    style REPLY fill:#c8e6c9
```

### 2.4 Dreaming 评分模型

```mermaid
flowchart LR
    subgraph Signals["评分信号"]
        FREQ[频率: 被召回次数]
        RECALL[召回频率: 时间分布]
        DIVERSITY[查询多样性: 不同场景]
    end

    subgraph Gates["三道门"]
        SCORE[分数阈值]
        FREQ_GATE[频率门槛]
        DIV_GATE[多样性格]
    end

    subgraph Result["结果"]
        PROMOTE["→ 提升到 MEMORY.md"]
        DIARY["→ 写入 DREAMS.md"]
        REJECT["→ 留在短期存储"]
    end

    Signals --> SCORE
    Signals --> FREQ_GATE
    Signals --> DIV_GATE
    SCORE & FREQ_GATE & DIV_GATE -->|全部通过| PROMOTE
    SCORE & FREQ_GATE & DIV_GATE -->|通过部分| DIARY
    SCORE & FREQ_GATE & DIV_GATE -->|不通过| REJECT
```

---

## 三、记忆文件生命周期

```mermaid
stateDiagram-v2
    [*] --> 会话启动

    会话启动 --> 加载 MEMORY.md
    加载 MEMORY.md --> 加载昨日 memory/YYYY-MM-DD.md
    加载昨日 memory/YYYY-MM-DD.md --> 加载今日 memory/YYYY-MM-DD.md

    加载今日 memory/YYYY-MM-DD.md --> 会话进行中

    会话进行中 --> 用户主动记忆 : ask Agent to remember
    用户主动记忆 --> 写入 MEMORY.md

    会话进行中 --> 自动观察 : Agent 捕获上下文
    自动观察 --> 写入 memory/YYYY-MM-DD.md

    会话进行中 --> 压缩触发 : context window full
    压缩触发 --> 自动 Flush : 保存未持久化上下文
    自动 Flush --> 写入 memory/YYYY-MM-DD.md

    压缩触发 --> 压缩完成

    压缩完成 --> 会话进行中

    会话进行中 --> Dreaming 触发 : 定时/bg
    Dreaming 触发 --> 评分审核
    评分审核 --> 写入 MEMORY.md
    评分审核 --> 写入 DREAMS.md

    会话进行中 --> 会话结束
    会话结束 --> 写入 memory/YYYY-MM-DD.md
    会话结束 --> [*]
```

---

## 四、搜索算法详解

### 4.1 混合搜索评分

```
Score = w_vector × vectorSimilarity + w_bm25 × bm25Score

其中:
- w_vector 和 w_bm25 由配置决定，自动归一化
- vectorSimilarity = cosine(queryEmbedding, docEmbedding)
- bm25Score = Σ IDF(qi) × TF(qi, d) × (k1+1) / (TF(qi,d) + k1 × (1-b + b × |d|/avgdl))
```

### 4.2 后处理管道

```
Raw Results
  → 时间衰减: score × e^(-λ × daysSinceIndexed)
  → MMR 多样化: 惩罚与已选结果相似度高的候选
  → Top-K 截断
  → 返回
```

---

## 五、插件架构

```mermaid
flowchart TB
    subgraph Plugins["记忆插件体系"]
        MC["memory-core<br/>主动记忆 + 召回 + Dreaming"]
        MW["memory-wiki<br/>编译知识库 + 结构文档"]
    end

    subgraph Slots["插件槽位"]
        CE["contextEngine<br/>上下文引擎"]
        AM_SLOT["active-memory<br/>主动记忆子代理"]
    end

    subgraph Ext["外部集成"]
        QMD_P["QMD Provider"]
        HONCHO_P["Honcho Provider"]
    end

    MC -->|提供工具| memory_search
    MC -->|提供工具| memory_get
    MC -->|管理| Dreaming
    MC -->|管理| 自动 Flush

    MW -->|注册| wiki_search
    MW -->|注册| wiki_get
    MW -->|注册| wiki_apply
    MW -->|注册| wiki_lint

    MW -->|输出| OBSIDIAN[Obsidian 兼容]
    MW -->|输出| DASH[Dashboard]

    MC --> QMD_P & HONCHO_P
```

---

## 六、关键设计决策

### 6.1 为什么用文件而不是数据库？

| 维度 | 文件方案 | 纯数据库方案 |
|------|----------|-------------|
| 透明性 | 用户可直接编辑和查看 | 需工具访问 |
| 兼容性 | 版本控制友好 (git) | 需导出 |
| Agent 操控 | Agent 可直接读写 Markdown | 需 API |
| 搜索 | 依赖索引层 | 内置搜索 |

**决策**：文件作为持久化层 + SQLite 作为搜索索引，兼顾透明性和搜索性能。

### 6.2 为什么 Hybrid 搜索？

- 向量搜索擅长语义匹配但不擅长精确查找（ID、错误码）
- BM25 关键词擅长精确匹配但不理解语义
- 两者融合覆盖更广的查询场景

### 6.3 Active Memory 为什么是插件？

- 不是所有用户都需要主动记忆注入
- LLM 调用有额外成本（token + 延迟）
- 可配置性：目标 Agent、会话类型、超时时间、fallback 模型

---

## 七、数据流汇总

```mermaid
flowchart TB
    subgraph Write["写入路径"]
        A[用户/Agent 写 MEMORY.md]
        B["Agent 写 memory/YYYY-MM-DD.md"]
        C["Dreaming 提升到 MEMORY.md"]
        D["Auto Flush 保存上下文"]
    end

    subgraph Index["索引路径"]
        E[文件变更]
        F[防抖 1.5s]
        G[分块 + 嵌入 + FTS5]
        H[SQLite 索引更新]
    end

    subgraph Read["读取路径"]
        I["会话启动加载 MEMORY.md"]
        J["会话启动加载 memory/*.md"]
        K["memory_search 混合搜索"]
        L["Active Memory 自动注入"]
    end

    A --> E
    B --> E
    C --> E
    D --> E
    E --> F --> G --> H

    I --> MA[(MEMORY.md)]
    J --> DA[(memory/*.md)]
    K --> H
    L --> H
```
