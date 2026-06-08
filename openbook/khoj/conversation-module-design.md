# Khoj Conversation 模块设计文档

## 1. 模块概述

Conversation 模块是 Khoj 的核心对话引擎，负责处理用户从发送消息到接收完整响应的整个生命周期。该模块实现了以下关键职责：

- **多模型适配**：统一适配 OpenAI、Anthropic、Google 三大 LLM 提供商，以及兼容 OpenAI API 的第三方服务（DeepSeek、Groq、Cerebras 等）
- **工具编排**：根据用户查询智能选择数据源（笔记搜索、在线搜索、网页阅读、代码执行、浏览器操作、研究模式）和输出格式（文本、图像、图表）
- **流式响应**：支持 SSE（Server-Sent Events）和 WebSocket 两种流式传输协议，实现实时响应
- **上下文管理**：组装对话历史、搜索结果、在线结果、代码执行结果、用户记忆等多源上下文，并进行智能截断
- **对话持久化**：将对话消息、工具调用结果、推理过程等保存到数据库，支持中断恢复

### 模块架构总览

```mermaid
graph TB
    subgraph "API 层"
        WS[WebSocket /ws]
        HTTP[HTTP POST /]
        SSE[SSE Streaming]
    end

    subgraph "对话编排层"
        EG[event_generator]
        SC[send_event / send_llm_response]
        DM[disconnect_monitor]
    end

    subgraph "工具调度层"
        DS[aget_data_sources_and_output_format]
        SD[search_documents]
        SO[search_online]
        RW[read_webpages]
        RC[run_code]
        OP[operate_environment]
        RS[research]
        TI[text_to_image]
        MD[generate_mermaidjs_diagram]
    end

    subgraph "LLM 适配层"
        OAI[converse_openai]
        ANT[converse_anthropic]
        GEM[converse_gemini]
    end

    subgraph "上下文管理层"
        BCC[build_conversation_context]
        GCM[generate_chatml_messages_with_context]
        TM[truncate_messages]
        SMW[send_message_to_model_wrapper]
    end

    subgraph "持久化层"
        SCL[save_to_conversation_log]
        AUM[ai_update_memories]
        CT[commit_conversation_trace]
    end

    WS --> EG
    HTTP --> EG
    EG --> SC
    EG --> DM
    EG --> DS
    EG --> SD
    EG --> SO
    EG --> RW
    EG --> RC
    EG --> OP
    EG --> RS
    EG --> TI
    EG --> MD
    EG --> BCC
    BCC --> GCM
    GCM --> TM
    EG --> SMW
    SMW --> OAI
    SMW --> ANT
    SMW --> GEM
    EG --> SCL
    SCL --> AUM
    SCL --> CT
```

---

## 2. 核心组件

### 2.1 API 路由层

| 组件 | 文件 | 职责 |
|------|------|------|
| `api_chat` | `routers/api_chat.py` | FastAPI 路由器，提供 HTTP/WebSocket 对话端点 |
| `event_generator` | `routers/api_chat.py` | 核心异步生成器，编排整个对话处理流程 |
| `chat_ws` | `routers/api_chat.py` | WebSocket 端点，管理连接、中断和消息缓冲 |
| `process_chat_request` | `routers/api_chat.py` | WebSocket 请求处理器，实现消息缓冲和防抖 |

### 2.2 辅助函数层

| 组件 | 文件 | 职责 |
|------|------|------|
| `agenerate_chat_response` | `routers/helpers.py` | 构建上下文消息并调用对应 LLM 进行流式对话 |
| `send_message_to_model_wrapper` | `routers/helpers.py` | 带回退机制的模型调用封装（主模型失败后尝试备选模型） |
| `send_message_to_model` | `routers/helpers.py` | 根据模型类型分发到具体 LLM 提供商 |
| `aget_data_sources_and_output_format` | `routers/helpers.py` | 使用 LLM 推断用户查询需要的数据源和输出格式 |
| `build_conversation_context` | `routers/helpers.py` | 构建系统提示词、上下文消息和 ChatML 消息列表 |
| `ChatEvent` | `processor/conversation/utils.py` | 枚举类，定义流式事件类型 |

### 2.3 LLM 提供商适配层

| 组件 | 文件 | 职责 |
|------|------|------|
| `converse_openai` | `processor/conversation/openai/gpt.py` | OpenAI 流式对话（支持 Chat Completions API 和 Responses API） |
| `converse_anthropic` | `processor/conversation/anthropic/anthropic_chat.py` | Anthropic Claude 流式对话 |
| `converse_gemini` | `processor/conversation/google/gemini_chat.py` | Google Gemini 流式对话 |
| `*_send_message_to_model` | 各提供商目录 | 同步/非流式模型调用（用于工具推断等辅助任务） |

