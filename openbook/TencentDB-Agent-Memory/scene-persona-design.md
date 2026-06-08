# Scene + Persona 模块设计文档

## 1. 模块概述

Scene + Persona 模块是记忆系统的 L2/L3 层核心组件，负责将 L1 原始记忆碎片提炼为结构化的场景叙事（L2）和用户画像（L3）。

| 层级 | 模块 | 职责 | 核心文件 |
|------|------|------|----------|
| L2 | Scene | 场景块的提取、格式化、索引和导航 | `scene-extractor.ts`, `scene-format.ts`, `scene-index.ts`, `scene-navigation.ts`, `filename-normalizer.ts` |
| L3 | Persona | 用户画像的生成和触发 | `persona-generator.ts`, `persona-trigger.ts` |
| 同步 | Profile Sync | L2/L3 本地与远程双向同步 | `profile-sync.ts` |
| 提示词 | Prompts | 场景提取 & Persona 生成的系统提示词 | `scene-extraction.ts`, `persona-generation.ts` |

### 核心设计原则

- **LLM Agent 驱动**：场景提取和 Persona 生成均由 LLM Agent 通过文件工具自主完成，工程侧仅负责编排和后处理
- **沙箱隔离**：LLM Agent 的文件操作被严格限定在沙箱目录内，系统文件对 LLM 不可见
- **软删除机制**：LLM 无法直接删除文件，通过写入 `[DELETED]` 标记实现，工程侧负责清理
- **幂等安全**：文件名归一化、索引同步等操作均为幂等设计，可安全重复执行

---

## 2. 架构设计

```mermaid
graph TB
    subgraph "Scene 模块 (L2)"
        SE[SceneExtractor<br/>场景提取编排器]
        SF[scene-format.ts<br/>场景文件格式]
        SI[scene-index.ts<br/>场景索引]
        SN[scene-navigation.ts<br/>场景导航]
        FN[filename-normalizer.ts<br/>文件名归一化]
    end

    subgraph "Persona 模块 (L3)"
        PG[PersonaGenerator<br/>Persona 生成器]
        PT[PersonaTrigger<br/>Persona 触发器]
    end

    subgraph "Profile 同步"
        PS[profile-sync.ts<br/>L2/L3 双向同步]
    end

    subgraph "提示词"
        P1[scene-extraction.ts<br/>场景提取提示词]
        P2[persona-generation.ts<br/>Persona 生成提示词]
    end

    subgraph "外部依赖"
        LLM[LLM Runner<br/>CleanContextRunner]
        CK[CheckpointManager<br/>检查点管理]
        BK[BackupManager<br/>备份管理]
        STORE[IMemoryStore<br/>远程存储]
    end

    subgraph "文件系统"
        SB[scene_blocks/*.md<br/>场景文件]
        MD[.metadata/scene_index.json<br/>场景索引]
        PM[persona.md<br/>用户画像]
    end

    SE -->|1.构建提示词| P1
    SE -->|2.运行 Agent| LLM
    SE -->|3.清理软删除| SB
    SE -->|4.归一化文件名| FN
    SE -->|5.同步索引| SI
    SE -->|6.更新导航| SN
    SE -->|7.解析信号| PT
    SE -->|备份| BK
    SE -->|读取检查点| CK
    SF -->|解析/格式化| SB
    SI -->|读写| MD
    SN -->|追加导航| PM

    PG -->|1.构建提示词| P2
    PG -->|2.运行 Agent| LLM
    PG -->|3.剥离导航| SN
    PG -->|4.追加导航| SN
    PG -->|备份| BK
    PG -->|读取检查点| CK
    PG -->|读取索引| SI

    PT -->|评估触发| PG
    PT -->|读取检查点| CK

    PS -->|拉取远程| STORE
    PS -->|推送本地| STORE
    PS -->|同步索引| SI
    PS -->|刷新导航| SN
    PS -->|读写| SB
    PS -->|读写| PM
```

