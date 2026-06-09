# OpenSquilla Engine 模块深度分析

## 一、模块概述

`opensquilla.engine` 是 OpenSquilla 的核心运行时引擎，负责管理对话轮次（Turn）的完整生命周期，包括上下文构建、模型路由、工具调度、流式响应、压缩与回退等关键流程。

### 目录结构

| 子目录/文件 | 职责 |
|---|---|
| `pipeline.py` | Pipeline 编排：按顺序执行 steps |
| `steps/` | Pipeline 步骤集合（路由、模型解析、技能过滤等） |
| `hooks/` | 钩子机制（默认钩子、类型定义） |
| `turn_runner/` | TurnRunner 核心：输入阶段、附件阶段、上下文、结果 |
| `agent.py` | Agent 管理（251KB，核心大文件） |
| `runtime.py` | 运行时入口（231KB，核心大文件） |
| `context.py` | TurnContext 上下文对象 |
| `context_budget.py` | 上下文预算管理 |
| `compaction_control.py` | 上下文压缩控制 |
| `fallback.py` | 模型回退策略 |
| `history.py` | 对话历史管理 |
| `session_lock.py` | 会话级锁 |
| `subagent.py` | 子代理支持 |
| `stream_wrappers.py` | 流式响应包装器 |
| `tool_result_store.py` | 工具结果存储 |
| `usage.py` | Token 用量统计 |
| `outcome.py` | Turn 结果定义 |
| `pricing.py` | 定价计算 |
| `thinking.py` | 推理模式控制 |
| `turn_control.py` | Turn 控制流 |
| `turn_policy.py` | Turn 策略 |

## 二、核心类/函数列表

| 类/函数 | 文件 | 职责 |
|---|---|---|
| `TurnRunner` | `turn_runner/harness.py` | 对话轮次执行器，协调 pipeline、工具循环、流式输出 |
| `TurnContext` | `context.py` | 每轮对话的上下文对象，携带元数据、路由决策、工具注册表等 |
| `Pipeline` | `pipeline.py` | 步骤编排器，按顺序执行 steps |
| `apply_squilla_router()` | `steps/squilla_router.py` | Pipeline Step 2：模型路由决策 |
| `resolve_model()` | `steps/resolve_model.py` | Pipeline Step：模型解析 |
| `apply_skills_filter()` | `steps/skills_filter.py` | Pipeline Step：技能过滤与注入 |
| `inject_platform_hint()` | `steps/inject_platform_hint.py` | Pipeline Step：平台提示注入 |
| `inject_time_prefix()` | `steps/inject_time_prefix.py` | Pipeline Step：时间前缀注入 |
| `apply_meta_resolution()` | `steps/meta_resolution.py` | Pipeline Step：Meta-Skill 解析 |
| `apply_prompt_cache()` | `steps/prompt_cache.py` | Pipeline Step：提示缓存策略 |
| `InputStage` | `turn_runner/input_stage.py` | 输入预处理阶段 |
| `AttachmentStage` | `turn_runner/attachment_stage.py` | 附件处理阶段 |
| `TurnOutcome` | `turn_runner/outcome.py` | 轮次结果封装 |

## 三、TurnRunner 工作流程

