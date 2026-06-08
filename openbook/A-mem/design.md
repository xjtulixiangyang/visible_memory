# A-MEM: Agentic Memory 设计文档

## 1. 系统架构总览

### 1.1 分层架构

```mermaid
graph TB
    subgraph 应用层
        APP[应用/Agent]
        EXAMPLE[示例: sovereign_memory.py]
    end

    subgraph 核心层
        AMS[AgenticMemorySystem<br/>记忆系统核心]
        MN[MemoryNote<br/>记忆数据模型]
    end

    subgraph LLM 控制层
        LLMC[LLMController<br/>LLM 统一控制器]
        OAI[OpenAIController]
        OLL[OllamaController]
    end

    subgraph 检索层
        CR[ChromaRetriever<br/>向量检索器]
        PCR[PersistentChromaRetriever<br/>持久化检索器]
        CCR[CopiedChromaRetriever<br/>复制检索器]
    end

    subgraph 存储层
        MEM[内存字典<br/>memories: Dict]
        CDB[ChromaDB<br/>向量数据库]
    end

    APP --> AMS
    EXAMPLE --> AMS
    AMS --> MN
    AMS --> LLMC
    AMS --> CR
    LLMC --> OAI
    LLMC --> OLL
    CR --> PCR
    PCR --> CCR
    AMS --> MEM
    CR --> CDB
```

### 1.2 模块依赖关系

```mermaid
graph LR
    A[memory_system.py] --> B[llm_controller.py]
    A --> C[retrievers.py]
    B --> D[litellm]
    B --> E[openai]
    C --> F[chromadb]
    C --> G[sentence-transformers]
    C --> H[nltk]
    A --> I[rank_bm25]
    A --> J[scikit-learn]
```

---

## 2. 核心流程设计

### 2.1 记忆添加流程

```mermaid
flowchart TD
    START([add_note 调用]) --> CREATE[创建 MemoryNote 对象]
    CREATE --> PROCESS[process_memory 演化处理]
    
    PROCESS --> FIRST{是否为第一条记忆?}
    FIRST -->|是| SKIP[跳过演化, 返回 note]
    FIRST -->|否| SEARCH[find_related_memories<br/>查找最近邻记忆]
    
    SEARCH --> HAS_NEIGHBOR{有邻居记忆?}
    HAS_NEIGHBOR -->|否| SKIP
    HAS_NEIGHBOR -->|是| LLM_CALL[调用 LLM 进行演化决策]
    
    LLM_CALL --> EVOLVE{should_evolve?}
    EVOLVE -->|否| SKIP
    EVOLVE -->|是| ACTIONS{执行演化动作}
    
    ACTIONS --> STRENGTHEN[strengthen:<br/>添加链接 + 更新标签]
    ACTIONS --> UPDATE_NB[update_neighbor:<br/>更新邻居上下文和标签]
    
    STRENGTHEN --> DONE_EVOLVE
    UPDATE_NB --> DONE_EVOLVE[演化完成]
    SKIP --> DONE_EVOLVE
    
    DONE_EVOLVE --> STORE[存储到 memories 字典]
    STORE --> CHROMA[添加到 ChromaDB]
    CHROMA --> CHECK_EVO{evo_cnt % threshold == 0?}
    CHECK_EVO -->|是| CONSOLIDATE[consolidate_memories<br/>重建索引]
    CHECK_EVO -->|否| RETURN([返回 memory_id])
    CONSOLIDATE --> RETURN
```

### 2.2 记忆演化决策流程

```mermaid
flowchart TD
    START([process_memory]) --> BUILD_PROMPT[构建演化 Prompt]
    BUILD_PROMPT --> FILL[填充: content, context,<br/>keywords, nearest_neighbors]
    FILL --> LLM[LLM 返回 JSON 决策]
    
    LLM --> PARSE[解析 JSON 响应]
    PARSE --> SHOULD{should_evolve?}
    
    SHOULD -->|false| NO_EVOLVE([返回 False, note])
    
    SHOULD -->|true| ITERATE[遍历 actions 列表]
    ITERATE --> ACT_STRENGTHEN{action == strengthen?}
    ITERATE --> ACT_UPDATE{action == update_neighbor?}
    
    ACT_STRENGTHEN -->|是| DO_LINK[note.links.extend<br/>suggested_connections]
    DO_LINK --> DO_TAGS[note.tags = tags_to_update]
    DO_TAGS --> NEXT_ACTION
    
    ACT_UPDATE -->|是| DO_CTX[更新邻居 context<br/>new_context_neighborhood]
    DO_CTX --> DO_NTAGS[更新邻居 tags<br/>new_tags_neighborhood]
    DO_NTAGS --> NEXT_ACTION
    
    NEXT_ACTION{还有更多 action?}
    NEXT_ACTION -->|是| ITERATE
    NEXT_ACTION -->|否| RETURN_EVOLVE([返回 True, note])
```

