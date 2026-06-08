# Store 存储模块设计文档

## 1. 模块概述

Store 模块是 TencentDB Agent Memory 的**存储抽象层**，通过 `IMemoryStore` 接口实现存储后端的可插拔替换。上层模块（hooks、tools、pipeline、record）仅依赖接口，不依赖具体实现，遵循**依赖倒置原则**。

### 核心设计原则

| 原则 | 说明 |
|------|------|
| **后端无关** | 上层仅依赖 `IMemoryStore` 接口，从不直接引用 `VectorStore` 或 `TcvdbMemoryStore` |
| **能力驱动** | 通过 `StoreCapabilities` 标志位表达后端能力，调用方据此选择搜索策略并优雅降级 |
| **容错优先** | 所有方法失败时返回空结果或 `false`，而非抛出异常（除非显式文档说明） |
| **同步/异步统一** | `MaybePromise<T>` 类型统一 SQLite 同步与 TCVDB 异步返回，调用方始终 `await` |

### 支持的后端

| 后端 | 类 | 存储 | 嵌入方式 | 搜索能力 |
|------|-----|------|----------|----------|
| SQLite | `VectorStore` | 本地文件 | 客户端嵌入 | 向量搜索 + FTS5 |
| TCVDB | `TcvdbMemoryStore` | 腾讯云向量数据库 | 服务端嵌入 | 向量 + 稀疏向量 + 原生混合搜索 |

### 文件结构

```
src/core/store/
├── types.ts          # 核心接口与类型定义
├── factory.ts        # 存储后端工厂（StoreBundle 组装）
├── sqlite.ts         # SQLite 本地向量存储实现
├── tcvdb.ts          # 腾讯云向量数据库实现
├── tcvdb-client.ts   # TCVDB HTTP 客户端
├── embedding.ts      # 嵌入服务（OpenAI / Local / ZeroEntropy / Noop）
├── bm25-local.ts     # 本地 BM25 稀疏向量编码器
├── bm25-client.ts    # BM25 远程客户端（旧版）
└── search-utils.ts   # 搜索工具（RRF 融合算法）
```

---

## 2. 架构设计

```mermaid
graph TB
    subgraph 上层模块
        Hooks[Hooks 钩子层]
        Tools[Tools 工具层]
        Pipeline[Pipeline 管道层]
        Record[Record 记录层]
    end

    subgraph Store 存储抽象层
        IMS[IMemoryStore 接口]
        SC[StoreCapabilities 能力标志]
        Factory[StoreBundle 工厂]
    end

    subgraph 后端实现
        SQLite[VectorStore<br/>SQLite + sqlite-vec + FTS5]
        TCVDB[TcvdbMemoryStore<br/>腾讯云向量数据库]
    end

    subgraph 辅助服务
        Embed[EmbeddingService<br/>嵌入服务]
        BM25[BM25LocalEncoder<br/>BM25 编码器]
        SearchUtils[search-utils<br/>RRF 融合]
    end

    subgraph 存储引擎
        SQLiteDB[(SQLite DB<br/>vectors.db)]
        TCVDBAPI[TCVDB HTTP API<br/>腾讯云向量数据库]
    end

    Hooks --> IMS
    Tools --> IMS
    Pipeline --> IMS
    Record --> IMS

    IMS -.-> SQLite
    IMS -.-> TCVDB

    Factory --> IMS
    Factory --> Embed
    Factory --> BM25

    SQLite --> SQLiteDB
    SQLite --> SearchUtils
    TCVDB --> TCVDBAPI
    TCVDB --> BM25

    SQLite --> Embed
```

---

## 3. IMemoryStore 接口设计

