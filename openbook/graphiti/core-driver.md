# 核心模块分析：Driver 驱动层

## 1. 整体架构

Driver 模块采用**策略模式 + 抽象工厂模式**的组合设计，实现支持多种图数据库的统一数据访问层。

### 三层操作架构

```
调用层 (graphiti_core/nodes.py, edges.py 等)
  通过 driver.entity_node_ops.save(driver, node)
  或 driver.search_ops.node_fulltext_search(...)
      │
抽象操作层 (driver/operations/)
  EntityNodeOperations(ABC), SearchOperations(ABC)...
  方法签名: save(executor: QueryExecutor, ...)
      │
具体实现层 (driver/neo4j/operations/, 等)
  Neo4jEntityNodeOperations, FalkorEntityNodeOps...
  通过 executor.execute_query() 或 tx.run() 执行
```

---

## 2. 核心抽象类

### 2.1 QueryExecutor

```python
class QueryExecutor(ABC):
    @abstractmethod
    async def execute_query(self, cypher_query_, **kwargs) -> Any: ...
```

最精简的接口，仅包含 `execute_query`。设计目的是**打破循环导入**：操作抽象类只需依赖 `QueryExecutor`，而不必依赖完整的 `GraphDriver`。

### 2.2 Transaction

```python
class Transaction(ABC):
    @abstractmethod
    async def run(self, query, **kwargs) -> Any: ...
```

最小化事务接口。支持事务的驱动（Neo4j）提供真实的 commit/rollback 语义；不支持事务的驱动使用即时执行的薄包装。

### 2.3 GraphDriverSession

```python
class GraphDriverSession(ABC):
    async def __aenter__() / __aexit__()
    async def run(query, **kwargs)
    async def close()
    async def execute_write(func, *args, **kwargs)
```

会话级抽象，模拟数据库会话的生命周期。

### 2.4 GraphDriver

```python
class GraphDriver(ABC, QueryExecutor):
    provider: GraphProvider
    fulltext_syntax: str
    _database: str

    # 抽象方法
    execute_query(), session(), close(), build_indices_and_constraints()

    # 具体方法
    with_database(database), clone(database), transaction(), build_fulltext_query()

    # 11 个操作属性（全部默认返回 None）
    entity_node_ops, episode_node_ops, community_node_ops, saga_node_ops,
    entity_edge_ops, episodic_edge_ops, community_edge_ops,
    has_episode_edge_ops, next_episode_edge_ops,
    search_ops, graph_ops
```

### 2.5 GraphProvider 枚举

```python
class GraphProvider(Enum):
    NEO4J = 'neo4j'
    FALKORDB = 'falkordb'
    KUZU = 'kuzu'
    NEPTUNE = 'neptune'
```

---

## 3. 操作抽象类

所有操作抽象类遵循统一的设计模式：

| 抽象类 | 操作对象 | 核心方法 |
|--------|---------|---------|
| `EntityNodeOperations` | 实体节点 | save, save_bulk, delete, delete_by_group_id, delete_by_uuids, get_by_uuid, get_by_uuids, get_by_group_ids, load_embeddings, load_embeddings_bulk |
| `EpisodeNodeOperations` | 情景节点 | save, save_bulk, delete, delete_by_group_id, delete_by_uuids, get_by_uuid, get_by_uuids, get_by_group_ids, get_by_entity_node_uuid, retrieve_episodes |
| `CommunityNodeOperations` | 社区节点 | save, save_bulk, delete, delete_by_group_id, delete_by_uuids, get_by_uuid, get_by_uuids, get_by_group_ids, load_name_embedding |
| `SagaNodeOperations` | 传奇节点 | save, save_bulk, delete, delete_by_group_id, delete_by_uuids, get_by_uuid, get_by_uuids, get_by_group_ids |
| `EntityEdgeOperations` | 实体边 | save, save_bulk, delete, delete_by_uuids, get_by_uuid, get_by_uuids, get_by_group_ids, get_between_nodes, get_by_node_uuid, load_embeddings, load_embeddings_bulk |
| `EpisodicEdgeOperations` | 情景边 | save, save_bulk, delete, delete_by_uuids, get_by_uuid, get_by_uuids, get_by_group_ids |
| `CommunityEdgeOperations` | 社区边 | save, delete, delete_by_uuids, get_by_uuid, get_by_uuids, get_by_group_ids |
| `HasEpisodeEdgeOperations` | HAS_EPISODE边 | save, save_bulk, delete, delete_by_uuids, get_by_uuid, get_by_uuids, get_by_group_ids |
| `NextEpisodeEdgeOperations` | NEXT_EPISODE边 | save, save_bulk, delete, delete_by_uuids, get_by_uuid, get_by_uuids, get_by_group_ids |
| `SearchOperations` | 搜索 | node_fulltext_search, node_similarity_search, node_bfs_search, edge_fulltext_search, edge_similarity_search, edge_bfs_search, episode_fulltext_search, community_fulltext_search, community_similarity_search, node_distance_reranker, episode_mentions_reranker, build_node_search_filters, build_edge_search_filters, build_fulltext_query |
| `GraphMaintenanceOperations` | 图维护 | clear_data, build_indices_and_constraints, delete_all_indexes, get_community_clusters, remove_communities, determine_entity_community, get_mentioned_nodes, get_communities_by_nodes |

