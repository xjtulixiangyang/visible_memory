# Graphiti 架构设计文档

## 1. 系统总体架构

### 1.1 分层架构总览

```mermaid
graph TB
    subgraph "服务层 Service Layer"
        MCP["MCP Server<br/>(FastMCP)"]
        HTTP["HTTP Server<br/>(FastAPI)"]
    end

    subgraph "核心层 Core Layer"
        G["Graphiti<br/>Facade"]
        NS["Namespace API<br/>NodeNamespace / EdgeNamespace"]
        SEARCH["Search Pipeline"]
        MAINT["Maintenance<br/>Dedup / Community / Summary"]
    end

    subgraph "客户端层 Client Layer"
        LLM["LLM Client<br/>OpenAI / Anthropic / Gemini / Groq / GLiNER2"]
        EMB["Embedder Client<br/>OpenAI / Gemini / Azure / Voyage"]
        CE["CrossEncoder Client<br/>BGE / OpenAI / Gemini"]
        TRACE["Tracer<br/>OpenTelemetry"]
    end

    subgraph "驱动层 Driver Layer"
        DRV["GraphDriver<br/>(Abstract)"]
        N4J["Neo4j Driver"]
        FKDB["FalkorDB Driver"]
        KZ["Kuzu Driver"]
        NPT["Neptune Driver"]
    end

    subgraph "操作层 Operations Layer"
        OPS["Operations<br/>(Abstract)"]
        N4J_OPS["Neo4j Operations"]
        FKDB_OPS["FalkorDB Operations"]
        KZ_OPS["Kuzu Operations"]
        NPT_OPS["Neptune Operations"]
    end

    MCP --> G
    HTTP --> G
    G --> NS
    G --> SEARCH
    G --> MAINT
    G --> LLM
    G --> EMB
    G --> CE
    G --> TRACE
    NS --> DRV
    SEARCH --> DRV
    DRV --> N4J
    DRV --> FKDB
    DRV --> KZ
    DRV --> NPT
    N4J --> N4J_OPS
    FKDB --> FKDB_OPS
    KZ --> KZ_OPS
    NPT --> NPT_OPS
```

### 1.2 模块依赖关系

```mermaid
graph LR
    graphiti["graphiti.py<br/>(Facade)"]
    nodes["nodes.py"]
    edges["edges.py"]
    helpers["helpers.py"]
    types["graphiti_types.py"]
    search["search/"]
    driver["driver/"]
    llm["llm_client/"]
    embedder["embedder/"]
    cross_encoder["cross_encoder/"]
    prompts["prompts/"]
    utils["utils/maintenance/"]
    namespaces["namespaces/"]
    models["models/"]

    graphiti --> nodes
    graphiti --> edges
    graphiti --> types
    graphiti --> search
    graphiti --> driver
    graphiti --> llm
    graphiti --> embedder
    graphiti --> cross_encoder
    graphiti --> utils
    graphiti --> namespaces
    graphiti --> helpers

    search --> driver
    search --> embedder
    search --> cross_encoder

    utils --> llm
    utils --> driver
    utils --> embedder

    nodes --> driver
    nodes --> embedder
    edges --> driver
    edges --> embedder

    namespaces --> driver
    namespaces --> embedder

    llm --> prompts
    driver --> models
```

---

## 2. 核心数据模型

### 2.1 图数据模型

```mermaid
erDiagram
    SagaNode ||--o{ EpisodicNode : "HAS_EPISODE"
    EpisodicNode ||--o{ EpisodicNode : "NEXT_EPISODE"
    EpisodicNode }o--o{ EntityNode : "MENTIONS"
    EntityNode }o--o{ EntityNode : "RELATES_TO"
    CommunityNode }o--o{ EntityNode : "HAS_MEMBER"
    CommunityNode }o--o{ CommunityNode : "HAS_MEMBER"

    SagaNode {
        string uuid PK
        string name
        string group_id
        string summary
        string first_episode_uuid
        string last_episode_uuid
        datetime last_summarized_at
        datetime last_summarized_episode_valid_at
    }

    EpisodicNode {
        string uuid PK
        string name
        string group_id
        string content
        EpisodeType source
        datetime valid_at
        list entity_edges
        dict episode_metadata
    }

    EntityNode {
        string uuid PK
        string name
        string group_id
        list labels
        float_array name_embedding
        string summary
        dict attributes
    }

    CommunityNode {
        string uuid PK
        string name
        string group_id
        float_array name_embedding
        string summary
    }

    EntityEdge {
        string uuid PK
        string name
        string fact
        float_array fact_embedding
        string source_node_uuid FK
        string target_node_uuid FK
        datetime valid_at
        datetime invalid_at
        datetime expired_at
        list episodes
        dict attributes
    }
```

