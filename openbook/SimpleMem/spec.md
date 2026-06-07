# SimpleMem 项目规格说明 (Specification)

## 1. 项目概述

**SimpleMem** 是一个面向 LLM Agent 的统一终身记忆系统，核心理念是：以**语义无损压缩**的方式存储记忆，实现高信息密度，使 Agent 在大幅减少 token 消耗的同时获得更强的记忆召回能力。

项目由三个核心支柱组成：

| 支柱 | 功能 | 模态 |
|:--|:--|:--|
| **SimpleMem (Text)** | 文本记忆的语义压缩与检索 | 文本 |
| **Omni-SimpleMem** | 多模态记忆的统一存储与检索 | 文本/图像/音频/视频 |
| **EvolveMem** | 检索配置的自演化优化 | 跨基准自适应 |

此外还包含：
- **Cross-Session Memory**: 跨会话记忆持久化与上下文注入
- **MCP Server**: 基于 Model Context Protocol 的云端/本地记忆服务
- **SKILL**: Claude Skills 集成

## 2. 系统目标

### 2.1 核心目标

1. **语义无损压缩**: 将非结构化交互压缩为紧凑的、自包含的记忆单元，每个单元通过多视图索引实现灵活检索
2. **在线语义合成**: 在写入时合并相关上下文，消除冗余，而非查询时去重
3. **意图感知检索规划**: 推断查询背后的搜索意图，决定检索什么、如何组装上下文
4. **多模态统一**: 文本、图像、音频、视频以统一的原子单元 (MAU) 存储
5. **自演化检索**: 检索基础设施本身通过 LLM 驱动的闭环诊断自动演化

### 2.2 性能指标

| 基准 | 指标 | SimpleMem | Omni-SimpleMem | EvolveMem |
|:--|:--|:--|:--|:--|
| LoCoMo | F1 | +26.4% vs prior | F1=0.613 (+47%) | +25.7% relative |
| Mem-Gallery | F1 | - | F1=0.810 (+51%) | - |
| MemBench | F1 | - | - | +18.9% relative |

## 3. 用户与使用场景

### 3.1 目标用户

- LLM 应用开发者：需要为 Agent 添加长期记忆能力
- AI 助手用户：通过 MCP 协议接入 Claude Desktop / Cursor 等客户端
- 研究人员：复现论文基准结果

### 3.2 核心使用场景

```python
# 场景1: 文本记忆 (自动路由)
from simplemem import SimpleMem
mem = SimpleMem()
mem.add_dialogue("Alice", "Let's meet at 2pm", "2025-11-15T14:30:00")
mem.finalize()
answer = mem.ask("When will they meet?")

# 场景2: 多模态记忆 (自动路由)
mem = SimpleMem()
mem.add_text("User loves hiking.", tags=["session_id:D1"])
mem.add_image("photo.jpg", tags=["session_id:D1"])
result = mem.query("What does the user enjoy?", top_k=5)

# 场景3: 检索优化
import simplemem
config = simplemem.optimize(mem, dev_questions, max_rounds=3)
config.save("my_config.json")
```

## 4. 功能需求

### 4.1 文本记忆 (SimpleMem Core)

| ID | 功能 | 优先级 | 描述 |
|:--|:--|:--|:--|
| T-01 | 对话输入 | P0 | 支持单条/批量对话添加，含说话人、内容、时间戳 |
| T-02 | 滑动窗口压缩 | P0 | 基于语义密度门控的滑动窗口，将对话压缩为记忆单元 |
| T-03 | 无损重述 | P0 | 每条记忆为自包含事实，消除代词、相对时间 |
| T-04 | 多视图索引 | P0 | 语义层(稠密向量)、词汇层(BM25)、符号层(元数据过滤) |
| T-05 | 会话内合并 | P0 | 写入时合并相关记忆，消除冗余 |
| T-06 | 意图感知检索 | P0 | 分析查询意图，生成定向子查询 |
| T-07 | 反思检索 | P1 | 多轮反思判断信息充分性，补充检索 |
| T-08 | 并行处理 | P1 | 记忆构建与检索均支持并行加速 |
| T-09 | 答案生成 | P0 | 基于检索上下文生成简洁答案 |