---

## 3. 场景提取流程

`SceneExtractor.extract()` 实现了一个 8 阶段流水线，将原始记忆批量转化为场景块文件：

```mermaid
sequenceDiagram
    participant Client as 调用方
    participant SE as SceneExtractor
    participant BK as BackupManager
    participant SI as SceneIndex
    participant FN as FilenameNormalizer
    participant SN as SceneNavigation
    participant LLM as LLM Agent
    participant SB as scene_blocks/
    participant CK as CheckpointManager

    Client->>SE: extract(memories)

    rect rgb(240, 248, 255)
        Note over SE,CK: Phase 1: 备份
        SE->>CK: read() 获取检查点
        SE->>BK: backupDirectory(scene_blocks/)
        BK-->>SE: 备份完成
    end

    rect rgb(255, 248, 240)
        Note over SE,SI: Phase 2: 加载索引
        SE->>SI: readSceneIndex()
        SI-->>SE: 索引条目列表
        SE->>SB: 快照现有场景内容（用于后续 diff）
    end

    rect rgb(240, 255, 240)
        Note over SE,LLM: Phase 3: 构建提示词
        SE->>SE: buildSceneSummaries() 构建场景摘要
        SE->>SE: 构建场景数量警告（分级预警）
        SE->>SE: buildSceneExtractionPrompt()
        Note over SE: systemPrompt = 角色定义 + 架构模型 +<br/>文件操作约束 + 工作流 + 输出模板
        Note over SE: userPrompt = 新增记忆 + 场景摘要 +<br/>时间戳 + 场景文件清单
    end

    rect rgb(255, 240, 255)
        Note over SE,SB: Phase 4: 运行 LLM Agent
        SE->>LLM: run(systemPrompt, userPrompt, workspaceDir=scene_blocks/)
        Note over LLM,SB: LLM 在沙箱内操作<br/>read → write → edit
        LLM->>SB: 创建/更新/软删除场景文件
        LLM-->>SE: 文本输出（含可能的 PERSONA_UPDATE_REQUEST）
        Note over SE: 若 LLM 失败 → 从备份恢复 scene_blocks/
    end

    rect rgb(255, 255, 230)
        Note over SE,SB: Phase 5: 清理软删除
        SE->>SB: 扫描所有 .md 文件
        SE->>SB: 删除内容为 [DELETED] 或空的文件
        SE->>SB: 删除仅含 META 头但无正文的文件
    end

    rect rgb(230, 255, 255)
        Note over SE,FN: Phase 5b: 文件名归一化
        SE->>FN: normalizeSceneFilenames()
        FN->>SB: 重命名不合规文件（空格→短横线，删除括号等）
        FN->>FN: resolveUniqueScenePath() 解决冲突（追加 -2, -3 后缀）
    end

    rect rgb(245, 245, 255)
        Note over SE,SI: Phase 6: 同步索引
        SE->>SI: syncSceneIndex()
        SI->>SB: 扫描所有 .md 文件
        SI->>SI: 解析 META → 重建 scene_index.json
    end

    rect rgb(255, 245, 238)
        Note over SE,SN: Phase 7: 更新导航
        SE->>SN: updateSceneNavigation()
        SN->>SN: generateSceneNavigation() 按热度排序
        SN->>SN: stripSceneNavigation() 剥离旧导航
        Note over SN: 若 persona 正文为空则跳过<br/>（等待 PersonaGenerator 处理）
    end

    rect rgb(240, 255, 240)
        Note over SE,CK: Phase 8: 解析信号
        SE->>SE: parsePersonaUpdateSignal(llmOutput)
        alt 检测到 PERSONA_UPDATE_REQUEST
            SE->>CK: setPersonaUpdateRequest(reason)
        end
    end

    SE-->>Client: ExtractionResult
```

### 8 阶段流水线详解

