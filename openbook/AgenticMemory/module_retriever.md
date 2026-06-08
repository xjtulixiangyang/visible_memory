# 检索系统模块设计文档（Module Retriever）

## 1. 模块概述

检索系统是 AgenticMemory 项目的核心组件，负责从记忆库中高效地找到与查询最相关的记忆条目。系统提供两种检索策略：

- **HybridRetriever**：混合检索，结合 BM25 关键词匹配与语义嵌入检索，通过 `alpha` 参数灵活控制权重
- **SimpleEmbeddingRetriever**：纯语义嵌入检索，仅基于 SentenceTransformer 嵌入的余弦相似度

当前 `AgenticMemorySystem` 默认使用 `SimpleEmbeddingRetriever`。

**源文件**：`memory_layer.py`

**核心依赖**：
| 依赖 | 用途 |
|------|------|
| `rank_bm25.BM25Okapi` | BM25 关键词检索算法 |
| `sentence_transformers.SentenceTransformer` | 语义嵌入模型（默认 `all-MiniLM-L6-v2`） |
| `sklearn.metrics.pairwise.cosine_similarity` | 余弦相似度计算 |
| `nltk.tokenize.word_tokenize` | 文本分词 |
| `numpy` | 嵌入向量存储与运算 |
| `pickle` | 检索器状态序列化 |

---

## 2. 检索系统类图

```mermaid
classDiagram
    class HybridRetriever {
        -model: SentenceTransformer
        -alpha: float
        -bm25: BM25Okapi
        -corpus: List~str~
        -embeddings: ndarray
        -document_ids: Dict~str, int~
        +__init__(model_name, alpha)
        +add_documents(documents: List~str~) bool
        +add_document(document: str) bool
        +retrieve(query: str, k: int) List~int~
        +save(cache_file, embeddings_file)
        +load(cache_file, embeddings_file)$ HybridRetriever
        +load_from_local_memory(memories, model_name, alpha)$ HybridRetriever
    }

    class SimpleEmbeddingRetriever {
        -model: SentenceTransformer
        -corpus: List~str~
        -embeddings: ndarray
        -document_ids: Dict~str, int~
        +__init__(model_name)
        +add_documents(documents: List~str~)
        +search(query: str, k: int) List~int~
        +save(cache_file, embeddings_file)
        +load(cache_file, embeddings_file) SimpleEmbeddingRetriever
        +load_from_local_memory(memories, model_name)$ SimpleEmbeddingRetriever
    }

    class AgenticMemorySystem {
        -memories: Dict~str, MemoryNote~
        -retriever: SimpleEmbeddingRetriever
        -llm_controller: LLMController
        -evo_cnt: int
        -evo_threshold: int
        +add_note(content, time, **kwargs) str
        +find_related_memories(query, k) tuple
        +find_related_memories_raw(query, k) str
        +consolidate_memories()
        -process_memory(note) tuple
    }

    class MemoryNote {
        +id: str
        +content: str
        +keywords: List~str~
        +context: str
        +tags: List~str~
        +links: List
        +importance_score: float
        +retrieval_count: int
        +timestamp: str
        +category: str
    }

    AgenticMemorySystem --> SimpleEmbeddingRetriever : retriever
    AgenticMemorySystem --> MemoryNote : memories
    HybridRetriever ..> MemoryNote : load_from_local_memory 引用
    SimpleEmbeddingRetriever ..> MemoryNote : load_from_local_memory 引用
```

---

## 3. 混合检索评分流程图

`HybridRetriever.retrieve()` 方法将 BM25 稀疏检索与语义密集检索融合，通过 `alpha` 参数加权合并得分。

```mermaid
flowchart TD
    A[输入查询 query] --> B[BM25 分支]
    A --> C[语义检索分支]

    B --> B1[查询分词: query.lower.split]
    B1 --> B2[bm25.get_scores 获取原始 BM25 分数]
    B2 --> B3[Min-Max 归一化<br/>scores - min / max - min + 1e-6]
    B3 --> D[归一化后的 bm25_scores]

    C --> C1[查询编码: model.encode query]
    C1 --> C2[cosine_similarity query_embedding, embeddings]
    C2 --> E[semantic_scores]

    D --> F[混合加权<br/>hybrid = alpha × bm25 + 1-alpha × semantic]
    E --> F

    F --> G[np.argsort 按 hybrid_scores 降序排列]
    G --> H[取 Top-K 索引]
    H --> I[返回 List~int~ 索引列表]

    style A fill:#e1f5fe
    style I fill:#c8e6c9
    style F fill:#fff9c4
```

