# 记忆进化与演化模块设计文档

## 1. 模块概述

记忆进化是 A-MEM 系统的核心创新机制，使记忆能够自动关联、更新和演化。当新记忆进入系统时，系统会检索与之语义最相近的邻居记忆，由 LLM 判断是否需要进化，并执行相应的进化动作（建立连接、更新邻居），从而实现记忆网络的自我组织和持续优化。

系统提供两种进化实现：

| 版本 | 文件 | LLM 调用方式 | 特点 |
|------|------|-------------|------|
| 标准版 | `memory_layer.py` | 单次调用 + JSON Schema 约束 | 简洁高效，依赖模型 JSON 输出能力 |
| 鲁棒版 | `memory_layer_robust.py` | 3 步顺序纯文本调用 | 容错性强，适配任意 LLM 后端 |

---

## 2. 记忆进化整体流程

```mermaid
flowchart TD
    A["用户输入新记忆内容"] --> B["创建 MemoryNote<br/>（LLM 提取元数据）"]
    B --> C["process_memory 进化处理"]
    C --> D{"检索相关记忆<br/>find_related_memories"}
    D -->|"无邻居"| E["不进化，直接存储"]
    D -->|"有邻居"| F["LLM 进化决策"]
    F --> G{"should_evolve?"}
    G -->|"否"| E
    G -->|"是"| H["执行进化动作"]
    H --> I{"动作类型"}
    I -->|"strengthen"| J["建立连接 + 更新标签"]
    I -->|"update_neighbor"| K["更新邻居 context 和 tags"]
    I -->|"两者组合"| L["同时执行 J 和 K"]
    J --> M["存入 memories dict"]
    K --> M
    L --> M
    E --> N["更新 retriever 索引"]
    M --> N
    N --> O{"evo_cnt 达到阈值?"}
    O -->|"否"| P["完成"]
    O -->|"是"| Q["consolidate_memories<br/>重建检索索引"]
    Q --> P
```

---

## 3. add_note 时序图

```mermaid
sequenceDiagram
    participant User as 调用方
    participant AMS as AgenticMemorySystem
    participant MN as MemoryNote
    participant LLM as LLMController
    participant PM as process_memory
    participant Ret as SimpleEmbeddingRetriever

    User->>AMS: add_note(content, time, **kwargs)
    
    rect rgb(230, 245, 255)
        Note over AMS,MN: 步骤1: 创建记忆笔记
        AMS->>MN: MemoryNote(content, llm_controller, timestamp, ...)
        MN->>LLM: analyze_content(content) — 提取元数据
        LLM-->>MN: {keywords, context, tags}
        MN-->>AMS: 返回完整 MemoryNote 对象
    end

    rect rgb(255, 240, 230)
        Note over AMS,Ret: 步骤2: 进化处理
        AMS->>PM: process_memory(note)
        PM->>Ret: find_related_memories(note.content, k=5)
        Ret-->>PM: neighbor_memory_str, indices
        PM->>LLM: 进化决策 + 动作执行
        LLM-->>PM: 进化结果
        PM-->>AMS: (evo_label, evolved_note)
    end

    rect rgb(230, 255, 230)
        Note over AMS,Ret: 步骤3: 存储与索引
        AMS->>AMS: memories[note.id] = note
        AMS->>Ret: add_documents([content + metadata])
    end

    rect rgb(255, 230, 255)
        Note over AMS: 步骤4: 进化计数与整合
        alt evo_label == True
            AMS->>AMS: evo_cnt += 1
            alt evo_cnt % evo_threshold == 0
                AMS->>AMS: consolidate_memories()
            end
        end
    end

    AMS-->>User: note.id
```

---

## 4. 标准版 process_memory 时序图

标准版通过**单次 LLM 调用**完成进化决策与动作执行，依赖 JSON Schema 约束输出格式。

