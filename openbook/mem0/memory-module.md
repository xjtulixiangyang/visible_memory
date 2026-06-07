# Mem0 Memory 子模块深度解析

## 1. 模块概述

### 1.1 模块职责

Memory 子模块是 mem0 项目的核心，负责为 AI 代理和助手提供持久化、个性化的记忆层。它实现了记忆的添加、搜索、更新、删除和变更历史追踪等完整生命周期管理。

### 1.2 核心类

| 类名 | 职责 |
|------|------|
| `MemoryBase` | 抽象基类，定义 Memory 的公共接口契约 |
| `Memory` | 同步实现，提供完整的记忆 CRUD 操作 |
| `AsyncMemory` | 异步实现，基于 `asyncio.to_thread` 的非阻塞版本 |

### 1.3 文件结构

| 文件 | 职责 |
|------|------|
| `mem0/memory/base.py` | 抽象基类，定义 6 个抽象方法 |
| `mem0/memory/main.py` | Memory / AsyncMemory 完整实现（约 3279 行） |
| `mem0/memory/utils.py` | 工具函数：消息解析、JSON 提取、视觉处理、遥测过滤等 |
| `mem0/memory/storage.py` | SQLiteManager，负责历史记录和消息持久化 |
| `mem0/memory/telemetry.py` | 遥测事件上报 |

---

## 2. 类层次设计

```mermaid
classDiagram
    class MemoryBase {
        <<abstract>>
        +get(memory_id)* dict
        +get_all()* dict
        +update(memory_id, data)* dict
        +delete(memory_id)* void
        +history(memory_id)* list
    }

    class Memory {
        +config: MemoryConfig
        +embedding_model: Embedder
        +vector_store: VectorStore
        +llm: LLM
        +db: SQLiteManager
        +reranker: Reranker
        -_entity_store: VectorStore
        +add(messages, **kwargs) dict
        +get(memory_id) dict
        +get_all(filters, top_k) dict
        +search(query, top_k, filters, threshold, rerank, explain) dict
        +update(memory_id, data) dict
        +delete(memory_id) dict
        +delete_all(**kwargs) dict
        +history(memory_id) list
        +reset() void
        +close() void
        -_add_to_vector_store(messages, metadata, filters, infer) list
        -_search_vector_store(query, filters, limit, threshold) list
        -_create_memory(data, embeddings, metadata) str
        -_update_memory(memory_id, data, embeddings, metadata) str
        -_delete_memory(memory_id, existing_memory) str
        -_create_procedural_memory(messages, metadata, prompt) dict
        -_upsert_entity(entity_text, entity_type, memory_id, filters) void
        -_remove_memory_from_entity_store(memory_id, filters) void
        -_link_entities_for_memory(memory_id, text, filters) void
        -_compute_entity_boosts(query_entities, filters) dict
        -_process_metadata_filters(metadata_filters) dict
        -_has_advanced_operators(filters) bool
    }

    class AsyncMemory {
        +config: MemoryConfig
        +embedding_model: Embedder
        +vector_store: VectorStore
        +llm: LLM
        +db: SQLiteManager
        +reranker: Reranker
        -_entity_store: VectorStore
        +add(messages, **kwargs)~ dict
        +get(memory_id)~ dict
        +get_all(filters, top_k)~ dict
        +search(query, top_k, filters, threshold, rerank, explain)~ dict
        +update(memory_id, data)~ dict
        +delete(memory_id)~ dict
        +delete_all(**kwargs)~ dict
        +history(memory_id)~ list
        +reset()~ void
        +close() void
        -_add_to_vector_store(messages, metadata, filters, infer)~ list
        -_search_vector_store(query, filters, limit, threshold)~ list
        -_create_memory(data, embeddings, metadata)~ str
        -_update_memory(memory_id, data, embeddings, metadata)~ str
        -_delete_memory(memory_id, existing_memory)~ str
        -_create_procedural_memory(messages, metadata, llm, prompt)~ dict
        -_upsert_entity_async(entity_text, entity_type, memory_id, filters)~ void
        -_remove_memory_from_entity_store(memory_id, filters)~ void
        -_link_entities_for_memory(memory_id, text, filters)~ void
        -_compute_entity_boosts_async(query_entities, filters)~ dict
    }

    MemoryBase <|-- Memory
    MemoryBase <|-- AsyncMemory
```