```mermaid
classDiagram
    class IMemoryStore {
        <<interface>>
        +supportsDeferredEmbedding: boolean

        +init(providerInfo?) StoreInitResult
        +isDegraded() boolean
        +getCapabilities() StoreCapabilities
        +close() void

        +upsertL1(record, embedding?) MaybePromise~boolean~
        +deleteL1(recordId) MaybePromise~boolean~
        +deleteL1Batch(recordIds) MaybePromise~boolean~
        +deleteL1Expired(cutoffIso) MaybePromise~number~
        +countL1() MaybePromise~number~
        +queryL1Records(filter?) MaybePromise~L1RecordRow[]
        +getAllL1Texts() MaybePromise~Array~

        +searchL1Vector(queryEmbedding, topK?, queryText?) MaybePromise~L1SearchResult[]~
        +searchL1Fts(ftsQuery, limit?) MaybePromise~L1FtsResult[]~
        +searchL1Hybrid(params) MaybePromise~L1SearchResult[]~

        +upsertL0(record, embedding?) MaybePromise~boolean~
        +updateL0Embedding(recordId, embedding) MaybePromise~boolean~
        +deleteL0(recordId) MaybePromise~boolean~
        +deleteL0Expired(cutoffIso) MaybePromise~number~
        +countL0() MaybePromise~number~
        +queryL0ForL1(sessionKey, afterMs?, limit?) MaybePromise~L0QueryRow[]~
        +queryL0GroupedBySessionId(sessionKey, afterMs?, limit?) MaybePromise~L0SessionGroup[]~
        +getAllL0Texts() MaybePromise~Array~

        +searchL0Vector(queryEmbedding, topK?) MaybePromise~L0SearchResult[]~
        +searchL0Fts(ftsQuery, limit?) MaybePromise~L0FtsResult[]~

        +pullProfiles() Promise~ProfileRecord[]~
        +syncProfiles(records) Promise~void~
        +deleteProfiles(recordIds) Promise~void~

        +reindexAll(embedFn, onProgress?) Promise~ReindexResult~
        +isFtsAvailable() boolean
    }

    class StoreCapabilities {
        <<interface>>
        +vectorSearch: boolean
        +ftsSearch: boolean
        +nativeHybridSearch: boolean
        +sparseVectors: boolean
    }

    class StoreInitResult {
        <<interface>>
        +needsReindex: boolean
        +reason?: string
    }

    class MaybePromise~T~ {
        T | Promise~T~
    }

    IMemoryStore --> StoreCapabilities : getCapabilities()
    IMemoryStore --> StoreInitResult : init()
```

### 接口分组说明

| 分组 | 方法 | 说明 |
|------|------|------|
| **生命周期** | `init` / `isDegraded` / `getCapabilities` / `close` | 初始化、状态查询、关闭 |
| **L1 写入** | `upsertL1` / `deleteL1` / `deleteL1Batch` / `deleteL1Expired` | 结构化记忆的增删改 |
| **L1 读取** | `countL1` / `queryL1Records` / `getAllL1Texts` | 记录查询与统计 |
| **L1 搜索** | `searchL1Vector` / `searchL1Fts` / `searchL1Hybrid` | 向量/关键词/混合搜索 |
| **L0 写入** | `upsertL0` / `updateL0Embedding` / `deleteL0` / `deleteL0Expired` | 原始对话的增删改 |
| **L0 读取** | `countL0` / `queryL0ForL1` / `queryL0GroupedBySessionId` / `getAllL0Texts` | 对话查询与分组 |
| **L0 搜索** | `searchL0Vector` / `searchL0Fts` | 向量/关键词搜索 |
| **Profile 同步** | `pullProfiles` / `syncProfiles` / `deleteProfiles` | L2/L3 画像同步（仅 TCVDB） |
| **重建索引** | `reindexAll` / `isFtsAvailable` | 全量重建与 FTS 状态 |

### 能力标志对比

| 能力 | SQLite | TCVDB |
|------|--------|-------|
| `vectorSearch` | ✅（vec0 表就绪时） | ✅（服务端嵌入） |
| `ftsSearch` | ✅（FTS5 可用时） | ✅（BM25 编码器可用时） |
| `nativeHybridSearch` | ❌ | ✅（BM25 编码器可用时） |
| `sparseVectors` | ❌ | ✅（BM25 编码器可用时） |