```mermaid
sequenceDiagram
    participant PM as process_memory
    participant Ret as Retriever
    participant LLM as LLMController
    participant Mem as memories dict

    PM->>PM: 输入 note
    PM->>Ret: search(note.content, k=5)
    Ret-->>PM: indices (top-5 相关记忆索引)

    PM->>PM: 构造邻居记忆文本<br/>neighbor_memory (含 index/timestamp/content/context/keywords/tags)

    rect rgb(240, 248, 255)
        Note over PM,LLM: 单次 LLM 调用 — 进化决策 + 动作
        PM->>PM: 构造 evolution_system_prompt<br/>（新记忆 context/content/keywords + 邻居记忆 + 邻居数量）
        PM->>LLM: get_completion(prompt, response_format=JSON Schema)
        
        Note right of LLM: JSON Schema 约束输出:<br/>should_evolve: boolean<br/>actions: string[]<br/>suggested_connections: integer[]<br/>tags_to_update: string[]<br/>new_context_neighborhood: string[]<br/>new_tags_neighborhood: string[][]
        
        LLM-->>PM: JSON 响应
    end

    PM->>PM: 解析 JSON 响应

    alt should_evolve == False
        PM-->>PM: return (False, note)
    else should_evolve == True
        loop 遍历 actions
            alt action == "strengthen"
                PM->>PM: note.links.extend(suggested_connections)
                PM->>PM: note.tags = tags_to_update
            else action == "update_neighbor"
                loop i in range(min(len(indices), len(new_tags_neighborhood)))
                    PM->>Mem: 更新邻居 noteslist[indices[i]].tags
                    PM->>Mem: 更新邻居 noteslist[indices[i]].context
                end
            end
        end
        PM-->>PM: return (True, note)
    end
```

### 标准版 evolution_system_prompt 结构

```
You are an AI memory evolution agent responsible for managing and evolving a knowledge base.

The new memory context:
{context}
content: {content}
keywords: {keywords}

The nearest neighbors memories:
{nearest_neighbors_memories}

Return JSON:
{
    "should_evolve": True/False,
    "actions": ["strengthen", "update_neighbor"],
    "suggested_connections": [neighbor_index, ...],
    "tags_to_update": ["tag1", ...],
    "new_context_neighborhood": ["ctx1", ...],
    "new_tags_neighborhood": [["tag1", ...], ...]
}
```

---

## 5. 鲁棒版 process_memory 时序图（3 步调用）

鲁棒版将进化过程拆分为 **3 步顺序 LLM 调用**，每步使用纯文本 prompt + 段落标记解析，不依赖 JSON Schema，适配任意 LLM 后端。

```mermaid
sequenceDiagram
    participant PM as process_memory
    participant Ret as Retriever
    participant LLM as RobustLLMController
    participant Parser as llm_text_parsers
    participant Mem as memories dict

    PM->>PM: 输入 note
    PM->>Ret: search(note.content, k=5)
    Ret-->>PM: indices

    alt indices 为空
        PM-->>PM: return (False, note)
    end

    rect rgb(255, 250, 230)
        Note over PM,Parser: Call 1 — 进化决策
        PM->>PM: 构造 EVOLUTION_DECISION_PROMPT
        PM->>LLM: get_completion(decision_prompt)
        LLM-->>PM: 纯文本响应
        PM->>Parser: parse_evolution_decision(response)
        Parser-->>PM: {decision, reason}
        
        Note right of Parser: 决策类型:<br/>NO_EVOLUTION<br/>STRENGTHEN<br/>UPDATE_NEIGHBOR<br/>STRENGTHEN_AND_UPDATE
    end

    alt decision == "NO_EVOLUTION"
        PM-->>PM: return (False, note)
    end

    PM->>PM: should_strengthen = decision ∈ {STRENGTHEN, STRENGTHEN_AND_UPDATE}
    PM->>PM: should_update = decision ∈ {UPDATE_NEIGHBOR, STRENGTHEN_AND_UPDATE}

    rect rgb(230, 255, 230)
        Note over PM,Parser: Call 2 — 加强连接详情（条件执行）
        alt should_strengthen == True
            PM->>PM: 构造 STRENGTHEN_DETAILS_PROMPT
            PM->>LLM: get_completion(strengthen_prompt)
            LLM-->>PM: 纯文本响应
            PM->>Parser: parse_strengthen_details(response)
            Parser-->>PM: {connections: [int], tags: [str]}
            PM->>PM: note.links.extend(connections)
            PM->>PM: note.tags = tags（非空时更新）
        end
    end

    rect rgb(230, 230, 255)
        Note over PM,Parser: Call 3 — 更新邻居详情（条件执行）
        alt should_update == True
            PM->>PM: 构造 UPDATE_NEIGHBORS_PROMPT
            PM->>LLM: get_completion(update_prompt)
            LLM-->>PM: 纯文本响应
            PM->>Parser: parse_update_neighbors(response, num_neighbors)
            Parser-->>PM: [{context, tags}, ...]
            loop i in range(min(len(indices), len(neighbor_updates)))
                PM->>Mem: 更新邻居 tags（非空时）
                PM->>Mem: 更新邻居 context（非空时）
            end
        end
    end

    PM-->>PM: return (True, note)

    Note over PM: 异常处理: 任何步骤失败<br/>→ logger.error + return (False, note)<br/>记忆仍被存储，但不进化
```

