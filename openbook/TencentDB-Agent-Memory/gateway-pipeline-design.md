# Gateway + Pipeline 模块设计文档

## 1. 模块概述

Gateway 模块提供 HTTP 服务，将 TdaiCore 的记忆能力暴露为 REST API，使外部客户端（如 Hermes Sidecar）能够通过标准 HTTP 协议进行记忆召回、对话捕获、记忆搜索等操作。Pipeline 模块管理 L0→L1→L2→L3 四层记忆提取的调度，负责阈值触发、空闲超时、预热模式、向下只进定时器等核心调度逻辑。

**核心设计原则：**

- **主机无关**：TdaiCore 通过 HostAdapter 接口与宿主环境解耦，Gateway（Standalone）和 OpenClaw 各自提供适配器实现
- **串行保证**：同层任务通过 SerialQueue 保证不并发，避免竞态条件
- **优雅降级**：VectorStore 不可用时自动回退到 JSONL 文件读取
- **检查点恢复**：所有管道状态持久化到检查点文件，重启后自动恢复未完成的工作

---

## 2. Gateway 架构设计

```mermaid
graph TB
    subgraph Client["外部客户端"]
        C1[Hermes Sidecar]
        C2[CLI 工具]
        C3[自定义集成]
    end

    subgraph Gateway["TdaiGateway HTTP 服务器"]
        HTTP["Node.js http 模块"]
        ROUTER["请求路由器"]
        AUTH["认证中间件<br/>Bearer Token"]
        CORS["CORS 中间件<br/>白名单控制"]

        subgraph Endpoints["REST 端点"]
            E1["GET /health"]
            E2["POST /recall"]
            E3["POST /capture"]
            E4["POST /search/memories"]
            E5["POST /search/conversations"]
            E6["POST /session/end"]
            E7["POST /seed"]
        end
    end

    subgraph Config["配置层"]
        CFG["loadGatewayConfig()"]
        YAML["tdai-gateway.yaml"]
        ENV["环境变量"]
        OVERRIDE["传入覆盖"]
    end

    subgraph Core["核心层"]
        ADAPTER["StandaloneHostAdapter"]
        TDAI["TdaiCore"]
        SF["SessionFilter"]
    end

    Client -->|HTTP 请求| HTTP
    HTTP --> ROUTER
    ROUTER --> CORS
    CORS --> AUTH
    AUTH --> Endpoints
    Endpoints --> TDAI
    CFG -->|配置注入| Gateway
    YAML --> CFG
    ENV --> CFG
    OVERRIDE --> CFG
    ADAPTER -->|桥接| TDAI
    SF --> TDAI
```

**关键组件说明：**

| 组件 | 文件 | 职责 |
|------|------|------|
| TdaiGateway | `src/gateway/server.ts` | HTTP 服务器，路由分发，请求/响应序列化 |
| loadGatewayConfig | `src/gateway/config.ts` | 多源配置合并（文件→环境变量→覆盖） |
| API 类型 | `src/gateway/types.ts` | 请求/响应 TypeScript 接口定义 |
| StandaloneHostAdapter | `src/adapters/standalone/host-adapter.ts` | 无 OpenClaw 依赖的 HostAdapter 实现 |

---

## 3. Gateway 请求处理流程

以 `/recall` 端点为例，展示完整的请求处理时序：

