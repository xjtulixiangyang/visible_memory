# 核心模块分析：搜索管线与嵌入/重排

## 1. 搜索管线流程

### 1.1 入口函数 search()

搜索管线的核心入口是 `search()` 异步函数，完整流程如下：

**第一步: 前置校验与客户端获取**
- 校验 `group_ids` 合法性
- 从 `GraphitiClients` 中获取 `driver`、`embedder`、`cross_encoder` 和 `tracer`
- 若查询为空字符串，直接返回空的 `SearchResults`

**第二步: 查询向量化**
- 仅当配置中使用了 `cosine_similarity` 搜索方法或 `mmr` 重排器时，才会调用 `embedder.create()` 生成查询向量
- 若调用方已提供 `query_vector`，则直接复用
- 否则使用零向量 `[0.0] * EMBEDDING_DIM` 占位

**第三步: 四路并行搜索**
- 使用 `semaphore_gather` 并发执行四个子搜索：
  - `edge_search()` — 边/事实搜索
  - `node_search()` — 实体节点搜索
  - `episode_search()` — 剧集/片段搜索
  - `community_search()` — 社区搜索
- 每个子搜索返回 `(结果列表, 重排分数列表)` 元组

**第四步: 组装结果**
- 将四路结果打包为 `SearchResults` Pydantic 模型返回

### 1.2 子搜索的统一模式

每个子搜索都遵循相同的三阶段模式：

**阶段一: 多方法候选召回**
- 根据配置的 `search_methods` 列表，并行执行多种搜索方法
- 候选数量上限为 `2 * limit`（为重排留出余量）

**阶段二: BFS 扩展（条件触发）**
- 当配置了 BFS 但未提供 `bfs_origin_node_uuids` 时，系统会用阶段一的搜索结果作为 BFS 起始节点
- 自适应 BFS 机制：先通过 BM25/向量搜索定位锚点，再从锚点扩展图邻域

**阶段三: 重排 (Reranking)**
- 对去重后的候选集执行重排，根据 `reranker` 配置选择策略
- 返回 `limit` 个结果及其分数

### 1.3 各子搜索差异

| 维度 | Edge Search | Node Search | Episode Search | Community Search |
|------|-------------|-------------|----------------|------------------|
| 搜索方法 | BM25 + CosSim + BFS | BM25 + CosSim + BFS | 仅 BM25 | BM25 + CosSim |
| 重排器 | RRF/MMR/CrossEncoder/NodeDistance/EpisodeMentions | RRF/MMR/CrossEncoder/EpisodeMentions/NodeDistance | RRF/CrossEncoder | RRF/MMR/CrossEncoder |
| CrossEncoder 排序对象 | edge.fact | node.name | episode.content | community.name |

---

## 2. 配置与配方系统

### 2.1 配置模型层级

```
SearchConfig (顶层)
  ├── edge_config: EdgeSearchConfig | None
  ├── node_config: NodeSearchConfig | None
  ├── episode_config: EpisodeSearchConfig | None
  ├── community_config: CommunitySearchConfig | None
  ├── limit: int = 10
  └── reranker_min_score: float = 0
```

每个子配置包含：
- `search_methods: list[Enum]` — 搜索方法列表
- `reranker: Enum` — 重排策略
- `sim_min_score: float = 0.6` — 向量相似度最低阈值
- `mmr_lambda: float = 0.5` — MMR 多样性权重
- `bfs_max_depth: int = 3` — BFS 最大深度

### 2.2 搜索方法枚举

| 枚举类 | 可选值 |
|--------|--------|
| `EdgeSearchMethod` | `cosine_similarity`, `bm25`, `breadth_first_search` |
| `NodeSearchMethod` | `cosine_similarity`, `bm25`, `breadth_first_search` |
| `EpisodeSearchMethod` | 仅 `bm25` |
| `CommunitySearchMethod` | `cosine_similarity`, `bm25` |

### 2.3 重排器枚举

| 枚举类 | 可选值 |
|--------|--------|
| `EdgeReranker` | `rrf`, `node_distance`, `episode_mentions`, `mmr`, `cross_encoder` |
| `NodeReranker` | `rrf`, `node_distance`, `episode_mentions`, `mmr`, `cross_encoder` |
| `EpisodeReranker` | `rrf`, `cross_encoder` |
| `CommunityReranker` | `rrf`, `mmr`, `cross_encoder` |

### 2.4 预定义配方 (15 个)

**综合搜索**：COMBINED_HYBRID_SEARCH_RRF / MMR / CROSS_ENCODER

**边搜索**：EDGE_HYBRID_SEARCH_RRF / MMR / NODE_DISTANCE / EPISODE_MENTIONS / CROSS_ENCODER

**节点搜索**：NODE_HYBRID_SEARCH_RRF / MMR / NODE_DISTANCE / EPISODE_MENTIONS / CROSS_ENCODER

**社区搜索**：COMMUNITY_HYBRID_SEARCH_RRF / MMR / CROSS_ENCODER

---

## 3. 搜索过滤器

### SearchFilters 模型

```
SearchFilters
  ├── node_labels: list[str] | None       -- 节点标签过滤
  ├── edge_types: list[str] | None        -- 边类型过滤
  ├── valid_at: list[list[DateFilter]]     -- OR-of-AND 语义
  ├── invalid_at: list[list[DateFilter]]
  ├── created_at: list[list[DateFilter]]
  ├── expired_at: list[list[DateFilter]]
  ├── edge_uuids: list[str] | None
  └── property_filters: list[PropertyFilter] | None
```