### 鲁棒版 3 步 Prompt 模板

| 步骤 | Prompt 常量 | 输入 | 输出格式 |
|------|------------|------|---------|
| Call 1 | `EVOLUTION_DECISION_PROMPT` | context, content, keywords, neighbors | `DECISION: <类型>`<br/>`REASON: <原因>` |
| Call 2 | `STRENGTHEN_DETAILS_PROMPT` | content, keywords, neighbors | `CONNECTIONS: 0, 2, 3`<br/>`TAGS: tag1, tag2, tag3` |
| Call 3 | `UPDATE_NEIGHBORS_PROMPT` | content, context, neighbors, count | `NEIGHBOR 0:`<br/>`CONTEXT: ...`<br/>`TAGS: ...` |

### 解析策略：JSON 优先 + 段落标记兜底

```mermaid
flowchart LR
    A["LLM 响应文本"] --> B{"尝试 JSON 解析"}
    B -->|"成功"| C["使用 JSON 结果"]
    B -->|"失败"| D["段落标记解析<br/>_extract_section()"]
    D --> E["提取 KEYWORDS/CONTEXT/TAGS<br/>等标记段"]
    E --> F["validate_analysis_result()<br/>验证与启发式修复"]
    C --> F
    F --> G["返回结构化结果"]
```

---

## 6. 进化决策状态机图

```mermaid
stateDiagram-v2
    [*] --> 检索邻居: 新记忆进入 process_memory

    检索邻居 --> 无邻居: indices 为空
    无邻居 --> [*]: return (False, note)

    检索邻居 --> 进化决策: indices 非空

    state 进化决策 {
        [*] --> LLM决策: Call 1
        LLM决策 --> NO_EVOLUTION: 决策 = NO_EVOLUTION
        LLM决策 --> STRENGTHEN: 决策 = STRENGTHEN
        LLM决策 --> UPDATE_NEIGHBOR: 决策 = UPDATE_NEIGHBOR
        LLM决策 --> STRENGTHEN_AND_UPDATE: 决策 = STRENGTHEN_AND_UPDATE
    }

    NO_EVOLUTION --> [*]: return (False, note)

    STRENGTHEN --> 执行Strengthen: should_strengthen = True
    UPDATE_NEIGHBOR --> 执行UpdateNeighbor: should_update = True
    STRENGTHEN_AND_UPDATE --> 执行Strengthen: should_strengthen = True
    STRENGTHEN_AND_UPDATE --> 执行UpdateNeighbor: should_update = True

    state 执行Strengthen {
        [*] --> Call2: STRENGTHEN_DETAILS_PROMPT
        Call2 --> 解析连接: parse_strengthen_details
        解析连接 --> 更新Links: note.links.extend(connections)
        更新Links --> 更新Tags: note.tags = tags
        更新Tags --> [*]
    }

    state 执行UpdateNeighbor {
        [*] --> Call3: UPDATE_NEIGHBORS_PROMPT
        Call3 --> 解析邻居更新: parse_update_neighbors
        解析邻居更新 --> 遍历邻居: for each neighbor
        遍历邻居 --> 更新邻居Tags: notetmp.tags = upd.tags
        更新邻居Tags --> 更新邻居Context: notetmp.context = upd.context
        更新邻居Context --> [*]
    }

    执行Strengthen --> 完成
    执行UpdateNeighbor --> 完成
    完成 --> [*]: return (True, note)
```

---

## 7. consolidate_memories 流程图

```mermaid
flowchart TD
    A["consolidate_memories() 被调用"] --> B["获取当前 retriever 的模型名称"]
    B --> C{"model.get_config_dict()<br/>是否可用?"}
    C -->|"是"| D["使用配置中的 model_name"]
    C -->|"否"| E["回退到默认<br/>'all-MiniLM-L6-v2'"]
    D --> F["创建新的 SimpleEmbeddingRetriever<br/>（完全重置索引）"]
    E --> F

    F --> G["遍历所有 memories"]
    G --> H["对每条记忆构造文档:<br/>content + ', ' + context + keywords + tags"]
    H --> I["retriever.add_documents([doc])"]
    I --> J{"还有更多记忆?"}
    J -->|"是"| G
    J -->|"否"| K["整合完成<br/>检索索引已重建"]
```

### 触发条件

```mermaid
flowchart LR
    A["add_note 完成"] --> B{"evo_label == True?"}
    B -->|"否"| C["不增加计数"]
    B -->|"是"| D["evo_cnt += 1"]
    D --> E{"evo_cnt % evo_threshold == 0?"}
    E -->|"否"| F["不触发整合"]
    E -->|"是"| G["触发 consolidate_memories"]
```

