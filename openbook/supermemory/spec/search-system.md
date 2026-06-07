# 搜索系统核心模块设计文档

## 1. 模块概述

Supermemory 搜索系统是项目的核心能力模块，负责在海量记忆和文档数据中进行高效的语义检索。系统提供两代搜索接口——V3（文档块级别）和 V4（记忆条目级别），并支持查询改写、重排序、高级过滤等增强能力。

```mermaid
graph TB
    subgraph 搜索系统
        V3[V3 搜索<br/>文档块级别]
        V4[V4 搜索<br/>记忆条目级别]
        QR[查询改写]
        RR[重排序]
        FS[过滤系统]
    end

    Client[客户端调用] --> V3
    Client --> V4
    MCP[MCP 服务器] --> V4
    AISDK[AI SDK 工具] --> V3

    V3 --> QR
    V3 --> RR
    V3 --> FS
    V4 --> QR
    V4 --> RR
    V4 --> FS

    style V3 fill:#4a9eff,color:#fff
    style V4 fill:#ff6b6b,color:#fff
    style QR fill:#ffd93d,color:#333
    style RR fill:#6bcb77,color:#fff
    style FS fill:#9b59b6,color:#fff
```

---

## 2. V3 与 V4 搜索对比

```mermaid
graph LR
    subgraph V3["V3 搜索 — 文档块级别"]
        direction TB
        V3Q[查询字符串] --> V3VEC[向量相似度计算]
        V3VEC --> V3CHUNK[文档块检索]
        V3CHUNK --> V3RERANK[可选重排序]
        V3RERANK --> V3RESULT[SearchResultSchema]
    end

    subgraph V4["V4 搜索 — 记忆条目级别"]
        direction TB
        V4Q[查询字符串] --> V4SEM[语义搜索]
        V4SEM --> V4MEM[记忆条目检索]
        V4MEM --> V4CTX[上下文关联<br/>parents/children]
        V4CTX --> V4RESULT[MemorySearchResult]
    end
```

### 核心差异对照

| 维度 | V3 搜索 | V4 搜索 |
|------|---------|---------|
| **检索单位** | 文档块（Chunk） | 记忆条目（Memory） |
| **核心算法** | 向量相似度 | 语义搜索 |
| **容器标签** | `containerTags`（复数，数组） | `containerTag`（单数，字符串） |
| **阈值参数** | `chunkThreshold` + `documentThreshold` | `threshold`（默认 0.6） |
| **结果结构** | chunks 数组 + 文档元数据 | memory + context 关系链 |
| **搜索模式** | 无 | memories / documents / hybrid |
| **关联能力** | 无 | parents/children 关系 |
| **包含选项** | includeFullDocs / includeSummary / onlyMatchingChunks | include: documents / summaries / relatedMemories |

---

## 3. V3 搜索流程

### 3.1 请求参数

```mermaid
classDiagram
    class SearchRequestSchema {
        +string q [必填]
        +string[] containerTags
        +float chunkThreshold 0~1
        +float documentThreshold 0~1
        +FilterGroup filters
        +int limit 1~100
        +bool rerank
        +bool rewriteQuery
        +bool includeFullDocs
        +bool includeSummary
        +bool onlyMatchingChunks
        +string docId
    }

    class FilterGroup {
        +FilterCondition[] AND
        +FilterCondition[] OR
    }

    class FilterCondition {
        +string key
        +string operator
        +string value
    }

    SearchRequestSchema --> FilterGroup : filters
    FilterGroup --> FilterCondition : AND/OR
```

### 3.2 搜索流程

```mermaid
flowchart TD
    A[接收 SearchRequest] --> B{rewriteQuery?}
    B -->|是| C[查询改写<br/>≈400ms 延迟]
    B -->|否| D[使用原始查询]
    C --> D

    D --> E{docId 存在?}
    E -->|是| F[限定文档内搜索]
    E -->|否| G[全局搜索]

    F --> H[向量相似度计算]
    G --> H

    H --> I[应用 chunkThreshold 过滤]
    I --> J[应用 documentThreshold 过滤]
    J --> K[应用 containerTags 过滤]
    K --> L[应用 filters 高级过滤]

    L --> M{rerank?}
    M -->|是| N[重排序]
    M -->|否| O[跳过重排序]
    N --> O

    O --> P[按 limit 截断结果]
    P --> Q{includeFullDocs?}
    Q -->|是| R[附加完整文档内容]
    Q -->|否| S[仅返回匹配块]

    R --> T{includeSummary?}
    S --> T
    T -->|是| U[附加文档摘要]
    T -->|否| V[跳过摘要]

    U --> W{onlyMatchingChunks?}
    V --> W
    W -->|是| X[仅保留匹配块]
    W -->|否| Y[返回全部块]
    X --> Z[返回 SearchResultSchema]
    Y --> Z
```