### 2.3 检索流程

```mermaid
flowchart TD
    START([search_agentic]) --> EMPTY{memories 为空?}
    EMPTY -->|是| RETURN_EMPTY([返回空列表])
    EMPTY -->|否| CHROMA_SEARCH[ChromaDB 语义搜索]
    
    CHROMA_SEARCH --> PROCESS_RESULTS[处理搜索结果]
    PROCESS_RESULTS --> BUILD_DICT[构建结果字典<br/>含 id, content, context,<br/>keywords, tags, score]
    
    BUILD_DICT --> ADD_NEIGHBORS[添加邻居记忆]
    ADD_NEIGHBORS --> CHECK_LINKS{记忆有 links?}
    CHECK_LINKS -->|是| FETCH_LINKED[获取链接记忆详情]
    CHECK_LINKS -->|否| NEXT_MEM
    
    FETCH_LINKED --> ADD_RESULT[添加到结果列表<br/>is_neighbor=True]
    ADD_RESULT --> NEXT_MEM{还有更多记忆?}
    NEXT_MEM -->|是| CHECK_LINKS
    NEXT_MEM -->|否| RETURN_K([返回前 k 个结果])
```

---

## 3. 类设计

### 3.1 类继承关系

```mermaid
classDiagram
    class BaseLLMController {
        <<abstract>>
        +get_completion(prompt, response_format, temperature)* str
    }

    class OpenAIController {
        -client: OpenAI
        -model: str
        +get_completion(prompt, response_format, temperature) str
    }

    class OllamaController {
        -model: str
        +get_completion(prompt, response_format, temperature) str
        -_generate_empty_value(schema_type, schema_items) Any
        -_generate_empty_response(response_format) dict
    }

    class LLMController {
        -llm: BaseLLMController
        +get_completion(prompt, response_format, temperature) str
    }

    BaseLLMController <|-- OpenAIController
    BaseLLMController <|-- OllamaController
    LLMController o-- BaseLLMController

    class ChromaRetriever {
        -client: chromadb.Client
        -embedding_function: SentenceTransformerEmbeddingFunction
        -collection: chromadb.Collection
        +add_document(document, metadata, doc_id)
        +delete_document(doc_id)
        +search(query, k) Dict
        -_convert_metadata_types(metadatas) List
        -_convert_metadata_dict(metadata)
    }

    class PersistentChromaRetriever {
        -collection_name: str
        +__init__(directory, collection_name, model_name, extend)
    }

    class CopiedChromaRetriever {
        -_src_client: chromadb.PersistentClient
        -_dst_client: chromadb.PersistentClient
        -_tmpdir: TemporaryDirectory
        +close()
        +__exit__()
    }

    ChromaRetriever <|-- PersistentChromaRetriever
    PersistentChromaRetriever <|-- CopiedChromaRetriever

    class MemoryNote {
        +id: str
        +content: str
        +keywords: List~str~
        +links: List~str~
        +context: str
        +category: str
        +tags: List~str~
        +timestamp: str
        +last_accessed: str
        +retrieval_count: int
        +evolution_history: List
    }

    class AgenticMemorySystem {
        -memories: Dict~str, MemoryNote~
        -retriever: ChromaRetriever
        -llm_controller: LLMController
        -evo_cnt: int
        -evo_threshold: int
        -_evolution_system_prompt: str
        +add_note(content, time, **kwargs) str
        +read(memory_id) MemoryNote
        +update(memory_id, **kwargs) bool
        +delete(memory_id) bool
        +search(query, k) List~Dict~
        +search_agentic(query, k) List~Dict~
        +find_related_memories(query, k) Tuple
        +find_related_memories_raw(query, k) str
        +analyze_content(content) Dict
        +process_memory(note) Tuple~bool, MemoryNote~
        +consolidate_memories()
    }

    AgenticMemorySystem o-- MemoryNote
    AgenticMemorySystem o-- ChromaRetriever
    AgenticMemorySystem o-- LLMController
```

---

## 4. 数据流设计

### 4.1 记忆写入数据流