---

## 4. SQLite 后端设计

### 4.1 表结构设计

```mermaid
erDiagram
    l1_records {
        TEXT record_id PK
        TEXT content
        TEXT type
        INTEGER priority
        TEXT scene_name
        TEXT session_key
        TEXT session_id
        TEXT timestamp_str
        TEXT timestamp_start
        TEXT timestamp_end
        TEXT created_time
        TEXT updated_time
        TEXT metadata_json
    }

    l1_vec {
        TEXT record_id PK
        BLOB embedding
        TEXT updated_time
    }

    l1_fts {
        TEXT content "分词后文本(索引用)"
        TEXT content_original UNINDEXED "原始文本(展示用)"
        TEXT record_id UNINDEXED
        TEXT type UNINDEXED
        INTEGER priority UNINDEXED
        TEXT scene_name UNINDEXED
        TEXT session_key UNINDEXED
        TEXT session_id UNINDEXED
        TEXT timestamp_str UNINDEXED
        TEXT timestamp_start UNINDEXED
        TEXT timestamp_end UNINDEXED
        TEXT metadata_json UNINDEXED
    }

    l0_conversations {
        TEXT record_id PK
        TEXT session_key
        TEXT session_id
        TEXT role
        TEXT message_text
        TEXT recorded_at
        INTEGER timestamp
    }

    l0_vec {
        TEXT record_id PK
        BLOB embedding
        TEXT recorded_at
    }

    l0_fts {
        TEXT message_text "分词后文本(索引用)"
        TEXT message_text_original UNINDEXED "原始文本(展示用)"
        TEXT record_id UNINDEXED
        TEXT session_key UNINDEXED
        TEXT session_id UNINDEXED
        TEXT role UNINDEXED
        TEXT recorded_at UNINDEXED
        INTEGER timestamp UNINDEXED
    }

    embedding_meta {
        TEXT key PK
        TEXT value
    }

    l1_records ||--o{ l1_vec : "record_id"
    l1_records ||--o{ l1_fts : "record_id"
    l0_conversations ||--o{ l0_vec : "record_id"
    l0_conversations ||--o{ l0_fts : "record_id"
```

### 4.2 写入流程

```mermaid
flowchart TD
    A[upsertL1 调用] --> B{degraded?}
    B -->|是| C[返回 false]
    B -->|否| D{embedding 有效?<br/>非零且非 undefined}

    D -->|否| E[仅写入元数据]
    D -->|是| F[写入向量]

    E --> G[BEGIN 事务]
    F --> G

    G --> H[INSERT OR REPLACE<br/>l1_records 元数据]

    H --> I{embedding 有效?}
    I -->|是| J[DELETE l1_vec<br/>WHERE record_id = ?]
    J --> K[INSERT l1_vec<br/>record_id, embedding, updated_time]
    I -->|否| L[跳过向量写入]

    K --> M{FTS5 可用?}
    L --> M

    M -->|是| N[DELETE l1_fts<br/>WHERE record_id = ?]
    N --> O[INSERT l1_fts<br/>分词后 content + 原始 content_original]
    M -->|否| P[跳过 FTS 写入]

    O --> Q[COMMIT]
    P --> Q

    Q --> R[返回 true]

    G -.->|异常| S[ROLLBACK]
    S --> T[记录警告, 返回 false]
```

### 4.3 向量搜索流程