| 阶段 | 名称 | 说明 | 失败处理 |
|------|------|------|----------|
| 1 | 备份 | 使用 BackupManager 备份 scene_blocks/ 目录 | — |
| 2 | 加载索引 | 读取 scene_index.json，快照现有场景内容和索引 | 返回空列表 |
| 3 | 构建提示词 | 组装 systemPrompt + userPrompt，含场景数量分级预警 | — |
| 4 | 运行 LLM Agent | 沙箱限定 scene_blocks/，LLM 自主读写场景文件 | 从备份恢复 scene_blocks/ |
| 5 | 清理软删除 | 删除 `[DELETED]` 标记文件和 META-only 文件 | 非致命，继续执行 |
| 5b | 文件名归一化 | 修正 LLM 产生的不合规文件名 | 非致命，继续执行 |
| 6 | 同步索引 | 扫描磁盘重建 scene_index.json | — |
| 7 | 更新导航 | 更新 persona.md 末尾的场景导航区段 | 非致命，继续执行 |
| 8 | 解析信号 | 解析 LLM 输出中的 PERSONA_UPDATE_REQUEST 信号 | — |

### 场景数量分级预警

```mermaid
graph LR
    A[场景数量检查] --> B{count >= maxScenes?}
    B -->|是| C[🔴 红色预警<br/>必须先 MERGE]
    B -->|否| D{count == maxScenes - 1?}
    D -->|是| E[🟠 橙色预警<br/>只能 UPDATE，禁止 CREATE]
    D -->|否| F{count >= maxScenes - 3?}
    F -->|是| G[🟡 黄色预警<br/>建议 UPDATE 或主动 MERGE]
    F -->|否| H[🟢 正常<br/>自由操作]
```

---

## 4. 场景文件格式设计

### 数据结构

```mermaid
classDiagram
    class SceneBlock {
        +string filename
        +SceneBlockMeta meta
        +string content
    }

    class SceneBlockMeta {
        +string created
        +string updated
        +string summary
        +number heat
    }

    class SceneIndexEntry {
        +string filename
        +string summary
        +number heat
        +string created
        +string updated
    }

    SceneBlock --> SceneBlockMeta : meta
    SceneIndexEntry ..> SceneBlockMeta : 同构字段
```

### 文件格式示例

```markdown
-----META-START-----
created: 2026-01-10T08:30:00Z
updated: 2026-03-15T14:20:00Z
summary: 用户在后端开发领域的技能栈与学习轨迹，涵盖 Python 异步框架和 Rust 探索
heat: 42
-----META-END-----

## 用户核心特征
用户在后端开发方面表现出对 Python 的强烈偏好...

## 用户偏好
- 偏好异步编程模型
- 代码洁癖，拒绝打补丁

## 隐性信号
从 Python 向 Rust 的转型意图暗示对系统级编程的渴望

## 核心叙事
本周用户主要集中在后端重构上...

## 演变轨迹
- [2026-01-10]: 从 "反对加班" 转向 "接受弹性工作"

## 待确认/矛盾点
- 对 Rust 的兴趣是长期转型还是短期探索？
```

### 解析与格式化流程

```mermaid
flowchart LR
    A[原始 .md 文件] --> B{包含 META 段?}
    B -->|是| C[提取 META 段<br/>-----META-START----- ~ -----META-END-----]
    B -->|否| D[整文件作为 content<br/>meta 使用默认值]
    C --> E[extractMetaField<br/>逐字段解析]
    E --> F[SceneBlock]
    D --> F

    G[SceneBlockMeta + content] --> H[formatSceneBlock]
    H --> I[formatMeta<br/>生成 META 段]
    I --> J[META 段 + 空行 + content]
```

---

## 5. 场景索引与导航设计

### 场景索引

场景索引存储在 `.metadata/scene_index.json`，是场景块的快速查找表，**由工程侧独占写入**，LLM 无法访问。

