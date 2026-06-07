# Mem0 项目规格说明书 (Specification)

## 1. 项目概述

**Mem0**（"mem-zero"）是一个面向 AI Agent 和助手的智能记忆层，提供持久化、个性化的记忆能力。项目以双模式运行：自托管开源 SDK 和托管平台 API。

- **仓库**: https://github.com/mem0ai/mem0
- **许可证**: Apache-2.0
- **语言**: Python (核心 SDK) + TypeScript (TS SDK)
- **Python 版本**: 3.9+ (CLI: 3.10+)
- **Node.js 版本**: v18+ (推荐 v20/v22)

---

## 2. 系统边界与范围

### 2.1 核心范围

| 功能域 | 描述 |
|--------|------|
| 记忆存储与检索 | 添加、搜索、获取、更新、删除记忆 |
| 智能提取 | 从对话消息中自动提取结构化记忆事实 |
| 混合检索 | 语义搜索 + BM25 关键词搜索 + 实体增强 |
| 实体管理 | 实体提取、去重、关联记忆 |
| 记忆重排序 | 可选的 Reranker 精排 |
| 托管平台 | 通过 API Key 访问云端 Mem0 服务 |
| 历史追踪 | 记忆变更历史记录 |

### 2.2 不在核心范围内

| 功能 | 说明 |
|------|------|
| 独立图数据库 | 当前版本使用 Entity Store（基于 Vector Store）替代独立 Graph Store |
| 实时流处理 | 当前为请求-响应模式 |
| 多租户隔离 | 自托管模式依赖 Vector Store 的过滤能力 |

---

## 3. 核心模块规格

### 3.1 Memory 模块

#### 3.1.1 类层次

| 类 | 位置 | 说明 |
|----|------|------|
| `MemoryBase` | `mem0/memory/base.py` | 抽象基类，定义 `get`/`get_all`/`update`/`delete`/`history` |
| `Memory` | `mem0/memory/main.py` | 同步自托管记忆实现 |
| `AsyncMemory` | `mem0/memory/main.py` | 异步自托管记忆实现 |
| `MemoryClient` | `mem0/client/main.py` | 同步托管平台客户端 |
| `AsyncMemoryClient` | `mem0/client/async_main.py` | 异步托管平台客户端 |

#### 3.1.2 公开 API 规格

**add() - 添加记忆**

| 参数 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| `messages` | `str/dict/list` | 是 | - | 输入消息，支持多种格式 |
| `user_id` | `str` | 否* | None | 用户标识 |
| `agent_id` | `str` | 否* | None | Agent 标识 |
| `run_id` | `str` | 否* | None | 会话标识 |
| `metadata` | `dict` | 否 | None | 附加元数据 |
| `infer` | `bool` | 否 | True | 是否启用智能提取 |
| `memory_type` | `str` | 否 | None | 仅支持 `"procedural_memory"` |
| `prompt` | `str` | 否 | None | 自定义提取提示 |

> *`user_id`/`agent_id`/`run_id` 至少提供一个

**返回值**: `{"results": [{"id": str, "memory": str, "event": "ADD"}]}`

**search() - 搜索记忆**

| 参数 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| `query` | `str` | 是 | - | 查询文本 |
| `user_id` | `str` | 否* | None | 用户标识 |
| `agent_id` | `str` | 否* | None | Agent 标识 |
| `run_id` | `str` | 否* | None | 会话标识 |
| `limit` | `int` | 否 | 100 | 最大返回数 |
| `filters` | `dict` | 否 | None | 高级过滤条件 |
| `threshold` | `float` | 否 | 0.0 | 相似度阈值 [0, 1] |
| `rerank` | `bool` | 否 | False | 是否启用重排序 |
| `explain` | `bool` | 否 | False | 是否返回评分解释 |

**返回值**: `{"results": [{"id": str, "memory": str, "score": float, "metadata": dict}]}`

**get() - 获取单条记忆**

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `memory_id` | `str` | 是 | 记忆 ID |