```mermaid
flowchart TD
    A[searchL1Vector 调用] --> B{degraded 或<br/>vecTablesReady?}
    B -->|是| C[返回空数组]
    B -->|否| D[计算 retrieveCount<br/>= topK + 10<br/>零向量缓冲]

    D --> E[vec0 KNN 搜索<br/>WHERE embedding MATCH ?<br/>AND k = retrieveCount<br/>ORDER BY distance]

    E --> F{结果为空?}
    F -->|是| G[返回空数组]

    F -->|否| H[遍历结果行]
    H --> I{distance 为 null/NaN?}
    I -->|是| J[跳过零向量占位符]
    I -->|否| K[查询 l1_records 元数据]

    K --> L{元数据存在?}
    L -->|否| M[跳过孤儿向量]
    L -->|是| N[计算 score = 1.0 - distance]

    N --> O[组装搜索结果]

    O --> P[截断至 topK]
    P --> Q[返回结果数组]
```

### 4.4 FTS5 搜索流程

```mermaid
flowchart TD
    A[searchL1Fts 调用] --> B{degraded 或<br/>ftsAvailable?}
    B -->|是| C[返回空数组]
    B -->|否| D[使用预编译语句<br/>stmtL1FtsSearch]

    D --> E[MATCH ? 查询<br/>ORDER BY rank ASC<br/>LIMIT ?]

    E --> F[遍历结果行]
    F --> G[bm25RankToScore 转换<br/>负值: relevance / 1 + relevance<br/>正值: 1 / 1 + rank]

    G --> H[组装 FtsSearchResult<br/>score ∈ 0,1]

    H --> I[返回结果数组]
```

### 4.5 FTS5 中文分词策略

```mermaid
flowchart LR
    subgraph 写入侧 tokenizeForFts
        A[原始文本] --> B{jieba 可用?}
        B -->|是| C[jieba.cutForSearch<br/>搜索引擎模式分词]
        B -->|否| D[保留原文]
        C --> E[空格连接分词结果<br/>unicode61 二次切分]
        D --> E
    end

    subgraph 查询侧 buildFtsQuery
        F[查询文本] --> G{jieba 可用?}
        G -->|是| H[jieba.cutForSearch<br/>+ 停用词过滤<br/>+ 去重]
        G -->|否| I[Unicode 正则切分<br/>/\\p{L}\\p{N}_+/gu]
        H --> J[引号包裹 + OR 连接<br/>"词1" OR "词2" OR ...]
        I --> J
    end

    E --> K[l1_fts.content<br/>l0_fts.message_text]
    J --> L[MATCH 表达式]
```

### 4.6 重建索引流程

```mermaid
flowchart TD
    A[reindexAll 调用] --> B{degraded 或<br/>vecTablesReady?}
    B -->|是| C[返回 l1Count=0, l0Count=0]
    B -->|否| D[读取所有 L1 文本<br/>getAllL1Texts]

    D --> E[遍历 L1 记录]
    E --> F[embedFn 生成新嵌入]
    F --> G[BEGIN 事务]
    G --> H[DELETE l1_vec]
    H --> I[INSERT l1_vec]
    I --> J[COMMIT]
    J --> K[onProgress 回调]
    K --> E

    E -->|完成| L[读取所有 L0 文本<br/>getAllL0Texts]
    L --> M[遍历 L0 记录<br/>同上流程]
    M --> N[返回 l1Count, l0Count]
```

---

## 5. TCVDB 后端设计

### 5.1 初始化流程

```mermaid
flowchart TD
    A[init 调用] --> B[启动异步初始化<br/>_initAsync]
    B --> C[createDatabase<br/>幂等创建数据库]

    C --> D{数据库刚创建?}
    D -->|是| E[等待 5s<br/>数据库就绪延迟]
    D -->|否| F[继续]

    E --> F

    F --> G[创建 L1 Collection<br/>_createCollectionWithVectorFallback]
    G --> H[尝试 DISK_FLAT<br/>向量索引]
    H --> I{DISK_FLAT 支持?}
    I -->|是| J[使用 DISK_FLAT]
    I -->|否 apiCode=15113| K[回退 HNSW<br/>M=16, efConstruction=200]

    J --> L[创建 L0 Collection<br/>同上回退策略]
    K --> L

    L --> M[创建 Profiles Collection<br/>无向量嵌入<br/>FLAT 索引 dimension=1]

    M --> N[初始化完成]

    B -.->|异常| O[degraded = true]
    C -.->|15201 已存在| F
```

