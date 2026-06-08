# Offload 上下文卸载模块设计文档

## 1. 模块概述

Offload 模块实现了一套**四层递进式上下文压缩系统**，旨在通过 Mermaid 符号图和工具日志卸载来减少 LLM 上下文窗口的 token 使用量，使 Agent 能够在长任务中持续运行而不超出上下文限制。

### 1.1 核心目标

- **延长 Agent 运行时间**：通过分层压缩策略，使 Agent 在 200K token 上下文窗口下可持续运行数千轮工具调用
- **保留任务方向感**：通过 Mermaid 流程图（MMD）为 LLM 提供任务进度的可视化摘要
- **最小化信息损失**：优先替换可恢复的工具结果，保留关键用户消息和当前任务上下文
- **异步非阻塞**：L1/L1.5/L2 均为异步执行，不阻塞主 LLM 调用路径

### 1.2 四层架构总览

| 层级 | 名称 | 触发时机 | 核心功能 | 执行方式 |
|------|------|----------|----------|----------|
| **L1** | 摘要化 | after_tool_call / assemble | 工具调用+结果 → LLM 摘要 + score | 异步（fire-and-forget） |
| **L1.5** | 任务边界判断 | assemble / before_prompt_build | 判断任务完成/延续/新任务，管理 MMD 文件 | 异步（fire-and-forget） |
| **L2** | Mermaid 生成 | 轮询调度（null 条目数/超时） | 将 offload 条目归入 MMD 节点，生成/更新流程图 | 异步（后台轮询） |
| **L3** | 上下文压缩 | assemble / after_tool_call / before_prompt_build | Mild 替换 / Aggressive 删除 / Emergency 截断 | 同步（阻塞式） |
| **L4** | Skill 生成 | before_agent_start（/create-skill 命令） | 从 MMD + offload 条目生成可复用 Skill | 同步 |

### 1.3 源码结构

```
src/offload/
├── index.ts                  # 模块入口与编排中心
├── types.ts                  # 核心类型定义
├── state-manager.ts          # 会话状态管理
├── storage.ts                # 文件 I/O 层
├── session-registry.ts       # 会话注册表（LRU）
├── reclaimer.ts              # 过期数据回收
├── mmd-injector.ts           # MMD 注入器
├── mmd-meta.ts               # MMD 元数据解析
├── context-token-tracker.ts  # tiktoken 精确计数
├── fast-token-estimate.ts    # 快速 Token 估算
├── l3-helpers.ts             # L3 压缩辅助函数
├── l3-token-counter.ts       # L3 Token 计数器
├── l3-token-helpers.ts       # L3 Token 辅助
├── backend-client.ts         # 远程后端 HTTP 客户端
├── opik-tracer.ts            # Opik 链路追踪
├── state-reporter.ts         # 状态上报
├── time-utils.ts             # 时间工具
├── user-id.ts                # 用户标识解析
├── hooks/
│   ├── after-tool-call.ts    # after_tool_call 钩子
│   ├── before-agent-start.ts # L1.5 任务转换处理
│   ├── before-prompt-build.ts# before_prompt_build 钩子
│   ├── llm-input-l3.ts       # L3 压缩核心算法
│   └── llm-output.ts         # L1 强制触发判断
├── local-llm/
│   ├── index.ts              # 本地 LLM 客户端入口
│   ├── llm-caller.ts         # Vercel AI SDK 封装
│   ├── parsers/
│   │   ├── json-utils.ts     # JSON 解析工具
│   │   ├── l1-parser.ts      # L1 响应解析
│   │   ├── l15-parser.ts     # L1.5 响应解析
│   │   └── l2-parser.ts      # L2 响应解析
│   └── prompts/
│       ├── l1-prompt.ts      # L1 提示词模板
│       ├── l15-prompt.ts     # L1.5 提示词模板
│       └── l2-prompt.ts      # L2 提示词模板
└── pipelines/
    └── l2-mermaid.ts         # L2 Mermaid 流水线
```

---

## 2. 架构设计

