# Letta Tool/Function 系统模块设计文档

## 1. 模块概述

Tool/Function 系统是 Letta Agent 框架的核心子系统，负责定义、注册、发现、调度和执行 Agent 可调用的工具函数。该系统使 LLM 能够通过结构化的函数调用（Function Calling）与外部世界交互，包括操作内存、搜索对话历史、调用 MCP 外部工具、执行沙箱代码等。

### 核心职责

| 职责 | 说明 |
|------|------|
| **Tool 定义与 Schema 生成** | 从 Python 源码/Pydantic 模型/MCP 协议自动生成 OpenAI 兼容的 JSON Schema |
| **Tool 类型体系** | 区分内置核心工具、内存工具、文件工具、多 Agent 工具、MCP 外部工具、自定义沙箱工具 |
| **Tool 注册与发现** | 通过模块加载、MCP 服务发现等方式注册工具，支持语义搜索 |
| **Tool 执行调度** | 基于 ToolType 的工厂模式分派到不同执行器（进程内/沙箱/MCP 客户端） |
| **Tool 规则约束** | 通过 ToolRule 体系控制工具调用顺序、终止条件、审批流程等 |
| **Schema 验证** | 验证 JSON Schema 是否符合 OpenAI strict mode 要求 |

### 目录结构

```
letta/
├── functions/                    # Tool 函数核心定义
│   ├── functions.py              # 源码解析、Schema 推导、模块加载
│   ├── helpers.py                # MCP/LangChain 包装器、Pydantic 模型转换
│   ├── schema_generator.py       # 核心 Schema 生成器（Python→JSON Schema）
│   ├── schema_validator.py       # OpenAI strict mode Schema 验证
│   ├── types.py                  # Pydantic 请求模型（SearchTask, FileOpenRequest）
│   ├── interface.py              # 多 Agent 消息接口
│   ├── function_sets/            # 内置工具函数集
│   │   ├── base.py               # 内存/消息/检索核心工具
│   │   ├── builtin.py            # 代码执行/Web 搜索/网页抓取
│   │   ├── files.py              # 文件操作工具
│   │   ├── multi_agent.py        # 多 Agent 通信工具
│   │   └── voice.py              # 语音/睡眠时工具
│   └── mcp_client/               # MCP 客户端类型
│       ├── types.py              # MCPTool、服务器配置（SSE/STDIO/StreamableHTTP）
│       └── exceptions.py         # MCP 超时异常
├── schemas/
│   ├── tool.py                   # Tool/ToolCreate/ToolUpdate 数据模型
│   └── tool_rule.py              # ToolRule 类型体系
├── services/
│   └── tool_executor/            # Tool 执行器
│       ├── tool_executor_base.py # 抽象基类
│       ├── tool_execution_manager.py  # 执行管理器 + 工厂
│       ├── core_tool_executor.py      # 核心工具执行器
│       ├── builtin_tool_executor.py   # 内置工具执行器
│       ├── files_tool_executor.py     # 文件工具执行器
│       ├── mcp_tool_executor.py       # MCP 工具执行器
│       └── sandbox_tool_executor.py   # 沙箱工具执行器
└── helpers/
    ├── tool_execution_helper.py  # Schema strict mode / pre-execution message
    ├── tool_helpers.py           # Tool 哈希 / Modal 函数名生成
    └── tool_rule_solver.py       # Tool 规则求解器
```

---

## 2. Tool 类型体系

### 2.1 类型继承关系

