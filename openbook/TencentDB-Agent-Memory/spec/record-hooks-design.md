# Record + Hooks 模块设计文档

## 1. 模块概述

Record + Hooks 是 TencentDB-Agent-Memory 系统中负责**对话捕获、记忆提取、去重写入和智能召回**的核心模块。它实现了从原始对话到结构化记忆的完整生命周期管理，以及从用户输入到上下文注入的实时召回链路。

### 核心职责

| 子模块 | 文件 | 职责 |
|--------|------|------|
| L0 对话记录器 | `l0-recorder.ts` | 原始对话的增量捕获、去重、清洗和 JSONL 持久化 |
| L1 记忆提取编排器 | `l1-extractor.ts` | 编排完整的 L1 提取流水线：质量门控 → 场景切分 → LLM 提取 → 类型归一化 → 批量去重 → 持久化写入 |
| L1 记忆持久化写入器 | `l1-writer.ts` | 根据 DedupDecision 执行 store/update/merge/skip，双写 JSONL + 向量库 |
| L1 记忆读取器 | `l1-reader.ts` | 双路径读取：SQLite 优先 / JSONL 回退 |
| L1 批量冲突检测器 | `l1-dedup.ts` | 两阶段去重：候选召回（向量/FTS）→ 批量 LLM 判决，三级降级策略 |
| 自动捕获钩子 | `auto-capture.ts` | agent_end 触发：L0 写入 + 向量索引 + 调度通知 |
| 自动召回钩子 | `auto-recall.ts` | before_prompt_build 触发：多策略搜索 + L3 人设 + L2 场景导航 + 上下文注入 |
| 提取提示词 | `prompts/l1-extraction.ts` | 场景切分 + 记忆提取的 System/User Prompt 模板 |
| 去重提示词 | `prompts/l1-dedup.ts` | 批量冲突检测的 System/User Prompt 模板 |

### 关键数据流概览

```mermaid
flowchart LR
    subgraph Hooks["Hooks 钩子层"]
        AC["auto-capture<br/>(agent_end)"]
        AR["auto-recall<br/>(before_prompt_build)"]
    end

    subgraph Record["Record 记录层"]
        L0["L0 Recorder"]
        L1E["L1 Extractor"]
        L1W["L1 Writer"]
        L1R["L1 Reader"]
        L1D["L1 Dedup"]
    end

    subgraph Storage["存储层"]
        JSONL["JSONL 文件"]
        VDB["VectorStore<br/>(SQLite/TCVDB)"]
    end

    subgraph External["外部服务"]
        LLM["LLM API"]
        EMB["Embedding API"]
    end

    AC -->|"原始消息"| L0
    L0 -->|"JSONL 追加"| JSONL
    L0 -->|"向量索引"| VDB
    AC -->|"调度通知"| Scheduler["Pipeline Manager"]

    Scheduler -->|"触发提取"| L1E
    L1E -->|"LLM 提取"| LLM
    L1E -->|"去重请求"| L1D
    L1D -->|"候选召回"| VDB
    L1D -->|"LLM 判决"| LLM
    L1D -->|"去重决策"| L1E
    L1E -->|"写入指令"| L1W
    L1W -->|"双写"| JSONL
    L1W -->|"双写"| VDB

    AR -->|"搜索请求"| L1R
    L1R -->|"SQLite 查询"| VDB
    L1R -->|"JSONL 回退"| JSONL
    AR -->|"上下文注入"| Agent["Agent Context"]
```

---

## 2. 架构设计

