# Graphiti 项目规格说明书

## 1. 项目概述

Graphiti 是一个基于知识图谱的时序记忆系统，用于从非结构化文本中提取实体与关系，构建可查询的知识图谱。系统支持多种图数据库后端（Neo4j、FalkorDB、Kuzu、Neptune），并提供 MCP 和 HTTP 两种服务接口，面向 AI 代理和通用客户端。

### 1.1 核心定位

- **时序知识图谱**：支持事实的时间有效性（valid_at / invalid_at），自动处理矛盾事实的失效
- **增量式知识摄入**：通过 LLM 提取 + 去重 + 矛盾检测，实现知识的增量更新
- **多路混合搜索**：BM25 全文检索 + 向量余弦相似度 + BFS 图遍历，支持 5 种重排策略
- **多数据库适配**：统一接口支持 Neo4j / FalkorDB / Kuzu / Neptune 四种图数据库

### 1.2 技术栈

| 层次 | 技术 |
|------|------|
| 语言 | Python 3.10+ |
| 数据模型 | Pydantic v2 |
| 异步框架 | asyncio |
| 图数据库 | Neo4j / FalkorDB / Kuzu / Neptune |
| LLM | OpenAI / Anthropic / Gemini / Groq / GLiNER2 |
| 嵌入模型 | OpenAI / Gemini / Azure OpenAI / Voyage |
| 重排模型 | BGE / OpenAI / Gemini |
| 服务框架 | FastMCP (MCP) / FastAPI (HTTP) |
| 可观测性 | OpenTelemetry |
| 包管理 | uv |

---

## 2. 功能规格

### 2.1 知识摄入 (Ingestion)

#### 2.1.1 单条添加 (`add_episode`)

| 参数 | 类型 | 说明 |
|------|------|------|
| name | str | Episode 名称 |
| episode_body | str | 原始内容 |
| source | EpisodeType | 来源类型 (message/json/text/fact_triple) |
| reference_time | datetime | 参考时间 |
| group_id | str | 图分区标识 |
| entity_types | list[type] | 自定义实体类型 |
| excluded_entity_types | list[str] | 排除的实体类型 |
| update_communities | bool | 是否更新社区 |
| saga | str | 关联的 Saga 名称 |

**处理流水线**：

1. 验证实体类型与排除类型
2. 处理 group_id（若提供则克隆驱动切换数据库）
3. 检索历史片段作为上下文
4. 创建 EpisodicNode
5. 提取节点（LLM）→ 去重/合并
6. 提取边（LLM）→ 去重/失效处理
7. 属性提取（LLM）
8. 持久化节点、边、片段边
9. Saga 关联（可选）
10. 社区更新（可选）

#### 2.1.2 批量添加 (`add_episode_bulk`)

- 先保存所有片段，再批量检索上下文
- 内存去重 + 批量解析
- Saga 关联按 valid_at 排序构建 NEXT_EPISODE 链

#### 2.1.3 三元组添加 (`add_triplet`)

手动添加 (source, relation, target) 三元组，自动处理节点去重和边冲突。

### 2.2 知识搜索 (Search)

#### 2.2.1 基础搜索 (`search`)

混合搜索，返回 `list[EntityEdge]`。

#### 2.2.2 高级搜索 (`search_`)

支持自定义 SearchConfig，返回 `SearchResults`（包含 edges、nodes、episodes、communities 四类结果）。

**搜索方法**：

| 方法 | 适用对象 | 说明 |
|------|---------|------|
| BM25 | Edge/Node/Episode/Community | 全文检索 |
| cosine_similarity | Edge/Node/Community | 向量余弦相似度 |
| breadth_first_search | Edge/Node | 广度优先图遍历 |

**重排策略**：

| 策略 | 说明 |
|------|------|
| RRF | Reciprocal Rank Fusion，多路结果排名融合 |
| MMR | Maximal Marginal Relevance，平衡相关性与多样性 |
| CrossEncoder | 语义精确排序（BGE/OpenAI/Gemini） |
| NodeDistance | 基于图距离重排 |
| EpisodeMentions | 基于被提及次数重排 |

**预定义配方**：15 个配方覆盖综合搜索、边搜索、节点搜索、社区搜索场景。

### 2.3 知识维护 (Maintenance)

#### 2.3.1 社区构建 (`build_communities`)

- 标签传播算法聚类
- 层次化摘要合并
- 生成 CommunityNode + CommunityEdge

#### 2.3.2 Saga 摘要 (`summarize_saga`)

增量摘要，维护两个水位标记：
- `last_summarized_at`：墙钟时间
- `last_summarized_episode_valid_at`：事件时间

#### 2.3.3 删除操作

- `remove_episode`：删除片段及其独占的边和节点
- `clear_data`：清空图谱数据（支持按 group_id 过滤）

### 2.4 命名空间 API

通过 `graphiti.nodes.<subtype>` 和 `graphiti.edges.<subtype>` 提供细粒度 CRUD：

| 命名空间 | 操作 |
|----------|------|
| nodes.entity | save/save_bulk/delete/get_by_uuid/get_by_group_ids/load_embeddings |
| nodes.episode | save/save_bulk/delete/get_by_uuid/get_by_entity_node_uuid/retrieve_episodes |
| nodes.community | save/save_bulk/delete/get_by_uuid/load_name_embedding |
| nodes.saga | save/save_bulk/delete/get_by_uuid |
| edges.entity | save/save_bulk/delete/get_by_uuid/get_between_nodes/get_by_node_uuid/load_embeddings |
| edges.episodic | save/save_bulk/delete/get_by_uuid |
| edges.community | save/delete/get_by_uuid |
| edges.has_episode | save/save_bulk/delete/get_by_uuid |
| edges.next_episode | save/save_bulk/delete/get_by_uuid |

