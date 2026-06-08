# 记忆函数层（Memory Functions Layer）模块分析

## 1. 模块概述

记忆函数层是 agentmemory 的核心业务逻辑层，位于 iii-sdk 引擎之上、MCP/REST 接口之下。它通过 `sdk.registerFunction()` 注册 50+ 个 `mem::` 命名空间的函数，覆盖记忆的完整生命周期：**观察捕获 → 压缩 → 存储 → 搜索 → 整合 → 结晶 → 上下文注入 → 遗忘/驱逐**。

所有函数遵循统一范式：
- 通过 `sdk.registerFunction(functionId, handler)` 注册
- 通过 `sdk.trigger({ function_id, payload })` 跨函数调用
- 状态读写通过 `StateKV`（底层为 iii-engine 的 StateModule，文件型 SQLite）
- 结构性变更通过 `recordAudit()` 记录审计日志
- 并发安全通过 `withKeyedLock()` 保证

### 模块文件清单

| 文件 | 核心函数 ID | 职责 |
|------|------------|------|
| `observe.ts` | `mem::observe` | 观察捕获 |
| `remember.ts` | `mem::remember`, `mem::forget` | 记忆存储与删除 |
| `compress.ts` | `mem::compress` | LLM 压缩 |
| `compress-synthetic.ts` | —（被 observe 调用） | 零 LLM 合成压缩 |
| `consolidate.ts` | `mem::consolidate` | 记忆整合 |
| `consolidation-pipeline.ts` | `mem::consolidate-pipeline` | 整合管道 |
| `search.ts` | `mem::search` | 搜索（BM25 + 向量） |
| `smart-search.ts` | `mem::smart-search` | 智能搜索 |
| `context.ts` | `mem::context` | 上下文注入 |
| `summarize.ts` | `mem::summarize` | 会话摘要 |
| `crystallize.ts` | `mem::crystallize`, `mem::crystal-list`, `mem::crystal-get`, `mem::auto-crystallize` | 结晶化 |
| `reflect.ts` | `mem::reflect`, `mem::insight-list`, `mem::insight-search`, `mem::insight-decay-sweep` | 反思 |
| `enrich.ts` | `mem::enrich` | 丰富化 |
| `evict.ts` | `mem::evict` | 驱逐 |
| `auto-forget.ts` | `mem::auto-forget` | 自动遗忘 |
| `retention.ts` | `mem::retention-score`, `mem::retention-evict` | 保留评分 |
| `dedup.ts` | —（类 DedupMap） | 去重 |
| `privacy.ts` | `mem::privacy` | 隐私清洗 |
| `audit.ts` | —（工具函数） | 审计 |

---

## 2. 核心函数详解

### 2.1 observe — 观察捕获

**职责**：从 Hook 脚本接收原始观察数据，完成去重、隐私清洗、图像提取、KV 持久化、流推送和压缩触发。

**注册函数 ID**：`mem::observe`

**输入**（HookPayload）：
```typescript
{
  sessionId: string;      // 必需
  hookType: string;       // 必需，如 "post_tool_use"、"prompt_submit"
  timestamp: string;      // 必需，ISO 8601
  data: unknown;          // 原始观察数据
  project?: string;
  cwd?: string;
}
```

**输出**：
```typescript
{ observationId: string }  // 成功
{ deduplicated: true, sessionId: string }  // 去重跳过
{ success: false, error: string }  // 失败
```

**核心算法**：
1. **输入校验**：验证 sessionId、hookType、timestamp 必需字段
2. **去重检查**：通过 DedupMap 计算 SHA-256 哈希（sessionId + toolName + toolInput 前 500 字符），5 分钟 TTL 内重复则跳过
3. **隐私清洗**：`stripPrivateData()` 移除 `<private>` 标签和密钥模式（API Key、Bearer Token、JWT 等）
4. **图像提取**：递归搜索 `image_data`、`imageBase64` 等字段，base64 图像存入磁盘并记录引用计数
5. **会话感知**：继承已有会话的 agentId，无会话记录时从环境变量获取
6. **KV 持久化**：写入 `KV.observations(sessionId)` 作用域
7. **流推送**：通过 `stream::set` 和 `stream::send` 推送原始观察到实时查看器
8. **压缩分支**：
   - `AGENTMEMORY_AUTO_COMPRESS=true` → 触发 `mem::compress`（LLM 压缩）
   - 默认 → `buildSyntheticCompression()`（零 LLM 合成压缩），直接更新 KV 并索引

### 2.2 remember — 记忆存储

**职责**：将结构化内容保存为长期记忆，支持版本演进（supersede）和级联更新。

**注册函数 ID**：`mem::remember`、`mem::forget`

**输入**（remember）：
```typescript
{
  content: string;           // 必需
  type?: string;             // pattern|preference|architecture|bug|workflow|fact，默认 fact
  concepts?: string[];
  files?: string[];
  ttlDays?: number;          // 生存天数，过期后 forgetAfter 生效
  sourceObservationIds?: string[];
  agentId?: string;
  project?: string;
}
```

**输出**：`{ success: true, memory: Memory }` 或 `{ success: false, error: string }`

