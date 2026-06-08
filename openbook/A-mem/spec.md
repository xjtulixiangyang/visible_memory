# A-MEM: Agentic Memory 规格说明文档

## 1. 项目概述

**项目名称**: A-MEM (Agentic Memory for LLM Agents)

**版本**: 0.0.1

**论文**: [A-MEM: Agentic Memory for LLM Agents](https://arxiv.org/pdf/2502.12110)

**核心定位**: 一种面向 LLM Agent 的新型智能记忆系统，基于 Zettelkasten 原则实现记忆的动态组织、演化和互联。

---

## 2. 项目目标

| 目标 | 描述 |
|------|------|
| 动态记忆组织 | 基于 Zettelkasten 原则，自动对记忆进行索引、链接和分类 |
| 智能检索 | 通过 ChromaDB 向量数据库实现语义相似性搜索 |
| 记忆演化 | 记忆可随时间自动演化，更新标签、上下文和关联关系 |
| 多后端支持 | 支持 OpenAI 和 Ollama 两种 LLM 后端 |
| 知识网络 | 构建记忆间的互联知识网络，支持邻居记忆检索 |

---

## 3. 系统功能规格

### 3.1 记忆管理 (CRUD)

| 功能 | 方法 | 描述 |
|------|------|------|
| 创建记忆 | `add_note()` | 添加新记忆，自动触发内容分析和演化决策 |
| 读取记忆 | `read()` | 根据 ID 获取单个记忆笔记 |
| 更新记忆 | `update()` | 更新记忆的任意字段，同步更新 ChromaDB |
| 删除记忆 | `delete()` | 从内存和 ChromaDB 中同时删除记忆 |

### 3.2 内容分析

| 功能 | 方法 | 描述 |
|------|------|------|
| 关键词提取 | `analyze_content()` | 使用 LLM 提取记忆内容的关键词 |
| 上下文生成 | `analyze_content()` | 自动生成记忆的上下文描述 |
| 标签分类 | `analyze_content()` | 自动为记忆生成分类标签 |

### 3.3 记忆检索

| 功能 | 方法 | 描述 |
|------|------|------|
| 语义搜索 | `search()` | 基于 ChromaDB 的语义相似性搜索 |
| 智能搜索 | `search_agentic()` | 语义搜索 + 邻居记忆扩展检索 |
| 关联查找 | `find_related_memories()` | 查找与查询相关的记忆（格式化输出） |
| 原始关联查找 | `find_related_memories_raw()` | 查找关联记忆并包含链接记忆 |

### 3.4 记忆演化

| 功能 | 方法 | 描述 |
|------|------|------|
| 演化决策 | `process_memory()` | LLM 判断记忆是否需要演化 |
| 连接强化 | `strengthen` action | 建立记忆间的链接，更新标签 |
| 邻居更新 | `update_neighbor` action | 更新邻居记忆的上下文和标签 |
| 记忆整合 | `consolidate_memories()` | 重建 ChromaDB 索引，同步所有记忆 |

---

## 4. 数据模型

### 4.1 MemoryNote 数据结构

```
MemoryNote
├── id: str                    # UUID 唯一标识
├── content: str               # 记忆文本内容
├── keywords: List[str]        # 关键词列表
├── links: List[str]           # 关联记忆 ID 列表
├── context: str               # 上下文描述（默认 "General"）
├── category: str              # 分类（默认 "Uncategorized"）
├── tags: List[str]            # 标签列表
├── timestamp: str             # 创建时间 (YYYYMMDDHHmm)
├── last_accessed: str         # 最后访问时间
├── retrieval_count: int       # 检索次数
└── evolution_history: List    # 演化历史记录
```

### 4.2 ChromaDB 元数据映射

MemoryNote 的所有字段均作为 ChromaDB 文档的 metadata 存储，其中 `keywords`、`links`、`tags`、`evolution_history` 等 List/Dict 类型会序列化为 JSON 字符串存储，检索时自动反序列化。

---

## 5. 外部依赖

| 依赖 | 版本要求 | 用途 |
|------|----------|------|
| sentence-transformers | >=2.2.2 | 文本嵌入生成 |
| chromadb | >=0.4.22 | 向量数据库存储与检索 |
| rank_bm25 | >=0.2.2 | BM25 关键词检索 |
| nltk | >=3.8.1 | 文本分词 |
| litellm | >=1.16.11 | 统一 LLM API 调用 |
| numpy | >=1.24.3 | 数值计算 |
| scikit-learn | >=1.3.2 | 余弦相似度计算 |
| openai | >=1.3.7 | OpenAI API 客户端 |

---

## 6. 接口规格

### 6.1 AgenticMemorySystem 初始化参数

| 参数 | 类型 | 默认值 | 描述 |
|------|------|--------|------|
| model_name | str | 'all-MiniLM-L6-v2' | Sentence Transformer 嵌入模型 |
| llm_backend | str | "openai" | LLM 后端 ("openai" / "ollama") |
| llm_model | str | "gpt-4o-mini" | LLM 模型名称 |
| evo_threshold | int | 100 | 触发记忆整合的演化计数阈值 |
| api_key | Optional[str] | None | OpenAI API Key |

### 6.2 LLM Controller 接口

| 后端 | 类 | 说明 |
|------|-----|------|
| OpenAI | `OpenAIController` | 通过 OpenAI SDK 调用，支持 JSON Schema 响应格式 |
| Ollama | `OllamaController` | 通过 litellm 调用本地 Ollama，含空响应兜底机制 |

### 6.3 Retriever 接口

| 类 | 说明 |
|-----|------|
| `ChromaRetriever` | 内存模式 ChromaDB 客户端 |
| `PersistentChromaRetriever` | 持久化 ChromaDB 客户端，支持跨会话 |
| `CopiedChromaRetriever` | 从现有集合复制到临时数据库的客户端 |

---

## 7. 演化机制规格

### 7.1 演化触发条件

- 每次调用 `add_note()` 时自动执行 `process_memory()`
- 当 `evo_cnt` 达到 `evo_threshold` 的整数倍时，触发 `consolidate_memories()`

### 7.2 演化决策 JSON Schema

```json
{
  "should_evolve": boolean,
  "actions": ["strengthen", "update_neighbor"],
  "suggested_connections": ["neighbor_memory_ids"],
  "tags_to_update": ["tag_1", ..., "tag_n"],
  "new_context_neighborhood": ["new context", ..., "new context"],
  "new_tags_neighborhood": [["tag_1", ..., "tag_n"], ..., ["tag_1", ..., "tag_n"]]
}
```

### 7.3 演化动作

| 动作 | 效果 |
|------|------|
| `strengthen` | 将建议连接的记忆 ID 添加到当前记忆的 `links`，更新当前记忆的 `tags` |
| `update_neighbor` | 按顺序更新邻居记忆的 `tags` 和 `context` |

---

## 8. 测试覆盖

| 测试文件 | 覆盖范围 |
|----------|----------|
| `test_memory_system.py` | CRUD、元数据持久化、记忆关系、演化、删除、整合、关联查找 |
| `test_retriever.py` | ChromaDB 初始化、文档增删查、元数据类型转换、持久化、集合访问控制 |
| `test_utils.py` | MockLLMController 用于测试替身 |