```mermaid
graph TB
    subgraph "钩子层 (Hooks)"
        BTC["before_tool_call<br/>缓存工具参数"]
        ATC["after_tool_call<br/>收集工具对 + MMD注入 + L3压缩"]
        BAS["before_agent_start<br/>L4 Skill 生成"]
        BPB["before_prompt_build<br/>Fast-Path + Token守卫 + MMD注入"]
        LI["llm_input<br/>Token快照 + 上下文缓存"]
        LO["llm_output<br/>L1 强制触发判断"]
    end

    subgraph "Context Engine 生命周期"
        CE_BOOT["bootstrap()<br/>会话初始化"]
        CE_ING["ingest()<br/>消息摄入"]
        CE_ASM["assemble()<br/>核心编排"]
        CE_CMP["compact()<br/>紧急压缩"]
        CE_AT["afterTurn()<br/>轮次结束清理"]
        CE_DIS["dispose()<br/>资源释放"]
    end

    subgraph "L1 摘要化管线"
        L1_FLUSH["flushL1()<br/>批量发送 → Backend/LocalLLM"]
        L1_REF["writeRefMd()<br/>写原始结果到 refs/"]
        L1_JSONL["appendOffloadEntries()<br/>写摘要到 offload.jsonl"]
    end

    subgraph "L1.5 任务判断管线"
        L15_JUDGE["judgeL15()<br/>发送上下文 → LLM"]
        L15_TRANS["handleTaskTransition()<br/>创建/重新激活/清除 MMD"]
        L15_BOUND["pushBoundary()<br/>记录任务边界"]
    end

    subgraph "L2 Mermaid 管线"
        L2_POLL["L2 轮询调度器<br/>null阈值 / 超时"]
        L2_CHECK["checkL2Trigger()<br/>边界分组 + 触发条件"]
        L2_GEN["runL2WithBackend()<br/>后端生成 MMD"]
        L2_BACK["backfillNodeIds()<br/>回填 node_id"]
    end

    subgraph "L3 压缩引擎"
        L3_FP["Fast-Path 重应用<br/>confirmed 替换 / deleted 删除"]
        L3_MILD["Mild 压缩<br/>score 级联替换"]
        L3_AGG["Aggressive 压缩<br/>最旧前缀删除"]
        L3_EMG["Emergency 压缩<br/>头部+尾部+原地截断"]
    end

    subgraph "基础设施"
        SM["OffloadStateManager<br/>会话状态管理"]
        SR["SessionRegistry<br/>LRU 会话路由"]
        ST["Storage<br/>文件 I/O (JSONL/MMD/refs/state)"]
        TT["TokenTracker<br/>tiktoken + WeakMap 缓存"]
        FT["FastEstimate<br/>字符分类快速估算"]
        BC["BackendClient<br/>HTTP 后端"]
        RC["Reclaimer<br/>过期数据回收"]
        MI["MmdInjector<br/>MMD 消息注入"]
    end

    ATC --> L1_FLUSH
    ATC --> L3_MILD
    ATC --> L3_AGG
    ATC --> L3_EMG
    ATC --> MI

    BPB --> L3_FP
    BPB --> L3_MILD
    BPB --> L3_AGG
    BPB --> L3_EMG
    BPB --> MI

    CE_ASM --> L15_JUDGE
    CE_ASM --> L1_FLUSH
    CE_ASM --> L3_FP
    CE_ASM --> L3_MILD
    CE_ASM --> L3_AGG
    CE_ASM --> L3_EMG

    L1_FLUSH --> L1_REF
    L1_FLUSH --> L1_JSONL
    L1_FLUSH --> L2_POLL

    L15_JUDGE --> L15_TRANS
    L15_JUDGE --> L15_BOUND

    L2_POLL --> L2_CHECK
    L2_CHECK --> L2_GEN
    L2_GEN --> L2_BACK

    L1_FLUSH --> BC
    L15_JUDGE --> BC
    L2_GEN --> BC

    SM --> ST
    SR --> SM
    L3_MILD --> TT
    L3_AGG --> TT
    L3_EMG --> TT
    CE_ASM --> FT
    ATC --> FT
```

---

## 3. 上下文卸载整体流程

```mermaid
sequenceDiagram
    participant User as 用户
    participant Framework as OpenClaw 框架
    participant CE as Context Engine
    participant L1 as L1 摘要化
    participant L15 as L1.5 任务判断
    participant L2 as L2 Mermaid 生成
    participant L3 as L3 压缩引擎
    participant L4 as L4 Skill 生成

    User->>Framework: 发送消息
    Framework->>CE: bootstrap(sessionKey)
    CE->>CE: resolveIfAllowed → OffloadStateManager

    Framework->>CE: assemble(messages, tokenBudget, prompt)

    Note over CE,L15: ── L1.5 任务判断（异步 fire-and-forget）──
    CE->>L15: judgeL15(stateManager, event)
    L15->>L15: flushL1 (pre-flush 待处理工具对)
    L15->>L15: attemptL15 → backendClient.l15Judge()
    L15->>L15: handleTaskTransition (创建/重新激活/清除 MMD)
    L15->>L15: pushBoundary (记录 long/short 边界)

    Note over CE,L1: ── L1 摘要化（异步 fire-and-forget）──
    CE->>L1: flushL1 (批量发送工具对)
    L1->>L1: writeRefMd (写原始结果)
    L1->>L1: backendClient.l1Summarize()
    L1->>L1: appendOffloadEntries (写 JSONL)

    Note over CE,L3: ── L3 压缩（同步阻塞）──
    CE->>L3: Fast-Path 重应用
    L3->>L3: confirmed → replaceWithSummary
    L3->>L3: deleted → splice 删除

    CE->>L3: Token 快照 + 阈值判断
    alt tokens >= aggressiveThreshold
        L3->>L3: Aggressive 压缩 (头部删除)
        L3->>L3: buildHistoryMmdInjection (历史 MMD 注入)
    end
    alt tokens >= mildThreshold
        L3->>L3: Mild 压缩 (score 级联替换)
    end
    alt tokens >= emergencyThreshold
        L3->>L3: Emergency 压缩 (头部+尾部+截断)
    end

    CE-->>Framework: { messages, estimatedTokens }

    Note over Framework,L2: ── L2 后台轮询（异步）──
    Framework->>L2: notifyL2NewNullEntries (新 null 条目)
    L2->>L2: 轮询检查 null 数量 / 超时
    L2->>L2: checkL2Trigger (边界分组)
    L2->>L2: runL2WithBackend → backendClient.l2Generate()
    L2->>L2: patchMmd / writeMmd (更新 MMD 文件)
    L2->>L2: backfillNodeIds (回填 node_id)

    Note over Framework,L4: ── L4 Skill 生成（/create-skill 命令）──
    User->>Framework: /create-skill <name>
    Framework->>L4: createSkillWithBackend()
    L4->>L4: backendClient.l4Generate()
    L4-->>Framework: appendSystemContext (Skill 内容)
```