**核心算法**：
1. **输入校验**：content 必需且非空，files/concepts 必须为数组
2. **版本演进检测**：遍历所有 `isLatest` 记忆，计算 Jaccard 相似度（> 0.7 阈值），同项目内匹配则标记旧记忆为 `isLatest=false`，新记忆 `version = oldVersion + 1`
3. **项目隔离**：显式 project 字段的记忆不会跨项目 supersede
4. **Agent 标记**：优先使用请求中的 agentId，回退到环境变量 `AGENT_ID`
5. **TTL 设置**：`ttlDays` 转换为 `forgetAfter` 绝对时间戳
6. **索引更新**：同步写入 BM25 索引和向量索引
7. **级联更新**：supersede 时触发 `mem::cascade-update`

**mem::forget** 支持三种粒度删除：
- 按 `memoryId` 删除单条记忆
- 按 `sessionId + observationIds` 删除指定观察
- 按 `sessionId` 删除整个会话（含所有观察 + 摘要）

### 2.3 compress — LLM 压缩

**职责**：将原始观察通过 LLM 调用压缩为结构化 CompressedObservation。

**注册函数 ID**：`mem::compress`

**输入**：
```typescript
{
  observationId: string;
  sessionId: string;
  raw: RawObservation;
}
```

**输出**：`{ success: true, compressed: CompressedObservation, qualityScore: number }` 或 `{ success: false, error: string }`

**核心算法**：
1. **图像描述**：若观察含图像且 provider 支持 `describeImage`，先调用视觉模型生成描述
2. **Prompt 构建**：`buildCompressionPrompt()` 组装 hookType、toolName、toolInput/Output、userPrompt
3. **LLM 调用 + 自纠错**：`compressWithRetry()` 调用 LLM，验证器检查 XML 解析和 schema 合规性，失败则重试一次
4. **XML 解析**：提取 type、title、subtitle、facts、narrative、concepts、files、importance
5. **质量评分**：`scoreCompression()` 计算压缩质量分数（0-100）
6. **持久化 + 索引**：更新 KV、BM25 索引、向量索引
7. **流推送**：推送压缩结果到实时查看器
8. **指标记录**：延迟和成功率写入 MetricsStore

### 2.4 compress-synthetic — 零 LLM 合成压缩

**职责**：不调用 LLM，通过启发式规则将 RawObservation 转换为 CompressedObservation。自 v0.8.8 起为默认压缩路径。

**注册函数 ID**：无（被 `mem::observe` 直接调用）

**核心算法**：
1. **类型推断** `inferType()`：根据 hookType 和 toolName 模式匹配推断观察类型（如 `post_tool_failure` → error，含 "fetch" 的工具 → web_fetch）
2. **文件提取** `extractFiles()`：从 toolInput 对象中提取 `file_path`、`filepath`、`path` 等字段
3. **叙事拼接**：将 userPrompt + toolInput + toolOutput 用 `|` 连接，截断至 400 字符
4. **置信度**：固定 0.3（vs LLM 压缩的质量评分 / 100）
5. **重要性**：固定 5

### 2.5 consolidate — 记忆整合

**职责**：将相关观察按概念分组，通过 LLM 合成为长期记忆，支持新记忆创建和已有记忆演进。

**注册函数 ID**：`mem::consolidate`

**输入**：
```typescript
{
  project?: string;          // 项目过滤
  minObservations?: number;  // 最少观察数，默认 10
}
```

**输出**：`{ consolidated: number, totalObservations: number }` 或 `{ consolidated: 0, reason: "insufficient_observations" }`

**核心算法**：
1. **观察收集**：按项目过滤会话，批量加载观察（每批 10 个会话），筛选 importance ≥ 5
2. **概念分组**：按 concept 字段分组，仅保留 ≥ 3 条观察的组
3. **LLM 调用**：每组取 importance 最高的 8 条观察，30 秒超时，最多 10 次 LLM 调用
4. **XML 解析**：提取 type、title、content、concepts、files、strength
5. **去重检查**：与已有记忆按 title 匹配（同项目内），匹配则演进（version + 1），否则新建
6. **审计记录**：演进记录为 "evolve"，新建记录为 "remember"

### 2.6 consolidation-pipeline — 整合管道

**职责**：编排四阶段整合流程：语义合并 → 反思 → 程序提取 → 衰减。

**注册函数 ID**：`mem::consolidate-pipeline`

**输入**：
```typescript
{
  tier?: string;    // "all"|"semantic"|"reflect"|"procedural"|"decay"
  force?: boolean;  // 跳过启用检查
  project?: string;
}
```

**输出**：`{ success: true, results: Record<string, unknown> }`

**核心算法（四阶段）**：

**阶段一：语义合并（semantic）**
- 前提：≥ 5 条会话摘要
- 取最近 20 条摘要，LLM 提取 `<fact confidence="...">` 标签
- 已有事实：增加 accessCount，取最大 confidence
- 新事实：创建 SemanticMemory 条目

**阶段二：反思（reflect）**
- 委托 `mem::reflect` 函数

**阶段三：程序提取（procedural）**
- 前提：≥ 2 条频率 ≥ 2 的 pattern 类型记忆
- LLM 提取 `<procedure name="..." trigger="...">` 标签
- 已有程序：增加 frequency，提升 strength
- 新程序：创建 ProceduralMemory 条目

**阶段四：衰减（decay）**
- 对语义记忆和程序记忆应用指数衰减：`strength *= 0.9^decayPeriods`
- 衰减周期 = 自上次访问以来的天数 / decayDays

**可选**：若 `OBSIDIAN_AUTO_EXPORT=true`，触发 Obsidian 导出