---

## 3. Memory 类核心组件

### 3.1 组件初始化与协作关系

```mermaid
graph TB
    subgraph Memory初始化
        Config[MemoryConfig]
        Config --> Embedder[EmbedderFactory.create]
        Config --> VS[VectorStoreFactory.create]
        Config --> LLM[LlmFactory.create]
        Config --> DB[SQLiteManager]
        Config --> Reranker[RerankerFactory.create]
    end

    subgraph 核心组件
        Embedder -->|"embed / embed_batch"| VS
        LLM -->|"generate_response"| Extract[事实提取]
        VS -->|"insert / search / list / update / delete"| Persist[持久化层]
        DB -->|"add_history / batch_add_history / save_messages"| SQLite[(SQLite)]
        Reranker -->|"rerank"| SearchResult[搜索结果重排]
    end

    subgraph 懒加载组件
        EntityStore[Entity Store<br/>VectorStore实例]
    end

    VS -.->|"首次访问时创建"| EntityStore
    EntityStore -->|"search_batch / insert / update"| EntityPersist[实体持久化]
```

### 3.2 各组件职责

| 组件 | 类型 | 职责 |
|------|------|------|
| `embedding_model` | Embedder | 将文本转为向量，支持单条 `embed()` 和批量 `embed_batch()` |
| `vector_store` | VectorStore | 记忆的向量存储与检索，支持语义搜索和关键词搜索 |
| `llm` | LLM | 事实提取、程序性记忆生成，支持 `response_format` |
| `db` | SQLiteManager | 历史记录持久化、消息保存，线程安全（`threading.Lock`） |
| `reranker` | Reranker（可选） | 搜索结果重排序 |
| `entity_store` | VectorStore（懒加载） | 实体存储，独立集合 `{collection_name}_entities` |

---

## 4. add() 方法完整流程

### 4.1 V3 批量管线 8 阶段详解

```mermaid
sequenceDiagram
    participant Client
    participant Memory
    participant DB as SQLiteManager
    participant Embedder as EmbeddingModel
    participant VS as VectorStore
    participant LLM as LLM
    participant ES as EntityStore
    participant EE as EntityExtraction

    Note over Memory: Phase 0: 上下文收集
    Memory->>DB: get_last_messages(session_scope, limit=10)
    DB-->>Memory: last_messages
    Memory->>Memory: parse_messages(messages)

    Note over Memory: Phase 1: 已有记忆检索
    Memory->>Embedder: embed(parsed_messages, "search")
    Embedder-->>Memory: query_embedding
    Memory->>VS: search(query, vectors, top_k=10, filters)
    VS-->>Memory: existing_results
    Memory->>Memory: UUID→整数映射（反幻觉）

    Note over Memory: Phase 2: LLM 事实提取（单次调用）
    Memory->>LLM: generate_response(system_prompt + user_prompt, response_format=json_object)
    LLM-->>Memory: response (JSON)
    Memory->>Memory: remove_code_blocks → json.loads → extract_json 回退

    alt 无提取结果
        Memory->>DB: save_messages(messages, session_scope)
        Memory-->>Client: []
    end

    Note over Memory: Phase 3: 批量嵌入
    Memory->>Embedder: embed_batch(mem_texts, "add")
    Embedder-->>Memory: mem_embeddings_list
    alt 批量嵌入失败
        loop 逐条嵌入
            Memory->>Embedder: embed(text, "add")
        end
    end

    Note over Memory: Phase 4+5: CPU处理 + Hash去重
    Memory->>Memory: 构建 existing_hashes 集合
    loop 每条提取的记忆
        Memory->>Memory: MD5 hash 计算
        alt hash 已存在
            Memory->>Memory: 跳过（去重）
        else hash 不存在
            Memory->>Memory: lemmatize_for_bm25(text)
            Memory->>Memory: 构建 metadata
            Memory->>Memory: 追加到 records
        end
    end

    Note over Memory: Phase 6: 批量持久化
    Memory->>VS: insert(vectors, ids, payloads) [批量]
    alt 批量插入失败
        loop 逐条插入
            Memory->>VS: insert([vec], [mid], [pay])
        end
    end
    Memory->>DB: batch_add_history(history_records)

    Note over Memory: Phase 7: 批量实体链接
    Memory->>EE: extract_entities_batch(all_texts)
    EE-->>Memory: all_entities
    Memory->>Memory: 全局去重实体
    Memory->>Embedder: embed_batch(entity_texts, "add")
    Memory->>ES: search_batch(queries, vectors_list, top_k=1)
    loop 每个实体
        alt 相似度 >= 0.95 (已有实体)
            Memory->>ES: update(linked_memory_ids)
        else 新实体
            Memory->>ES: insert(vector, id, payload)
        end
    end

    Note over Memory: Phase 8: 保存消息 + 返回
    Memory->>DB: save_messages(messages, session_scope)
    Memory-->>Client: {"results": returned_memories}
```