**返回值**: `{"id": str, "memory": str, "hash": str, "metadata": dict, "created_at": str, "updated_at": str}` 或 `None`

**get_all() - 获取所有记忆**

| 参数 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| `user_id` | `str` | 否* | None | 用户标识 |
| `agent_id` | `str` | 否* | None | Agent 标识 |
| `run_id` | `str` | 否* | None | 会话标识 |
| `limit` | `int` | 否 | 100 | 最大返回数 |

**update() - 更新记忆**

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `memory_id` | `str` | 是 | 记忆 ID |
| `data` | `str` | 是 | 新的记忆内容 |
| `metadata` | `dict` | 否 | 更新的元数据 |

**delete() - 删除记忆**

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `memory_id` | `str` | 是 | 记忆 ID |

**delete_all() - 删除所有记忆**

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `user_id` | `str` | 否* | 用户标识 |
| `agent_id` | `str` | 否* | Agent 标识 |
| `run_id` | `str` | 否* | 会话标识 |

**history() - 获取变更历史**

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `memory_id` | `str` | 是 | 记忆 ID |

### 3.2 Provider 模块规格

#### 3.2.1 LLM Provider

| Provider | 标识 | 默认模型 | 配置类 |
|----------|------|----------|--------|
| OpenAI | `openai` | `gpt-5-mini` | `OpenAIConfig` |
| Anthropic | `anthropic` | `claude-sonnet-4-20250514` | `AnthropicConfig` |
| Ollama | `ollama` | `llama3.1:70b` | `OllamaConfig` |
| Groq | `groq` | - | `BaseLlmConfig` |
| Together | `together` | - | `BaseLlmConfig` |
| AWS Bedrock | `aws_bedrock` | - | `AWSBedrockConfig` |
| LiteLLM | `litellm` | - | `BaseLlmConfig` |
| Azure OpenAI | `azure_openai` | - | `AzureOpenAIConfig` |
| OpenAI Structured | `openai_structured` | - | `OpenAIConfig` |
| Azure Structured | `azure_openai_structured` | - | `AzureOpenAIConfig` |
| Gemini | `gemini` | - | `BaseLlmConfig` |
| DeepSeek | `deepseek` | - | `DeepSeekConfig` |
| MiniMax | `minimax` | - | `MinimaxConfig` |
| xAI | `xai` | - | `BaseLlmConfig` |
| Sarvam | `sarvam` | - | `BaseLlmConfig` |
| LM Studio | `lmstudio` | - | `LMStudioConfig` |
| vLLM | `vllm` | - | `VllmConfig` |
| LangChain | `langchain` | - | `BaseLlmConfig` |

**接口契约**: `generate_response(messages, tools, tool_choice, **kwargs) -> str | dict`

#### 3.2.2 Embedding Provider

| Provider | 标识 | 默认模型 | 默认维度 |
|----------|------|----------|----------|
| OpenAI | `openai` | `text-embedding-3-small` | 1536 |
| Ollama | `ollama` | `nomic-embed-text` | 512 |
| Azure OpenAI | `azure_openai` | - | - |
| HuggingFace | `huggingface` | - | - |
| Gemini | `gemini` | - | - |
| Vertex AI | `vertex_ai` | - | - |
| Together | `together` | - | - |
| LM Studio | `lmstudio` | - | - |
| LangChain | `langchain` | - | - |
| AWS Bedrock | `aws_bedrock` | - | - |
| FastEmbed | `fastembed` | - | - |

**接口契约**: `embed(text, memory_action) -> list[float]`，`embed_batch(texts, memory_action) -> list[list[float]]`

#### 3.2.3 Vector Store Provider