```mermaid
classDiagram
    direction TB

    class LettaBase {
        <<Pydantic BaseModel>>
        +generate_id_field() str
    }

    class BaseTool {
        <<Pydantic Model>>
        +__id_prefix__ = "tool"
    }

    class Tool {
        +id: str
        +tool_type: ToolType
        +description: Optional~str~
        +source_type: Optional~str~
        +name: Optional~str~
        +tags: List~str~
        +source_code: Optional~str~
        +json_schema: Optional~Dict~
        +return_char_limit: int
        +pip_requirements: Optional~list~
        +npm_requirements: Optional~list~
        +default_requires_approval: Optional~bool~
        +enable_parallel_execution: Optional~bool~
        +metadata_: Optional~Dict~
        +refresh_source_code_and_json_schema()
    }

    class ToolCreate {
        +source_code: str
        +source_type: str
        +json_schema: Optional~Dict~
        +from_mcp(mcp_server_name, mcp_tool) ToolCreate
        +validate_typescript_requires_schema()
    }

    class ToolUpdate {
        +source_code: Optional~str~
        +source_type: Optional~str~
    }

    class ToolRunFromSource {
        +source_code: str
        +args: Dict
        +env_vars: Dict
    }

    class ToolSearchRequest {
        +query: Optional~str~
        +search_mode: Literal
        +tool_types: Optional~List~
        +tags: Optional~List~
    }

    class ToolSearchResult {
        +tool: Tool
        +embedded_text: Optional~str~
        +combined_score: float
    }

    LettaBase <|-- BaseTool
    BaseTool <|-- Tool
    LettaBase <|-- ToolCreate
    LettaBase <|-- ToolUpdate
    LettaBase <|-- ToolRunFromSource
    LettaBase <|-- ToolSearchRequest
    LettaBase <|-- ToolSearchResult
```

### 2.2 ToolType 枚举

```mermaid
classDiagram
    class ToolType {
        <<enumeration>>
        CUSTOM
        LETTA_CORE
        LETTA_MEMORY_CORE
        LETTA_MULTI_AGENT_CORE
        LETTA_SLEEPTIME_CORE
        LETTA_VOICE_SLEEPTIME_CORE
        LETTA_BUILTIN
        LETTA_FILES_CORE
        EXTERNAL_MCP
        EXTERNAL_LANGCHAIN ~deprecated~
        EXTERNAL_COMPOSIO ~deprecated~
    }
```

### 2.3 ToolRule 类型体系

```mermaid
classDiagram
    direction TB

    class BaseToolRule {
        <<abstract>>
        +tool_name: str
        +type: ToolRuleType
        +prompt_template: Optional~str~
        +get_valid_tools(tool_call_history, available_tools, last_function_response) Set~str~
        +render_prompt() str|None
        +requires_force_tool_call: bool
    }

    class InitToolRule {
        +type = run_first
        +args: Optional~Dict~
        +requires_force_tool_call = true
    }

    class ChildToolRule {
        +type = constrain_child_tools
        +children: List~str~
        +child_arg_nodes: Optional~List~ToolCallNode~~
        +requires_force_tool_call = true
        +get_child_names() List~str~
        +get_child_args_map() Dict
    }

    class ParentToolRule {
        +type = parent_last_tool
        +children: List~str~
        +requires_force_tool_call = true
    }

    class ConditionalToolRule {
        +type = conditional
        +default_child: Optional~str~
        +child_output_mapping: Dict
        +require_output_mapping: bool
        +requires_force_tool_call = true
    }

    class TerminalToolRule {
        +type = exit_loop
    }

    class ContinueToolRule {
        +type = continue_loop
    }

    class RequiredBeforeExitToolRule {
        +type = required_before_exit
    }

    class MaxCountPerStepToolRule {
        +type = max_count_per_step
        +max_count_limit: int
    }

    class RequiresApprovalToolRule {
        +type = requires_approval
    }

    class ToolCallNode {
        +name: str
        +args: Optional~Dict~
    }

    BaseToolRule <|-- InitToolRule
    BaseToolRule <|-- ChildToolRule
    BaseToolRule <|-- ParentToolRule
    BaseToolRule <|-- ConditionalToolRule
    BaseToolRule <|-- TerminalToolRule
    BaseToolRule <|-- ContinueToolRule
    BaseToolRule <|-- RequiredBeforeExitToolRule
    BaseToolRule <|-- MaxCountPerStepToolRule
    BaseToolRule <|-- RequiresApprovalToolRule

    ChildToolRule *-- ToolCallNode : child_arg_nodes
```