```mermaid
sequenceDiagram
    participant C as 客户端
    participant H as HTTP Server
    participant R as 路由器
    participant CORS as CORS 中间件
    participant A as 认证中间件
    participant H as /recall 处理器
    participant TC as TdaiCore
    participant VS as VectorStore
    participant ES as EmbeddingService

    C->>H: POST /recall<br/>Authorization: Bearer xxx<br/>{"query": "...", "session_key": "..."}

    H->>R: handleRequest(req, res)
    R->>CORS: applyCorsHeaders(req, res)
    CORS-->>R: Origin 匹配 / 拒绝

    alt OPTIONS 请求
        R-->>C: 204 No Content
    end

    R->>A: checkAuth(req, res)
    alt 无 apiKey 配置
        A-->>R: true（跳过认证）
    else apiKey 已配置
        alt Bearer Token 有效
            A-->>R: true
        else Token 缺失/无效
            A-->>C: 401 Unauthorized
        end
    end

    R->>H: handleRecall(req, res)
    H->>H: parseJsonBody&lt;RecallRequest&gt;(req)

    alt 缺少必填字段
        H-->>C: 400 Bad Request
    end

    H->>TC: handleBeforeRecall(query, sessionKey)
    TC->>ES: 查询嵌入向量
    ES-->>TC: 嵌入结果
    TC->>VS: 向量搜索 L1 记忆
    VS-->>TC: 匹配的记忆列表
    TC-->>H: RecallResult

    H->>H: 构建 RecallResponse
    H-->>C: 200 OK<br/>{"context": "...", "strategy": "...", "memory_count": 5}
```

**端点一览：**

| 端点 | 方法 | 认证 | 功能 | 核心调用 |
|------|------|------|------|----------|
| `/health` | GET | 否 | 健康检查 | `core.getVectorStore()` |
| `/recall` | POST | 是 | 记忆召回 | `core.handleBeforeRecall()` |
| `/capture` | POST | 是 | 对话捕获 | `core.handleTurnCommitted()` |
| `/search/memories` | POST | 是 | L1 记忆搜索 | `core.searchMemories()` |
| `/search/conversations` | POST | 是 | L0 对话搜索 | `core.searchConversations()` |
| `/session/end` | POST | 是 | 会话结束 | `core.handleSessionEnd()` |
| `/seed` | POST | 是 | 批量种子导入 | `executeSeed()` |

---

## 4. Gateway 认证与安全设计

```mermaid
flowchart TD
    REQ["收到请求"] --> CORS_CHECK{"CORS 检查"}

    CORS_CHECK -->|"Origin 在白名单中"| CORS_PASS["设置 Access-Control-Allow-Origin<br/>Access-Control-Allow-Methods<br/>Access-Control-Allow-Headers"]
    CORS_CHECK -->|"Origin 不在白名单"| CORS_DENY["不设置 CORS 头<br/>浏览器将阻止跨域请求"]
    CORS_CHECK -->|"白名单为空"| CORS_NONE["不设置任何 CORS 头<br/>最严格默认"]
    CORS_CHECK -->|"白名单含 *"| CORS_WILD["设置 Access-Control-Allow-Origin: *"]

    CORS_PASS --> METHOD_CHECK
    CORS_DENY --> METHOD_CHECK
    CORS_NONE --> METHOD_CHECK
    CORS_WILD --> METHOD_CHECK

    METHOD_CHECK{"OPTIONS 请求?"} -->|是| OPTIONS_RESP["204 No Content"]
    METHOD_CHECK -->|否| HEALTH_CHECK

    HEALTH_CHECK{"GET /health?"} -->|是| HEALTH_RESP["200 OK（无需认证）"]
    HEALTH_CHECK -->|否| AUTH_CHECK

    AUTH_CHECK{"apiKey 已配置?"} -->|否| SKIP_AUTH["跳过认证<br/>（兼容旧版行为）"]
    AUTH_CHECK -->|是| BEARER_CHECK

    BEARER_CHECK{"Authorization 头存在<br/>且以 Bearer 开头?"} -->|否| AUTH_FAIL["401 Unauthorized<br/>missing Bearer token"]
    BEARER_CHECK -->|是| TOKEN_COMPARE

    TOKEN_COMPARE{"timingSafeEqual<br/>常量时间比较"} -->|"匹配"| AUTH_PASS["认证通过"]
    TOKEN_COMPARE -->|"不匹配"| AUTH_FAIL2["401 Unauthorized<br/>invalid token"]

    SKIP_AUTH --> ROUTE_HANDLER["路由处理器"]
    AUTH_PASS --> ROUTE_HANDLER

    style AUTH_FAIL fill:#f66,color:#fff
    style AUTH_FAIL2 fill:#f66,color:#fff
    style HEALTH_RESP fill:#6f6,color:#fff
    style AUTH_PASS fill:#6f6,color:#fff
```