---

## 4. L1 摘要化流程

L1 负责将工具调用+结果对发送给 LLM，生成摘要并持久化。

```mermaid
sequenceDiagram
    participant ATC as after_tool_call 钩子
    participant SM as OffloadStateManager
    participant FL as flushL1()
    participant BC as BackendClient / LocalLLM
    participant ST as Storage

    ATC->>SM: addToolPair(pair)
    Note over SM: pendingToolPairs 缓冲区

    ATC->>SM: shouldForceL1()?
    alt pending >= forceTriggerThreshold (默认4)
        ATC->>FL: flushL1("force_threshold", fireAndForget=true)
    end

    Note over FL: ── flushL1 详细流程 ──

    FL->>SM: acquireL1Lock() (互斥锁)
    FL->>SM: takePending() (取出待处理对)
    FL->>FL: 过滤 heartbeat 工具对

    Note over FL,ST: Step 1: 写原始结果到 refs/
    loop 每个 tool pair
        FL->>ST: writeRefMd(timestamp, toolName, content)
        ST-->>FL: refPath (如 "refs/2026-04-12T17-26-08.md")
        Note over FL: refByToolCallId.set(toolCallId, refPath)
    end

    Note over FL,BC: Step 2: 分批发送到 LLM
    FL->>FL: 按 L1_BATCH_SIZE=5 分批
    loop 每个批次
        FL->>BC: l1Summarize({ recentMessages, toolPairs })
        alt 成功
            BC-->>FL: { entries: OffloadEntry[] }
            FL->>FL: 回填 result_ref
            FL->>ST: appendOffloadEntries(entries)
            Note over SM: entryCounter += entries.length
        else 失败 (重试 < 3次)
            FL->>SM: 重新入队 pairs
            Note over SM: _l1ChunkFailCounts++
        else 失败 (重试 >= 3次)
            FL->>FL: 生成本地降级条目
            Note over FL: summary = "[L1 degraded] ..."
            FL->>ST: appendOffloadEntries(fallbackEntries)
        end
    end

    FL->>SM: release() (释放互斥锁)
    FL->>FL: notifyL2NewNullEntries(nullCount)
```

### 4.1 OffloadEntry 数据结构

```typescript
interface OffloadEntry {
  timestamp: string;        // ISO 时间戳
  node_id: string | null;   // L2 分配的 Mermaid 节点 ID
  tool_call: string;        // 工具调用描述
  summary: string;          // LLM 生成的摘要
  result_ref: string;       // 原始结果的 refs/ 路径
  tool_call_id: string;     // 原始工具调用 ID
  session_key?: string;     // 所属会话
  score?: number;           // 可替换性评分 (0-10)
}
```

---

## 5. L1.5 任务边界判断流程

L1.5 判断当前用户消息是否标志着任务切换、延续或新任务开始。

```mermaid
flowchart TD
    START["assemble() / before_prompt_build<br/>收到新 prompt"] --> HASH{"prompt hash<br/>与上次相同?"}
    HASH -->|是| SKIP["SKIP L1.5"]
    HASH -->|否| RESET["重置 l15Settled=false<br/>记录 lastL15PromptHash"]

    RESET --> PRE["L1.5 pre-flush<br/>flushL1(l15_pre_flush)"]
    PRE --> IDX["记录 startIndex = entryCounter"]

    IDX --> ATTEMPT["attemptL15(startIndex)"]
    ATTEMPT --> BUILD["构建 L15Request:<br/>recentMessages + currentMmd + availableMmdMetas"]
    BUILD --> CALL["backendClient.l15Judge(req)"]

    CALL -->|成功| NORM["normalizeJudgment(resp)"]
    CALL -->|失败| RETRY{"重试 1 次<br/>(延迟 3s)"}

    RETRY -->|重试成功| NORM
    RETRY -->|重试失败| FAILSAFE["L1.5 Fail-Safe:<br/>activeMmd=null<br/>boundary=short<br/>l15Settled=true"]

    NORM --> JUDGE{"TaskJudgment 解析"}
    JUDGE --> TRANS["handleTaskTransition(judgment)"]

    TRANS --> TC{taskCompleted?}
    TC -->|是 + isContinuation| REACT["reactivateMmd(continuationMmdFile)"]
    TC -->|是 + isLongTask + newLabel| CREATE["createNewMmd(newLabel)"]
    TC -->|是 + 短任务| CLEAR["setActiveMmd(null, null)"]
    TC -->|否 + isContinuation| REACT2["reactivateMmd<br/>(如有 continuationMmdFile)"]
    TC -->|否 + isLongTask + newLabel| CREATE2["createNewMmd(newLabel)"]
    TC -->|否 + 继续| KEEP["保持当前 activeMmd"]

    REACT --> BOUND
    CREATE --> BOUND
    CLEAR --> BOUND
    REACT2 --> BOUND
    CREATE2 --> BOUND
    KEEP --> BOUND

    BOUND["pushBoundary({startIndex, result, targetMmd})"]
    BOUND --> SAVE["stateManager.save()"]
    SAVE --> READY["setMmdInjectionReady(true)<br/>l15Settled=true"]

    FAILSAFE --> DONE["L1.5 完成"]
    READY --> DONE

    style FAILSAFE fill:#f66,stroke:#333,color:#fff
    style READY fill:#6f6,stroke:#333
```