---

## 3. Tool 注册与发现

### 3.1 注册流程

```mermaid
flowchart TD
    A[Tool 注册入口] --> B{注册方式}

    B --> C[内置工具注册]
    B --> D[自定义工具注册]
    B --> E[MCP 工具注册]

    C --> C1[function_sets 模块]
    C1 --> C2[load_function_set]
    C2 --> C3[遍历模块公开函数]
    C3 --> C4[generate_schema 生成 JSON Schema]
    C4 --> C5[存入 Tool 数据库]

    D --> D1[用户提供 source_code]
    D1 --> D2{source_type?}
    D2 -->|python| D3[derive_openai_json_schema]
    D2 -->|typescript| D4[要求显式 json_schema]
    D3 --> D5[AST 静态解析源码]
    D5 --> D6[构建 MockFunction]
    D6 --> D7[generate_schema]
    D7 --> D8[存入 Tool 数据库]
    D4 --> D8

    E --> E1[MCP Server 连接]
    E1 --> E2[list_tools 获取 MCPTool 列表]
    E2 --> E3[generate_tool_schema_for_mcp]
    E3 --> E4[generate_mcp_tool_wrapper]
    E4 --> E5[ToolCreate.from_mcp]
    E5 --> E6[存入 Tool 数据库]

    subgraph Schema 生成核心
        SG1[generate_schema]
        SG2[validate_google_style_docstring]
        SG3[type_to_json_schema_type]
        SG4[pydantic_model_to_json_schema]
        SG2 --> SG1
        SG3 --> SG1
        SG4 --> SG1
    end

    D7 -.-> SG1
    C4 -.-> SG1
    E3 -.-> SG1
```

### 3.2 Schema 生成详解

```mermaid
flowchart LR
    subgraph Python 函数 Schema 生成
        A1[Python 函数对象] --> A2[inspect.signature]
        A2 --> A3[docstring_parser.parse]
        A3 --> A4[遍历参数]
        A4 --> A5{参数类型?}
        A5 -->|Pydantic Model| A6[pydantic_model_to_json_schema]
        A5 -->|基础类型| A7[type_to_json_schema_type]
        A5 -->|Optional/Union| A8[解包内部类型]
        A5 -->|List/Dict| A9[递归处理 items/values]
        A6 --> A10[组装 OpenAI JSON Schema]
        A7 --> A10
        A8 --> A10
        A9 --> A10
    end

    subgraph MCP Schema 生成
        B1[MCPTool.inputSchema] --> B2[normalize_mcp_schema]
        B2 --> B3[inline_ref 内联 $ref]
        B3 --> B4[deduplicate_anyof]
        B4 --> B5[append_heartbeat 参数]
        B5 --> B6{strict mode?}
        B6 -->|yes| B7[heal optional fields]
        B6 -->|no| B8[返回标准 Schema]
        B7 --> B8
    end

    subgraph 源码静态解析
        C1[源码字符串] --> C2[ast.parse]
        C2 --> C3[_build_imports_map]
        C3 --> C4[_extract_pydantic_classes]
        C4 --> C5[_parse_function_from_source]
        C5 --> C6[MockFunction]
        C6 --> C7[generate_schema]
    end
```

### 3.3 Tool 发现机制

```mermaid
flowchart TD
    A[Tool 发现请求] --> B{发现方式}

    B --> C[按 Agent 加载]
    C --> C1[agent_state.tools]
    C1 --> C2[按 tool_type 分组]
    C2 --> C3[内置工具: 动态生成 json_schema]
    C3 --> C4[自定义工具: 读取已存储 schema]

    B --> D[语义搜索]
    D --> D1[ToolSearchRequest]
    D1 --> D2{search_mode}
    D2 -->|vector| D3[向量搜索]
    D2 -->|fts| D4[全文搜索]
    D2 -->|hybrid| D5[混合搜索]
    D3 --> D6[ToolSearchResult 列表]
    D4 --> D6
    D5 --> D6

    B --> E[MCP 工具发现]
    E --> E1[MCPManager.list_mcp_server_tools]
    E1 --> E2[连接 MCP Server]
    E2 --> E3[list_tools]
    E3 --> E4[validate_complete_json_schema]
    E4 --> E5[返回 MCPTool 列表]
```