```mermaid
sequenceDiagram
    participant Caller as 调用方
    participant TR as TurnRunner
    participant Pipeline as Pipeline
    participant Steps as Steps (路由/模型/技能)
    participant Provider as LLM Provider
    participant Tools as ToolRegistry
    participant Stream as StreamWrapper

    Caller->>TR: run(message, session_key, ...)
    TR->>TR: 构建 TurnContext
    TR->>Pipeline: execute(ctx)
    Pipeline->>Steps: 1. inject_platform_hint
    Pipeline->>Steps: 2. apply_squilla_router (路由决策)
    Pipeline->>Steps: 3. resolve_model (模型解析)
    Pipeline->>Steps: 4. apply_skills_filter (技能注入)
    Pipeline->>Steps: 5. inject_time_prefix
    Pipeline->>Steps: 6. apply_prompt_cache
    Pipeline->>Steps: 7. apply_meta_resolution
    Steps-->>Pipeline: ctx (含路由决策+模型+技能)
    Pipeline-->>TR: ctx

    loop 工具循环 (max N 轮)
        TR->>Provider: chat(messages, tools, stream=True)
        Provider-->>Stream: 流式响应
        Stream-->>TR: text_delta / tool_call / usage

        alt 有工具调用
            TR->>Tools: execute(tool_call)
            Tools-->>TR: tool_result
            TR->>TR: 追加到 messages
        else 无工具调用
            TR->>TR: 退出循环
        end
    end

    TR->>TR: 计算 usage + pricing
    TR->>TR: 构建 TurnOutcome
    TR-->>Caller: Stream[TurnEvent]
```

## 四、Pipeline 步骤流程

```mermaid
flowchart TD
    A[TurnContext 输入] --> B[Step 1: inject_platform_hint]
    B --> C[Step 2: apply_squilla_router]
    C --> D[Step 3: resolve_model]
    D --> E[Step 4: apply_skills_filter]
    E --> F[Step 5: inject_time_prefix]
    F --> G[Step 6: apply_prompt_cache]
    G --> H[Step 7: apply_meta_resolution]
    H --> I[Step 8: reasoning_hint_observer]
    I --> J[TurnContext 输出 - 进入 LLM 调用]
```

## 五、上下文管理机制

```mermaid
flowchart TD
    A[对话消息累积] --> B{上下文是否超预算?}
    B -- 否 --> C[正常发送给 LLM]
    B -- 是 --> D{压缩策略}
    D --> E[AUTO_SUMMARIZE: LLM 摘要压缩]
    D --> F[HARD_TRUNCATE: 硬截断旧消息]
    D --> G[REFUSE: 拒绝请求]
    E --> H[保留摘要 + 最近消息]
    F --> H
    H --> I[检查 KV Cache 连续性]
    I --> J{压缩是否破坏缓存前缀?}
    J -- 是 --> K[记录 cache_break 事件]
    J -- 否 --> L[正常继续]
    K --> L
```

## 六、会话锁机制

```mermaid
sequenceDiagram
    participant Req1 as 请求1
    participant Req2 as 请求2
    participant Lock as SessionLock
    participant Runner as TurnRunner

    Req1->>Lock: acquire(session_key)
    Lock-->>Req1: 获得锁
    Req1->>Runner: run_turn(...)

    Req2->>Lock: acquire(session_key)
    Lock-->>Req2: 等待...

    Runner-->>Req1: 完成
    Req1->>Lock: release(session_key)
    Lock-->>Req2: 获得锁
    Req2->>Runner: run_turn(...)
```

## 七、Agent 与 Subagent

```mermaid
flowchart TD
    A[主 Agent] --> B{需要委派子任务?}
    B -- 是 --> C[创建 Subagent]
    C --> D[设置深度边界 max_depth]
    D --> E[子 Agent 独立 TurnRunner 循环]
    E --> F{子任务完成?}
    F -- 是 --> G[返回结果给主 Agent]
    F -- 需要再委派 --> H{depth < max_depth?}
    H -- 是 --> C
    H -- 否 --> I[返回部分结果]
    B -- 否 --> J[直接执行]
```

## 八、设计模式

| 模式 | 应用 |
|---|---|
| **Pipeline 模式** | Steps 按顺序执行，每个步骤修改 TurnContext |
| **策略模式** | 路由策略（V4Phase3 / Unavailable）、压缩策略、回退策略 |
| **观察者模式** | Hooks 机制，步骤执行前后触发回调 |
| **锁模式** | SessionLock 保证同一会话的 Turn 串行执行 |
| **流式迭代器** | TurnRunner 返回 Stream[TurnEvent]，支持实时推送 |
| **递归委派** | Subagent 通过深度边界控制递归 |