```mermaid
flowchart TB
    subgraph "读取路径"
        A1[readSceneIndex] --> A2[读取 .metadata/scene_index.json]
        A2 --> A3[JSON.parse + 类型校验]
        A3 --> A4[SceneIndexEntry[]]
    end

    subgraph "同步路径（重建）"
        B1[syncSceneIndex] --> B2[扫描 scene_blocks/*.md]
        B2 --> B3[逐文件 parseSceneBlock]
        B3 --> B4[提取 meta 字段]
        B4 --> B5[writeSceneIndex<br/>原子写入 scene_index.json]
    end

    subgraph "写入路径"
        C1[writeSceneIndex] --> C2[JSON.stringify(entries)]
        C2 --> C3[fs.writeFile<br/>scene_index.json]
    end

    B5 --> C1
```

### 场景导航

场景导航是追加在 `persona.md` 末尾的索引区段，按热度降序排列，提供场景的绝对路径以便 Agent 直接 `read_file`。

```mermaid
flowchart TB
    subgraph "导航生成"
        A1[SceneIndexEntry[]] --> A2[按 heat 降序排序]
        A2 --> A3[生成热度 emoji<br/>🔥 × 热度等级]
        A3 --> A4[生成 Markdown 导航块]
        A4 --> A5[NAV_HEADER + 条目列表 + NAV_FOOTER]
    end

    subgraph "导航追加到 persona.md"
        B1[读取 persona.md] --> B2[stripSceneNavigation<br/>剥离旧导航]
        B2 --> B3{正文为空?}
        B3 -->|是| B4[跳过，等待 PersonaGenerator]
        B3 -->|否| B5[正文 + 新导航 → 写入]
    end

    subgraph "导航剥离"
        C1[persona.md 内容] --> C2[查找 NAV_HEADER 位置]
        C2 --> C3{找到?}
        C3 -->|是| C4[截取 NAV_HEADER 之前的内容]
        C3 -->|否| C5[原样返回]
    end
```

### 热度 emoji 映射

| 热度范围 | emoji | 含义 |
|----------|-------|------|
| ≥ 1000 | 🔥🔥🔥🔥🔥 | 极高 |
| ≥ 500 | 🔥🔥🔥🔥 | 很高 |
| ≥ 200 | 🔥🔥🔥 | 高 |
| ≥ 100 | 🔥🔥 | 中 |
| ≥ 50 | 🔥 | 低 |
| < 50 | （无） | 极低 |

---

## 6. Persona 生成流程

`PersonaGenerator.generateLocalPersona()` 实现了四层深度扫描模型，支持首次生成和增量更新两种模式：