---

## 4. Tool 执行流程

### 4.1 完整执行时序图

```mermaid
sequenceDiagram
    participant Agent as Agent Loop
    participant LLM as LLM (OpenAI等)
    participant TEM as ToolExecutionManager
    participant TEF as ToolExecutorFactory
    participant TE as ToolExecutor
    participant TRS as ToolRulesSolver
    participant DB as 数据库/服务

    Note over Agent,TRS: Step 1: 准备工具列表
    Agent->>TRS: get_allowed_tool_names(available_tools)
    TRS-->>Agent: allowed_tools + prefilled_args

    Agent->>TRS: should_force_tool_call()
    TRS-->>Agent: force (bool)

    Note over Agent,LLM: Step 2: LLM 调用
    Agent->>LLM: chat_completion(tools=allowed_tools, tool_choice=force?"required":"auto")
    LLM-->>Agent: tool_calls[{name, arguments}]

    Note over Agent,TRS: Step 3: 规则校验
    Agent->>TRS: is_terminal_tool(tool_name)
    Agent->>TRS: is_requires_approval_tool(tool_name)
    Agent->>TRS: register_tool_call(tool_name)

    Note over Agent,TE: Step 4: 执行工具
    Agent->>TEM: execute_tool_async(function_name, function_args, tool)
    TEM->>TEF: get_executor(tool.tool_type)
    TEF-->>TEM: executor instance

    alt tool_type = LETTA_CORE / LETTA_MEMORY_CORE
        TEM->>TE: LettaCoreToolExecutor.execute()
        TE->>DB: 直接调用 AgentManager/BlockManager/MessageManager
        DB-->>TE: 执行结果
    else tool_type = LETTA_BUILTIN
        TEM->>TE: LettaBuiltinToolExecutor.execute()
        TE->>DB: 执行内置工具逻辑
    else tool_type = LETTA_FILES_CORE
        TEM->>TE: LettaFileToolExecutor.execute()
        TE->>DB: 文件操作
    else tool_type = EXTERNAL_MCP
        TEM->>TE: ExternalMCPToolExecutor.execute()
        TE->>DB: MCPManager.execute_mcp_server_tool()
        DB-->>TE: MCP 远程调用结果
    else tool_type = CUSTOM / MULTI_AGENT
        TEM->>TE: SandboxToolExecutor.execute()
        TE->>DB: 沙箱执行（Modal/E2B/Local）
        DB-->>TE: 沙箱执行结果
    end

    TE-->>TEM: ToolExecutionResult

    Note over TEM: Step 5: 结果截断
    TEM->>TEM: len(return_str) > return_char_limit?
    TEM-->>Agent: ToolExecutionResult{status, func_return}

    Note over Agent,TRS: Step 6: 循环控制
    Agent->>TRS: is_terminal_tool?
    alt 终止工具
        Agent->>Agent: 退出 Agent Loop
    else 继续工具
        Agent->>Agent: 继续下一轮 LLM 调用
    end
```

### 4.2 执行器分派策略

```mermaid
flowchart LR
    subgraph ToolExecutorFactory
        direction TB
        F[tool_type] --> M{匹配 ToolType}
        M -->|LETTA_CORE / LETTA_MEMORY_CORE / LETTA_SLEEPTIME_CORE| C1[LettaCoreToolExecutor]
        M -->|LETTA_BUILTIN| C2[LettaBuiltinToolExecutor]
        M -->|LETTA_FILES_CORE| C3[LettaFileToolExecutor]
        M -->|EXTERNAL_MCP| C4[ExternalMCPToolExecutor]
        M -->|LETTA_MULTI_AGENT_CORE / CUSTOM| C5[SandboxToolExecutor]
        M -->|默认 fallback| C5
    end
```