```mermaid
graph TB
    subgraph AgentFramework["Agent 框架"]
        HookEnd["agent_end 钩子"]
        HookBPB["before_prompt_build 钩子"]
    end

    subgraph CaptureHook["自动捕获钩子"]
        AC_Main["performAutoCapture()"]
        AC_L0["L0 JSONL 写入"]
        AC_Vec["L0 向量索引"]
        AC_Sched["调度通知"]
        AC_CP["CheckpointManager<br/>(原子游标)"]
    end

    subgraph RecallHook["自动召回钩子"]
        AR_Main["performAutoRecall()"]
        AR_Search["多策略搜索"]
        AR_Persona["L3 人设加载"]
        AR_Scene["L2 场景导航"]
        AR_Inject["上下文分割注入"]
    end

    subgraph L0Layer["L0 对话层"]
        L0_Rec["recordConversation()"]
        L0_Slice["位置切片去重"]
        L0_Replace["污染消息替换"]
        L0_Sanitize["清洗过滤"]
        L0_Write["JSONL 追加写入"]
        L0_Read["多种读取函数"]
    end

    subgraph L1Layer["L1 记忆层"]
        L1_Ext["extractL1Memories()"]
        L1_QG["质量门控"]
        L1_Split["背景/新消息切分"]
        L1_LLM["LLM 场景切分+提取"]
        L1_Norm["类型归一化"]
        L1_Dedup["batchDedup()"]
        L1_Write["writeMemory()"]
    end

    subgraph DedupEngine["去重引擎"]
        DD_Vec["向量候选召回"]
        DD_FTS["FTS5 BM25 召回"]
        DD_LLM["批量 LLM 判决"]
    end

    subgraph StorageLayer["存储层"]
        S_JSONL["JSONL 文件<br/>conversations/YYYY-MM-DD.jsonl<br/>records/YYYY-MM-DD.jsonl"]
        S_VDB["VectorStore<br/>(SQLite vec0 / TCVDB)"]
        S_FTS["FTS5 全文索引"]
        S_Persona["persona.md"]
        S_Scene["scene-index.json"]
    end

    HookEnd --> AC_Main
    AC_Main --> AC_CP
    AC_CP --> L0_Rec
    L0_Rec --> L0_Slice --> L0_Replace --> L0_Sanitize --> L0_Write
    AC_Main --> AC_Vec
    AC_Main --> AC_Sched

    AC_Sched -.->|"异步触发"| L1_Ext
    L1_Ext --> L1_QG --> L1_Split --> L1_LLM --> L1_Norm --> L1_Dedup --> L1_Write
    L1_Dedup --> DD_Vec
    L1_Dedup --> DD_FTS
    L1_Dedup --> DD_LLM
    L1_Write --> S_JSONL
    L1_Write --> S_VDB

    HookBPB --> AR_Main
    AR_Main --> AR_Search
    AR_Search --> S_VDB
    AR_Search --> S_FTS
    AR_Main --> AR_Persona
    AR_Persona --> S_Persona
    AR_Main --> AR_Scene
    AR_Scene --> S_Scene
    AR_Main --> AR_Inject

    L0_Write --> S_JSONL
    AC_Vec --> S_VDB
    L0_Read --> S_JSONL
```

---

## 3. L0 对话捕获流程

L0 捕获由 `auto-capture` 钩子在 `agent_end` 事件触发，负责将原始对话消息增量写入 JSONL 文件和向量索引。

```mermaid
sequenceDiagram
    participant Agent as Agent 框架
    participant Hook as auto-capture
    participant CP as CheckpointManager
    participant L0 as l0-recorder
    participant FS as JSONL 文件
    participant VS as VectorStore
    participant EMB as EmbeddingService
    participant Sched as Pipeline Manager

    Agent->>Hook: agent_end(messages, sessionKey, ...)

    rect rgb(240, 248, 255)
        Note over Hook,CP: Step 1+2: 原子捕获（文件锁保护）
        Hook->>CP: captureAtomically(sessionKey, callback)
        CP->>CP: 获取文件锁
        CP->>CP: 读取当前游标 afterTimestamp
        CP-->>Hook: callback(afterTimestamp)

        Hook->>L0: recordConversation(rawMessages, afterTimestamp, ...)
        
        Note over L0: Step 1: 位置切片
        L0->>L0: 根据 originalUserMessageCount 切片<br/>仅保留新增消息

        Note over L0: Step 1.5: 时间游标过滤
        L0->>L0: 过滤 timestamp > afterTimestamp 的消息

        Note over L0: Step 2: 替换被污染用户消息
        L0->>L0: 用 originalUserText 替换<br/>被 prependContext 污染的消息

        Note over L0: Step 3: 清洗过滤
        L0->>L0: sanitizeText() + stripCodeBlocks()<br/>+ shouldCaptureL0()

        Note over L0: Step 4: JSONL 追加写入
        L0->>FS: appendFile(conversations/YYYY-MM-DD.jsonl)
        L0-->>Hook: filteredMessages[]

        Hook-->>CP: {maxTimestamp, messageCount}
        CP->>CP: 更新游标 → maxTimestamp
        CP->>CP: 释放文件锁
    end

    rect rgb(255, 248, 240)
        Note over Hook,EMB: Step 1.5: L0 向量索引
        alt Store 支持延迟嵌入 (SQLite)
            Hook->>VS: upsertL0(record, undefined)<br/>仅写元数据+FTS
            Hook->>EMB: [后台异步] embedBatch(texts)
            EMB-->>VS: [后台异步] updateL0Embedding(id, vec)
        else Store 需同步嵌入 (VDB)
            Hook->>EMB: embed(content)
            EMB-->>Hook: embedding vector
            Hook->>VS: upsertL0(record, embedding)
        end
    end

    rect rgb(240, 255, 240)
        Note over Hook,Sched: Step 3: 调度通知
        Hook->>Sched: notifyConversation(sessionKey, [])
    end
```

### 增量去重双重保障

| 层级 | 机制 | 适用场景 |
|------|------|----------|
| **Layer 1: 位置切片** | `originalUserMessageCount` 缓存于 `before_prompt_build`，切片 `rawMessages[originalUserMessageCount:]` | 免疫网关重启后的时间戳漂移 |
| **Layer 2: 时间游标** | `afterTimestamp` 过滤 `timestamp > cursor` | 位置切片不可用时的回退（缓存过期、进程重启） |