### 2.2 节点继承体系

```mermaid
classDiagram
    class Node {
        <<abstract>>
        +uuid: str
        +name: str
        +group_id: str
        +labels: list~str~
        +created_at: datetime
        +save(driver)*
        +delete(driver)
        +delete_by_group_id(driver, group_id)
        +get_by_uuid(driver, uuid)*
    }

    class EpisodicNode {
        +content: str
        +source: EpisodeType
        +valid_at: datetime
        +entity_edges: list~str~
        +episode_metadata: dict
    }

    class EntityNode {
        +name_embedding: list~float~
        +summary: str
        +attributes: dict
        +generate_name_embedding(embedder)
        +load_name_embedding(driver)
    }

    class CommunityNode {
        +name_embedding: list~float~
        +summary: str
    }

    class SagaNode {
        +summary: str
        +first_episode_uuid: str
        +last_episode_uuid: str
        +last_summarized_at: datetime
    }

    Node <|-- EpisodicNode
    Node <|-- EntityNode
    Node <|-- CommunityNode
    Node <|-- SagaNode
```

### 2.3 边继承体系

```mermaid
classDiagram
    class Edge {
        <<abstract>>
        +uuid: str
        +group_id: str
        +source_node_uuid: str
        +target_node_uuid: str
        +created_at: datetime
        +save(driver)*
        +delete(driver)
    }

    class EpisodicEdge {
        MENTIONS relationship
    }

    class EntityEdge {
        RELATES_TO relationship
        +name: str
        +fact: str
        +fact_embedding: list~float~
        +valid_at: datetime
        +invalid_at: datetime
        +expired_at: datetime
        +episodes: list~str~
        +attributes: dict
        +generate_embedding(embedder)
        +get_between_nodes(driver, source, target)
        +get_by_node_uuid(driver, uuid)
    }

    class CommunityEdge {
        HAS_MEMBER relationship
    }

    class HasEpisodeEdge {
        HAS_EPISODE relationship
    }

    class NextEpisodeEdge {
        NEXT_EPISODE relationship
    }

    Edge <|-- EpisodicEdge
    Edge <|-- EntityEdge
    Edge <|-- CommunityEdge
    Edge <|-- HasEpisodeEdge
    Edge <|-- NextEpisodeEdge
```

---

## 3. 知识摄入流水线

### 3.1 单条添加流程

```mermaid
flowchart TD
    A["用户输入<br/>episode_body"] --> B["验证实体类型"]
    B --> C["处理 group_id<br/>(克隆驱动切换数据库)"]
    C --> D["检索历史片段<br/>作为上下文"]
    D --> E["创建 EpisodicNode"]
    E --> F["extract_nodes()<br/>LLM 提取实体"]
    F --> G["resolve_extracted_nodes()<br/>去重/合并"]
    G --> H["extract_edges()<br/>LLM 提取关系"]
    H --> I["resolve_extracted_edges()<br/>去重/失效处理"]
    I --> J["extract_attributes_from_nodes()<br/>LLM 补充属性"]
    J --> K["_process_episode_data()<br/>持久化"]
    K --> L{需要 Saga 关联?}
    L -->|是| M["创建 HasEpisodeEdge<br/>+ NextEpisodeEdge"]
    L -->|否| N{需要社区更新?}
    M --> N
    N -->|是| O["更新社区"]
    N -->|否| P["返回 AddEpisodeResults"]
    O --> P
```

### 3.2 节点去重策略

```mermaid
flowchart TD
    A["提取的 EntityNode 列表"] --> B["语义候选搜索<br/>embedding 相似度 >= 0.6"]
    B --> C{确定性去重}
    C -->|精确名称匹配| D["直接合并"]
    C -->|多个精确匹配| E["升级到 LLM"]
    C -->|高熵名称<br/>Shannon >= 1.5| F["MinHash/LSH<br/>Jaccard >= 0.9"]
    C -->|低熵名称| E
    F -->|匹配| D
    F -->|不匹配| E
    E --> G["LLM 去重<br/>dedupe_nodes prompt"]
    G --> H["去重后的节点 + UUID 映射"]
```

