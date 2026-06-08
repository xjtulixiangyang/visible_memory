# 核心模块分析：MCP 服务器、HTTP 服务器与命名空间系统

## 1. MCP 服务器架构

### 1.1 入口与核心架构

MCP 服务器基于 `FastMCP` 框架构建，服务名为 `Graphiti Agent Memory`，通过 Model Context Protocol 向 AI 代理暴露知识图谱操作能力。

核心架构包含：
- **GraphitiService**: 封装 Graphiti 客户端的全局服务单例，负责初始化 LLM、Embedder、数据库驱动，管理并发信号量（`SEMAPHORE_LIMIT`，默认 10）
- **QueueService**: 按 `group_id` 维度对 Episode 处理进行串行化排队，避免同一分组内的竞态条件
- **FastMCP 实例**: 通过 `@mcp.tool()` 装饰器注册工具函数，通过 `@mcp.custom_route` 注册自定义 HTTP 路由

### 1.2 工具定义

| 工具名称 | 功能 | 关键参数 |
|---|---|---|
| `add_memory` | 添加 Episode 到知识图谱（异步排队处理） | name, episode_body, group_id, source, source_description, uuid |
| `search_nodes` | 搜索实体节点（混合搜索 RRF 策略） | query, group_ids, max_nodes, entity_types |
| `search_memory_facts` | 搜索事实（实体间关系） | query, group_ids, max_facts, center_node_uuid |
| `delete_entity_edge` | 按 UUID 删除实体边 | uuid |
| `delete_episode` | 按 UUID 删除 Episode | uuid |
| `get_entity_edge` | 按 UUID 获取实体边详情 | uuid |
| `get_episodes` | 按 group_ids 获取 Episode 列表 | group_ids, max_episodes |
| `clear_graph` | 清除指定 group_ids 的图谱数据 | group_ids |
| `get_status` | 获取服务器与数据库连接状态 | 无 |

自定义路由：`GET /health` — 健康检查端点

### 1.3 传输层支持

| 模式 | 端点 | 说明 |
|------|------|------|
| HTTP (streamable) | `/mcp/` | 推荐模式 |
| SSE | - | 已废弃 |
| stdio | - | 标准输入输出模式 |

### 1.4 配置体系

配置采用 `pydantic-settings` + YAML + 环境变量的三层合并策略，优先级为：**CLI 参数 > 环境变量 > YAML 配置 > 默认值**。

核心配置模型：
- **GraphitiConfig** (根配置): 包含 server、llm、embedder、database、graphiti 五个子配置
- **ServerConfig**: transport(http/stdio/sse)、host(0.0.0.0)、port(8000)
- **LLMConfig**: provider(openai/azure_openai/anthropic/gemini/groq)、model(gpt-4o-mini)、temperature、max_tokens(4096)
- **EmbedderConfig**: provider(openai/azure_openai/gemini/voyage)、model(text-embedding-3-small)、dimensions(1536)
- **DatabaseConfig**: provider(falkordb/neo4j)
- **GraphitiAppConfig**: group_id(main)、episode_id_prefix、user_id(mcp_user)、entity_types 列表

YAML 配置支持 `${VAR}` 和 `${VAR:default}` 语法进行环境变量插值。

### 1.5 实体类型定义

定义了 9 种预置实体类型：

| 类型 | 用途 | 分类优先级 |
|------|------|-----------|
| Preference | 用户偏好 | 最高 |
| Requirement | 需求/功能规格 | 高 |
| Procedure | 操作步骤/流程 | 高 |
| Event | 时间绑定的事件 | 中 |
| Organization | 组织/公司/机构 | 中 |
| Location | 物理或虚拟位置 | 中 |
| Document | 文档/信息内容 | 低 |
| Topic | 话题/知识领域 | 最后手段 |
| Object | 物理物品/工具 | 最后手段 |

### 1.6 工厂模式

三个工厂类采用静态方法 + `match/case` 模式创建客户端：

- **LLMClientFactory**: 支持 openai（含推理模型特殊处理）、azure_openai、anthropic、gemini、groq
- **EmbedderFactory**: 支持 openai、azure_openai、gemini、voyage
- **DatabaseDriverFactory**: 返回配置字典而非驱动实例，支持 neo4j 和 falkordb

关键设计：通过 `try/except ImportError` 检测可选依赖是否可用，实现优雅降级。

### 1.7 队列服务

