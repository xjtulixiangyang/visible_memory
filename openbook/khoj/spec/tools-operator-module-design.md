# Khoj Tools 与 Operator 模块设计文档

## 1. 模块概述

### 1.1 Tools 模块

Tools 模块是 Khoj 的工具系统核心，负责为 AI Agent 提供与外部世界交互的能力。该模块包含三大工具：

| 工具 | 文件 | 职责 |
|------|------|------|
| **在线搜索** | `online_search.py` | 搜索互联网、读取网页、提取和去重信息 |
| **代码执行** | `run_code.py` | 生成 Python 代码并在沙箱中安全执行 |
| **MCP 协议** | `mcp.py` | 通过 Model Context Protocol 连接外部工具服务器 |

### 1.2 Operator 模块

Operator 模块实现了 Khoj 的计算机/浏览器自动化操作能力，使 AI 能够像人类一样操控图形界面完成任务。该模块采用 **Agent-Environment** 分离架构：

- **Agent 层**：负责感知环境状态、决策下一步动作
- **Environment 层**：负责执行动作、返回环境状态
- **Action 层**：定义标准化的操作动作模型

```mermaid
graph TB
    subgraph "Tools 模块"
        T1[在线搜索]
        T2[代码执行]
        T3[MCP 协议]
    end
    subgraph "Operator 模块"
        A[Agent 层]
        E[Environment 层]
        AC[Action 层]
    end
    subgraph "多模态输出"
        M1[图片生成]
        M2[语音合成]
    end
    R[Research 模块] --> T1
    R --> T2
    R --> T3
    R --> A
    R --> M1
    A --> AC
    AC --> E
```

---

## 2. 工具系统架构

### 2.1 工具选择与调用流程

Khoj 的工具选择采用两级路由机制：

1. **第一级**：`aget_data_sources_and_output_format` — 判断用户查询需要哪些数据源和输出模式
2. **第二级**：`apick_next_tool` — 在 Research 模式中，由 LLM 选择具体工具及参数

```mermaid
flowchart TD
    U[用户查询] --> DS{aget_data_sources<br/>and_output_format}
    DS -->|选择数据源| R[Research 模块]
    DS -->|直接回答| CHAT[对话回复]

    R --> PT{apick_next_tool}
    PT -->|LLM 决策| TOOLS[可用工具列表]

    TOOLS --> SS[SemanticSearchFiles]
    TOOLS --> SW[SearchWeb]
    TOOLS --> RW[ReadWebpage]
    TOOLS --> PC[PythonCoder]
    TOOLS --> OC[OperateComputer]
    TOOLS --> VF[ViewFile / ListFiles]
    TOOLS --> RS[RegexSearchFiles]
    TOOLS --> MCP[MCP 工具]

    SS --> EXEC[execute_tool]
    SW --> EXEC
    RW --> EXEC
    PC --> EXEC
    VF --> EXEC
    RS --> EXEC
    MCP --> EXEC
    OC --> STREAM[流式执行<br/>operate_environment]

    EXEC --> RESULT[ToolExecutionResult]
    STREAM --> RESULT

    RESULT --> NEXT{继续研究?}
    NEXT -->|是| PT
    NEXT -->|否| SUMMARY[汇总结果]
```

### 2.2 工具可用性判断

工具的可用性由多个条件动态决定：

```mermaid
flowchart LR
    subgraph "可用性检查"
        C1{用户有文档?} -->|否| HIDE1[隐藏文档搜索工具]
        C2{网络搜索启用?} -->|否| HIDE2[隐藏网页搜索工具]
        C3{代码沙箱启用?} -->|否| HIDE3[隐藏代码执行工具]
        C4{Operator 启用?} -->|否| HIDE4[隐藏计算机操作工具]
        C5{Agent 工具限制} -->|有限制| HIDE5[仅显示允许的工具]
    end
```

---

## 3. 在线搜索工具

### 3.1 搜索流程