| Provider | 标识 | BM25 支持 | 批量搜索 |
|----------|------|-----------|----------|
| Qdrant | `qdrant` | 是 | 是 |
| ChromaDB | `chroma` | 否 | 否 |
| PGVector | `pgvector` | 否 | 否 |
| Pinecone | `pinecone` | 否 | 否 |
| Milvus | `milvus` | 否 | 否 |
| MongoDB | `mongodb` | 否 | 否 |
| Redis | `redis` | 否 | 否 |
| Elasticsearch | `elasticsearch` | 否 | 否 |
| Weaviate | `weaviate` | 否 | 否 |
| FAISS | `faiss` | 否 | 否 |
| Supabase | `supabase` | 否 | 否 |
| Upstash Vector | `upstash_vector` | 否 | 否 |
| Azure AI Search | `azure_ai_search` | 否 | 否 |
| Azure MySQL | `azure_mysql` | 否 | 否 |
| OpenSearch | `opensearch` | 否 | 否 |
| Valkey | `valkey` | 否 | 否 |
| Databricks | `databricks` | 否 | 否 |
| Google Matching Engine | `google_matching_engine` | 否 | 否 |
| LangChain | `langchain` | 否 | 否 |
| S3 Vectors | `s3_vectors` | 否 | 否 |
| Baidu | `baidu` | 否 | 否 |
| Cassandra | `cassandra` | 否 | 否 |
| Neptune Analytics | `neptune_analytics` | 否 | 否 |
| Turbopuffer | `turbopuffer` | 否 | 否 |

**接口契约**: 10 个抽象方法 + 2 个可选覆写方法

#### 3.2.4 Reranker Provider

| Provider | 标识 | 默认模型 | 类型 |
|----------|------|----------|------|
| Cohere | `cohere` | `rerank-english-v3.0` | API |
| Sentence Transformer | `sentence_transformer` | `cross-encoder/ms-marco-MiniLM-L-6-v2` | 本地 |
| HuggingFace | `huggingface` | `BAAI/bge-reranker-base` | 本地 |
| LLM | `llm` | `gpt-4o-mini` | API |
| Zero Entropy | `zero_entropy` | `zerank-1` | API |

**接口契约**: `rerank(query, documents, top_k) -> list[dict]`

### 3.3 配置系统规格

#### 3.3.1 MemoryConfig 结构

```python
class MemoryConfig(BaseModel):
    vector_store: VectorStoreConfig    # provider + config
    llm: LlmConfig                     # provider + config
    embedder: EmbedderConfig           # model, api_key, dims, ...
    history_db_path: str = "~/.mem0/history.db"
    reranker: Optional[RerankerConfig] = None
    version: str = "v1.1"
    custom_instructions: Optional[str] = None
```

#### 3.3.2 验证规则

| 规则 | 适用场景 |
|------|----------|
| 至少提供一个实体 ID (`user_id`/`agent_id`/`run_id`) | `add`/`search`/`get_all`/`delete_all` |
| 实体 ID 不允许包含内部空格 | 所有接受实体 ID 的方法 |
| `threshold` 必须在 [0, 1] 范围内 | `search` |
| `top_k`/`limit` 必须为非负整数 | `search`/`get_all` |
| `memory_type` 仅支持 `"procedural_memory"` | `add` |
| `provider` 必须在支持列表中 | 所有 Provider 配置 |

---

## 4. 数据模型规格

### 4.1 MemoryItem

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | `str` | 记忆唯一标识 (UUID) |
| `memory` | `str` | 记忆文本内容 |
| `hash` | `str` | MD5 哈希（去重用） |
| `metadata` | `dict` | 元数据（含 `user_id`/`agent_id`/`run_id`/`actor_id`/`role`） |
| `score` | `float` | 搜索相似度分数 |
| `created_at` | `str` | 创建时间 (ISO 8601) |
| `updated_at` | `str` | 更新时间 (ISO 8601) |

### 4.2 Entity Store 数据模型

| 字段 | 类型 | 说明 |
|------|------|------|
| `data` | `str` | 实体文本 |
| `entity_type` | `str` | 实体类型 (PERSON/ORG/GPE 等) |
| `linked_memory_ids` | `list[str]` | 关联的记忆 ID 列表 |
| `hash` | `str` | MD5 哈希 |