### 5.1 TaskJudgment 数据结构

```typescript
interface TaskJudgment {
  taskCompleted: boolean;       // 当前任务是否完成
  isContinuation: boolean;     // 是否为近期任务的延续
  continuationMmdFile?: string;// 延续的 MMD 文件名
  newTaskLabel?: string;       // 新任务标签（用于 MMD 文件名）
  isLongTask: boolean;         // 是否为长任务（vs 闲聊）
}
```

### 5.2 L15Boundary 数据结构

```typescript
interface L15Boundary {
  startIndex: number;              // entryCounter 值
  result: "long" | "short" | "pending";
  targetMmd: string | null;       // 目标 MMD 文件
}
```

---

## 6. L2 Mermaid 生成流程

L2 独立于 L1 运行，通过后台轮询调度器触发。

```mermaid
sequenceDiagram
    participant POLL as L2 轮询调度器
    participant CHECK as checkL2Trigger()
    participant SM as OffloadStateManager
    participant RUN as runL2WithBackend()
    participant BC as BackendClient
    participant ST as Storage

    Note over POLL: 每 5s 轮询一次

    POLL->>SM: l15Settled?
    alt 未 settled
        alt 等待 > 60s
            POLL->>SM: force-settle l15Settled=true
        else
            POLL-->>POLL: 等待下次轮询
        end
    end

    POLL->>ST: readAllOffloadEntries()
    POLL->>POLL: 统计 node_id=null 的条目数

    alt nullCount >= l2NullThreshold (默认4)
        POLL->>CHECK: 条件 A: null 阈值触发
    else 等待时间 >= l2TimeoutSeconds (默认300s)
        POLL->>CHECK: 条件 B: 超时触发
    else
        POLL-->>POLL: 等待下次轮询
    end

    CHECK->>SM: resolveEntryBoundary(i) (逐条)
    Note over CHECK: 按 boundary 分组:<br/>result="long" → targetMmd<br/>result="short" → 跳过

    CHECK->>CHECK: 按 MMD 文件分组条目
    CHECK-->>RUN: entriesByMmd: Map<mmdFile, entries[]>

    loop 每个 MMD 文件
        RUN->>RUN: 按 L2_BATCH_SIZE=30 分批
        loop 每个批次
            RUN->>ST: 标记 node_id="wait"
            RUN->>ST: readMmd(existingMmd)

            RUN->>BC: l2Generate({ existingMmd, newEntries, taskLabel, ... })

            alt fileAction="replace"
                RUN->>ST: patchMmd(mmdFile, replaceBlocks)
                alt patchMmd 失败 && mmdContent 存在
                    RUN->>ST: writeMmd(mmdFile, mmdContent) (全量回退)
                end
            else fileAction="write"
                RUN->>ST: writeMmd(mmdFile, mmdContent)
            end

            RUN->>ST: readMmd(mmdFile) (读取更新后内容)
            RUN->>RUN: backfillNodeIds(nodeMapping)
            Note over RUN: 将 tool_call_id → node_id<br/>映射写回 offload.jsonl
        end
    end

    RUN->>POLL: 检查剩余 null 条目
    alt 仍有 nullCount >= threshold
        POLL->>POLL: 再次触发 L2
    else 仍有 nullCount > 0
        POLL->>POLL: 重新启动轮询
    else nullCount = 0
        POLL->>POLL: 停止轮询
    end
```

---

## 7. L3 压缩算法设计

L3 是同步阻塞式压缩，在三个触发点执行：`assemble()`、`after_tool_call`、`before_prompt_build`。