```mermaid
flowchart TD
    Q[用户查询] --> GOS[generate_online_subqueries<br/>生成子查询]
    GOS --> DEDUP[去重: 排除已搜索的子查询]
    DEDUP --> SE{选择搜索引擎}

    SE -->|Serper| SP[search_with_serper]
    SE -->|Exa| EX[search_with_exa]
    SE -->|Firecrawl| FC[search_with_firecrawl]
    SE -->|Google| GG[search_with_google]
    SE -->|SearXNG| SX[search_with_searxng]

    SP --> RES[搜索结果]
    EX --> RES
    FC --> RES
    GG --> RES
    SX --> RES

    RES -->|有结果| DEDUP_ORG[deduplicate_organic_results<br/>跨查询去重]
    RES -->|无结果| NEXT_SE[尝试下一个搜索引擎]
    NEXT_SE --> SE

    DEDUP_ORG --> AUTO{自动读取网页?}
    AUTO -->|是 KHOJ_AUTO_READ_WEBPAGE| READ[读取网页内容]
    AUTO -->|否| YIELD[返回搜索结果]

    READ --> SCRAPE[scrape_webpage_with_fallback<br/>多爬虫回退]
    SCRAPE --> EXTRACT[extract_from_webpage<br/>提取相关信息]
    EXTRACT --> YIELD
```

### 3.2 搜索引擎支持

| 引擎 | API | 特点 |
|------|-----|------|
| **Serper** | `google.serper.dev` | 支持 answerBox、knowledgeGraph、peopleAlsoAsk |
| **Exa** | `api.exa.ai` | 支持内容直接返回，减少二次请求 |
| **Firecrawl** | `api.firecrawl.dev/v2/search` | 支持 Markdown 内容直接返回 |
| **Google** | `customsearch/v1` | 支持知识图谱 |
| **SearXNG** | 自托管实例 | 隐私友好，HTML 解析 |

### 3.3 网页读取与信息提取

网页读取采用**多爬虫回退**策略：

```mermaid
flowchart TD
    URL[目标 URL] --> CHECK{内部 URL?}
    CHECK -->|是| DIRECT[仅使用 Direct 爬虫]
    CHECK -->|否| SCRAPERS[获取已启用爬虫列表]

    SCRAPERS --> S1[Firecrawl]
    S1 -->|失败| S2[Olostep]
    S2 -->|失败| S3[Exa]
    S3 -->|失败| S4[Direct<br/>aiohttp + BeautifulSoup]
    S4 -->|失败| ERR[所有爬虫失败]

    S1 -->|成功| CONTENT[网页内容]
    S2 -->|成功| CONTENT
    S3 -->|成功| CONTENT
    S4 -->|成功| CONTENT

    CONTENT --> MD[markdownify 转换]
    MD --> LLM[LLM 提取相关信息<br/>extract_relevant_info]
    LLM --> RESULT[结构化提取结果]
```

### 3.4 结果去重

`deduplicate_organic_results` 函数基于链接 URL 在所有查询间去重，确保同一链接只出现一次：

```mermaid
flowchart LR
    R1[查询1结果] --> SEEN[seen_links 集合]
    R2[查询2结果] --> SEEN
    R3[查询3结果] --> SEEN
    SEEN --> FILTER[过滤已见链接]
    FILTER --> FINAL[去重后的结果]
```

---

## 4. 代码执行工具

### 4.1 代码执行流程

