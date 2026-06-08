# 核心模块分析：LLM 客户端、Prompt 系统与维护管道

## 1. LLM 客户端模块

### 1.1 类继承体系

```
LLMClient (抽象基类, client.py)
├── BaseOpenAIClient (抽象中间层, openai_base_client.py)
│   ├── OpenAIClient (openai_client.py)
│   └── AzureOpenAILLMClient (azure_openai_client.py)
├── OpenAIGenericClient (openai_generic_client.py) — 兼容任意 OpenAI 兼容端点
├── AnthropicClient (anthropic_client.py)
├── GeminiClient (gemini_client.py)
├── GroqClient (groq_client.py)
└── GLiNER2Client (gliner2_client.py) — 委托模式，内部持有 llm_client
```

### 1.2 LLMClient 抽象基类

核心契约：
- `generate_response()` — 公共入口，统一处理：属性提取前置指令注入、多语言指令追加、输入清洗、缓存查询/写入、Tracing 埋点、重试编排
- `_generate_response()` — 抽象方法，各子类实现具体的 API 调用逻辑
- `_generate_response_with_retry()` — 基于 tenacity 的重试装饰器（最多 4 次，指数退避 5~120s），仅对 `RateLimitError`、5xx 错误、JSON 解码错误重试
- `_apply_attribute_extraction_preamble()` — 幂等地向系统消息追加属性提取防幻觉指令
- `_get_cache_key()` — 基于 `model + messages` 的 MD5 哈希

### 1.3 BaseOpenAIClient

提取了 OpenAI 系列的公共逻辑：
- 消息格式转换（`Message` -> `ChatCompletionMessageParam`）
- 模型尺寸选择（`ModelSize.small` / `ModelSize.medium`）
- 结构化响应解析（`_handle_structured_response` / `_handle_json_response`）
- 应用层重试（MAX_RETRIES=2，失败时将错误上下文作为新 user message 追加）

### 1.4 各实现特点

| 实现 | 结构化输出方式 | 特殊处理 |
|------|--------------|---------|
| OpenAIClient | `responses.parse` API | 支持推理模型（gpt-5/o1/o3）的 `reasoning` 和 `verbosity` 参数 |
| AzureOpenAILLMClient | 区分推理/普通模型 | 推理模型走 `responses.parse`，普通模型走 `beta.chat.completions.parse` |
| OpenAIGenericClient | `json_schema` response_format | 默认 16K max_tokens 以兼容本地模型 |
| AnthropicClient | **工具调用 (Tool Use)** | 将 JSON Schema 包装为 Anthropic tool 定义，模型特定 max_tokens 映射表 |
| GeminiClient | `response_schema` 参数 | 安全过滤检查 + JSON 截断抢救（`salvage_json`） |
| GroqClient | `json_object` 响应格式 | 最简实现，默认 max_tokens 仅 2048 |
| GLiNER2Client | 本地 GLiNER2 模型 | 委托模式：仅实体提取自行处理，其他操作委托内部 llm_client |

### 1.5 配置系统

`LLMConfig` 包含：`api_key`, `model`, `base_url`, `small_model`, `temperature`, `max_tokens`

`ModelSize` 枚举：`small`（轻量任务如去重/分类）和 `medium`（主提取任务）

### 1.6 缓存系统

`LLMCache` 使用 **SQLite + JSON** 替代了不安全的 `diskcache`（pickle 反序列化漏洞）：
- 单表 `cache(key TEXT PRIMARY KEY, value TEXT)`
- 仅存储 JSON 可序列化数据
- 注意：OpenAI 系列客户端显式禁用了缓存

### 1.7 Token 追踪

`TokenUsageTracker` 提供线程安全的按 prompt 类型统计：
- `TokenUsage` — 单次调用的 input/output token
- `PromptTokenUsage` — 按 prompt_name 聚合的累计统计
- `TokenUsageTracker` — 基于 `threading.Lock` 的线程安全追踪器

### 1.8 错误体系

