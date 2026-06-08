# Supermemory 系统设计文档

## 1. 设计理念

Supermemory 的核心设计理念是 **"记忆即上下文"**——将 AI 的无状态交互转化为有状态的个性化体验。系统围绕以下原则构建：

1. **单一记忆结构**：记忆、RAG、用户画像统一在同一数据模型和本体中
2. **自动遗忘**：临时事实过期自动遗忘，矛盾自动解决，噪声不会成为永久记忆
3. **混合检索**：知识库检索（RAG）与个性化上下文（Memory）默认同时返回
4. **框架无关**：通过共享层抽象，支持任意 AI 框架的即插即用集成

---

## 2. 系统分层架构

```mermaid
graph TB
    subgraph 接入层
        WebUI[Web UI<br/>Next.js]
        BrowserExt[Browser Extension]
        MCP[MCP Server]
        SDKIntegrations[SDK Integrations<br/>Vercel/OpenAI/Mastra/VoltAgent]
    end

    subgraph 集成层
        SharedLayer[共享层<br/>memory-client / prompt-builder / cache / logger]
        ConversationsClient[Conversations Client<br/>/v4/conversations]
        ToolsShared[Tools Shared<br/>描述 / 默认值 / 去重]
    end

    subgraph API层
        DocAPI[Documents API<br/>CRUD + 文件上传]
        SearchAPI[Search API<br/>V3 块搜索 + V4 记忆搜索]
        ProfileAPI[Profile API<br/>用户画像]
        ConnectorAPI[Connectors API<br/>OAuth + 同步]
        ConversationAPI[Conversations API<br/>对话保存]
    end

    subgraph 处理层
        IngestWorkflow[IngestContentWorkflow<br/>6阶段处理流水线]
        MemoryEngine[Memory Engine<br/>提取 / 关系 / 遗忘]
        ProfileEngine[Profile Engine<br/>静态 + 动态画像]
    end

    subgraph 存储层
        PG[(PostgreSQL<br/>Drizzle ORM)]
        VectorIndex[向量索引<br/>嵌入检索]
        CFAI[Cloudflare AI<br/>Embedding 生成]
        CFKV[Cloudflare KV<br/>缓存]
    end

    WebUI --> SharedLayer
    BrowserExt --> API层
    MCP --> SharedLayer
    SDKIntegrations --> SharedLayer
    SharedLayer --> ConversationsClient
    SharedLayer --> ToolsShared

    SharedLayer --> API层
    API层 --> 处理层
    处理层 --> 存储层
    API层 --> 存储层
```

---

## 3. 核心模块设计

### 3.1 内容处理流水线（IngestContentWorkflow）

内容处理是系统的入口，所有内容都经过 6 阶段流水线：

```mermaid
flowchart LR
    Input[输入内容] --> Stage1[1. Queued<br/>类型检测 + 元数据验证]
    Stage1 --> Stage2[2. Extracting<br/>格式提取<br/>PDF/OCR/转录]
    Stage2 --> Stage3[3. Chunking<br/>语义分块<br/>上下文重叠]
    Stage3 --> Stage4[4. Embedding<br/>向量嵌入<br/>Cloudflare AI]
    Stage4 --> Stage5[5. Indexing<br/>建立记忆关系<br/>updates/extends/derives]
    Stage5 --> Stage6[6. Done<br/>可搜索]

    Stage2 -->|错误| Failed[Failed]
    Stage3 -->|错误| Failed
    Stage4 -->|错误| Failed
    Stage5 -->|错误| Failed
```

**处理元数据追踪**：

每个文档的 `processingMetadata` 记录完整的处理过程：

```mermaid
classDiagram
    class ProcessingMetadata {
        +startTime: string
        +endTime: string
        +duration: number
        +error: string
        +finalStatus: string
        +chunkingStrategy: string
        +tokenCount: number
        +steps: ProcessingStep[]
    }

    class ProcessingStep {
        +name: string
        +startTime: string
        +endTime: string
        +status: string
        +error: string
        +metadata: Record
    }

    ProcessingMetadata "1" *-- "0..*" ProcessingStep
```

**内容类型处理策略**：

| 输入类型 | 提取方式 | 分块策略 |
|---------|---------|---------|
| text | 直接使用 | 语义分块 |
| url | Web 抓取 + HTML 清理 | 语义分块 |
| pdf | PDF 解析器 | 语义分块 |
| image | OCR | 语义分块 |
| video | 音频转录 | 语义分块 |
| audio | 语音转文字 | 语义分块 |
| code | AST 感知解析 | AST 感知分块 |