### 3.3 边去重与矛盾检测

```mermaid
flowchart TD
    A["提取的 EntityEdge 列表"] --> B["精确去重<br/>source+target+fact 归一化"]
    B --> C["查询已有边<br/>get_between_nodes"]
    C --> D["混合搜索相关边<br/>EDGE_HYBRID_SEARCH_RRF"]
    D --> E["搜索矛盾候选边"]
    E --> F{fact 精确匹配?}
    F -->|是| G["直接复用"]
    F -->|否| H["LLM 去重<br/>dedupe_edges prompt"]
    H --> I["duplicate_facts<br/>+ contradicted_facts"]
    I --> J["属性提取 + 时间戳提取"]
    J --> K{存在矛盾?}
    K -->|是| L["矛盾检测算法<br/>设置 invalid_at + expired_at"]
    K -->|否| M["生成 embedding"]
    L --> M
```

### 3.4 矛盾检测算法

```mermaid
flowchart TD
    A{"旧边 invalid_at < 新边 valid_at?"} -->|是| B["不矛盾<br/>旧边已失效"]
    A -->|否| C{"新边 invalid_at < 旧边 valid_at?"}
    C -->|是| D["不矛盾<br/>新边已失效"]
    C -->|否| E{"旧边 valid_at < 新边 valid_at?"}
    E -->|是| F["新边使旧边失效<br/>设置 invalid_at + expired_at"]
    E -->|否| G["时间重叠<br/>需要进一步判断"]
```

---

## 4. 搜索管线架构

### 4.1 搜索总流程

```mermaid
flowchart TD
    A["search_(query, config)"] --> B["校验 group_ids"]
    B --> C["查询向量化<br/>embedder.create()"]
    C --> D["四路并行搜索"]

    D --> E1["edge_search()"]
    D --> E2["node_search()"]
    D --> E3["episode_search()"]
    D --> E4["community_search()"]

    E1 --> F["SearchResults"]
    E2 --> F
    E3 --> F
    E4 --> F
```

### 4.2 子搜索统一模式

```mermaid
flowchart TD
    A["子搜索入口"] --> B["阶段一: 多方法候选召回"]

    B --> C1["BM25 全文检索"]
    B --> C2["Cosine Similarity<br/>向量搜索"]
    B --> C3["BFS 图遍历"]

    C1 --> D["候选集合并<br/>上限 2*limit"]
    C2 --> D
    C3 --> D

    D --> E{配置了 BFS<br/>但无起始节点?}
    E -->|是| F["自适应 BFS<br/>用阶段一结果作为锚点"]
    E -->|否| G["阶段三: 重排"]
    F --> G

    G --> H1["RRF"]
    G --> H2["MMR"]
    G --> H3["CrossEncoder"]
    G --> H4["NodeDistance"]
    G --> H5["EpisodeMentions"]

    H1 --> I["返回 limit 个结果"]
    H2 --> I
    H3 --> I
    H4 --> I
    H5 --> I
```

### 4.3 搜索方法与重排器矩阵

```mermaid
graph TB
    subgraph "搜索方法"
        BM25["BM25<br/>全文检索"]
        COS["Cosine Similarity<br/>向量搜索"]
        BFS["BFS<br/>图遍历"]
    end

    subgraph "Edge 重排器"
        E_RRF["RRF"]
        E_MMR["MMR"]
        E_CE["CrossEncoder"]
        E_ND["NodeDistance"]
        E_EM["EpisodeMentions"]
    end

    subgraph "Node 重排器"
        N_RRF["RRF"]
        N_MMR["MMR"]
        N_CE["CrossEncoder"]
        N_ND["NodeDistance"]
        N_EM["EpisodeMentions"]
    end

    subgraph "Episode 重排器"
        EP_RRF["RRF"]
        EP_CE["CrossEncoder"]
    end

    subgraph "Community 重排器"
        C_RRF["RRF"]
        C_MMR["MMR"]
        C_CE["CrossEncoder"]
    end
```

---

## 5. 驱动层架构

### 5.1 驱动继承体系