```mermaid
sequenceDiagram
    participant Client as 调用方
    participant PG as PersonaGenerator
    participant CK as CheckpointManager
    participant SI as SceneIndex
    participant SN as SceneNavigation
    participant BK as BackupManager
    participant LLM as LLM Agent
    participant PM as persona.md

    Client->>PG: generateLocalPersona(triggerReason)

    rect rgb(240, 248, 255)
        Note over PG,PM: Step 1: 读取现有 Persona
        PG->>PM: 读取 persona.md
        PG->>SN: stripSceneNavigation() 剥离导航
        Note over PG: existingPersona = 剥离导航后的正文
    end

    rect rgb(255, 248, 240)
        Note over PG,SI: Step 2: 加载场景索引 & 识别变化场景
        PG->>CK: read() 获取 last_persona_time
        PG->>SI: readSceneIndex()
        PG->>PG: 过滤 updated > last_persona_time 的场景
        Note over PG: changedScenes = 增量变化的场景列表
    end

    rect rgb(240, 255, 240)
        Note over PG: Step 3: 读取变化场景完整内容
        PG->>PG: 逐文件读取 scene_blocks/*.md
        Note over PG: 包含 META 段的完整原始内容
    end

    rect rgb(255, 255, 230)
        Note over PG: Step 4: 判断生成模式
        alt existingPersona 存在
            Note over PG: mode = "incremental"（增量更新）
        else existingPersona 不存在
            Note over PG: mode = "first"（首次生成）
        end
    end

    rect rgb(245, 245, 255)
        Note over PG,LLM: Step 5-6: 构建提示词
        PG->>PG: buildPersonaPrompt()
        Note over PG: systemPrompt = 四层深度扫描模型 +<br/>输出模板 + 文件操作约束
        Note over PG: userPrompt = 统计数据 + 变化场景内容 +<br/>现有 Persona + 迭代指南
    end

    rect rgb(255, 240, 255)
        Note over PG,BK: Step 7: 备份
        PG->>BK: backupFile(persona.md)
    end

    rect rgb(240, 255, 255)
        Note over PG,PM: Step 8: 运行 LLM Agent
        PG->>LLM: run(systemPrompt, userPrompt, workspaceDir=dataDir)
        Note over LLM,PM: LLM 通过 write/edit 工具<br/>直接写入 persona.md
        LLM-->>PG: 完成
    end

    rect rgb(255, 245, 238)
        Note over PG,PM: Step 9-11: 后处理
        PG->>PM: 读取 LLM 写入的 persona.md
        PG->>SN: stripSceneNavigation() 剥离 LLM 可能添加的导航
        PG->>PG: escapeXmlTags() XML 标签消毒
        PG->>SN: generateSceneNavigation() 生成新导航
        PG->>PM: 写入最终内容（正文 + 导航）
    end

    PG-->>Client: true/false
```

### 四层深度扫描模型

```mermaid
graph TB
    subgraph "🟢 Layer 1: 基础锚点 (The Base & Facts)"
        L1A[确凿事实] --> L1B[人口统计学特征]
        L1B --> L1C[当前状态]
        L1C --> L1D[实用价值: 破冰话题 + 上下文感知]
    end

    subgraph "🔵 Layer 2: 兴趣图谱 (The Interest Graph)"
        L2A[活跃爱好] --> L2B[被动消费]
        L2B --> L2C[休眠兴趣]
        L2C --> L2D[实用价值: 高质量闲聊 + 生活推荐]
    end

    subgraph "🟡 Layer 3: 交互协议 (The Interface)"
        L3A[沟通习惯] --> L3B[雷区]
        L3B --> L3C[工作流偏好]
        L3C --> L3D[实用价值: 如何说话 + 如何交付]
    end

    subgraph "🔴 Layer 4: 认知内核 (The Core)"
        L4A[决策逻辑] --> L4B[矛盾点]
        L4B --> L4C[终极驱动力]
        L4C --> L4D[实用价值: 替用户做决策的副驾驶]
    end

    L1D --> L2A
    L2D --> L3A
    L3D --> L4A
```

### Persona 输出模板结构

```mermaid
graph TD
    A[persona.md] --> B[Archetype 核心原型<br/>一句话定义]
    A --> C[基本信息<br/>年龄/性别/职业等]
    A --> D[长期偏好<br/>最稳定可复用的偏好]
    A --> E[Chapter 1: 全景语境<br/>基础事实 + 当前状态]
    A --> F[Chapter 2: 生活的肌理<br/>兴趣 + 消费 + 生活品味]
    A --> G[Chapter 3: 交互与认知协议<br/>3.1 沟通策略 + 3.2 决策逻辑]
    A --> H[Chapter 4: 深层洞察与演变<br/>矛盾统一性 + 演变轨迹 + 涌现特征]
    A --> I[Scene Navigation<br/>工程自动追加]
```

---

## 7. Persona 触发条件设计

`PersonaTrigger.shouldGenerate()` 实现了 5 级优先级触发条件评估：

