# 核心模块分析：Graphiti 主入口与数据模型

## 1. graphiti.py — 系统主入口

### 1.1 Graphiti 类

Graphiti 是整个知识图谱系统的门面(Facade)，封装了所有核心操作。

**构造函数参数**：
- `uri`/`user`/`password`: Neo4j 数据库连接凭证
- `llm_client`: LLM 客户端(默认 OpenAIClient)
- `embedder`: 嵌入客户端(默认 OpenAIEmbedder)
- `cross_encoder`: 重排序客户端(默认 OpenAIRerankerClient)
- `graph_driver`: 图数据库驱动(默认 Neo4jDriver)
- `store_raw_episode_content`: 是否存储原始片段内容
- `max_coroutines`: 最大并发协程数
- `tracer`: OpenTelemetry 追踪器

**初始化流程**：
1. 建立图数据库驱动连接
2. 初始化 LLM/嵌入/重排序客户端
3. 创建追踪器并注入到 LLM 客户端
4. 将所有客户端打包为 `GraphitiClients` 对象
5. 初始化命名空间 API (`NodeNamespace`, `EdgeNamespace`)
6. 发送遥测事件

### 1.2 核心方法

#### add_episode() — 知识摄入

处理流水线：
1. 验证实体类型与排除类型
2. 处理 group_id（若提供则克隆驱动切换数据库）
3. 检索历史片段作为上下文
4. 创建 EpisodicNode
5. **提取节点**: 调用 `extract_nodes()` 从片段中提取实体
6. **解析节点**: 调用 `resolve_extracted_nodes()` 去重/合并
7. **提取边**: 调用 `extract_edges()` 从片段中提取关系
8. **解析边**: 调用 `resolve_extracted_edges()` 去重/失效处理
9. **属性提取**: 调用 `extract_attributes_from_nodes()` 为节点补充属性摘要
10. **保存数据**: 调用 `_process_episode_data()` 持久化节点、边、片段边
11. **Saga 关联**: 若提供 saga 参数，创建 HAS_EPISODE 和 NEXT_EPISODE 边
12. **社区更新**: 若 `update_communities=True`，更新相关社区

#### add_episode_bulk() — 批量知识摄入

与单条添加的区别：
- 先保存所有片段，再批量检索上下文
- 使用 `_extract_and_dedupe_nodes_bulk()` 进行内存去重
- 使用 `_resolve_nodes_and_edges_bulk()` 批量解析
- 边通过 `dedupe_edges_bulk()` 去重
- Saga 关联按 `valid_at` 排序构建 NEXT_EPISODE 链

#### search() / search_() — 搜索方法

- `search()`: 基础混合搜索，返回 `list[EntityEdge]`
- `search_()`: 高级搜索，支持自定义 SearchConfig、BFS 起始节点、过滤器等，返回 `SearchResults`

#### add_triplet() — 手动添加三元组

处理流程：生成嵌入 → 解析源/目标节点(若不存在则去重解析) → 搜索相关边 → 调用 `resolve_extracted_edge()` 处理冲突 → 保存

#### remove_episode() — 删除片段

仅删除由该片段创建的边，以及仅被该片段引用的节点。

#### build_communities() — 构建社区

先清除已有社区，再运行聚类算法，生成社区节点和边。

#### summarize_saga() — 增量摘要

维护两个水位标记：
- `last_summarized_at`: 墙钟时间，作为下次运行过滤条件
- `last_summarized_episode_valid_at`: 事件时间，表示摘要覆盖的最新参考时间

### 1.3 结果模型类

| 类名 | 用途 |
|------|------|
| `AddEpisodeResults` | 单个片段添加的结果，包含 episode、episodic_edges、nodes、edges、communities、community_edges |
| `AddBulkEpisodeResults` | 批量片段添加的结果，结构与上面类似但 episodes 为列表 |
| `AddTripletResults` | 三元组添加结果，仅包含 nodes 和 edges |

---

## 2. graphiti_types.py — 类型定义

### GraphitiClients — 客户端聚合容器

```python
class GraphitiClients(BaseModel):
    driver: GraphDriver          # 图数据库驱动
    llm_client: LLMClient        # LLM 客户端
    embedder: EmbedderClient      # 嵌入客户端
    cross_encoder: CrossEncoderClient  # 重排序客户端
    tracer: Tracer               # OpenTelemetry 追踪器
```

设计意义：避免在各函数间逐一传递多个客户端参数，简化了函数签名和调用链。

---

## 3. nodes.py — 节点定义

### 3.1 类继承体系

```
Node (ABC, BaseModel)
  ├── EpisodicNode    -- 片段节点(原始输入数据)
  ├── EntityNode      -- 实体节点(提取的知识实体)
  ├── CommunityNode   -- 社区节点(聚类结果)
  └── SagaNode        -- 传奇节点(叙事线索)
```

### 3.2 Node 抽象基类

| 字段 | 类型 | 说明 |
|------|------|------|
| `uuid` | str | 唯一标识，自动生成 |
| `name` | str | 节点名称 |
| `group_id` | str | 图分区标识 |
| `labels` | list[str] | 节点标签 |
| `created_at` | datetime | 创建时间 |