### 4.3 沙箱执行策略

```mermaid
flowchart TD
    A[SandboxToolExecutor.execute] --> B{tool.metadata_.sandbox == modal?}
    B -->|yes| C{Modal 已配置?}
    C -->|yes| D[AsyncToolSandboxModal.run]
    D -->|成功| E[返回结果]
    D -->|失败| F[降级到 E2B/Local]
    C -->|no| F
    B -->|no| F

    F --> G{sandbox_type 配置}
    G -->|E2B| H[AsyncToolSandboxE2B.run]
    G -->|Local| I[AsyncToolSandboxLocal.run]
    H --> E
    I --> E
```

---

## 5. MCP 工具集成

### 5.1 MCP 连接、发现与执行流程

```mermaid
sequenceDiagram
    participant User as 用户/API
    participant API as MCP API 端点
    participant Mgr as MCPManager
    participant Client as MCP Client
    participant Server as MCP Server (远程)
    participant TM as ToolManager
    participant DB as 数据库

    Note over User,Server: Phase 1: MCP Server 注册
    User->>API: 创建 MCP Server (SSE/STDIO/StreamableHTTP)
    API->>Mgr: add_mcp_server(server_config)
    Mgr->>DB: 存储 MCPServer 记录

    Note over User,Server: Phase 2: 工具发现
    User->>API: list_mcp_server_tools(server_name)
    API->>Mgr: list_mcp_server_tools(server_name, actor)
    Mgr->>Mgr: get_mcp_client(server_config)
    Mgr->>Client: 创建对应类型客户端
    Client->>Server: connect_to_server()
    Server-->>Client: 连接建立
    Client->>Server: list_tools()
    Server-->>Client: Tool[] (MCP 协议格式)
    Client-->>Mgr: MCPTool[]

    loop 每个 MCPTool
        Mgr->>Mgr: normalize_mcp_schema(inputSchema)
        Mgr->>Mgr: validate_complete_json_schema()
        Mgr->>Mgr: 设置 MCPToolHealth
    end

    Mgr-->>User: MCPTool[] (含 Schema 健康状态)

    Note over User,Server: Phase 3: 工具同步到 Agent
    User->>API: update_mcp_server(agent_id, server_name)
    API->>Mgr: resync_mcp_server(agent_id, server_name)
    Mgr->>Mgr: list_mcp_server_tools()
    loop 每个 MCPTool
        Mgr->>Mgr: ToolCreate.from_mcp(mcp_server_name, mcp_tool)
        Note right of Mgr: generate_tool_schema_for_mcp<br/>generate_mcp_tool_wrapper
        Mgr->>TM: create_or_update_tool(tool_create)
        TM->>DB: 存储 Tool 记录
    end
    Mgr->>DB: 关联 Tool 到 Agent

    Note over User,Server: Phase 4: 工具执行
    User->>API: Agent 消息 (触发 tool_call)
    API->>Mgr: execute_mcp_server_tool(server_name, tool_name, tool_args)
    Mgr->>Mgr: get_mcp_client(server_config)
    Mgr->>Client: connect_to_server()
    Client->>Server: call_tool(tool_name, tool_args)
    Server-->>Client: 执行结果
    Client-->>Mgr: function_response + success
    Mgr-->>API: 执行结果
```

### 5.2 MCP 服务器配置类型