```mermaid
flowchart TD
    START([shouldGenerate 调用]) --> P1{P1: 显式请求?<br/>checkpoint.request_persona_update}

    P1 -->|是| R1[✅ 触发<br/>原因: 主动请求]

    P1 -->|否| P2{P2: 冷启动?<br/>scenes_processed > 0 AND<br/>last_persona_at === 0 AND<br/>有场景文件}

    P2 -->|是| R2[✅ 触发<br/>原因: 首次冷启动]

    P2 -->|否| P2_5{P2.5: 恢复?<br/>last_persona_at > 0 AND<br/>有场景文件 AND<br/>persona 正文丢失}

    P2_5 -->|是| R2_5[✅ 触发<br/>原因: Persona 正文丢失]

    P2_5 -->|否| P3{P3: 首次场景提取?<br/>scenes_processed === 1 AND<br/>memories_since_last_persona > 0}

    P3 -->|是| R3[✅ 触发<br/>原因: 首次 Scene Block 提取完成]

    P3 -->|否| P4{P4: 阈值触发?<br/>memories_since_last_persona<br/>>= interval}

    P4 -->|是| R4[✅ 触发<br/>原因: 达到阈值]

    P4 -->|否| R0[❌ 不触发]

    R1 --> END([返回 TriggerResult])
    R2 --> END
    R2_5 --> END
    R3 --> END
    R4 --> END
    R0 --> END

    style P1 fill:#ff6b6b,color:#fff
    style P2 fill:#ffa94d,color:#fff
    style P2_5 fill:#ffd43b,color:#333
    style P3 fill:#69db7c,color:#333
    style P4 fill:#74c0fc,color:#333
    style R1 fill:#ff6b6b,color:#fff
    style R2 fill:#ffa94d,color:#fff
    style R2_5 fill:#ffd43b,color:#333
    style R3 fill:#69db7c,color:#333
    style R4 fill:#74c0fc,color:#333
    style R0 fill:#dee2e6,color:#333
```

### 5 级优先级详解

| 优先级 | 名称 | 条件 | 触发原因示例 |
|--------|------|------|-------------|
| P1 | 显式请求 | `checkpoint.request_persona_update === true` | LLM 在场景提取时发出 `PERSONA_UPDATE_REQUEST` 信号 |
| P2 | 冷启动 | `scenes_processed > 0` 且 `last_persona_at === 0` 且有场景文件 | 首次提取完成，从未生成过 Persona |
| P2.5 | 恢复 | `last_persona_at > 0` 且有场景文件且 `persona.md` 正文为空 | Persona 文件损坏或丢失 |
| P3 | 首次场景提取 | `scenes_processed === 1` 且 `memories_since_last_persona > 0` | 第一个场景块提取完成 |
| P4 | 阈值触发 | `memories_since_last_persona >= interval` | 新记忆数量达到配置的间隔阈值 |

### PERSONA_UPDATE_REQUEST 信号流

```mermaid
sequenceDiagram
    participant LLM as LLM Agent (场景提取)
    participant SE as SceneExtractor
    participant CK as CheckpointManager
    participant PT as PersonaTrigger
    participant PG as PersonaGenerator

    LLM->>LLM: 检测到重大价值观转变<br/>或跨场景突破性洞察
    LLM-->>SE: 文本输出含 [PERSONA_UPDATE_REQUEST]reason: xxx[/PERSONA_UPDATE_REQUEST]
    SE->>SE: parsePersonaUpdateSignal()
    SE->>CK: setPersonaUpdateRequest(reason)

    Note over PT: 下一次触发评估周期
    PT->>CK: read()
    PT->>PT: P1 条件满足
    PT->>PG: shouldGenerate() → true
    PG->>PG: generateLocalPersona("主动请求: xxx")
```

---

## 8. Profile 同步设计

`profile-sync.ts` 实现 L2/L3 本地与远程的双向同步，支持拉取（远程→本地）和推送（本地→远程）。

### 拉取流程（远程 → 本地）