```mermaid
classDiagram
    class QueryExecutor {
        <<abstract>>
        +execute_query(cypher, **kwargs)*
    }

    class GraphDriver {
        <<abstract>>
        +provider: GraphProvider
        +fulltext_syntax: str
        +entity_node_ops: EntityNodeOperations
        +episode_node_ops: EpisodeNodeOperations
        +community_node_ops: CommunityNodeOperations
        +saga_node_ops: SagaNodeOperations
        +entity_edge_ops: EntityEdgeOperations
        +episodic_edge_ops: EpisodicEdgeOperations
        +community_edge_ops: CommunityEdgeOperations
        +has_episode_edge_ops: HasEpisodeEdgeOperations
        +next_episode_edge_ops: NextEpisodeEdgeOperations
        +search_ops: SearchOperations
        +graph_ops: GraphMaintenanceOperations
        +execute_query(cypher, **kwargs)*
        +session(database)*
        +close()*
        +build_indices_and_constraints()*
        +with_database(database)
        +transaction()
    }

    class Neo4jDriver {
        事务支持: 真实事务
        全文索引: Lucene
        向量搜索: db.index.vector
    }

    class FalkorDBDriver {
        事务支持: 无
        全文索引: RedisSearch
        向量搜索: vecf32()
    }

    class KuzuDriver {
        事务支持: 无
        全文索引: 不支持
        边属性: 中间节点模拟
    }

    class NeptuneDriver {
        事务支持: 无
        全文索引: AOSS
        向量存储: 逗号分隔字符串
    }

    QueryExecutor <|-- GraphDriver
    GraphDriver <|-- Neo4jDriver
    GraphDriver <|-- FalkorDBDriver
    GraphDriver <|-- KuzuDriver
    GraphDriver <|-- NeptuneDriver
```

### 5.2 操作层组合模式

```mermaid
graph TB
    subgraph "GraphDriver (组合)"
        DRV["driver"]
        DRV --- ENO["entity_node_ops"]
        DRV --- EPNO["episode_node_ops"]
        DRV --- CNO["community_node_ops"]
        DRV --- SNO["saga_node_ops"]
        DRV --- EEO["entity_edge_ops"]
        DRV --- EPEO["episodic_edge_ops"]
        DRV --- CEO["community_edge_ops"]
        DRV --- HEO["has_episode_edge_ops"]
        DRV --- NEO["next_episode_edge_ops"]
        DRV --- SO["search_ops"]
        DRV --- GO["graph_ops"]
    end

    subgraph "Neo4j 实现"
        N4J_ENO["Neo4jEntityNodeOperations"]
        N4J_SO["Neo4jSearchOperations"]
    end

    subgraph "FalkorDB 实现"
        FK_ENO["FalkorEntityNodeOperations"]
        FK_SO["FalkorSearchOperations"]
    end

    ENO -.-> N4J_ENO
    ENO -.-> FK_ENO
    SO -.-> N4J_SO
    SO -.-> FK_SO
```

### 5.3 操作执行模式

```mermaid
sequenceDiagram
    participant Caller as 调用层
    participant NS as Namespace
    participant Ops as Operations
    participant QE as QueryExecutor
    participant DB as 数据库

    Caller->>NS: save(node)
    NS->>Ops: save(executor, node)
    Ops->>Ops: 构建 Cypher + 参数
    alt 有事务
        Ops->>QE: tx.run(query, params)
    else 无事务
        Ops->>QE: executor.execute_query(query, params)
    end
    QE->>DB: 执行 Cypher
    DB-->>QE: 返回结果
    QE-->>Ops: 返回结果
    Ops-->>NS: 返回
    NS-->>Caller: 返回
```

### 5.4 IoC 回退模式

```mermaid
flowchart TD
    A["节点/边数据库操作"] --> B{driver.graph_operations_interface<br/>是否实现?}
    B -->|是| C["调用 IoC 接口"]
    B -->|否/NotImplementedError| D["回退到内联 Cypher"]
    C --> E["返回结果"]
    D --> E
```

---

## 6. LLM 客户端架构

### 6.1 客户端继承体系