### 2.4 上下文与工具层

| 组件 | 文件 | 职责 |
|------|------|------|
| `generate_chatml_messages_with_context` | `processor/conversation/utils.py` | 组装完整 ChatML 消息列表（历史+上下文+当前查询） |
| `truncate_messages` | `processor/conversation/utils.py` | 按 token 预算截断消息 |
| `save_to_conversation_log` | `processor/conversation/utils.py` | 异步保存对话到数据库 |
| `prompts` | `processor/conversation/prompts.py` | 所有提示词模板集合 |

---

## 3. 对话流程时序图

### 3.1 完整对话处理流程

```mermaid
sequenceDiagram
    participant U as 用户
    participant API as API路由层
    participant EG as event_generator
    participant DS as 工具选择器
    participant Tools as 工具执行器
    participant LLM as LLM适配层
    participant DB as 数据库

    U->>API: 发送消息 (HTTP/WebSocket)
    API->>EG: 调用 event_generator(body)

    rect rgb(240, 248, 255)
        Note over EG: 阶段1: 初始化
        EG->>EG: 解析请求参数 (q, images, files, location)
        EG->>EG: 上传图片到存储桶
        EG->>EG: 收集附加文件上下文
        EG->>EG: 启动断连监控任务
    end

    rect rgb(255, 248, 240)
        Note over EG: 阶段2: 获取对话与Agent
        EG->>DB: aget_conversation_by_user()
        DB-->>EG: conversation 对象
        EG->>DB: 获取 Agent 配置
        EG->>DB: 获取用户记忆 (recent + long_term)
        EG->>EG: 检查中断消息恢复
    end

    rect rgb(240, 255, 240)
        Note over EG: 阶段3: 工具选择
        EG->>DS: aget_data_sources_and_output_format()
        DS->>LLM: send_message_to_model_wrapper(pick_relevant_tools)
        LLM-->>DS: {"source": [...], "output": "..."}
        DS-->>EG: chosen_io (数据源 + 输出格式)
        EG->>U: STATUS: "Selected Tools: ..."
    end

    rect rgb(255, 255, 240)
        Note over EG: 阶段4: 上下文收集
        alt Research 模式
            EG->>Tools: research()
            Tools-->>EG: ResearchIteration 流
        else 常规模式
            alt Notes 命令
                EG->>Tools: search_documents()
                Tools-->>EG: compiled_references
            end
            alt Online 命令
                EG->>Tools: search_online()
                Tools-->>EG: online_results
            end
            alt Webpage 命令
                EG->>Tools: read_webpages()
                Tools-->>EG: direct_web_pages
            end
            alt Code 命令
                EG->>Tools: run_code()
                Tools-->>EG: code_results
            end
            alt Operator 命令
                EG->>Tools: operate_environment()
                Tools-->>EG: operator_results
            end
        end
        EG->>U: REFERENCES 事件
    end

    rect rgb(248, 240, 255)
        Note over EG: 阶段5: 输出生成
        alt Image 输出
            EG->>Tools: text_to_image()
            Tools-->>EG: generated_image
            EG->>U: GENERATED_ASSETS 事件
        else Diagram 输出
            EG->>Tools: generate_mermaidjs_diagram()
            Tools-->>EG: mermaidjs_diagram
            EG->>U: GENERATED_ASSETS 事件
        end
        EG->>U: STATUS: "Generating a well-informed response"
        EG->>LLM: agenerate_chat_response()
        LLM-->>EG: AsyncGenerator[ResponseWithThought]
        loop 流式输出
            EG->>U: THOUGHT / MESSAGE 事件
        end
    end

    rect rgb(240, 240, 255)
        Note over EG: 阶段6: 收尾
        EG->>DB: save_to_conversation_log() (异步)
        EG->>DB: ai_update_memories() (异步)
        EG->>U: END_LLM_RESPONSE + USAGE + END_RESPONSE
    end
```

### 3.2 WebSocket 连接生命周期

