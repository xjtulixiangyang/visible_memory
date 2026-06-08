# MemForest 架构设计文档

## 1. 系统架构全景

```mermaid
graph TB
    subgraph 外部接口
        USER[用户/Agent]
    end

    subgraph 协调层
        MF[MemForest<br/>多用户协调器<br/>用户注册/并行操作/持久化]
    end

    subgraph 用户森林层
        UF[UserForest<br/>单用户记忆森林<br/>ingest/query/delete/save]
        REG[SessionRegistry<br/>会话/轮次/事实追踪]
        CS[CellStore<br/>MemCell 缓存]
        SC[SummaryCache<br/>内容哈希→摘要缓存]
    end

    subgraph 抽取管线
        NORM[TurnNormalizer<br/>对话标准化]
        CHK[Chunker<br/>确定性分块]
        EXT[ExtractionPipeline<br/>LLM 事实抽取]
        DEDUP[Deduplicator<br/>Embedding+LLM 去重]
        FM[FactManager<br/>FAISS 向量索引+JSONL 持久化]
    end

    subgraph 树构建管线
        ER[EntityRouter<br/>实体路由 lazy/active/suppressed]
        SCR[SceneRouter<br/>场景路由 在线聚类+域门控]
        TB[TreeBuilder<br/>三阶段构建编排]
        TREE[Tree Primitives<br/>B+树构建/插入/删除/分裂]
        SUMM[SummaryManager<br/>并行 LLM 摘要]
        NI[NodeIndex<br/>FAISS 全节点索引]
        RI[RootIndex<br/>FAISS 根摘要索引]
    end

    subgraph 查询管线
        RET[ForestRetriever<br/>TD+BU 双路召回]
        PLAN[BrowsePlanner<br/>LLM 查询分解]
        BROWSE[TreeBrowser<br/>Beam Search 浏览]
        RERANK[FactReranker<br/>Embedding 重排]
        CTX[ContextBuilder<br/>上下文渲染]
    end

    subgraph 基础设施
        API_CHAT[OpenAIChatClient]
        API_EMB[OpenAIEmbeddingClient]
        STORE[TreeStore<br/>JSON 持久化]
        CFG[Config<br/>YAML+Dataclass]
        LOG[Logger<br/>API+Extraction 日志]
        PRMPT[PromptManager<br/>模板渲染]
    end

    USER --> MF
    MF --> UF
    UF --> REG
    UF --> CS
    UF --> SC
    UF --> FM
    UF --> TB
    UF --> RET

    FM --> DEDUP
    FM --> EXT
    EXT --> CHK
    CHK --> NORM

    TB --> ER
    TB --> SCR
    TB --> TREE
    TB --> SUMM
    TB --> NI
    TB --> RI

    RET --> PLAN
    PLAN --> BROWSE
    BROWSE --> RERANK
    RERANK --> CTX

    UF --> API_CHAT
    UF --> API_EMB
    TB --> STORE
    UF --> CFG
    UF --> LOG
    EXT --> PRMPT
    SUMM --> PRMPT
```

---

## 2. 写入路径设计（Ingest Pipeline）

### 2.1 完整写入流程

```mermaid
flowchart TD
    INPUT["输入: session_id + turns"] --> NORM["normalize_turns<br/>字段归一化 + 时间解析"]

    NORM --> CHK["chunk_session<br/>按 max_turns/max_chars/time_gap/hard_boundary 切分"]
    CHK --> CELLS["MemCell 列表"]

    CELLS --> SCHED["ExtractionManager<br/>max_inflight 并发调度"]
    SCHED --> EXT["ChunkExtractionPipeline._extract_cell_impl<br/>渲染 Prompt → 调用 LLM → 解析 JSON"]
    EXT --> ITEMS["MemoryItem 列表"]

    ITEMS --> AGG["聚合为 SessionExtractionResult"]
    AGG --> FM_ADD["FactManager.add_session_result"]

    FM_ADD --> TIME_FB["_apply_cell_time_fallbacks<br/>时间信息回退"]
    TIME_FB --> DEDUP_FLOW["add_memory_items 三阶段去重"]

    subgraph 三阶段去重
        DEDUP_FLOW --> PA["Phase A: 候选收集<br/>精确匹配 + FAISS 近邻 + 批内配对"]
        PA --> PB["Phase B: LLM 判定<br/>LLMFactEquivalenceJudge<br/>并查集合并等价组"]
        PB --> PC["Phase C: 应用决策<br/>插入新事实 / 合并到已有事实"]
    end

    PC --> FACT_IDS["canonical_fact_ids"]
    FACT_IDS --> C2F["构建 cell_to_facts 映射"]

    C2F --> TB_ING["TreeBuilder.ingest_session"]

    subgraph 树构建三阶段
        TB_ING --> P1["Phase 1: Structure<br/>EntityRouter + SceneRouter 路由<br/>insert_fact / insert_cell"]
        P1 --> P2["Phase 2: Summarize<br/>L0 直通 + L1+ LLM 并行摘要"]
        P2 --> P3["Phase 3: Index<br/>embed_roots + build_node_index"]
    end

    P3 --> SAVE["TreeStore.save_tree + SessionRegistry.register"]
    SAVE --> REWIRE["_rewire_query_pipeline<br/>确保查询管线引用最新索引"]

    style 三阶段去重 fill:#fff9c4
    style 树构建三阶段 fill:#e8f5e9
```

