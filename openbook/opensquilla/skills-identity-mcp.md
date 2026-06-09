# OpenSquilla Skills, Identity & MCP 模块深度分析

## 一、Skills 系统

### 1.1 六层优先级架构

| 层级 | 枚举值 | 来源目录 | 说明 |
|---|---|---|---|
| 1 | EXTRA | 配置指定 | 最低优先级 |
| 2 | BUNDLED | `skills/bundled/` | 随产品发布 |
| 3 | MANAGED | `~/.opensquilla/skills/` | 社区安装 |
| 4 | PERSONAL | `~/.agents/skills/` | 用户个人 |
| 5 | PROJECT | `{workspace}/.agents/skills/` | 项目级 |
| 6 | WORKSPACE | `{workspace}/skills/` | 最高优先级 |

### 1.2 技能生命周期

```mermaid
flowchart TD
    A[启动/请求] --> B{快照缓存命中?}
    B -->|是| C[从 JSON 快照加载]
    B -->|否| D[扫描六层目录]
    D --> E[查找 SKILL.md]
    E --> F[安全检查 + 解析 frontmatter]
    F --> G[构建 SkillSpec]
    G --> H[按层优先级合并]
    H --> I[编译 meta_sop 技能]
    I --> J[保存快照缓存]
    J --> K[资格检查 Eligibility]
    K --> L[混合检索 HybridRetriever]
    L --> M[SkillInjector 注入系统提示]
```

### 1.3 核心类

| 类 | 职责 |
|---|---|
| `SkillSpec` | 技能元数据与内容完整数据类 |
| `SkillLoader` | 核心加载器：多层级扫描、快照缓存、frontmatter 解析 |
| `SkillInjector` | 系统提示注入器：全量/紧凑/自动模式 |
| `HybridRetriever` | 混合检索器：词法+语义融合排序 |
| `EligibilityContext` | 资格检查上下文 |

---

## 二、Meta-Skill 工作流程

```mermaid
flowchart TD
    A[用户消息] --> B[meta_resolution 步骤]
    B --> C{触发词匹配?}
    C -- 否 --> D[正常对话]
    C -- 是 --> E[MetaOrchestrator.iter_events]
    E --> F[scheduler.run_dag]
    F --> G[拓扑排序 steps]
    G --> H[并发执行无依赖步骤]

    H --> I{步骤类型}
    I -->|agent| J[启动子 Agent]
    I -->|llm_classify| K[受限 LLM 调用]
    I -->|llm_chat| L[开放 LLM 调用]
    I -->|tool_call| M[直接工具调用]
    I -->|skill_exec| N[CLI 入口点执行]
    I -->|user_input| O[暂停等待用户输入]

    J & K & L & M & N --> P{步骤成功?}
    O --> Q{用户已回复?}
    Q -- 否 --> R[抛出 MetaPaused]
    Q -- 是 --> P

    P -- 是 --> S[解锁下游依赖]
    P -- 否 --> T{有 on_failure?}
    T -- 是 --> U[故障转移到替代步骤]
    T -- 否 --> V[取消所有任务]

    S --> W{还有未完成步骤?}
    W -- 是 --> H
    W -- 否 --> X[生成最终 MetaResult]
```

### SOP 编译流程

```mermaid
flowchart LR
    A[SKILL.md body] --> B[_lex 词法分析]
    B --> C[_parse 语法解析]
    C --> D[_resolve 技能引用解析]
    D --> E[_emit DAG 生成]
    E --> F[composition_raw steps 列表]
```

---

## 三、Identity 系统

### 3.1 身份文件体系

| 文件 | 内容 |
|---|---|
| `SOUL.md` | 持久人格指导：语气、风格 |
| `IDENTITY.md` | 身份字段：name/emoji/creature/vibe |
| `AGENTS.md` | 项目规则与行为指令 |
| `TOOLS.md` | 工具使用指南 |
| `USER.md` | 用户身份/偏好 |
| `MEMORY.md` | 长期记忆 |
| `BOOTSTRAP.md` | 一次性引导指令 |

### 3.2 提示生成流程

```mermaid
sequenceDiagram
    participant Runtime as TurnRunner
    participant WSLoader as load_workspace_files_budgeted
    participant Parser as parse_soul/identity/agents
    participant Prompt as assemble_system_prompt
    participant Template as system_prompt.j2

    Runtime->>WSLoader: 加载工作区文件 (预算控制)
    WSLoader->>WSLoader: 注入扫描 + 逐文件截断
    Runtime->>Parser: 解析 SOUL/IDENTITY/AGENTS
    Parser-->>Runtime: 身份字段
    Runtime->>Prompt: assemble_system_prompt(profile, tools, skills)
    Prompt->>Template: Jinja2 渲染
    Template-->>Prompt: 系统提示基础部分(可缓存)
    Note over Runtime: 运行时追加易失性内容:<br/>记忆召回/日记/上下文
```

---

## 四、MCP 协议实现

### 4.1 架构

```mermaid
flowchart TD
    A[MCPServerConfig] --> B{transport}
    B -->|stdio| C[MCPStdioClient]
    B -->|sse| D[MCPSSEClient]
    C --> E[子进程 + Content-Length 帧]
    D --> F[HTTP POST + SSE 流]
    C & D --> G[discover_and_register]
    G --> H[initialize 握手]
    H --> I[tools/list 获取工具]
    I --> J[注册 mcp_ 前缀 ToolSpec]
    J --> K[ToolRegistry 可调用]
```

### 4.2 核心类

| 类 | 职责 |
|---|---|
| `MCPClient` | 抽象基类：connect/close/list_tools/call_tool |
| `MCPStdioClient` | Stdio 传输：子进程 + JSON-RPC |
| `MCPSSEClient` | SSE 传输：HTTP + SSE 流 |
| `discover_and_register()` | 连接 → 列举 → 注册到 ToolRegistry |

## 五、设计模式

| 模式 | 应用 |
|---|---|
| **分层覆盖** | Skills 六层目录优先级 |
| **快照缓存** | Skills JSON 快照加速冷启动 |
| **混合检索+降级** | 词法+语义双路，语义失败自动降级 |
| **DAG 并行调度** | Meta-Skill 步骤依赖图并行执行 |
| **故障转移** | Meta-Skill on_failure 替代步骤 |
| **暂停/恢复** | Meta-Skill user_input CAS 原子恢复 |
| **缓存分离** | Identity 可缓存基础 + 易失性后缀 |
| **注入防护** | Skills XML 转义；Identity 工作区扫描 |
| **策略模式** | MCP 传输层抽象 |
| **前缀命名空间** | MCP 工具 mcp_ 前缀 |