| 异常 | 说明 |
|------|------|
| `RateLimitError` | 触发重试 |
| `RefusalError` | LLM 拒绝生成（不重试） |
| `EmptyResponseError` | 空响应 |

---

## 2. Prompt 模板系统

### 2.1 核心数据模型

```
Message(role: str, content: str)           — 消息对象
PromptVersion = Protocol                   — (context: dict) -> list[Message]
PromptFunction = Callable[[dict], list]    — 具体实现签名
```

### 2.2 版本化 Prompt 库

```
PromptLibraryWrapper
├── extract_nodes: PromptTypeWrapper
│   ├── extract_message: VersionWrapper
│   ├── extract_json: VersionWrapper
│   ├── extract_text: VersionWrapper
│   ├── classify_nodes: VersionWrapper
│   ├── extract_attributes: VersionWrapper
│   ├── extract_summary: VersionWrapper
│   ├── extract_summaries_batch: VersionWrapper
│   └── extract_entity_summaries_from_episodes: VersionWrapper
├── dedupe_nodes: PromptTypeWrapper
│   ├── node / nodes / node_list
├── extract_edges: PromptTypeWrapper
│   ├── edge / extract_attributes / extract_timestamps / extract_timestamps_batch
├── extract_nodes_and_edges: PromptTypeWrapper
│   └── extract_message
├── dedupe_edges: PromptTypeWrapper
│   └── resolve_edge
├── summarize_nodes: PromptTypeWrapper
│   ├── summarize_pair / summarize_context / summary_description
├── summarize_sagas: PromptTypeWrapper
│   └── summarize_saga
└── eval: PromptTypeWrapper
```

**VersionWrapper** 在每次调用时自动向 system 消息追加 `DO_NOT_ESCAPE_UNICODE` 指令。

### 2.3 各 Prompt 模块详解

#### extract_nodes.py — 实体提取

- **8 个版本**：extract_message / extract_json / extract_text / classify_nodes / extract_attributes / extract_summary / extract_summaries_batch / extract_entity_summaries_from_episodes
- **Pydantic 模型**：`ExtractedEntity`、`ExtractedEntities`、`SummarizedEntity`、`SummarizedEntities`
- extract_message 针对话语消息，有详尽的负面提取规则和 6 个正反例
- extract_attributes 有严格的 7 条硬规则防止 LLM 将推理/注释写入字段值

#### extract_edges.py — 关系提取

- **4 个版本**：edge / extract_attributes / extract_timestamps / extract_timestamps_batch
- **Pydantic 模型**：`Edge`、`ExtractedEdges`、`EdgeTimestamps`、`BatchEdgeTimestamps`
- 包含实体名验证、去重规则、细节保留规则、时间解析规则

#### extract_nodes_and_edges.py — 联合提取

- **1 个版本**：extract_message
- **Pydantic 模型**：`CombinedEntity` / `CombinedFact` / `CombinedExtraction`
- 单次 LLM 调用同时提取实体和关系，减少孤立节点

#### dedupe_nodes.py — 节点去重

- **3 个版本**：node / nodes / node_list
- **Pydantic 模型**：`NodeDuplicate`、`NodeResolutions`

#### dedupe_edges.py — 边去重

- **1 个版本**：resolve_edge
- **Pydantic 模型**：`EdgeDuplicate`（duplicate_facts + contradicted_facts）
- 同时检测重复和矛盾

#### summarize_nodes.py — 节点摘要

- **3 个版本**：summarize_pair / summarize_context / summary_description

#### summarize_sagas.py — Saga 摘要

- **1 个版本**：summarize_saga
- 支持增量合并（EXISTING_KNOWLEDGE）

### 2.4 辅助工具

- **prompt_helpers.py**：`to_prompt_json()` — 序列化数据为 JSON，默认 `ensure_ascii=False` 保留 Unicode
- **snippets.py**：`summary_instructions` — 共享的摘要生成指南（10 条规则 + 正反例）