### 2.2 事实去重策略

```mermaid
flowchart TD
    A[新 MemoryItem 列表] --> B{Phase A: 候选收集}

    B --> B1[_group_exact_duplicates<br/>批内精确归一化匹配]
    B --> B2[_find_candidate_pairs<br/>FAISS 向量搜索已有事实]
    B --> B3[批内 FAISS 相似度配对<br/>构建临时索引]

    B1 --> C{Phase B: LLM 判定}
    B2 --> C
    B3 --> C

    C --> C1[LLMFactEquivalenceJudge<br/>.are_equivalent_batch<br/>并发判定等价性]
    C1 --> C2[并查集合并等价组<br/>解析 intra-batch merges]

    C2 --> D{Phase C: 应用决策}

    D -->|精确匹配| E[_merge_items_into_fact<br/>合并到已有 canonical fact]
    D -->|Embedding+LLM 匹配| E
    D -->|批内等价合并| F[主组吸收副组<br/>构建新 ManagedFact]
    D -->|无匹配| G[插入新 canonical fact<br/>FAISS.add + exact_index 更新]

    E --> H[FactWriteResult]
    F --> H
    G --> H
```

---

## 3. 树构建设计（Tree Building）

### 3.1 三阶段构建管线

```mermaid
flowchart TD
    subgraph Phase1["Phase 1: Structure（无 LLM）"]
        F[ManagedFact 列表] --> SORT[按 time_start 排序]
        SORT --> ER[EntityRouter.assign<br/>按实体归一化键路由<br/>lazy/active/suppressed 生命周期]
        SORT --> SR_BOOT[SceneRouter.bootstrap<br/>最远优先采样初始化]
        SR_BOOT --> SR_ASSIGN[SceneRouter.assign_many<br/>域门控 + 质心相似度 + 双分配]
        SR_ASSIGN --> SUPPRESS[计算 suppressed 实体<br/>被场景完全包含的小实体]

        ER --> EB[entity_buckets]
        SR_ASSIGN --> SB[scene_buckets]
        CELLS[MemCell 列表] --> SESS_B[session buckets]

        EB --> BTF["build_tree_from_facts()<br/>entity/scene 树"]
        SB --> BTF
        SESS_B --> BTC["build_tree_from_cells()<br/>session 树"]
    end

    subgraph Phase2["Phase 2: Summarize（LLM 并行）"]
        BTF --> CDR[collect_dirty_requests]
        BTC --> CDR
        CDR --> L0["L0 直通: fact_text/raw_text → summary"]
        L0 --> FILL[fill_upper_level_inputs]
        FILL --> LLM["SummaryManager.generate_summaries<br/>ThreadPoolExecutor 并行"]
        LLM --> HEADER["_prepend_node_headers<br/>日期范围头"]
    end

    subgraph Phase3["Phase 3: Index（嵌入）"]
        HEADER --> EMB["embed_roots → tree.centroid"]
        EMB --> NI["build_node_index → NodeIndex"]
        EMB --> RI["RootIndex.add_tree"]
    end

    style Phase1 fill:#e8f5e9
    style Phase2 fill:#fff3e0
    style Phase3 fill:#e3f2fd
```