关键方法：
- `save(driver)`: 抽象方法，子类实现
- `delete(driver)`: 删除节点及其关联边，按 provider 分支处理
- `delete_by_group_id()`: 按组批量删除
- `delete_by_uuids()`: 按 UUID 列表批量删除

**IoC 模式**：每个数据库操作方法都先尝试 `driver.graph_operations_interface`，若抛出 `NotImplementedError` 则回退到原生 Cypher 查询。

### 3.3 EpisodicNode — 片段节点

| 额外字段 | 类型 | 说明 |
|---------|------|------|
| `source` | EpisodeType | 来源类型(message/json/text/fact_triple) |
| `source_description` | str | 数据源描述 |
| `content` | str | 原始片段数据 |
| `valid_at` | datetime | 原始文档创建时间 |
| `entity_edges` | list[str] | 引用的实体边 UUID 列表 |
| `episode_metadata` | dict | 自定义元数据 |

### 3.4 EntityNode — 实体节点

| 额外字段 | 类型 | 说明 |
|---------|------|------|
| `name_embedding` | list[float] | 名称嵌入向量 |
| `summary` | str | 周边边的区域摘要 |
| `attributes` | dict[str, Any] | 附加属性(依赖标签类型) |

关键方法：
- `generate_name_embedding(embedder)`: 生成名称嵌入
- `load_name_embedding(driver)`: 从数据库加载嵌入
- `save(driver)`: 保存时将 attributes 展开为顶级属性(Kuzu 除外，需 JSON 序列化)

### 3.5 EpisodeType 枚举

| 值 | 说明 |
|------|------|
| `message` | 消息类型，格式为 "actor: content" |
| `json` | JSON 结构化数据 |
| `text` | 纯文本 |
| `fact_triple` | 事实三元组 |

---

## 4. edges.py — 边定义

### 4.1 类继承体系

```
Edge (ABC, BaseModel)
  ├── EpisodicEdge      -- 片段边(Episodic -MENTIONS-> Entity)
  ├── EntityEdge        -- 实体边(Entity -RELATES_TO-> Entity)
  ├── CommunityEdge     -- 社区边(Community -HAS_MEMBER-> Entity)
  ├── HasEpisodeEdge    -- 传奇-片段边(Saga -HAS_EPISODE-> Episodic)
  └── NextEpisodeEdge   -- 片段链边(Episodic -NEXT_EPISODE-> Episodic)
```

### 4.2 EntityEdge — 核心边类型

| 字段 | 类型 | 说明 |
|------|------|------|
| `name` | str | 关系名称 |
| `fact` | str | 事实描述 |
| `fact_embedding` | list[float] | 事实嵌入向量 |
| `episodes` | list[str] | 引用此边的片段 UUID 列表 |
| `valid_at` | datetime | 事实成立时间 |
| `invalid_at` | datetime | 事实失效时间 |
| `expired_at` | datetime | 事实过期时间(被新信息取代) |
| `reference_time` | datetime | 产生此边的片段参考时间 |
| `attributes` | dict[str, Any] | 附加属性 |

时间语义：
- `valid_at`/`invalid_at`：事实在现实世界中的有效时间窗口
- `expired_at`：系统层面的过期时间(被新信息取代)

---

## 5. helpers.py — 辅助函数

| 函数 | 用途 |
|------|------|
| `parse_db_date()` | 将 Neo4j 时间类型或 ISO 字符串转为 Python datetime |
| `get_default_group_id()` | 根据 provider 返回默认 group_id |
| `lucene_sanitize()` | 转义 Lucene 全文检索特殊字符 |
| `normalize_l2()` | L2 归一化嵌入向量 |
| `semaphore_gather()` | 带信号量限制的 `asyncio.gather()` |
| `validate_group_id()` | 验证 group_id 仅含合法字符 |

### 重要常量

| 常量 | 默认值 | 说明 |
|------|--------|------|
| `SEMAPHORE_LIMIT` | 20 | 默认最大并发协程数 |
| `CHUNK_TOKEN_SIZE` | 3000 | 内容分块 token 大小 |
| `CHUNK_OVERLAP_TOKENS` | 200 | 分块重叠 token 数 |
| `CHUNK_DENSITY_THRESHOLD` | 0.15 | 实体密度阈值 |

---

## 6. errors.py — 错误类型

```
GraphitiError (Exception)
  ├── EdgeNotFoundError
  ├── EdgesNotFoundError
  ├── GroupsEdgesNotFoundError
  ├── GroupsNodesNotFoundError
  ├── NodeNotFoundError
  ├── SearchRerankerError
  ├── EntityTypeValidationError
  ├── GroupIdValidationError
  └── NodeLabelValidationError (同时继承 ValueError)
```

---

## 7. decorators.py — 装饰器

### `handle_multiple_group_ids` 装饰器

为 FalkorDB 处理多 group_id 场景。FalkorDB 不支持跨数据库查询，装饰器自动将调用拆分为每个 group_id 单独执行，然后合并结果。

合并逻辑：
- `SearchResults`: 调用 `SearchResults.merge()` 合并
- `list`: 展平合并
- `tuple`: 逐分量展平合并
- 其他类型: 直接返回