### 3.3 结果结构

```mermaid
classDiagram
    class SearchResultSchema {
        +Chunk[] chunks
        +string documentId
        +float score
        +object metadata
        +string title
        +string summary
        +string content
    }

    class Chunk {
        +string content
        +bool isRelevant
        +float score
    }

    SearchResultSchema --> Chunk : chunks
```

---

## 4. V4 搜索流程

### 4.1 请求参数

```mermaid
classDiagram
    class Searchv4RequestSchema {
        +string q [必填]
        +string containerTag
        +float threshold 0~1 默认0.6
        +FilterGroup filters
        +IncludeOptions include
        +bool rerank
        +bool rewriteQuery
    }

    class IncludeOptions {
        +bool documents
        +bool summaries
        +bool relatedMemories
    }

    Searchv4RequestSchema --> IncludeOptions : include
```

### 4.2 三种搜索模式

```mermaid
graph TB
    subgraph 搜索模式
        MEM[memories<br/>仅记忆条目]
        DOC[documents<br/>仅文档]
        HYB[hybrid<br/>混合模式]
    end

    QUERY[查询请求] --> MODE{搜索模式}
    MODE -->|memories| MEM
    MODE -->|documents| DOC
    MODE -->|hybrid| HYB

    MEM --> MEM_R[记忆条目结果集]
    DOC --> DOC_R[文档结果集]
    HYB --> HYB_R[合并结果集]

    HYB_R --> MERGE[相关性融合排序]
    MERGE --> FINAL[最终结果]
    MEM_R --> FINAL
    DOC_R --> FINAL
```

### 4.3 搜索流程

```mermaid
flowchart TD
    A[接收 Searchv4Request] --> B{rewriteQuery?}
    B -->|是| C[查询改写]
    B -->|否| D[使用原始查询]
    C --> D

    D --> E[语义搜索执行]
    E --> F[应用 threshold 过滤<br/>默认 0.6]
    F --> G[应用 containerTag 过滤]
    G --> H[应用 filters 高级过滤]

    H --> I{rerank?}
    I -->|是| J[重排序]
    I -->|否| K[跳过]
    J --> K

    K --> L[构建 context 关系链]
    L --> L1[查询 parents 关系<br/>updates / extends / derives]
    L --> L2[查询 children 关系<br/>updates / extends / derives]

    L1 --> M[组装 MemorySearchResult]
    L2 --> M

    M --> N{include.documents?}
    N -->|是| O[附加关联文档]
    N -->|否| P[跳过]
    O --> P

    P --> Q{include.summaries?}
    Q -->|是| R[附加摘要]
    Q -->|否| S[跳过]
    R --> S

    S --> T{include.relatedMemories?}
    T -->|是| U[附加关联记忆]
    T -->|否| V[跳过]
    U --> V

    V --> W[返回 MemorySearchResult]
```

### 4.4 结果结构

```mermaid
classDiagram
    class MemorySearchResult {
        +string id
        +string memory
        +float similarity
        +int version
        +Context context
        +Document[] documents
    }

    class Context {
        +RelationNode[] parents
        +RelationNode[] children
    }

    class RelationNode {
        +string relation
        +int version
        +string memory
        +object metadata
    }

    class Document {
        +string id
        +string title
        +string content
    }

    MemorySearchResult --> Context : context
    Context --> RelationNode : parents/children
    MemorySearchResult --> Document : documents
```

### 4.5 记忆关系链

```mermaid
graph TD
    M1[记忆 v1] -->|updates| M2[记忆 v2]
    M2 -->|extends| M3[记忆 v3]
    M1 -->|derives| M4[派生记忆]

    M3 -.->|parents| M2
    M3 -.->|parents| M1
    M2 -.->|children| M3
    M1 -.->|children| M4

    style M1 fill:#4a9eff,color:#fff
    style M2 fill:#6bcb77,color:#fff
    style M3 fill:#ffd93d,color:#333
    style M4 fill:#9b59b6,color:#fff
```

关系类型说明：

| 关系 | 含义 | 方向 |
|------|------|------|
| `updates` | 新版本更新 | 旧 → 新 |
| `extends` | 内容扩展 | 基础 → 扩展 |
| `derives` | 派生关系 | 原始 → 派生 |

---

## 5. 查询改写（rewriteQuery）

查询改写通过 LLM 对用户原始查询进行语义优化，提升检索质量，代价是增加约 400ms 延迟。