```mermaid
sequenceDiagram
    participant PS as profile-sync
    participant STORE as IMemoryStore (远程)
    participant TEMP as 临时目录
    participant LOCAL as 本地文件系统
    participant SI as SceneIndex
    participant SN as SceneNavigation

    PS->>STORE: pullProfiles()
    STORE-->>PS: ProfileRecord[]

    PS->>TEMP: mkdtemp(.profiles-pull-)
    PS->>TEMP: mkdir(scene_blocks/)

    loop 遍历每条记录
        alt type === "l2"
            PS->>TEMP: writeFile(scene_blocks/filename, content)
            PS->>PS: MD5 校验
            alt MD5 不匹配
                PS->>TEMP: 删除该文件
            end
        else type === "l3"
            PS->>PS: stripSceneNavigation(content)
            PS->>TEMP: writeFile(persona.md, body)
            PS->>PS: MD5 校验
            alt MD5 不匹配
                PS->>TEMP: 删除该文件
            end
        end
    end

    rect rgb(255, 240, 240)
        Note over PS,LOCAL: 原子替换 scene_blocks/
        PS->>LOCAL: rm -rf scene_blocks/
        PS->>LOCAL: rename(temp/scene_blocks → scene_blocks/)
        Note over PS,LOCAL: 处理 rename 竞争<br/>ENOTEMPTY/EEXIST → 使用已有结果
    end

    rect rgb(240, 255, 240)
        Note over PS,LOCAL: 原子替换 persona.md
        alt 临时目录有 persona.md
            PS->>LOCAL: rm persona.md
            PS->>LOCAL: rename(temp/persona.md → persona.md)
        else 临时目录无 persona.md
            PS->>LOCAL: rm persona.md（远程无 Persona）
        end
    end

    PS->>SI: syncSceneIndex() 重建索引
    PS->>SN: refreshPersonaNavigation() 刷新导航
    PS->>TEMP: rm -rf 临时目录

    PS-->>PS: 返回 baseline Map
```

### 推送流程（本地 → 远程）

```mermaid
sequenceDiagram
    participant PS as profile-sync
    participant LOCAL as 本地文件系统
    participant STORE as IMemoryStore (远程)

    PS->>LOCAL: listLocalProfiles()
    Note over PS: 扫描 scene_blocks/*.md → L2 记录<br/>扫描 persona.md → L3 记录
    LOCAL-->>PS: ProfileRecord[]

    PS->>PS: 对比 baseline Map
    Note over PS: 过滤条件:<br/>contentMd5 变化 OR 新增记录

    alt 有变更记录
        PS->>STORE: syncProfiles(changedRecords)
        Note over STORE: 增量同步变更的 L2/L3 记录
    end

    PS->>PS: 检测已删除的记录
    Note over PS: baseline 中存在但本地不存在的 ID

    alt 有删除记录
        PS->>STORE: deleteProfiles(deletedIds)
        Note over STORE: 清理远程已删除的记录
    end
```

### Profile 记录结构

```mermaid
classDiagram
    class ProfileRecord {
        +string id
        +string type
        +string filename
        +string content
        +string contentMd5
        +number version
        +number createdAtMs
        +number updatedAtMs
    }

    class ProfileSyncRecord {
        +string id
        +string type
        +string filename
        +string content
        +string contentMd5
        +number version
        +number createdAtMs
        +number updatedAtMs
        +number baselineVersion
    }

    class ProfileBaseline {
        +number version
        +string contentMd5
        +number createdAtMs
    }

    ProfileRecord <|-- ProfileSyncRecord : 继承
    ProfileSyncRecord --> ProfileBaseline : baselineVersion 引用
```

### ID 生成规则

Profile 记录使用确定性 ID，基于 `scope + type + filename` 的 SHA-256 哈希：

```
id = "profile:v1:" + sha256(scope + "\u0000" + type + "\u0000" + filename)
```

---

## 9. LLM Agent 沙箱设计

### 沙箱隔离架构