**安全设计要点：**

1. **常量时间比较**：使用 `crypto.timingSafeEqual` 防止时序攻击，攻击者无法通过响应时间推断 API Key 前缀
2. **/health 免认证**：健康检查端点始终可访问，支持 k8s liveness probe 和 Docker health-check
3. **启动安全态势报告**：`logSecurityPosture()` 在启动时输出认证状态、绑定地址、CORS 配置，特别警告非回环地址绑定且未设置 apiKey 的情况
4. **CORS 白名单**：默认空列表（最严格），支持精确域名匹配和 `*` 通配符（仅限开发环境）
5. **配置优先级**：YAML 中的 `corsOrigins` 优先于环境变量 `TDAI_CORS_ORIGINS`

---

## 5. Pipeline 调度架构

```mermaid
graph TB
    subgraph Input["输入层"]
        CAP["auto-capture<br/>agent_end 事件"]
        SE["session/end<br/>手动结束"]
    end

    subgraph PipelineManager["MemoryPipelineManager"]
        NOTIFY["notifyConversation()"]

        subgraph L1_Layer["L0→L1 层"]
            L1_BUF["消息缓冲区<br/>messageBuffers"]
            L1_THRESH["阈值触发<br/>everyNConversations"]
            L1_IDLE["空闲超时<br/>l1IdleTimeoutSeconds"]
            L1_WARMUP["预热模式<br/>1→2→4→8→...→N"]
            L1_QUEUE["SerialQueue<br/>'L1'"]
            L1_RUNNER["L1Runner<br/>extractL1Memories"]
        end

        subgraph L2_Layer["L1→L2 层"]
            L2_DELAY["延迟触发<br/>delayAfterL1Seconds"]
            L2_MIN["最小间隔<br/>minIntervalSeconds"]
            L2_MAX["最大间隔<br/>maxIntervalSeconds"]
            L2_ACTIVE["活跃窗口<br/>sessionActiveWindowHours"]
            L2_QUEUE["SerialQueue<br/>'L2'"]
            L2_RUNNER["L2Runner<br/>SceneExtractor.extract"]
        end

        subgraph L3_Layer["L2→L3 层"]
            L3_TRIGGER["PersonaTrigger<br/>shouldGenerate"]
            L3_DEDUP["全局去重<br/>l3Pending + l3Running"]
            L3_QUEUE["SerialQueue<br/>'L3'"]
            L3_RUNNER["L3Runner<br/>PersonaGenerator"]
        end

        PERSISTER["检查点持久化<br/>CheckpointManager"]
    end

    CAP --> NOTIFY
    SE --> NOTIFY
    NOTIFY --> L1_BUF
    L1_BUF --> L1_THRESH
    L1_BUF --> L1_IDLE
    L1_THRESH --> L1_QUEUE
    L1_IDLE --> L1_QUEUE
    L1_WARMUP --> L1_THRESH
    L1_QUEUE --> L1_RUNNER
    L1_RUNNER -->|"成功"| L2_DELAY
    L1_RUNNER -->|"成功"| PERSISTER

    L2_DELAY --> L2_QUEUE
    L2_MAX --> L2_QUEUE
    L2_QUEUE --> L2_RUNNER
    L2_RUNNER -->|"成功"| L3_TRIGGER
    L2_RUNNER -->|"成功"| PERSISTER

    L3_TRIGGER -->|"should=true"| L3_DEDUP
    L3_DEDUP --> L3_QUEUE
    L3_QUEUE --> L3_RUNNER
    L3_RUNNER --> PERSISTER
```

