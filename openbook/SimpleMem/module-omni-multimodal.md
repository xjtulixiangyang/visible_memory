# 核心模块设计: Omni-SimpleMem (多模态记忆)

> 源码路径: `simplemem/multimodal/`

## 1. 模块概述

Omni-SimpleMem 将 SimpleMem 的压缩优先哲学扩展到四种模态（文本、图像、音频、视频），基于三大原则：
1. **选择性摄入 (Selective Ingestion)**: 每种模态的熵驱动过滤
2. **渐进式检索 (Progressive Retrieval)**: FAISS + BM25 混合检索，金字塔 token 预算扩展
3. **知识图谱增强 (Knowledge Graph Augmentation)**: 多跳跨模态推理

## 2. 模块架构

```mermaid
graph TB
    subgraph "编排层"
        ORC["OmniMemoryOrchestrator<br/>(统一入口)"]
    end

    subgraph "处理层 (Processors)"
        TP["TextProcessor"]
        IP["ImageProcessor"]
        AP["AudioProcessor"]
        VP["VideoProcessor"]
    end

    subgraph "核心数据"
        MAU["MAU<br/>(多模态原子单元)"]
        EVT["EventNode<br/>(事件节点)"]
    end

    subgraph "存储层"
        MS["MAUStore<br/>(内存索引)"]
        HVS["HybridVectorStore<br/>(FAISS)"]
        CS["ColdStorage<br/>(原始数据)"]
        ES["EventStore<br/>(事件存储)"]
        BM25["BM25Store<br/>(关键词)"]
    end

    subgraph "检索层"
        PR["PyramidRetriever<br/>(金字塔检索)"]
        QP["QueryProcessor"]
        EM["ExpansionManager"]
    end

    subgraph "知识层"
        KG["KnowledgeGraph"]
        EE["EntityExtractor"]
        GR["GraphRetriever"]
    end

    subgraph "参数化层"
        PS["ParametricMemoryStore"]
        MC["MemoryConsolidator"]
        MD["MemoryDistiller"]
    end

    subgraph "演化层 (可选)"
        MEC["MetaController"]
        EXP["ExperienceEngine"]
        SO["StrategyOptimizer"]
    end

    ORC --> TP & IP & AP & VP
    TP & IP & AP & VP --> MAU
    ORC --> MS & HVS & CS & ES & BM25
    ORC --> PR & QP & EM
    ORC --> KG & EE & GR
    ORC --> PS & MC & MD
    ORC -.-> MEC & EXP & SO
```

## 3. 核心数据模型: MAU

```mermaid
classDiagram
    class ModalityType {
        <<enumeration>>
        TEXT
        VISUAL
        AUDIO
        VIDEO
        MULTIMODAL
    }

    class QualityMetrics {
        +trigger_score: float
        +confidence: float
        +entropy_delta: float
        +to_dict() Dict
    }

    class MAUMetadata {
        +session_id: Optional~str~
        +source: Optional~str~
        +tags: List~str~
        +quality: QualityMetrics
        +duration_ms: Optional~int~
        +frame_index: Optional~int~
        +speaker_id: Optional~str~
        +persons: Optional~List~str~~
        +entities: Optional~List~str~~
        +keywords: Optional~List~str~~
        +location: Optional~str~
        +topic: Optional~str~
    }

    class MAULinks {
        +event_id: Optional~str~
        +prev_mau_id: Optional~str~
        +next_mau_id: Optional~str~
        +related_mau_ids: List~str~
    }

    class MultimodalAtomicUnit {
        +id: str
        +timestamp: float
        +modality_type: ModalityType
        +summary: str
        +embedding: Optional~List~float~~
        +raw_pointer: Optional~str~
        +details: Optional~Dict~
        +metadata: MAUMetadata
        +links: MAULinks
        +status: str
        +storage_tier: str
        +to_dict() Dict
        +get_lightweight_dict() Dict
        +has_details() bool
        +clear_details()
        +add_tag(tag)
        +add_related(mau_id)
    }

    MultimodalAtomicUnit --> ModalityType
    MultimodalAtomicUnit --> MAUMetadata
    MultimodalAtomicUnit --> MAULinks
    MAUMetadata --> QualityMetrics
```