---

## 8. 进化动作详解

### 8.1 strengthen — 建立连接与标签更新

`strengthen` 动作在新记忆与相关邻居之间建立显式连接，并更新新记忆的分类标签。

```mermaid
flowchart TD
    subgraph 执行前
        N1["新记忆 Note<br/>links: []<br/>tags: [old_tag1, old_tag2]"]
        NB1["邻居记忆 #0"]
        NB2["邻居记忆 #2"]
        NB3["邻居记忆 #3"]
    end

    subgraph LLM决策
        L["suggested_connections: [0, 2, 3]<br/>tags_to_update: [new_tag1, new_tag2, new_tag3]"]
    end

    subgraph 执行后
        N2["新记忆 Note<br/>links: [0, 2, 3]<br/>tags: [new_tag1, new_tag2, new_tag3]"]
        NB1b["邻居记忆 #0<br/>（未修改）"]
        NB2b["邻居记忆 #2<br/>（未修改）"]
        NB3b["邻居记忆 #3<br/>（未修改）"]
    end

    N1 --> L --> N2
```

**代码实现（标准版）：**

```python
if action == "strengthen":
    suggest_connections = response_json["suggested_connections"]
    new_tags = response_json["tags_to_update"]
    note.links.extend(suggest_connections)  # 添加邻居索引到连接列表
    note.tags = new_tags                     # 替换标签
```

**代码实现（鲁棒版）：**

```python
# Call 2: STRENGTHEN_DETAILS_PROMPT
strengthen = parse_strengthen_details(response)
note.links.extend(strengthen["connections"])  # 添加邻居索引
if strengthen["tags"]:
    note.tags = strengthen["tags"]             # 非空时替换标签
```

### 8.2 update_neighbor — 更新邻居记忆

`update_neighbor` 动作根据新记忆带来的新理解，更新邻居记忆的上下文描述和分类标签。

```mermaid
flowchart TD
    subgraph 执行前
        N["新记忆 Note<br/>（触发更新）"]
        NB1["邻居 #0<br/>context: '旧上下文A'<br/>tags: [old_a, old_b]"]
        NB2["邻居 #1<br/>context: '旧上下文B'<br/>tags: [old_c, old_d]"]
    end

    subgraph LLM决策
        L["new_context_neighborhood:<br/>['更新后上下文A', '更新后上下文B']<br/><br/>new_tags_neighborhood:<br/>[['new_a', 'new_b'], ['new_c']]"]
    end

    subgraph 执行后
        N2["新记忆 Note<br/>（未修改）"]
        NB1b["邻居 #0<br/>context: '更新后上下文A'<br/>tags: [new_a, new_b]"]
        NB2b["邻居 #1<br/>context: '更新后上下文B'<br/>tags: [new_c]"]
    end

    N --> L --> N2
```

**代码实现（标准版）：**

```python
elif action == "update_neighbor":
    new_context_neighborhood = response_json["new_context_neighborhood"]
    new_tags_neighborhood = response_json["new_tags_neighborhood"]
    noteslist = list(self.memories.values())
    notes_id = list(self.memories.keys())
    for i in range(min(len(indices), len(new_tags_neighborhood))):
        tag = new_tags_neighborhood[i]
        context = new_context_neighborhood[i] if i < len(new_context_neighborhood) else noteslist[indices[i]].context
        notetmp = noteslist[indices[i]]
        notetmp.tags = tag
        notetmp.context = context
        self.memories[notes_id[memorytmp_idx]] = notetmp
```

**代码实现（鲁棒版）：**

```python
# Call 3: UPDATE_NEIGHBORS_PROMPT
neighbor_updates = parse_update_neighbors(response, len(indices))
noteslist = list(self.memories.values())
notes_id = list(self.memories.keys())
for i in range(min(len(indices), len(neighbor_updates))):
    upd = neighbor_updates[i]
    memorytmp_idx = indices[i]
    if memorytmp_idx >= len(noteslist):
        continue
    notetmp = noteslist[memorytmp_idx]
    if upd["tags"]:
        notetmp.tags = upd["tags"]
    if upd["context"]:
        notetmp.context = upd["context"]
    self.memories[notes_id[memorytmp_idx]] = notetmp
```

### 8.3 动作组合执行

`strengthen` 和 `update_neighbor` 可以组合执行（actions 同时包含两者，或鲁棒版决策为 `STRENGTHEN_AND_UPDATE`）：