**管道工厂核心方法：**

| 方法 | 功能 |
|------|------|
| `initDataDirectories()` | 确保 conversations/records/scene_blocks/.metadata/.backup 子目录存在 |
| `initStores()` | 初始化 VectorStore + EmbeddingService（按 dataDir 缓存单例） |
| `createL1Runner()` | 读取 L0 → 按 sessionId 分组 → extractL1Memories → 更新检查点 |
| `createL2Runner()` | 读取 L1 → SceneExtractor.extract → 同步 Profile → 更新检查点 |
| `createL3Runner()` | PersonaTrigger.shouldGenerate → PersonaGenerator.generateLocalPersona → 同步 Profile → 更新检查点 |
| `createPipelineManager()` | 创建 MemoryPipelineManager 调度器 |

---

## 6. L0→L1 调度流程

```mermaid
sequenceDiagram
    participant AE as agent_end 事件
    participant PM as PipelineManager
    participant BUF as 消息缓冲区
    participant STATE as 会话状态
    participant TIMER as L1 空闲定时器
    participant Q as L1 SerialQueue
    participant R as L1Runner
    participant CP as CheckpointManager

    Note over PM: 两条触发路径

    rect rgb(230, 245, 255)
        Note over PM: 路径 A：阈值触发
        AE->>PM: notifyConversation(sessionKey, messages)
        PM->>STATE: conversation_count += 1
        PM->>BUF: buffer.push(...messages)

        alt conversation_count >= effectiveThreshold
            PM->>TIMER: cancel()（阈值已触发，无需空闲定时器）
            PM->>Q: enqueueL1(sessionKey, "threshold")
        else conversation_count < effectiveThreshold
            PM->>TIMER: schedule(l1IdleTimeoutMs, onL1IdleTimeout)
            Note over TIMER: 重置倒计时
        end
    end

    rect rgb(255, 245, 230)
        Note over PM: 路径 B：空闲超时触发
        TIMER-->>PM: onL1IdleTimeout(sessionKey)
        PM->>BUF: 检查缓冲区
        alt 缓冲区非空
            PM->>Q: enqueueL1(sessionKey, "idle_timeout")
        else 缓冲区为空
            Note over PM: 跳过，无待处理消息
        end
    end

    rect rgb(230, 255, 230)
        Note over Q: L1 执行
        Q->>R: runL1(sessionKey)
        R->>BUF: drain 缓冲区消息
        R->>R: extractL1Memories(messages)
        R->>CP: markL1ExtractionComplete()

        alt L1 成功
            R->>STATE: conversation_count = 0
            R->>STATE: advanceWarmupThreshold()
            R->>CP: persistStates()
            R->>PM: advanceL2Timer(sessionKey)
        else L1 失败
            R->>BUF: 恢复消息到缓冲区
            R->>TIMER: schedule(L1_RETRY_DELAY_MS)
            Note over R: 最多重试 5 次
        end
    end
```

**L1 调度关键参数：**

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `everyNConversations` | 5 | 触发 L1 批处理的对话轮次阈值 |
| `l1IdleTimeoutSeconds` | 60 | 空闲超时（秒），用户停止对话后触发 L1 |
| `L1_RETRY_DELAY_MS` | 30,000 | L1 失败后重试延迟 |
| `L1_MAX_RETRIES` | 5 | 最大连续重试次数 |

---

## 7. L1→L2→L3 调度流程