```mermaid
classDiagram
    direction TB

    class BaseServerConfig {
        <<abstract>>
        +server_name: str
        +type: MCPServerType
        +is_templated_tool_variable(value) bool
        +get_tool_variable(value, env_vars) Optional~str~
        +resolve_custom_headers(headers, env_vars) Optional~Dict~
        +resolve_environment_variables(env_vars)*
    }

    class HTTPBasedServerConfig {
        +server_url: str
        +auth_header: Optional~str~
        +auth_token: Optional~str~
        +custom_headers: Optional~dict~
        +resolve_token() Optional~str~
        +_build_headers_dict() Optional~dict~
    }

    class SSEServerConfig {
        +type = SSE
        +to_dict() dict
    }

    class StreamableHTTPServerConfig {
        +type = STREAMABLE_HTTP
        +to_dict() dict
    }

    class StdioServerConfig {
        +type = STDIO
        +command: str
        +args: List~str~
        +env: Optional~dict~
        +to_dict() dict
    }

    class MCPServerType {
        <<enumeration>>
        SSE
        STDIO
        STREAMABLE_HTTP
    }

    BaseServerConfig <|-- HTTPBasedServerConfig
    BaseServerConfig <|-- StdioServerConfig
    HTTPBasedServerConfig <|-- SSEServerConfig
    HTTPBasedServerConfig <|-- StreamableHTTPServerConfig
```

### 5.3 MCP Schema 健康检查

```mermaid
flowchart TD
    A[MCPTool.inputSchema] --> B[normalize_mcp_schema]
    B --> B1[添加 additionalProperties: false]
    B --> B2[内联 $ref 引用]
    B --> B3[去重 anyOf 条目]

    B --> C[validate_complete_json_schema]
    C --> C1{验证结果}

    C1 -->|STRICT_COMPLIANT| D[可启用 OpenAI strict mode]
    C1 -->|NON_STRICT_ONLY| E[仅非 strict 模式可用]
    C1 -->|INVALID| F[Schema 无效, 需修复]

    D --> G[MCPToolHealth.status = STRICT_COMPLIANT]
    E --> H[MCPToolHealth.status = NON_STRICT_ONLY]
    F --> I[MCPToolHealth.status = INVALID]

    G --> J[enable_strict_mode 正常启用]
    H --> K[enable_strict_mode 跳过 strict]
    I --> L[记录错误日志]
```

---

## 6. Tool 规则系统

### 6.1 ToolRulesSolver 求解逻辑

```mermaid
flowchart TD
    A[get_allowed_tool_names] --> B{有 tool_call_history?}

    B -->|无历史| C{有 InitToolRule?}
    C -->|yes| D[allowed = InitToolRule.tool_name 列表]
    C -->|no| E[allowed = available_tools]

    B -->|有历史| F[遍历 child_based_tool_rules + parent_tool_rules]
    F --> G[每个 rule.get_valid_tools]
    G --> H[取所有 valid_tool_sets 的交集]
    H --> I[intersection & available_tools]
    I --> J{结果为空?}
    J -->|yes & error_on_empty| K[抛出 ValueError]
    J -->|no| L[allowed = 交集结果]

    D --> M[构建 prefilled_args 缓存]
    L --> M

    M --> N{InitToolRule 有 args?}
    N -->|yes| O[缓存 InitToolRule.args]
    N -->|no| P{ChildToolRule 父工具是最后调用?}
    P -->|yes| Q[缓存 child_arg_nodes 的 args]
    P -->|no| R[返回 allowed 列表]
    O --> R
    Q --> R
```

### 6.2 各规则类型行为

```mermaid
flowchart TD
    subgraph InitToolRule
        I1[无历史时] --> I2[仅允许 InitToolRule 指定的工具]
        I2 --> I3[强制 tool_choice = required]
        I3 --> I4[可选预填参数 args]
    end

    subgraph ChildToolRule
        C1[父工具最后被调用] --> C2[仅允许 children 列表中的工具]
        C1 --> C3[强制 tool_choice = required]
        C2 --> C4[可选 child_arg_nodes 预填参数]
    end

    subgraph ParentToolRule
        P1[父工具最后被调用] --> P2[仅允许 children 工具]
        P1 --> P3[父工具未被调用] --> P4[从可用工具中排除 children]
    end

    subgraph ConditionalToolRule
        CO1[工具最后被调用] --> CO2[解析 last_function_response]
        CO2 --> CO3[匹配 child_output_mapping]
        CO3 --> CO4[返回匹配的子工具]
        CO3 --> CO5[无匹配 → default_child 或全部]
    end

    subgraph TerminalToolRule
        T1[工具被调用] --> T2[标记为终止工具]
        T2 --> T3[Agent Loop 退出]
    end

    subgraph ContinueToolRule
        CR1[工具被调用] --> CR2[标记为继续工具]
        CR2 --> CR3[Agent Loop 继续]
    end

    subgraph MaxCountPerStepToolRule
        MC1[调用次数 < max_count] --> MC2[工具仍可用]
        MC1 --> MC3[调用次数 >= max_count] --> MC4[从可用工具中移除]
    end

    subgraph RequiredBeforeExitToolRule
        RB1[Agent Loop 退出前] --> RB2[检查是否已调用]
        RB2 --> RB3[未调用 → 阻止退出]
    end

    subgraph RequiresApprovalToolRule
        RA1[工具被调用] --> RA2[触发人工审批流程]
    end
```