### 3.2 MemTree 内部结构

```mermaid
graph TD
    ROOT["Root Node<br/>level=3<br/>summary: 全局摘要"] --> A["Internal Node<br/>level=2<br/>summary: 区间摘要"]
    ROOT --> B["Internal Node<br/>level=2"]

    A --> C["Internal Node<br/>level=1"]
    A --> D["Internal Node<br/>level=1"]
    B --> E["Internal Node<br/>level=1"]
    B --> F["Internal Node<br/>level=1"]

    C --> L1["L0 Leaf<br/>child_ids=[fact_1]"]
    C --> L2["L0 Leaf<br/>child_ids=[fact_2]"]
    C --> L3["L0 Leaf<br/>child_ids=[fact_3]"]
    D --> L4["L0 Leaf"]
    D --> L5["L0 Leaf"]
    D --> L6["L0 Leaf"]

    E --> L7["L0 Leaf"]
    E --> L8["L0 Leaf"]
    F --> L9["L0 Leaf"]
    F --> L10["L0 Leaf"]

    L1 -.->|prev_leaf| L2
    L2 -.->|prev_leaf| L3
    L3 -.->|prev_leaf| L4

    style ROOT fill:#e1bee7
    style A fill:#b39ddb
    style B fill:#b39ddb
    style C fill:#9fa8da
    style D fill:#9fa8da
    style E fill:#9fa8da
    style F fill:#9fa8da
    style L1 fill:#a5d6a7
    style L2 fill:#a5d6a7
    style L3 fill:#a5d6a7
    style L4 fill:#a5d6a7
    style L5 fill:#a5d6a7
    style L6 fill:#a5d6a7
    style L7 fill:#a5d6a7
    style L8 fill:#a5d6a7
    style L9 fill:#a5d6a7
    style L10 fill:#a5d6a7
```

### 3.3 场景路由器核心逻辑

```mermaid
flowchart TD
    A[新事实列表] --> BOOT{已有聚类?}
    BOOT -->|否| B[bootstrap<br/>最远优先采样<br/>初始化聚类原型]
    BOOT -->|是| C[assign_many]

    B --> C
    C --> D[域门控<br/>scope_key 匹配]
    D --> E[质心相似度排序]
    E --> F{最高相似度 ≥ theta_assign?}
    F -->|是| G[分配到主聚类]
    F -->|否| H[生成新聚类<br/>theta_spawn]

    G --> I{次高相似度 ≥ theta_second?}
    I -->|是| J[双分配到次聚类]
    I -->|否| K[仅主聚类]

    J --> L{需要合并检查?}
    K --> L
    H --> L

    L --> M{同域质心相似度 ≥ theta_merge?}
    M -->|是| N[自动合并聚类]
    M -->|否| O[保持独立]

    N --> P[可选: LLM relabel + judge_merge]
    O --> P
```

---

## 4. 查询路径设计（Query Pipeline）

### 4.1 五阶段查询流水线

```mermaid
flowchart TD
    Q["用户问题"] --> EMB["Embedding API<br/>question → query_emb"]

    EMB --> TD["TD 召回<br/>FAISS 根向量检索<br/>→ top-K tree_ids"]
    EMB --> BU["BU 召回<br/>Fact FAISS top-M<br/>→ 反投影到树"]
    TD --> UNION["Union 合并<br/>TD ∪ BU + force_include"]
    BU --> UNION
    UNION --> CARDS["list[TreeCard]"]

    CARDS --> PLAN["BrowsePlanner<br/>1 次 LLM<br/>问题分解 + 子查询生成"]
    Q --> PLAN

    PLAN --> BPS["list[BrowsePlan]<br/>每棵树一个定制子查询"]

    BPS --> BROWSE["TreeBrowser<br/>并行 Beam Search"]
    BROWSE --> RAW_IDS["raw_ids (fact_id/cell_id)"]

    RAW_IDS --> LOAD["FactLoader<br/>→ list[ManagedFact]"]
    LOAD --> RERANK["FactReranker<br/>cos_sim 重排"]
    RERANK --> CTX["build_context<br/>渲染上下文"]
    CTX --> ANSWER["LLM Answerer"]

    style Q fill:#e1f5fe
    style EMB fill:#b3e5fc
    style TD fill:#fff9c4
    style BU fill:#fff9c4
    style UNION fill:#fff9c4
    style PLAN fill:#c8e6c9
    style BROWSE fill:#f0f4c3
    style RERANK fill:#ffccbc
    style CTX fill:#f8bbd0
    style ANSWER fill:#e1bee7
```