```mermaid
sequenceDiagram
    participant L1 as L1 完成
    participant PM as PipelineManager
    participant L2T as L2 定时器
    participant L2Q as L2 SerialQueue
    participant L2R as L2Runner
    participant L3T as L3 触发器
    participant L3Q as L3 SerialQueue
    participant L3R as L3Runner
    participant CP as CheckpointManager

    Note over PM: L1→L2：向下只进定时器

    L1->>PM: advanceL2Timer(sessionKey)
    PM->>PM: T_desired = max(now + delay, lastL2 + minInterval)
    PM->>L2T: tryAdvanceTo(T_desired)

    alt T_desired < 当前计划时间
        L2T-->>PM: 定时器提前（向下只进）
    else T_desired >= 当前计划时间
        L2T-->>PM: 不调整（保持更早的计划时间）
    end

    L2T-->>PM: onL2TimerFired(sessionKey, "delay-after-l1")
    PM->>L2Q: enqueueL2(sessionKey, "timer:delay-after-l1")

    L2Q->>L2R: runL2(sessionKey)
    L2R->>L2R: SceneExtractor.extract(memories)
    L2R->>L2R: syncLocalProfilesToStore()
    L2R->>CP: incrementScenesProcessed()

    alt L2 成功
        L2R->>PM: armL2MaxInterval(sessionKey)
        Note over PM: 设置下次最大间隔定时器
        L2R->>L3T: triggerL3()
    else L2 失败
        L2R->>PM: armL2MaxInterval(sessionKey)
        Note over PM: 仍然设置重试定时器
    end

    Note over PM: L2→L3：全局互斥 + 去重

    L3T->>PM: triggerL3()
    alt l3Running = false
        PM->>L3Q: enqueueL3()
    else l3Running = true
        PM->>PM: l3Pending = true
        Note over PM: L3 完成后会检查此标志
    end

    L3Q->>L3R: runL3()
    L3R->>L3R: PersonaTrigger.shouldGenerate()
    alt should = false
        L3R-->>L3Q: 跳过
    else should = true
        L3R->>L3R: PersonaGenerator.generateLocalPersona()
        L3R->>L3R: syncLocalProfilesToStore()
        L3R->>CP: markPersonaGenerated()
    end

    L3Q-->>PM: l3Running = false
    alt l3Pending = true
        PM->>L3Q: enqueueL3()（再次执行）
    end

    Note over PM: 最大间隔保证

    L2T-->>PM: onL2TimerFired(sessionKey, "max-interval")
    PM->>PM: 检查会话活跃状态
    alt 会话活跃（inactive < activeWindow）
        PM->>L2Q: enqueueL2(sessionKey, "timer:max-interval")
    else 会话已冷却
        Note over PM: 取消定时器，等待下次 L1 事件重新激活
    end
```

**L2/L3 调度关键参数：**

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `delayAfterL1Seconds` | 90 | L1 完成后延迟触发 L2（秒） |
| `minIntervalSeconds` | 900 | L2 最小间隔（秒），防止过于频繁 |
| `maxIntervalSeconds` | 3600 | L2 最大间隔（秒），保证活跃会话定期提取 |
| `sessionActiveWindowHours` | 24 | 会话活跃窗口（小时），超过则停止 L2 轮询 |

---

## 8. 预热模式设计

```mermaid
stateDiagram-v2
    [*] --> Warmup_1: 新会话创建<br/>enableWarmup=true

    Warmup_1 --> Warmup_2: L1 成功<br/>threshold *= 2
    note right of Warmup_1
        warmup_threshold = 1
        第1轮对话即触发 L1
    end note

    Warmup_2 --> Warmup_4: L1 成功<br/>threshold *= 2
    note right of Warmup_2
        warmup_threshold = 2
        每2轮对话触发 L1
    end note

    Warmup_4 --> Warmup_8: L1 成功<br/>threshold *= 2
    note right of Warmup_4
        warmup_threshold = 4
        每4轮对话触发 L1
    end note

    Warmup_8 --> SteadyState: next >= everyNConversations
    note right of Warmup_8
        warmup_threshold = 8
        每8轮对话触发 L1
    end note

    SteadyState --> SteadyState: L1 成功<br/>使用固定阈值
    note right of SteadyState
        warmup_threshold = 0
        使用 everyNConversations
    end note

    [*] --> SteadyState: enableWarmup=false<br/>直接使用固定阈值

    note left of Warmup_1
        预热序列：1 → 2 → 4 → 8 → ... → N
        确保早期对话被快速处理
        同时逐步降低处理频率
    end note
```