### 污染消息替换策略

框架在 `before_prompt_build` 之后将 `prependContext` 注入用户消息，导致 `rawMessages` 中的用户消息被污染。替换策略：

1. 位置切片激活时：`slicedMessages[0]` 即被污染的用户消息
2. 否则回退到 `rawMessages[originalUserMessageCount]`
3. 匹配时间戳后在 `extracted` 中替换为 `originalUserText`
4. 匹配失败时，`sanitizeText()` 作为安全兜底

---

## 4. L1 记忆提取流程

L1 提取由 Pipeline Manager 异步触发，编排完整的记忆提取流水线。

```mermaid
sequenceDiagram
    participant Sched as Pipeline Manager
    participant Ext as l1-extractor
    participant L0R as l0-reader
    participant LLM as LLM API
    participant Dedup as l1-dedup
    participant Writer as l1-writer
    participant VS as VectorStore
    participant FS as JSONL 文件
    participant Metrics as Reporter

    Sched->>Ext: extractL1Memories(messages, sessionKey, ...)

    rect rgb(255, 240, 240)
        Note over Ext: Step 0: 质量门控
        Ext->>Ext: shouldExtractL1(content)<br/>过滤短消息/符号/注入
        Note right of Ext: L0 捕获一切<br/>L1 严格过滤
    end

    rect rgb(240, 240, 255)
        Note over Ext: Step 1: 消息切分
        Ext->>Ext: newMessages = qualified.slice(-maxNewMessages)
        Ext->>Ext: backgroundMessages = qualified.slice(bgStart, bgEnd)
    end

    rect rgb(240, 255, 240)
        Note over Ext,LLM: Step 2: LLM 场景切分 + 记忆提取
        Ext->>Ext: formatExtractionPrompt(newMsgs, bgMsgs, previousSceneName)
        Ext->>LLM: System: EXTRACT_MEMORIES_SYSTEM_PROMPT<br/>User: 格式化提示词
        LLM-->>Ext: JSON: [{scene_name, message_ids, memories}]
        Ext->>Ext: parseExtractionResult(raw)
        Ext->>Ext: normalizeType():<br/>episode→episodic, preference→persona
    end

    rect rgb(255, 255, 240)
        Note over Ext,Dedup: Step 3: 批量去重
        Ext->>Ext: generateMemoryId() 分配临时 ID
        Ext->>Dedup: batchDedup(memoriesWithIds, ...)

        Dedup->>Dedup: Phase 1: 候选召回
        Dedup->>VS: 向量搜索 / FTS5 搜索
        VS-->>Dedup: 候选记忆列表

        Dedup->>LLM: Phase 2: 批量 LLM 判决
        LLM-->>Dedup: [{record_id, action, target_ids, merged_content, ...}]

        Dedup-->>Ext: DedupDecision[]
    end

    rect rgb(240, 248, 255)
        Note over Ext,Writer: Step 4: 持久化写入
        loop 每条记忆 + 对应决策
            alt action = "store"
                Ext->>Writer: writeMemory(memory, {action: "store"})
                Writer->>FS: appendFile(records/YYYY-MM-DD.jsonl)
                Writer->>VS: upsertL1(record, embedding)
            else action = "update"
                Ext->>Writer: writeMemory(memory, {action: "update", target_ids})
                Writer->>VS: deleteL1Batch(target_ids)
                Writer->>FS: appendFile(更新后的记录)
                Writer->>VS: upsertL1(newRecord, embedding)
            else action = "merge"
                Ext->>Writer: writeMemory(memory, {action: "merge", target_ids, merged_content})
                Writer->>VS: deleteL1Batch(target_ids)
                Writer->>FS: appendFile(合并后的记录)
                Writer->>VS: upsertL1(mergedRecord, embedding)
            else action = "skip"
                Note over Writer: 不执行任何操作
            end
        end
    end

    rect rgb(248, 240, 255)
        Note over Ext,Metrics: Step 5: 指标上报
        Ext->>Metrics: report("l1_extraction", {<br/>  inputMessageCount,<br/>  memoriesExtracted,<br/>  memoriesStored,<br/>  memoriesByType,<br/>  totalDurationMs<br/>})
    end

    Ext-->>Sched: L1ExtractionResult
```

### 质量门控规则

L0 阶段捕获一切原始对话，L1 阶段通过 `shouldExtractL1()` 进行严格过滤：

- 消息长度不足（过短）
- 纯符号/表情
- 疑似提示注入
- 其他低质量信号

### 消息切分策略

```
qualifiedMessages: [bg...bg | new...new]
                   ← maxBgMessages → ← maxNewMessages →
```

- **newMessages**：最近 N 条消息，LLM 从中提取记忆
- **backgroundMessages**：更早的 M 条消息，仅供 LLM 理解上下文，**严禁从中提取记忆**

### 类型归一化映射