### 2.7 search — 搜索

**职责**：BM25 全文搜索 + 向量语义搜索，支持项目/目录过滤和 token 预算控制。

**注册函数 ID**：`mem::search`

**输入**：
```typescript
{
  query: string;           // 必需
  limit?: number;          // 默认 20，上限 100
  project?: string;
  cwd?: string;
  format?: string;         // "full"|"compact"|"narrative"
  token_budget?: number;
}
```

**核心算法**：
1. **索引懒加载**：若 BM25 索引为空，触发 `rebuildIndex()` 重建
2. **过度获取**：有项目/目录过滤时，获取 10× limit 的候选结果
3. **后过滤**：按 session.project / session.cwd 过滤，记忆条目回查 KV.memories 获取 project
4. **结果丰富**：并行加载观察详情，记忆条目通过 `memoryToObservation()` 转换
5. **访问追踪**：`recordAccessBatch()` 记录被访问的条目 ID
6. **Token 预算**：按 `JSON.stringify().length / 3` 估算 token 数，超预算截断

**辅助功能**：
- `rebuildIndex()`：全量重建 BM25 + 向量索引，批量嵌入（默认 32 条/批）
- `vectorIndexAddGuarded()`：单条向量写入，维度不匹配或嵌入失败时软降级
- `vectorIndexAddBatchGuarded()`：批量向量写入，单次 embedBatch 调用

### 2.8 smart-search — 智能搜索

**职责**：混合搜索（BM25 + 向量 + Lesson 回忆），支持 Agent 作用域隔离和 followup 率诊断。

**注册函数 ID**：`mem::smart-search`

**输入**：
```typescript
{
  query?: string;
  expandIds?: Array<string | { obsId: string; sessionId: string }>;
  limit?: number;
  project?: string;
  includeLessons?: boolean;
  agentId?: string;        // "*" 表示全部
  sessionId?: string;      // followup 诊断锚点
  source?: string;         // "viewer" 排除诊断
}
```

**核心算法**：
1. **expandIds 分支**：直接按 ID 加载观察，应用 agentId 过滤
2. **混合搜索**：调用外部 `searchFn`（BM25 + 向量混合），过度获取 3× limit 后按 agentId 过滤
3. **Lesson 回忆**：并行调用 `mem::lesson-recall`，项目匹配的 lesson 权重 ×1.5
4. **Followup 诊断**（#771）：
   - 仅 agent 发起的搜索（source ≠ "viewer"）且有 sessionId
   - 与同 session 上次搜索比较：窗口内（默认 60s）、不同查询、结果集无交集 → 计为 followup
   - 指标用于检测 "检索失败 → 重新搜索" 模式

### 2.9 context — 上下文注入

**职责**：根据项目上下文组装记忆块，在 token 预算内注入到 Agent 提示词中。

**注册函数 ID**：`mem::context`

**输入**：
```typescript
{
  sessionId: string;
  project: string;
  budget?: number;         // token 预算，覆盖默认值
}
```

**输出**：`{ context: string, blocks: number, tokens: number }`

**核心算法**：
1. **并行加载**：固定槽位（slots）、项目画像（profile）、教训（lessons）
2. **槽位内容**：`renderPinnedContext()` 渲染用户固定的高优先级记忆
3. **项目画像**：topConcepts（前 8）、topFiles（前 5）、conventions、commonErrors（前 3）
4. **教训排序**：项目匹配 ×1.5 权重 × confidence，取前 10
5. **会话摘要**：同项目其他会话的摘要，无摘要则取 importance ≥ 5 的前 5 条观察
6. **预算分配**：按 recency 降序排列所有块，贪心填充直到 token 预算耗尽
7. **XML 包装**：`<agentmemory-context project="...">...</agentmemory-context>`

### 2.10 summarize — 摘要

**职责**：将会话的压缩观察汇总为结构化 SessionSummary。

**注册函数 ID**：`mem::summarize`

**输入**：`{ sessionId: string }`

**输出**：`{ success: true, summary: SessionSummary, qualityScore: number }` 或 `{ success: false, error: string }`

**核心算法**：
1. **分块处理**：观察数 > chunkSize（默认 400）时，分块并行处理（并发度默认 6）
2. **块级重试**：每块最多 2 次尝试（LLM 调用 + XML 解析）
3. **跳过率熔断**：跳过块 > 50% 时整体失败
4. **Reduce 合并**：所有块摘要通过 `REDUCE_SYSTEM` prompt 合并为最终摘要
5. **顶层重试**：最终解析失败时重试一次（处理 markdown 包裹等格式问题）
6. **质量验证**：`validateOutput()` + `scoreSummary()` 评估摘要质量
7. **noop 降级**：无 LLM provider 时返回明确错误

### 2.11 crystallize — 结晶化

**职责**：将已完成的 Action 链压缩为 Crystal 摘要，自动提取教训。

**注册函数 ID**：`mem::crystallize`、`mem::crystal-list`、`mem::crystal-get`、`mem::auto-crystallize`

**核心算法**（crystallize）：
1. **Action 校验**：所有 actionId 必须存在且状态为 done/cancelled
2. **依赖图构建**：加载 ActionEdge，筛选相关边
3. **LLM 摘要**：生成 JSON 格式的 digest（narrative、keyOutcomes、filesAffected、lessons）
4. **教训保存**：每条 lesson 触发 `mem::lesson-save`（confidence 0.6）
5. **Action 标记**：更新 `crystallizedInto` 字段