### 5.2 Collection 结构

```mermaid
graph TB
    subgraph L1 Collection
        L1_ID[id: string PK]
        L1_VEC[vector: float1024<br/>COSINE 距离]
        L1_SPARSE[sparse_vector: sparseVector<br/>IP 距离]
        L1_TEXT[text: string<br/>嵌入源字段]
        L1_FIELDS[type, priority, scene_name<br/>session_key, session_id<br/>timestamp_start/end<br/>metadata_json<br/>created/updated_time_ms]
    end

    subgraph L0 Collection
        L0_ID[id: string PK]
        L0_VEC[vector: float1024<br/>COSINE 距离]
        L0_SPARSE[sparse_vector: sparseVector<br/>IP 距离]
        L0_MSG[message_text: string<br/>嵌入源字段]
        L0_FIELDS[session_key, session_id<br/>role, agent_id<br/>recorded_at_ms, timestamp]
    end

    subgraph Profiles Collection
        P_ID[id: string PK]
        P_VEC[vector: float1<br/>占位 FLAT]
        P_FIELDS[type, filename, content<br/>content_md5, agent_id<br/>version<br/>created/updated_at_ms]
    end
```

### 5.3 混合搜索流程

```mermaid
flowchart TD
    A[searchL1HybridAsync 调用] --> B[_ensureInit]
    B --> C{degraded?}
    C -->|是| D[返回空数组]
    C -->|否| E{bm25Encoder 可用?}

    E -->|是| F[构建完整混合搜索]
    E -->|否| G[仅密集向量搜索]

    F --> H[ann: fieldName=text<br/>data=queryText<br/>服务端嵌入]
    F --> I[match: fieldName=sparse_vector<br/>data=BM25 编码结果]
    F --> J[rerank: method=rrf<br/>k=60]

    H --> K[hybridSearch API]
    I --> K
    J --> K

    G --> L[search API<br/>embeddingItems=queryText<br/>服务端嵌入]

    K --> M[解析搜索结果<br/>_parseL1SearchResults]
    L --> M

    M --> N[返回 L1SearchResult[]]
```

### 5.4 Profile 同步流程

```mermaid
flowchart TD
    A[syncProfiles 调用] --> B[_ensureInit]
    B --> C[查询远程所有 Profile<br/>_queryAllDocs]

    C --> D[遍历本地 records]
    D --> E{远程存在?}

    E -->|否| F[新增: version=1<br/>content_md5=当前MD5]
    E -->|是| G{MD5 相同?}

    G -->|是| H[跳过, 无变更]
    G -->|否| I{baselineVersion<br/>== 远程 version?}

    I -->|否| J[冲突! 跳过同步<br/>记录警告]
    I -->|是| K[更新: version+1<br/>乐观锁通过]

    F --> L[收集 upserts]
    K --> L

    L --> M{upserts 非空?}
    M -->|是| N[批量 upsert<br/>profilesCollection]
    M -->|否| O[无需同步]
```

---

## 6. 嵌入服务设计