### 4.2 Beam Search 内部流程

```mermaid
flowchart TD
    ROOT["Root Node"] --> EXPAND["展开所有 frontier 子节点"]
    EXPAND --> SCORE["全局打分<br/>embedding cos_sim<br/>或 LLM 排序"]
    SCORE --> TOPK["保留全局 top-beam"]
    TOPK --> CHECK{子节点 level?}
    CHECK -->|"level = 0"| L0["settled_l0<br/>叶子事实"]
    CHECK -->|"level > 0"| INTERNAL["next_frontier"]
    INTERNAL --> EXPAND
    L0 --> RESULT["提取 child_ids<br/>= fact_ids / cell_ids"]

    style ROOT fill:#e3f2fd
    style SCORE fill:#90caf9
    style L0 fill:#a5d6a7
    style RESULT fill:#ce93d8
```

### 4.3 两种运行模式对比

| 维度 | Lightweight | Agentic |
|------|------------|---------|
| Planner | 关闭 | 开启（1 次 LLM） |
| Beam Width | 3 | 10 |
| LLM Browse | 关闭 | 开启 |
| LLM 调用次数 | 1（仅 answer） | 2 + N_nodes |
| 目标场景 | 低延迟、低成本 | 高精度、复杂问题 |

---

## 5. 删除与维护设计

### 5.1 删除流程

```mermaid
flowchart TD
    A["delete_session / delete_turn"] --> B["SessionRegistry.delete_*<br/>标记 deleted=True"]
    B --> C["compute_orphaned_facts<br/>所有产生 cell 均已死亡的事实"]

    C --> D["_apply_local_deletions"]
    D --> E["1. 提取当前摘要缓存"]
    D --> F["2. delete_cell: 从 session 树移除"]
    D --> G["3. delete_facts: 从 FactManager 移除"]
    D --> H["4. delete_fact: 从 entity/scene 树移除"]
    D --> I["5. 从 Router 移除 fact_id"]
    D --> J["6. _strip_deleted_occurrences"]

    E --> K["flush_dirty_trees"]
    F --> K
    G --> K
    H --> K

    K --> L["CachingTreeBuilder<br/>检查内容哈希缓存<br/>跳过未变节点"]
    L --> M["仅对变化节点调用 LLM 重新摘要"]
    M --> N["刷新 NodeIndex"]
    N --> O["_rewire_query_pipeline"]
```

### 5.2 摘要缓存优化

删除重建时，`CachingTreeBuilder` 通过内容哈希缓存避免冗余 LLM 调用：

```mermaid
flowchart LR
    A["脏内部节点"] --> B["收集后代 fact_ids"]
    B --> C["计算 content_key = MD5(sorted_fact_ids)"]
    C --> D{缓存命中?}
    D -->|是| E["恢复旧摘要<br/>跳过 LLM"]
    D -->|否| F["调用 LLM 生成新摘要"]
```

---

## 6. 森林合并设计（Forest Merge）

### 6.1 七步合并流水线

```mermaid
flowchart TD
    S1[Source Forest 1] --> S1_FM
    S2[Source Forest 2] --> S2_FM
    T[Empty Target] --> S1_FM

    subgraph Step1["步骤1: 事实合并"]
        S1_FM["merge_from src1<br/>→ fact_remap_1"]
        S2_FM["merge_from src2<br/>→ fact_remap_2"]
    end

    subgraph Step2["步骤2: 路由器合并"]
        S1_FM --> ER1["EntityRouter.merge_from"]
        S1_FM --> SR1["SceneRouter.merge_from → scene_remap"]
    end

    subgraph Step3["步骤3: 按目标 tree_id 分组"]
        ER1 --> GRP["merge_groups"]
        SR1 --> GRP
    end

    subgraph Step4["步骤4: 复制或合并"]
        GRP --> G1{组成员数?}
        G1 -->|1| CPY["复制路径<br/>深拷贝 + remap<br/>零 LLM 调用"]
        G1 -->|>1| MRG["合并路径<br/>选最大为 trunk<br/>其他叶子插入 trunk"]
    end

    subgraph Step5["步骤5: Flush 脏树"]
        MRG --> FL["flush → LLM 重新摘要<br/>摘要缓存跳过未变节点"]
    end

    subgraph Step6["步骤6: 重建 SessionRegistry"]
        FL --> RG["_populate_registry_from_trees"]
    end

    subgraph Step7["步骤7: 持久化"]
        RG --> SV["save + _rewire_query_pipeline"]
    end

    CPY --> SV
```

