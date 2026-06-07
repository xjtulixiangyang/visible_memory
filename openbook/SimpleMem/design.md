# SimpleMem 系统设计文档 (Design)

## 1. 系统架构总览

SimpleMem 采用**分层路由架构**，统一入口通过自动路由机制分发到不同的后端实现。

```mermaid
graph TB
    subgraph "用户接口层"
        User["用户代码"]
        MCP["MCP Client"]
        API["HTTP API"]
    end

    subgraph "路由层"
        Router["AutoMemory<br/>(自动路由)"]
        Config["Config<br/>(统一配置)"]
    end

    subgraph "后端引擎层"
        Text["SimpleMem Text<br/>(文本记忆)"]
        Omni["Omni-SimpleMem<br/>(多模态记忆)"]
        Evolver["EvolveMem<br/>(自演化)"]
        Cross["Cross-Session<br/>(跨会话)"]
    end

    subgraph "存储层"
        LanceDB["LanceDB<br/>(向量+FTS)"]
        SQLite["SQLite<br/>(关系数据)"]
        ColdStorage["Cold Storage<br/>(原始数据)"]
        FAISS["FAISS<br/>(稠密检索)"]
    end

    subgraph "外部依赖"
        LLM["LLM API<br/>(OpenAI兼容)"]
        Embedding["Embedding Model<br/>(Qwen3/OpenAI)"]
    end

    User --> Router
    MCP --> API --> Router
    Router -->|add_dialogue/ask| Text
    Router -->|add_text/image/audio/video/query| Omni
    Router -->|optimize| Evolver
    User -->|跨会话| Cross

    Text --> LanceDB
    Text --> LLM
    Text --> Embedding

    Omni --> LanceDB
    Omni --> FAISS
    Omni --> ColdStorage
    Omni --> LLM
    Omni --> Embedding

    Evolver --> Text
    Evolver --> LLM

    Cross --> SQLite
    Cross --> LanceDB

    Config --> Router
    Config --> Text
    Config --> Omni
    Config --> Evolver
```

## 2. 核心设计原则

### 2.1 语义无损压缩 (Semantic Lossless Compression)

系统不丢弃任何语义信息，而是通过结构化重述将非结构化交互转换为紧凑的、自包含的记忆单元：

- **代词消解 (Φ_coref)**: 禁止使用代词，所有指代必须显式化
- **时间绝对化 (Φ_time)**: 相对时间转换为 ISO 8601 绝对时间
- **完整主语**: 每条记忆必须包含完整的主语、宾语、时间、地点

### 2.2 多视图索引 (Multi-View Indexing)

每条记忆通过三个互补的索引层进行检索：

```mermaid
graph LR
    Memory["记忆单元 m_k"] --> Semantic["语义层 s_k<br/>E_dense(m_k)<br/>稠密向量相似度"]
    Memory --> Lexical["词汇层 l_k<br/>E_sparse(m_k)<br/>BM25 关键词匹配"]
    Memory --> Symbolic["符号层 r_k<br/>E_sym(m_k)<br/>元数据过滤"]

    Semantic --> Merge["结果合并<br/>C_q = R_sem ∪ R_lex ∪ R_sym"]
    Lexical --> Merge
    Symbolic --> Merge
```

### 2.3 写时合并 (Online Semantic Synthesis)

记忆在写入时即进行冗余合并，而非在查询时去重：

- 滑动窗口处理确保信息完整覆盖
- 前一窗口的记忆条目作为去重上下文
- 生成足够多的记忆条目确保所有信息被捕获

### 2.4 意图感知检索 (Intent-Aware Retrieval Planning)

检索不是简单的向量搜索，而是多步规划过程：

```mermaid
sequenceDiagram
    participant Q as 用户查询
    participant P as 检索规划器
    participant S as 语义搜索
    participant K as 关键词搜索
    participant M as 元数据搜索
    participant R as 反思引擎
    participant A as 答案生成器

    Q->>P: 分析信息需求
    P->>P: 生成定向子查询
    loop 每个子查询
        P->>S: 语义检索
        P->>K: BM25 检索
        P->>M: 结构化检索
    end
    P->>P: 合并去重
    P->>R: 信息充分性检查
    alt 信息不足
        R->>P: 生成补充查询
        P->>S: 补充检索
    end
    P->>A: 传递上下文
    A->>Q: 生成答案
```