---

## 3. 维护管道模块

### 3.1 节点操作 (node_operations.py)

#### 提取流程

```
extract_nodes()
  ├── _build_entity_types_context() — 构建实体类型上下文
  ├── _extract_nodes_single() → _call_extraction_llm()
  │     └── 根据 EpisodeType 选择 prompt
  ├── _create_entity_nodes() — ExtractedEntity → EntityNode 转换
  └── _collapse_exact_duplicate_extracted_nodes() — 同消息内精确去重
```

#### 去重流程（三阶段混合策略）

1. **语义候选搜索**：对每个提取节点，用其名称做 embedding 相似度搜索（余弦阈值 0.6，最多 15 个候选）
2. **确定性去重**：
   - 精确名称匹配（大小写+空白归一化后完全一致）→ 直接合并
   - 多个精确匹配 → 升级到 LLM
   - 高熵名称（Shannon 熵 >= 1.5，长度 >= 6）→ MinHash/LSH 模糊匹配（Jaccard >= 0.9）
   - 低熵名称 → 升级到 LLM
3. **LLM 去重**：将未解决节点批量发送给 `dedupe_nodes.nodes` prompt

#### 属性与摘要提取

```
extract_attributes_from_nodes()
  ├── 并行提取属性（per-entity LLM calls, ModelSize.small）
  ├── 批量提取摘要（MAX_NODES=30 per flight）
  │     ├── 快速路径：摘要+边事实 <= 2*MAX_SUMMARY_CHARS → 直接拼接
  │     └── LLM 路径：extract_summaries_batch
  └── create_entity_node_embeddings()
```

### 3.2 边操作 (edge_operations.py)

#### 去重流程

```
resolve_extracted_edges()
  ├── 精确去重（source+target+fact 归一化）
  ├── 并行查询已有边
  ├── 混合搜索相关边
  ├── 搜索矛盾候选边
  ├── 并行调用 resolve_extracted_edge()
  │     ├── 快速路径：fact 精确匹配 → 直接复用
  │     ├── LLM 去重：dedupe_edges.resolve_edge
  │     ├── 属性提取 + 时间戳提取
  │     └── 矛盾处理：resolve_edge_contradictions()
  └── 生成 embedding
```

#### 矛盾检测算法

- 如果旧边 invalid_at < 新边 valid_at → 不矛盾（旧边已失效）
- 如果新边 invalid_at < 旧边 valid_at → 不矛盾（新边已失效）
- 如果旧边 valid_at < 新边 valid_at → 新边使旧边失效（设置 invalid_at + expired_at）

### 3.3 去重辅助 (dedup_helpers.py)

确定性去重引擎，核心算法：

**精确匹配**：`_normalize_string_exact()` — 小写 + 空白归一化

**模糊匹配（MinHash + LSH）**：
1. `_normalize_name_for_fuzzy()` — 保留字母数字和撇号
2. `_name_entropy()` — Shannon 熵计算，过滤短/低熵名称（阈值 1.5）
3. `_shingles()` — 3-gram 字符 shingle
4. `_minhash_signature()` — 32 个排列的 MinHash 签名（blake2b 哈希）
5. `_lsh_bands()` — 4 个 band 一组的 LSH 分桶
6. `_jaccard_similarity()` — Jaccard 相似度（阈值 0.9）

**数据结构**：
- `DedupCandidateIndexes` — 预计算的查找索引
- `DedupResolutionState` — 可变的解析状态
- `_promote_resolved_node()` — 当提取节点有更具体的类型标签时升级已有节点

### 3.4 社区检测 (community_operations.py)

#### 标签传播算法

- 初始化：每个节点一个社区
- 迭代：每个节点采用邻居中边数加权的多数社区
- 平局打破：选择最大社区
- 终止：无社区变化时收敛

#### 社区构建

采用**层次化摘要合并**：
1. 将社区内所有实体摘要配对
2. 每对通过 `summarize_pair` LLM 调用合并
3. 奇数个时保留落单项
4. 重复直到只剩一个摘要
5. 用 `generate_summary_description` 生成社区名称