**MAU 的内存-计算解耦设计**:

```mermaid
graph LR
    subgraph "热存储 (RAM)"
        L1["id, timestamp, modality_type"]
        L2["summary (短文本)"]
        L3["embedding (向量)"]
        L4["metadata, links"]
        L5["raw_pointer (指针)"]
    end

    subgraph "冷存储 (Disk)"
        RAW["原始图像/音频/视频"]
        DET["details (完整内容)"]
    end

    L5 -.->|指针引用| RAW
    L1 & L2 & L3 & L4 -->|按需加载| DET

    style L1 fill:#e1f5fe
    style L2 fill:#e1f5fe
    style L3 fill:#e1f5fe
    style L4 fill:#e1f5fe
    style L5 fill:#fff3e0
    style RAW fill:#fce4ec
    style DET fill:#fce4ec
```

## 4. 处理器设计

### 4.1 处理器基类与继承

```mermaid
classDiagram
    class BaseProcessor {
        <<abstract>>
        +config: OmniMemoryConfig
        +cold_storage: ColdStorageManager
        +process(input, session_id, **kwargs) ProcessingResult
    }

    class TextProcessor {
        +process(text, session_id, force) ProcessingResult
        -_compute_entropy(text) float
        -_should_store(text, force) bool
    }

    class ImageProcessor {
        +process(image, session_id, force) ProcessingResult
        -_compute_visual_entropy(image) float
        -_encode_image(image) List~float~
    }

    class AudioProcessor {
        +process(audio, session_id, force) ProcessingResult
        -_detect_speech(audio) bool
        -_transcribe(audio) str
    }

    class VideoProcessor {
        +process(video_path, session_id, max_frames) ProcessingResult
        -_extract_keyframes(video) List
        -_extract_audio_track(video) Audio
    }

    BaseProcessor <|-- TextProcessor
    BaseProcessor <|-- ImageProcessor
    BaseProcessor <|-- AudioProcessor
    BaseProcessor <|-- VideoProcessor
```

### 4.2 熵触发机制

```mermaid
flowchart TB
    Input["模态输入"] --> Trigger{"熵触发检查"}

    Trigger -->|Text| TE["文本熵<br/>语义密度评估"]
    Trigger -->|Image| IE["视觉熵<br/>图像信息量评估"]
    Trigger -->|Audio| AE["VAD 检测<br/>语音活动检测"]
    Trigger -->|Video| VE["帧级熵<br/>关键帧差异度"]

    TE -->|高熵| Store["存储为 MAU"]
    IE -->|高熵| Store
    AE -->|有语音| Store
    VE -->|显著变化| Store

    TE -->|低熵| Skip["跳过 (冗余)"]
    IE -->|低熵| Skip
    AE -->|静音| Skip
    VE -->|静态帧| Skip

    Store --> QM["QualityMetrics<br/>trigger_score, confidence, entropy_delta"]
```

## 5. 金字塔检索

### 5.1 检索层级

```mermaid
graph TB
    subgraph "Level 1: SUMMARY (预览)"
        S["summary 字段<br/>~10 tokens/MAU<br/>快速浏览"]
    end

    subgraph "Level 2: DETAILS (详情)"
        D["details 字段<br/>~100 tokens/MAU<br/>按需展开"]
    end

    subgraph "Level 3: EVIDENCE (证据)"
        E["raw_content<br/>完整原始数据<br/>延迟加载"]
    end

    S -->|"auto_expand=True"| D
    D -->|"需要原始数据"| E

    style S fill:#c8e6c9
    style D fill:#fff9c4
    style E fill:#ffccbc
```

### 5.2 检索流程