| LLM 输出类型 | 归一化后 | 说明 |
|-------------|---------|------|
| `persona` | `persona` | 用户稳定属性/偏好 |
| `episodic` | `episodic` | 客观事件记忆 |
| `instruction` | `instruction` | 全局行为指令 |
| `episode` | `episodic` | 旧版兼容 |
| `instruct` | `instruction` | 旧版兼容 |
| `preference` | `persona` | 偏好折叠为人设 |

---

## 5. L1 去重算法设计

L1 去重采用**两阶段策略**：先快速召回候选，再批量 LLM 精确判决。同时设计了三级降级保障可用性。

```mermaid
flowchart TD
    Start(["batchDedup(memories)"]) --> Check{"检查召回能力"}

    Check -->|"hasVectorData && embeddingService"| T1["Tier 1: 向量候选召回"]
    Check -->|"!hasVectorData && hasFts"| T2["Tier 2: FTS5 BM25 召回"]
    Check -->|"!hasVectorData && !hasFts"| T3["Tier 3: 跳过去重"]

    subgraph Tier1["Tier 1: 向量候选召回"]
        T1 --> T1_Embed["embedBatch(新记忆内容)"]
        T1_Embed --> T1_Search["searchL1Vector(vec, topK+N)"]
        T1_Search --> T1_Filter["排除当前批次记录<br/>(newRecordIds 过滤)"]
        T1_Filter --> T1_OK{"召回成功?"}
        T1_OK -->|"是"| Phase2
        T1_OK -->|"否: 异常"| T1_Fallback["降级到 Tier 2"]
        T1_Fallback --> T2
    end

    subgraph Tier2["Tier 2: FTS5 BM25 召回"]
        T2 --> T2_Query["buildFtsQuery(记忆内容)"]
        T2_Query --> T2_Search["searchL1Fts(query, 10)"]
        T2_Search --> T2_Filter["排除当前批次记录"]
        T2_Filter --> Phase2
    end

    subgraph Tier3["Tier 3: 跳过去重"]
        T3 --> StoreAll["所有记忆 action=store"]
        StoreAll --> End(["返回 DedupDecision[]"])
    end

    subgraph Phase2["Phase 2: 批量 LLM 判决"]
        Phase2 --> HasCandidates{"有候选?"}
        HasCandidates -->|"否"| StoreAll2["所有记忆 action=store"]
        HasCandidates -->|"是"| BuildPool["构建统一候选池<br/>(去重合并所有候选)"]
        BuildPool --> FormatPrompt["formatBatchConflictPrompt(matches)"]
        FormatPrompt --> LLMCall["LLM 批量判决"]
        LLMCall --> Parse["解析 JSON 响应"]
        Parse --> Validate["校验 action 合法性<br/>补全缺失决策"]
        Validate --> End2(["返回 DedupDecision[]"])

        LLMCall -->|"异常"| FallbackStore["默认全部 action=store"]
        FallbackStore --> End3(["返回 DedupDecision[]"])
        Parse -->|"解析失败"| FallbackStore
    end

    StoreAll2 --> End

    style Tier1 fill:#e8f5e9
    style Tier2 fill:#fff3e0
    style Tier3 fill:#ffebee
    style Phase2 fill:#e3f2fd
```

### 三级降级策略详解

| 等级 | 召回方式 | 前置条件 | 精度 | 性能 |
|------|---------|---------|------|------|
| **Tier 1** | 向量余弦相似度搜索 | `vectorStore` + `embeddingService` 可用，且库中有数据 | 最高 | 需嵌入计算 |
| **Tier 2** | FTS5 BM25 关键词搜索 | `vectorStore` 可用且 FTS 索引就绪 | 中等 | 快速，无需嵌入 |
| **Tier 3** | 跳过去重 | 以上均不可用 | 最低（允许重复） | 零开销 |

### 去重决策类型

```mermaid
flowchart LR
    New["新记忆"] --> Decision{"LLM 判决"}

    Decision -->|"store"| S["新增记忆<br/>target_ids=[]"]
    Decision -->|"skip"| K["忽略新记忆<br/>已有记忆更优"]
    Decision -->|"update"| U["覆盖旧记忆<br/>删除 target_ids<br/>写入更新内容"]
    Decision -->|"merge"| M["合并多记忆<br/>删除 target_ids<br/>写入合并内容<br/>合并 timestamps"]

    S --> R["MemoryRecord"]
    K --> N["null (不写入)"]
    U --> R
    M --> R
```

### 向量候选召回细节

```
新记忆 batch → embedBatch() → 逐条 searchL1Vector(vec, topK+N)
                                      ↓
                              排除当前批次自身 (newRecordIds 过滤)
                                      ↓
                              截取 topK 条候选
```

请求 `topK + batch.size` 条结果，过滤掉当前批次自身后截取 `topK` 条，避免自我匹配。

---

## 6. L1 记忆召回流程

L1 召回由 `auto-recall` 钩子在 `before_prompt_build` 事件触发，负责搜索相关记忆并注入 Agent 上下文。