```mermaid
classDiagram
    class EmbeddingService {
        <<interface>>
        +embed(text, options?) Promise~Float32Array~
        +embedBatch(texts, options?) Promise~Float32Array[]~
        +getDimensions() number
        +getProviderInfo() EmbeddingProviderInfo
        +isReady() boolean
        +startWarmup() void
        +close() void | Promise~void~
    }

    class OpenAIEmbeddingService {
        -baseUrl: string
        -apiKey: string
        -model: string
        -dims: number
        -sendDimensions: boolean
        -proxyUrl?: string
        -maxInputChars?: number
        -timeoutMs: number
        +embed(text, options?) Promise~Float32Array~
        +embedBatch(texts, options?) Promise~Float32Array[]~
        +getDimensions() number
        +getProviderInfo() EmbeddingProviderInfo
        +isReady() boolean  Always true
        +startWarmup() void  No-op
    }

    class LocalEmbeddingService {
        -modelPath: string
        -initState: LocalInitState
        -embeddingContext: object
        +embed(text, options?) Promise~Float32Array~
        +embedBatch(texts, options?) Promise~Float32Array[]~
        +getDimensions() number  768
        +getProviderInfo() EmbeddingProviderInfo
        +isReady() boolean
        +startWarmup() void  触发后台加载
        +close() void  释放资源
    }

    class ZeroEntropyEmbeddingService {
        -baseUrl: string
        -apiKey: string
        -model: string
        -dims: number
        +embed(text, options?) Promise~Float32Array~
        +embedBatch(texts, options?) Promise~Float32Array[]~
        +getDimensions() number
        +getProviderInfo() EmbeddingProviderInfo  provider=zeroentropy
        +isReady() boolean  Always true
        +startWarmup() void  No-op
    }

    class NoopEmbeddingService {
        +embed(text) Promise~Float32Array~  返回空数组
        +embedBatch(texts) Promise~Float32Array[]~  返回空数组
        +getDimensions() number  返回 0
        +getProviderInfo() EmbeddingProviderInfo  provider=noop
        +isReady() boolean  Always true
        +startWarmup() void  No-op
    }

    class EmbeddingProviderInfo {
        <<interface>>
        +provider: string
        +model: string
    }

    EmbeddingService <|.. OpenAIEmbeddingService
    EmbeddingService <|.. LocalEmbeddingService
    EmbeddingService <|.. ZeroEntropyEmbeddingService
    EmbeddingService <|.. NoopEmbeddingService

    EmbeddingService --> EmbeddingProviderInfo : getProviderInfo()
```

### 嵌入服务对比

| 特性 | OpenAI | Local | ZeroEntropy | Noop |
|------|--------|-------|-------------|------|
| **维度** | 可配置 | 768 | 可配置 | 0 |
| **模型** | 用户指定 | embeddinggemma-300m | zembed-1 | 无 |
| **就绪状态** | 始终就绪 | 需后台加载 | 始终就绪 | 始终就绪 |
| **预热** | 无需 | `startWarmup()` | 无需 | 无需 |
| **用途** | SQLite 后端 | SQLite 后端 | SQLite 后端 | TCVDB 后端 |
| **API 端点** | `/embeddings` | 本地推理 | `/models/embed` | 无 |
| **特殊参数** | `dimensions` | 无 | `input_type` | 无 |

### LocalEmbeddingService 状态机

```mermaid
stateDiagram-v2
    [*] --> idle: 构造函数
    idle --> initializing: startWarmup()
    initializing --> ready: 模型加载成功
    initializing --> failed: 模型加载失败
    failed --> initializing: startWarmup() 重试
    ready --> idle: close()
    failed --> idle: close()

    note right of ready: embed() 可用
    note right of initializing: embed() 抛出 EmbeddingNotReadyError
    note right of failed: embed() 抛出 EmbeddingNotReadyError
    note right of idle: embed() 抛出 EmbeddingNotReadyError
```

---

## 7. 搜索策略设计

### 7.1 三种搜索路径