```mermaid
sequenceDiagram
    participant Q as 查询
    participant QP as QueryProcessor
    participant PR as PyramidRetriever
    participant FAISS as HybridVectorStore
    participant BM25 as BM25Store
    participant KG as KnowledgeGraph
    participant PM as ParametricStore
    participant EXP as ExpansionManager

    Q->>QP: process(query)
    QP-->>QP: 清洗查询 + 确定策略

    QP->>PR: retrieve_preview(query, top_k)

    PR->>FAISS: 向量搜索 (text + visual)
    FAISS-->>PR: 候选 MAU IDs + scores

    PR->>PR: 构建 RetrievalResult<br/>(SUMMARY 级别)

    PR->>BM25: search(query, top_k)
    BM25-->>PR: BM25 补充结果
    Note over PR: FAISS 顺序保持不变<br/>BM25 结果追加到末尾

    PR->>PM: recall(query, top_k=3)
    PM-->>PR: 参数化答案 (confidence > 0.8)

    PR->>KG: retrieve(query, max_entities=5)
    KG-->>PR: 图谱实体上下文

    opt auto_expand=True
        PR->>EXP: expand(high_score_mau_ids)
        EXP->>EXP: 加载 details/raw_content
        EXP-->>PR: 展开后的 RetrievalResult
    end

    PR-->>Q: RetrievalResult
```

## 6. 知识图谱

```mermaid
flowchart LR
    subgraph "实体抽取"
        MAU["MAU summary"] --> EE["EntityExtractor<br/>(LLM驱动)"]
        EE --> Entities["ExtractedEntity<br/>(name, type, attributes)"]
        EE --> Relations["ExtractedRelation<br/>(source, target, relation_type)"]
    end

    subgraph "图谱构建"
        Entities --> KG["KnowledgeGraph<br/>(内存图结构)"]
        Relations --> KG
    end

    subgraph "图谱检索"
        Query["查询"] --> GR["GraphRetriever"]
        GR --> KG
        GR --> Context["跨模态关联上下文<br/>(多跳推理)"]
    end
```

## 7. 参数化记忆

```mermaid
flowchart TB
    subgraph "蒸馏过程"
        MAUs["频繁访问的 MAUs<br/>(importance > 0.7)"] --> MD["MemoryDistiller<br/>(LLM 驱动)"]
        MD --> Facts["参数化知识<br/>(key-value 事实)"]
    end

    subgraph "存储"
        Facts --> PS["ParametricMemoryStore<br/>(磁盘持久化)"]
    end

    subgraph "快速召回"
        Query["查询"] --> PS
        PS -->|confidence > 0.8| FastAnswer["直接答案<br/>(无需向量检索)"]
        PS -->|confidence ≤ 0.8| Fallback["降级到向量检索"]
    end
```

## 8. 记忆合并与归档

```mermaid
sequenceDiagram
    participant ORC as Orchestrator
    participant CON as MemoryConsolidator
    participant MS as MAUStore
    participant KG as KnowledgeGraph

    ORC->>CON: consolidate_memories(force)
    CON->>MS: 获取所有 MAUs
    CON->>CON: 计算重要性分数<br/>(访问频率 + 时间衰减)

    CON->>CON: 识别低重要性 MAUs
    CON->>CON: 识别可合并的重复 MAUs

    loop 每个合并组
        CON->>CON: LLM 合并摘要
        CON->>MS: 更新/删除 MAUs
        CON->>KG: 更新实体关系
    end

    CON->>CON: 归档低重要性 MAUs<br/>(status=ARCHIVED, storage_tier=COLD)
    CON-->>ORC: 合并报告
```

## 9. 多模态摄入策略

### 9.1 图像摄入