---

### 3.2 记忆引擎（Memory Engine）

记忆引擎是系统的核心，负责从内容中提取事实、建立关系、解决矛盾和自动遗忘。

```mermaid
flowchart TB
    Input[对话/内容输入] --> Extract[事实提取<br/>LLM 提取结构化事实]
    Extract --> Classify[分类<br/>静态事实 vs 动态上下文]
    Classify --> Check{是否与已有记忆矛盾?}

    Check -->|是| Resolve[矛盾解决<br/>创建 updates 关系<br/>旧记忆标记 isLatest=false]
    Check -->|否| CheckExtend{是否补充已有记忆?}

    CheckExtend -->|是| Extend[扩展关系<br/>创建 extends 关系]
    CheckExtend -->|否| Derive[派生关系<br/>创建 derives 关系]

    Resolve --> VersionChain[版本链更新<br/>parentMemoryId / rootMemoryId]
    Extend --> VersionChain
    Derive --> VersionChain

    VersionChain --> Embedding[嵌入生成]
    Embedding --> Index[索引更新]

    subgraph 遗忘机制
        ForgetCheck[遗忘检查] --> Expiry{forgetAfter 到期?}
        Expiry -->|是| MarkForgotten[标记 isForgotten=true]
        Expiry -->|否| Keep[保留]
        MarkForgotten --> ForgetReason[记录 forgetReason]
    end

    Index --> ForgetCheck
```

**记忆版本链设计**：

```mermaid
sequenceDiagram
    participant User as 用户
    participant Engine as 记忆引擎
    participant DB as 数据库

    Note over User,DB: 场景：用户搬家
    User->>Engine: "我住在纽约"
    Engine->>DB: 创建 Memory v1 (isLatest=true)
    DB-->>Engine: memoryId=v1

    User->>Engine: "我搬到旧金山了"
    Engine->>DB: 查找矛盾记忆 → v1
    Engine->>DB: v1.isLatest = false
    Engine->>DB: 创建 Memory v2 (parentMemoryId=v1, rootMemoryId=v1, isLatest=true)
    DB-->>Engine: memoryId=v2

    User->>Engine: "我搬到伦敦了"
    Engine->>DB: 查找矛盾记忆 → v2
    Engine->>DB: v2.isLatest = false
    Engine->>DB: 创建 Memory v3 (parentMemoryId=v2, rootMemoryId=v1, isLatest=true)
    DB-->>Engine: memoryId=v3

    Note over User,DB: 版本链: v1 → v2 → v3
```

**记忆数据模型**：

```mermaid
erDiagram
    MemoryEntry {
        string id PK
        string memory "记忆内容"
        string spaceId FK "所属空间"
        int version "版本号"
        boolean isLatest "是否最新版本"
        string parentMemoryId FK "父记忆ID"
        string rootMemoryId FK "根记忆ID"
        json memoryRelations "关系记录"
        int sourceCount "来源计数"
        boolean isInference "是否推理生成"
        boolean isForgotten "是否已遗忘"
        boolean isStatic "是否静态记忆"
        string forgetAfter "遗忘时间"
        string forgetReason "遗忘原因"
        vector memoryEmbedding "嵌入向量"
        vector memoryEmbeddingNew "新模型嵌入"
    }

    MemoryEntry ||--o{ MemoryEntry : "parentMemoryId"
    MemoryEntry ||--o{ MemoryEntry : "rootMemoryId"
```

---

### 3.3 用户画像引擎（Profile Engine）

用户画像自动从记忆中构建，分为静态和动态两部分：

```mermaid
flowchart TB
    Memories[所有记忆条目] --> Split{isStatic?}

    Split -->|true| Static[静态画像<br/>永久事实]
    Split -->|false| Dynamic[动态上下文<br/>近期活动]

    Static --> StaticFacts[示例:<br/>- 姓名/职业<br/>- 偏好/习惯<br/>- 长期目标]
    Dynamic --> DynamicFacts[示例:<br/>- 当前项目<br/>- 近期讨论<br/>- 临时上下文]

    StaticFacts --> ProfileAPI[GET /v3/container-tags/:tag/profile]
    DynamicFacts --> ProfileAPI

    ProfileAPI --> Response[响应<br/>{ profile: { static: string[], dynamic: string[] } }]
```

**画像在 AI 集成中的注入流程**：