**预热模式设计意图：**

- **早期快速处理**：新会话的第 1 轮对话立即触发 L1 提取，用户能尽快看到记忆效果
- **逐步收敛**：阈值指数增长（1→2→4→8→...），逐步降低处理频率，减少资源消耗
- **平滑过渡**：当 `warmup_threshold * 2 >= everyNConversations` 时，标记预热完成（`warmup_threshold = 0`），切换到稳态阈值
- **向后兼容**：旧检查点文件缺少 `warmup_threshold` 字段时，默认为 0（视为已完成预热）

---

## 9. 适配器模式设计

```mermaid
classDiagram
    class HostAdapter {
        <<interface>>
        +hostType: string
        +getRuntimeContext() RuntimeContext
        +getLogger() Logger
        +getLLMRunnerFactory() LLMRunnerFactory
    }

    class LLMRunnerFactory {
        <<interface>>
        +createRunner(opts?) LLMRunner
    }

    class LLMRunner {
        <<interface>>
        +run(params) Promise~string~
    }

    class LLMRunnerCreateOptions {
        +modelRef?: string
        +enableTools?: boolean
    }

    class OpenClawHostAdapter {
        -api: OpenClawPluginApi
        -pluginDataDir: string
        -openclawConfig: unknown
        -runnerFactory: OpenClawLLMRunnerFactory
        +hostType: "openclaw"
        +getRuntimeContext() RuntimeContext
        +buildRuntimeContextForSession(sessionKey, sessionId?) RuntimeContext
        +getLogger() Logger
        +getLLMRunnerFactory() LLMRunnerFactory
        +getPluginApi() OpenClawPluginApi
        +getOpenClawConfig() unknown
        +getPluginDataDir() string
    }

    class StandaloneHostAdapter {
        -dataDir: string
        -logger: Logger
        -runnerFactory: StandaloneLLMRunnerFactory
        -defaultUserId: string
        -platform: string
        +hostType: "standalone"
        +getRuntimeContext() RuntimeContext
        +buildRuntimeContextForRequest(params) RuntimeContext
        +getLogger() Logger
        +getLLMRunnerFactory() LLMRunnerFactory
    }

    class OpenClawLLMRunnerFactory {
        -config: unknown
        -agentRuntime?: EmbeddedAgentRuntimeLike
        -logger?: Logger
        +createRunner(opts?) LLMRunner
    }

    class StandaloneLLMRunnerFactory {
        -config: StandaloneLLMConfig
        -logger?: Logger
        +createRunner(opts?) LLMRunner
    }

    class OpenClawLLMRunner {
        -runner: CleanContextRunner
        +run(params) Promise~string~
    }

    class StandaloneLLMRunner {
        -config: StandaloneLLMConfig
        -model: string
        -enableTools: boolean
        -logger?: Logger
        +run(params) Promise~string~
    }

    class StandaloneLLMConfig {
        +baseUrl: string
        +apiKey: string
        +model: string
        +maxTokens?: number
        +timeoutMs?: number
    }

    HostAdapter <|.. OpenClawHostAdapter : 实现
    HostAdapter <|.. StandaloneHostAdapter : 实现
    LLMRunnerFactory <|.. OpenClawLLMRunnerFactory : 实现
    LLMRunnerFactory <|.. StandaloneLLMRunnerFactory : 实现
    LLMRunner <|.. OpenClawLLMRunner : 实现
    LLMRunner <|.. StandaloneLLMRunner : 实现

    OpenClawHostAdapter *-- OpenClawLLMRunnerFactory : 创建
    StandaloneHostAdapter *-- StandaloneLLMRunnerFactory : 创建
    OpenClawLLMRunnerFactory ..> OpenClawLLMRunner : 创建
    StandaloneLLMRunnerFactory ..> StandaloneLLMRunner : 创建
    OpenClawLLMRunner *-- CleanContextRunner : 包装
    StandaloneLLMRunner ..> StandaloneLLMConfig : 使用

    LLMRunnerFactory ..> LLMRunnerCreateOptions : 参数
    LLMRunner ..> LLMRunParams : 参数
```