```mermaid
sequenceDiagram
    participant Agent as Agent 框架
    participant Hook as auto-recall
    participant Search as 搜索调度器
    participant FTS as FTS5 BM25
    participant Vec as 向量搜索
    participant VDB as VectorStore
    participant EMB as EmbeddingService
    participant Persona as persona.md
    participant Scene as scene-index
    participant Context as 上下文注入

    Agent->>Hook: before_prompt_build(userText, ...)

    rect rgb(255, 240, 240)
        Note over Hook: 超时保护
        Hook->>Hook: Promise.race([recall, timeout])<br/>timeout = cfg.recall.timeoutMs ?? 5000
    end

    rect rgb(240, 248, 255)
        Note over Hook,Search: Step 1: L1 记忆搜索
        Hook->>Hook: sanitizeText(userText)
        Hook->>Search: searchMemories(cleanText, strategy)

        alt strategy = "keyword"
            Search->>FTS: buildFtsQuery(text) → searchL1Fts(query, maxResults*2)
            FTS-->>Search: BM25 排序结果
            Search->>Search: 过滤 score >= threshold

        else strategy = "embedding"
            Search->>EMB: embed(userText)
            EMB-->>Search: queryVector
            Search->>Vec: searchL1Vector(queryVector, maxResults*2)
            Vec-->>Search: 余弦相似度结果
            Search->>Search: 过滤 score >= threshold

        else strategy = "hybrid"
            alt VDB 支持原生混合搜索
                Search->>VDB: searchL1Hybrid(query, topK)
                VDB-->>Search: 服务端 RRF 合并结果
            else 客户端 RRF 合并
                par 并行搜索
                    Search->>FTS: searchL1Fts(query, candidateK)
                    FTS-->>Search: keywordResults[]
                and
                    Search->>EMB: embed(userText)
                    EMB-->>Search: queryVector
                    Search->>Vec: searchL1Vector(queryVector, candidateK)
                    Vec-->>Search: embeddingResults[]
                end
                Search->>Search: RRF 合并 (k=60)<br/>score = Σ 1/(k + rank)
            end
        end

        Search-->>Hook: memoryLines[]
    end

    rect rgb(255, 248, 240)
        Note over Hook: Step 2: 预算截断
        Hook->>Hook: applyRecallBudget(lines, cfg.recall)
        Note right of Hook: maxCharsPerMemory<br/>maxTotalRecallChars
    end

    rect rgb(240, 255, 240)
        Note over Hook,Persona: Step 3: L3 人设加载
        Hook->>Persona: readFile(persona.md)
        Persona-->>Hook: personaContent
        Hook->>Hook: stripSceneNavigation(personaContent)
    end

    rect rgb(248, 240, 255)
        Note over Hook,Scene: Step 4: L2 场景导航
        Hook->>Scene: readSceneIndex(pluginDataDir)
        Scene-->>Hook: sceneIndex[]
        Hook->>Hook: generateSceneNavigation(sceneIndex)
    end

    rect rgb(255, 255, 240)
        Note over Hook,Context: Step 5: 上下文分割注入
        Hook->>Hook: 分割为 prependContext + appendSystemContext

        Note right of Hook: prependContext (动态，每轮变化)<br/>→ 注入用户提示词前缀<br/><relevant-memories>...</relevant-memories>

        Note right of Hook: appendSystemContext (稳定，可缓存)<br/>→ 注入系统提示词末尾<br/><user-persona>...</user-persona><br/><scene-navigation>...</scene-navigation><br/><memory-tools-guide>...</memory-tools-guide>
    end

    Hook-->>Agent: RecallResult { prependContext, appendSystemContext }
```

### 三种搜索策略对比

| 策略 | 召回方式 | 合并算法 | 依赖 | 适用场景 |
|------|---------|---------|------|---------|
| **keyword** | FTS5 BM25 | 无需合并 | VectorStore + FTS 索引 | 精确关键词匹配 |
| **embedding** | 向量余弦相似度 | 无需合并 | VectorStore + EmbeddingService | 语义相似匹配 |
| **hybrid** | FTS5 + 向量并行 | RRF (k=60) 或原生混合 | 全部 | 综合最优，默认策略 |

### RRF (Reciprocal Rank Fusion) 算法

```
RRF_score(record) = Σ 1 / (k + rank_i)    其中 k = 60

对每条记录：
  - 在关键词结果中的排名 rank_kw → 贡献 1/(60 + rank_kw)
  - 在向量结果中的排名 rank_vec → 贡献 1/(60 + rank_vec)
  - 若同时出现在两个列表，分数相加
```

### 上下文分割设计