### 4.2 各阶段要点

| 阶段 | 名称 | 核心操作 | 容错策略 |
|------|------|----------|----------|
| Phase 0 | 上下文收集 | 获取最近 10 条消息 + 解析当前消息 | - |
| Phase 1 | 已有记忆检索 | 语义搜索 top_k=10，UUID 映射为整数 | - |
| Phase 2 | LLM 事实提取 | 单次 LLM 调用，`response_format=json_object` | LLM 失败返回空列表 |
| Phase 3 | 批量嵌入 | `embed_batch` 批量向量化 | 失败回退逐条嵌入 |
| Phase 4 | CPU 处理 | 构建 metadata、lemmatize | - |
| Phase 5 | Hash 去重 | MD5 hash 对比已有 + 批次内去重 | - |
| Phase 6 | 批量持久化 | 批量 insert + 批量 history | 批量失败回退逐条 |
| Phase 7 | 实体链接 | 批量提取→批量嵌入→批量搜索→批量写入 | 整体失败仅 warning |
| Phase 8 | 保存返回 | save_messages + 构建返回值 | - |

---

## 5. search() 方法完整流程

### 5.1 混合检索三路信号融合

```mermaid
sequenceDiagram
    participant Client
    participant Memory
    participant Embedder as EmbeddingModel
    participant VS as VectorStore
    participant EE as EntityExtraction
    participant ES as EntityStore
    participant Scorer as score_and_rank
    participant Reranker as Reranker

    Client->>Memory: search(query, top_k, filters, threshold, rerank, explain)

    Note over Memory: Step 1: 查询预处理
    Memory->>Memory: lemmatize_for_bm25(query)
    Memory->>EE: extract_entities(query)
    EE-->>Memory: query_entities

    Note over Memory: Step 2: 查询嵌入
    Memory->>Embedder: embed(query, "search")
    Embedder-->>Memory: embeddings

    Note over Memory: Step 3: 语义搜索（过取）
    Memory->>VS: search(query, vectors, top_k=max(limit*4, 60), filters)
    VS-->>Memory: semantic_results

    Note over Memory: Step 4: 关键词搜索
    Memory->>VS: keyword_search(query_lemmatized, top_k, filters)
    VS-->>Memory: keyword_results

    Note over Memory: Step 5: BM25 评分归一化
    Memory->>Memory: normalize_bm25() sigmoid 归一化

    Note over Memory: Step 6: 实体增强计算
    alt query_entities 非空
        Memory->>Embedder: embed_batch(entity_texts, "search")
        Memory->>ES: search(entity_text, vectors, top_k=500, filters)
        Memory->>Memory: boost = similarity * WEIGHT * count_weight
    end

    Note over Memory: Step 7-8: 评分与排序
    Memory->>Scorer: score_and_rank(semantic, bm25, entity_boosts, threshold, top_k)
    Scorer-->>Memory: scored_results

    Note over Memory: Step 9: 可选重排序
    alt rerank == True and reranker 可用
        Memory->>Reranker: rerank(query, results, top_k)
        Reranker-->>Memory: reranked_results
    end

    Memory-->>Client: {"results": [...]}
```

### 5.2 评分公式

```
combined = (semantic_score + bm25_score + entity_boost) / max_possible

其中:
- semantic_score: 向量余弦相似度 [0, 1]
- bm25_score: sigmoid 归一化后的 BM25 分数 [0, 1]
- entity_boost: similarity * ENTITY_BOOST_WEIGHT(0.5) * memory_count_weight [0, 0.5]
- max_possible: 1.0 + (has_bm25 ? 1.0 : 0) + (has_entity ? 0.5 : 0)

阈值门控: semantic_score < threshold 的候选直接排除
```