```mermaid
flowchart TB
    IMG["图像输入<br/>(路径/PIL/numpy/bytes)"] --> ENT{"熵触发"}
    ENT -->|高熵| PROC["处理"]
    ENT -->|低熵 + force=False| SKIP["跳过"]

    PROC --> EMB["生成视觉嵌入"]
    PROC --> CAP["生成文本摘要"]
    PROC --> COLD["原始图像存入 ColdStorage"]
    PROC --> MAU["创建 MAU (VISUAL)"]

    MAU --> STRAT{"嵌入策略"}
    STRAT -->|默认| SEP["text_dim + visual_dim<br/>分别存储"]
    STRAT -->|averaged| AVG["(caption_emb + image_emb) / 2<br/>单向量存储"]
    STRAT -->|on_demand| OD["仅 caption 嵌入<br/>raw_pointer 延迟加载"]
```

### 9.2 视频摄入

```mermaid
flowchart TB
    VID["视频输入"] --> FRAME["关键帧提取<br/>(max_frames)"]
    VID --> AUDIO["音频轨道提取"]

    FRAME --> FE["帧级熵触发"]
    FE -->|显著变化| KF["关键帧 MAU (VISUAL)"]
    FE -->|静态| SKIP["跳过"]

    AUDIO --> VAD["VAD 检测"]
    VAD -->|有语音| AMAU["音频 MAU (AUDIO)"]
    VAD -->|静音| SKIP2["跳过"]

    KF --> VMAU["视频 MAU (VIDEO)<br/>包含 frame_maus + audio_mau"]
    AMAU --> VMAU
```

## 10. 自演化集成 (可选)

```mermaid
flowchart TB
    subgraph "演化组件"
        MC["MetaController<br/>自适应参数调节"]
        EXE["ExperienceEngine<br/>经验记录与检索"]
        SO["StrategyOptimizer<br/>Thompson Sampling 策略选择"]
    end

    subgraph "策略选择"
        Query["查询"] --> FEAT["提取查询特征"]
        FEAT --> PAST["检索相关经验"]
        PAST --> TS["Thompson Sampling<br/>选择策略"]
        TS -->|vector| VEC["向量检索路径"]
        TS -->|parametric| PAR["参数化检索路径"]
        TS -->|graph| GRA["图谱检索路径"]
        TS -->|hybrid| HYB["混合检索路径"]
    end

    subgraph "反馈闭环"
        Result["检索结果"] --> QUAL["答案质量评分"]
        QUAL --> MC
        QUAL --> EXE
        QUAL --> SO
    end
```

## 11. 会话管理

```mermaid
stateDiagram-v2
    [*] --> Active: start_session()
    Active --> Active: add_text/image/audio/video()
    Active --> Active: query()
    Active --> Closed: end_session()

    state Active {
        [*] --> Collecting
        Collecting --> Collecting: 持续摄入
        Collecting --> EventOpen: EventManager 维护事件
        EventOpen --> EventOpen: MAU 关联到事件
    }

    state Closed {
        EventOpen --> EventsClosed: close_all_open_events()
        EventsClosed --> Saved: save()
    }

    Saved --> [*]
```

## 12. 配置体系

```mermaid
classDiagram
    class OmniMemoryConfig {
        +storage: StorageConfig
        +embedding: EmbeddingConfig
        +llm: LLMConfig
        +retrieval: RetrievalConfig
        +evolution: EvolutionConfig
        +enable_self_evolution: bool
        +create_default() OmniMemoryConfig
        +ensure_directories()
    }

    class StorageConfig {
        +base_dir: str
        +cold_storage_dir: str
        +index_dir: str
    }

    class EmbeddingConfig {
        +embedding_dim: int
        +visual_embedding_dim: int
        +model_name: str
    }

    class LLMConfig {
        +api_key: Optional~str~
        +api_base_url: Optional~str~
        +query_model: str
    }

    class RetrievalConfig {
        +top_k: int
        +auto_expand_threshold: float
        +max_expanded_items: int
        +enable_multi_query_retrieval: bool
        +multi_query_max_queries: int
    }

    OmniMemoryConfig --> StorageConfig
    OmniMemoryConfig --> EmbeddingConfig
    OmniMemoryConfig --> LLMConfig
    OmniMemoryConfig --> RetrievalConfig
```