```mermaid
flowchart TB
    subgraph UserPrompt["用户提示词 (动态，每轮变化)"]
        PC["prependContext<br/><relevant-memories><br/>  - [persona|场景] 记忆内容<br/>  - [episodic] 事件内容<br/></relevant-memories>"]
        UT["用户原始输入"]
    end

    subgraph SystemPrompt["系统提示词 (稳定，可缓存)"]
        SP["原始系统提示词"]
        ASC["appendSystemContext<br/><user-persona><br/>  人设内容<br/></user-persona><br/><br/><scene-navigation><br/>  场景导航内容<br/></scene-navigation><br/><br/><memory-tools-guide><br/>  工具调用指南<br/></memory-tools-guide>"]
    end

    style PC fill:#e3f2fd
    style ASC fill:#e8f5e9
    style UT fill:#fff3e0
    style SP fill:#f5f5f5
```

**设计意图**：将频繁变化的 L1 记忆放在用户提示词中，稳定的人设/场景/工具指南放在系统提示词中，最大化利用 LLM 提供商的 Prompt Caching 机制。

---

## 7. 数据模型设计

```mermaid
classDiagram
    class ConversationMessage {
        +string id
        +"user"|"assistant" role
        +string content
        +number timestamp
    }

    class L0MessageRecord {
        +string sessionKey
        +string sessionId
        +string recordedAt
        +string id
        +"user"|"assistant" role
        +string content
        +number timestamp
    }

    class L0ConversationRecord {
        +string sessionKey
        +string sessionId
        +string recordedAt
        +number messageCount
        +ConversationMessage[] messages
    }

    class MemoryType {
        <<enumeration>>
        persona
        episodic
        instruction
    }

    class EpisodicMetadata {
        +string activity_start_time
        +string activity_end_time
    }

    class ExtractedMemory {
        +string content
        +MemoryType type
        +number priority
        +string[] source_message_ids
        +EpisodicMetadata metadata
        +string scene_name
    }

    class MemoryRecord {
        +string id
        +string content
        +MemoryType type
        +number priority
        +string scene_name
        +string[] source_message_ids
        +EpisodicMetadata metadata
        +string[] timestamps
        +string createdAt
        +string updatedAt
        +string sessionKey
        +string sessionId
    }

    class DedupAction {
        <<enumeration>>
        store
        update
        merge
        skip
    }

    class DedupDecision {
        +string record_id
        +DedupAction action
        +string[] target_ids
        +string merged_content
        +MemoryType merged_type
        +number merged_priority
        +string[] merged_timestamps
    }

    class SceneSegment {
        +string scene_name
        +string[] message_ids
        +ExtractedMemory[] memories
    }

    class L1ExtractionResult {
        +boolean success
        +number extractedCount
        +number storedCount
        +MemoryRecord[] records
        +string[] sceneNames
        +string lastSceneName
    }

    class AutoCaptureResult {
        +boolean schedulerNotified
        +number l0RecordedCount
        +number l0VectorsWritten
        +ConversationMessage[] filteredMessages
    }

    class RecallResult {
        +string prependContext
        +string appendSystemContext
        +RecalledMemory[] recalledL1Memories
        +string recalledL3Persona
        +string recallStrategy
    }

    class RecalledMemory {
        +string content
        +number score
        +string type
    }

    class CandidateMatch {
        +ExtractedMemory newMemory
        +MemoryRecord[] candidates
    }

    ConversationMessage "1" --> L0MessageRecord : 序列化为
    ConversationMessage "*" --> L0ConversationRecord : 分组为
    ExtractedMemory <|-- MemoryRecord : 持久化扩展
    ExtractedMemory --> SceneSegment : 属于
    DedupDecision --> DedupAction : 决策类型
    DedupDecision --> MemoryRecord : 操作目标
    CandidateMatch --> ExtractedMemory : 新记忆
    CandidateMatch --> MemoryRecord : 候选记忆
    L1ExtractionResult --> MemoryRecord : 包含
    AutoCaptureResult --> ConversationMessage : 包含
    RecallResult --> RecalledMemory : 包含
```

### MemoryRecord v3 持久化结构

MemoryRecord 是 L1 记忆的最终持久化形态，v3 版本的关键变更：

| 字段 | v2 | v3 | 说明 |
|------|----|----|------|
| `importance` | `"high"|"medium"|"low"` | 移除 | 改用 `priority` 数值 |
| `priority` | 无 | `number (0-100, -1)` | 更精细的优先级控制 |
| `scene_name` | 无 | `string` | 场景归属 |
| `source_message_ids` | 无 | `string[]` | 溯源到 L0 消息 |
| `metadata` | 无 | `EpisodicMetadata` | 类型特定元数据 |
| `timestamps` | 无 | `string[]` | 合并历史时间线 |
| `keywords` | `string[]` | 移除 | 从 content 重建 |
| MemoryType | 4 种 | 3 种 | `preference` 折叠为 `persona` |

### 优先级评分标准

| 范围 | persona | episodic | instruction |
|------|---------|----------|-------------|
| **-1** | - | - | 极严格全局死命令 |
| **80-100** | 健康/禁忌/核心特质 | 重要事件/计划 | 核心行为规则 |
| **60-79** | 一般喜好/技能 | 一般完整活动 | 重要要求 |
| **<60** | 模糊次要（可丢弃） | 琐碎事项（直接丢弃） | 临时要求（直接丢弃） |