### 6.3 规则在 Agent Loop 中的集成

```mermaid
flowchart TD
    A[Agent Loop 开始] --> B[ToolRulesSolver.get_allowed_tool_names]
    B --> C[构建 LLM 请求 tools 参数]
    C --> D[ToolRulesSolver.should_force_tool_call]
    D --> E[设置 tool_choice = required/auto]

    E --> F[LLM 返回 tool_calls]
    F --> G{tool_calls 为空?}
    G -->|yes| H{has_required_tools_been_called?}
    H -->|no| I[注入提示: 必须调用 Required 工具]
    I --> F
    H -->|yes| J[Agent Loop 退出]

    G -->|no| K[遍历 tool_calls]
    K --> L{is_requires_approval_tool?}
    L -->|yes| M[触发审批流程]
    M -->|拒绝| N[返回错误给 LLM]
    M -->|批准| O[执行工具]

    L -->|no| O
    O --> P[ToolExecutionManager.execute_tool_async]
    P --> Q[获取执行结果]

    Q --> R{is_terminal_tool?}
    R -->|yes| S[Agent Loop 退出]
    R -->|no| T{is_continue_tool?}
    T -->|yes| U[继续 Agent Loop → B]
    T -->|no| V[继续 Agent Loop → B]

    S --> W{has_required_tools_been_called?}
    W -->|no| X[阻止退出 → B]
    W -->|yes| Y[Agent Loop 结束]
```

---

## 7. 关键设计决策分析

### 7.1 AST 静态解析 vs 运行时反射

**决策**：自定义 Python 工具的 Schema 生成采用 AST 静态解析（`_parse_function_from_source`），而非运行时 `import` + `inspect`。

**原因**：
- **安全性**：避免执行用户提交的任意 Python 代码
- **沙箱隔离**：源码可能引用不可用的依赖，运行时 import 会失败
- **兼容性**：支持 Pydantic 模型定义在源码中的内联解析

**代价**：
- 复杂嵌套 Pydantic 模型的 forward reference 支持有限
- 仅支持基本 `Field()` 定义（description、Ellipsis）
- 需要维护 AST 解析逻辑的健壮性

### 7.2 内置工具的"声明+实现"分离

**决策**：内置工具（`function_sets/`）中的函数体通常为 `raise NotImplementedError`，实际实现在对应的 `ToolExecutor` 中。

**原因**：
- **Schema 生成**：函数签名和 docstring 用于自动生成 JSON Schema，供 LLM 理解工具能力
- **执行解耦**：实际执行逻辑需要访问服务层（AgentManager、BlockManager 等），不适合在函数体内直接实现
- **安全边界**：确保 Schema 生成过程不依赖任何运行时状态

**影响**：
- `generate_schema` 从函数签名提取参数信息
- `LettaCoreToolExecutor` 通过 `function_map` 分派到实际实现方法
- 修改工具行为需要同时更新声明（Schema）和实现（Executor）

### 7.3 MCP Schema 规范化与 Strict Mode 适配

**决策**：对 MCP 工具的 inputSchema 进行多层规范化处理（`normalize_mcp_schema` → `inline_ref` → `deduplicate_anyof` → `enable_strict_mode`）。