```mermaid
flowchart TD
    A[搜索请求] --> B{搜索类型?}

    B -->|keyword| C[关键词搜索路径]
    B -->|embedding| D[向量搜索路径]
    B -->|hybrid| E[混合搜索路径]

    subgraph 关键词搜索
        C --> C1{后端类型?}
        C1 -->|SQLite| C2[buildFtsQuery<br/>jieba 分词 + OR 连接]
        C2 --> C3[FTS5 MATCH 搜索<br/>BM25 排序]
        C3 --> C4[bm25RankToScore<br/>归一化 0-1]

        C1 -->|TCVDB| C5[BM25 encodeQueries<br/>稀疏向量编码]
        C5 --> C6[hybridSearch<br/>sparse-only 路径]
    end

    subgraph 向量搜索
        D --> D1{后端类型?}
        D1 -->|SQLite| D2[客户端 embed<br/>生成 queryEmbedding]
        D2 --> D3[vec0 KNN 搜索<br/>cosine distance]
        D3 --> D4[score = 1.0 - distance<br/>过滤零向量]

        D1 -->|TCVDB| D5[服务端嵌入<br/>embeddingItems]
        D5 --> D6[/document/search<br/>服务端向量匹配]
    end

    subgraph 混合搜索
        E --> E1{后端类型?}
        E1 -->|SQLite| E2[并行: 向量搜索 + FTS 搜索]
        E2 --> E3[rrfMerge 融合<br/>k=60]
        E3 --> E4[按 RRF 分数排序]

        E1 -->|TCVDB| E5[ann: 服务端密集向量]
        E5 --> E6[match: BM25 稀疏向量]
        E6 --> E7[rerank: RRF k=60]
        E7 --> E8[/document/hybridSearch]
    end
```

### 7.2 RRF 融合算法

```mermaid
flowchart LR
    subgraph 输入
        L1[向量搜索结果<br/>按 score 排序]
        L2[FTS 搜索结果<br/>按 score 排序]
    end

    subgraph RRF 计算
        R1[遍历列表 L1<br/>rank=0,1,2...<br/>score += 1/k+rank+1]
        R2[遍历列表 L2<br/>rank=0,1,2...<br/>score += 1/k+rank+1]
        R3[相同 ID 的<br/>RRF 分数累加]
    end

    subgraph 输出
        O[按 RRF 分数降序<br/>合并去重结果]
    end

    L1 --> R1
    L2 --> R2
    R1 --> R3
    R2 --> R3
    R3 --> O
```

**RRF 公式**：`score(d) = Σ 1/(k + rank(d) + 1)`，其中 `k = 60`

---

## 8. 工厂模式设计

### 8.1 StoreBundle 组装流程

```mermaid
sequenceDiagram
    participant Caller as 调用方
    participant Factory as createStoreBundle
    participant BM25 as createBM25Encoder
    participant Store as IMemoryStore
    participant Embed as EmbeddingService

    Caller->>Factory: createStoreBundle(config, options)

    Factory->>BM25: createBM25Encoder(config.bm25)
    BM25-->>Factory: bm25Encoder | undefined

    alt config.storeBackend === "tcvdb"
        Factory->>Factory: 校验 tcvdb.url + tcvdb.apiKey + tcvdb.database
        Factory->>Store: new TcvdbMemoryStore(tcvdbConfig + bm25Encoder)
        Factory->>Embed: new NoopEmbeddingService()
        Factory-->>Caller: {store, embedding: Noop, bm25Encoder, storeSnapshot: tcvdb}
    else config.storeBackend === "sqlite" (默认)
        alt embedding.enabled && provider !== "local" && apiKey 存在
            Factory->>Embed: createEmbeddingService(embeddingConfig)
            Note over Embed: OpenAI / ZeroEntropy
        else
            Factory->>Embed: undefined
            Note over Embed: 延迟嵌入 / 本地嵌入
        end
        Factory->>Store: new VectorStore(dbPath, dimensions)
        Factory-->>Caller: {store, embedding, bm25Encoder, storeSnapshot: sqlite}
    end
```

### 8.2 StoreBundle 结构

```mermaid
classDiagram
    class StoreBundle {
        +store: IMemoryStore
        +embedding: IEmbeddingService
        +bm25Encoder?: BM25LocalEncoder
        +storeSnapshot: StoreConfigSnapshot
    }

    class StoreConfigSnapshot {
        +type: "sqlite" | "tcvdb"
        +sqlitePath?: string
        +tcvdbUrl?: string
        +tcvdbDatabase?: string
        +tcvdbAlias?: string
    }

    StoreBundle --> StoreConfigSnapshot : storeSnapshot
```