```mermaid
sequenceDiagram
    participant U as 用户
    participant WS as chat_ws
    participant PQ as process_chat_request
    participant EG as event_generator
    participant IQ as interrupt_queue

    U->>WS: WebSocket 连接
    WS->>WS: 验证 Origin
    WS->>WS: 检查连接数限制
    WS->>WS: 注册连接

    loop 消息循环
        U->>WS: 发送 JSON 消息

        alt 中断消息
            WS->>IQ: 放入中断信号
            WS->>U: interrupt_acknowledged
        else 聊天消息
            WS->>WS: 速率限制检查
            WS->>PQ: 创建异步任务
            PQ->>EG: event_generator()
            loop 流式输出
                EG-->>PQ: 事件流
                PQ->>PQ: 消息缓冲 (100ms / 512B)
                PQ->>U: 批量发送文本 + END_EVENT
            end
        end
    end

    U->>WS: 断开连接
    WS->>IQ: 发送 INTERRUPT 信号
    WS->>WS: 注销连接
```

---

## 4. 多模型适配

### 4.1 模型分发架构

```mermaid
graph TD
    SMW[send_message_to_model_wrapper] -->|获取主模型| PCM[primary_chat_model]
    SMW -->|获取备选模型| FB[fallback_models]
    SMW -->|构建模型列表| MTL[models_to_try]

    MTL -->|遍历尝试| SMM[send_message_to_model]

    SMM -->|model_type=OPENAI| OAI[openai_send_message_to_model]
    SMM -->|model_type=ANTHROPIC| ANT[anthropic_send_message_to_model]
    SMM -->|model_type=GOOGLE| GEM[gemini_send_message_to_model]

    OAI --> CO[converse_openai]
    ANT --> CA[converse_anthropic]
    GEM --> CG[converse_gemini]

    CO -->|supports_responses_api| RCB[responses_chat_completion_with_backoff]
    CO -->|else| CCB[chat_completion_with_backoff]

    CA --> ACB[anthropic_chat_completion_with_backoff]
    CG --> GCB[gemini_chat_completion_with_backoff]
```

### 4.2 各提供商适配差异

```mermaid
graph LR
    subgraph "OpenAI 适配"
        O1[Chat Completions API] -->|标准模型| O2[completion_with_backoff]
        O3[Responses API] -->|o1/o3/o4/gpt-5| O4[responses_completion_with_backoff]
        O5[推理模型] -->|o1/o3/o4/gpt-5| O6[reasoning_effort 配置]
        O7[DeepSeek/Qwen] -->|流式思维| O8[in_stream_thought_processor]
    end

    subgraph "Anthropic 适配"
        A1[Messages API] --> A2[anthropic_completion_with_backoff]
        A3[推理模型] -->|claude-3-7/sonnet-4/opus-4| A4[thinking 配置]
        A5[JSON 输出] -->|prefill '{'| A6[assistant 消息预填充]
        A7[工具调用] -->|ToolParam| A8[Anthropic 工具格式]
    end

    subgraph "Google 适配"
        G1[Generate Content API] --> G2[gemini_completion_with_backoff]
        G3[推理模型] -->|gemini-2.5/3| G4[ThinkingConfig 配置]
        G5[安全过滤] -->|SafetySetting| G6[BLOCK_ONLY_HIGH]
        G7[速率限制] -->|retryDelay| G8[自定义等待策略]
    end
```

### 4.3 回退机制

```mermaid
flowchart TD
    START[开始调用] --> PRIMARY[尝试主模型]
    PRIMARY -->|成功| SUCCESS[返回结果]
    PRIMARY -->|失败| CHECK{是否可重试?}

    CHECK -->|RateLimit/Timeout/500| FALLBACK[尝试备选模型]
    CHECK -->|其他错误| RAISE[抛出异常]

    FALLBACK -->|成功| SUCCESS
    FALLBACK -->|失败| CHECK2{还有备选模型?}
    CHECK2 -->|是| FALLBACK
    CHECK2 -->|否| RAISE2[RetryableModelError]

    style PRIMARY fill:#4CAF50,color:white
    style FALLBACK fill:#FF9800,color:white
    style RAISE fill:#f44336,color:white
    style RAISE2 fill:#f44336,color:white
```

### 4.4 结构化输出支持等级

| 等级 | 值 | 说明 | 适用场景 |
|------|-----|------|---------|
| `NONE` | 0 | 不支持结构化输出 | deepseek-reasoner |
| `OBJECT` | 1 | 支持 JSON Object 模式 | Azure OpenAI, DeepInfra |
| `SCHEMA` | 2 | 支持 JSON Schema 约束 | 部分 OpenAI 兼容 API |
| `TOOL` | 3 | 支持工具调用方式 | OpenAI 官方 API |

---

## 5. 流式响应机制

### 5.1 两种流式传输协议