### 6.2 合并优化要点

| 优化 | 说明 |
|------|------|
| 复制路径零 LLM | 单成员树深拷贝，摘要和嵌入原封不动 |
| 仅 flush 合并树 | 只有吸收了新叶子的 trunk 需要 LLM 重新摘要 |
| 摘要缓存复用 | CachingTreeBuilder 检查内容哈希，未变节点跳过 LLM |
| FAISS 向量直传 | NodeIndex 条目直接复制嵌入向量 |
| 精确+余弦双路映射 | fact_remap 先精确文本匹配，再 FAISS 余弦回退 |

---

## 7. 实体生命周期管理

```mermaid
stateDiagram-v2
    [*] --> Lazy: 首次观察到实体标签
    Lazy --> Active: 达到 min_facts + min_sessions + 标签特异性
    Active --> Suppressed: 被场景完全包含且事实数 ≤ 阈值
    Suppressed --> Active: 场景分裂后不再完全包含
    Active --> [*]: 用户删除
    Lazy --> [*]: 无足够支持
```

---

## 8. ID 生成策略

所有 ID 均为确定性生成（基于内容哈希），确保幂等处理：

| ID 类型 | 生成规则 |
|---------|---------|
| `turn_id` | `{session_id}#turn_{idx:04d}` 或 `content_id` |
| `cell_id` | `{session_id}#cell_{chunk_index:04d}_{md5_digest[:12]}` |
| `item_id` | `item_{md5(prefix\|seed\|index\|value)[:16]}` |
| `fact_id` | `fact_{md5(session_id\|cell_id\|item_id\|fact_text)[:16]}` |
| `node_id` | `{tree_id}:L{level}:{index}` |
| `tree_id` | `entity:{key}` / `scene:{label}_{uuid8}` / `session:{id}` |

---

## 9. 持久化策略

```mermaid
graph LR
    subgraph FactManager
        FM1[manifest.json<br/>schema+模型+维度]
        FM2[facts.jsonl<br/>所有 ManagedFact]
        FM3[faiss.index<br/>FAISS 向量索引]
        FM4[faiss_ids.json<br/>向量 ID 映射]
        FM5[duplicates.jsonl<br/>去重记录]
    end

    subgraph TreeStore
        TS1[trees/session/*.json<br/>Session 树]
        TS2[trees/entity/*.json<br/>Entity 树]
        TS3[trees/scene/*.json<br/>Scene 树]
    end

    subgraph NodeIndex
        NI1[node_index/faiss.index]
        NI2[node_index/emb_store.json]
        NI3[node_index/entries.json]
    end

    subgraph UserMeta
        UM1[session_registry.json]
        UM2[summary_cache.json]
        UM3[cell_store.json]
        UM4[metadata.json]
    end
```

---

## 10. 关键设计决策

### 10.1 为什么使用三种树视图？

- **Session 树**：保留对话的原始时序结构和上下文，支持 cell 级精确检索
- **Entity 树**：按实体聚合跨会话事实，解决"关于 X 的一切"类查询
- **Scene 树**：数据驱动的主题聚类，捕获隐含的语义关联，弥补实体路由的盲区

### 10.2 为什么 L0 摘要直通？

Entity/Scene L0 是单个原子事实，无需压缩；Session L0 的 LLM 摘要实际会膨胀文本（0.7x 压缩比），真正的压缩从 L1 开始（k 个子节点合并）。

### 10.3 为什么使用 Union 召回？

TD（根向量检索）速度快但可能遗漏深层事实；BU（事实级检索再聚合）精度高但需 FactManager。Union 合并在 K=10 时达到 100% essential fact coverage。

### 10.4 为什么使用全局 Beam Search？

旧实现按父节点保留 top-beam（导致 beam^depth 复杂度），新实现全局 top-beam（保证最终 L0 集合不超过 beam 个节点），更可控。