```mermaid
flowchart TD
    INST[用户指令] --> GEN[generate_python_code<br/>LLM 生成代码]
    GEN --> PROMPT[构建代码生成 Prompt]
    PROMPT --> SANDBOX_CTX{沙箱类型?}
    SANDBOX_CTX -->|E2B| E2B_CTX[e2b_sandbox_context]
    SANDBOX_CTX -->|Terrarium| TERR_CTX[terrarium_sandbox_context]

    E2B_CTX --> LLM[调用 LLM 生成代码]
    TERR_CTX --> LLM

    LLM --> PARSE[解析 Markdown 代码块]
    PARSE --> FILE_MATCH[匹配用户文件引用]
    FILE_MATCH --> PREP[准备输入数据<br/>Base64 编码]

    PREP --> EXEC_TYPE{沙箱类型?}
    EXEC_TYPE -->|E2B| E2B_EXEC[execute_e2b<br/>E2B 云沙箱]
    EXEC_TYPE -->|Terrarium| TERR_EXEC[execute_terrarium<br/>Terrarium 本地沙箱]

    E2B_EXEC --> RESULT[执行结果]
    TERR_EXEC --> RESULT

    RESULT --> TRUNC[truncate_code_context<br/>截断过长输出]
    TRUNC --> YIELD[返回代码 + 结果]
```

### 4.2 沙箱执行对比

| 特性 | E2B | Terrarium |
|------|-----|-----------|
| 执行环境 | 云端沙箱 | 本地 Docker |
| 网络访问 | 支持 | 不支持 |
| 文件 I/O | 原生支持 | Base64 传输 |
| 超时 | 60s（代码）+ 120s（沙箱） | 30s |
| 重试 | 3次，指数退避 | 3次，指数退避 |

---

## 5. MCP 协议集成

### 5.1 MCP 客户端架构

```mermaid
classDiagram
    class MCPClient {
        +name: str
        +path: str
        +api_key: Optional~str~
        +session: Optional~ClientSession~
        +exit_stack: AsyncExitStack
        +connect() async
        +get_tools() async List~Tool~
        +run_tool(tool_name, input_data) async
        +close() async
    }

    class ClientSession {
        +list_tools() async
        +call_tool(name, args) async
        +initialize() async
    }

    MCPClient --> ClientSession : 持有
```

### 5.2 连接方式

```mermaid
flowchart TD
    PATH[服务器路径] --> CHECK{路径类型?}
    CHECK -->|http:// 或 https://| SSE[SSE 连接]
    CHECK -->|本地路径| STDIO[Stdio 连接]

    SSE --> SSE_CLIENT[sse_client]
    SSE_CLIENT --> SESSION1[ClientSession]

    STDIO --> LANG{语言类型?}
    LANG -->|.py 文件| PYTHON[python script.py]
    LANG -->|.js 文件| NODE[node script.js]
    LANG -->|npm 包| NPX[npx @package]

    PYTHON --> STDIO_CLIENT[stdio_client]
    NODE --> STDIO_CLIENT
    NPX --> STDIO_CLIENT
    STDIO_CLIENT --> SESSION2[ClientSession]
```

---

## 6. Operator 系统架构

### 6.1 类继承体系

```mermaid
classDiagram
    class OperatorAgent {
        <<abstract>>
        +query: str
        +vision_model: ChatModel
        +environment_type: EnvironmentType
        +max_iterations: int
        +messages: List~AgentMessage~
        +act(current_state: EnvState) async AgentActResult
        +add_action_results(env_steps, agent_action) void
        +summarize(current_state, summarize_prompt) async str
        +reset() void
    }

    class AnthropicOperatorAgent {
        +act(current_state) async AgentActResult
        +add_action_results(env_steps, agent_action) void
        +get_instructions(env_type, current_state) str
        +get_tools(env_type, current_state) list
    }

    class OpenAIOperatorAgent {
        +act(current_state) async AgentActResult
        +add_action_results(env_steps, agent_action) void
        +summarize(current_state, summarize_prompt) async str
        +get_instructions(env_type, current_state) str
        +get_tools(env_type, current_state) list
    }

    class BinaryOperatorAgent {
        +reasoning_model: ChatModel
        +grounding_model: ChatModel
        +grounding_agent: GroundingAgent | GroundingAgentUitars
        +act(current_state) async AgentActResult
        +act_reason(current_state) async dict
        +act_ground(action_instruction, current_state) async AgentActResult
    }

    class GroundingAgent {
        +model: ChatModel
        +client: OpenAI | AzureOpenAI
        +act(instruction, current_state) async tuple
    }

    class GroundingAgentUitars {
        +model_name: str
        +client: AsyncOpenAI
        +act(instruction, current_state) async tuple
    }

    OperatorAgent <|-- AnthropicOperatorAgent
    OperatorAgent <|-- OpenAIOperatorAgent
    OperatorAgent <|-- BinaryOperatorAgent
    BinaryOperatorAgent --> GroundingAgent : 使用
    BinaryOperatorAgent --> GroundingAgentUitars : 使用
```