---

## 9. 容错与降级设计

### 9.1 降级策略总览

```mermaid
flowchart TD
    A[操作调用] --> B{degraded 模式?}
    B -->|是| C[安全空操作<br/>返回 false / 空数组 / 0]
    B -->|否| D[正常执行]

    D --> E{执行成功?}
    E -->|是| F[返回正常结果]
    E -->|否| G[捕获异常]

    G --> H[记录警告日志]
    H --> I[返回安全默认值<br/>false / 空数组 / 0]

    style C fill:#ffcccc
    style F fill:#ccffcc
    style I fill:#fff3cc
```

### 9.2 各组件降级行为

| 组件 | 触发条件 | 降级行为 |
|------|----------|----------|
| **VectorStore** | sqlite-vec 加载失败 | `degraded = true`，所有操作返回空/false |
| **VectorStore** | FTS5 不可用 | `ftsAvailable = false`，FTS 搜索返回空数组 |
| **VectorStore** | dimensions = 0 | `vecTablesReady = false`，跳过向量读写 |
| **TcvdbMemoryStore** | 初始化失败 | `degraded = true`，所有操作返回空/false |
| **TcvdbMemoryStore** | BM25 不可用 | `nativeHybridSearch = false`，回退到纯密集搜索 |
| **TcvdbMemoryStore** | DISK_FLAT 不支持 | 自动回退到 HNSW 索引 |
| **LocalEmbeddingService** | 模型加载失败 | `initState = "failed"`，embed() 抛出 EmbeddingNotReadyError |
| **OpenAIEmbeddingService** | API 调用失败 | 抛出异常，由调用方决定降级 |
| **BM25LocalEncoder** | 编码失败 | 返回空数组，TCVDB 跳过稀疏向量 |

### 9.3 删除保护机制

```mermaid
flowchart TD
    A[deleteExpired 调用] --> B[计算过期数量 expiredCount]
    B --> C[计算总量 total]
    C --> D[计算比例 ratio = expiredCount / total]

    D --> E{ratio > 80%?}
    E -->|是| F[BLOCKED: 拒绝删除<br/>记录警告日志<br/>返回 0]
    E -->|否| G[正常执行删除<br/>事务保证一致性]

    G --> H[返回删除数量]
```

### 9.4 TCVDB 客户端重试机制

```mermaid
flowchart TD
    A[HTTP 请求] --> B[发送请求<br/>attempt = 0]
    B --> C{响应状态?}

    C -->|code = 0| D[成功返回]
    C -->|4xx 非 429| E[客户端错误<br/>立即抛出 TcvdbApiError]
    C -->|5xx / 超时 / 429| F{attempt < MAX_RETRIES?}

    F -->|是| G[指数退避等待<br/>delay = 500 * attempt+1<br/>ms]
    G --> H[attempt++<br/>重试请求]
    H --> C

    F -->|否| I[抛出最后一个错误]

    C -->|网络异常| F
```

**重试参数**：最大重试 2 次，退避间隔 500ms / 1000ms

### 9.5 SQLite 事务安全

```mermaid
flowchart TD
    A[BEGIN 事务] --> B[元数据写入]
    B --> C[向量写入]
    C --> D[FTS 写入]
    D --> E[COMMIT]

    A -.->|任意步骤异常| F[ROLLBACK]
    F --> G[记录警告]
    G --> H[返回 false]

    E --> I[返回 true]
```

**关键设计**：
- vec0 不支持 `ON CONFLICT`，upsert 采用 `DELETE + INSERT` 策略
- FTS 写入失败为非致命错误，不影响主事务
- 零向量不写入 vec0 表，避免 KNN 搜索返回 null/NaN 距离
- KNN 搜索额外获取 10 条结果（`ZERO_VEC_BUFFER`）补偿遗留零向量