```mermaid
graph TB
    subgraph "SSE (HTTP POST)"
        H1[HTTP POST /api/chat] --> H2[event_generator]
        H2 --> H3[StreamingResponse]
        H3 -->|text/plain| H4[客户端接收流]
    end

    subgraph "WebSocket"
        W1[WS /api/chat/ws] --> W2[process_chat_request]
        W2 --> W3[event_generator]
        W3 --> W4[消息缓冲 MessageBuffer]
        W4 -->|100ms/512B 防抖| W5[websocket.send_text]
    end
```

### 5.2 事件类型体系

```mermaid
classDiagram
    class ChatEvent {
        <<enumeration>>
        START_LLM_RESPONSE
        END_LLM_RESPONSE
        MESSAGE
        REFERENCES
        GENERATED_ASSETS
        STATUS
        THOUGHT
        METADATA
        USAGE
        END_RESPONSE
        INTERRUPT
        END_EVENT
    }

    class SSEEvent {
        type: ChatEvent
        data: str | dict
    }

    ChatEvent --> SSEEvent : 产生
```

### 5.3 SSE 事件流格式

```mermaid
sequenceDiagram
    participant S as 服务端
    participant C as 客户端

    S->>C: {"type":"metadata","data":{"conversationId":"...","turnId":"..."}}
    S->>C: {"type":"status","data":"**Selected Tools:** notes, text"}
    S->>C: {"type":"references","data":{...}}
    S->>C: {"type":"status","data":"**Generating a well-informed response**"}
    S->>C: ␃🔚␗ (START_LLM_RESPONSE)
    S->>C: {"type":"thought","data":"Let me analyze..."}
    S->>C: message chunk 1
    S->>C: message chunk 2
    S->>C: message chunk 3
    S->>C: ␃🔚␗ (END_LLM_RESPONSE)
    S->>C: {"type":"usage","data":{...}}
    S->>C: ␃🔚␗ (END_RESPONSE)
```

### 5.4 WebSocket 消息缓冲机制

```mermaid
flowchart TD
    A[收到 event_generator 事件] --> B{事件类型判断}

    B -->|JSON 对象| C{END_LLM_RESPONSE?}
    C -->|是| D[刷新消息缓冲]
    C -->|否| E{THOUGHT 事件?}

    E -->|是| F[写入 thought_buffer]
    F --> G{缓冲区 >= 512B 或 超过 100ms?}
    G -->|是| H[刷新 thought_buffer → WebSocket]
    G -->|否| I[设置延迟刷新定时器]

    E -->|否| J[直接发送 → WebSocket]

    B -->|纯文本| K[写入 message_buffer]
    K --> L{缓冲区 >= 512B 或 超过 100ms?}
    L -->|是| M[刷新 message_buffer → WebSocket]
    L -->|否| N[设置延迟刷新定时器]

    D --> O[发送缓冲内容 + END_EVENT]
    H --> P[发送 thought JSON + END_EVENT]
    M --> Q[发送消息文本 + END_EVENT]
```

### 5.5 中断处理流程

```mermaid
sequenceDiagram
    participant U as 用户
    participant WS as WebSocket
    participant IQ as interrupt_queue
    participant EG as event_generator
    participant CE as cancellation_event

    U->>WS: {"type":"interrupt","query":"补充说明..."}
    WS->>IQ: put(query)
    WS->>U: {"type":"interrupt_message_acknowledged"}

    IQ-->>EG: get_message_from_queue()
    EG->>EG: q += "\n\n" + interrupt_query
    EG->>CE: 检查 cancellation_event

    alt END_EVENT
        EG->>CE: cancellation_event.set()
        EG->>EG: 保存部分对话到数据库
    else 新指令
        EG->>EG: 继续处理，追加指令到查询
    end
```

---

## 6. 上下文管理

### 6.1 上下文组装流程

