# 核心模块设计: Text Memory Core (文本记忆核心)

> 源码路径: `simplemem/core/`, `simplemem/text/`

## 1. 模块概述

Text Memory Core 是 SimpleMem 的基础引擎，实现了论文中的三阶段流水线：
1. **语义结构化压缩** (Section 3.1): 将对话压缩为紧凑的记忆单元
2. **在线语义合成** (Section 3.2): 写入时合并冗余
3. **意图感知检索规划** (Section 3.3): 推断查询意图，多视图检索

## 2. 模块架构

```mermaid
graph TB
    subgraph "入口"
        System["SimpleMemSystem<br/>(main.py)"]
    end

    subgraph "三阶段流水线"
        MB["MemoryBuilder<br/>(Stage 1 + 2)"]
        HR["HybridRetriever<br/>(Stage 3)"]
        AG["AnswerGenerator<br/>(答案合成)"]
    end

    subgraph "数据层"
        VS["VectorStore<br/>(LanceDB)"]
        ME["MemoryEntry<br/>(数据模型)"]
        Dialogue["Dialogue<br/>(输入模型)"]
    end

    subgraph "工具层"
        LLM["LLMClient<br/>(OpenAI兼容)"]
        EMB["EmbeddingModel<br/>(Qwen3/OpenAI)"]
        CFG["Settings<br/>(配置)"]
    end

    System --> MB
    System --> HR
    System --> AG

    MB --> VS
    MB --> LLM
    HR --> VS
    HR --> LLM
    AG --> LLM

    VS --> ME
    VS --> EMB
    MB --> Dialogue
    MB --> CFG
    HR --> CFG
```

## 3. 核心类设计

### 3.1 SimpleMemSystem (系统入口)

**职责**: 组装三大模块，提供统一的 `add_dialogue` / `ask` 接口

```mermaid
classDiagram
    class SimpleMemSystem {
        +llm_client: LLMClient
        +embedding_model: EmbeddingModel
        +vector_store: VectorStore
        +memory_builder: MemoryBuilder
        +hybrid_retriever: HybridRetriever
        +answer_generator: AnswerGenerator
        +add_dialogue(speaker, content, timestamp)
        +add_dialogues(dialogues)
        +finalize()
        +ask(question) str
        +get_all_memories() List~MemoryEntry~
    }
```

**初始化流程**:

```mermaid
sequenceDiagram
    participant User as 用户
    participant Sys as SimpleMemSystem
    participant LLM as LLMClient
    participant EMB as EmbeddingModel
    participant VS as VectorStore
    participant MB as MemoryBuilder
    participant HR as HybridRetriever
    participant AG as AnswerGenerator

    User->>Sys: SimpleMemSystem(api_key, model, ...)
    Sys->>LLM: 初始化 (api_key, model, base_url)
    Sys->>EMB: 初始化 (默认模型)
    Sys->>VS: 初始化 (db_path, embedding_model)
    Sys->>MB: 初始化 (llm_client, vector_store)
    Sys->>HR: 初始化 (llm_client, vector_store)
    Sys->>AG: 初始化 (llm_client)
    Sys-->>User: 系统就绪
```

### 3.2 MemoryBuilder (记忆构建器)

**职责**: Stage 1 (语义结构化压缩) + Stage 2 (在线语义合成)

```mermaid
classDiagram
    class MemoryBuilder {
        +llm_client: LLMClient
        +vector_store: VectorStore
        +window_size: int
        +overlap_size: int
        +step_size: int
        +enable_parallel_processing: bool
        +max_parallel_workers: int
        +dialogue_buffer: List~Dialogue~
        +processed_count: int
        +previous_entries: List~MemoryEntry~
        +add_dialogue(dialogue, auto_process)
        +add_dialogues(dialogues, auto_process)
        +add_dialogues_parallel(dialogues)
        +process_window()
        +process_remaining()
        -_generate_memory_entries(dialogues) List~MemoryEntry~
        -_build_extraction_prompt(text, ids, context) str
        -_parse_llm_response(response, ids) List~MemoryEntry~
        -_process_windows_parallel(windows)
    }
```