### 4.2 多模态记忆 (Omni-SimpleMem)

| ID | 功能 | 优先级 | 描述 |
|:--|:--|:--|:--|
| M-01 | 文本处理 | P0 | 文本内容的熵触发与 MAU 生成 |
| M-02 | 图像处理 | P0 | 图像的熵触发、嵌入生成与冷存储 |
| M-03 | 音频处理 | P0 | VAD 触发、语音转文本与 MAU 生成 |
| M-04 | 视频处理 | P1 | 帧级熵触发、关键帧提取与 MAU 生成 |
| M-05 | 金字塔检索 | P0 | 摘要预览 → 按需展开的 token 高效检索 |
| M-06 | BM25 混合检索 | P0 | FAISS + BM25 关键词混合搜索 |
| M-07 | 知识图谱 | P1 | 实体抽取、关系构建、多跳推理 |
| M-08 | 参数化记忆 | P1 | 频繁访问记忆蒸馏为参数化知识 |
| M-09 | 记忆合并 | P1 | 基于重要性的记忆保留与归档 |
| M-10 | 按需图像加载 | P1 | 视觉记忆的延迟加载与多模态提示 |

### 4.3 自演化检索 (EvolveMem)

| ID | 功能 | 优先级 | 描述 |
|:--|:--|:--|:--|
| E-01 | 闭环演化 | P0 | Extract → Index → Retrieve → Answer → Evaluate → Diagnose → Adjust |
| E-02 | LLM 诊断 | P0 | 基于失败案例的 LLM 驱动根因分析 |
| E-03 | 精英演化 | P0 | 严格爬山策略，每轮仅接受 F1 提升的配置变更 |
| E-04 | 配置建议 | P0 | 诊断引擎生成参数化配置建议 |
| E-05 | 食谱系统 | P1 | 预编译的能力食谱，症状匹配后原子化应用 |
| E-06 | 元分析 | P1 | 跨轮次的元演化分析，支持回退/聚焦/探索 |
| E-07 | 意图规划 | P1 | SimpleMem 风格的查询意图分析与子查询生成 |
| E-08 | 查询分解 | P1 | 多跳问题的子问题分解与合并检索 |
| E-09 | 答案验证 | P1 | 二次 LLM 调用验证/纠正候选答案 |
| E-10 | 基准适配 | P1 | LoCoMo / MemBench / LongMemEval 适配器 |

### 4.4 跨会话记忆 (Cross-Session)

| ID | 功能 | 优先级 | 描述 |
|:--|:--|:--|:--|
| C-01 | 会话生命周期 | P0 | start → record → stop(finalize) → end |
| C-02 | 上下文注入 | P0 | 基于 token 预算的跨会话上下文组装 |
| C-03 | 语义搜索 | P0 | 跨所有会话的向量相似度搜索 |
| C-04 | 记忆持久化 | P0 | SQLite + LanceDB 双存储 |
| C-05 | 多租户隔离 | P1 | 基于 tenant_id 的数据隔离 |

### 4.5 MCP 服务

| ID | 功能 | 优先级 | 描述 |
|:--|:--|:--|:--|
| S-01 | MCP 协议 | P0 | Streamable HTTP + JSON-RPC 2.0 |
| S-02 | 多租户 | P0 | Token 认证 + 每用户数据表 |
| S-03 | 混合检索 | P0 | 语义 + 关键词 + 元数据过滤 |
| S-04 | Docker 部署 | P1 | Docker Compose 一键部署 |

## 5. 非功能需求

| ID | 类别 | 需求 | 指标 |
|:--|:--|:--|:--|
| N-01 | 性能 | 并行记忆构建 | 支持 8 workers 并行 |
| N-02 | 性能 | Token 效率 | 推理 token 消耗降低 ~30x |
| N-03 | 可扩展性 | 云存储 | LanceDB 支持 GCS/S3/Azure |
| N-04 | 兼容性 | Python 版本 | >= 3.10 |
| N-05 | 兼容性 | LLM 后端 | OpenAI / Qwen / Azure / Ollama |
| N-06 | 可部署性 | 安装方式 | pip install / Docker / MCP 云服务 |
| N-07 | 可观测性 | 遥测 | 演化过程全量日志与指标 |

## 6. 数据模型