```mermaid
flowchart TD
    START[build_conversation_context] --> SP[构建系统提示词]

    SP --> SP1{有 Agent?}
    SP1 -->|是| SP2[custom_personality 模板]
    SP1 -->|否| SP3[默认 personality 模板]
    SP2 --> SP4[+ Gemini 增强提示]
    SP3 --> SP4
    SP4 --> SP5[+ 位置信息]
    SP5 --> SP6[+ 用户名]

    SP6 --> CM[构建上下文消息]
    CM --> CM1{有笔记引用?}
    CM1 -->|是| CM2[+ notes_conversation]
    CM1 -->|否| CM3
    CM2 --> CM3{有在线结果?}
    CM3 -->|是| CM4[+ online_search_conversation]
    CM3 -->|否| CM5
    CM4 --> CM5{有代码结果?}
    CM5 -->|是| CM6[+ code_executed_context]
    CM5 -->|否| CM7
    CM6 --> CM7{有操作结果?}
    CM7 -->|是| CM8[+ operator_execution_context]
    CM7 -->|否| CM9

    CM8 --> CM9[generate_chatml_messages_with_context]
    CM9 --> HIST[提取对话历史]
    HIST --> H1[遍历 chat_history]
    H1 --> H2[添加在线上下文消息]
    H2 --> H3[添加代码上下文消息]
    H3 --> H4[添加操作上下文消息]
    H4 --> H5[添加笔记引用消息]
    H5 --> H6[添加生成资产消息]
    H6 --> H7[添加用户/助手消息]

    H7 --> CTX[添加额外上下文]
    CTX --> CTX1[+ 程序执行上下文]
    CTX1 --> CTX2[+ 上下文消息]
    CTX2 --> CTX3[+ 生成资产结果]
    CTX3 --> CTX4[+ 用户记忆]
    CTX4 --> CTX5[+ 当前用户消息]

    CTX5 --> TRUNC[truncate_messages]
    TRUNC --> RESULT[最终消息列表]
```

### 6.2 消息截断策略

```mermaid
flowchart TD
    START[truncate_messages] --> SEP[分离 system 和非 system 消息]
    SEP --> COUNT[计算总 token 数]
    COUNT --> CHECK{总 token > max_prompt_size?}

    CHECK -->|否| DONE[返回消息]
    CHECK -->|是| LOOP[截断循环]

    LOOP --> C1{首条消息 content > 1?}
    C1 -->|是且非 tool_call| C2[弹出最旧 content 部分]
    C1 -->|否| C3[弹出最旧消息]

    C3 --> C4{弹出的是 tool_call?}
    C4 -->|是| C5[同时弹出配对的 tool_result]
    C4 -->|否| C6[继续]

    C2 --> RECOUNT[重新计算 token]
    C5 --> RECOUNT
    C6 --> RECOUNT
    RECOUNT --> CHECK

    DONE --> FINAL{仍超限?}
    FINAL -->|否| RETURN[返回 system + messages]
    FINAL -->|是| LAST[单消息截断]
    LAST --> L1[分离上下文和问题]
    L1 --> L2[保留问题，截断上下文]
    L2 --> RETURN
```

### 6.3 Token 预算分配

```mermaid
pie title 模型 Token 预算分配 (以 gpt-4o 为例, 60K)
    "系统提示词" : 2000
    "对话历史" : 30000
    "上下文消息(搜索/在线/代码)" : 20000
    "当前查询+图片" : 5000
    "输出预留" : 3000
```

### 6.4 各模型上下文窗口配置

| 模型 | max_prompt_size | lookback_turns |
|------|----------------|----------------|
| gpt-4o | 60,000 | 80 |
| gpt-4.1-mini | 120,000 | 160 |
| o3 | 60,000 | 80 |
| claude-3-7-sonnet | 60,000 | 80 |
| gemini-2.5-flash | 120,000 | 160 |
| gemini-2.5-pro | 60,000 | 80 |

> `lookback_turns = max_prompt_size // 750`，用于限制从对话历史中提取的轮次数量。

---

## 7. 工具调用流程

### 7.1 工具选择流程

```mermaid
flowchart TD
    START[用户查询] --> CMD{有显式命令前缀?}

    CMD -->|/notes, /online 等| EXPLICIT[使用指定命令]
    CMD -->|无前缀| INFER[推断工具选择]

    INFER --> LLM1[LLM: pick_relevant_tools 提示词]
    LLM1 --> PARSE[解析 JSON: source + output]

    PARSE --> VALID{验证选择有效性}
    VALID -->|有效| APPLY[应用选择]
    VALID -->|无效| FALLBACK[回退到 Default/General]

    APPLY --> RESEARCH{包含 Research?}
    RESEARCH -->|是| RESEARCH_ONLY[仅使用 Research]
    RESEARCH -->|否| COMBINE[组合 sources + output]

    EXPLICIT --> RATE[速率限制检查]
    APPLY --> RATE
    FALLBACK --> RATE

    RATE --> EXEC[执行工具]
```

### 7.2 工具类型与执行流程

```mermaid
flowchart LR
    subgraph "数据源工具"
        N[Notes<br/>笔记搜索]
        O[Online<br/>在线搜索]
        W[Webpage<br/>网页阅读]
        C[Code<br/>代码执行]
        OP[Operator<br/>浏览器操作]
        R[Research<br/>深度研究]
    end

    subgraph "输出工具"
        T[Text<br/>文本输出]
        I[Image<br/>图像生成]
        D[Diagram<br/>图表生成]
    end

    N -->|search_documents| T
    O -->|search_online| T
    W -->|read_webpages| T
    C -->|run_code| T
    OP -->|operate_environment| T
    R -->|research| T

    N --> I
    O --> I
    N --> D
    O --> D
```