**两种适配器对比：**

| 特性 | OpenClawHostAdapter | StandaloneHostAdapter |
|------|---------------------|----------------------|
| 宿主环境 | OpenClaw 插件系统 | Gateway / Hermes Sidecar |
| LLM 调用 | CleanContextRunner → runEmbeddedPiAgent | Vercel AI SDK → OpenAI 兼容 API |
| 工具支持 | OpenClaw 内置工具循环 | 沙盒文件工具（read/write/replace） |
| 上下文来源 | OpenClaw 事件 ctx | HTTP 请求参数 |
| 日志来源 | api.logger | console logger |
| 配置来源 | api.pluginConfig | tdai-gateway.yaml + 环境变量 |

---

## 10. 插件注册流程

```mermaid
sequenceDiagram
    participant OC as OpenClaw
    participant REG as register()
    participant CFG as parseConfig()
    participant ADP as OpenClawHostAdapter
    participant TC as TdaiCore
    participant PF as PipelineFactory
    participant PM as PipelineManager

    OC->>REG: register(api)

    alt CLI 元数据模式
        REG->>OC: registerCli(memory-tdai)
        Note over REG: 仅注册 CLI 命令，跳过运行时初始化
    else 完整模式
        REG->>REG: pluginStartTimestamp = Date.now()
        REG->>REG: setPreferredEmbeddedAgentRuntime()
        REG->>CFG: parseConfig(api.pluginConfig)
        CFG-->>REG: MemoryTdaiConfig

        REG->>REG: initTimeModule()
        REG->>REG: ensurePluginHookPolicy()

        REG->>REG: resolveOpenClawStateDir()
        REG->>PF: initDataDirectories(pluginDataDir)

        REG->>ADP: new OpenClawHostAdapter({api, pluginDataDir, openclawConfig})
        REG->>TC: new TdaiCore({hostAdapter, config, sessionFilter})

        REG->>TC: core.initialize()
        TC->>PF: createPipeline()
        PF->>PF: initStores()
        PF->>PM: createPipelineManager()
        PF->>PM: setL1Runner(createL1Runner())
        PF->>PM: setPersister(createPersister())
        TC->>PM: start(restoredStates)
        PM->>PM: recoverPendingSessions()

        REG->>REG: getOrCreateInstanceId()
        REG->>REG: initReporter()

        alt memoryCleanup.enabled
            REG->>REG: new LocalMemoryCleaner()
            REG->>REG: memoryCleaner.start()
        end

        Note over REG: 注册工具
        REG->>OC: registerTool(tdai_memory_search)
        REG->>OC: registerTool(tdai_conversation_search)

        Note over REG: 注册生命周期钩子
        REG->>OC: on("before_prompt_build", auto-recall)
        REG->>OC: on("before_message_write", strip memories)
        REG->>OC: on("agent_end", auto-capture)
        REG->>OC: on("gateway_stop", destroy)

        alt offload.enabled
            REG->>OC: registerOffload(api, cfg.offload)
        end

        REG->>OC: registerCli(memory-tdai)
    end
```

**钩子处理流程：**

| 钩子 | 触发时机 | 核心操作 |
|------|----------|----------|
| `before_prompt_build` | 每轮对话前 | 缓存原始提示 → handleBeforeRecall → 注入记忆上下文 |
| `before_message_write` | 消息持久化前 | 剥离 `<relevant-memories>` 标签，防止污染历史记录 |
| `agent_end` | Agent 回复完成后 | handleTurnCommitted → L0 记录 → 通知调度器 → 上报指标 |
| `gateway_stop` | 网关关闭时 | destroy → 停止清理器 → 排空管道 → 关闭存储 |