---

## 3. 数据模型规格

### 3.1 节点类型

| 节点 | 数据库标签 | 核心字段 | 说明 |
|------|-----------|---------|------|
| EpisodicNode | Episodic | content, source, valid_at, entity_edges | 原始输入片段 |
| EntityNode | Entity + 自定义标签 | name, name_embedding, summary, attributes | 提取的知识实体 |
| CommunityNode | Community | name, name_embedding, summary | 实体聚类社区 |
| SagaNode | Saga | summary, first_episode_uuid, last_episode_uuid | 叙事线索 |

### 3.2 边类型

| 边 | 图关系 | 核心字段 | 说明 |
|----|--------|---------|------|
| EpisodicEdge | (Episodic)-[MENTIONS]->(Entity) | - | 片段提及实体 |
| EntityEdge | (Entity)-[RELATES_TO]->(Entity) | name, fact, fact_embedding, valid_at, invalid_at, expired_at | 实体间关系/事实 |
| CommunityEdge | (Community)-[HAS_MEMBER]->(Entity/Community) | - | 社区成员关系 |
| HasEpisodeEdge | (Saga)-[HAS_EPISODE]->(Episodic) | - | Saga 包含 Episode |
| NextEpisodeEdge | (Episodic)-[NEXT_EPISODE]->(Episodic) | - | Episode 时序链 |

### 3.3 时间语义

| 字段 | 语义 |
|------|------|
| valid_at | 事实在现实世界中成立的时间 |
| invalid_at | 事实在现实世界中失效的时间 |
| expired_at | 系统层面的过期时间（被新信息取代） |
| reference_time | 产生此边的片段参考时间 |
| created_at | 系统创建时间 |

---

## 4. 服务接口规格

### 4.1 MCP 服务器

| 工具 | 功能 |
|------|------|
| add_memory | 添加 Episode（异步排队） |
| search_nodes | 搜索实体节点 |
| search_memory_facts | 搜索事实 |
| delete_entity_edge | 删除实体边 |
| delete_episode | 删除 Episode |
| get_entity_edge | 获取实体边详情 |
| get_episodes | 获取 Episode 列表 |
| clear_graph | 清除图谱数据 |
| get_status | 获取服务状态 |

传输模式：HTTP (streamable) / SSE (deprecated) / stdio

### 4.2 HTTP 服务器

| 方法 | 路径 | 功能 |
|------|------|------|
| POST | /messages | 批量添加消息 |
| POST | /entity-node | 添加实体节点 |
| DELETE | /entity-edge/{uuid} | 删除实体边 |
| DELETE | /group/{group_id} | 删除分组 |
| DELETE | /episode/{uuid} | 删除 Episode |
| POST | /clear | 清空图谱 |
| POST | /search | 搜索事实 |
| GET | /entity-edge/{uuid} | 获取实体边 |
| GET | /episodes/{group_id} | 获取 Episode 列表 |
| POST | /get-memory | 获取记忆 |

---

## 5. 配置规格

### 5.1 环境变量

| 变量 | 默认值 | 说明 |
|------|--------|------|
| EMBEDDING_DIM | 1024 | 嵌入向量维度 |
| GRAPHITI_ATTRIBUTE_MAX_LENGTH | 250 | 属性字段最大长度 |
| SEMAPHORE_LIMIT | 20 | 最大并发协程数 |
| CHUNK_TOKEN_SIZE | 3000 | 内容分块 token 大小 |
| CHUNK_OVERLAP_TOKENS | 200 | 分块重叠 token 数 |
| CHUNK_MIN_TOKENS | 1000 | 触发分块的最小 token 数 |
| CHUNK_DENSITY_THRESHOLD | 0.15 | 实体密度阈值 |

### 5.2 搜索默认值

| 参数 | 默认值 | 说明 |
|------|--------|------|
| DEFAULT_MIN_SCORE | 0.6 | 向量相似度最低阈值 |
| DEFAULT_MMR_LAMBDA | 0.5 | MMR 多样性权重 |
| MAX_SEARCH_DEPTH | 3 | BFS 最大深度 |
| MAX_QUERY_LENGTH | 128 | 全文检索最大查询长度 |

---

## 6. 非功能性规格

### 6.1 并发控制

- 信号量限制（默认 20 协程）
- MCP 服务器按 group_id 隔离队列
- HTTP 服务器使用全局 AsyncWorker

### 6.2 可观测性

- OpenTelemetry 全链路追踪
- Token 使用量追踪（按 prompt 类型统计）
- LLM 响应缓存（SQLite + JSON）

### 6.3 容错

- LLM 调用重试（tenacity，最多 4 次，指数退避 5~120s）
- 仅对 RateLimitError、5xx 错误、JSON 解码错误重试
- Gemini 安全过滤检查 + JSON 截断抢救
- 属性长度限制防御 LLM 幻觉

### 6.4 多数据库适配

| 数据库 | 事务 | 全文索引 | 向量搜索 | 特殊处理 |
|--------|------|---------|---------|---------|
| Neo4j | 支持 | Lucene | db.index.vector | IN TRANSACTIONS 批量 |
| FalkorDB | 不支持 | RedisSearch | vecf32() | datetime 转 ISO 字符串 |
| Kuzu | 不支持 | 不支持 | 不支持 | 边属性通过中间节点模拟 |
| Neptune | 不支持 | AOSS | 逗号分隔字符串 | 双存储架构 |