**滑动窗口处理时序**:

```mermaid
sequenceDiagram
    participant Buf as 对话缓冲区
    participant MB as MemoryBuilder
    participant LLM as LLMClient
    participant VS as VectorStore

    Note over Buf: 对话持续添加到缓冲区
    Buf->>MB: add_dialogue(d1)
    Buf->>MB: add_dialogue(d2)
    Buf->>MB: ... (达到 window_size)

    MB->>MB: process_window()
    MB->>MB: 提取窗口: buffer[0:W]
    MB->>MB: 剩余: buffer[step_size:]

    MB->>LLM: 生成记忆条目<br/>(含前一窗口上下文)
    LLM-->>MB: JSON 格式记忆条目

    MB->>MB: _parse_llm_response()
    MB->>VS: add_entries(entries)
    MB->>MB: previous_entries = entries

    Note over Buf: 继续处理下一个窗口...
```

**并行处理流程**:

```mermaid
flowchart TB
    Input["批量对话输入"] --> Buffer["添加到缓冲区"]
    Buffer --> Group["按 step_size 分组为窗口"]
    Group --> Pool["ThreadPoolExecutor<br/>(max_workers)"]

    Pool --> W1["Worker 1: 窗口1"]
    Pool --> W2["Worker 2: 窗口2"]
    Pool --> W3["Worker 3: 窗口3"]

    W1 --> Collect["收集所有结果"]
    W2 --> Collect
    W3 --> Collect

    Collect --> Batch["批量写入 VectorStore"]

    Group -->|异常| Fallback["降级为串行处理"]
    Fallback --> Serial["逐窗口 process_window()"]
```

### 3.3 HybridRetriever (混合检索器)

**职责**: Stage 3 (意图感知检索规划)，实现多视图检索与反思

```mermaid
classDiagram
    class HybridRetriever {
        +llm_client: LLMClient
        +vector_store: VectorStore
        +semantic_top_k: int
        +keyword_top_k: int
        +structured_top_k: int
        +enable_planning: bool
        +enable_reflection: bool
        +max_reflection_rounds: int
        +enable_parallel_retrieval: bool
        +max_retrieval_workers: int
        +retrieve(query, enable_reflection) List~MemoryEntry~
        -_retrieve_with_planning(query, enable_reflection)
        -_analyze_information_requirements(query) Dict
        -_generate_targeted_queries(query, plan) List~str~
        -_semantic_search(query) List~MemoryEntry~
        -_keyword_search(query, analysis) List~MemoryEntry~
        -_structured_search(analysis) List~MemoryEntry~
        -_merge_and_deduplicate_entries(entries) List~MemoryEntry~
        -_retrieve_with_intelligent_reflection(query, results, plan)
        -_analyze_information_completeness(query, results, plan) str
        -_generate_missing_info_queries(query, results, plan) List~str~
    }
```

**检索规划详细时序**:

```mermaid
sequenceDiagram
    participant Q as 查询
    participant Plan as 检索规划器
    participant Sem as 语义层
    participant Lex as 词汇层
    participant Sym as 符号层
    participant Merge as 合并器
    participant Reflect as 反思引擎

    Q->>Plan: retrieve(query)

    alt enable_planning = True
        Plan->>Plan: _analyze_information_requirements(query)
        Note over Plan: 识别问题类型、关键实体、<br/>所需信息类型、关系
        Plan->>Plan: _generate_targeted_queries(query, plan)
        Note over Plan: 生成1-4个定向子查询

        loop 每个子查询 (并行/串行)
            Plan->>Sem: _semantic_search(sub_query)
            Sem-->>Plan: R_sem
        end
    else enable_planning = False
        Plan->>Sem: _semantic_search(query)
        Sem-->>Plan: R_sem
    end

    Plan->>Plan: _analyze_query(query)
    Plan->>Lex: _keyword_search(keywords)
    Lex-->>Plan: R_lex
    Plan->>Sym: _structured_search(persons, time, location, entities)
    Sym-->>Plan: R_sym

    Plan->>Merge: _merge_and_deduplicate(R_sem + R_lex + R_sym)

    alt enable_reflection = True
        Merge->>Reflect: _retrieve_with_intelligent_reflection()
        loop 每轮反思 (max_reflection_rounds)
            Reflect->>Reflect: _analyze_information_completeness()
            alt 信息不完整
                Reflect->>Reflect: _generate_missing_info_queries()
                Reflect->>Sem: 补充检索
                Reflect->>Merge: 合并新结果
            else 信息完整
                Note over Reflect: 提前退出
            end
        end
    end

    Merge-->>Q: 返回合并后的记忆条目
```