---

## 11. 优雅关闭与恢复设计

### 11.1 优雅关闭流程

```mermaid
sequenceDiagram
    participant SIG as SIGINT/SIGTERM
    participant GS as gateway_stop 钩子
    participant MC as MemoryCleaner
    participant PM as PipelineManager
    participant L1Q as L1 SerialQueue
    participant L2Q as L2 SerialQueue
    participant L3Q as L3 SerialQueue
    participant VS as VectorStore
    participant ES as EmbeddingService

    SIG->>GS: 触发关闭
    GS->>GS: await coreReady

    GS->>MC: memoryCleaner.destroy()
    MC-->>GS: 清理器已停止

    GS->>PM: core.destroy()

    Note over PM: Step 1: 刷新 L1 空闲定时器
    PM->>PM: 遍历 sessionTimers
    PM->>L1Q: enqueueL1(sessionKey, "flush")<br/>（仅缓冲区非空的会话）

    Note over PM: Step 2: 等待 L1 排空
    PM->>L1Q: onIdle()
    L1Q-->>PM: L1 队列已排空

    Note over PM: Step 3: 刷新 L2 定时器
    PM->>PM: 遍历 sessionTimers
    PM->>L2Q: l2Schedule.flush()

    Note over PM: Step 4: 等待 L2 + L3 排空
    PM->>L2Q: onIdle()
    PM->>L3Q: onIdle()
    L2Q-->>PM: L2 队列已排空
    L3Q-->>PM: L3 队列已排空

    PM->>PM: persistStates()
    PM-->>GS: 管道已销毁

    GS->>VS: vectorStore.close()
    GS->>ES: embeddingService.close()
    GS->>GS: resetStores()

    alt 超时（2s）
        Note over GS: Promise.race 超时保护
        GS->>PM: persistStates()（保存未完成状态）
        Note over GS: 未完成工作将在下次启动时恢复
    end
```

### 11.2 恢复流程

```mermaid
flowchart TD
    START["PipelineManager.start()"] --> RESTORE["从检查点恢复<br/>sessionStates"]
    RESTORE --> FILTER["过滤内部会话<br/>SessionFilter.shouldSkip()"]
    FILTER --> BACKFILL["回填 warmup_threshold<br/>缺失字段默认为 0"]
    BACKFILL --> RECOVER["recoverPendingSessions()"]

    RECOVER --> LOOP["遍历所有会话状态"]
    LOOP --> CHECK{"conversation_count > 0<br/>或 l2_pending_l1_count > 0?"}
    CHECK -->|"否"| NEXT["跳过"]
    CHECK -->|"是"| MERGE["l2_pending_l1_count =<br/>max(l2_pending_l1_count, conversation_count)"]
    MERGE --> RESET["conversation_count = 0<br/>（消息缓冲区已丢失）"]
    RESET --> ARM["advanceL2Timer(sessionKey)<br/>延迟触发 L2"]
    ARM --> NEXT

    NEXT --> MORE{"还有更多会话?"}
    MORE -->|"是"| LOOP
    MORE -->|"否"| DONE["恢复完成<br/>管道开始运行"]

    style DONE fill:#6f6,color:#fff
```

**关闭与恢复设计要点：**

1. **分层排空**：L1→L2→L3 依次排空，保证数据完整性
2. **超时保护**：管道排空 2s 超时，网关关闭 3s 超时，确保进程不会无限挂起
3. **状态持久化**：无论排空成功与否，始终持久化当前状态到检查点
4. **消息丢失容忍**：重启后内存消息缓冲区为空，但 L0 JSONL 文件中的数据不会丢失；恢复时通过 `advanceL2Timer` 触发 L2 重新处理
5. **会话 GC**：定期清理长期不活跃的会话（3 倍 activeWindow），防止内存无限增长；被清理的会话可从检查点恢复