```mermaid
sequenceDiagram
    participant App as AI 应用
    participant MW as 中间件
    participant Shared as 共享层
    participant API as API

    App->>MW: 用户消息
    MW->>Shared: buildMemoriesText()
    Shared->>API: POST /v4/profile { containerTag, q }
    API-->>Shared: { profile: { static, dynamic }, searchResults }

    Shared->>Shared: deduplicateMemories()<br/>优先级: static > dynamic > searchResults
    Shared->>Shared: convertProfileToMarkdown()

    Note over Shared: 格式化结果:<br/>## Static Profile<br/>- 事实1<br/>- 事实2<br/><br/>## Dynamic Profile<br/>- 上下文1

    Shared->>MW: 记忆文本
    MW->>App: 注入到 system prompt
```

---

### 3.4 搜索系统设计

```mermaid
flowchart TB
    Query[查询文本] --> Rewrite{rewriteQuery?}

    Rewrite -->|是| Rewritten[查询改写<br/>+400ms 延迟]
    Rewrite -->|否| Direct[直接查询]

    Rewritten --> Embed[向量嵌入]
    Direct --> Embed

    Embed --> Search{搜索模式}

    Search -->|V3: 块搜索| ChunkSearch[块级别匹配<br/>chunkThreshold + documentThreshold]
    Search -->|V4: 记忆搜索| MemorySearch[记忆级别匹配<br/>threshold]
    Search -->|hybrid| Both[两者结合]

    ChunkSearch --> Rerank{rerank?}
    MemorySearch --> Rerank
    Both --> Rerank

    Rerank -->|是| Reranked[重排序结果]
    Rerank -->|否| Raw[原始排序]

    Reranked --> Result[返回结果]
    Raw --> Result

    subgraph V3结果
        ChunkResult[文档块结果<br/>chunks[] + documentId + score]
    end

    subgraph V4结果
        MemoryResult[记忆结果<br/>memory + similarity + context<br/>{ parents, children }]
    end
```

**V4 记忆搜索的关系上下文**：

```mermaid
graph TB
    subgraph 搜索结果
        M[匹配的记忆<br/>similarity: 0.92]
    end

    subgraph 上下文 - Parents
        P1[父记忆 v1<br/>relation: updates]
        P2[父记忆 v2<br/>relation: updates]
    end

    subgraph 上下文 - Children
        C1[子记忆<br/>relation: extends]
        C2[子记忆<br/>relation: derives]
    end

    P1 -->|updates| P2
    P2 -->|updates| M
    M -->|extends| C1
    M -->|derives| C2
```

---

### 3.5 多框架集成架构

```mermaid
graph TB
    subgraph 框架适配层
        VercelMW[Vercel AI SDK<br/>Proxy doGenerate/doStream]
        OpenAIMW[OpenAI SDK<br/>Monkey-patch<br/>chat.completions.create<br/>responses.create]
        MastraMW[Mastra<br/>inputProcessors<br/>outputProcessors]
        VoltAgentMW[VoltAgent<br/>onPrepareMessages<br/>onEnd hooks]
        ClaudeMem[Claude Memory<br/>文件系统模拟]
        AITools[AI SDK Tools<br/>函数调用工具]
    end

    subgraph 共享层
        MemoryClient[memory-client.ts<br/>buildMemoriesText()<br/>supermemoryProfileSearch()]
        PromptBuilder[prompt-builder.ts<br/>convertProfileToMarkdown()<br/>formatMemoriesForPrompt()]
        Cache[cache.ts<br/>MemoryCache LRU max=100]
        Logger[logger.ts<br/>createLogger()]
        Context[context.ts<br/>createSupermemoryClient()]
        ToolsShared[tools-shared.ts<br/>TOOL_DESCRIPTIONS<br/>deduplicateMemories()]
        ConvClient[conversations-client.ts<br/>addConversation()]
    end

    subgraph Supermemory API
        ProfileAPI[POST /v4/profile]
        SearchAPI[POST /v4/search<br/>POST /v3/search]
        ConvAPI[POST /v4/conversations]
        MemAPI[POST /v4/memories]
    end

    VercelMW --> MemoryClient
    OpenAIMW --> MemoryClient
    MastraMW --> MemoryClient
    VoltAgentMW --> MemoryClient
    ClaudeMem --> MemoryClient
    AITools --> ToolsShared

    MemoryClient --> ProfileAPI
    MemoryClient --> SearchAPI
    ToolsShared --> SearchAPI

    VercelMW --> ConvClient
    OpenAIMW --> ConvClient
    MastraMW --> ConvClient
    VoltAgentMW --> ConvClient

    ConvClient --> ConvAPI

    MemoryClient --> Cache
    MemoryClient --> PromptBuilder
    MemoryClient --> Context
    VercelMW --> Logger
```

