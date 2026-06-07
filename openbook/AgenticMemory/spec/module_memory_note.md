# 记忆笔记与元数据提取模块设计文档

## 1. 模块概述

MemoryNote / RobustMemoryNote 是 AgenticMemory 系统的基本记忆单元，负责存储记忆内容和元数据，并通过 LLM 自动提取元数据（关键词、上下文、标签）。系统提供两个实现版本：

- **标准版** (`MemoryNote`)：依赖 JSON Schema 强制结构化输出，适用于支持 `response_format` 的 LLM 后端（如 OpenAI）
- **鲁棒版** (`RobustMemoryNote`)：使用纯文本 Prompt + Section-Marker 解析，兼容任意 LLM 后端，具备多层降级策略

---

## 2. MemoryNote 数据模型

### 2.1 类图

```mermaid
classDiagram
    class MemoryNote {
        +str id
        +str content
        +List~str~ keywords
        +List~Dict~ links
        +float importance_score
        +int retrieval_count
        +str timestamp
        +str last_accessed
        +str context
        +List~Any~ evolution_history
        +str category
        +List~str~ tags
        +analyze_content(content, llm_controller) Dict
    }

    class RobustMemoryNote {
        +str id
        +str content
        +List~str~ keywords
        +List~Dict~ links
        +float importance_score
        +int retrieval_count
        +str timestamp
        +str last_accessed
        +str context
        +List~Any~ evolution_history
        +str category
        +List~str~ tags
        +analyze_content(content, llm_controller) Dict
    }

    class LLMController {
        +BaseLLMController llm
        +__init__(backend, model, api_key, ...)
    }

    class RobustLLMController {
        +RobustBaseLLMController llm
        +__init__(backend, model, api_key, ..., check_connection)
    }

    MemoryNote --> LLMController : 使用
    RobustMemoryNote --> RobustLLMController : 使用

    note for MemoryNote "标准版：JSON Schema 强制输出"
    note for RobustMemoryNote "鲁棒版：纯文本 Prompt + 降级策略"
```

### 2.2 属性说明

| 属性 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `id` | `str` | `uuid.uuid4()` | 唯一标识符 |
| `content` | `str` | 必填 | 记忆内容 |
| `keywords` | `List[str]` | `[]` | 关键词列表 |
| `links` | `List[Dict]` | `[]` | 连接的其他记忆 ID 列表 |
| `importance_score` | `float` | `1.0` | 重要性评分 |
| `retrieval_count` | `int` | `0` | 被检索次数 |
| `timestamp` | `str` | 当前时间 | 创建时间（格式 `%Y%m%d%H%M`） |
| `last_accessed` | `str` | 当前时间 | 最后访问时间 |
| `context` | `str` | `"General"` | 上下文描述（若为 list 则自动 join） |
| `evolution_history` | `List[Any]` | `[]` | 进化历史 |
| `category` | `str` | `"Uncategorized"` | 分类 |
| `tags` | `List[str]` | `[]` | 标签列表 |

### 2.3 构造函数元数据提取逻辑

```mermaid
flowchart TD
    A["创建 MemoryNote / RobustMemoryNote"] --> B{"llm_controller 存在？"}
    B -- 否 --> C["使用传入值或默认值"]
    B -- 是 --> D{"keywords / context /<br/>category / tags 有缺失？"}
    D -- 否 --> C
    D -- 是 --> E["调用 analyze_content()"]
    E --> F["用分析结果填充缺失字段"]
    F --> G["设置默认值"]
    C --> G
    G --> H["返回 MemoryNote 实例"]
```

---

## 3. 元数据提取流程

### 3.1 标准版提取流程（时序图）

```mermaid
sequenceDiagram
    participant Note as MemoryNote.__init__
    participant Analyze as MemoryNote.analyze_content
    participant LLM as LLMController
    participant Backend as OpenAI / Ollama / SGLang

    Note->>Analyze: 调用 analyze_content(content, llm_controller)
    Analyze->>Analyze: 构造 Prompt（含 JSON 格式要求）
    Analyze->>LLM: get_completion(prompt, response_format)
    LLM->>Backend: 请求（带 JSON Schema 约束）
    Backend-->>LLM: JSON 字符串响应
    LLM-->>Analyze: 返回 response 字符串

    alt JSON 解析成功
        Analyze->>Analyze: json.loads(response)
        Analyze-->>Note: 返回 {"keywords", "context", "tags"}
    else JSON 解析失败
        Analyze->>Analyze: 返回空默认值<br/>{"keywords": [], "context": "General", "tags": []}
    end

    alt LLM 调用异常
        Analyze->>Analyze: 返回空默认值<br/>含 "category": "Uncategorized"
    end
```

### 3.2 鲁棒版提取流程（时序图）