**auto-crystallize**：
- 按 parentId/project 分组已完成但未结晶的 Action
- 支持 dryRun 模式预览

### 2.12 reflect — 反思

**职责**：从知识图谱和语义记忆中发现概念聚类，通过 LLM 生成洞察（Insight），支持衰减清扫。

**注册函数 ID**：`mem::reflect`、`mem::insight-list`、`mem::insight-search`、`mem::insight-decay-sweep`

**核心算法**（reflect）：
1. **聚类策略**：
   - 优先：基于知识图谱（GraphNode + GraphEdge）的 BFS 聚类，深度 ≤ 2
   - 回退：基于 Jaccard 相似度的语义记忆聚类（阈值 0.3）
2. **LLM 洞察生成**：每簇最多 5 条洞察，总上限 50 条
3. **去重**：`fingerprintId("ins", content)` 内容寻址
4. **强化**：已有洞察被再次发现时，confidence 按 `min(1.0, confidence + 0.1 * (1 - confidence))` 提升

**insight-decay-sweep**：
- 每周衰减：`confidence -= decayRate * weeksSince`
- confidence ≤ 0.1 且无强化 → 软删除

### 2.13 enrich — 丰富化

**职责**：为当前工作上下文补充相关记忆信息（文件上下文 + 搜索结果 + Bug 记忆）。

**注册函数 ID**：`mem::enrich`

**输入**：
```typescript
{
  sessionId: string;
  files: string[];
  terms?: string[];
  toolName?: string;
  project?: string;
}
```

**核心算法**：
1. **并行三路获取**：
   - `mem::file-context`：文件级上下文
   - `mem::search`：按文件名 + 术语搜索
   - Bug 记忆：匹配文件路径的 type=bug 记忆（前 3 条）
2. **XML 包装**：搜索结果 → `<agentmemory-relevant-context>`，Bug → `<agentmemory-past-errors>`
3. **截断**：总长度上限 4000 字符

### 2.14 evict — 驱逐

**职责**：按多维度策略清理过期和低价值数据。

**注册函数 ID**：`mem::evict`

**输入**：`{ dryRun?: boolean }`

**核心算法（四维度清理）**：
1. **过期会话**：超过 staleSessionDays（默认 30 天）且无摘要的会话
   - 有压缩观察 → 先尝试恢复（`event::session::stopped`）再驱逐
   - 恢复后触发 `mem::consolidate-pipeline`
2. **低重要性观察**：超过 lowImportanceMaxDays（默认 90 天）且 importance < 3
3. **项目观察上限**：超过 maxObservationsPerProject（默认 10000）时按 importance 升序驱逐
4. **过期记忆**：forgetAfter 已过期的记忆 + isLatest=false 且超过 90 天的旧版本记忆

### 2.15 auto-forget — 自动遗忘

**职责**：三维度自动清理：TTL 过期、矛盾检测、低价值观察。

**注册函数 ID**：`mem::auto-forget`

**输入**：`{ dryRun?: boolean }`

**核心算法**：
1. **TTL 过期**：forgetAfter 已过期的记忆直接删除
2. **矛盾检测**：共享概念的最新记忆对，Jaccard 相似度 > 0.9 时标记较旧的为 `isLatest=false`
3. **低价值观察**：超过 180 天且 importance ≤ 2 的观察删除

### 2.16 retention — 保留评分

**职责**：基于指数衰减模型计算记忆保留分数，支持分层驱逐。

**注册函数 ID**：`mem::retention-score`、`mem::retention-evict`

**核心算法**：

**保留分数公式**：
```
score = min(1, salience × e^(-λ × Δt) + reinforcementBoost)
```

- **salience**（显著性）：基于记忆类型权重（architecture=0.9, preference=0.85, pattern=0.8, bug=0.7, workflow=0.6, fact=0.5）+ 访问次数加成（最多 +0.2）
- **temporalDecay**（时间衰减）：`e^(-λ × daysSinceCreation)`，默认 λ=0.01
- **reinforcementBoost**（强化提升）：`Σ(1/daysSinceAccess) × σ`，默认 σ=0.3

**分层阈值**：
- hot：≥ 0.7
- warm：≥ 0.4
- cold：≥ 0.15
- evictable：< 0.15

**retention-evict**：驱逐 score < threshold 的记忆，自动路由到 episodic（KV.memories）或 semantic（KV.semantic）作用域。

### 2.17 dedup — 去重

**职责**：基于 SHA-256 哈希的短期去重映射，防止同一会话内重复观察。

**核心实现**（DedupMap 类）：
- **哈希计算**：`SHA-256(sessionId:toolName:toolInput[:500])`
- **TTL**：5 分钟
- **清理**：每 60 秒定时清理过期条目（`unref()` 不阻塞进程退出）

### 2.18 privacy — 隐私清洗

**职责**：移除原始数据中的敏感信息。

**注册函数 ID**：`mem::privacy`

**核心算法**：
1. **私有标签移除**：`<private>...</private>` → `[REDACTED]`
2. **密钥模式匹配**（12 种模式）：
   - API Key / Secret / Token / Password 模式
   - Bearer Token
   - OpenAI `sk-proj-` / `sk-` / `pk-` / `rk-` / `ak-`
   - Anthropic `sk-ant-`
   - GitHub `ghp_` / `ghs_` / `ghu_` / `ghp_` / `github_pat_`
   - Slack `xoxb-`
   - AWS `AKIA`
   - Google `AIza`
   - JWT `eyJ...`
   - npm `npm_`
   - GitLab `glpat-`
   - Doppler `dop_v1_`