### 5.3 BM25 自适应参数

| 查询词数 | midpoint | steepness |
|----------|----------|-----------|
| <=3 | 5.0 | 0.7 |
| 4-6 | 7.0 | 0.6 |
| 7-9 | 9.0 | 0.5 |
| 10-15 | 10.0 | 0.5 |
| >15 | 12.0 | 0.5 |

---

## 6. 其他方法流程

### 6.1 get() 流程

```mermaid
flowchart TD
    A[get memory_id] --> B[vector_store.get vector_id]
    B --> C{memory 存在?}
    C -->|否| D[返回 None]
    C -->|是| E[构建 MemoryItem]
    E --> F[提升 payload 键: user_id, agent_id, run_id, actor_id, role]
    F --> G[分离 additional_metadata]
    G --> H[返回结果字典]
```

### 6.2 update() 流程

```mermaid
sequenceDiagram
    participant Client
    participant Memory
    participant Embedder as EmbeddingModel
    participant VS as VectorStore
    participant DB as SQLiteManager
    participant ES as EntityStore

    Client->>Memory: update(memory_id, data, metadata)
    Memory->>Embedder: embed(data, "update")
    Memory->>VS: get(vector_id=memory_id)
    VS-->>Memory: existing_memory
    Memory->>VS: update(vector_id, vector, payload)
    Memory->>DB: add_history(memory_id, prev_value, data, "UPDATE")
    Memory->>ES: _remove_memory_from_entity_store(memory_id, filters)
    Memory->>ES: _link_entities_for_memory(memory_id, data, filters)
    Memory-->>Client: {"message": "Memory updated successfully!"}
```

### 6.3 delete() 流程

```mermaid
sequenceDiagram
    participant Client
    participant Memory
    participant VS as VectorStore
    participant DB as SQLiteManager
    participant ES as EntityStore

    Client->>Memory: delete(memory_id)
    Memory->>VS: get(vector_id=memory_id)
    VS-->>Memory: existing_memory
    Memory->>VS: delete(vector_id=memory_id)
    Memory->>DB: add_history(memory_id, prev_value, None, "DELETE", is_deleted=1)
    Memory->>ES: _remove_memory_from_entity_store(memory_id, filters)
    Memory-->>Client: {"message": "Memory deleted successfully!"}
```

---

## 7. Entity Store 设计

### 7.1 架构概览

```mermaid
graph TB
    subgraph 主记忆存储
        MV[VectorStore<br/>collection: mem0]
        MP[payload: data, hash, text_lemmatized,<br/>user_id, agent_id, run_id, ...]
    end

    subgraph 实体存储
        EV[EntityStore<br/>collection: mem0_entities]
        EP[payload: data, entity_type,<br/>linked_memory_ids, user_id, ...]
    end

    MV -.->|"memory_id 关联"| EP
    EP -.->|"linked_memory_ids"| MV
```

### 7.2 实体提取类型

| 类型 | 说明 | 示例 |
|------|------|------|
| PROPER | 专有名词序列 | "John Smith", "San Francisco" |
| QUOTED | 引号内文本 | "Machine Learning" |
| COMPOUND | 名词复合短语 | "machine_learning", "neural_network" |
| NOUN | 单个名词（回退） | "tennis" |

### 7.3 搜索增强：Entity Boost 计算

```mermaid
flowchart TD
    A[查询实体提取] --> B[去重 最多8个实体]
    B --> C[批量嵌入 entity_texts]
    C --> D[并发搜索实体存储<br/>top_k=500, threshold>=0.5]
    D --> E[遍历匹配实体]
    E --> F[计算 boost = similarity * 0.5 * memory_count_weight]
    F --> G[memory_count_weight = 1 / 1 + 0.001 * num_linked-1^2]
    G --> H[更新 memory_boosts<br/>取每个 memory_id 的最大 boost]
```

---

## 8. 同步 vs 异步差异