## 3. 模块间交互设计

### 3.1 自动路由机制

```mermaid
stateDiagram-v2
    [*] --> Auto: SimpleMem()
    Auto --> Text: add_dialogue()
    Auto --> Omni: add_text/image/audio/video()
    Text --> Text: finalize()/ask()
    Omni --> Omni: query()
    Text --> [*]: close()
    Omni --> [*]: close()

    note right of Auto: 首次调用决定后端
    note right of Text: 文本模式锁定
    note right of Omni: 多模态模式锁定
```

路由逻辑基于**首次调用方法**自动选择后端：
- `add_dialogue()` → Text 后端
- `add_text()` / `add_image()` / `add_audio()` / `add_video()` → Omni 后端
- 一旦选定，生命周期内不可切换

### 3.2 文本记忆三阶段流水线

```mermaid
flowchart TB
    subgraph "Stage 1: 语义结构化压缩"
        D1["对话输入"] --> SW["滑动窗口<br/>(window_size, overlap)"]
        SW --> GATE["语义密度门控<br/>Φ_gate(W) → {m_k}"]
        GATE --> ME["MemoryEntry<br/>(lossless_restatement,<br/>keywords, timestamp,<br/>persons, entities, topic)"]
    end

    subgraph "Stage 2: 在线语义合成"
        ME --> CTX["前一窗口上下文"]
        CTX --> DEDUP["去重合并"]
        DEDUP --> STORE["写入 VectorStore"]
    end

    subgraph "Stage 3: 意图感知检索规划"
        Q["用户查询"] --> PLAN["P(q, H) → {q_sem, q_lex, q_sym}"]
        PLAN --> SEM["R_sem = Top-n(cos(E(q), E(m)))"]
        PLAN --> LEX["R_lex = Top-n(BM25(q, m))"]
        PLAN --> SYM["R_sym = Top-n(Meta(m) ⊨ q)"]
        SEM --> MERGE["C_q = R_sem ∪ R_lex ∪ R_sym"]
        LEX --> MERGE
        SYM --> MERGE
        MERGE --> REFLECT["反思检索<br/>(可选)"]
        REFLECT --> ANS["答案生成"]
    end

    STORE --> SEM
    STORE --> LEX
    STORE --> SYM
```

### 3.3 多模态记忆架构

```mermaid
flowchart TB
    subgraph "摄入层 (Ingestion)"
        TXT["TextProcessor"] --> MAU1["MAU (TEXT)"]
        IMG["ImageProcessor"] --> MAU2["MAU (VISUAL)"]
        AUD["AudioProcessor"] --> MAU3["MAU (AUDIO)"]
        VID["VideoProcessor"] --> MAU4["MAU (VIDEO)"]
    end

    subgraph "存储层 (Storage)"
        MAU1 --> MAUStore["MAUStore<br/>(内存索引)"]
        MAU2 --> MAUStore
        MAU3 --> MAUStore
        MAU4 --> MAUStore

        MAU1 --> VS["HybridVectorStore<br/>(FAISS)"]
        MAU2 --> VS
        MAU3 --> VS
        MAU4 --> VS

        MAU2 --> COLD["ColdStorage<br/>(原始数据)"]
        MAU3 --> COLD
        MAU4 --> COLD
    end

    subgraph "检索层 (Retrieval)"
        Q["查询"] --> QP["QueryProcessor"]
        QP --> PR["PyramidRetriever<br/>(预览→展开)"]
        PR --> BM25["BM25Store<br/>(关键词补充)"]
        PR --> KG["KnowledgeGraph<br/>(多跳推理)"]
        PR --> PM["ParametricStore<br/>(参数化知识)"]
    end

    MAUStore --> PR
    VS --> PR
    COLD --> PR
```

### 3.4 自演化闭环