```mermaid
classDiagram
    class LLMClient {
        <<abstract>>
        +generate_response(messages, response_model)
        #_generate_response(messages, response_model)*
        #_generate_response_with_retry()
        #_apply_attribute_extraction_preamble()
        #_get_cache_key()
    }

    class BaseOpenAIClient {
        <<abstract>>
        #_create_completion()*
        #_create_structured_completion()*
        #_handle_structured_response()
        #_handle_json_response()
    }

    class OpenAIClient {
        responses.parse API
        支持推理模型
    }

    class AzureOpenAILLMClient {
        区分推理/普通模型
    }

    class OpenAIGenericClient {
        json_schema response_format
        兼容本地模型
    }

    class AnthropicClient {
        Tool Use 结构化输出
        模型特定 max_tokens
    }

    class GeminiClient {
        response_schema 结构化输出
        安全过滤检查
        JSON 截断抢救
    }

    class GroqClient {
        json_object 响应格式
        最简实现
    }

    class GLiNER2Client {
        委托模式
        本地 GLiNER2 实体提取
        其他操作委托 llm_client
    }

    LLMClient <|-- BaseOpenAIClient
    LLMClient <|-- OpenAIGenericClient
    LLMClient <|-- AnthropicClient
    LLMClient <|-- GeminiClient
    LLMClient <|-- GroqClient
    LLMClient <|-- GLiNER2Client
    BaseOpenAIClient <|-- OpenAIClient
    BaseOpenAIClient <|-- AzureOpenAILLMClient
```

### 6.2 LLM 调用流程

```mermaid
sequenceDiagram
    participant Caller as 调用方
    participant LLM as LLMClient
    participant Cache as LLMCache
    participant Tracer as OpenTelemetry
    participant API as LLM API

    Caller->>LLM: generate_response(messages, response_model)
    LLM->>LLM: _apply_attribute_extraction_preamble()
    LLM->>LLM: _get_cache_key()
    LLM->>Cache: 查询缓存
    alt 缓存命中
        Cache-->>LLM: 返回缓存结果
    else 缓存未命中
        LLM->>Tracer: start span
        LLM->>API: _generate_response_with_retry()
        alt 成功
            API-->>LLM: 返回响应
            LLM->>Cache: 写入缓存
        else 失败
            API-->>LLM: 抛出异常
            LLM->>LLM: tenacity 重试
        end
        LLM->>Tracer: end span
    end
    LLM-->>Caller: 返回结果
```

---

## 7. Prompt 模板系统

### 7.1 版本化 Prompt 库

```mermaid
graph TB
    subgraph "PromptLibraryWrapper"
        EN["extract_nodes"]
        DN["dedupe_nodes"]
        EE["extract_edges"]
        ENE["extract_nodes_and_edges"]
        DE["dedupe_edges"]
        SN["summarize_nodes"]
        SS["summarize_sagas"]
        EV["eval"]
    end

    subgraph "extract_nodes 版本"
        EN_M["extract_message"]
        EN_J["extract_json"]
        EN_T["extract_text"]
        EN_C["classify_nodes"]
        EN_A["extract_attributes"]
        EN_S["extract_summary"]
        EN_SB["extract_summaries_batch"]
        EN_ES["extract_entity_summaries_from_episodes"]
    end

    EN --> EN_M
    EN --> EN_J
    EN --> EN_T
    EN --> EN_C
    EN --> EN_A
    EN --> EN_S
    EN --> EN_SB
    EN --> EN_ES
```

### 7.2 Prompt 调用链

```mermaid
flowchart LR
    A["PromptLibraryWrapper<br/>.<type>.<version>(context)"] --> B["VersionWrapper<br/>追加 DO_NOT_ESCAPE_UNICODE"]
    B --> C["PromptFunction(context)<br/>生成 Message 列表"]
    C --> D["LLMClient<br/>.generate_response(messages, model)"]
    D --> E["结构化 Pydantic 模型"]
```

---

## 8. 服务层架构

### 8.1 MCP 服务器架构