```mermaid
graph TB
    subgraph "LLM Agent 可见区域（沙箱）"
        direction TB
        SB1[scene_blocks/*.md<br/>场景文件]
        SB2[persona.md<br/>仅 Persona 生成时可见]
    end

    subgraph "LLM Agent 不可见区域（系统文件）"
        direction TB
        H1[.metadata/scene_index.json]
        H2[.metadata/recall_checkpoint.json]
        H3[.backup/]
        H4[其他系统文件]
    end

    subgraph "工程侧管控"
        direction TB
        E1[SceneExtractor<br/>workspaceDir = scene_blocks/]
        E2[PersonaGenerator<br/>workspaceDir = dataDir/]
    end

    E1 -->|"workspaceDir 限定"| SB1
    E2 -->|"workspaceDir 限定"| SB2

    style SB1 fill:#d4edda,stroke:#28a745
    style SB2 fill:#d4edda,stroke:#28a745
    style H1 fill:#f8d7da,stroke:#dc3545
    style H2 fill:#f8d7da,stroke:#dc3545
    style H3 fill:#f8d7da,stroke:#dc3545
    style H4 fill:#f8d7da,stroke:#dc3545
```

### 两种沙箱模式对比

| 维度 | 场景提取 (SceneExtractor) | Persona 生成 (PersonaGenerator) |
|------|--------------------------|-------------------------------|
| workspaceDir | `scene_blocks/` | `dataDir/` |
| 可操作文件 | 仅 `*.md` 场景文件 | 仅 `persona.md` |
| 不可见文件 | checkpoint, scene_index, persona.md | scene_blocks/, .metadata/ |
| 文件工具 | read, write, edit | write, edit（无需 read） |
| 删除方式 | write `[DELETED]` 标记 | 不涉及删除 |

### 安全机制

```mermaid
flowchart TB
    A[LLM Agent 执行] --> B{文件操作类型}

    B -->|read| C{文件在沙箱内?}
    C -->|是| D[✅ 允许读取]
    C -->|否| E[❌ 拒绝：文件不存在]

    B -->|write| F{内容为空/纯空白?}
    F -->|是| G[❌ 拒绝：参数校验失败]
    F -->|否| H{内容为 '[DELETED]'?}
    H -->|是| I[✅ 软删除标记]
    H -->|否| J[✅ 正常写入]

    B -->|edit| K{目标文件在沙箱内?}
    K -->|是| L[✅ 允许编辑]
    K -->|否| M[❌ 拒绝：文件不存在]

    I --> N[工程侧后续清理<br/>fs.unlink 删除文件]
```

### 软删除流程

```mermaid
sequenceDiagram
    participant LLM as LLM Agent
    participant SB as scene_blocks/
    participant SE as SceneExtractor

    LLM->>SB: write(path, "[DELETED]")
    Note over LLM: LLM 无法直接删除文件<br/>写入空字符串会被参数校验拒绝<br/>所以使用 [DELETED] 标记

    Note over SE: LLM 执行完成后<br/>工程侧接管清理

    SE->>SB: 扫描所有 .md 文件
    SE->>SE: 检测内容为 [DELETED] 或空的文件
    SE->>SE: 检测仅含 META 头但无正文的文件
    SE->>SB: fs.unlink() 物理删除
    SE->>SE: syncSceneIndex() 重建索引
```

### 错误恢复机制

```mermaid
flowchart TB
    A[LLM Agent 执行失败] --> B[从 Phase 1 备份恢复]
    B --> C{恢复成功?}
    C -->|是| D[返回失败结果<br/>scene_blocks/ 已恢复]
    C -->|否| E[记录恢复失败日志<br/>返回原始 LLM 错误]

    F[LLM Agent 执行成功] --> G[后处理阶段]
    G --> H{后处理步骤失败?}
    H -->|是| I[非致命：记录日志<br/>继续执行后续步骤]
    H -->|否| J[正常完成]
```