```mermaid
sequenceDiagram
    participant Note as RobustMemoryNote.__init__
    participant Analyze as RobustMemoryNote.analyze_content
    participant LLM as RobustLLMController
    participant Parser as llm_text_parsers
    participant Backend as OpenAI / Ollama / SGLang / vLLM

    Note->>Analyze: 调用 analyze_content(content, llm_controller)
    Analyze->>Analyze: 构造 Prompt（ANALYZE_CONTENT_PROMPT）
    Analyze->>LLM: get_completion(prompt)
    LLM->>Backend: 请求（纯文本，无 JSON Schema）
    Backend-->>LLM: 纯文本响应
    LLM-->>Analyze: 返回 response 字符串

    Analyze->>Parser: parse_analyze_content(response, content)

    alt 尝试 JSON 解析
        Parser->>Parser: strip_markdown_fences → json.loads
    else JSON 失败，回退 Section-Marker
        Parser->>Parser: _extract_section("KEYWORDS")<br/>_extract_section("CONTEXT")<br/>_extract_section("TAGS")
    end

    Parser->>Parser: validate_analysis_result(result, content)

    alt keywords 仍为空
        Analyze->>LLM: get_completion(FOCUSED_KEYWORDS_PROMPT)
        LLM->>Backend: 重试请求
        Backend-->>LLM: 响应
        LLM-->>Analyze: retry_response
        Analyze->>Parser: _parse_list_items(retry_response)
    end

    Analyze->>Parser: validate_analysis_result(analysis, content)
    Parser-->>Analyze: 最终结果

    alt 全部失败（异常）
        Analyze->>Parser: _heuristic_keywords(content)
        Analyze->>Parser: _heuristic_context(content)
        Analyze-->>Note: 返回启发式结果
    else 正常
        Analyze-->>Note: 返回 {"keywords", "context", "tags"}
    end
```

---

## 4. 标准版 vs 鲁棒版提取对比

```mermaid
flowchart LR
    subgraph 标准版["MemoryNote（标准版）"]
        direction TB
        S1["Prompt 含 JSON 格式要求"] --> S2["LLM 调用带 response_format<br/>(JSON Schema)"]
        S2 --> S3["json.loads 解析"]
        S3 --> S4{"解析成功？"}
        S4 -- 是 --> S5["返回结构化结果"]
        S4 -- 否 --> S6["返回空默认值"]
    end

    subgraph 鲁棒版["RobustMemoryNote（鲁棒版）"]
        direction TB
        R1["Prompt 使用 Section-Marker"] --> R2["LLM 调用（纯文本）"]
        R2 --> R3["先尝试 JSON 解析"]
        R3 --> R4{"JSON 成功？"}
        R4 -- 是 --> R5["validate_analysis_result"]
        R4 -- 否 --> R6["Section-Marker 解析"]
        R6 --> R5
        R5 --> R7{"keywords 为空？"}
        R7 -- 否 --> R8["返回结果"]
        R7 -- 是 --> R9["FOCUSED_KEYWORDS_PROMPT 重试"]
        R9 --> R5
    end

    标准版 -.->|对比| 鲁棒版
```

### 关键差异对照表

| 维度 | 标准版 (MemoryNote) | 鲁棒版 (RobustMemoryNote) |
|------|---------------------|---------------------------|
| **Prompt 格式** | 内联 JSON 格式要求 | `ANALYZE_CONTENT_PROMPT` 模板 |
| **LLM 调用参数** | `response_format` (JSON Schema) | 纯文本，无 `response_format` |
| **响应解析** | `json.loads` | 先 JSON → 回退 Section-Marker |
| **关键词为空处理** | 无重试 | `FOCUSED_KEYWORDS_PROMPT` 重试 |
| **验证修复** | 无 | `validate_analysis_result` |
| **最终降级** | 返回空默认值 | 启发式方法 (`_heuristic_*`) |
| **LLM 后端支持** | OpenAI, Ollama, SGLang | OpenAI, Ollama, SGLang, vLLM |
| **重试机制** | 无 | `retry_llm_call` 装饰器（指数退避） |
| **连接检查** | 无 | `check_connectivity()` |
| **日志** | `print()` | `logging` 结构化日志 |

---

## 5. 鲁棒版降级策略流程图