```mermaid
flowchart TD
    START["L3 压缩入口"] --> FP["Fast-Path 重应用"]

    FP --> FP_DEL{"deletedOffloadIds<br/>非空?"}
    FP_DEL -->|是| DEL["删除已标记消息<br/>（splice by toolCallId）"]
    FP_DEL -->|否| FP_CON
    DEL --> FP_CON{"confirmedOffloadIds<br/>非空?"}
    FP_CON -->|是| REPL["replaceWithSummary<br/>（tool result → 摘要）"]
    FP_CON -->|否| FP_DONE
    REPL --> FP_DONE["Fast-Path 完成"]

    FP_DONE --> EST{"快速 Token 估算<br/>fastEstimateMessages()"}
    EST -->|fastEst < aggressive × 0.85| SKIP_TIK["跳过 tiktoken<br/>直接用估算值"]
    EST -->|fastEst >= aggressive × 0.85| TIK["精确 tiktoken 计数<br/>buildTiktokenContextSnapshot()"]

    SKIP_TIK --> AGG_CHECK
    TIK --> AGG_CHECK{"tokens >=<br/>aggressiveThreshold<br/>(默认 85%)"}

    AGG_CHECK -->|否| MILD_CHECK
    AGG_CHECK -->|是| AGG["Aggressive 压缩"]

    subgraph "Aggressive 压缩策略"
        AGG --> AGG_TAIL{"有 boundary 缓存?"}
        AGG_TAIL -->|否| TAIL_ACC["TAIL-ACCUMULATE<br/>从尾部累积 token<br/>直到 60% budget<br/>删除头部"]
        AGG_TAIL -->|是| STD_AGG["标准 Aggressive<br/>computeAggressiveDeleteCount<br/>一次性计算删除数量"]

        TAIL_ACC --> AGG_SPLICE["splice(0, keepFrom)<br/>删除头部消息"]
        STD_AGG --> AGG_SPLICE2["splice(0, deleteCount)<br/>删除头部消息"]

        AGG_SPLICE --> AGG_MMD["buildHistoryMmdInjection<br/>注入历史 MMD"]
        AGG_SPLICE2 --> AGG_MMD

        AGG_MMD --> AGG_BOUND["记录 _lastAggressiveBoundary<br/>(originalIndex + fingerprint)"]
    end

    AGG_BOUND --> MILD_CHECK
    AGG_SPLICE --> MILD_CHECK
    AGG_SPLICE2 --> MILD_CHECK

    MILD_CHECK{"tokens >=<br/>mildThreshold<br/>(默认 50%)"} -->|否| EMG_CHECK
    MILD_CHECK -->|是| MILD["Mild 压缩"]

    subgraph "Mild 压缩策略 (score 级联)"
        MILD --> SCAN["扫描 scanRatio=70% 范围内的消息"]
        SCAN --> CAND["收集候选: toolResult + assistant(tool_use)<br/>按 score 降序排列"]
        CAND --> CASCADE["级联替换: score 7→6→5→...→1"]
        CASCADE --> REPLACE["replaceWithSummary(msg, entry)<br/>tool result → 摘要文本"]
        REPLACE --> ASST["replaceAssistantToolUseWithSummary<br/>tool_use input → 紧凑摘要"]
        ASST --> MIN{"replacedCount >=<br/>MILD_CASCADE_MIN_COUNT (10)?"}
        MIN -->|是| MILD_DONE["Mild 完成"]
        MIN -->|否| CASCADE
    end

    MILD_DONE --> EMG_CHECK
    CASCADE -->|"score 降至 1"| MILD_DONE2["Mild 完成 (最低级)"]
    MILD_DONE2 --> EMG_CHECK

    EMG_CHECK{"tokens >=<br/>emergencyThreshold<br/>(默认 95%)<br/>或 _forceEmergencyNext?"} -->|否| DONE["L3 完成"]
    EMG_CHECK -->|是| EMG["Emergency 压缩"]

    subgraph "Emergency 压缩策略"
        EMG --> HEAD["头部删除<br/>按 excessRatio 计算删除量"]
        HEAD --> HEAD_OK{"头部删除成功?"}
        HEAD_OK -->|是| EMG_LOOP{"tokens <= target?"}
        HEAD_OK -->|否 (用户消息阻挡)| TAIL_DEL["尾部删除<br/>按 token 大小删除最大工具对组"]
        TAIL_DEL --> TAIL_OK{"尾部删除成功?"}
        TAIL_OK -->|否| TRUNC["原地截断<br/>最大消息 → 短桩文本"]
        TAIL_OK -->|是| EMG_LOOP
        TRUNC --> EMG_LOOP
        EMG_LOOP -->|否| HEAD
        EMG_LOOP -->|是| EMG_DONE["Emergency 完成"]
    end

    EMG_DONE --> DONE

    style EMG fill:#f66,stroke:#333,color:#fff
    style AGG fill:#fa0,stroke:#333
    style MILD fill:#6f6,stroke:#333
```

### 7.1 三级压缩对比

| 特性 | Mild | Aggressive | Emergency |
|------|------|------------|-----------|
| **触发阈值** | 50% contextWindow | 85% contextWindow | 95% contextWindow |
| **目标** | 替换可替换的工具结果 | 删除最旧的消息前缀 | 删除至 60% contextWindow |
| **操作** | tool result → 摘要文本 | splice(0, N) 头部删除 | 头部+尾部+原地截断 |
| **信息保留** | 保留摘要 + result_ref | 保留 MMD 流程图 | 仅保留最小消息集 |
| **可逆性** | 可通过 result_ref 恢复 | 不可逆 | 不可逆 |
| **用户消息保护** | N/A | 保护最后一条用户消息 | 保护最后一条用户消息 |
| **工具对完整性** | 保持 tool_use/tool_result 配对 | 调整删除边界保持配对 | 尾部删除按工具对组删除 |