### 6.2 环境类继承体系

```mermaid
classDiagram
    class Environment {
        <<abstract>>
        +start(width, height) async void
        +step(action: OperatorAction) async EnvStepResult
        +close() async void
        +get_state() async EnvState
    }

    class BrowserEnvironment {
        +playwright: Playwright
        +browser: Browser
        +page: Page
        +visited_urls: Set~str~
        +start(width, height) async void
        +step(action) async EnvStepResult
        +get_state() async EnvState
        +close() async void
    }

    class ComputerEnvironment {
        +provider: Literal
        +docker_container_name: str
        +start(width, height) async void
        +step(action) async EnvStepResult
        +get_state() async EnvState
        +close() async void
    }

    class EnvState {
        +height: int
        +width: int
        +screenshot: Optional~str~
        +url: Optional~str~
    }

    class EnvStepResult {
        +type: Literal
        +output: Optional
        +error: Optional~str~
        +current_url: Optional~str~
        +screenshot_base64: Optional~str~
    }

    Environment <|-- BrowserEnvironment
    Environment <|-- ComputerEnvironment
    Environment ..> EnvState : 返回
    Environment ..> EnvStepResult : 返回
```

---

## 7. 操作员执行流程

### 7.1 主循环时序图

```mermaid
sequenceDiagram
    participant R as Research 模块
    participant OE as operate_environment
    participant Agent as OperatorAgent
    participant Env as Environment

    R->>OE: 启动操作 (query, environment_type)
    OE->>OE: 获取视觉模型
    OE->>OE: 创建 Agent (Anthropic/OpenAI/Binary)
    OE->>Env: environment.start(width, height)
    Env-->>OE: 环境就绪

    loop 迭代循环 (max_iterations)
        OE->>Env: environment.get_state()
        Env-->>OE: EnvState (screenshot, url)

        OE->>Agent: agent.act(current_state)
        Agent-->>OE: AgentActResult (actions, action_results)

        loop 每个 Action
            OE->>Env: environment.step(action)
            Env-->>OE: EnvStepResult (output, screenshot, error)
        end

        OE->>R: yield 状态更新 (流式)

        alt 任务完成 / 迭代上限
            OE->>Agent: agent.summarize(env_state)
            Agent-->>OE: 总结文本
        end
    end

    OE->>Env: environment.close()
    OE->>R: yield OperatorRun (最终结果)
```

### 7.2 Binary Agent 双模型协作

```mermaid
sequenceDiagram
    participant OE as operate_environment
    participant BA as BinaryOperatorAgent
    participant RM as Reasoning LLM
    participant GA as GroundingAgent
    participant Env as Environment

    OE->>BA: act(current_state)

    Note over BA: Step 1: 推理
    BA->>RM: act_reason(current_state)
    RM-->>BA: 自然语言动作指令

    alt 动作类型 = error/done
        BA-->>OE: AgentActResult (终止)
    else 动作类型 = action
        Note over BA: Step 2: 接地
        BA->>GA: act(instruction, current_state)
        GA-->>BA: (rendered_response, actions)
        BA-->>OE: AgentActResult
    end

    OE->>Env: 执行 actions
    Env-->>OE: EnvStepResult
```

---

## 8. Research 模块

### 8.1 多轮研究迭代流程