**alpha 权重说明**：

| alpha 值 | 行为 | 适用场景 |
|-----------|------|----------|
| `0.0` | 纯 BM25 关键词匹配 | 精确关键词检索 |
| `0.5`（默认） | 均衡混合 | 通用场景 |
| `1.0` | 纯语义检索 | 语义相似但词汇不同 |

**BM25 分数归一化**：由于 BM25 原始分数范围不固定，而余弦相似度在 `[-1, 1]` 之间，必须对 BM25 分数做 Min-Max 归一化到 `[0, 1]` 后才能与语义分数加权合并。分母加 `1e-6` 防止除零。

---

## 4. 文档索引构建时序图

### 4.1 增量添加（add_note → add_documents）

当 `AgenticMemorySystem.add_note()` 被调用时，系统会构建包含内容与元数据的文档字符串并添加到检索器。

```mermaid
sequenceDiagram
    participant User
    participant AMS as AgenticMemorySystem
    participant MN as MemoryNote
    participant SER as SimpleEmbeddingRetriever
    participant ST as SentenceTransformer

    User->>AMS: add_note(content, time, **kwargs)
    AMS->>MN: 创建 MemoryNote(content, llm_controller, timestamp, **kwargs)
    MN->>MN: LLM 分析生成 keywords, context, tags
    AMS->>AMS: process_memory(note) → 演化处理
    AMS->>AMS: memories[note.id] = note

    Note over AMS: 构建文档字符串<br/>"content:{content} context:{context}<br/>keywords:{keywords} tags:{tags}"

    AMS->>SER: add_documents([doc_string])

    alt corpus 为空（首次添加）
        SER->>SER: corpus = documents
        SER->>ST: model.encode(documents)
        ST-->>SER: embeddings
        SER->>SER: document_ids = {doc: idx}
    else corpus 非空（追加）
        SER->>SER: corpus.extend(documents)
        SER->>ST: model.encode(documents)
        ST-->>SER: new_embeddings
        SER->>SER: embeddings = np.vstack([embeddings, new_embeddings])
        SER->>SER: document_ids[doc] = start_idx + idx
    end

    AMS-->>User: note.id
```

### 4.2 从记忆字典重建索引（load_from_local_memory）

```mermaid
sequenceDiagram
    participant Caller
    participant SER as SimpleEmbeddingRetriever
    participant ST as SentenceTransformer

    Caller->>SER: load_from_local_memory(memories, model_name)

    loop 遍历 memories.values()
        SER->>SER: 构建 metadata_text<br/>= context + keywords.join + tags.join
        SER->>SER: 构建 doc<br/>= content + " , " + metadata_text
        SER->>SER: all_docs.append(doc)
    end

    SER->>SER: 创建实例 cls(model_name)
    SER->>SER: add_documents(all_docs)
    SER->>ST: model.encode(all_docs)
    ST-->>SER: embeddings

    SER-->>Caller: 返回重建后的 retriever
```

**文档构建格式对比**：

| 场景 | 文档格式 | 代码位置 |
|------|----------|----------|
| `add_note` 增量添加 | `"content:{content} context:{context} keywords:{kw} tags:{tags}"` | `AgenticMemorySystem.add_note()` |
| `load_from_local_memory` 重建 | `"{content} , {context} {keywords} {tags}"` | `SimpleEmbeddingRetriever.load_from_local_memory()` |
| `consolidate_memories` 重建 | `"{content} , {context} {keywords} {tags}"` | `AgenticMemorySystem.consolidate_memories()` |

> ⚠️ 注意：`add_note` 与 `load_from_local_memory`/`consolidate_memories` 的文档拼接格式存在差异，前者使用 `content:`/`context:`/`keywords:`/`tags:` 前缀标签，后者使用逗号分隔。这可能导致同一记忆在不同阶段生成的嵌入向量不一致。

---

## 5. 持久化存储流程

### 5.1 保存流程