```mermaid
sequenceDiagram
    participant User as 用户/Agent
    participant AMS as AgenticMemorySystem
    participant LLM as LLMController
    participant CR as ChromaRetriever
    participant CDB as ChromaDB

    User->>AMS: add_note(content, tags, ...)
    AMS->>AMS: 创建 MemoryNote
    
    alt 非第一条记忆
        AMS->>CR: search(note.content, k=5)
        CR->>CDB: query(query_texts, n_results=k)
        CDB-->>CR: 返回最近邻结果
        CR-->>AMS: neighbors_text, indices
        
        AMS->>LLM: get_completion(evolution_prompt)
        LLM-->>AMS: 演化决策 JSON
        
        alt should_evolve == True
            AMS->>AMS: 执行 strengthen / update_neighbor
        end
    end
    
    AMS->>AMS: memories[note.id] = note
    AMS->>CR: add_document(content, metadata, id)
    CR->>CDB: collection.add(documents, metadatas, ids)
    
    alt evo_cnt % threshold == 0
        AMS->>AMS: consolidate_memories()
        AMS->>CR: 重建 ChromaRetriever
        loop 每个记忆
            AMS->>CR: add_document(...)
        end
    end
    
    AMS-->>User: 返回 memory_id
```

### 4.2 记忆检索数据流

```mermaid
sequenceDiagram
    participant User as 用户/Agent
    participant AMS as AgenticMemorySystem
    participant CR as ChromaRetriever
    participant CDB as ChromaDB
    participant MEM as memories Dict

    User->>AMS: search_agentic(query, k=5)
    AMS->>CR: search(query, k)
    CR->>CDB: collection.query(query_texts, n_results=k)
    CDB-->>CR: ids, metadatas, distances
    CR-->>AMS: 搜索结果
    
    loop 每个搜索结果
        AMS->>AMS: 构建结果字典 (从 metadata)
        
        alt 记忆有 links
            loop 每个 link_id
                AMS->>MEM: memories.get(link_id)
                MEM-->>AMS: 邻居 MemoryNote
                AMS->>AMS: 添加邻居到结果 (is_neighbor=True)
            end
        end
    end
    
    AMS-->>User: 返回前 k 个结果
```

---

## 5. 存储设计

### 5.1 双重存储架构

```mermaid
graph TB
    subgraph 内存存储
        DICT["memories: Dict[str, MemoryNote]<br/>完整 MemoryNote 对象<br/>支持快速 ID 查找"]
    end
    
    subgraph 向量数据库
        COLL["ChromaDB Collection<br/>documents: 文本内容<br/>metadatas: 序列化元数据<br/>ids: 文档 ID<br/>embeddings: 自动生成向量"]
    end
    
    DICT -.->|ID 一致性| COLL
    
    subgraph 写入
        W1[add_note] -->|存储对象| DICT
        W1 -->|存储向量+元数据| COLL
    end
    
    subgraph 读取
        R1[read by ID] -->|直接查找| DICT
        R2[search by query] -->|向量搜索| COLL
    end
```

### 5.2 元数据序列化策略

| 字段类型 | 存储方式 | 反序列化方式 |
|----------|----------|--------------|
| `str` | 直接存储 | 直接读取 |
| `int` | `str(value)` | `ast.literal_eval()` |
| `List[str]` | `json.dumps()` | `ast.literal_eval()` |
| `Dict` | `json.dumps()` | `ast.literal_eval()` |

---

## 6. LLM 交互设计

### 6.1 Prompt 设计

```mermaid
graph TB
    subgraph 内容分析 Prompt
        A1["输入: 记忆文本内容"]
        A2["输出: JSON {keywords, context, tags}"]
        A1 --> A2
    end
    
    subgraph 演化决策 Prompt
        B1["输入: 新记忆 (content, context, keywords)<br/>+ 最近邻记忆列表"]
        B2["输出: JSON {should_evolve, actions,<br/>suggested_connections, tags_to_update,<br/>new_context_neighborhood,<br/>new_tags_neighborhood}"]
        B1 --> B2
    end
```

### 6.2 JSON Schema 约束

两个 LLM 调用均使用 `json_schema` 响应格式，确保输出严格遵循预定义的结构：

- **内容分析**: `keywords: array<string>`, `context: string`, `tags: array<string>`
- **演化决策**: `should_evolve: boolean`, `actions: array<string>`, `suggested_connections: array<string>`, `tags_to_update: array<string>`, `new_context_neighborhood: array<string>`, `new_tags_neighborhood: array<array<string>>`

---

## 7. 演化系统设计

### 7.1 演化生命周期

```mermaid
stateDiagram-v2
    [*] --> Created: add_note()
    Created --> Analyzing: process_memory()
    
    Analyzing --> NoEvolution: 第一条记忆 / 无邻居
    Analyzing --> LLMDecision: 有邻居记忆
    
    LLMDecision --> NoEvolution: should_evolve=false
    LLMDecision --> Evolving: should_evolve=true
    
    Evolving --> Strengthening: action=strengthen
    Evolving --> UpdatingNeighbor: action=update_neighbor
    
    Strengthening --> Stored: 添加 links + 更新 tags
    UpdatingNeighbor --> Stored: 更新邻居 context + tags
    
    NoEvolution --> Stored
    Stored --> Consolidating: evo_cnt % threshold == 0
    Consolidating --> [*]: 重建 ChromaDB 索引
    Stored --> [*]: 正常返回
```