### 7.3 Research 模式详细流程

```mermaid
sequenceDiagram
    participant EG as event_generator
    participant RS as research()
    participant LLM as LLM (planner)
    participant TE as execute_tool
    participant TOOLS as 工具执行器

    EG->>RS: 启动研究模式
    RS->>LLM: plan_function_execution 提示词
    LLM-->>RS: 工具调用列表

    loop 每次迭代 (max_iterations)
        RS->>TE: 并行执行工具调用

        par 并行执行
            TE->>TOOLS: search_documents
            TOOLS-->>TE: document_results
        and
            TE->>TOOLS: search_online
            TOOLS-->>TE: online_results
        and
            TE->>TOOLS: run_code
            TOOLS-->>TE: code_results
        and
            TE->>TOOLS: operate_environment
            TOOLS-->>TE: operator_results
        end

        TE-->>RS: ToolExecutionResult
        RS->>RS: 构建 ResearchIteration

        RS->>LLM: 下一轮规划 (含前序结果)
        LLM-->>RS: 新工具调用 或 终止信号

        RS->>EG: yield ResearchIteration / status
    end

    RS-->>EG: 研究完成
```

### 7.4 工具调用在 LLM 中的消息格式

```mermaid
graph TD
    subgraph "Research 模式 - 工具调用消息"
        TC[Assistant: tool_call<br/>type=tool_call] --> TR[User: tool_result<br/>type=tool_result]
    end

    subgraph "OpenAI 格式"
        OTC["role: assistant<br/>tool_calls: [{id, function: {name, arguments}}]"]
        OTR["role: tool<br/>tool_call_id: id<br/>content: result"]
    end

    subgraph "Anthropic 格式"
        ATC["role: assistant<br/>content: [{type: tool_use, id, name, input}]"]
        ATR["role: user<br/>content: [{type: tool_result, tool_use_id, content}]"]
    end

    subgraph "Gemini 格式"
        GTC["role: model<br/>functionCall: {name, args}"]
        GTR["role: user<br/>functionResponse: {name, response}"]
    end

    TC --> OTC
    TR --> OTR
    TC --> ATC
    TR --> ATR
    TC --> GTC
    TR --> GTR
```

---

## 8. 关键数据结构

### 8.1 核心数据结构类图

```mermaid
classDiagram
    class ChatMessageModel {
        +by: str
        +message: str | list
        +created: str
        +images: list~str~
        +context: list
        +onlineContext: dict
        +codeContext: dict
        +operatorContext: list
        +researchContext: list
        +intent: Intent
        +queryFiles: list~FileAttachment~
        +trainOfThought: list
        +turnId: str
        +mermaidjsDiagram: str
    }

    class Intent {
        +type: str
        +query: str
        +inferred_queries: list~str~
        +memory_type: str
    }

    class ResponseWithThought {
        +text: str
        +thought: str
        +raw_content: list
    }

    class ResearchIteration {
        +query: ToolCall | str
        +context: list
        +onlineContext: dict
        +codeContext: dict
        +operatorContext: OperatorRun
        +summarizedResult: str
        +warning: str
        +raw_response: list
        +to_dict() dict
    }

    class OperatorRun {
        +query: str
        +trajectory: list~AgentMessage~
        +response: str
        +webpages: list~dict~
        +to_dict() dict
    }

    class ToolCall {
        +name: str
        +args: dict
        +id: str
    }

    class AgentMessage {
        +role: str
        +content: str | list
    }

    class ChatEvent {
        <<enumeration>>
        START_LLM_RESPONSE
        END_LLM_RESPONSE
        MESSAGE
        REFERENCES
        GENERATED_ASSETS
        STATUS
        THOUGHT
        METADATA
        USAGE
        END_RESPONSE
        INTERRUPT
        END_EVENT
    }

    class ConversationCommand {
        <<enumeration>>
        Default
        General
        Notes
        Online
        Webpage
        Code
        Image
        Diagram
        Research
        Operator
        AutomatedTask
    }

    ChatMessageModel --> Intent
    ResearchIteration --> ToolCall
    ResearchIteration --> OperatorRun
    OperatorRun --> AgentMessage
```

### 8.2 ChatMessageModel 详解