### 2.19 audit — 审计

**职责**：记录所有结构性变更的审计日志。

**核心函数**：
- `recordAudit()`：写入 AuditEntry 到 KV.audit，包含 id、timestamp、operation、functionId、targetIds、details
- `safeAudit()`：try/catch 包装，审计写入失败不影响主流程
- `queryAudit()`：按 operation、日期范围查询审计记录

**审计覆盖策略**（#125）：
- 作用域删除：每次调用一条审计记录，targetIds 列出所有被删除项
- 批量删除：每次调用一条批量审计记录，details.evicted 记录数量

---

## 3. 函数调用链与依赖关系

### 3.1 核心调用链

```
Hook 脚本 → mem::observe
              ├→ dedup.DedupMap（去重检查）
              ├→ privacy.stripPrivateData（隐私清洗）
              ├→ image-store.saveImageToDisk（图像持久化）
              ├→ mem::disk-size-delta（磁盘配额追踪）
              ├→ mem::vision-embed（图像嵌入，可选）
              ├→ stream::set / stream::send（实时推送）
              ├→ [AGENTMEMORY_AUTO_COMPRESS=true]
              │   └→ mem::compress
              │       ├→ provider.compress（LLM 调用）
              │       ├→ compressWithRetry（自纠错重试）
              │       └→ stream::set / stream::send
              └→ [默认] buildSyntheticCompression
                  ├→ search.getSearchIndex().add（BM25 索引）
                  └→ search.vectorIndexAddGuarded（向量索引）

mem::remember → search.getSearchIndex().add
              → search.vectorIndexAddGuarded
              → mem::cascade-update（supersede 时）

mem::consolidate-pipeline
  ├→ [semantic] provider.summarize → KV.semantic
  ├→ [reflect]  mem::reflect
  │               ├→ buildGraphClusters / buildJaccardClusters
  │               └→ provider.summarize → KV.insights
  ├→ [procedural] provider.summarize → KV.procedural
  └→ [decay] applyDecay → KV.semantic + KV.procedural

mem::context → slots.renderPinnedContext
             → KV.profiles（项目画像）
             → KV.lessons（教训）
             → KV.summaries（会话摘要）
             → KV.observations（观察回退）

mem::smart-search → searchFn（混合搜索）
                 → mem::lesson-recall
                 → detectFollowup（诊断）

mem::evict → event::session::stopped（恢复过期会话）
           → mem::consolidate-pipeline（恢复后整合）

mem::crystallize → mem::lesson-save（教训提取）
```

### 3.2 模块间依赖图

```
observe ──────→ compress-synthetic
  │                 │
  ├──────→ compress │
  │                 │
  ├──────→ privacy  │
  │                 │
  ├──────→ dedup    │
  │                 │
  └──────→ search ←─┘ (BM25 + 向量索引)
              ↑
remember ─────┘
  │
  └──→ audit

consolidate-pipeline ──→ reflect
  │                        │
  └──→ audit ←─────────────┘

context ──→ slots
  │
  └──→ access-tracker

smart-search ──→ search
  │
  └──→ access-tracker

evict ──→ audit
  │
  └──→ access-tracker

auto-forget ──→ search (索引清理)
  │
  └──→ audit

retention ──→ search (索引清理)
  │
  └──→ audit

crystallize ──→ lesson-save
  │
  └──→ audit
```

---

## 4. 记忆生命周期分析

记忆在 agentmemory 中经历以下完整生命周期：

### 阶段一：捕获（Capture）
- Hook 脚本将 Agent 行为事件发送到 `mem::observe`
- 原始观察（RawObservation）包含 sessionId、hookType、timestamp、raw data
- 去重（DedupMap 5 分钟窗口）和隐私清洗（stripPrivateData）在此阶段完成

### 阶段二：压缩（Compression）
- **默认路径**：零 LLM 合成压缩（buildSyntheticCompression），启发式推断类型和元数据
- **可选路径**：LLM 压缩（mem::compress），生成更丰富的结构化摘要
- 压缩后更新 KV、BM25 索引和向量索引

### 阶段三：存储（Storage）
- 压缩观察存储在 `KV.observations(sessionId)` 作用域
- 显式记忆通过 `mem::remember` 存储在 `KV.memories` 作用域
- 记忆支持版本演进（supersede）和 TTL（forgetAfter）

### 阶段四：检索（Retrieval）
- `mem::search`：BM25 + 向量混合搜索
- `mem::smart-search`：增强搜索 + Lesson 回忆 + Agent 作用域
- `mem::enrich`：上下文丰富化（文件 + 搜索 + Bug 记忆）
- 每次检索触发访问追踪（recordAccessBatch）

### 阶段五：整合（Consolidation）
- `mem::summarize`：会话级摘要
- `mem::consolidate`：概念聚类 → 长期记忆
- `mem::consolidate-pipeline`：四阶段管道（语义 + 反思 + 程序 + 衰减）
- `mem::crystallize`：Action 链 → Crystal 摘要 + Lesson
- `mem::reflect`：概念聚类 → Insight

