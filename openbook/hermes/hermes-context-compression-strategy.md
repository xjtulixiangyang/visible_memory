# Hermes 上下文压缩策略详解

> **核心文件**: [context_compressor.py](file:///Users/lixiangyang/Desktop/代码/hermes-agent-main/hermes-agent-main/agent/context_compressor.py)  
> **基类定义**: [ContextEngine](file:///Users/lixiangyang/Desktop/代码/hermes-agent-main/hermes-agent-main/agent/context_engine.py) (抽象基类，支持插件替换)  
> **调用入口**: [run_agent.py `_compress_context()`](file:///Users/lixiangyang/Desktop/代码/hermes-agent-main/hermes-agent-main/run_agent.py#L7519-L7638)  
> **辅助客户端**: [auxiliary_client.py `call_llm()`](file:///Users/lixiangyang/Desktop/代码/hermes-agent-main/hermes-agent-main/agent/auxiliary_client.py)

---

## 目录

1. [设计目标与核心思想](#1-设计目标与核心思想)
2. [架构总览](#2-架构总览)
3. [压缩触发机制 — 6 个调用点](#3-压缩触发机制--6-个调用点)
4. [五阶段压缩算法详解](#4-五阶段压缩算法详解)
5. [工具输出预剪枝（Phase 1）](#5-工具输出预剪枝phase-1)
6. [边界确定策略（Phase 2）](#6-边界确定策略phase-2)
7. [结构化摘要生成（Phase 3）](#7-结构化摘要生成phase-3)
8. [消息重组与清理（Phase 4 & 5）](#8-消息重组与清理phase-4--5)
9. [辅助模型选择与回退链](#9-辅助模型选择与回退链)
10. [反抖动与安全保护](#10-反抖动与安全保护)
11. [聚焦主题压缩（Focus Topic）](#11-聚焦主题压缩focus-topic)
12. [会话切分与持久化](#12-会话切分与持久化)
13. [配置参数一览](#13-配置参数一览)
14. [完整时序图](#14-完整时序图)

---

## 1. 设计目标与核心思想

### 1.1 问题背景

Hermes Agent 的对话是**多轮工具调用循环**：用户发一条消息 → 模型可能连续执行 10~90 轮工具调用 → 每轮都产生 assistant message + tool result。对于长任务（如大型重构、多文件调试），消息列表会迅速膨胀到超出模型的上下文窗口。

### 1.2 核心设计原则

| 原则 | 说明 |
|------|------|
| **有损但可控** | 压缩是有损的——中间轮次被 LLM 总结为摘要。但通过结构化模板和迭代更新最大化保留关键信息 |
| **头尾保护** | 系统提示（head）和最近上下文（tail）**永远不被压缩**，只压缩"中间段" |
| **先 flush 再压缩** | 压缩前先让 memory provider 提取有价值的信息，避免原始细节丢失 |
| **不阻塞主流程** | 压缩使用独立的**辅助模型**（便宜/快速），不影响主模型的推理质量 |
| **可恢复** | 压缩失败时优雅降级——丢弃中间轮次但不注入错误摘要 |
| **防抖动** | 连续多次压缩效果不佳时自动停止，避免无限压缩循环 |

### 1.3 与简单方案对比

```
简单聊天机器人:  滑动窗口 / 截断旧消息
                ↓ 缺点: 直接丢失信息

Hermes 方案:     结构化 LLM 摘要 + 迭代更新 + 头尾保护
                ↓ 优势: 保留决策过程、文件状态、待办事项等关键上下文
```

---

## 2. 架构总览

```mermaid
classDiagram
    class ContextEngine {
        <<abstract>>
        +name: str
        +last_prompt_tokens: int
        +threshold_tokens: int
        +context_length: int
        +compression_count: int
        +threshold_percent: float = 0.75
        +protect_first_n: int = 3
        +protect_last_n: int = 6
        +update_from_response(usage)*
        +should_compress(tokens)* bool
        +compress(messages)* list
        +on_session_reset()
        +update_model(model, ctx_len)
        +get_tool_schemas() list
    }

    class ContextCompressor {
        -model: str
        -base_url: str
        -api_key: str
        -provider: str
        -api_mode: str
        -threshold_percent: float = 0.50
        -protect_first_n: int = 3
        -protect_last_n: int = 20
        -summary_target_ratio: float = 0.20
        -tail_token_budget: int
        -max_summary_tokens: int = 12000
        -_previous_summary: str | None
        -_ineffective_compression_count: int
        -_summary_failure_cooldown_until: float
        +should_compress(tokens) bool
        +compress(messages, current_tokens, focus_topic) list
        -_prune_old_tool_results() tuple
        -_find_tail_cut_by_tokens() int
        -_generate_summary() str | None
        -_serialize_for_summary() str
        -_sanitize_tool_pairs() list
        -_align_boundary_forward() int
        -_align_backward() int
        -_ensure_last_user_message_in_tail() int
    }

    ContextEngine <|-- ContextCompressor
```

### 2.1 继承体系

`ContextEngine` 是抽象基类，定义了所有上下文引擎必须实现的接口。`ContextCompressor` 是内置实现。第三方引擎（如 LCM DAG 引擎）可以通过插件系统替换它：

```
config.yaml:
  context:
    engine: "compressor"   ← 默认（内置）
    # engine: "lcm"       ← 第三方插件
```

---

## 3. 压缩触发机制 — 6 个调用点

`run_agent.py` 中 `_compress_context()` 共被调用 **6 次**，分布在不同的场景中：

### 3.1 触发点总览

```mermaid
flowchart TD
    START[用户消息进入] --> PREFLIGHT{Preflight 检查}
    PREFLIGHT -->|tokens >= threshold| C1[调用点①: Preflight 压缩<br/>主循环前]
    PREFLIGHT -->|tokens < threshold| MAINLOOP[进入主循环]

    MAINLOOP --> APICALL[API 调用]
    APICALL --> APIOK{API 成功?}
    APIOK -->|是| RESPONSE[解析响应]
    APIOK -->|否: context_length_exceeded| C3[调用点③: 上下文超限恢复]
    APIOK -->|否: 413 payload_too_large| C4[调用点④: 请求体过大恢复]
    APIOK -->|否: anthropic_long_context_tier| C2[调用点②: Anthropic 长上下文层降级]

    RESPONSE --> TOOLCALL{有 tool_calls?}
    TOOLCALL -->|是| TOOLEXEC[执行工具]
    TOOLEXEC --> C5{工具后检查<br/>should_compress?}
    C5 -->|tokens > threshold| C5_CALL[调用点⑤: 工具执行后正常压缩]
    C5 -->|tokens <= threshold| NEXTLOOP[继续下一轮]
    C5_CALL --> NEXTLOOP
    TOOLCALL -->|否| FINAL[最终响应]

    C1 --> MAINLOOP
    C2 --> APICALL_RETRY[重试 API]
    C3 --> API_RETRY_OR_FAIL[重试或失败返回]
    C4 --> API_RETRY_OR_FAIL
    API_CALL_RETRY --> APICALL
    API_RETRY_OR_FAIL --> APICALL

    style C1 fill:#e3f2fd
    style C2 fill:#fff3e0
    style C3 fill:#ffebee
    style C4 fill:#ffebee
    style C5 fill:#e8f5e9
```

### 3.2 各调用点详情

| # | 调用位置 | 触发条件 | 场景说明 | 代码位置 |
|---|----------|----------|----------|----------|
| **①** | 主循环入口前 (Preflight) | `estimated_tokens >= threshold_tokens` | 会话历史很长，进入主循环前先压缩一轮 | [run_agent.py:8870](file:///Users/lixiangyang/Desktop/代码/hermes-agent-main/hermes-agent-main/run_agent.py#L8870) |
| **②** | API 错误恢复 (Anthropic) | Anthropic long-context tier 错误 | Anthropic 的 extended thinking 模式超出长上下文层级限制 | [run_agent.py:10381](file:///Users/lixiangyang/Desktop/代码/hermes-agent-main/hermes-agent-main/run_agent.py#L10381) |
| **③** | API 错误恢复 (通用) | `context_length_exceeded` 分类错误 | 模型返回 token 超限错误 | [run_agent.py:10615](file:///Users/lixiangyang/Desktop/代码/hermes-agent-main/hermes-agent-main/run_agent.py#L10615) |
| **④** | API 错误恢复 (413) | HTTP 413 Payload Too Large | 请求体过大（通常因消息过多） | [run_agent.py:10479](file:///Users/lixiangyang/Desktop/代码/hermes-agent-main/hermes-agent-main/run_agent.py#L10479) |
| **⑤** | 工具执行后 (正常路径) | `should_compress(last_prompt_tokens)` == True | 每轮工具执行后检查，最常见的触发方式 | [run_agent.py:11339](file:///Users/lixiangyang/Desktop/代码/hermes-agent-main/hermes-agent-main/run_agent.py#L11339) |

> **注意**: 调用点 ②③④ 属于**错误恢复路径**，最多重试 `max_compression_attempts`（默认 3 次）。调用点 ① 和 ⑤ 是**正常路径**。

---

## 4. 五阶段压缩算法详解

`ContextCompressor.compress()` 是核心入口，执行五阶段算法：

```mermaid
flowchart TD
    INPUT[messages 列表] --> P1["Phase 1: 工具输出预剪枝<br/>_prune_old_tool_results()<br/>无 LLM 调用"]
    P1 --> P2["Phase 2: 边界确定<br/>head 保护 + tail Token 预算"]
    P2 --> P3["Phase 3: 结构化摘要生成<br/>_generate_summary()<br/>调用辅助 LLM"]
    P3 --> P4["Phase 4: 消息重组<br/>head + summary + tail"]
    P4 --> P5["Phase 5: 清理<br/>_sanitize_tool_pairs()<br/>修复孤立 tool_call/result"]

    P1 -->|剪掉 N 个旧工具结果| PRUNE_STATS["pruned_count 个工具输出<br/>被替换为单行摘要"]
    P3 -->|LLM 成功| SUMMARY["结构化摘要文本"]
    P3 -->|LLM 失败| FALLBACK["静态降级摘要<br/>或直接丢弃中间段"]
    P5 --> OUTPUT[compressed messages]

    style P1 fill:#e8f5e9
    style P2 fill:#e3f2fd
    style P3 fill:#fff3e0
    style P4 fill:#fce4ec
    style P5 fill:#f3e5f5
```

---

## 5. 工具输出预剪枝（Phase 1）

### 5.1 为什么需要预剪枝

工具调用的输出往往非常大：
- `read_file` 一个大文件 → 数万字符
- `terminal` 执行 `npm test` → 数千行输出
- `search_files` 全项目搜索 → 大量匹配内容

如果把这些原样送给摘要 LLM，会浪费大量 token 且降低摘要质量。

### 5.2 剪枝策略

**不是简单的 `[pruned]` 占位符**，而是生成**信息丰富的单行摘要**：

```python
def _summarize_tool_result(tool_name, tool_args, tool_content) -> str:
```

| 工具类型 | 摘要示例 |
|----------|----------|
| `terminal` | `[terminal] ran \`npm test\` -> exit 0, 47 lines output` |
| `read_file` | `[read_file] read config.py from line 1 (1,200 chars)` |
| `write_file` | `[write_file] wrote to auth.py (45 lines)` |
| `patch` | `[patch] replace in user.py (3,400 chars result)` |
| `search_files` | `[search_files] content search for 'compress' in agent/ -> 12 matches` |
| `execute_code` | `[execute_code] \`print("hello")\` (5 lines output)` |
| `delegate_task` | `[delegate_task] '重构认证模块' (2,100 chars result)` |
| `memory` | `[memory] add on preferences` |

### 5.3 三步预处理流程

```mermaid
flowchart LR
    A[完整 messages] --> B["Pass 1: 去重<br/>同一文件读多次→只保留最新一次"]
    B --> C["Pass 2: 替换旧工具结果<br/>tail 保护区外的 tool result → 单行摘要"]
    C --> D["Pass 3: 截断工具参数<br/>assistant 消息中的 tool_call arguments > 200 字符 → 截断"]
    D --> E[剪枝后 messages]
```

#### Pass 1: 去重

当同一个文件被 `read_file` 读取了 5 次，只保留**最新一次的完整内容**，旧的替换为回引：

```
[read_file] config.py was read again — see latest result above
```

#### Pass 2: 替换旧工具结果

从消息末尾向前遍历，**跳过 tail 保护区**内的消息。保护区外的每个 `tool` role 消息的内容被替换为 `_summarize_tool_result()` 生成的单行摘要。

#### Pass 3: 截断工具参数

`assistant` 消息中的 `tool_call[].function.arguments` 如果超过 200 字符，会被截断（保持 JSON 合法性）：

```python
# 截断前: {"path": "/very/long/path/to/some/file.py", "content": "# very long markdown..."}
# 截断后: {"path": "/very/long/path/to/some/file.py", "content": "# very long mark...[truncated]"}
```

> **关键细节**: 使用 JSON 解析后递归截断字符串叶子节点再重新序列化，而不是盲目切片——避免产生非法 JSON（issue #11762）。

---

## 6. 边界确定策略（Phase 2）

### 6.1 三区域划分

```
messages[0..N]:
┌─────────────────────────────────────────────────────┐
│  HEAD (保护)          │  MIDDLE (压缩)    │ TAIL (保护) │
│  ─────────            │  ────────────     │  ─────────  │
│  system prompt        │  turn K           │  turn M     │
│  user msg 1           │  turn K+1         │  ...        │
│  assistant msg 1      │  ...              │  user msg   │
│  (前 protect_first_n │                    │  (Token预算) │
│   条消息)              │                    │             │
└─────────────────────────────────────────────────────┘
↑ compress_start=0      ↑ compress_start      ↑ compress_end
(固定)                  (_align_forward)     (_find_tail_cut_by_tokens)
```

### 6.2 Head 保护

固定保护前 `protect_first_n`（默认 **3**）条消息。通常是：
- `messages[0]`: system prompt
- `messages[1]`: 第一条 user 消息
- `messages[2]`: 第一条 assistant 回复

### 6.3 Tail 保护 — Token 预算模式（v3 改进）

旧版本使用**固定消息数**保护尾部（如最后 20 条），但这不合理——有些消息只有几个字符，有些有几万字符。

v3 改为 **Token 预算模式**：

```python
def _find_tail_cut_by_tokens(messages, head_end, token_budget=None):
    """从末尾向前累加 token，直到超过预算"""
```

**算法**:

1. 从 `messages` 末尾向前遍历
2. 累加每条消息的估算 token 数（`len(content) // 4 + 10`）
3. 当累计超过 `soft_ceiling = token_budget * 1.5` 时停止
4. **硬性最低保护**: 至少保留 3 条消息在 tail
5. **对齐修正**: 不在 tool_call/result 组中间切割
6. **用户消息锚定**: 最后一条 user 消息必须在 tail 内（修复 bug #10896）

**Tail 预算计算**:

```
tail_token_budget = threshold_tokens * summary_target_ratio
                   = (context_length * 0.50) * 0.20
                   ≈ context_length * 0.10

例如: 200K 上下文 → threshold=100K → tail_budget≈20K tokens
```

### 6.4 边界对齐规则

```mermaid
flowchart TD
    RAW_CUT[初始 cut_idx] --> FWD["_align_boundary_forward()<br/>如果 cut_idx 落在 tool result 上<br/>向前移到非 tool 消息"]
    FWD --> BWD["_align_backward()<br/>如果 cut_idx 后面有连续 tool results<br/>向后拉到它们的 parent assistant 消息"]
    BWD --> USER_ANCHOR["_ensure_last_user_message_in_tail()<br/>如果最后一条 user 消息落在 middle 区<br/>把 cut_idx 拉回到该 user 消息处"]
    USER_ANCHOR --> FINAL_CUT[最终 cut_idx]

    style FWD fill:#e3f2fd
    style BWD fill:#fff3e0
    style USER_ANCHOR fill:#ffebee
```

这三条对齐规则确保：
- 压缩不会从一组 tool_call/result 的**中间**开始
- 压缩不会丢失用户的**最新指令**
- API 收到的消息列表始终是**合法的**

---

## 7. 结构化摘要生成（Phase 3）

### 7.1 序列化

中间段消息先被序列化为纯文本，供摘要 LLM 消费：

```python
def _serialize_for_summary(self, turns) -> str:
```

**处理规则**:

| 角色 | 格式 | 截断限制 |
|------|------|----------|
| `user` | `[USER]: {content}` | 每条消息最多 6000 字符（头 4000 + 尾 1500） |
| `assistant` | `[ASSISTANT]: {content}\n[Tool calls:\n  name(args)\n]` | 同上 + tool call 参数最多 1500 字符 |
| `tool` | `[TOOL RESULT {id}]: {content}` | 同上 |

**所有内容经过 `redact_sensitive_text()` 处理**，防止 API key / password 泄露到摘要中。

### 7.2 摘要 Prompt 结构

使用**结构化模板**，包含以下字段：

```
## Active Task              ← 最重要！用户最新的未完成任务（逐字复制）
## Goal                     ← 用户总体目标
## Constraints & Preferences ← 用户偏好/约束
## Completed Actions        ← 编号列表: N. ACTION target — outcome [tool: name]
## Active State             ← 当前工作状态（目录/分支/文件/测试/进程）
## In Progress              ← 压缩时刻正在进行的工作
## Blocked                  ← 未解决的阻碍和精确错误信息
## Key Decisions            ← 重要技术决策及其原因
## Resolved Questions       ← 已回答的问题及答案
## Pending User Asks        ← 用户未满足的请求
## Relevant Files           ← 读/改/创建的文件清单
## Remaining Work           ← 待完成工作（作为上下文而非指令）
## Critical Context          ← 不能丢失的具体值/配置/数据
```

### 7.3 关键设计决策

#### (a) "Do Not Respond" 前言

```
You are a summarization agent creating a context checkpoint.
Your output will be injected as reference material for a DIFFERENT assistant.
Do NOT respond to any questions or requests in the conversation.
```

来源: OpenCode 项目。防止摘要 LLM "帮忙回答"用户之前的问题。

#### (b) Handoff Framing

```
This is a handoff from a previous context window — 
treat it as background reference, NOT as active instructions.
```

来源: Codex 项目。告诉下一个助手这是**参考材料**而非**当前指令**。

#### (c) "Remaining Work" 而非 "Next Steps"

避免下一个模型把 "Next Steps" 当作**需要执行的指令**来执行，造成重复操作。

#### (d) 语言一致性

```
Write the summary in the same language the user was using in the conversation.
```

### 7.4 迭代式摘要更新

第二次及以后的压缩**不是从头总结**，而是基于上次摘要做增量更新：

```mermaid
sequenceDiagram
    participant OldSummary as 上次摘要
    participant NewTurns as 新增轮次
    participant Summarizer as 摘要 LLM
    participant NewSummary as 更新后摘要

    Summarizer->>Summarizer: 读取 PREVIOUS SUMMARY
    Summarizer->>Summarizer: 读取 NEW TURNS TO INCORPORATE
    Summarizer->>Summarizer: 生成增量更新 Prompt
    Note over Summarizer: PRESERVE 已有相关信息<br/>ADD 新 completed actions 到编号列表<br/>MOVE In Progress → Completed<br/>UPDATE Active Task 为最新未完成任务
    Summarizer-->>NewSummary: 更新后的结构化摘要
```

**好处**: 多次压缩后仍能保留早期的重要决策和上下文，不会因为每次都从零开始而丢失信息。

### 7.5 摘要 Budget 计算

```python
def _compute_summary_budget(self, turns_to_summarize) -> int:
    content_tokens = estimate_messages_tokens_rough(turns_to_summarize)
    budget = int(content_tokens * _SUMMARY_RATIO)  # _SUMMARY_RATIO = 0.20
    return max(_MIN_SUMMARY_TOKENS, min(budget, _SUMMARY_TOKENS_CEILING))
    # _MIN_SUMMARY_TOKENS = 2000
    # _SUMMARY_TOKENS_CEILING = 12000
```

即：摘要 token 数 = 被压缩内容的 **20%**，范围 **2000 ~ 12000 tokens**。

---

## 8. 消息重组与清理（Phase 4 & 5）

### 8.4 Phase 4: 组装压缩后的消息列表

```
压缩前: [sys][u1][a1][t1][u2][a2][t2][t3][u3][a3][t4][u4][a4][t5][u5]
                        ↑ start                       ↑ end
压缩后: [sys'][u1][a1][ SUMMARY_TEXT ][u4][a4][t4][u5]
         ↑ head   ↑        ↑ middle (summary)  ↑   tail
```

具体操作：
1. **Head 区域**原样保留，system 消息追加压缩标注
2. **Middle 区域**替换为一条 user 消息，内容 = `SUMMARY_PREFIX + 摘要文本`
3. **Tail 区域**原样保留

**System 消息标注**:

```
[Note: Some earlier conversation turns have been compacted into a handoff 
summary to preserve context space. The current session state may still 
reflect earlier work, so build on that summary and state rather than 
re-doing work.]
```

**摘要前缀 (SUMMARY_PREFIX)**:

```
[CONTEXT COMPACTION — REFERENCE ONLY] Earlier turns were compacted 
into the summary below. This is a handoff from a previous context 
window — treat it as background reference, NOT as active instructions. 
Do NOT answer questions or fulfill requests mentioned in this summary; 
they were already addressed. Your current task is identified in the 
'## Active Task' section of the summary — resume exactly from there. 
Respond ONLY to the latest user message that appears AFTER this summary.
```

### 8.5 Phase 5: Tool Pair 清理 (`_sanitize_tool_pairs`)

压缩可能导致 **tool_call / tool_result 配对断裂**：

```mermaid
flowchart LR
    A["问题 1: 孤立 tool_result<br/>对应的 assistant tool_call 被压缩掉了"] --> FIX1["删除孤立的 tool result"]
    B["问题 2: 孤立 tool_call<br/>对应的 tool result 被压缩掉了"] --> FIX2["插入 stub tool result:<br/>'[Result from earlier conversation —<br/>see context summary above]'"]
    
    FIX1 --> CLEAN[合法的消息列表]
    FIX2 --> CLEAN
```

如果不修复这两种情况，API 会返回错误：
- `"No tool call found for function call output with call_id ..."`
- `"Every tool_call must be followed by a matching tool result"`

---

## 9. 辅助模型选择与回退链

压缩使用的 LLM 不是主模型，而是**独立配置的辅助模型**（更便宜、更快）。

### 9.1 解析优先级

`call_llm()` 在 `auxiliary_client.py` 中的解析顺序（text 任务）：

| 优先级 | Provider | 默认模型 | 说明 |
|--------|----------|----------|------|
| 1 | OpenRouter | `google/gemini-3-flash-preview` | 聚合路由 |
| 2 | Nous Portal | `google/gemini-3-flash-preview` | Nous 推理服务 |
| 3 | Custom Endpoint | 用户配置 | `config.yaml model.base_url` |
| 4 | Codex OAuth | `gpt-5.2-codex` | ChatGPT Responses API |
| 5 | Native Anthropic | `claude-haiku-4-5` | 官方 Anthropic |
| 6 | Direct API Key providers | 见下表 | 按 provider 选择 |
| 7 | None | — | 无可用 provider |

**Direct Provider 默认辅助模型**:

| Provider | 默认辅助模型 |
|----------|-------------|
| Gemini | `gemini-3-flash-preview` |
| GLM (智谱) | `glm-4.5-flash` |
| Kimi/Moonshot | `kimi-k2-turbo-preview` |
| MiniMax | `MiniMax-M2.7` |
| Anthropic | `claude-haiku-4-5-20251001` |

### 9.2 可覆盖配置

```yaml
# config.yaml
auxiliary:
  compression:
    model: "gpt-4o-mini"          # 自定义压缩模型
    provider: "openrouter"        # 或指定 provider
    context_length: 128000        # 覆盖模型上下文长度
```

### 9.3 摘要模型回退

当专用摘要模型不可用时（404/503），自动回退到**主模型**：

```mermaid
flowchart TD
    CALL[调用摘要模型] --> OK[成功 → 返回摘要]
    CALL --> FAIL{失败}
    FAIL -->|model_not_found / 404 / 503<br/>且 summary_model != main_model| FALLBACK["回退到主模型重试"]
    FALLBACK --> RETRY[重新调用]
    RETRY --> OK
    RETRY --> FAIL2{仍然失败?}
    FAIL2 -->|瞬态错误| COOLDOWN["冷却 60 秒<br/>返回 None (降级)"]
    FAIL2 -->|无 provider| LONG_COOLDOWN["冷却 600 秒<br/>返回 None (降级)"]
    FALLBACK -->|已回退过一次| COOLDOWN
```

---

## 10. 反抖动与安全保护

### 10.1 Anti-Thrashing（反抖动）

```python
def should_compress(self, prompt_tokens=None) -> bool:
    if tokens < self.threshold_tokens:
        return False
    if self._ineffective_compression_count >= 2:
        return False  # 连续两次压缩效果差 → 跳过
    return True
```

**判定标准**: 如果一次压缩节省的 token **不足 10%**，计为"无效压缩"。连续 **2 次**无效后暂停压缩。

**原因**: 某些场景下（如大量小消息），每轮压缩只能删掉 1-2 条消息，形成无限循环。

### 10.2 失败冷却

| 错误类型 | 冷却时间 | 行为 |
|----------|----------|------|
| 无可用 provider | 600 秒 (10 分钟) | 中间段直接丢弃，不注入摘要 |
| 瞬态错误 (超时/限流) | 60 秒 | 下次尝试前等待 |
| 模型不存在 (404/503) | 不冷却 | 立即回退到主模型重试 |

### 10.3 敏感信息脱敏

两层防护：
1. **序列化前**: `redact_sensitive_text()` 处理每条消息内容
2. **摘要后**: 对 LLM 输出的摘要再次 `redact_sensitive_text()`（防止 LLM 忽略指令回显密钥）

---

## 11. 聚焦主题压缩（Focus Topic）

灵感来自 Claude Code 的 `/compact <focus>` 命令。

### 11.1 用法

```python
# 通过 /compress <topic> 命令手动触发
messages, new_prompt = self._compress_context(
    messages, system_message,
    focus_topic="authentication module refactor"
)
```

### 11.2 效果

摘要 Prompt 末尾追加：

```
FOCUS TOPIC: "authentication module refactor"
The user has requested that this compaction PRIORITISE preserving all 
information related to the focus topic above. For content related to 
"authentication module refactor", include full detail — exact values, 
file paths, command outputs, error messages, and decisions. For content 
NOT related to the focus topic, summarise more aggressively (brief 
one-liners or omit if truly irrelevant). The focus topic sections should 
receive roughly 60-70% of the summary token budget.
```

### 11.3 典型场景

用户在一个长会话中完成了多个任务，现在想专注于其中一个继续深入：

```
用户: /compress JWT authentication
效果: 与 JWT/auth 相关的细节完整保留（文件路径、错误信息、决策原因）
      其他任务（如 UI 调整、日志配置）被激进压缩为一行概括
```

---

## 12. 会话切分与持久化

### 12.1 `_compress_context()` 完整流程

[`run_agent.py:_compress_context()`](file:///Users/lixiangyang/Desktop/代码/hermes-agent-main/hermes-agent-main/run_agent.py#L7519-L7638) 不仅调用压缩器，还负责完整的生命周期管理：

```mermaid
sequenceDiagram
    participant Agent as AIAgent
    participant MemMgr as MemoryManager
    participant CC as ContextCompressor
    participant AuxLLM as 辅助 LLM
    participant DB as SessionDB
    participant Todo as TodoStore

    Agent->>Agent: 1. flush_memories(min_turns=0)
    Note over Agent: 压缩前先把值得保存的记忆落盘
    
    Agent->>MemMgr: 2. on_pre_compress(messages)
    Note over MemMgr: 通知外部 memory provider<br/>即将丢弃上下文

    Agent->>CC: 3. compress(messages, approx_tokens, focus_topic)
    CC->>CC: 3a. _prune_old_tool_results()
    CC->>CC: 3b. 确定边界 (head/tail/middle)
    CC->>AuxLLM: 3c. _generate_summary()
    AuxLLM-->>CC: 摘要文本 or None
    CC->>CC: 3d. 组装消息 + _sanitize_tool_pairs()
    CC-->>Agent: compressed messages

    Agent->>Todo: 4. 注入 todo snapshot
    Todo-->>Agent: todo Markdown 文本
    Note over Agent: todo 列表不会被压缩丢失

    Agent->>Agent: 5. _invalidate_system_prompt()
    Agent->>Agent: 6. _build_system_prompt() → new_system_prompt
    Note over Agent: 重建 system prompt<br/>(prefix cache 位置变了)

    Agent->>DB: 7. commit_memory_session(old messages)
    Agent->>DB: 8. end_session(old_session_id, "compression")
    Agent->>DB: 9. create_session(new_session_id, parent=old)
    Agent->>DB: 10. set_title(auto-numbered lineage)
    Agent->>DB: 11. update_system_prompt(new_prompt)
    Note over DB: SQLite 会话切分<br/>新 session 继承旧 session 的 title lineage

    Agent->>Agent: 12. 重置 file-read dedup cache
    Agent->>Agent: 13. 更新 last_prompt_tokens 估算
    Agent-->>Agent: (compressed_messages, new_system_prompt)
```

### 12.2 为什么要在压缩前 flush memories

这是一个关键设计决策：

```
时间线:
  t1: 用户说 "记住我喜欢用 JWT 认证"
  t2: agent 调用 memory add → 写入 MEMORY.md
  t3...tN: 大量工具调用 → 上下文膨胀
  tN+1: 触发压缩 → 中间段（含 t2 的 memory 调用结果）被总结
  
如果不在压缩前 flush:
  → t2 的 memory add 结果被 LLM 总结为 "[memory] add on preferences"
  → 具体的 "JWT 认证" 偏好细节丢失
  
如果在压缩前 flush:
  → memory provider 已经把 "JWT 认证偏好" 持久化了
  → 即使摘要丢失细节，MEMORY.md 里还有完整记录
```

### 12.3 Session Lineage（会话谱系）

压缩时会创建一个新的 SQLite session，并记录父子关系：

```
Session A (original)
  ├── title: "Refactor auth module"
  ├── messages: [sys][u1][a1]...[u50][a50]  (压缩前的完整历史)
  └── status: ended (compression)

Session B (child, auto-created by compression)
  ├── parent_session_id: A
  ├── title: "Refactor auth module (2)"  (自动编号)
  ├── messages: [sys'][u1][a1][SUMMARY][u48][a48][u50]  (压缩后)
  └── status: active
```

---

## 13. 配置参数一览

### 13.1 config.yaml 配置项

```yaml
compression:
  enabled: true                    # 是否启用自动压缩
  threshold: 0.50                  # 触发阈值 (占上下文窗口比例), 默认 50%
  target_ratio: 0.20               # 摘要预算比例 (相对阈值), 默认 20%
  protect_last_n: 20               # tail 保护最小消息数 (兼容旧版)
  # model: ""                      # 覆盖压缩模型 (留空=自动选择)

auxiliary:
  compression:
    model: ""                      # 覆盖辅助压缩模型
    provider: ""                   # 覆盖辅助压缩 provider
    context_length: null            # 覆盖辅助模型上下文长度

context:
  engine: "compressor"             # 上下文引擎选择
```

### 13.2 内部常量

| 常量 | 值 | 说明 |
|------|-----|------|
| `_SUMMARY_RATIO` | 0.20 | 摘要预算占被压缩内容的比例 |
| `_MIN_SUMMARY_TOKENS` | 2000 | 摘要最低 token 数 |
| `_SUMMARY_TOKENS_CEILING` | 12000 | 摘要最高 token 数 |
| `_CHARS_PER_TOKEN` | 4 | 粗略估算: 1 token ≈ 4 字符 |
| `_CONTENT_MAX` | 6000 | 摘要输入中每条消息最大字符数 |
| `_CONTENT_HEAD` | 4000 | 消息头部保留字符数 |
| `_CONTENT_TAIL` | 1500 | 消息尾部保留字符数 |
| `_TOOL_ARGS_MAX` | 1500 | tool call 参数最大字符数 |
| `_PRUNED_TOOL_PLACEHOLDER` | `[Old tool output cleared...]` | 降级占位符 |
| `_SUMMARY_FAILURE_COOLDOWN_SECONDS` | 600 | 无 provider 时冷却秒数 |
| `threshold_percent` (默认) | 0.50 | 压缩阈值 = 上下文长度 × 50% |
| `protect_first_n` (默认) | 3 | head 保护消息数 |
| `protect_last_n` (默认) | 20 | tail 保护最小消息数 |
| `summary_target_ratio` (默认) | 0.20 | tail token 预算比例 |
| `max_summary_tokens` (上限) | min(ctx×0.05, 12000) | 摘要绝对上限 |

### 13.3 参数影响示意

假设主模型上下文窗口为 **200K tokens**：

| 参数 | 值 | 结果 |
|------|-----|------|
| `threshold_percent` | 0.50 | **100K tokens** 时触发压缩 |
| `summary_target_ratio` | 0.20 | tail 保护约 **20K tokens** |
| `max_summary_tokens` | min(200K×0.05, 12K) = **12K** | 摘要最长 12K tokens |
| `_SUMMARY_RATIO` | 0.20 | 若压缩 80K tokens 内容 → 摘要预算 **16K** → 上限封顶 **12K** |

---

## 14. 完整时序图

```mermaid
sequenceDiagram
    participant User as 用户
    participant Agent as AIAgent
    participant CC as ContextCompressor
    participant MemMgr as MemoryManager
    participant AuxLLM as 辅助 LLM
    participant DB as SessionDB

    rect rgba(255, 240, 240, 0.3)
        Note over User,CC: ═══ 触发条件: tokens >= threshold ═══
        
        alt 调用点①: Preflight (主循环前)
            User->>Agent: 发送消息
            Agent->>CC: should_compress(estimated_tokens)?
        else 调用点⑤: 正常 (工具执行后)
            Agent->>CC: should_compress(last_prompt_tokens)?
        else 调用点②③④: 错误恢复
            Agent->>Agent: API 返回 context_length_exceeded / 413 / tier error
            Agent->>CC: compress(...) (立即触发)
        end
    end

    rect rgba(240, 245, 255, 0.3)
        Note over Agent,DB: ═══ Phase 0: 压缩前准备 ═══
        Agent->>MemMgr: flush_memories(messages, min_turns=0)
        Note over MemMgr: 提取值得长期保存的记忆
        Agent->>MemMgr: on_pre_compress(messages)
        Note over MemMgr: 通知外部 provider 即将丢上下文
    end

    rect rgba(240, 255, 240, 0.3)
        Note over CC,AuxLLM: ═══ Phase 1: 工具输出预剪枝 (无 LLM) ═══
        CC->>CC: Pass 1: 去重相同文件读取
        CC->>CC: Pass 2: 旧 tool result → 单行摘要
        CC->>CC: Pass 3: 截断超长 tool_call args
        Note over CC: pruned_count 个工具输出被精简
    end

    rect rgba(255, 255, 240, 0.3)
        Note over CC: ═══ Phase 2: 边界确定 ═══
        CC->>CC: head = messages[0..protect_first_n]
        CC->>CC: _find_tail_cut_by_tokens(token_budget)
        Note over CC: 从末尾向前累加 token 直到超预算
        CC->>CC: _align_boundary_forward()
        CC->>CC: _align_backward()
        CC->>CC: _ensure_last_user_message_in_tail()
        Note over CC: 确定 middle = messages[start:end]
    end

    rect rgba(255, 240, 245, 0.3)
        Note over CC,AuxLLM: ═══ Phase 3: 结构化摘要生成 ═══
        CC->>CC: _serialize_for_summary(middle turns)
        Note over CC: redact 敏感信息 + 截断长消息
        
        alt 有上次摘要 (迭代更新)
            CC->>AuxLLM: "更新摘要: PREVIOUS + NEW_TURNS"
        else 首次压缩
            CC->>AuxLLM: "从头总结: TURNS_TO_SUMMARIZE"
        end
        
        AuxLLM-->>CC: 结构化摘要 or 异常
        
        alt 摘要成功
            CC->>CC: redact_sensitive_text(摘要)
            Note over CC: 存储 _previous_summary 供下次迭代
        else 摘要模型 404/503 + 有备用模型
            CC->>CC: 回退到主模型重试
        else 摘要失败
            Note over CC: 返回 None → 静态降级
        end
    end

    rect rgba(245, 240, 255, 0.3)
        Note over CC: ═══ Phase 4: 消息重组 ═══
        CC->>CC: head + [SUMMARY_PREFIX + 摘要] + tail
        CC->>CC: system 消息追加压缩标注
    end

    rect rgba(240, 240, 255, 0.3)
        Note over CC: ═══ Phase 5: Tool Pair 清理 ═══
        CC->>CC: _sanitize_tool_pairs()
        Note over CC: 删除孤立 tool_result<br/>补充缺失 stub result
        CC-->>Agent: compressed messages
    end

    rect rgba(255, 255, 220, 0.3)
        Note over Agent,DB: ═══ 后处理 ═══
        Agent->>Agent: 注入 todo snapshot
        Agent->>Agent: _invalidate_system_prompt()
        Agent->>Agent: _build_system_prompt() → new_prompt
        Agent->>DB: commit_memory_session(old)
        Agent->>DB: end_session(old, "compression")
        Agent->>DB: create_session(new, parent=old)
        Agent->>DB: set_title(lineage auto-numbered)
        Agent->>Agent: 重置 file dedup cache
        Agent->>Agent: 更新 token 估算
        Agent-->>Agent: (compressed_messages, new_system_prompt)
    end
```

---

## 附录: 常见问题

### Q1: 压缩会影响模型质量吗？

会有一定损失，但通过以下手段最小化：
- 结构化模板保留了**决策原因**而不仅是结论
- 迭代更新避免了多轮压缩的信息累积损失
- 头尾保护确保**系统指令**和**最新上下文**完整保留

### Q2: 压缩后为什么需要重建 system prompt？

因为压缩改变了消息列表中首条消息的位置，Anthropic 等 provider 的 **prefix cache** 基于 token 位置命中。重建确保缓存一致性。

### Q3: 可以手动触发压缩吗？

可以。通过 `/compress [topic]` 命令手动触发，支持可选的聚焦主题。

### Q4: 压缩和 memory 系统的关系？

它们互补：
- **Memory** (MEMORY.md): 长期、跨会话的持久记忆（用户偏好、项目知识）
- **Compression**: 会话内的中期上下文管理（本轮任务的中间状态）

压缩前会先 flush memory，确保重要信息已经落盘。