日期过滤采用 **OR-of-AND** 语义：外层列表是 OR 关系，内层列表是 AND 关系。

比较运算符支持：`=`, `<>`, `>`, `<`, `>=`, `<=`, `IS NULL`, `IS NOT NULL`

---

## 4. 搜索工具函数 (search_utils.py)

### 搜索函数

每种图元素各有 BM25 / CosSim / BFS 三种搜索函数：
- 边: `edge_fulltext_search`, `edge_similarity_search`, `edge_bfs_search`
- 节点: `node_fulltext_search`, `node_similarity_search`, `node_bfs_search`
- 剧集: `episode_fulltext_search`（仅 BM25）
- 社区: `community_fulltext_search`, `community_similarity_search`

**共同特征**：
- 优先委托给 `driver.search_interface`（若存在）
- 对不同图数据库使用不同的 Cypher 方言
- Neptune 后端使用 OpenSearch 进行全文检索

### 重排算法

| 算法 | 函数 | 说明 |
|------|------|------|
| RRF | `rrf()` | Reciprocal Rank Fusion，对多路搜索结果的排名进行融合 |
| MMR | `maximal_marginal_relevance()` | 平衡相关性与多样性 |
| NodeDistance | `node_distance_reranker()` | 基于到中心节点的图距离重排 |
| EpisodeMentions | `episode_mentions_reranker()` | 基于被剧集提及次数重排 |

### 其他工具

- `fulltext_query()` — 构造全文检索查询，处理 Lucene 语法和查询长度限制
- `calculate_cosine_similarity()` — NumPy 实现的余弦相似度（Neptune 后端使用）
- `hybrid_node_search()` — 独立的混合节点搜索
- `get_relevant_nodes()` / `get_relevant_edges()` — 基于嵌入相似度查找相关节点/边

---

## 5. Cross-Encoder / 重排器

### 5.1 抽象基类

```python
class CrossEncoderClient(ABC):
    @abstractmethod
    async def rank(self, query: str, passages: list[str]) -> list[tuple[str, float]]: ...
```

### 5.2 三种实现

**BGERerankerClient**
- 模型: `BAAI/bge-reranker-v2-m3`（本地 sentence-transformers）
- 机制: 构建 `[query, passage]` 对，调用 `model.predict()` 获取相关性分数
- 异步: 通过 `loop.run_in_executor()` 将同步预测转为异步

**OpenAIRerankerClient**
- 模型: 默认 `gpt-4.1-nano`
- 机制: **基于 logprob 的布尔分类器** — 对每个 passage 并发调用 Chat Completions API，提示模型回答 "True/False"，通过 logit_bias 限制输出，从 logprobs 提取概率作为相关性分数
- 分数计算: 若 top token 为 "True"，分数 = `exp(logprob)`；若为 "False"，分数 = `1 - exp(logprob)`

**GeminiRerankerClient**
- 模型: 默认 `gemini-2.5-flash-lite`
- 机制: **直接评分** — 提示模型输出 0-100 分数，再归一化到 [0, 1]
- 容错: 使用正则提取数字，解析失败时分数为 0.0

### 5.3 Cross-Encoder 在搜索管线中的调用

- **Edge Search**: 对 `fact` 文本调用
- **Node Search**: 对 `name` 调用
- **Episode Search**: 先用 RRF 筛选到 limit 个候选，再对 `content` 调用（两阶段策略）
- **Community Search**: 对 `name` 调用

---

## 6. Embedder 客户端

### 6.1 抽象基类

```python
class EmbedderClient(ABC):
    embedding_dim: int = EMBEDDING_DIM  # 默认 1024

    @abstractmethod
    async def create(self, input_data) -> list[float]: ...

    async def create_batch(self, input_data_list: list[str]) -> list[list[float]]: ...
```

### 6.2 四种实现

| 实现 | 默认模型 | 特点 |
|------|---------|------|
| OpenAIEmbedder | text-embedding-3-small | 支持维度截断 |
| GeminiEmbedder | text-embedding-001 | 特定模型 batch_size=1 |
| AzureOpenAIEmbedderClient | text-embedding-3-small | 无维度截断 |
| VoyageAIEmbedder | voyage-3 | 支持维度截断 |

### 6.3 Embedder 在搜索中的角色

仅在 `search()` 入口函数中调用一次：当配置需要向量搜索或 MMR 重排时，调用 `embedder.create(input_data=[query])` 生成查询向量。

---

## 7. 架构总结

Graphiti 的搜索系统采用 **"多路召回 + 统一重排"** 的经典检索架构：

1. **多路召回**: BM25 全文检索 + 余弦相似度向量检索 + BFS 图遍历，三种正交检索信号互补
2. **自适应 BFS**: 在无明确起点时，先用 BM25/CosSim 定位锚点，再从锚点扩展图邻域
3. **灵活重排**: 5 种重排策略覆盖不同场景
4. **配置驱动**: 通过 Pydantic 模型 + 预定义配方，实现搜索行为的声明式配置
5. **多后端适配**: 搜索函数内部针对四种图数据库使用不同的 Cypher 方言
6. **可观测性**: 全链路 OpenTelemetry 追踪