### 7.2 记忆整合机制

> **核心问题**: "演进"并不增加记忆数量，而是修改已有记忆的元数据（links、tags、context）。`consolidate_memories()` 的设计意图是定期将这些元数据变更同步到 ChromaDB。

#### 7.2.1 整合流程

当演化计数 `evo_cnt` 达到 `evo_threshold` 的整数倍时，系统执行 `consolidate_memories()`：

1. 创建全新的 `ChromaRetriever` 实例（清空旧索引）
2. 遍历 `self.memories` 字典中所有记忆
3. 重新将每条记忆及其最新元数据写入 ChromaDB

#### 7.2.2 为什么需要整合？——双存储不一致问题

系统采用双重存储架构（Python Dict + ChromaDB），但演进过程中两种动作对 ChromaDB 的同步行为不同：

| 演进动作 | 修改内容 | 是否即时同步 ChromaDB |
|----------|----------|----------------------|
| `strengthen` | 修改**当前新记忆**的 `links` 和 `tags` | **是**（`add_note` 后续会 `add_document`） |
| `update_neighbor` | 修改**邻居记忆**的 `context` 和 `tags` | **否**（仅修改 Python 对象） |

`update_neighbor` 只修改了 `self.memories` 字典中的 Python 对象，**完全没有同步到 ChromaDB**。因此 `consolidate_memories()` 实际上是一个**延迟同步机制**——把积累的元数据变更批量刷到 ChromaDB：

```mermaid
flowchart LR
    subgraph 多次演进后
        M["self.memories (Python Dict)<br/>tags/context 已更新 ✓"]
        C["ChromaDB<br/>tags/context 仍是旧值 ✗"]
    end

    M -->|consolidate_memories()| REBUILD["重建 ChromaDB<br/>从 memories 重新写入"]
    REBUILD --> SYNC["双存储重新一致"]
```

#### 7.2.3 已知缺陷

**缺陷 1: 窗口期数据不一致**

在两次 consolidation 之间，ChromaDB 里的邻居记忆元数据是过时的。`search_agentic()` 从 ChromaDB 读取 metadata，返回的 `tags` 和 `context` 是旧值，而非 `update_neighbor` 更新后的新值。

**缺陷 2: `update_neighbor` 索引定位错误**

`update_neighbor` 使用 `list(self.memories.values())` 的序号索引去定位邻居记忆，但这个列表的遍历顺序与 ChromaDB 搜索返回的邻居顺序**没有对应关系**，因此可能更新**错误的记忆**。这意味着 consolidation 同步到 ChromaDB 的数据本身就可能是错的。

```mermaid
flowchart TD
    SEARCH["ChromaDB 搜索返回邻居<br/>按语义相似度排序"] --> INDICES["indices = [2, 0, 4]<br/>ChromaDB 结果的序号"]
    INDICES --> NOTESLIST["noteslist = list(memories.values())<br/>按字典插入顺序"]
    NOTESLIST --> MISMATCH{"indices[i] 与 noteslist<br/>无对应关系!"}
    MISMATCH -->|可能| WRONG["更新了错误的记忆"]
    MISMATCH -->|偶然| RIGHT["碰巧更新正确"]
```

**缺陷 3: `retrieval_count` 和 `last_accessed` 未维护**

`read()` 和 `search_agentic()` 方法在检索记忆时，没有更新 `retrieval_count` 和 `last_accessed` 字段，导致这些元数据始终为初始值，consolidation 也无法修复。

---

## 8. 检索器继承体系

```mermaid
graph TD
    CR[ChromaRetriever<br/>内存模式<br/>chromadb.Client] 
    
    CR --> PCR[PersistentChromaRetriever<br/>持久化模式<br/>chromadb.PersistentClient]
    
    PCR --> CCR[CopiedChromaRetriever<br/>复制模式<br/>源集合 → 临时数据库]
    
    style CR fill:#e1f5fe
    style PCR fill:#fff3e0
    style CCR fill:#e8f5e9
```

| 检索器 | 客户端类型 | 适用场景 |
|--------|-----------|----------|
| `ChromaRetriever` | `chromadb.Client` | 单会话、测试、临时使用 |
| `PersistentChromaRetriever` | `chromadb.PersistentClient` | 跨会话持久化、多 Agent 共享 |
| `CopiedChromaRetriever` | 临时 `PersistentClient` | 从共享集合创建隔离副本 |