```mermaid
flowchart LR
    subgraph SimpleEmbeddingRetriever.save
        A1[embeddings ≠ None?] -->|Yes| A2["np.save(embeddings_file, embeddings)"]
        A1 -->|No| A3[跳过嵌入保存]
        A2 --> A4["pickle.dump({corpus, document_ids})"]
        A3 --> A4
    end

    subgraph HybridRetriever.save
        B1[embeddings ≠ None?] -->|Yes| B2["np.save(embeddings_file, embeddings)"]
        B1 -->|No| B3[跳过嵌入保存]
        B2 --> B4["pickle.dump({alpha, bm25, corpus, document_ids, model_name})"]
        B3 --> B4
    end

    style A4 fill:#bbdefb
    style B4 fill:#bbdefb
```

### 5.2 加载流程

```mermaid
flowchart TD
    subgraph SimpleEmbeddingRetriever.load
        S1["embeddings_file 存在?"] -->|Yes| S2["np.load(embeddings_file)"]
        S1 -->|No| S3["打印警告，embeddings=None"]
        S2 --> S4["cache_file 存在?"]
        S3 --> S4
        S4 -->|Yes| S5["pickle.load → 恢复 corpus, document_ids"]
        S4 -->|No| S6["打印警告"]
        S5 --> S7["return self"]
        S6 --> S7
    end

    subgraph HybridRetriever.load
        H1["pickle.load(cache_file)"] --> H2["创建实例 cls(model_name, alpha)"]
        H2 --> H3["恢复 bm25, corpus, document_ids"]
        H3 --> H4["embeddings_file 存在?"]
        H4 -->|Yes| H5["np.load(embeddings_file)"]
        H4 -->|No| H6["embeddings=None"]
        H5 --> H7["return retriever"]
        H6 --> H7
    end
```

**存储文件说明**：

| 文件 | 格式 | 内容 | 适用检索器 |
|------|------|------|------------|
| `retriever_cache_*.pkl` | Pickle | corpus, document_ids, alpha, bm25, model_name | 两者共用 |
| `retriever_cache_embeddings_*.npy` | NumPy | embeddings 矩阵 | 两者共用 |

> 💡 `SimpleEmbeddingRetriever.load()` 是实例方法（返回 `self`），而 `HybridRetriever.load()` 是类方法（返回新实例）。

---

## 6. 两种检索器对比

```mermaid
graph TB
    subgraph HybridRetriever
        direction TB
        H_IN[输入查询] --> H_BM[BM25 稀疏检索]
        H_IN --> H_SEM[语义密集检索]
        H_BM --> H_NORM[BM25 分数归一化]
        H_SEM --> H_COMB[alpha 加权合并]
        H_NORM --> H_COMB
        H_COMB --> H_TOP[Top-K 排序]
        H_TOP --> H_OUT[返回索引列表]
    end

    subgraph SimpleEmbeddingRetriever
        direction TB
        S_IN[输入查询] --> S_ENC[查询编码]
        S_ENC --> S_COS[余弦相似度计算]
        S_COS --> S_TOP[Top-K 排序]
        S_TOP --> S_OUT[返回索引列表]
    end

    style H_COMB fill:#fff9c4
    style S_COS fill:#c8e6c9
```

| 特性 | HybridRetriever | SimpleEmbeddingRetriever |
|------|-----------------|--------------------------|
| **检索方法** | BM25 + 语义嵌入混合 | 纯语义嵌入 |
| **权重控制** | `alpha` 参数（0=纯BM25, 1=纯语义） | 无 |
| **检索入口** | `retrieve(query, k)` | `search(query, k)` |
| **增量添加** | `add_document()` 支持单文档增量 | `add_documents()` 仅批量（追加模式） |
| **BM25 索引** | 需要，增量时调用 `bm25.add_document()` | 不需要 |
| **嵌入存储** | `torch.Tensor`（add_document 用 `torch.cat`） | `numpy.ndarray`（用 `np.vstack` 追加） |
| **load 方法** | `@classmethod`，返回新实例 | 实例方法，返回 `self` |
| **load_from_local_memory 文档** | 仅使用 `keywords` | 使用 `content + context + keywords + tags` |
| **当前使用** | 未被 `AgenticMemorySystem` 使用 | ✅ 默认检索器 |
| **复杂度** | 较高，需维护 BM25 + 嵌入双索引 | 较低，仅维护嵌入索引 |