**关键设计特征**：
1. 所有操作方法的第一个参数是 `executor: QueryExecutor`，而非 `driver: GraphDriver`
2. 写操作支持可选的 `tx: Transaction | None = None` 参数
3. 批量操作支持 `batch_size` 参数

---

## 4. 四种数据库驱动对比

### 4.1 Neo4j

| 特性 | 实现 |
|------|------|
| provider | `GraphProvider.NEO4J` |
| 连接方式 | `AsyncGraphDatabase.driver(uri, auth)` |
| fulltext_syntax | `''`（空字符串） |
| default_group_id | `''`（空字符串） |
| 事务支持 | **真实事务**：`session.begin_transaction()`，支持 commit/rollback |
| 索引构建 | 使用 `get_range_indices()` + `get_fulltext_indices()` 并行执行 |
| 全文搜索 | Lucene 语法 |
| 向量搜索 | `db.index.vector.queryNodes` / `db.index.vector.queryRelationships` |
| 批量删除 | `IN TRANSACTIONS OF $batch_size ROWS` |

### 4.2 FalkorDB

| 特性 | 实现 |
|------|------|
| provider | `GraphProvider.FALKORDB` |
| 连接方式 | `FalkorDB(host, port)`，多租户通过 `select_graph()` |
| fulltext_syntax | `'@'`（RedisSearch 风格） |
| default_group_id | `'\\_'` |
| 事务支持 | **无原生事务** |
| 全文搜索 | RedisSearch 语法（`@field:value`） |
| 向量搜索 | `vecf32()` |
| 特殊处理 | datetime 转 ISO 字符串，NUL 字节清除 |

### 4.3 Kuzu

| 特性 | 实现 |
|------|------|
| provider | `GraphProvider.KUZU` |
| 连接方式 | `kuzu.Database(db)` + `kuzu.AsyncConnection(db)` |
| fulltext_syntax | 不适用（不支持全文索引） |
| 事务支持 | **无原生事务** |
| 索引构建 | **空操作**：使用显式 schema |
| 特殊处理 | **边属性通过中间节点模拟**：`(Entity)-[:RELATES_TO]->(RelatesToNode_)-[:RELATES_TO]->(Entity)` |
| 特殊处理 | 不支持 `UNWIND`，save_bulk 退化为逐条 save |
| 特殊处理 | attributes 序列化为 JSON 字符串 |

### 4.4 Neptune

| 特性 | 实现 |
|------|------|
| provider | `GraphProvider.NEPTUNE` |
| 连接方式 | `NeptuneGraph` 或 `NeptuneAnalyticsGraph` |
| fulltext_syntax | 不适用（使用 OpenSearch 替代） |
| 事务支持 | **无原生事务** |
| 索引构建 | **使用 Amazon OpenSearch Service (AOSS)** |
| 特殊处理 | **双存储架构**：图数据存 Neptune，全文索引存 AOSS |
| 特殊处理 | 向量嵌入存储为逗号分隔字符串 |
| 特殊处理 | AOSS 4 个索引：node_name_and_summary、community_name、episode_content、edge_name_and_fact |

---

## 5. 操作方法统一模式

所有具体操作实现遵循相同的方法模式：

```python
async def save(self, executor: QueryExecutor, node: EntityNode, tx: Transaction | None = None):
    # 1. 构建参数字典
    entity_data = {'uuid': node.uuid, 'name': node.name, ...}

    # 2. 获取提供商特定的 Cypher 查询
    query = get_entity_node_save_query(GraphProvider.NEO4J, labels)

    # 3. 根据是否有事务选择执行路径
    if tx is not None:
        await tx.run(query, entity_data=entity_data)
    else:
        await executor.execute_query(query, entity_data=entity_data)
```

---

## 6. 遗留接口

### SearchInterface
基于 Pydantic `BaseModel` 的搜索接口，包含 14 个方法。标记为"遗留"，保留用于向后兼容。

### GraphOperationsInterface
同样基于 Pydantic `BaseModel`，是所有节点/边 CRUD 操作的统一大接口（约 50 个方法）。也是遗留接口，新的操作模式已拆分为独立的操作类。

---

## 7. 关键设计决策

1. **QueryExecutor 解耦**：操作类仅依赖 `QueryExecutor` 接口而非 `GraphDriver`，彻底消除循环导入
2. **操作类组合优于继承**：`GraphDriver` 通过属性组合 11 个操作类实例，而非通过继承大基类
3. **事务可选传递**：所有写操作接受 `tx: Transaction | None`，灵活选择是否在事务中执行
4. **提供商特定查询生成**：通过 `GraphProvider` 枚举参数化查询生成函数
5. **Kuzu 中间节点模式**：将边属性提升为节点属性，适配 Kuzu 不支持边全文索引的限制
6. **Neptune 双存储模式**：图数据库 + 搜索引擎的双存储架构