按 `group_id` 隔离的串行处理队列：
- 每个 group_id 维护独立的 `asyncio.Queue` 和 worker 协程
- worker 是长生命周期任务，持续从队列取任务执行
- 处理失败时记录错误但不中断 worker

---

## 2. HTTP 服务器架构

### 2.1 入口与架构

HTTP 服务器基于 **FastAPI** 构建，采用 lifespan 模式在启动时初始化 Graphiti 索引和约束。

### 2.2 配置

使用 `pydantic-settings` 的 `BaseSettings`，从 `.env` 文件加载配置：
- openai_api_key, openai_base_url, model_name, embedding_model_name
- neo4j_uri, neo4j_user, neo4j_password
- falkordb_host, falkordb_port, falkordb_database
- db_backend (默认 neo4j)

### 2.3 Zep 集成层

`ZepGraphiti` 继承自 `Graphiti`，扩展了以下方法：
- `save_entity_node`: 创建 EntityNode 并生成 name_embedding 后保存
- `get_entity_edge`: 按 UUID 获取边，404 时抛出 HTTPException
- `delete_group`: 按 group_id 级联删除边、节点、Episode
- `delete_entity_edge` / `delete_episodic_node`: 按 UUID 删除，404 处理

### 2.4 数据传输对象 (DTO)

**common.py**:
- `Result`: 通用操作结果 (message + success)
- `Message`: 消息结构 (content, uuid, name, role_type, role, timestamp, source_description)

**ingest.py**:
- `AddMessagesRequest`: group_id + messages 列表
- `AddEntityNodeRequest`: uuid, group_id, name, summary

**retrieve.py**:
- `SearchQuery`: group_ids, query, max_facts
- `FactResult`: uuid, name, fact, valid_at, invalid_at, created_at, expired_at
- `SearchResults`: facts 列表
- `GetMemoryRequest`: group_id, max_facts, center_node_uuid, messages
- `GetMemoryResponse`: facts 列表

### 2.5 API 路由

**Ingest 路由**:

| 方法 | 路径 | 状态码 | 功能 |
|---|---|---|---|
| POST | `/messages` | 202 | 批量添加消息（异步排队） |
| POST | `/entity-node` | 201 | 添加实体节点 |
| DELETE | `/entity-edge/{uuid}` | 200 | 删除实体边 |
| DELETE | `/group/{group_id}` | 200 | 删除整个分组 |
| DELETE | `/episode/{uuid}` | 200 | 删除 Episode |
| POST | `/clear` | 200 | 清空图谱并重建索引 |

**Retrieve 路由**:

| 方法 | 路径 | 状态码 | 功能 |
|---|---|---|---|
| POST | `/search` | 200 | 搜索事实 |
| GET | `/entity-edge/{uuid}` | 200 | 获取实体边详情 |
| GET | `/episodes/{group_id}` | 200 | 获取 Episode 列表 |
| POST | `/get-memory` | 200 | 基于消息组合查询获取记忆 |

---

## 3. MCP vs HTTP 服务器对比

| 维度 | MCP 服务器 | HTTP 服务器 |
|---|---|---|
| 协议 | MCP (Model Context Protocol) | REST HTTP |
| 框架 | FastMCP | FastAPI |
| 目标用户 | AI 代理/LLM 客户端 | 通用 HTTP 客户端 |
| 搜索能力 | 节点搜索 + 事实搜索 | 仅事实搜索 |
| 并发控制 | Semaphore + 按 group_id 队列 | AsyncWorker (全局单队列) |
| 客户端生命周期 | 全局单例 | 每请求创建/销毁 |
| 配置来源 | YAML + 环境变量 + CLI | .env 文件 |
| 实体类型 | 可配置自定义类型 | 无 |

---

## 4. 命名空间系统

### 4.1 节点命名空间

```
NodeNamespace
  ├── EntityNodeNamespace    (graphiti.nodes.entity)
  ├── EpisodeNodeNamespace   (graphiti.nodes.episode)
  ├── CommunityNodeNamespace (graphiti.nodes.community)
  └── SagaNodeNamespace      (graphiti.nodes.saga)
```

每个子命名空间提供统一的 CRUD 接口：
- `save` / `save_bulk`: 保存节点（Entity 和 Community 保存前自动生成 name_embedding）
- `delete` / `delete_by_group_id` / `delete_by_uuids`: 删除节点
- `get_by_uuid` / `get_by_uuids` / `get_by_group_ids`: 查询节点
- `load_embeddings` / `load_embeddings_bulk`: 按需加载向量嵌入