---

## 8. MMD 注入机制

MMD 注入分为两种模式：全量注入（assemble/before_prompt_build）和增量更新（after_tool_call）。

```mermaid
sequenceDiagram
    participant ASM as assemble() / before_prompt_build
    participant ATC as after_tool_call
    participant MI as MmdInjector
    participant SM as OffloadStateManager
    participant ST as Storage

    Note over ASM,MI: ── 全量注入 (每轮用户消息) ──

    ASM->>MI: injectMmdIntoMessages(messages, waitForL15=true)
    MI->>SM: l15Settled?
    alt 未 settled
        MI-->>ASM: 跳过注入 (保留已有 MMD)
    end

    MI->>SM: isMmdInjectionReady()?
    alt 未 ready
        MI->>MI: removeMmdMessages()
        MI-->>ASM: mmdTokens=0
    end

    MI->>SM: getActiveMmdFile()
    MI->>ST: readMmd(activeMmdFile)
    MI->>MI: buildActiveMmdBlock()<br/>构建 <current_task_context> 文本

    MI->>MI: removeMmdMessages() (清除旧注入)
    MI->>MI: findActiveMmdInsertionPoint(messages)
    Note over MI: 插入策略:<br/>1. 最新 user 消息之后<br/>2. 避免拆分 tool_use/tool_result 对<br/>3. 不超过尾部 30 条

    MI->>MI: messages.splice(insertIdx, 0, activeMsg)
    Note over MI: activeMsg._mmdContextMessage = "active"

    MI-->>ASM: { mmdTokens }

    Note over ATC,MI: ── 增量更新 (每次工具调用) ──

    ATC->>SM: l15Settled && activeMmdFile?
    alt 已 settled 且有 activeMmd
        ATC->>ST: readMmd(activeMmdFile)
        ATC->>ATC: 构建 MMD 文本

        ATC->>ATC: 查找已有 _mmdContextMessage="active"
        alt 已存在
            ATC->>ATC: 比较内容是否变化
            alt 内容变化 (L2 更新)
                ATC->>ATC: 替换已有消息
            else 无变化
                ATC-->>ATC: 跳过
            end
        else 不存在
            ATC->>ATC: findActiveMmdInsertionPoint()
            ATC->>ATC: messages.splice(insertIdx, 0, newMsg)
        end
    end

    Note over ASM,MI: ── 历史 MMD 注入 (Aggressive 删除后) ──

    ASM->>MI: buildHistoryMmdInjection(deletedToolCallIds)
    MI->>MI: 收集被删除条目关联的 MMD 前缀
    MI->>ST: listMmds() (列出所有 MMD 文件)
    MI->>MI: 过滤: 属于被删除前缀 && 非当前 activeMmd

    loop 每个候选 MMD (最新优先)
        MI->>ST: readMmd(filename)
        alt 全量内容 <= mmdTokenBudget (20% contextWindow)
            MI->>MI: 注入完整 MMD (含 mermaid 代码块)
        else 仅元信息 <= budget
            MI->>MI: 注入 meta-only (节点摘要+状态)
        else 超出预算
            MI->>MI: 跳过此 MMD
        end
    end

    MI->>MI: removeExistingMmdInjections()
    MI->>MI: findHistoryMmdInsertionPoint()
    Note over MI: 插入位置: active MMD 之前<br/>顺序: 最旧 → 最新

    MI->>MI: messages.splice(histInsertIdx, 0, ...injectedMessages)
    Note over MI: injectedMsg._mmdInjection = true
```

---

## 9. 会话管理设计