```mermaid
flowchart TD
    A["analyze_content(content, llm_controller)"] --> B["构造 ANALYZE_CONTENT_PROMPT"]
    B --> C["llm.get_completion(prompt)"]

    C -->|"成功"| D["parse_analyze_content(response, content)"]
    C -->|"异常"| E["进入启发式降级"]

    D --> D1["先尝试 JSON 解析<br/>strip_markdown_fences → json.loads"]
    D1 -->|"JSON 成功"| D3["validate_analysis_result"]
    D1 -->|"JSON 失败"| D2["Section-Marker 解析<br/>_extract_section × 3"]
    D2 --> D3

    D3 --> D4{"keywords 为空？"}
    D4 -- 否 --> D5["返回最终结果 ✓"]
    D4 -- 是 --> D6["FOCUSED_KEYWORDS_PROMPT 重试"]
    D6 --> D7["llm.get_completion(retry_prompt, temperature=0.3)"]
    D7 -->|"成功"| D8["_parse_list_items(retry_response)"]
    D7 -->|"失败"| E
    D8 --> D3

    E --> E1["_heuristic_keywords(content)"]
    E --> E2["_heuristic_context(content)"]
    E --> E3["tags = _heuristic_keywords(content, 3)"]
    E1 --> E4["返回启发式结果 ✓"]
    E2 --> E4
    E3 --> E4

    style E fill:#ffcccc
    style D5 fill:#ccffcc
    style E4 fill:#ffffcc
```

### 降级层级说明

| 层级 | 策略 | 触发条件 |
|------|------|----------|
| **L1** | JSON 解析 | LLM 返回有效 JSON（即使未要求 JSON Schema） |
| **L2** | Section-Marker 解析 | JSON 解析失败，使用 `KEYWORDS:` / `CONTEXT:` / `TAGS:` 标记 |
| **L3** | FOCUSED_KEYWORDS 重试 | keywords 解析后为空，使用聚焦 Prompt 重新请求 LLM |
| **L4** | 启发式方法 | LLM 调用完全失败，基于规则提取关键词和上下文 |

---

## 6. 验证与修复流程

### 6.1 validate_analysis_result 流程图

```mermaid
flowchart TD
    A["validate_analysis_result(result, content)"] --> B{"result 是 dict？"}
    B -- 否 --> C["重置为空结构<br/>{keywords: [], context: '', tags: []}"]
    B -- 是 --> D["提取 keywords, context, tags"]
    C --> D

    D --> E{"keywords 是 str？"}
    E -- 是 --> F["_parse_list_items(keywords)"]
    E -- 否 --> G["保持 list"]
    F --> H{"tags 是 str？"}
    G --> H

    H -- 是 --> I["_parse_list_items(tags)"]
    H -- 否 --> J["保持 list"]
    I --> K{"context 是 list？"}
    J --> K

    K -- 是 --> L["' '.join(context)"]
    K -- 否 --> M["保持 str"]
    L --> N{"keywords 为空？"}
    M --> N

    N -- 是且 content 非空 --> O["_heuristic_keywords(content)"]
    N -- 否 --> P{"context 为空？"}
    O --> P

    P -- 是且 content 非空 --> Q["_heuristic_context(content)"]
    P -- 否 --> R{"tags 为空？"}
    Q --> R

    R -- 是且 keywords 非空 --> S["tags = keywords[:3]"]
    R -- 否 --> T["返回修复后的 result ✓"]
    S --> T

    style O fill:#fff3cd
    style Q fill:#fff3cd
    style S fill:#d1ecf1
```

### 6.2 启发式方法详解

#### _heuristic_keywords

```mermaid
flowchart LR
    A["输入: content 文本"] --> B["正则提取 ≥3 字符的英文单词"]
    B --> C["过滤停用词（约 80 个）"]
    C --> D["去重"]
    D --> E["评分：首字母大写 +2 分<br/>其他 +1 分"]
    E --> F["按分数降序排列"]
    F --> G["取前 max_keywords 个"]
    G --> H["输出: 关键词列表"]
```

**停用词列表涵盖**：冠词、连词、介词、代词、助动词、常见副词等（如 `the`, `is`, `have`, `about`, `said`, `speaker` 等）。

#### _heuristic_context

```mermaid
flowchart LR
    A["输入: content 文本"] --> B{"匹配第一句话<br/>(.+?[.!?])\\s"}
    B -- 匹配成功 --> C["返回第一句话"]
    B -- 匹配失败 --> D["返回前 200 字符"]
    C --> E["输出: context 字符串"]
    D --> E
```

---

## 7. LLM 控制器架构

### 7.1 标准版控制器

```mermaid
classDiagram
    class BaseLLMController {
        <<abstract>>
        +get_completion(prompt, response_format, temperature) str
    }

    class OpenAIController {
        +str model
        +OpenAI client
        +get_completion(prompt, response_format, temperature) str
    }

    class OllamaController {
        +str model
        +get_completion(prompt, response_format, temperature) str
        +_generate_empty_value(schema_type, schema_items) Any
        +_generate_empty_response(response_format) dict
    }

    class SGLangController {
        +str model
        +str base_url
        +get_completion(prompt, response_format, temperature) str
        +_generate_empty_value(schema_type, schema_items) Any
        +_generate_empty_response(response_format) dict
    }

    class LiteLLMController {
        +str model
        +str api_base
        +str api_key
        +get_completion(prompt, response_format, temperature) str
        +_generate_empty_value(schema_type, schema_items) Any
        +_generate_empty_response(response_format) dict
    }

    class LLMController {
        +BaseLLMController llm
        +__init__(backend, model, api_key, ...)
    }

    BaseLLMController <|-- OpenAIController
    BaseLLMController <|-- OllamaController
    BaseLLMController <|-- SGLangController
    BaseLLMController <|-- LiteLLMController
    LLMController --> BaseLLMController : 持有
```