`ChatMessageModel` 是对话持久化的核心数据结构，每条消息包含：

| 字段 | 类型 | 说明 |
|------|------|------|
| `by` | `"you"` / `"khoj"` | 消息发送者 |
| `message` | `str` / `list` | 消息内容（文本或结构化内容） |
| `context` | `list` | 笔记搜索引用结果 |
| `onlineContext` | `dict` | 在线搜索结果 |
| `codeContext` | `dict` | 代码执行结果 |
| `operatorContext` | `list[dict]` | 浏览器操作结果 |
| `researchContext` | `list[dict]` | 研究迭代结果 |
| `intent` | `Intent` | 意图信息（类型、推断查询） |
| `queryFiles` | `list[FileAttachment]` | 用户附加文件 |
| `trainOfThought` | `list` | 推理过程记录 |
| `turnId` | `str` | 对话轮次 ID |
| `images` | `list[str]` | 生成的图像 URL |

### 8.3 ResponseWithThought 详解

`ResponseWithThought` 是 LLM 流式输出的统一封装：

| 字段 | 类型 | 说明 |
|------|------|------|
| `text` | `str` | 模型生成的文本内容（流式增量） |
| `thought` | `str` | 模型的推理/思考过程（流式增量） |
| `raw_content` | `list` | 原始响应内容（用于缓存命中） |

在流式模式下，每个 chunk 只包含 `text` 或 `thought` 其中之一，由 `event_generator` 分别路由为 `MESSAGE` 或 `THOUGHT` 事件。

### 8.4 ResearchIteration 详解

`ResearchIteration` 记录研究模式的每次迭代：

| 字段 | 类型 | 说明 |
|------|------|------|
| `query` | `ToolCall` / `str` | 工具调用描述或文本指令 |
| `context` | `list` | 本次迭代的笔记搜索结果 |
| `onlineContext` | `dict` | 本次迭代的在线搜索结果 |
| `codeContext` | `dict` | 本次迭代的代码执行结果 |
| `operatorContext` | `OperatorRun` | 本次迭代的浏览器操作结果 |
| `summarizedResult` | `str` | 工具执行结果的摘要 |
| `warning` | `str` | 警告信息 |
| `raw_response` | `list` | 原始 LLM 响应（用于多轮对话缓存） |

### 8.5 OperatorRun 详解

`OperatorRun` 记录浏览器操作的完整轨迹：

| 字段 | 类型 | 说明 |
|------|------|------|
| `query` | `str` | 操作任务描述 |
| `trajectory` | `list[AgentMessage]` | 操作轨迹（user/assistant/environment 消息序列） |
| `response` | `str` | 操作结果摘要 |
| `webpages` | `list[dict]` | 访问过的网页列表 |

---

## 9. 提示词体系

### 9.1 提示词分类

```mermaid
mindmap
  root((提示词体系))
    人格
      personality (默认)
      custom_personality (Agent 自定义)
      gemini_verbose_language_personality
    工具选择
      pick_relevant_tools
      extract_questions_system_prompt
      online_search_conversation_subqueries
      infer_webpages_to_read
    上下文注入
      notes_conversation
      online_search_conversation
      code_executed_context
      operator_execution_context
      generated_assets_context
    输出生成
      enhance_image_system_message
      improve_mermaid_js_diagram_description_prompt
      mermaid_js_diagram_generation_prompt
      python_code_generation_prompt
    研究
      plan_function_execution
    记忆
      extract_facts_from_query
    自动化
      crontime_prompt
      subject_generation
      to_notify_or_not
      automation_format_prompt
    安全
      personality_prompt_safety_expert
      personality_prompt_safety_expert_lax
    对话管理
      conversation_title_generation
      help_message
```

### 9.2 系统提示词组装

```mermaid
flowchart TD
    START[系统提示词构建] --> A{有 Agent?}
    A -->|是| B[custom_personality<br/>name + bio]
    A -->|否| C[personality<br/>Khoj 默认人格]

    B --> D{Google 模型?}
    C --> D
    D -->|是| E[+ gemini_verbose_language_personality]
    D -->|否| F[继续]

    E --> G{有位置信息?}
    F --> G
    G -->|是| H[+ user_location]
    G -->|否| I[继续]

    H --> J{有用户名?}
    I --> J
    J -->|是| K[+ user_name]
    J -->|否| L[完成]
    K --> L
```

---

## 10. 推理模型支持

### 10.1 各提供商推理模型配置