---

## 8. 提示词工程设计

### 8.1 L1 记忆提取提示词 (`prompts/l1-extraction.ts`)

#### System Prompt 结构

```mermaid
flowchart TD
    SP["EXTRACT_MEMORIES_SYSTEM_PROMPT"] --> T1["任务一：情境切分"]
    SP --> T2["任务二：核心记忆提取"]
    SP --> T3["任务三：输出格式规范"]

    T1 --> T1_R1["继承：无明显切换，沿用上一情境"]
    T1 --> T1_R2["切换：明确指令/意图转变/独立新目标"]
    T1 --> T1_R3["命名：我(AI)在和xxx做xxx (30-50字符)"]

    T2 --> T2_P["通用原则"]
    T2_P --> P1["宁缺毋滥：过滤琐碎/临时/一次性"]
    T2_P --> P2["独立完整：跳出对话依然成立"]
    T2_P --> P3["归纳合并：强关联消息合并为一条"]

    T2 --> T2_T["三大类型"]
    T2_T --> TY1["persona: 稳定属性/偏好<br/>priority: 80-100(核心) / 50-70(一般)"]
    T2_T --> TY2["episodic: 客观事件/决定/计划<br/>priority: 80-100(重要) / 60-70(一般)"]
    T2_T --> TY3["instruction: 长期行为规则<br/>priority: -1(死命令) / 90-100(核心)"]

    T3 --> T3_F["JSON 数组输出"]
    T3_F --> F1["scene_name + message_ids + memories[]"]
    T3_F --> F2["每条 memory: content, type, priority,<br/>source_message_ids, metadata"]
    T3_F --> F3["episodic 的 metadata 含<br/>activity_start_time / activity_end_time"]
```

#### User Prompt 模板

```
**{时区描述}**
**输出语言**：根据用户发言的主导语言书写

【上一个情境】：{previousSceneName}

【背景对话】（仅供理解上下文，严禁提取记忆）：
[{id}] [{role}] [{timestamp}]: {content}
...

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

【待提取的新消息】（只从这里提取记忆！）：
[{id}] [{role}] [{timestamp}]: {content}
...
```

### 8.2 L1 冲突检测提示词 (`prompts/l1-dedup.ts`)

#### System Prompt 核心规则

```mermaid
flowchart TD
    CD["CONFLICT_DETECTION_SYSTEM_PROMPT"] --> R1["跨 type 合并"]
    CD --> R2["多对多合并"]
    CD --> R3["判断逻辑"]
    CD --> R4["策略倾向"]

    R1 --> R1_D["不同 type 可合并<br/>(persona + episodic → persona)"]
    R2 --> R2_D["一条新记忆可替换多条旧记忆<br/>(target_ids 数组)"]

    R3 --> R3_Class["分辨记忆性质"]
    R3_Class --> C1["状态类 (persona/instruction)<br/>偏好/特质/规则"]
    R3_Class --> C2["事件类 (episodic)<br/>一次性经历/带时间点"]

    R3 --> R3_Action["选择动作"]
    R3_Action --> A1["store: 新信息"]
    R3_Action --> A2["skip: 无增量"]
    R3_Action --> A3["update: 同一事实，新更优"]
    R3_Action --> A4["merge: 互补不矛盾"]

    R4 --> R4_State["状态类: 多条同偏好 → merge<br/>无增量 → skip<br/>明确更新 → update"]
    R4 --> R4_Event["事件类: 同事件前因后果 → merge<br/>完全相同 → skip"]
```

#### User Prompt 结构（统一候选池模式）

```
## 统一候选记忆池（共 N 条已有记忆）
[
  {record_id, content, type, priority, scene_name, timestamps},
  ...
]

════════════════════════════════════════════════

## 待判断的新记忆（共 M 条）

### 第 1 条新记忆 (record_id: xxx)
{record_id, content, type, priority, scene_name}

【关联候选 ID】["id1", "id2"]

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

### 第 2 条新记忆 ...
```

**设计优势**：统一候选池让 LLM 看到全局视图，支持跨记忆去重判断，一次 LLM 调用处理所有新记忆。

---

## 9. 容错与降级设计

### 9.1 整体容错架构