### 7.2 鲁棒版控制器

```mermaid
classDiagram
    class RobustBaseLLMController {
        <<abstract>>
        +SYSTEM_MESSAGE: str
        +get_completion(prompt, temperature) str
        +check_connectivity() void
    }

    class RobustOpenAIController {
        +str model
        +OpenAI client
        +get_completion(prompt, temperature) str
    }

    class RobustOllamaController {
        +str model
        +get_completion(prompt, temperature) str
    }

    class RobustSGLangController {
        +str model
        +str base_url
        +get_completion(prompt, temperature) str
    }

    class RobustVLLMController {
        +str model
        +str base_url
        +get_completion(prompt, temperature) str
    }

    class RobustLiteLLMController {
        +str model
        +str api_base
        +str api_key
        +get_completion(prompt, temperature) str
    }

    class RobustLLMController {
        +RobustBaseLLMController llm
        +__init__(backend, model, ..., check_connection)
    }

    RobustBaseLLMController <|-- RobustOpenAIController
    RobustBaseLLMController <|-- RobustOllamaController
    RobustBaseLLMController <|-- RobustSGLangController
    RobustBaseLLMController <|-- RobustVLLMController
    RobustBaseLLMController <|-- RobustLiteLLMController
    RobustLLMController --> RobustBaseLLMController : 持有

    note for RobustBaseLLMController "关键差异：无 response_format 参数\n所有方法均使用 @retry_llm_call 装饰"
```

---

## 8. 解析工具链（llm_text_parsers.py）

### 8.1 解析函数总览

```mermaid
flowchart TD
    subgraph Prompt模板
        P1["ANALYZE_CONTENT_PROMPT"]
        P2["EVOLUTION_DECISION_PROMPT"]
        P3["STRENGTHEN_DETAILS_PROMPT"]
        P4["UPDATE_NEIGHBORS_PROMPT"]
        P5["FOCUSED_KEYWORDS_PROMPT"]
    end

    subgraph 解析函数
        F1["parse_analyze_content()"]
        F2["parse_evolution_decision()"]
        F3["parse_strengthen_details()"]
        F4["parse_update_neighbors()"]
    end

    subgraph 基础工具
        T1["parse_with_json_fallback()"]
        T2["_extract_section()"]
        T3["_parse_list_items()"]
        T4["strip_markdown_fences()"]
    end

    subgraph 验证修复
        V1["validate_analysis_result()"]
        V2["_heuristic_keywords()"]
        V3["_heuristic_context()"]
    end

    P1 --> F1
    P2 --> F2
    P3 --> F3
    P4 --> F4

    F1 --> T1
    F2 --> T1
    F3 --> T1
    F4 --> T1

    T1 --> T4
    T1 --> T2
    T2 --> T3

    F1 --> V1
    V1 --> V2
    V1 --> V3
```

### 8.2 parse_with_json_fallback 通用解析策略

```mermaid
flowchart TD
    A["输入: LLM 响应文本"] --> B["strip_markdown_fences()"]
    B --> C["json.loads()"]
    C -->|"成功且为 dict"| D["返回 JSON 解析结果"]
    C -->|"失败"| E["调用 plain_text_parser()"]
    E --> F["Section-Marker 解析"]
    F --> G["返回解析结果"]
```

### 8.3 _extract_section 工作原理

```mermaid
flowchart LR
    A["输入: text, marker, next_markers"] --> B["正则匹配 MARKER: 行"]
    B --> C["提取标记后的同行内容"]
    C --> D["查找下一个标记的位置"]
    D --> E["截取两标记之间的文本"]
    E --> F["合并同行内容 + 后续行"]
    F --> G["输出: section 文本"]
```

---

## 9. 源文件索引

| 文件 | 核心类/函数 | 说明 |
|------|------------|------|
| `memory_layer.py` | `MemoryNote`, `LLMController` | 标准版记忆单元与 LLM 控制器 |
| `memory_layer_robust.py` | `RobustMemoryNote`, `RobustLLMController` | 鲁棒版记忆单元与 LLM 控制器 |
| `llm_text_parsers.py` | Prompt 模板、`parse_*`、`validate_*`、`_heuristic_*` | 纯文本解析、验证与启发式修复 |