### 6.1 MemoryEntry (文本记忆单元)

```
MemoryEntry
├── entry_id: str (UUID)
├── lossless_restatement: str (语义无损重述)
├── keywords: List[str] (词汇层索引)
├── timestamp: Optional[str] (ISO 8601)
├── location: Optional[str]
├── persons: List[str] (符号层索引)
├── entities: List[str] (符号层索引)
└── topic: Optional[str]
```

### 6.2 MAU (多模态原子单元)

```
MultimodalAtomicUnit
├── id: str
├── timestamp: float
├── modality_type: TEXT | VISUAL | AUDIO | VIDEO | MULTIMODAL
├── summary: str (轻量摘要)
├── embedding: List[float] (稠密向量)
├── raw_pointer: Optional[str] (冷存储指针)
├── details: Optional[Dict] (延迟加载)
├── metadata: MAUMetadata
│   ├── session_id, source, tags
│   ├── quality: QualityMetrics (trigger_score, confidence, entropy_delta)
│   ├── persons, entities, keywords, location, topic
│   └── duration_ms, frame_index, speaker_id
├── links: MAULinks
│   ├── event_id, prev_mau_id, next_mau_id
│   └── related_mau_ids
├── status: ACTIVE | ARCHIVED | PINNED
└── storage_tier: HOT | COLD
```

### 6.3 RetrievalConfig (检索配置)

```
RetrievalConfig
├── semantic_top_k, keyword_top_k, structured_top_k
├── max_context, fusion_mode, weights
├── enable_entity_swap, enable_query_decomposition
├── enable_intent_planning, enable_answer_verification
├── reflection_rounds, answer_style
├── per_category_overrides: Dict[str, Dict]
└── benchmark-specific flags (locomo_*, longmemeval_*, membench_*)
```

## 7. 接口规范

### 7.1 统一 API (AutoMemory)

```python
class AutoMemory:
    # 文本模式 API
    def add_dialogue(speaker, content, timestamp) -> None
    def add_dialogues(dialogues) -> None
    def finalize() -> None
    def ask(question) -> str

    # 多模态 API
    def add_text(text, **kwargs) -> ProcessingResult
    def add_image(image, **kwargs) -> ProcessingResult
    def add_audio(audio, **kwargs) -> ProcessingResult
    def add_video(video_path, **kwargs) -> ProcessingResult
    def query(query, top_k, **kwargs) -> RetrievalResult

    # 通用
    def close() -> None
    def get_all_memories() -> List[MemoryEntry]
```

### 7.2 演化 API

```python
def optimize(mem, dev_questions, max_rounds=7, **kwargs) -> Config
class Config:
    def save(path) -> None
    @classmethod
    def from_file(path) -> Config
```

### 7.3 跨会话 API

```python
class CrossMemOrchestrator:
    async def start_session(content_session_id, user_prompt) -> Dict
    async def record_message(memory_session_id, content, role) -> None
    async def record_tool_use(memory_session_id, tool_name, tool_input, tool_output) -> None
    async def stop_session(memory_session_id) -> FinalizationReport
    async def end_session(memory_session_id) -> None
    def search(query, top_k) -> List[CrossMemoryEntry]
    def get_context_for_prompt(user_prompt) -> str
```

## 8. 约束与依赖

| 约束 | 描述 |
|:--|:--|
| LLM API | 需要 OpenAI 兼容的 API 密钥 |
| 向量数据库 | LanceDB (本地/云存储) |
| 嵌入模型 | Qwen3-Embedding / OpenAI Embeddings |
| 关系数据库 | SQLite (跨会话存储) |
| Python | >= 3.10 |
| 全文搜索 | Tantivy (本地) / LanceDB Native FTS (云) |

## 9. 版本与路线图

- **v1.0**: 文本记忆核心 (SimpleMem)
- **v2.0**: 多模态记忆 (Omni-SimpleMem)
- **v3.0**: 自演化检索 (EvolveMem)
- **当前**: 统一包 `simplemem` v0.3.0，自动路由

### 计划中

- [ ] MCP 支持多模态 (image/audio/video tools)
- [ ] MCP 支持 EvolveMem optimize() 工具
- [ ] Docker 继承多模态与自演化能力