### 阶段六：注入（Injection）
- `mem::context`：按 token 预算组装上下文块
- 优先级：固定槽位 > 项目画像 > 教训 > 会话摘要 > 重要观察

### 阶段七：衰减与遗忘（Decay & Forgetting）
- `mem::retention-score`：指数衰减评分
- `mem::evict`：多维度策略驱逐
- `mem::auto-forget`：TTL 过期 + 矛盾检测 + 低价值清理
- `mem::insight-decay-sweep`：洞察衰减清扫
- `mem::forget`：用户主动删除

---

## 5. 关键设计模式

### 5.1 双路径压缩策略
observe 函数根据 `AGENTMEMORY_AUTO_COMPRESS` 环境变量选择压缩路径：
- **零 LLM 路径**（默认）：`buildSyntheticCompression()` 纯启发式，零 token 消耗，confidence=0.3
- **LLM 路径**（opt-in）：`mem::compress` 调用 LLM 生成高质量摘要，带自纠错重试

这种设计确保默认安装不依赖 LLM API Key 即可运行，同时为需要更丰富摘要的用户提供升级路径。

### 5.2 内容寻址去重
- 记忆（Memory）使用 `generateId("mem")` 生成唯一 ID
- 洞察（Insight）使用 `fingerprintId("ins", content)` 内容寻址 ID
- 观察（Observation）使用 DedupMap 的 SHA-256 哈希短期去重

### 5.3 版本演进链
记忆通过 `parentId` / `supersedes` / `version` / `isLatest` 四个字段维护版本链：
- Jaccard 相似度 > 0.7 触发 supersede
- 旧版本标记 `isLatest=false`
- 新版本 `version = oldVersion + 1`
- `mem::cascade-update` 处理级联更新

### 5.4 软降级模式
- 向量索引写入失败（维度不匹配、嵌入服务不可用）→ 跳过，仅 BM25 生效
- LLM 压缩失败 → 返回 `{ success: false }`，不影响观察存储
- 审计写入失败 → `safeAudit()` 静默吞错
- 搜索索引为空 → 自动触发 `rebuildIndex()`

### 5.5 键锁并发控制
`withKeyedLock(key, fn)` 提供细粒度互斥：
- `mem::remember` 使用 `mem:remember` 锁保护版本演进检测
- `mem::observe` 使用 `obs:{sessionId}` 锁保护会话内观察计数
- `smart-search` 使用 `recent-searches:{sessionId}` 锁保护 followup 检测

### 5.6 分层衰减模型
- **语义/程序记忆**：`strength *= 0.9^decayPeriods`（consolidation-pipeline 的 decay 阶段）
- **洞察**：`confidence -= decayRate * weeksSince`（insight-decay-sweep）
- **保留评分**：`score = salience × e^(-λΔt) + reinforcementBoost`（retention-score）

### 5.7 项目隔离
- 记忆的 `project` 字段实现跨项目隔离
- supersede、consolidate、search、enrich 均尊重项目边界
- 无 project 字段的记忆（遗留数据）视为通配，可被任何项目访问

---

## 6. Mermaid 图表

### 6.1 记忆生命周期全流程图

```mermaid
flowchart TD
    A[Hook 脚本触发] --> B[mem::observe]
    B --> C{去重检查}
    C -->|重复| D[返回 deduplicated]
    C -->|新观察| E[隐私清洗]
    E --> F[图像提取与持久化]
    F --> G[KV 存储 RawObservation]
    G --> H{压缩模式?}

    H -->|AUTO_COMPRESS=true| I[mem::compress]
    I --> J[LLM 调用 + 自纠错]
    J --> K[CompressedObservation]
    K --> L[更新 KV + 索引]

    H -->|默认| M[buildSyntheticCompression]
    M --> N[启发式推断类型/元数据]
    N --> K

    L --> O[流推送]
    O --> P[观察就绪]

    P --> Q{检索方式}
    Q -->|全文搜索| R[mem::search]
    Q -->|智能搜索| S[mem::smart-search]
    Q -->|上下文注入| T[mem::context]
    Q -->|丰富化| U[mem::enrich]

    P --> V{整合路径}
    V -->|会话摘要| W[mem::summarize]
    V -->|概念整合| X[mem::consolidate]
    V -->|管道整合| Y[mem::consolidate-pipeline]
    V -->|结晶化| Z[mem::crystallize]
    V -->|反思| AA[mem::reflect]

    Y --> AB[语义合并]
    Y --> AC[反思]
    Y --> AD[程序提取]
    Y --> AE[衰减]

    Z --> AF[Crystal + Lesson]

    AA --> AG[Insight]

    P --> AH{衰减与遗忘}
    AH -->|保留评分| AI[mem::retention-score]
    AH -->|策略驱逐| AJ[mem::evict]
    AH -->|自动遗忘| AK[mem::auto-forget]
    AH -->|洞察衰减| AL[mem::insight-decay-sweep]
    AH -->|主动删除| AM[mem::forget]

    style A fill:#e1f5fe
    style P fill:#c8e6c9
    style AH fill:#ffcdd2
```

### 6.2 observe 函数内部时序图