### 3.4 VectorStore (向量存储)

**职责**: 三层索引的存储与检索

```mermaid
classDiagram
    class VectorStore {
        +db: lancedb.DBConnection
        +table: lancedb.Table
        +embedding_model: EmbeddingModel
        +table_name: str
        +_fts_initialized: bool
        +_is_cloud_storage: bool
        +add_entries(entries)
        +semantic_search(query, top_k) List~MemoryEntry~
        +keyword_search(keywords, top_k) List~MemoryEntry~
        +structured_search(persons, timestamp_range, location, entities, top_k) List~MemoryEntry~
        +get_all_entries() List~MemoryEntry~
        +optimize()
        +clear()
        -_init_table()
        -_init_fts_index()
        -_results_to_entries(results) List~MemoryEntry~
    }
```

**三层索引检索示意**:

```mermaid
flowchart LR
    subgraph "语义层 (Semantic)"
        Q1["查询文本"] --> EMB["EmbeddingModel.encode()"]
        EMB --> VEC["查询向量"]
        VEC --> LDB["LanceDB.search()<br/>cosine similarity"]
        LDB --> R1["Top-K 语义结果"]
    end

    subgraph "词汇层 (Lexical)"
        Q2["关键词列表"] --> FTS["Tantivy FTS /<br/>LanceDB Native FTS"]
        FTS --> R2["Top-K BM25 结果"]
    end

    subgraph "符号层 (Symbolic)"
        Q3["persons, time,<br/>location, entities"] --> SQL["SQL WHERE 子句"]
        SQL --> R3["Top-K 结构化结果"]
    end

    R1 --> MERGE["合并去重<br/>(按 entry_id)"]
    R2 --> MERGE
    R3 --> MERGE
```

### 3.5 AnswerGenerator (答案生成器)

**职责**: 基于检索上下文生成简洁答案

```mermaid
sequenceDiagram
    participant Q as 查询
    participant AG as AnswerGenerator
    participant LLM as LLMClient

    Q->>AG: generate_answer(question, contexts)
    AG->>AG: _format_contexts(contexts)<br/>(格式化为可读文本)

    AG->>LLM: chat_completion(system + user prompt)
    Note over AG,LLM: Prompt 要求:<br/>1. 先推理<br/>2. 简洁答案<br/>3. 仅基于上下文<br/>4. 日期格式 DD Month YYYY<br/>5. JSON 输出

    LLM-->>AG: JSON 响应
    AG->>AG: extract_json(response)
    AG->>AG: 提取 answer 字段

    alt 解析失败
        AG->>LLM: 重试 (最多3次)
    end

    AG-->>Q: 返回答案字符串
```

## 4. 数据模型

### 4.1 MemoryEntry

```mermaid
classDiagram
    class MemoryEntry {
        +entry_id: str (UUID)
        +lossless_restatement: str
        +keywords: List~str~
        +timestamp: Optional~str~
        +location: Optional~str~
        +persons: List~str~
        +entities: List~str~
        +topic: Optional~str~
    }

    class Dialogue {
        +dialogue_id: int
        +speaker: str
        +content: str
        +timestamp: Optional~str~
        +__str__() str
    }

    Dialogue -->|"压缩转换"| MemoryEntry : MemoryBuilder
```

**索引映射**:

| MemoryEntry 字段 | 索引层 | 检索方式 |
|:--|:--|:--|
| lossless_restatement | 语义层 | 向量相似度 (cos) |
| lossless_restatement | 词汇层 | BM25 全文搜索 |
| keywords | 词汇层 | BM25 关键词匹配 |
| timestamp | 符号层 | 时间范围过滤 |
| location | 符号层 | LIKE 模糊匹配 |
| persons | 符号层 | array_has_any |
| entities | 符号层 | array_has_any |
| topic | 符号层 | 精确匹配 |

## 5. 关键交互流程

### 5.1 完整问答流程

```mermaid
sequenceDiagram
    participant User as 用户
    participant Sys as SimpleMemSystem
    participant MB as MemoryBuilder
    participant HR as HybridRetriever
    participant AG as AnswerGenerator
    participant VS as VectorStore
    participant LLM as LLMClient

    Note over User,LLM: === 记忆构建阶段 ===

    User->>Sys: add_dialogue("Alice", "Meet at 2pm", timestamp)
    Sys->>MB: add_dialogue(dialogue)
    MB->>MB: 缓冲区添加

    Note over MB: 当缓冲区达到 window_size

    MB->>LLM: 生成记忆条目 (含去重上下文)
    LLM-->>MB: JSON 记忆条目
    MB->>VS: add_entries(entries)
    MB->>MB: 更新 previous_entries

    User->>Sys: finalize()
    Sys->>MB: process_remaining()

    Note over User,LLM: === 查询阶段 ===

    User->>Sys: ask("When will they meet?")
    Sys->>HR: retrieve(question)

    HR->>LLM: 分析信息需求
    LLM-->>HR: 信息需求计划
    HR->>LLM: 生成定向子查询
    LLM-->>HR: 子查询列表

    par 并行检索
        HR->>VS: semantic_search(sub_query)
        VS-->>HR: R_sem
    and
        HR->>VS: keyword_search(keywords)
        VS-->>HR: R_lex
    and
        HR->>VS: structured_search(meta)
        VS-->>HR: R_sym
    end

    HR->>HR: 合并去重

    HR->>LLM: 检查信息充分性
    LLM-->>HR: "insufficient" / "complete"

    alt 信息不足
        HR->>LLM: 生成补充查询
        HR->>VS: 补充检索
        HR->>HR: 合并新结果
    end

    HR-->>Sys: contexts (List[MemoryEntry])
    Sys->>AG: generate_answer(question, contexts)

    AG->>LLM: 生成答案 (含推理过程)
    LLM-->>AG: JSON {reasoning, answer}
    AG-->>Sys: 答案字符串
    Sys-->>User: "16 November 2025 at 2:00 PM"
```

## 6. 配置参数

| 参数 | 默认值 | 描述 |
|:--|:--|:--|
| WINDOW_SIZE | 配置文件 | 滑动窗口大小 |
| OVERLAP_SIZE | 0 | 窗口重叠对话数 |
| SEMANTIC_TOP_K | 配置文件 | 语义检索 top-k |
| KEYWORD_TOP_K | 配置文件 | 关键词检索 top-k |
| STRUCTURED_TOP_K | 配置文件 | 结构化检索 top-k |
| ENABLE_PLANNING | True | 启用检索规划 |
| ENABLE_REFLECTION | True | 启用反思检索 |
| MAX_REFLECTION_ROUNDS | 2 | 最大反思轮数 |
| ENABLE_PARALLEL_PROCESSING | True | 启用并行记忆构建 |
| MAX_PARALLEL_WORKERS | 4 | 最大并行工作线程 |
| ENABLE_PARALLEL_RETRIEVAL | True | 启用并行检索 |
| MAX_RETRIEVAL_WORKERS | 3 | 最大检索工作线程 |
| USE_JSON_FORMAT | 配置文件 | LLM 输出 JSON 格式 |

## 7. 错误处理

| 场景 | 处理策略 |
|:--|:--|
| LLM 响应解析失败 | 最多重试 3 次，每次重新调用 LLM |
| 并行处理异常 | 自动降级为串行处理，恢复缓冲区状态 |
| 向量搜索异常 | 返回空列表，打印错误信息 |
| 空检索结果 | 答案生成器返回 "No relevant information found" |
| 时间表达式解析失败 | 跳过时间范围过滤 |