```mermaid
classDiagram
    class SessionRegistry {
        -_sessions: Map~string, SessionCtx~
        -_dataRoot: string
        +_registryId: number
        +resolve(sessionKey, realSessionId?) SessionCtx
        +resolveIfAllowed(sessionKey, realSessionId?) SessionCtx|null
        +get(sessionKey) SessionCtx|undefined
        +size: number
        +keys(): IterableIterator~string~
        +values(): IterableIterator~SessionCtx~
        -_evictOldest(): void
    }

    class SessionCtx {
        +sessionKey: string
        +manager: OffloadStateManager
        +lastAccessMs: number
    }

    class OffloadStateManager {
        -_ctx: StorageContext|null
        +pendingToolPairs: ToolPair[]
        +processedToolCallIds: Set~string~
        +confirmedOffloadIds: Set~string~
        +deletedOffloadIds: Set~string~
        +l15Settled: boolean
        +l15Boundaries: L15Boundary[]
        +entryCounter: number
        +_lastAggressiveBoundary: object|null
        +_forceEmergencyNext: boolean
        +cachedSystemPrompt: string|null
        +cachedUserPrompt: string|null
        +cachedSystemPromptTokens: number|null
        +cachedRecentHistory: string|null
        +lastL15PromptHash: number|null
        +init(dataRoot, agentName, sessionId): Promise~void~
        +save(): Promise~void~
        +addToolPair(pair): void
        +takePending(max): ToolPair[]
        +acquireL1Lock(): Promise~Function~
        +switchSession(sessionKey, dataRoot, realSessionId?): Promise~boolean~
        +pushBoundary(boundary): void
        +resolveEntryBoundary(entryIndex): L15Boundary|null
    }

    class StorageContext {
        <<interface>>
        +dataRoot: string
        +dataDir: string
        +refsDir: string
        +mmdsDir: string
        +offloadJsonl: string
        +stateFile: string
        +agentName: string
        +sessionId: string
    }

    class Storage {
        +createStorageContext(dataRoot, agentName, sessionId): StorageContext
        +ensureDirs(ctx): Promise~void~
        +appendOffloadEntries(ctx, entries): Promise~void~
        +readOffloadEntries(ctx): Promise~OffloadEntry[]~
        +readAllOffloadEntries(ctx): Promise~OffloadEntry[]~
        +rewriteAllOffloadEntries(ctx, entries): Promise~void~
        +markOffloadStatus(ctx, updates): Promise~void~
        +writeRefMd(ctx, timestamp, toolName, content): Promise~string~
        +writeMmd(ctx, filename, content): Promise~void~
        +readMmd(ctx, filename): Promise~string|null~
        +patchMmd(ctx, filename, blocks): Promise~boolean~
        +listMmds(ctx): Promise~string[]~
        +readStateFile(ctx, defaultValue): Promise~T~
        +writeStateFile(ctx, state): Promise~void~
        +parseSessionKey(sessionKey): object|null
    }

    SessionRegistry "1" --> "*" SessionCtx : routes
    SessionCtx --> OffloadStateManager : holds
    OffloadStateManager --> StorageContext : uses
    Storage ..> StorageContext : creates
    OffloadStateManager ..> Storage : reads/writes
```

### 9.1 存储路径隔离

```
~/.openclaw/context-offload/
└── {agentName}/                          # 每个 Agent 独立目录
    ├── offload-{sessionId}.jsonl         # 每会话独立 JSONL
    ├── offload-{sessionId2}.jsonl
    ├── state.json                        # Agent 级共享状态
    ├── sessions-registry.json            # sessionKey → sessionId 映射
    ├── refs/                             # 工具结果原始文件
    │   └── 2026-04-12T17-26-08-123p08-00.md
    └── mmds/                             # Mermaid 流程图文件
        ├── 001-database-debug.mmd
        ├── 002-api-refactor.mmd
        └── 003-perf-optimization.mmd
```

### 9.2 LRU 淘汰策略

- `SessionRegistry` 最多缓存 20 个会话（`MAX_CACHED_SESSIONS`）
- 每次访问更新 `lastAccessMs`
- 超出上限时淘汰最久未访问的会话

---

## 10. Token 计数设计

### 10.1 双层计数架构

```mermaid
flowchart LR
    subgraph "快速估算层 (~5ms)"
        FE["fastEstimateMessages()"]
        FE_CJK["CJK 查表<br/>(cjk_token_table.bin)"]
        FE_LAT["Latin 词长系数"]
        FE_DIG["数字/标点系数"]
        FE --> FE_CJK
        FE --> FE_LAT
        FE --> FE_DIG
    end

    subgraph "精确计数层 (~3-10s)"
        TT["buildTiktokenContextSnapshot()"]
        TT_ENC["js-tiktoken<br/>(o200k_base / cl100k_base)"]
        TT_CACHE["WeakMap 每消息缓存"]
        TT --> TT_ENC
        TT --> TT_CACHE
    end

    subgraph "启发式估算层"
        HE["l3-token-helpers.ts"]
        HE_CJK["CJK × 1.7"]
        HE_REST["其余 ÷ 4"]
        HE --> HE_CJK
        HE --> HE_REST
    end

    FE -->|"fastEst < aggressive × 0.85"| SKIP["跳过 tiktoken"]
    FE -->|"fastEst >= aggressive × 0.85"| TT
    TT -->|"l3TokenCountMode=tiktoken"| TT_ENC
    TT -->|"l3TokenCountMode=heuristic"| HE
```

### 10.2 快速估算算法

`fastEstimateTokens()` 采用单遍字符分类 + 每类别系数：

| 字符类别 | 系数 | 示例 |
|----------|------|------|
| CJK 汉字 | 查表 (1-3 tok/char) | 中文汉字 |
| Latin 单词 | 按词长分段 (1.0-1.5+0.3/char) | English |
| 日文假名 | 按连续长度分段 | あいう |
| 韩文 | 1.4 tok/char | 한글 |
| 西里尔 | 0.55 tok/char | Русский |
| 阿拉伯 | 0.82 tok/char | العربية |
| 数字 | 按格式分段 | 1,234.56 |
| ASCII 标点 | 0.6 tok/char | {}[]() |
| 其他 Unicode | 2.5 tok/char | Emoji |

### 10.3 性能优化策略

1. **Quick-SKIP**：`after_tool_call` 中使用启发式估算，连续 5 次跳过后强制精确计算
2. **Boundary 增量估算**：有 `_lastAggressiveBoundary` 时，仅估算新增消息的 token 增量
3. **WeakMap 缓存**：每消息对象缓存 token 计数，内容变更时自动失效
4. **JSON Replacer**：序列化时剔除 `_offloaded`/`_mmdContextMessage` 等内部标记