**原因**：
- MCP Server 返回的 Schema 质量参差不齐，可能缺少 `additionalProperties`、使用 `$ref` 引用等
- OpenAI strict mode 要求所有属性必须 `required`、`additionalProperties: false`
- 需要在保持语义正确性的前提下适配 strict mode

**策略**：
1. `normalize_mcp_schema`：添加 `additionalProperties: false`，解析 `$ref` 类型
2. `inline_ref`：递归内联所有 `$ref`，避免 OpenAI 不支持 `$defs`
3. `enable_strict_mode`：将可选字段加入 `required` 并添加 `null` 类型
4. `SchemaHealth` 分级：STRICT_COMPLIANT / NON_STRICT_ONLY / INVALID

### 7.4 ToolRule 的声明式约束模型

**决策**：采用声明式规则（ToolRule）而非命令式逻辑控制工具调用流程。

**原因**：
- **可序列化**：规则可持久化到数据库，随 Agent 配置存储
- **可组合**：多条规则通过集合交集自然组合，无需复杂的状态机
- **可审计**：每条规则可独立 `render_prompt()` 生成自然语言约束提示
- **可扩展**：新增规则类型只需继承 `BaseToolRule` 并实现 `get_valid_tools`

**规则组合语义**：
- `child_based_tool_rules` + `parent_tool_rules`：取所有规则 `get_valid_tools` 返回集合的**交集**
- `InitToolRule`：仅在无历史时生效，覆盖其他规则
- `TerminalToolRule` / `ContinueToolRule`：不限制可用工具，控制循环流程
- `RequiresApprovalToolRule`：不限制可用工具，触发审批流程

### 7.5 执行器工厂模式

**决策**：使用 `ToolExecutorFactory` 根据 `ToolType` 分派到不同的执行器实现。

**原因**：
- **隔离关注点**：每种工具类型有不同的执行环境和依赖
- **可测试性**：每种执行器可独立测试
- **可扩展**：新增工具类型只需注册新的执行器

**执行器映射**：

| ToolType | 执行器 | 执行环境 |
|----------|--------|----------|
| LETTA_CORE / LETTA_MEMORY_CORE / LETTA_SLEEPTIME_CORE | LettaCoreToolExecutor | 进程内，直接调用服务层 |
| LETTA_BUILTIN | LettaBuiltinToolExecutor | 进程内 |
| LETTA_FILES_CORE | LettaFileToolExecutor | 进程内 |
| EXTERNAL_MCP | ExternalMCPToolExecutor | 通过 MCP Client 远程调用 |
| CUSTOM / MULTI_AGENT | SandboxToolExecutor | Modal / E2B / Local 沙箱 |

### 7.6 沙箱执行的多层降级

**决策**：SandboxToolExecutor 支持 Modal → E2B → Local 三层降级策略。

**原因**：
- **云优先**：Modal 提供更好的隔离和弹性扩展
- **兼容性**：E2B 作为备选云沙箱
- **本地开发**：Local 沙箱方便开发和调试

**安全措施**：
- 沙箱执行前深拷贝 `agent_state`，移除 `tools` 和 `tool_rules` 防止嵌套执行
- 执行后验证内存完整性（`orig_memory_str == new_memory_str`）
- 凭证通过 `SandboxCredentialsService` 安全注入

### 7.7 MCP 工具的"占位源码"模式

**决策**：MCP 工具注册时存储一个抛出 `RuntimeError` 的占位函数作为 `source_code`，实际执行通过 MCP Client 远程调用。

**原因**：
- **统一存储**：所有工具在数据库中使用相同的 Schema，`source_code` 字段必须非空
- **执行路径分离**：MCP 工具不通过源码执行，而是通过 `ExternalMCPToolExecutor` 路由到 MCP Server
- **防误用**：占位函数的 `RuntimeError` 确保不会被意外直接执行

**标识机制**：MCP 工具通过 `tags` 中的 `mcp_tool:{server_name}` 标签关联到对应的 MCP Server。