EpisodeNodeNamespace 额外提供：
- `get_by_entity_node_uuid`: 获取与实体关联的 Episode
- `retrieve_episodes`: 按时间检索最近的 Episode

### 4.2 边命名空间

```
EdgeNamespace
  ├── EntityEdgeNamespace      (graphiti.edges.entity)
  ├── EpisodicEdgeNamespace    (graphiti.edges.episodic)
  ├── CommunityEdgeNamespace   (graphiti.edges.community)
  ├── HasEpisodeEdgeNamespace  (graphiti.edges.has_episode)
  └── NextEpisodeEdgeNamespace (graphiti.edges.next_episode)
```

EntityEdgeNamespace 额外提供：
- `get_between_nodes`: 查询两节点间的边
- `get_by_node_uuid`: 查询与某节点关联的所有边
- `load_embeddings` / `load_embeddings_bulk`: 按需加载 fact_embedding

### 4.3 命名空间初始化机制

命名空间通过驱动（Driver）提供的 Operations 对象初始化。如果某个驱动不支持特定操作（如 `entity_node_ops` 为 None），对应子命名空间不会被设置。访问未设置的子命名空间时，`__getattr__` 会抛出 `NotImplementedError`。

---

## 5. 数据模型与数据库查询层

### 5.1 节点查询 (node_db_queries.py)

针对每种图数据库提供差异化的 Cypher 查询：
- **Episodic 节点**: MERGE 按 uuid，SET 全属性。Neptune 需要将 entity_edges 数组 join 为字符串；Kuzu 需要逐字段 SET
- **Entity 节点**: MERGE 按 uuid，动态设置标签。FalkorDB 使用 `vecf32()` 存储向量；Neo4j 使用 `db.create.setNodeVectorProperty()`；Neptune 将向量 join 为逗号分隔字符串
- **Community 节点**: 类似 Entity，含 name_embedding
- **Saga 节点**: 含 first_episode_uuid、last_episode_uuid 等字段

### 5.2 边查询 (edge_db_queries.py)

- **EpisodicEdge**: MERGE (episode)-[MENTIONS]->(node)，支持 UNWIND 批量操作
- **EntityEdge**: MERGE (source)-[RELATES_TO]->(target)。FalkorDB 使用 `vecf32()` 存储 fact_embedding；Neptune 将 episodes 数组 join 为字符串；Kuzu 将边建模为中间节点 `RelatesToNode_`
- **CommunityEdge**: Neptune/Kuzu 需要 UNION 匹配 Entity 和 Community 两种目标

关键设计决策：
- 向量嵌入默认不返回（需手动 `load_embeddings` / `load_name_embedding`）
- Neptune 的数组/向量存储使用字符串序列化
- Kuzu 的边属性通过中间节点模拟

---

## 6. 集成模式

### 6.1 MCP 服务器集成模式

1. **工厂模式创建客户端**: 根据配置动态创建客户端实例，支持多供应商切换
2. **异步排队处理**: add_memory 工具立即返回，Episode 通过 QueueService 按 group_id 串行处理
3. **信号量控制并发**: 全局 Semaphore 限制同时处理的 Episode 数量
4. **配置多层合并**: YAML + 环境变量 + CLI 参数三层配置
5. **自定义实体类型**: 通过 YAML 配置动态生成 Pydantic 模型

### 6.2 HTTP 服务器集成模式

1. **依赖注入**: 通过 FastAPI 的 `Depends` 注入 ZepGraphiti 实例
2. **异步 Worker**: ingest 路由使用 AsyncWorker 处理消息添加，返回 202 Accepted
3. **Zep 扩展**: ZepGraphiti 继承 Graphiti，添加业务特定的便捷方法
4. **消息到 Episode 的转换**: 将 Message 的 role_type 和 role 拼接为 Episode 的 episode_body

### 6.3 核心库集成模式

1. **命名空间隔离**: 通过 NodeNamespace / EdgeNamespace 将操作按领域隔离
2. **Operations 抽象**: 驱动层通过 Operations 接口暴露数据库操作
3. **多数据库适配**: 查询层通过 `GraphProvider` 枚举分发差异化的 Cypher 查询
4. **向量嵌入延迟加载**: 嵌入向量默认不随节点/边返回，需显式调用 load_embeddings
5. **事务支持**: 命名空间方法接受可选的 `Transaction` 参数