---

## 11. 容错与降级设计

### 11.1 L1 容错

```mermaid
flowchart TD
    L1_CALL["L1 后端调用"] -->|成功| L1_OK["写入 OffloadEntry"]
    L1_CALL -->|失败| L1_RETRY{"重试次数 < 3?"}
    L1_RETRY -->|是| L1_REQUEUE["重新入队 pendingToolPairs<br/>_l1ChunkFailCounts++"]
    L1_RETRY -->|否| L1_DEGRADE["降级: 生成本地 fallback 条目<br/>summary = '[L1 degraded] ...'"]
    L1_REQUEUE --> L1_CALL2["下次 flushL1 重试"]
    L1_CALL2 -->|成功| L1_OK
    L1_CALL2 -->|失败| L1_RETRY

    style L1_DEGRADE fill:#fa0,stroke:#333
```

### 11.2 L1.5 容错

```mermaid
flowchart TD
    L15_CALL["L1.5 后端调用"] -->|成功| L15_OK["handleTaskTransition + pushBoundary"]
    L15_CALL -->|失败| L15_RETRY["延迟 3s 后重试 1 次"]
    L15_RETRY -->|成功| L15_OK
    L15_RETRY -->|失败| L15_FAILSAFE["Fail-Safe 激活:<br/>1. activeMmd = null<br/>2. boundary = short<br/>3. l15Settled = true<br/>4. MMD 注入禁用"]

    style L15_FAILSAFE fill:#f66,stroke:#333,color:#fff
```

**Fail-Safe 影响**：
- L2 不会触发（无 activeMmd）
- 所有 null 条目标记为 short（不参与未来 L2）
- 当前轮次无 MMD 构建
- L3 压缩仍正常工作

### 11.3 L2 容错

```mermaid
flowchart TD
    L2_CALL["L2 后端调用"] -->|成功| L2_OK["patchMmd / writeMmd + backfillNodeIds"]
    L2_CALL -->|失败| L2_SKIP["跳过此批次<br/>条目保持 node_id='wait'"]
    L2_CALL -->|降级响应<br/>(fileAction 为空)| L2_FALLBACK["fallback backfill<br/>使用 MMD 中最大 node_id"]
    L2_SKIP --> L2_RETRY["下次轮询重试<br/>(wait 条目超时后)"]

    style L2_FALLBACK fill:#fa0,stroke:#333
```

### 11.4 L3 容错

| 场景 | 处理策略 |
|------|----------|
| Aggressive 被用户消息阻挡 | `stalledByUserMsg=true` → 强制 Emergency |
| Emergency 头部删除被阻挡 | 尾部删除最大工具对组 |
| 尾部删除仍不足 | 原地截断最大消息（保留 tool_use 结构） |
| Token 溢出错误检测 | `isTokenOverflowError()` → `_forceEmergencyNext=true` |
| tiktoken 计数超时 | 回退到 `Math.ceil(text.length / 4)` |

### 11.5 降级模式

| 模式 | L1 | L1.5 | L2 | L3 | L4 | 说明 |
|------|----|------|----|----|----|------|
| **backend** | ✅ | ✅ | ✅ | ✅ | ✅ | 完整功能 |
| **local** | ✅ | ✅ | ✅ | ✅ | ✅ | 使用本地 LLM |
| **collect** | ✅ | ✅ | ✅ | ❌ | ❌ | 仅收集数据，不压缩 |
| **无 LLM 客户端** | ❌ | ❌ | ❌ | ✅ | ❌ | 仅 L3 压缩可用 |

### 11.6 数据回收

`Reclaimer` 每 24 小时执行 5 步清理：

```mermaid
flowchart LR
    S1["Step 1: 过期 JSONL<br/>(mtime > retentionDays)"]
    S2["Step 2: 孤立 refs/<br/>(未被 JSONL 引用)"]
    S3["Step 3: 过期 MMD<br/>(保留最近 15 个 + activeMmd)"]
    S4["Step 4: 日志轮转<br/>(超过 logMaxSizeMb 截断)"]
    S5["Step 5: 注册表修剪<br/>(过期 sessionKey 条目)"]

    S1 --> S2 --> S3 --> S4 --> S5
```

### 11.7 JSONL 防御层

```mermaid
flowchart TD
    RAW["原始文本"] --> L0["Layer 0: sanitizeText()<br/>剥离 UNSAFE_CHAR_RE"]
    L0 --> L1["Layer 1: sanitizeJsonLine()<br/>写入时清洗 + JSON 往返验证"]
    L1 --> L2["Layer 2: parseJsonlSafe()<br/>容错解析 (跳过损坏行)"]
    L2 --> L3["Layer 3: validateEntry()<br/>schema 验证 (tool_call_id 必填)"]
    L3 --> CLEAN["干净的 OffloadEntry[]"]

    L2 -->|corruptCount > 0| WARN["记录损坏行数 + 样本"]
    L3 -->|invalidCount > 0| WARN2["记录无效行数"]
```