```mermaid
flowchart TD
    subgraph "MCP Server"
        FMCP["FastMCP<br/>Graphiti Agent Memory"]

        subgraph "工具 Tools"
            T1["add_memory"]
            T2["search_nodes"]
            T3["search_memory_facts"]
            T4["delete_entity_edge"]
            T5["delete_episode"]
            T6["get_entity_edge"]
            T7["get_episodes"]
            T8["clear_graph"]
            T9["get_status"]
        end

        subgraph "服务层"
            GS["GraphitiService<br/>单例 + Semaphore"]
            QS["QueueService<br/>按 group_id 隔离"]
        end

        subgraph "工厂层"
            LF["LLMClientFactory"]
            EF["EmbedderFactory"]
            DF["DatabaseDriverFactory"]
        end

        subgraph "配置层"
            CFG["GraphitiConfig<br/>YAML + 环境变量 + CLI"]
        end
    end

    FMCP --> T1 & T2 & T3 & T4 & T5 & T6 & T7 & T8 & T9
    T1 --> QS
    QS --> GS
    T2 & T3 --> GS
    GS --> LF & EF & DF
    LF & EF & DF --> CFG
```

### 8.2 HTTP 服务器架构

```mermaid
flowchart TD
    subgraph "HTTP Server (FastAPI)"
        MAIN["main.py<br/>lifespan 启动"]

        subgraph "路由层"
            RI["/messages (POST)<br/>/entity-node (POST)<br/>/entity-edge/{uuid} (DELETE)<br/>/group/{group_id} (DELETE)<br/>/episode/{uuid} (DELETE)<br/>/clear (POST)"]
            RR["/search (POST)<br/>/entity-edge/{uuid} (GET)<br/>/episodes/{group_id} (GET)<br/>/get-memory (POST)"]
        end

        subgraph "业务层"
            ZG["ZepGraphiti<br/>继承 Graphiti"]
            AW["AsyncWorker<br/>全局单队列"]
        end

        subgraph "DTO 层"
            DTO_I["AddMessagesRequest<br/>AddEntityNodeRequest"]
            DTO_R["SearchQuery<br/>FactResult<br/>GetMemoryRequest<br/>GetMemoryResponse"]
        end
    end

    MAIN --> RI & RR
    RI --> ZG & AW
    RR --> ZG
    RI --> DTO_I
    RR --> DTO_R
```

### 8.3 MCP vs HTTP 对比

```mermaid
graph LR
    subgraph "MCP Server"
        direction TB
        M_P["协议: MCP"]
        M_F["框架: FastMCP"]
        M_T["目标: AI 代理"]
        M_S["搜索: 节点+事实"]
        M_Q["并发: Semaphore+队列"]
        M_L["生命周期: 全局单例"]
    end

    subgraph "HTTP Server"
        direction TB
        H_P["协议: REST HTTP"]
        H_F["框架: FastAPI"]
        H_T["目标: 通用客户端"]
        H_S["搜索: 仅事实"]
        H_Q["并发: AsyncWorker"]
        H_L["生命周期: 每请求"]
    end
```

---

## 9. 命名空间系统

### 9.1 命名空间层级

```mermaid
graph TB
    subgraph "NodeNamespace"
        N_ENT["nodes.entity<br/>EntityNodeNamespace"]
        N_EPI["nodes.episode<br/>EpisodeNodeNamespace"]
        N_COM["nodes.community<br/>CommunityNodeNamespace"]
        N_SAG["nodes.saga<br/>SagaNodeNamespace"]
    end

    subgraph "EdgeNamespace"
        E_ENT["edges.entity<br/>EntityEdgeNamespace"]
        E_EPI["edges.episodic<br/>EpisodicEdgeNamespace"]
        E_COM["edges.community<br/>CommunityEdgeNamespace"]
        E_HAS["edges.has_episode<br/>HasEpisodeEdgeNamespace"]
        E_NXT["edges.next_episode<br/>NextEpisodeEdgeNamespace"]
    end

    subgraph "驱动能力注册"
        DRV["GraphDriver"]
        DRV -->|"entity_node_ops != None"| N_ENT
        DRV -->|"episode_node_ops != None"| N_EPI
        DRV -->|"community_node_ops != None"| N_COM
        DRV -->|"entity_edge_ops != None"| E_ENT
    end
```

### 9.2 命名空间初始化与回退

```mermaid
flowchart TD
    A["访问 graphiti.nodes.entity"] --> B{"driver.entity_node_ops<br/>是否已注册?"}
    B -->|是| C["返回 EntityNodeNamespace"]
    B -->|否| D["__getattr__ 抛出<br/>NotImplementedError"]
    C --> E["调用 ops.save(executor, node)"]
    E --> F["生成 name_embedding"]
    F --> G["持久化到数据库"]
```

---

## 10. 社区检测算法

### 10.1 标签传播