```mermaid
flowchart LR
    A[用户原始查询] --> B{rewriteQuery?}
    B -->|否| C[直接使用原始查询]
    B -->|是| D[LLM 查询改写]
    D --> E[优化后的查询]
    E --> F[向量检索]

    C --> F

    style D fill:#ffd93d,color:#333
```

### 改写策略

```mermaid
flowchart TD
    A[原始查询] --> B[意图分析]
    B --> C[关键词扩展]
    C --> D[语义消歧]
    D --> E[查询重构]
    E --> F[改写后查询]

    subgraph 改写流程
        B
        C
        D
        E
    end
```

**典型改写示例：**

| 原始查询 | 改写后查询 | 改写类型 |
|----------|-----------|---------|
| "那个API" | "Supermemory API 接口文档" | 消歧 + 扩展 |
| "怎么做" | "如何使用 Supermemory 创建记忆" | 上下文补全 |
| "错误" | "Supermemory 常见错误和异常处理" | 领域限定 |

---

## 6. 重排序（rerank）

重排序对初步检索结果进行二次排序，提升结果相关性。

```mermaid
flowchart TD
    A[初步检索结果] --> B[Rerank 模型]
    B --> C[相关性评分]
    C --> D[按新分数排序]
    D --> E[重排序后结果]

    style B fill:#6bcb77,color:#fff
```

### 重排序在搜索流程中的位置

```mermaid
flowchart LR
    A[向量检索] --> B[阈值过滤]
    B --> C[标签过滤]
    C --> D[高级过滤]
    D --> E[重排序]
    E --> F[截断输出]

    style E fill:#6bcb77,color:#fff
```

---

## 7. 过滤系统

### 7.1 过滤层级

```mermaid
flowchart TD
    A[原始检索结果] --> B[阈值过滤<br/>chunkThreshold / documentThreshold / threshold]
    B --> C[容器标签过滤<br/>containerTags / containerTag]
    C --> D[文档限定过滤<br/>docId]
    D --> E[高级过滤<br/>filters AND/OR 逻辑]
    E --> F[过滤后结果]
```

### 7.2 高级过滤 AND/OR 逻辑

```mermaid
flowchart TD
    A[filters 对象] --> B{逻辑类型}
    B -->|AND| C[所有条件必须满足]
    B -->|OR| D[任一条件满足即可]

    C --> C1[条件1: key=tag, op=eq, val=work]
    C --> C2[条件2: key=type, op=neq, val=draft]
    C1 --> E[交集结果]
    C2 --> E

    D --> D1[条件1: key=source, op=eq, val=web]
    D --> D2[条件2: key=source, op=eq, val=api]
    D1 --> F[并集结果]
    D2 --> F
```

### 7.3 过滤条件结构

```mermaid
classDiagram
    class FilterGroup {
        +FilterCondition[] AND
        +FilterCondition[] OR
    }

    class FilterCondition {
        +string key
        +string operator
        +string value
    }

    FilterGroup --> FilterCondition : AND 条件组
    FilterGroup --> FilterCondition : OR 条件组

    note for FilterCondition "operator 可选值:\neq - 等于\nneq - 不等于\ngt - 大于\nlt - 小于\ncontains - 包含"
```

---

## 8. 完整搜索时序图

### 8.1 V3 搜索时序

```mermaid
sequenceDiagram
    participant C as 客户端
    participant S as 搜索服务
    participant QR as 查询改写
    participant VDB as 向量数据库
    participant RR as 重排序服务
    participant DB as 文档存储

    C->>S: SearchRequest(q, containerTags, ...)
    opt rewriteQuery = true
        S->>QR: 改写查询
        QR-->>S: 改写后查询 (~400ms)
    end

    S->>VDB: 向量相似度检索
    VDB-->>S: 候选块列表

    S->>S: 应用 chunkThreshold 过滤
    S->>S: 应用 documentThreshold 过滤
    S->>S: 应用 containerTags 过滤
    S->>S: 应用 filters 高级过滤

    opt rerank = true
        S->>RR: 重排序候选结果
        RR-->>S: 重排序后结果
    end

    S->>S: 按 limit 截断

    opt includeFullDocs = true
        S->>DB: 获取完整文档
        DB-->>S: 文档内容
    end

    opt includeSummary = true
        S->>DB: 获取文档摘要
        DB-->>S: 摘要内容
    end

    S-->>C: SearchResultSchema
```

### 8.2 V4 搜索时序