```mermaid
flowchart TD
    START[用户查询] --> INIT[初始化 Research]
    INIT --> MCP_INIT[创建 MCP 客户端]
    MCP_INIT --> HISTORY[构建研究对话历史]

    HISTORY --> LOOP{迭代 < MAX_ITERATIONS?}
    LOOP -->|否| END[关闭 MCP 连接<br/>结束研究]

    LOOP -->|是| PICK[apick_next_tool]
    PICK --> LLM_DECIDE[LLM 选择工具]
    LLM_DECIDE --> ITERATIONS[ResearchIteration 列表]

    ITERATIONS --> SPLIT{工具类型?}
    SPLIT -->|流式工具 Operator| STREAM[顺序执行]
    SPLIT -->|并行工具 搜索/代码/MCP| PARALLEL[并行执行]

    STREAM --> EXEC_OP[operate_environment]
    PARALLEL --> EXEC_TOOL[execute_tool]

    EXEC_OP --> RESULT[ToolExecutionResult]
    EXEC_TOOL --> RESULT

    RESULT --> BUILD[构建 summarizedResult]
    BUILD --> APPEND[追加到 previous_iterations]
    APPEND --> YIELD[yield ResearchIteration]
    YIELD --> LOOP
```

---

## 9. 图片生成和语音合成

### 9.1 图片生成流程

```mermaid
flowchart TD
    MSG[用户消息] --> CONFIG[获取图片生成配置]
    CONFIG -->|未配置| ERR[返回 501 错误]
    CONFIG -->|已配置| MODEL{text2image_model]

    MODEL --> MULTI{多模态模型?}
    MULTI -->|否| ENHANCE[generate_better_image_prompt<br/>LLM 增强提示词]
    MULTI -->|是| DIRECT[直接使用原始消息]

    ENHANCE --> API{模型类型?}
    DIRECT --> API
    API -->|OpenAI| OPENAI[generate_image_with_openai]
    API -->|Replicate| REPLICATE[generate_image_with_replicate]
    API -->|Google| GOOGLE[generate_image_with_google]

    OPENAI --> WEBP[转换为 WebP]
    REPLICATE --> WEBP
    GOOGLE --> WEBP

    WEBP --> S3[上传到 S3]
    S3 -->|成功| URL[返回图片 URL]
    S3 -->|失败| B64[返回 Base64 图片]
```

### 9.2 语音合成

```mermaid
flowchart TD
    TEXT[输入文本] --> MD[Markdown 渲染]
    MD --> HTML[转换为 HTML]
    HTML --> PLAIN[提取纯文本]
    PLAIN --> API[ElevenLabs API]
    API --> STREAM[流式音频响应]
    STREAM --> AUDIO[返回音频数据]
```

---

## 附录：模块依赖关系

```mermaid
graph TB
    subgraph "路由层"
        R_RESEARCH[routers/research.py]
        R_HELPERS[routers/helpers.py]
    end

    subgraph "工具层"
        T_SEARCH[tools/online_search.py]
        T_CODE[tools/run_code.py]
        T_MCP[tools/mcp.py]
    end

    subgraph "操作员层"
        O_INIT[operator/__init__.py]
        O_BASE[operator/operator_agent_base.py]
        O_ANTHROPIC[operator/operator_agent_anthropic.py]
        O_OPENAI[operator/operator_agent_openai.py]
        O_BINARY[operator/operator_agent_binary.py]
        O_BROWSER[operator/operator_environment_browser.py]
        O_COMPUTER[operator/operator_environment_computer.py]
        O_GROUNDING[operator/grounding_agent.py]
    end

    subgraph "多模态层"
        M_IMAGE[image/generate.py]
        M_SPEECH[speech/text_to_speech.py]
    end

    R_RESEARCH --> T_SEARCH
    R_RESEARCH --> T_CODE
    R_RESEARCH --> T_MCP
    R_RESEARCH --> O_INIT
    R_RESEARCH --> R_HELPERS

    O_INIT --> O_BASE
    O_INIT --> O_ANTHROPIC
    O_INIT --> O_OPENAI
    O_INIT --> O_BINARY
    O_INIT --> O_BROWSER
    O_INIT --> O_COMPUTER

    O_BINARY --> O_GROUNDING
```