```mermaid
flowchart TD
    A["初始化: 每个节点一个社区"] --> B["迭代"]
    B --> C["每个节点采用邻居中<br/>边数加权的多数社区"]
    C --> D["平局打破: 选择最大社区"]
    D --> E{有社区变化?}
    E -->|是| B
    E -->|否| F["收敛, 返回社区划分"]
```

### 10.2 层次化摘要合并

```mermaid
flowchart TD
    A["社区内所有实体摘要"] --> B["配对"]
    B --> C["每对通过 LLM<br/>summarize_pair 合并"]
    C --> D{剩余 > 1?}
    D -->|是| B
    D -->|否| E["最终摘要"]
    E --> F["generate_summary_description<br/>生成社区名称"]
```

---

## 11. 嵌入与重排架构

### 11.1 Embedder 客户端体系

```mermaid
classDiagram
    class EmbedderClient {
        <<abstract>>
        +embedding_dim: int
        +create(input_data)* list~float~
        +create_batch(input_data_list) list~list~float~~
    }

    class OpenAIEmbedder {
        model: text-embedding-3-small
        支持维度截断
    }

    class GeminiEmbedder {
        model: text-embedding-001
        batch_size=1 (特定模型)
    }

    class AzureOpenAIEmbedderClient {
        model: text-embedding-3-small
        无维度截断
    }

    class VoyageAIEmbedder {
        model: voyage-3
        支持维度截断
    }

    EmbedderClient <|-- OpenAIEmbedder
    EmbedderClient <|-- GeminiEmbedder
    EmbedderClient <|-- AzureOpenAIEmbedderClient
    EmbedderClient <|-- VoyageAIEmbedder
```

### 11.2 CrossEncoder 客户端体系

```mermaid
classDiagram
    class CrossEncoderClient {
        <<abstract>>
        +rank(query, passages)* list~tuple~str,float~~
    }

    class BGERerankerClient {
        model: BAAI/bge-reranker-v2-m3
        本地 sentence-transformers
        同步转异步
    }

    class OpenAIRerankerClient {
        model: gpt-4.1-nano
        logprob 布尔分类器
        logit_bias 限制输出
    }

    class GeminiRerankerClient {
        model: gemini-2.5-flash-lite
        直接评分 0-100
        正则提取数字
    }

    CrossEncoderClient <|-- BGERerankerClient
    CrossEncoderClient <|-- OpenAIRerankerClient
    CrossEncoderClient <|-- GeminiRerankerClient
```

---

## 12. 关键设计决策

### 12.1 QueryExecutor 解耦

操作类仅依赖 `QueryExecutor` 接口而非 `GraphDriver`，消除循环导入：

```mermaid
graph LR
    Ops["Operations<br/>(抽象操作类)"] -->|"依赖"| QE["QueryExecutor<br/>(最小接口)"]
    DRV["GraphDriver"] -->|"继承"| QE
    DRV -->|"组合"| Ops
```

### 12.2 操作组合优于继承

```mermaid
graph LR
    DRV["GraphDriver"] -->|"组合 11 个"| ENO["EntityNodeOperations"]
    DRV -->|"组合"| EEO["EntityEdgeOperations"]
    DRV -->|"组合"| SO["SearchOperations"]
    DRV -->|"组合"| GO["GraphMaintenanceOperations"]
    DRV -->|"组合"| "..."]
```

### 12.3 Kuzu 中间节点模式

```mermaid
graph LR
    E1["Entity"] -->|"RELATES_TO"| RN["RelatesToNode_<br/>(中间节点)"]
    RN -->|"RELATES_TO"| E2["Entity"]
```

### 12.4 Neptune 双存储架构

```mermaid
graph TB
    APP["应用层"] --> NPT["Neptune<br/>(图数据)"]
    APP --> AOSS["Amazon OpenSearch<br/>(全文索引)"]
    NPT -.->|"向量存储为<br/>逗号分隔字符串"| APP
    AOSS -.->|"4 个索引"| APP
```

### 12.5 GLiNER2 委托模式

```mermaid
flowchart TD
    A["GLiNER2Client.generate_response()"] --> B{"response_model ==<br/>ExtractedEntities?"}
    B -->|是| C["本地 GLiNER2<br/>模型推理"]
    B -->|否| D["委托给内部<br/>llm_client"]
```