### 4.3 历史记录数据模型

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | `str` | 历史记录 ID |
| `memory_id` | `str` | 关联的记忆 ID |
| `old_memory` | `str` | 变更前的记忆内容 |
| `new_memory` | `str` | 变更后的记忆内容 |
| `event` | `str` | 事件类型 (ADD/UPDATE/DELETE) |
| `is_deleted` | `int` | 是否已删除 (0/1) |
| `created_at` | `str` | 事件时间 |

---

## 5. 搜索算法规格

### 5.1 混合检索策略

搜索采用三路信号融合：

| 信号 | 权重机制 | 说明 |
|------|----------|------|
| 语义搜索 | 主信号 | 向量余弦相似度 |
| BM25 关键词 | 补充信号 | Sigmoid 归一化后的 BM25 分数 |
| 实体增强 | Boost 信号 | `similarity * ENTITY_BOOST_WEIGHT * memory_count_weight` |

### 5.2 过获取策略

- 语义搜索获取 `max(limit * 4, 60)` 条候选
- 经评分排序后截断到 `limit` 条

### 5.3 实体增强公式

```
entity_boost = similarity * ENTITY_BOOST_WEIGHT * memory_count_weight
memory_count_weight = 1 / (1 + 0.001 * (num_linked - 1)^2)
```

### 5.4 高级过滤器规格

支持的操作符：
- 比较：`eq`, `ne`, `gt`, `gte`, `lt`, `lte`, `in`, `nin`, `contains`, `icontains`
- 逻辑：`AND`, `OR`, `NOT`
- 通配符：`*`（匹配任意值）

---

## 6. MemoryClient API 规格

### 6.1 认证

- Header: `Authorization: Token {api_key}`
- Header: `Mem0-User-ID: {md5(api_key)}`
- 初始化验证: `GET /v1/ping/`

### 6.2 API 端点

| 操作 | 方法 | 端点 | 版本 |
|------|------|------|------|
| 添加记忆 | POST | `/v3/memories/add/` | v3 |
| 搜索记忆 | POST | `/v3/memories/search/` | v3 |
| 获取所有记忆 | POST | `/v3/memories/` | v3 |
| 获取单条记忆 | GET | `/v1/memories/{id}/` | v1 |
| 更新记忆 | PUT | `/v1/memories/{id}/` | v1 |
| 删除记忆 | DELETE | `/v1/memories/{id}/` | v1 |
| 删除所有记忆 | DELETE | `/v1/memories/` | v1 |
| 历史记录 | GET | `/v1/memories/{id}/history/` | v1 |

### 6.3 错误类型

| 异常 | HTTP 状态码 | 说明 |
|------|-------------|------|
| `AuthenticationError` | 401 | API Key 无效 |
| `RateLimitError` | 429 | 请求频率超限 |
| `MemoryNotFoundError` | 404 | 记忆不存在 |

---

## 7. 非功能性规格

### 7.1 性能

- 批量操作优先：`embed_batch`、`insert` 批量、`search_batch`
- 失败回退：批量操作失败自动降级为逐条操作
- 并发控制：实体增强计算限制 4 个并发

### 7.2 可靠性

- Hash 去重：MD5 哈希防止重复记忆
- 反幻觉：UUID 映射为整数传给 LLM
- 容错：Reranker 失败回退原始排序，实体 upsert 失败仅 warning

### 7.3 可扩展性

- 运行时注册：`LlmFactory.register_provider()` 支持动态添加 Provider
- 延迟导入：`importlib` 按需加载，未使用的 Provider 不触发导入
- 插件化：新增 Provider 只需添加一个文件 + 注册一行映射

### 7.4 兼容性

- Python 3.9 - 3.12
- Node.js 18+ (推荐 20/22)
- 向后兼容：配置支持 None/dict/BaseConfig 三种输入形式