**三种记忆注入模式**：

| 模式 | 数据来源 | 适用场景 |
|------|---------|---------|
| `profile` | 仅用户画像 | 快速上下文注入 |
| `query` | 仅搜索结果 | 精确知识检索 |
| `full` | 画像 + 搜索结果 | 完整上下文（默认） |

**对话保存策略**：

| 模式 | 行为 |
|------|------|
| `always` | 每次 LLM 响应后保存对话 |
| `never` | 不保存对话 |

---

### 3.6 MCP 服务器设计

```mermaid
flowchart TB
    Client[MCP 客户端<br/>Claude/Cursor/VS Code] -->|SSE/Streamable HTTP| MCP[MCP Server<br/>Cloudflare Durable Objects]

    MCP --> Auth{认证方式}
    Auth -->|sm_ 前缀| APIKey[API Key 认证<br/>GET /v3/session]
    Auth -->|OAuth Token| OAuth[OAuth 认证<br/>GET /v3/mcp/session-with-key]

    APIKey --> UserAuth[AuthUser<br/>userId + apiKey]
    OAuth --> UserAuth

    UserAuth --> SMClient[SupermemoryClient]

    subgraph 工具处理
        SMClient --> Memory[memory 工具<br/>save: client.add()<br/>forget: 精确匹配 → 语义搜索]
        SMClient --> Recall[recall 工具<br/>client.search.memories()<br/>searchMode: hybrid]
        SMClient --> Profile[recall + includeProfile<br/>client.profile()]
        SMClient --> Projects[listProjects<br/>GET /v3/projects]
    end

    subgraph 资源处理
        SMClient --> ProfileRes[supermemory://profile<br/>用户画像]
        SMClient --> ProjectsRes[supermemory://projects<br/>项目列表]
    end

    subgraph App UI
        SMClient --> GraphTool[memory-graph<br/>记忆图谱可视化]
        SMClient --> FetchGraph[fetch-graph-data<br/>图谱分页数据]
    end
```

**遗忘双重策略时序**：

```mermaid
sequenceDiagram
    participant Client as MCP 客户端
    participant Server as MCP Server
    participant API as Supermemory API

    Client->>Server: memory(action: "forget", content: "...")
    Server->>API: client.memories.forget({ content })
    API-->>Server: 404 (未找到精确匹配)

    Note over Server: 回退到语义搜索
    Server->>API: client.search.memories({ q: content })
    API-->>Server: 搜索结果

    Server->>Server: 过滤 similarity > 0.85
    Server->>API: client.memories.forget({ id: matchedId })
    API-->>Server: 遗忘成功

    Server-->>Client: 遗忘确认
```

---

### 3.7 记忆图谱可视化设计

```mermaid
flowchart TB
    API[API 数据<br/>documents + memories] --> Hook[useGraphData Hook]

    Hook --> Cluster[集群计算<br/>BFS 连通分量]
    Hook --> Layout[布局计算<br/>文档: 黄金角螺旋<br/>记忆: 多环轨道]
    Hook --> Edges[边计算<br/>derives + updates + extends]

    Cluster --> Nodes[GraphNode[]]
    Layout --> Nodes
    Edges --> GraphEdges[GraphEdge[]]

    Nodes --> Sim[ForceSimulation<br/>d3-force 物理仿真]
    GraphEdges --> Sim

    Sim --> Canvas[GraphCanvas<br/>Canvas 2D 渲染]

    Canvas --> VP[ViewportState<br/>平移/缩放/坐标转换]
    Canvas --> SI[SpatialIndex<br/>网格命中测试]
    Canvas --> IH[InputHandler<br/>鼠标/触摸事件]
    Canvas --> VCI[VersionChainIndex<br/>版本链查询]

    VP --> Render[renderFrame<br/>LOD + 批量渲染]
    SI --> IH
    IH --> Render

    subgraph 渲染优化
        LOD[LOD 机制<br/>缩放 < 0.5: 采样关系边<br/>缩放 < 0.38: 采样派生边]
        Dense[稠密图模式<br/>节点 > 25000 且 缩放 < 0.42<br/>简单圆点渲染]
        Batch[批量渲染<br/>按样式分组<br/>减少 draw call]
    end

    Render --> LOD
    Render --> Dense
    Render --> Batch
```