```mermaid
sequenceDiagram
    participant C as 客户端
    participant S as 搜索服务
    participant QR as 查询改写
    participant SEM as 语义搜索引擎
    participant RR as 重排序服务
    participant CTX as 上下文服务
    participant DB as 文档存储

    C->>S: Searchv4Request(q, containerTag, ...)
    opt rewriteQuery = true
        S->>QR: 改写查询
        QR-->>S: 改写后查询 (~400ms)
    end

    S->>SEM: 语义搜索执行
    SEM-->>S: 候选记忆列表

    S->>S: 应用 threshold 过滤 (默认0.6)
    S->>S: 应用 containerTag 过滤
    S->>S: 应用 filters 高级过滤

    opt rerank = true
        S->>RR: 重排序候选结果
        RR-->>S: 重排序后结果
    end

    S->>CTX: 查询 parents 关系
    CTX-->>S: parents 关系链
    S->>CTX: 查询 children 关系
    CTX-->>S: children 关系链

    opt include.documents = true
        S->>DB: 获取关联文档
        DB-->>S: 文档列表
    end

    opt include.summaries = true
        S->>DB: 获取摘要
        DB-->>S: 摘要内容
    end

    opt include.relatedMemories = true
        S->>CTX: 获取关联记忆
        CTX-->>S: 关联记忆列表
    end

    S-->>C: MemorySearchResult
```

### 8.3 MCP 服务器搜索时序

```mermaid
sequenceDiagram
    participant Client as MCP 客户端
    participant MC as SupermemoryClient
    participant API as Supermemory API

    Client->>MC: search(query, threshold?)
    MC->>API: client.search.memories(q, searchMode: "hybrid", threshold?)
    API-->>MC: 搜索结果

    MC->>MC: 区分结果类型
    alt 类型 = memory
        MC->>MC: 处理记忆条目
    else 类型 = chunk
        MC->>MC: 处理文档块
    end

    MC-->>Client: 统一搜索结果
```

### 8.4 AI SDK 搜索工具时序

```mermaid
sequenceDiagram
    participant AI as AI 模型
    participant Tool as searchMemoriesTool
    participant API as Supermemory API

    AI->>Tool: informationToGet, includeFullDocs?, limit?
    Tool->>Tool: 设置默认参数<br/>includeFullDocs=true, limit=10<br/>chunkThreshold=0.6
    Tool->>API: client.search.execute(params)
    API-->>Tool: 搜索结果
    Tool-->>AI: 格式化结果
```

---

## 9. 外部集成搜索

### 9.1 MCP 服务器搜索

```mermaid
classDiagram
    class SupermemoryClient {
        +search(query, threshold?) SearchResult
    }

    class SearchParams {
        +string q
        +string searchMode "hybrid"
        +float threshold 可选
    }

    class SearchResult {
        +string type memory | chunk
        +object data
    }

    SupermemoryClient --> SearchParams : 构造参数
    SupermemoryClient --> SearchResult : 返回结果
```

MCP 服务器搜索特点：
- 使用 `client.search.memories()` 接口
- 固定 `searchMode: "hybrid"` 混合模式
- 支持可选 `threshold` 参数
- 结果区分 `memory` 和 `chunk` 两种类型

### 9.2 AI SDK 搜索工具

```mermaid
classDiagram
    class searchMemoriesTool {
        +execute(params) SearchResult
    }

    class ToolParams {
        +string informationToGet [必填]
        +bool includeFullDocs 默认true
        +int limit 默认10
    }

    class InternalConfig {
        +float chunkThreshold 0.6
    }

    searchMemoriesTool --> ToolParams : 外部参数
    searchMemoriesTool --> InternalConfig : 内部配置
```

AI SDK 搜索工具特点：
- 使用 `client.search.execute()` 接口
- `informationToGet` 为必填参数
- `includeFullDocs` 默认 `true`
- `limit` 默认 `10`
- `chunkThreshold` 内部固定为 `0.6`

---

## 10. 搜索模式对比总结

```mermaid
graph TB
    subgraph 入口层
        V3_API[V3 API]
        V4_API[V4 API]
        MCP_S[MCP 服务器]
        AI_SDK[AI SDK 工具]
    end

    subgraph 能力层
        QR[查询改写<br/>≈400ms]
        RR[重排序]
        FS[高级过滤<br/>AND/OR]
        CTX[上下文关系链]
    end

    subgraph 检索层
        VEC[向量相似度]
        SEM[语义搜索]
        HYB[混合搜索]
    end

    V3_API --> VEC
    V3_API --> QR
    V3_API --> RR
    V3_API --> FS

    V4_API --> SEM
    V4_API --> QR
    V4_API --> RR
    V4_API --> FS
    V4_API --> CTX

    MCP_S --> HYB
    AI_SDK --> VEC

    style QR fill:#ffd93d,color:#333
    style RR fill:#6bcb77,color:#fff
    style FS fill:#9b59b6,color:#fff
    style CTX fill:#4a9eff,color:#fff
```