```mermaid
sequenceDiagram
    participant Hook as Hook 脚本
    participant Observe as mem::observe
    participant Dedup as DedupMap
    participant Privacy as stripPrivateData
    participant Image as image-store
    participant KV as StateKV
    participant Stream as stream::set/send
    participant Compress as 压缩路径

    Hook->>Observe: HookPayload
    Observe->>Observe: 输入校验
    Observe->>Dedup: computeHash + isDuplicate
    Dedup-->>Observe: 去重结果

    alt 重复
        Observe-->>Hook: { deduplicated: true }
    else 新观察
        Observe->>Privacy: stripPrivateData
        Privacy-->>Observe: 清洗后数据

        Observe->>Observe: extractImage

        alt 含图像
            Observe->>Image: saveImageToDisk
            Image-->>Observe: filePath + bytesWritten
            Observe->>KV: incrementImageRef
            Observe->>Observe: mem::disk-size-delta
        end

        Observe->>KV: set(observations, obsId, raw)
        Observe->>Dedup: record(hash)

        Observe->>Stream: stream::set (raw)
        Observe->>Stream: stream::send (viewer)

        alt AUTO_COMPRESS=true
            Observe->>Compress: mem::compress (Void)
        else 默认
            Observe->>Observe: buildSyntheticCompression
            Observe->>KV: set(observations, obsId, synthetic)
            Observe->>Observe: BM25 + 向量索引
            Observe->>Stream: stream::set (compressed)
        end

        Observe-->>Hook: { observationId }
    end
```

### 6.3 remember 函数版本演进流程图

```mermaid
flowchart TD
    A[mem::remember 调用] --> B[输入校验]
    B --> C[获取 project]
    C --> D[withKeyedLock mem:remember]
    D --> E[遍历 isLatest 记忆]

    E --> F{Jaccard 相似度 > 0.7?}
    F -->|否| G[无 supersede]
    F -->|是| H{项目匹配?}

    H -->|不同项目| G
    H -->|同项目或无项目| I[找到 superseded 记忆]

    I --> J[旧记忆 isLatest = false]
    J --> K[KV.set 旧记忆]

    G --> L[创建新记忆]
    I --> L
    L --> M[设置 version / parentId / supersedes]
    M --> N[设置 agentId / project / forgetAfter]
    N --> O[KV.set 新记忆]

    O --> P[BM25 索引更新]
    O --> Q[向量索引更新]

    I --> R{有 supersededId?}
    R -->|是| S[mem::cascade-update]
    R -->|否| T[返回成功]

    S --> T

    style I fill:#fff9c4
    style S fill:#e1bee7
```

### 6.4 consolidation-pipeline 四阶段数据流图

```mermaid
flowchart TD
    A[mem::consolidate-pipeline] --> B{tier 参数}

    B -->|all / semantic| C[阶段一：语义合并]
    B -->|all / reflect| D[阶段二：反思]
    B -->|all / procedural| E[阶段三：程序提取]
    B -->|all / decay| F[阶段四：衰减]

    C --> C1[加载 SessionSummary ≥ 5 条]
    C1 --> C2[取最近 20 条摘要]
    C2 --> C3[LLM 提取 fact 标签]
    C3 --> C4{事实已存在?}
    C4 -->|是| C5[accessCount++ / confidence 取大]
    C4 -->|否| C6[创建 SemanticMemory]
    C5 --> C7[KV.semantic 更新]
    C6 --> C7

    D --> D1[mem::reflect]
    D1 --> D2[概念聚类]
    D2 --> D3[LLM 生成 Insight]
    D3 --> D4{洞察已存在?}
    D4 -->|是| D5[reinforceInsight]
    D4 -->|否| D6[创建 Insight]
    D5 --> D7[KV.insights 更新]
    D6 --> D7

    E --> E1[筛选 pattern 类型记忆 ≥ 2 条]
    E1 --> E2[频率 ≥ 2 的 pattern]
    E2 --> E3[LLM 提取 procedure 标签]
    E3 --> E4{程序已存在?}
    E4 -->|是| E5[frequency++ / strength + 0.1]
    E4 -->|否| E6[创建 ProceduralMemory]
    E5 --> E7[KV.procedural 更新]
    E6 --> E7

    F --> F1[加载 SemanticMemory]
    F --> F2[加载 ProceduralMemory]
    F1 --> F3[applyDecay: strength × 0.9^periods]
    F2 --> F3
    F3 --> F4[KV 批量更新]

    C7 --> G[汇总 results]
    D7 --> G
    E7 --> G
    F4 --> G

    G --> H{OBSIDIAN_AUTO_EXPORT?}
    H -->|true| I[mem::obsidian-export]
    H -->|false| J[recordAudit]

    I --> J
    J --> K[返回结果]

    style C fill:#e3f2fd
    style D fill:#f3e5f5
    style E fill:#e8f5e9
    style F fill:#fff3e0
```

### 6.5 auto-forget 三维度清理决策树