```mermaid
flowchart LR
    A["LLM 返回<br/>actions: [strengthen, update_neighbor]"] --> B["先执行 strengthen<br/>更新 note.links 和 note.tags"]
    B --> C["再执行 update_neighbor<br/>更新邻居 context 和 tags"]
    C --> D["进化完成"]
```

---

## 9. 进化计数与整合触发机制

```mermaid
flowchart TD
    A["add_note() 被调用"] --> B["process_memory() 返回 evo_label"]
    B --> C{"evo_label == True?"}
    C -->|"False"| D["记忆已存储<br/>不增加进化计数"]
    C -->|"True"| E["evo_cnt += 1"]
    E --> F{"evo_cnt % evo_threshold == 0?"}
    F -->|"False"| G["记忆已存储<br/>等待下次进化"]
    F -->|"True"| H["触发 consolidate_memories()"]
    
    H --> I["重建 SimpleEmbeddingRetriever"]
    I --> J["用所有记忆的 content + metadata<br/>重新编码嵌入向量"]
    J --> K["检索索引完全重建<br/>确保后续检索准确性"]

    style H fill:#ff9999,stroke:#cc0000
    style K fill:#99ff99,stroke:#009900
```

### 参数说明

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `evo_cnt` | 0 | 进化计数器，每次成功进化 +1 |
| `evo_threshold` | 100 | 整合阈值，每 N 次进化触发一次索引重建 |

### 为什么需要 consolidate_memories？

在多次进化过程中，记忆的 `context`、`tags`、`keywords` 等元数据会被不断更新，但 retriever 中缓存的嵌入向量仍基于旧数据。当进化次数累积到阈值时，系统需要重建索引，确保检索结果反映记忆的最新状态。

---

## 10. 标准版与鲁棒版对比

```mermaid
flowchart LR
    subgraph 标准版
        direction TB
        S1["单次 LLM 调用"] --> S2["JSON Schema 约束输出"]
        S2 --> S3["一次返回所有决策与动作"]
    end

    subgraph 鲁棒版
        direction TB
        R1["Call 1: 进化决策"] --> R2{"需要 strengthen?"}
        R2 -->|"是"| R3["Call 2: 连接详情"]
        R2 -->|"否"| R4{"需要 update?"}
        R3 --> R4
        R4 -->|"是"| R5["Call 3: 邻居更新"]
        R4 -->|"否"| R6["完成"]
        R5 --> R6
    end
```

| 对比维度 | 标准版 | 鲁棒版 |
|---------|--------|--------|
| LLM 调用次数 | 1 次 | 1-3 次（条件执行） |
| 输出格式 | JSON Schema（严格约束） | 纯文本 + 段落标记 |
| 解析策略 | JSON 解析 | JSON 优先 + 段落标记兜底 |
| 容错能力 | JSON 解析失败则不进化 | 重试机制 + 启发式修复 + 优雅降级 |
| 适配范围 | 支持 JSON Schema 的模型 | 任意 LLM 后端 |
| 失败处理 | `return (False, note)` | `logger.error` + `return (False, note)` |
| 重试机制 | 无 | `@retry_llm_call(max_retries=2)` |
| 连接检查 | 无 | `check_connectivity()` 可选 |

---

## 11. 关键数据结构

### MemoryNote 字段

| 字段 | 类型 | 说明 | 进化影响 |
|------|------|------|---------|
| `id` | str | UUID 唯一标识 | - |
| `content` | str | 记忆原文 | - |
| `keywords` | List[str] | LLM 提取的关键词 | - |
| `context` | str | 上下文描述 | update_neighbor 可修改 |
| `tags` | List[str] | 分类标签 | strengthen / update_neighbor 可修改 |
| `links` | List[int] | 连接的邻居记忆索引 | strengthen 可修改 |
| `importance_score` | float | 重要性评分（默认 1.0） | - |
| `retrieval_count` | int | 被检索次数（默认 0） | - |
| `timestamp` | str | 创建时间 | - |
| `last_accessed` | str | 最后访问时间 | - |
| `evolution_history` | List | 进化历史记录 | - |
| `category` | str | 分类（默认 "Uncategorized"） | - |

### AgenticMemorySystem 核心属性

| 属性 | 类型 | 说明 |
|------|------|------|
| `memories` | Dict[str, MemoryNote] | ID → 记忆笔记映射 |
| `retriever` | SimpleEmbeddingRetriever | 嵌入向量检索器 |
| `llm_controller` | LLMController / RobustLLMController | LLM 控制器 |
| `evo_cnt` | int | 进化计数器 |
| `evo_threshold` | int | 整合阈值 |