```mermaid
graph TB
    subgraph "OpenAI 推理模型"
        OR1[o1/o3/o4-mini/gpt-5] --> OR2["reasoning_effort: low/medium"]
        OR3[deepthought=true] --> OR4["reasoning_effort: medium"]
        OR5[temperature=1 固定]
    end

    subgraph "Anthropic 推理模型"
        AR1[claude-3-7/sonnet-4/opus-4] --> AR2["thinking: enabled"]
        AR3[deepthought=true] --> AR4["budget_tokens: 12000"]
        AR5[temperature=1 固定]
        AR6["betas: context-management"]
    end

    subgraph "Gemini 推理模型"
        GR1[gemini-2.5] --> GR2["thinking_budget: 512"]
        GR3[gemini-3] --> GR4["thinking_level: LOW/HIGH"]
        GR5[include_thoughts: true]
    end
```

### 10.2 思维过程提取方式

| 提供商 | 思维提取方式 | 处理器 |
|--------|-------------|--------|
| OpenAI (o系列) | `reasoning_content` 字段 | `astream_thought_processor` |
| OpenAI (Responses API) | `ResponseReasoningItem` | 直接提取 |
| DeepSeek/Qwen | `<think...</think` 标签 | `in_stream_thought_processor` |
| Anthropic | `thinking_delta` 事件 | 直接提取 |
| Gemini | `part.thought` 标记 | 直接提取 |

---

## 11. 错误处理与重试

### 11.1 重试策略

```mermaid
flowchart TD
    CALL[模型调用] --> RESULT{调用结果}
    RESULT -->|成功| OK[返回响应]
    RESULT -->|失败| ERR{错误类型}

    ERR -->|RateLimitError| RETRY1[指数退避重试]
    ERR -->|APITimeoutError| RETRY1
    ERR -->|InternalServerError| RETRY1
    ERR -->|ValueError 空响应| RETRY1
    ERR -->|NetworkError| RETRY1
    ERR -->|其他错误| RAISE[直接抛出]

    RETRY1 -->|重试成功| OK
    RETRY1 -->|重试耗尽| FALLBACK[尝试备选模型]
    FALLBACK -->|成功| OK
    FALLBACK -->|全部失败| FINAL[RetryableModelError]
```

### 11.2 各提供商重试配置

| 提供商 | 重试次数 | 等待策略 | 适用异常 |
|--------|---------|---------|---------|
| OpenAI (同步) | 2 | 随机指数退避 (1-10s) | Timeout, RateLimit, InternalServer, ValueError |
| OpenAI (异步) | 3 | 指数退避 (4-10s) | 同上 |
| Anthropic | 2 | 随机指数退避 (1-10s) | APIError, RateLimitError |
| Gemini | 2-3 | Gemini retryDelay + 指数退避 | 429, 502, 503, 504, ValueError |

---

## 12. 对话持久化

### 12.1 保存流程

```mermaid
sequenceDiagram
    participant EG as event_generator
    participant SCL as save_to_conversation_log
    participant MTL as message_to_log
    participant DB as ConversationAdapters
    participant MEM as ai_update_memories
    participant TRACE as commit_conversation_trace

    EG->>SCL: 异步调用 (asyncio.create_task)
    SCL->>SCL: 构建 user_message_metadata
    Note over SCL: created, images, turnId, queryFiles
    SCL->>SCL: 构建 khoj_message_metadata
    Note over SCL: context, intent, onlineContext,<br/>codeContext, operatorContext,<br/>researchContext, trainOfThought,<br/>images, mermaidjsDiagram

    SCL->>MTL: message_to_log()
    MTL->>MTL: 验证 ChatMessageModel
    MTL-->>SCL: [human_message, khoj_message]

    SCL->>DB: save_conversation()
    DB-->>SCL: db_conversation

    SCL->>MEM: ai_update_memories()
    Note over MEM: 非自动化任务时更新记忆

    SCL->>TRACE: commit_conversation_trace()
    Note over TRACE: 仅在 promptrace 启用时
```

### 12.2 中断恢复机制

当用户中断对话时，系统会保存部分上下文。下次对话时，`event_generator` 会检测并恢复中断消息的上下文：

```mermaid
flowchart TD
    START[对话开始] --> CHECK{有中断消息?}
    CHECK -->|是| RESTORE[恢复上下文]
    CHECK -->|否| NORMAL[正常流程]

    RESTORE --> R1[恢复 online_results]
    R1 --> R2[恢复 code_results]
    R2 --> R3[恢复 compiled_references]
    R3 --> R4[恢复 research_results]
    R4 --> R5[恢复 operator_results]
    R5 --> R6[恢复 train_of_thought]
    R6 --> NORMAL
```