```mermaid
flowchart TD
    A[mem::auto-forget] --> B[加载所有记忆]

    B --> C[维度一：TTL 过期]
    C --> C1{forgetAfter 存在?}
    C1 -->|否| D[维度二：矛盾检测]
    C1 -->|是| C2{当前时间 > forgetAfter?}
    C2 -->|否| D
    C2 -->|是| C3{dryRun?}
    C3 -->|是| C4[记录到 ttlExpired]
    C3 -->|否| C5[删除记忆 + 清理索引 + 审计]
    C4 --> D
    C5 --> D

    D --> D1[筛选 isLatest 记忆]
    D1 --> D2[按概念建立索引]
    D2 --> D3[遍历共享概念的记忆对]
    D3 --> D4{Jaccard 相似度 > 0.9?}
    D4 -->|否| D5[跳过]
    D4 -->|是| D6{dryRun?}
    D6 -->|是| D7[记录到 contradictions]
    D6 -->|否| D8[较旧记忆 isLatest = false + 审计]
    D5 --> E
    D7 --> E
    D8 --> E

    E[维度三：低价值观察] --> E1[加载所有会话观察]
    E1 --> E2{观察年龄 > 180 天?}
    E2 -->|否| E3[跳过]
    E2 -->|是| E4{importance ≤ 2?}
    E4 -->|否| E3
    E4 -->|是| E5{dryRun?}
    E5 -->|是| E6[记录到 lowValueObs]
    E5 -->|否| E7[删除观察 + 清理索引 + 审计]
    E3 --> F
    E6 --> F
    E7 --> F

    F{有实际删除?} -->|是| G[flushIndexSave]
    F -->|否| H[返回结果]
    G --> H

    style C fill:#ffcdd2
    style D fill:#fff9c4
    style E fill:#c8e6c9
```

### 6.6 retention-score 指数衰减模型图

```mermaid
flowchart TD
    A[mem::retention-score] --> B[解析 DecayConfig]
    B --> C[加载 Memory + SemanticMemory + AccessLog]

    C --> D[计算 Salience]
    D --> D1{记忆类型}
    D1 -->|architecture| D2[baseSalience = 0.9]
    D1 -->|preference| D3[baseSalience = 0.85]
    D1 -->|pattern| D4[baseSalience = 0.8]
    D1 -->|bug| D5[baseSalience = 0.7]
    D1 -->|workflow| D6[baseSalience = 0.6]
    D1 -->|fact| D7[baseSalience = 0.5]
    D2 --> D8[+ accessBonus: min 0.2, count × 0.02]
    D3 --> D8
    D4 --> D8
    D5 --> D8
    D6 --> D8
    D7 --> D8
    D8 --> D9[salience = min 1, base + bonus]

    C --> E[计算 TemporalDecay]
    E --> E1["temporalDecay = e^(-λ × Δt)"]
    E1 --> E2["Δt = 自创建以来的天数"]
    E2 --> E3["默认 λ = 0.01"]

    C --> F[计算 ReinforcementBoost]
    F --> F1["boost = Σ(1 / daysSinceAccess) × σ"]
    F1 --> F2["默认 σ = 0.3"]
    F2 --> F3["每次访问贡献 1/距今天数"]

    D9 --> G["score = min(1, salience × temporalDecay + boost)"]
    E3 --> G
    F3 --> G

    G --> H[分层分类]
    H --> I{score ≥ 0.7?}
    I -->|是| J[🔥 Hot]
    I -->|否| K{score ≥ 0.4?}
    K -->|是| L[🌤 Warm]
    K -->|否| M{score ≥ 0.15?}
    M -->|是| N[❄ Cold]
    M -->|否| O[🗑 Evictable]

    J --> P[批量写入 KV.retentionScores]
    L --> P
    N --> P
    O --> P

    P --> Q[recordAudit]
    Q --> R[返回分层统计 + 评分列表]

    style J fill:#ff6b6b
    style L fill:#ffd93d
    style N fill:#6bcbff
    style O fill:#cccccc
```

### 6.7 上下文组装 token 预算分配图

```mermaid
flowchart TD
    A[mem::context 调用] --> B[确定 token 预算]
    B --> C[并行加载上下文源]

    C --> D[源一：固定槽位 Slots]
    C --> E[源二：项目画像 Profile]
    C --> F[源三：教训 Lessons]
    C --> G[源四：会话摘要 Summaries]
    C --> H[源五：重要观察 Observations]

    D --> D1[renderPinnedContext]
    D1 --> D2[ContextBlock: type=memory]

    E --> E1{Profile 存在?}
    E1 -->|是| E2[topConcepts 前8 + topFiles 前5]
    E2 --> E3[conventions + commonErrors 前3]
    E3 --> E4[ContextBlock: type=memory]
    E1 -->|否| E5[跳过]

    F --> F1[过滤: 未删除 + 项目匹配]
    F1 --> F2[排序: 项目匹配×1.5 × confidence]
    F2 --> F3[取前 10 条]
    F3 --> F4[ContextBlock: type=memory]

    G --> G1[同项目其他会话]
    G1 --> G2{有摘要?}
    G2 -->|是| G3[title + narrative + decisions + files]
    G3 --> G4[ContextBlock: type=summary]
    G2 -->|否| G5[importance ≥ 5 的前 5 条观察]
    G5 --> G6[ContextBlock: type=observation]

    H --> G5

    D2 --> I[所有 ContextBlock 按 recency 降序排列]
    E4 --> I
    F4 --> I
    G4 --> I
    G6 --> I

    I --> J[预留 header + footer token]
    J --> K[贪心填充循环]

    K --> L{下一个 block 能放入?}
    L -->|是| M[加入 selected, usedTokens += block.tokens]
    M --> L
    L -->|否| N[跳过该 block]
    N --> L

    L -->|遍历完毕| O[recordAccessBatch]
    O --> P["组装: <agentmemory-context>...</agentmemory-context>"]
    P --> Q[返回 context + blocks + tokens]

    style D fill:#e1bee7
    style E fill:#b2dfdb
    style F fill:#fff9c4
    style G fill:#bbdefb
    style H fill:#c8e6c9
```