| 方面 | Memory（同步） | AsyncMemory（异步） |
|------|---------------|-------------------|
| 方法签名 | `def add(...)` | `async def add(...)` |
| I/O 调用 | 直接调用 | `await asyncio.to_thread(...)` |
| 实体 boost 并发 | `ThreadPoolExecutor(max_workers=4)` | `asyncio.Semaphore(4)` + `asyncio.gather` |
| delete_all | 逐条同步删除 | `asyncio.gather(*delete_tasks)` 并发删除 |
| 程序性记忆 | 直接调用 `self.llm.generate_response` | 支持 `llm` 参数（LangChain ChatModel） |
| reset | 同步清理 | 异步清理 + `gc.collect()` |

---

## 9. 关键设计决策

### 9.1 反幻觉设计

```mermaid
flowchart LR
    A[已有记忆 UUID] -->|"映射"| B[整数 ID: 0, 1, 2...]
    B -->|"传入 LLM"| C[LLM 返回 linked_memory_ids: 0, 2]
    C -->|"映射回"| D[真实 UUID]
```

### 9.2 Hash 去重

```mermaid
flowchart TD
    A[提取的记忆文本] --> B[计算 MD5 hash]
    B --> C{hash in existing_hashes?}
    C -->|是| D[跳过：已有重复]
    C -->|否| E{hash in seen_hashes?}
    E -->|是| F[跳过：批次内重复]
    E -->|否| G[加入 records + seen_hashes]
```

### 9.3 容错机制

```mermaid
graph TD
    subgraph 致命错误 - 中断流程
        F1[参数验证失败]
        F2[缺少必需的 entity ID]
    end

    subgraph 非致命错误 - 降级继续
        N1[LLM 提取失败 → 返回空列表]
        N2[批量嵌入失败 → 逐条嵌入]
        N3[批量插入失败 → 逐条插入]
        N4[批量历史失败 → 逐条写入]
        N5[实体链接失败 → 仅 warning]
        N6[Rerank 失败 → 使用原始结果]
    end
```

---

## 10. 工具函数详解

### 10.1 函数调用关系图

```mermaid
graph TB
    subgraph mem0/memory/utils.py
        P1[parse_messages]
        P2[parse_vision_messages]
        P3[extract_json]
        P4[remove_code_blocks]
        P5[get_fact_retrieval_messages]
        P6[ensure_json_instruction]
        P7[format_entities]
        P8[normalize_facts]
        P9[get_image_description]
        P10[process_telemetry_filters]
        P11[sanitize_relationship_for_cypher]
        P12[remove_spaces_from_entities]
    end

    subgraph 调用方
        ADD[add / _add_to_vector_store]
        SEARCH[search / _search_vector_store]
        TELEMETRY[遥测上报]
        GRAPH[图存储]
    end

    ADD --> P1
    ADD --> P2
    ADD --> P3
    ADD --> P4
    ADD --> P10
    SEARCH --> P10
    P2 --> P9
    TELEMETRY --> P10
    GRAPH --> P11
    GRAPH --> P12
```

### 10.2 各函数详解

| 函数 | 用途 | 调用链 |
|------|------|--------|
| `parse_messages(messages)` | 将消息列表转为纯文本（`system: ...\nuser: ...`） | `add()` → `_add_to_vector_store()` |
| `parse_vision_messages(messages, llm, vision_details)` | 处理图片消息，调用 LLM 生成描述 | `add()` → `parse_vision_messages()` → `get_image_description()` |
| `extract_json(text)` | 从 LLM 返回文本中提取 JSON（先匹配代码块，再找 `{}`） | `_add_to_vector_store()` → `json.loads` 失败后 |
| `remove_code_blocks(content)` | 移除代码块标记和 `<think...>` 标签 | `_add_to_vector_store()` → `json.loads()` 前 |
| `get_image_description(image_obj, llm, vision_details)` | 调用 LLM 视觉能力获取图片描述 | `parse_vision_messages()` 内部 |
| `process_telemetry_filters(filters)` | 对实体 ID 进行 MD5 哈希脱敏 | 所有公开方法的遥测上报 |
| `sanitize_relationship_for_cypher(relationship)` | 关系文本 Cypher 查询安全化 | 图存储相关 |
| `remove_spaces_from_entities(entity_list)` | 实体文本标准化（小写+下划线） | 图存储相关 |
| `normalize_facts(raw_facts)` | 规范化 LLM 提取的事实（兼容 dict 格式） | 旧版管线 |
| `format_entities(entities)` | 格式化实体三元组为文本 | 图存储相关 |