```mermaid
flowchart TD
    subgraph L0Capture["L0 捕获容错"]
        L0E1["JSONL 写入失败"] --> L0F1["返回 filteredMessages<br/>L1 仍可处理"]
        L0E2["向量索引失败"] --> L0F2["非阻塞，仅 warn<br/>JSONL 已写入"]
        L0E3["嵌入服务超时"] --> L0F3["写入元数据+FTS<br/>嵌入后台补全"]
        L0E4["调度通知失败"] --> L0F4["不影响 L0 写入<br/>下次触发时补提取"]
    end

    subgraph L1Extract["L1 提取容错"]
        L1E1["LLM 提取失败"] --> L1F1["返回 success=false<br/>不写入任何记忆"]
        L1E2["JSON 解析失败"] --> L1F2["返回空场景<br/>安全降级"]
        L1E3["类型归一化失败"] --> L1F3["跳过该条记忆<br/>warn 日志"]
        L1E4["去重引擎异常"] --> L1F4["全部 action=store<br/>直接写入"]
    end

    subgraph L1Dedup["L1 去重容错"]
        DE1["向量搜索异常"] --> DF1["降级到 FTS5 BM25"]
        DE2["FTS5 不可用"] --> DF2["跳过去重，全部 store"]
        DE3["LLM 判决异常"] --> DF3["默认全部 action=store"]
        DE4["JSON 解析失败"] --> DF4["fallbackStoreAll()"]
        DE5["决策缺失 record_id"] --> DF5["跳过该条，warn 日志"]
    end

    subgraph L1Write["L1 写入容错"]
        WE1["向量库删除失败"] --> WF1["warn 日志，继续写入新记录"]
        WE2["嵌入计算失败"] --> WF2["写入元数据+FTS，跳过向量"]
        WE3["向量库 upsert 失败"] --> WF3["JSONL 已写入，warn 日志"]
        WE4["单条写入失败"] --> WF4["跳过，继续处理下一条"]
    end

    subgraph L1Recall["L1 召回容错"]
        RE1["搜索超时"] --> RF1["返回 undefined<br/>不注入任何上下文"]
        RE2["嵌入服务不可用"] --> RF2["降级到 keyword 策略"]
        RE3["FTS5 不可用"] --> RF3["keyword 返回空"]
        RE4["人设文件不存在"] --> RF4["跳过人设注入"]
        RE5["场景索引不存在"] --> RF5["跳过场景导航"]
        RE6["全部为空"] --> RF6["返回 undefined<br/>不注入任何上下文"]
    end
```

### 9.2 降级策略矩阵

| 组件 | 故障场景 | 降级行为 | 影响范围 |
|------|---------|---------|---------|
| **L0 捕获** | JSONL 写入失败 | 返回已过滤消息，L1 仍可处理 | 无数据丢失 |
| **L0 向量索引** | 嵌入超时 | SQLite: 写元数据+FTS，后台补嵌入；VDB: 写元数据 | 召回可能延迟 |
| **L1 提取** | LLM 调用失败 | 返回 success=false，不写入 | 本轮无新记忆 |
| **L1 去重** | 向量搜索失败 | 降级 FTS5 → 跳过去重 | 可能产生重复记忆 |
| **L1 写入** | 向量库 upsert 失败 | JSONL 已写入，向量缺失 | 向量搜索可能遗漏 |
| **L1 召回** | 整体超时 | 返回 undefined，不注入 | 无记忆增强 |
| **L1 召回** | 嵌入不可用 | 降级到 keyword | 语义召回不可用 |
| **L1 读取** | VectorStore 不可用 | 回退到 JSONL 全扫描 | 性能下降 |

### 9.3 数据一致性保障

```mermaid
flowchart LR
    subgraph WritePath["写入路径"]
        W_JSONL["JSONL 追加写入<br/>(Source of Truth)"]
        W_VEC["VectorStore upsert<br/>(检索引擎)"]
    end

    subgraph Consistency["一致性保障"]
        C1["JSONL 优先写入<br/>向量库失败不影响"]
        C2["update/merge 时<br/>实时删除 VectorStore 旧记录"]
        C3["JSONL 仅追加<br/>旧记录由 memory-cleaner<br/>定期清理"]
        C4["VectorStore 为检索真相源<br/>JSONL 为备份/恢复源"]
    end

    subgraph Recovery["恢复路径"]
        R1["JSONL → 重建 VectorStore"]
        R2["VectorStore → 正常检索"]
    end

    W_JSONL --> C1
    W_VEC --> C2
    C3 --> R1
    C4 --> R2
```

**双写策略核心原则**：
1. **JSONL 是持久化真相源**：所有写入先确保 JSONL 成功
2. **VectorStore 是检索引擎**：写入失败仅 warn，不阻塞主流程
3. **update/merge 实时删除**：旧记录从 VectorStore 立即删除，保证检索准确性
4. **JSONL 仅追加**：旧记录留在文件中，由 `memory-cleaner` 定期对账清理
5. **恢复路径**：可从 JSONL 重建整个 VectorStore

### 9.4 并发安全

| 场景 | 保护机制 |
|------|---------|
| 多个 agent_end 并发写入 L0 | `CheckpointManager.captureAtomically()` 文件锁保护读游标→写记录→更新游标的原子性 |
| 同一消息被重复捕获 | 位置切片 + 时间游标双重去重 |
| L1 提取与 L0 写入并发 | L1 从 VectorStore/JSONL 读取，与 L0 写入无锁竞争 |
| 向量索引后台嵌入 | `bgTaskRegistry` 跟踪后台任务，`TdaiCore.destroy()` 等待完成后再关闭连接 |