---

## 7. consolidate_memories 重建流程

`consolidate_memories()` 在演化计数达到阈值时触发，完全重建检索索引以确保所有记忆的元数据（经过演化更新后的 context、tags 等）被正确反映到检索空间中。

```mermaid
flowchart TD
    A["evo_cnt % evo_threshold == 0"] --> B[触发 consolidate_memories]

    B --> C["尝试获取 model_name<br/>retriever.model.get_config_dict"]
    C -->|成功| D1["model_name = 配置中的名称"]
    C -->|失败| D2["model_name = 'all-MiniLM-L6-v2'"]

    D1 --> E["创建新的 SimpleEmbeddingRetriever(model_name)"]
    D2 --> E

    E --> F["旧 retriever 被丢弃<br/>self.retriever = 新实例"]

    F --> G["遍历 self.memories.values()"]

    G --> H["构建 metadata_text<br/>= context + keywords.join + tags.join"]
    H --> I["构建文档<br/>= content + ' , ' + metadata_text"]
    I --> J["retriever.add_documents([doc])"]

    J --> K{还有更多记忆?}
    K -->|Yes| G
    K -->|No| L[重建完成]

    style A fill:#ffcdd2
    style F fill:#fff9c4
    style L fill:#c8e6c9
```

### 重建触发机制

```mermaid
sequenceDiagram
    participant User
    participant AMS as AgenticMemorySystem
    participant SER as SimpleEmbeddingRetriever

    loop 每次添加记忆
        User->>AMS: add_note(content)
        AMS->>AMS: process_memory(note)
        AMS->>AMS: evo_label == True?
        alt 演化触发
            AMS->>AMS: evo_cnt += 1
            AMS->>AMS: evo_cnt % evo_threshold == 0?
            alt 达到阈值
                AMS->>AMS: consolidate_memories()
                AMS->>SER: 丢弃旧实例
                AMS->>SER: 创建新 SimpleEmbeddingRetriever
                loop 每个 memory
                    AMS->>SER: add_documents([doc])
                end
                Note over AMS: 检索索引完全重建
            end
        end
    end
```

**重建的必要性**：记忆演化（`process_memory`）会修改记忆的 `context`、`tags`、`links` 等元数据，但增量添加时文档字符串是固定的，不会随演化更新。因此需要定期重建索引，将演化后的最新元数据重新编码到嵌入空间中。

**重建代价**：每次重建需要对所有记忆重新编码嵌入，时间复杂度为 O(N)，其中 N 为记忆总数。`evo_threshold`（默认 100）控制重建频率。

---

## 8. 关键实现细节

### 8.1 文档索引映射

两个检索器均使用 `document_ids: Dict[str, int]` 维护文档内容到索引的映射，用于去重和定位：

```
document_ids = {
    "content:xxx context:yyy keywords:kkk tags:ttt": 0,
    "content:aaa context:bbb keywords:ccc tags:ddd": 1,
    ...
}
```

### 8.2 嵌入向量存储差异

| 操作 | HybridRetriever | SimpleEmbeddingRetriever |
|------|-----------------|--------------------------|
| 批量添加 | `model.encode()` → numpy | `model.encode()` → numpy |
| 增量添加 | `torch.cat([old, new])` → Tensor | `np.vstack([old, new])` → ndarray |

> ⚠️ `HybridRetriever.add_document()` 使用 `convert_to_tensor=True` 编码，导致增量添加后 `embeddings` 为 `torch.Tensor`，而批量添加时为 `numpy.ndarray`。这可能引发类型不一致问题。

### 8.3 检索结果使用方式

`AgenticMemorySystem` 通过检索器返回的索引列表，直接按索引访问 `list(self.memories.values())` 获取对应的 `MemoryNote` 对象：

```python
indices = self.retriever.search(query, k)
all_memories = list(self.memories.values())
for i in indices:
    memory = all_memories[i]
```

> ⚠️ 此设计假设检索器中文档的顺序与 `self.memories.values()` 的顺序一致。在 `consolidate_memories` 重建后，由于逐个调用 `add_documents`，顺序与字典插入顺序一致，可保证正确性。但 `load_from_local_memory` 重建时，顺序取决于 `memories.values()` 的迭代顺序。