**节点布局算法**：

```mermaid
flowchart LR
    subgraph 文档节点布局
        DocInput[N 个文档] --> Golden[黄金角螺旋<br/>angle = idx * 137.5°<br/>radius = sqrt(idx+1/count)]
        Golden --> Append[新文档追加<br/>空间网格碰撞检测<br/>最多 8 环 x 18 候选位]
    end

    subgraph 记忆节点布局
        MemInput[文档的记忆] --> Orbit[多环轨道<br/>内环容量 = 周长/84px<br/>半径 = 260 + ring*110]
        Orbit --> Angle[黄金角 + 哈希偏移<br/>均匀分布]
    end
```

---

### 3.8 Web 应用架构

```mermaid
flowchart TB
    subgraph Provider 链
        Theme[ThemeProvider<br/>forcedTheme=dark] --> Autumn[AutumnProvider<br/>计费/订阅]
        Autumn --> Query[QueryProvider<br/>TanStack React Query]
        Query --> Auth[AuthProvider<br/>Better Auth]
        Auth --> PostHog[PostHogProvider<br/>产品分析]
        PostHog --> ErrorTrack[ErrorTrackingProvider<br/>Sentry]
        ErrorTrack --> Nuqs[NuqsAdapter<br/>URL 状态]
    end

    subgraph 主页面路由
        Page[page.tsx<br/>视图路由器] --> ViewMode{?view=}
        ViewMode -->|dashboard| Dashboard[DashboardView<br/>每日亮点 + 记忆]
        ViewMode -->|chat| Chat[ChatSidebar<br/>Nova AI 对话]
        ViewMode -->|graph| Graph[GraphLayoutView<br/>知识图谱]
        ViewMode -->|list| List[MemoriesGrid<br/>记忆网格]
        ViewMode -->|integrations| Integrations[IntegrationsView]
    end

    subgraph URL 驱动的模态
        URLParams[nuqs 参数] --> Add[?add=note|link|file|connect]
        URLParams --> Search[?search=true]
        URLParams --> Doc[?doc=id]
        URLParams --> Thread[?thread=id]
    end

    subgraph 聊天系统
        ChatStore[Zustand + IndexedDB<br/>持久化聊天存储] --> ChatTransport[DefaultChatTransport<br/>POST /chat]
        ChatTransport --> Models[模型系统<br/>GPT-5.1 / Claude / Gemini]
        ChatTransport --> Reasoning[推理模式<br/>instant / thinking]
        ChatTransport --> Queue[消息队列<br/>最多 3 条排队]
    end
```

---

## 4. 数据流设计

### 4.1 端到端记忆流程

```mermaid
sequenceDiagram
    participant User as 用户
    participant App as AI 应用
    participant MW as 中间件
    participant API as Supermemory API
    participant Engine as 记忆引擎
    participant DB as 数据库

    Note over User,DB: 1. 记忆存储
    User->>App: 分享信息
    App->>MW: 用户消息
    MW->>API: POST /v4/conversations
    API->>Engine: 提取事实
    Engine->>Engine: 分类 (static/dynamic)
    Engine->>Engine: 矛盾检测
    Engine->>DB: 存储记忆 + 建立关系
    Engine->>DB: 生成嵌入向量
    MW->>App: LLM 响应

    Note over User,DB: 2. 记忆检索
    User->>App: 新对话
    App->>MW: 用户消息
    MW->>API: POST /v4/profile { containerTag, q }
    API->>DB: 查询画像 + 搜索记忆
    DB-->>API: profile + searchResults
    API-->>MW: 记忆数据
    MW->>MW: 去重 + 格式化
    MW->>App: 增强的 system prompt
    App->>User: 个性化响应

    Note over User,DB: 3. 自动遗忘
    Engine->>DB: 检查 forgetAfter
    DB-->>Engine: 过期记忆
    Engine->>DB: 标记 isForgotten=true
```

### 4.2 连接器同步流程