```mermaid
sequenceDiagram
    participant Data as 对话数据
    participant Ext as MemoryExtractor
    participant Idx as MultiViewIndex
    participant Ret as MultiRetriever
    participant Ans as AnswerGenerator
    participant Eval as Evaluator
    participant Diag as DiagnosisEngine
    participant Meta as MetaAnalyzer

    loop 每轮演化 (max_rounds)
        Data->>Ext: 提取记忆
        Ext->>Idx: 构建多视图索引
        Idx->>Ret: 检索相关记忆
        Ret->>Ans: 生成答案
        Ans->>Eval: 评分 (token F1)
        Eval->>Diag: 失败分析
        Diag->>Diag: 根因分类<br/>(提取缺失/检索遗漏/答案错误)
        Diag->>Diag: 配置建议<br/>(最多2个字段变更)
        Diag->>Meta: 元分析
        Meta->>Meta: 决策: 继续/回退/聚焦/探索
        Meta->>Idx: 应用配置变更
        Note over Eval,Meta: 精英策略: 仅接受F1提升的变更<br/>拒绝则回退到最佳配置
    end
```

## 4. 存储架构

### 4.1 文本记忆存储

```mermaid
graph TB
    subgraph "LanceDB Table Schema"
        T["memory_table"]
        T --> F1["entry_id: string"]
        T --> F2["lossless_restatement: string"]
        T --> F3["keywords: list<string>"]
        T --> F4["timestamp: string"]
        T --> F5["location: string"]
        T --> F6["persons: list<string>"]
        T --> F7["entities: list<string>"]
        T --> F8["topic: string"]
        T --> F9["vector: list<float32>"]
    end

    subgraph "索引"
        IDX1["向量索引<br/>(语义搜索)"]
        IDX2["Tantivy FTS<br/>(关键词搜索)"]
        IDX3["SQL Filter<br/>(结构化搜索)"]
    end

    F9 --> IDX1
    F2 --> IDX2
    F4 & F5 & F6 & F7 --> IDX3
```

### 4.2 多模态存储

```mermaid
graph LR
    subgraph "热存储 (RAM)"
        MAU["MAUStore<br/>summary + embedding<br/>+ metadata"]
        VEC["HybridVectorStore<br/>text_dim + visual_dim<br/>FAISS 索引"]
    end

    subgraph "冷存储 (Disk)"
        COLD["ColdStorage<br/>原始图像/音频/视频<br/>raw_pointer 引用"]
    end

    subgraph "辅助索引"
        BM25["BM25Store<br/>关键词倒排索引"]
        KG["KnowledgeGraph<br/>实体-关系图"]
        EVENT["EventStore<br/>事件时序索引"]
    end

    MAU --> VEC
    MAU -.->|raw_pointer| COLD
    MAU --> BM25
    MAU --> KG
    MAU --> EVENT
```

### 4.3 跨会话存储

```mermaid
graph TB
    subgraph "SQLite"
        Sessions["sessions 表"]
        Events["events 表"]
        Observations["observations 表"]
        Summaries["summaries 表"]
    end

    subgraph "LanceDB"
        CrossVS["cross_session_vectors<br/>(语义搜索)"]
    end

    Sessions --> Events --> Observations
    Sessions --> Summaries
    Observations --> CrossVS
    Summaries --> CrossVS
```

## 5. 关键算法设计

### 5.1 滑动窗口与语义密度门控

```
输入: 对话序列 D = {d_1, d_2, ..., d_n}, 窗口大小 W, 重叠 O
输出: 记忆条目集合 {m_k}

step_size = W - O
for pos = 0; pos + W <= n; pos += step_size:
    window = D[pos : pos + W]
    entries = LLM_extract(window, context=previous_entries)
    VectorStore.add(entries)
    previous_entries = entries
```

### 5.2 混合检索融合

```
输入: 查询 q, 配置 config
输出: 合并上下文 C_q

1. 分析查询: analysis = LLM_analyze(q) → {keywords, persons, time, location, entities}
2. 语义检索: R_sem = VectorStore.semantic_search(q, top_k=config.k_sem)
3. 关键词检索: R_lex = VectorStore.keyword_search(analysis.keywords, top_k=config.k_kw)
4. 结构化检索: R_sym = VectorStore.structured_search(persons, time_range, location, entities)
5. 融合合并:
   - first_found: 按 structured > semantic > keyword 优先级去重
   - rrf: Reciprocal Rank Fusion
   - weighted_sum: 加权分数合并
6. (可选) 反思: 检查信息充分性, 补充检索
```