### 3.5 联合提取 (combined_extraction.py)

`extract_nodes_and_edges()` 在**单次 LLM 调用**中同时提取实体和关系：
- 使用 `extract_nodes_and_edges.extract_message` prompt
- 批量时间戳提取
- **孤立节点淘汰**：没有连接边的实体节点被丢弃

### 3.6 属性工具 (attribute_utils.py)

**属性长度限制系统**，防御 LLM 将推理文本写入属性字段：
- 默认上限 250 字符（可通过 `GRAPHITI_ATTRIBUTE_MAX_LENGTH` 环境变量覆盖）
- 列表类型：单项上限 + 聚合上限（单项上限 x 8）
- Pydantic Field 的显式 `max_length` 优先于默认值
- 必填字段豁免：不丢弃（否则 Pydantic 验证失败），仅警告

**两种合并模式**：
- `overlay`（节点属性）：先验属性被 LLM 返回值覆盖，但 LLM 省略的字段保留先验值
- `replace`（边属性）：LLM 返回值完全替换先验值，但被截断丢弃的字段从先验恢复

---

## 4. 端到端数据流总览

```
Episode 输入
    │
    ▼
┌─────────────────────────────────┐
│  提取阶段                        │
│  ┌─────────────┐ ┌────────────┐ │
│  │ 分离提取路径 │ │ 联合提取路径│ │
│  │ extract_nodes│ │extract_    │ │
│  │ extract_edges│ │nodes_and_  │ │
│  └──────┬──────┘ │edges       │ │
│         │        └─────┬──────┘ │
└─────────┼──────────────┼────────┘
          │              │
          ▼              ▼
┌─────────────────────────────────┐
│  去重阶段                        │
│  ┌───────────────────────────┐  │
│  │ 节点去重                    │  │
│  │ 1. 语义搜索候选              │  │
│  │ 2. 精确名匹配               │  │
│  │ 3. MinHash/LSH 模糊匹配     │  │
│  │ 4. LLM 去重（兜底）          │  │
│  └───────────────────────────┘  │
│  ┌───────────────────────────┐  │
│  │ 边去重                      │  │
│  │ 1. 精确 fact 匹配            │  │
│  │ 2. 混合搜索 + LLM 去重       │  │
│  │ 3. 矛盾检测 + 时序失效       │  │
│  └───────────────────────────┘  │
└──────────────┬──────────────────┘
               │
               ▼
┌─────────────────────────────────┐
│  属性 & 摘要阶段                  │
│  ┌───────────────────────────┐  │
│  │ 属性提取（并行，per-entity）  │  │
│  │ + 长度限制 + 合并策略        │  │
│  └───────────────────────────┘  │
│  ┌───────────────────────────┐  │
│  │ 摘要提取（批量，30/flight）  │  │
│  │ 快速路径：直接拼接边事实      │  │
│  │ LLM 路径：批量摘要生成       │  │
│  └───────────────────────────┘  │
└──────────────┬──────────────────┘
               │
               ▼
┌─────────────────────────────────┐
│  社区检测阶段                     │
│  标签传播 → 层次化摘要合并        │
│  → CommunityNode + CommunityEdge │
└─────────────────────────────────┘
```

---

## 5. 关键设计特点

1. **混合去重策略**：确定性算法（精确匹配 + MinHash/LSH）处理高置信场景，LLM 仅处理模糊/歧义情况，兼顾效率与准确性
2. **属性安全防护**：多层防御（长度限制 + 必填豁免 + 合并策略）防止 LLM 幻觉污染属性字段
3. **Prompt 版本化管理**：通过三层包装实现灵活的 prompt 版本切换和统一的后处理（Unicode 保留）
4. **LLM 客户端多态**：统一接口 + 提供商特定实现，GLiNER2Client 的委托模式允许轻量本地模型处理实体提取