```mermaid
sequenceDiagram
    participant User as 用户
    participant Web as Web App
    participant API as API Backend
    participant OAuth as OAuth 提供商
    participant Workflow as CF Workflow
    participant Engine as 处理引擎

    User->>Web: 添加 Google Drive 连接
    Web->>API: POST /connections/google-drive
    API->>OAuth: 生成 OAuth 授权链接
    OAuth-->>API: authLink
    API-->>Web: authLink + expiresIn
    Web->>OAuth: 用户授权
    OAuth-->>API: 授权码回调
    API->>OAuth: 交换 Token
    OAuth-->>API: accessToken + refreshToken

    User->>Web: 触发同步
    Web->>API: POST /connections/google-drive/import
    API->>Workflow: 启动 IngestContentWorkflow
    Workflow->>OAuth: 获取文件列表
    OAuth-->>Workflow: 文件列表
    Workflow->>Engine: 逐文件处理
    Engine->>Engine: 提取 → 分块 → 嵌入 → 索引
    Engine-->>API: 处理状态更新
    API-->>Web: 同步进度
```

---

## 5. 缓存策略

```mermaid
flowchart TB
    subgraph 浏览器端缓存
        CacheAPI[Cache API<br/>space-highlights: 4h TTL]
        LocalStorage[localStorage<br/>memory-of-day: 24h TTL<br/>last-org-slug: 永久]
    end

    subgraph SDK 缓存
        LRUCache[MemoryCache<br/>LRU max=100<br/>key: containerTag:threadId:mode:msg]
        ContainerTagCache[ContainerTag 缓存<br/>MCP Server: 5min TTL]
    end

    subgraph 服务端缓存
        CFKV[Cloudflare KV<br/>API 响应缓存]
        ContentHash[内容哈希<br/>防重复处理]
    end

    LRUCache -->|命中| SkipSearch[跳过 API 调用]
    LRUCache -->|未命中| APICall[调用 /v4/profile]
```

---

## 6. 嵌入模型迁移设计

系统支持无缝的嵌入模型切换，所有嵌入相关表都有双字段设计：

```mermaid
flowchart LR
    subgraph 当前模型
        Embedding[xxxEmbedding<br/>当前使用的嵌入]
        Model[xxxEmbeddingModel<br/>当前模型名称]
    end

    subgraph 新模型
        NewEmbedding[xxxEmbeddingNew<br/>新模型嵌入]
        NewModel[xxxEmbeddingNewModel<br/>新模型名称]
    end

    subgraph 迁移流程
        Step1[1. 新内容写入双字段]
        Step2[2. 后台任务重新嵌入旧数据]
        Step3[3. 验证新嵌入质量]
        Step4[4. 切换读取到新字段]
        Step5[5. 清理旧字段]
    end

    Embedding --> Step1
    Model --> Step1
    NewEmbedding --> Step1
    NewModel --> Step1
    Step1 --> Step2 --> Step3 --> Step4 --> Step5
```

---

## 7. 错误处理策略

```mermaid
flowchart TB
    Error[错误发生] --> Type{错误类型}

    Type -->|API 调用失败| Retry[3 次线性重试]
    Type -->|认证失败| Auth[401 → 重新认证<br/>402 → 升级<br/>403 → 受限]
    Type -->|限流| RateLimit[429 → 等待重试]
    Type -->|记忆检索失败| Skip{skipMemoryOnError?}
    Type -->|内容处理失败| MarkFailed[标记文档 status=failed<br/>记录 processingMetadata.error]

    Skip -->|true| Continue[继续 LLM 调用]
    Skip -->|false| Propagate[向上传播错误]

    Retry -->|成功| Return[返回结果]
    Retry -->|3次失败| Sentry[上报 Sentry]
    Auth --> Sentry
    RateLimit --> Sentry
```

---

## 8. 关键设计决策

| 决策 | 原因 | 影响 |
|------|------|------|
| 记忆与 RAG 统一数据模型 | 避免两套独立系统的复杂性 | 混合搜索自然支持 |
| 版本链而非覆盖更新 | 保留知识演进历史 | 支持时间线回溯和关系推理 |
| 双嵌入字段设计 | 支持无缝模型迁移 | 存储开销增加但迁移零停机 |
| Cloudflare Workers 部署 | 全球低延迟 + 自动扩展 | 冷启动需优化 |
| Canvas 2D 而非 WebGL | 更好的兼容性和调试体验 | 超大图性能受限 |
| LOD + 批量渲染 | 大规模图谱性能优化 | 缩小时细节丢失 |
| LRU 缓存 + 轮次键 | 避免工具调用循环中重复请求 | 缓存一致性需注意 |
| 猴子补丁 OpenAI 客户端 | 最小侵入性集成 | OpenAI SDK 更新可能破坏兼容 |