### 5.3 精英演化策略

```
输入: 初始配置 C_0, QA 对, 最大轮数 R
输出: 最优配置 C*

best_f1 = -∞, best_config = C_0
for round = 0 to R:
    memories = extract(sessions)
    index = build_index(memories)
    qa_results = evaluate(index, qa_pairs, C_current)
    f1 = mean(r.f1 for r in qa_results)

    if round > 0:
        delta = f1 - best_f1
        if delta > acceptance_threshold:
            best_f1 = f1; best_config = C_current  # 接受
        else:
            C_current = best_config  # 回退

    report = diagnose(qa_results, memories, C_current)
    C_current = adjust(report, C_current, max_changes=2)
```

## 6. 配置体系

### 6.1 统一配置 (Config)

```python
@dataclass
class Config:
    # 检索通道 top-k
    k_sem: int = 0
    k_kw: int = 5
    k_str: int = 0
    # 上下文预算
    context_budget: int = 8
    # 融合模式
    fusion_mode: str = "sum"  # sum | weighted | rrf
    fusion_weights: Dict[str, float] = {"semantic": 1.0, "keyword": 1.0, "structured": 1.0}
    # 答案风格
    answer_style: str = "concise"
    # 增强能力
    enable_entity_swap: bool = False
    enable_query_decomposition: bool = False
    enable_answer_verification: bool = False
    # 分类覆盖
    category_overrides: Dict[str, Any] = {}
    # 演化元数据
    evolved: bool = False
    evolution_rounds: int = 0
```

### 6.2 配置流转

```mermaid
flowchart LR
    Default["默认配置<br/>(settings.py)"] --> Runtime["运行时覆盖<br/>(构造参数)"]
    Runtime --> Evolved["演化配置<br/>(optimize())"]
    Evolved --> Saved["持久化<br/>(JSON 文件)"]
    Saved --> Deployed["部署加载<br/>(load_config())"]
```

## 7. 部署架构

```mermaid
graph TB
    subgraph "Python 包 (pip install)"
        Pkg["simplemem 包"]
        Pkg --> TextMod["text 模块"]
        Pkg --> OmniMod["multimodal 模块"]
        Pkg --> EvolMod["evolver 模块"]
    end

    subgraph "MCP 服务"
        MCPServer["HTTP Server<br/>(FastAPI)"]
        MCPServer --> MCPHandler["MCP Handler<br/>(JSON-RPC 2.0)"]
        MCPServer --> Auth["Token Auth<br/>(多租户)"]
        MCPServer --> Retriever["Hybrid Retriever"]
    end

    subgraph "Docker 部署"
        Docker["Docker Compose"]
        Docker --> WebUI["Web UI :8000"]
        Docker --> RESTAPI["REST API :8000/api"]
        Docker --> MCPSSE["MCP SSE :8000/mcp/sse"]
        Docker --> Volume["数据卷 ./data"]
    end

    Pkg -.->|SDK 调用| MCPServer
```

## 8. 错误处理策略

| 层级 | 策略 |
|:--|:--|
| LLM 调用 | 3 次重试，JSON 解析失败自动重试 |
| 并行处理 | 异常自动降级为串行处理 |
| 向量搜索 | 空结果返回空列表，不抛异常 |
| 演化引擎 | 精英策略拒绝回退，连续拒绝触发早停 |
| MCP 服务 | Token 认证失败返回 401，数据隔离异常不泄露 |

## 9. 扩展点

| 扩展点 | 机制 | 示例 |
|:--|:--|:--|
| 新后端 | `register(mode, module_path, class_name)` | 自定义记忆后端 |
| 新基准 | `BenchmarkAdapter` 子类 | LoCoMo / MemBench / LongMemEval |
| 新模态 | `Processor` 子类 + `ModalityType` 扩展 | 未来传感器数据 |
| 新检索策略 | `RetrievalConfig` 新字段 + 诊断规则 | MMR / KG 扩展 |
| 新食谱 | `evolution_cookbook.py` 添加条目 | 症状匹配的预编译配置组合 |
