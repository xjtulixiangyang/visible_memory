# 评估系统模块设计文档

## 1. 模块概述

评估系统在 LoComo 基准数据集上评估 A-MEM 系统的问答性能。系统包含两种评估 Agent 实现（标准版与鲁棒版），通过多维度指标（精确匹配、F1、ROUGE、BLEU、BERTScore、METEOR、SBERT 相似度）对预测答案与参考答案进行量化评估，并按问题类别聚合统计结果。

### 核心文件

| 文件 | 职责 |
|------|------|
| `test_advanced.py` | 标准版评估 Agent 与评估流程 |
| `test_advanced_robust.py` | 鲁棒版评估 Agent（无 JSON Schema 依赖） |
| `utils.py` | 评估指标计算与聚合 |
| `load_dataset.py` | LoComo 数据集加载与解析 |
| `llm_text_parsers.py` | 纯文本 LLM 响应解析器（鲁棒版专用） |

---

## 2. 评估系统整体架构

```mermaid
graph TB
    subgraph 数据层
        DS[LoComo 数据集 JSON]
        LD[load_dataset.py]
    end

    subgraph Agent 层
        AA[advancedMemAgent<br/>标准版]
        RA[RobustAdvancedMemAgent<br/>鲁棒版]
    end

    subgraph 记忆系统层
        AMS[AgenticMemorySystem]
        RAMS[RobustAgenticMemorySystem]
        LC[LLMController]
        RLC[RobustLLMController]
    end

    subgraph 缓存层
        MC[memory_cache_sample_X.pkl]
        RC[retriever_cache_sample_X.pkl]
        RE[retriever_cache_embeddings_sample_X.npy]
    end

    subgraph 评估层
        UT[utils.py]
        CM[calculate_metrics]
        AM[aggregate_metrics]
    end

    DS --> LD
    LD --> AA
    LD --> RA

    AA --> AMS
    AA --> LC
    RA --> RAMS
    RA --> RLC

    AMS --> MC
    AMS --> RC
    AMS --> RE
    RAMS --> MC
    RAMS --> RC
    RAMS --> RE

    AA --> UT
    RA --> UT
    UT --> CM
    UT --> AM
```

---

## 3. 问答评估时序图

```mermaid
sequenceDiagram
    participant Main as evaluate_dataset
    participant Agent as advancedMemAgent / RobustAdvancedMemAgent
    participant MemSys as MemorySystem
    participant LLM as LLMController
    participant Utils as utils.py

    Main->>Main: load_locomo_dataset(dataset_path)
    Main->>Main: 按 ratio 截取样本子集

    loop 每个 LoCoMoSample
        Main->>Agent: 创建 Agent(model, backend, ...)
        Main->>Main: 检查记忆缓存是否存在

        alt 缓存存在
            Main->>Main: 加载 memory_cache_sample_X.pkl
            Main->>MemSys: 恢复 memories + retriever
        else 缓存不存在
            loop 每个 Session 的每个 Turn
                Main->>Agent: add_memory(conversation_text, time)
                Agent->>MemSys: add_note(content, time)
            end
            Main->>Main: 保存 memory_cache + retriever_cache
        end

        loop 每个 QA 对
            Main->>Agent: answer_question(question, category, answer)
            Agent->>LLM: generate_query_llm(question)
            LLM-->>Agent: keywords
            Agent->>MemSys: retrieve_memory(keywords, k)
            MemSys-->>Agent: raw_context
            Agent->>LLM: get_completion(prompt + context)
            LLM-->>Agent: prediction
            Agent-->>Main: prediction, user_prompt, raw_context

            Main->>Utils: calculate_metrics(prediction, reference)
            Utils-->>Main: metrics dict
        end
    end

    Main->>Utils: aggregate_metrics(all_metrics, all_categories)
    Utils-->>Main: aggregate_results
    Main->>Main: 输出结果到 JSON + 日志
```

---

## 4. 记忆缓存流程

```mermaid
flowchart TD
    START([开始处理 Sample]) --> CHECK{缓存文件<br/>是否存在?}

    CHECK -->|是| LOAD_MEM[加载 memory_cache_sample_X.pkl]
    LOAD_MEM --> LOAD_RET{retriever_cache<br/>是否存在?}
    LOAD_RET -->|是| LOAD_BOTH[加载 retriever_cache + embeddings]
    LOAD_RET -->|否| REBUILD_RET[从 memories 重建 retriever<br/>load_from_local_memory]
    LOAD_BOTH --> READY[Agent 就绪]
    REBUILD_RET --> READY

    CHECK -->|否| CREATE[创建新 Agent]
    CREATE --> LOOP[遍历所有 Session / Turn]
    LOOP --> ADD[add_memory<br/>Speaker X says: text<br/>time=date_time]
    ADD --> MORE{还有更多<br/>Turn?}
    MORE -->|是| LOOP
    MORE -->|否| SAVE_MEM[保存 memory_cache_sample_X.pkl]
    SAVE_MEM --> SAVE_RET[保存 retriever_cache + embeddings.npy]
    SAVE_RET --> READY

    READY --> EVAL[开始 QA 评估]

    subgraph 缓存文件结构
        direction LR
        D[cached_memories_advanced_backend_model/]
        D --> M[memory_cache_sample_0.pkl]
        D --> R[retriever_cache_sample_0.pkl]
        D --> E[retriever_cache_embeddings_sample_0.npy]
    end
```

---

## 5. answer_question 类别处理流程

```mermaid
flowchart TD
    START([answer_question]) --> GEN_KW[generate_query_llm<br/>从问题生成查询关键词]
    GEN_KW --> RETRIEVE[retrieve_memory<br/>检索相关记忆 k 条]
    RETRIEVE --> CAT{category?}

    CAT -->|1| C1["短语回答<br/>使用原文词汇<br/>temperature=0.7"]
    CAT -->|2| C2["日期相关问题<br/>使用对话日期<br/>temperature=0.7"]
    CAT -->|3| C3["短语回答<br/>使用原文词汇<br/>temperature=0.7"]
    CAT -->|4| C4["短语回答<br/>使用原文词汇<br/>temperature=0.7"]
    CAT -->|5| C5["对抗性问题<br/>二选一格式<br/>temperature=temperature_c5"]

    C1 --> PROMPT1["Based on the context...<br/>short phrase...<br/>exact words from context"]
    C2 --> PROMPT2["Use DATE of CONVERSATION<br/>approximate date<br/>shortest possible answer"]
    C3 --> PROMPT3["short phrase...<br/>exact words from context"]
    C4 --> PROMPT4["short phrase...<br/>exact words from context"]
    C5 --> PROMPT5["Select the correct answer:<br/>answer or 'Not mentioned'<br/>random order"]

    PROMPT1 --> LLM[LLM 生成回答]
    PROMPT2 --> LLM
    PROMPT3 --> LLM
    PROMPT4 --> LLM
    PROMPT5 --> LLM
    LLM --> RETURN([返回 prediction, user_prompt, raw_context])

    style C5 fill:#ff9999,stroke:#cc0000
    style C2 fill:#99ccff,stroke:#0066cc
    style C1 fill:#99ff99,stroke:#009900
    style C3 fill:#99ff99,stroke:#009900
    style C4 fill:#99ff99,stroke:#009900
```

### 类别详细说明

| 类别 | 描述 | Prompt 策略 | 温度 |
|------|------|-------------|------|
| 1 | 单跳事实回忆 | 短语回答，使用原文词汇 | 0.7 |
| 2 | 时间推理 | 使用对话日期回答近似日期，避免主语 | 0.7 |
| 3 | 多跳推理 | 短语回答，使用原文词汇 | 0.7 |
| 4 | 对话开集问答 | 短语回答，使用原文词汇 | 0.7 |
| 5 | 对抗性问题 | 二选一格式（正确答案 vs "Not mentioned"），随机排列 | `temperature_c5`（可配置） |

---

## 6. 评估指标计算流程

```mermaid
flowchart LR
    subgraph 输入
        P[prediction]
        R[reference]
    end

    P & R --> EM[exact_match<br/>精确匹配]
    P & R --> F1[token-level F1<br/>分词后集合交集]
    P & R --> ROUGE[ROUGE 指标<br/>rouge_scorer]
    P & R --> BLEU[BLEU 指标<br/>1-4 gram]
    P & R --> BERT[BERTScore<br/>bert_score]
    P & R --> METEOR[METEOR<br/>nltk.translate]
    P & R --> SBERT[SBERT 相似度<br/>all-MiniLM-L6-v2]

    EM --> COMBINE
    F1 --> COMBINE
    ROUGE --> COMBINE
    BLEU --> COMBINE
    BERT --> COMBINE
    METEOR --> COMBINE
    SBERT --> COMBINE

    COMBINE[calculate_metrics<br/>合并为 metrics dict] --> AGG[aggregate_metrics]

    AGG --> OVERALL[overall 统计<br/>mean / std / median / min / max]
    AGG --> CAT1[category_1 统计]
    AGG --> CAT2[category_2 统计]
    AGG --> CAT3[category_3 统计]
    AGG --> CAT4[category_4 统计]
    AGG --> CAT5[category_5 统计]
```

### 指标详情

| 指标 | 类型 | 说明 |
|------|------|------|
| `exact_match` | 精确匹配 | 预测与参考完全一致（忽略大小写） |
| `f1` | Token F1 | 分词后计算 Precision / Recall / F1 |
| `rouge1_f` | ROUGE-1 | 单字重叠 F 值 |
| `rouge2_f` | ROUGE-2 | 二元组重叠 F 值 |
| `rougeL_f` | ROUGE-L | 最长公共子序列 F 值 |
| `bleu1` ~ `bleu4` | BLEU | 1~4 gram BLEU 分数（SmoothingFunction） |
| `bert_f1` | BERTScore | 基于 BERT 的语义相似度 F1 |
| `meteor` | METEOR | 考虑同义词的翻译评估指标 |
| `sbert_similarity` | SBERT | SentenceTransformer 余弦相似度 |

### 聚合统计量

每个指标在每个分组（overall + 各 category）下计算以下统计量：

- **mean**: 均值
- **std**: 标准差（样本数 ≤ 1 时为 0）
- **median**: 中位数
- **min / max**: 最小 / 最大值
- **count**: 样本数

---

## 7. 数据集加载与解析流程

```mermaid
flowchart TD
    JSON[LoComo JSON 文件] --> PARSE_TOP[遍历每个 sample]

    PARSE_TOP --> PARSE_QA[解析 QA 列表]
    PARSE_TOP --> PARSE_CONV[解析 Conversation]
    PARSE_TOP --> PARSE_EVENT[解析 EventSummary]
    PARSE_TOP --> PARSE_OBS[解析 Observation]
    PARSE_TOP --> PARSE_SS[解析 SessionSummary]

    PARSE_QA --> QA_OBJ[QA 对象]
    PARSE_CONV --> CONV_OBJ[Conversation 对象]

    subgraph QA 数据结构
        direction TB
        QA_OBJ --> Q1[question: str]
        QA_OBJ --> Q2[answer: Optional str]
        QA_OBJ --> Q3[evidence: List str]
        QA_OBJ --> Q4[category: Optional int]
        QA_OBJ --> Q5[adversarial_answer: Optional str]
        QA_OBJ --> Q6["final_answer: category==5 ? adversarial_answer : answer"]
    end

    subgraph Conversation 解析
        direction TB
        CONV_OBJ --> C1[speaker_a / speaker_b]
        CONV_OBJ --> C2["sessions: Dict int → Session"]

        C2 --> S1[session_id: int]
        C2 --> S2[date_time: str]
        C2 --> S3["turns: List Turn"]

        S3 --> T1[speaker: str]
        S3 --> T2[dia_id: str]
        S3 --> T3[text: str]
    end

    PARSE_CONV --> CONV_OBJ

    QA_OBJ & CONV_OBJ & PARSE_EVENT & PARSE_OBS & PARSE_SS --> SAMPLE[LoCoMoSample]
    SAMPLE --> SAMPLES[List LoCoMoSample]
```

### 数据类关系

```mermaid
classDiagram
    class LoCoMoSample {
        +str sample_id
        +List~QA~ qa
        +Conversation conversation
        +EventSummary event_summary
        +Observation observation
        +Dict session_summary
    }

    class QA {
        +str question
        +Optional~str~ answer
        +List~str~ evidence
        +Optional~int~ category
        +Optional~str~ adversarial_answer
        +final_answer() Optional~str~
    }

    class Conversation {
        +str speaker_a
        +str speaker_b
        +Dict~int, Session~ sessions
    }

    class Session {
        +int session_id
        +str date_time
        +List~Turn~ turns
    }

    class Turn {
        +str speaker
        +str dia_id
        +str text
    }

    class EventSummary {
        +Dict events
    }

    class Observation {
        +Dict observations
    }

    LoCoMoSample --> QA
    LoCoMoSample --> Conversation
    LoCoMoSample --> EventSummary
    LoCoMoSample --> Observation
    Conversation --> Session
    Session --> Turn

    note for QA "final_answer: category==5 时返回 adversarial_answer，否则返回 answer"
```

---

## 8. 标准版 vs 鲁棒版评估对比

```mermaid
flowchart TB
    subgraph 标准版 advancedMemAgent
        direction TB
        A1[AgenticMemorySystem] --> A2[LLMController]
        A2 --> A3["get_completion(prompt,<br/>response_format=json_schema)"]
        A3 --> A4["JSON Schema 严格解析<br/>{relevant_parts: ...}<br/>{keywords: ...}<br/>{answer: ...}"]
    end

    subgraph 鲁棒版 RobustAdvancedMemAgent
        direction TB
        B1[RobustAgenticMemorySystem] --> B2[RobustLLMController]
        B2 --> B3["get_completion(prompt)<br/>无 response_format"]
        B3 --> B4["纯文本响应解析<br/>parse_plain_text_answer<br/>parse_relevant_parts<br/>parse_keywords_response"]
        B4 --> B5["JSON 优先 + 文本回退<br/>strip_markdown_fences<br/>→ json.loads → section 解析"]
    end
```

### 关键差异对比

| 维度 | 标准版 (`advancedMemAgent`) | 鲁棒版 (`RobustAdvancedMemAgent`) |
|------|---------------------------|----------------------------------|
| 记忆系统 | `AgenticMemorySystem` | `RobustAgenticMemorySystem` |
| LLM 控制器 | `LLMController` | `RobustLLMController` |
| LLM 调用方式 | `get_completion(prompt, response_format=json_schema)` | `get_completion(prompt)` 无 JSON Schema |
| 响应解析 | 直接 `json.loads` 取字段 | `parse_plain_text_answer` 等：JSON 优先 + 纯文本回退 |
| 查询关键词生成 | JSON Schema 强制 `{"keywords": "..."}` | 逗号分隔关键词 + `parse_keywords_response` |
| 记忆筛选 | JSON Schema 强制 `{"relevant_parts": "..."}` | 纯文本 prompt + `parse_relevant_parts` |
| 答案生成 | JSON Schema 强制 `{"answer": "..."}` | 纯文本 + `parse_plain_text_answer` |
| 缓存目录 | `cached_memories_advanced_{backend}_{model}/` | `cached_memories_robust_{backend}_{model}/` |
| 日志前缀 | `eval_ours_` | `eval_robust_` |
| 后端兼容性 | 仅支持 JSON Schema 的后端（OpenAI 等） | 任意后端（OpenAI / Ollama / SGLang / vLLM） |
| 错误处理 | JSON 解析失败时记录错误数 | 异常捕获返回空字符串 |

### 鲁棒版解析策略

```mermaid
flowchart TD
    RESP[LLM 响应] --> FENCE[strip_markdown_fences<br/>去除 ```json ... ```]
    FENCE --> TRY_JSON{尝试 json.loads?}
    TRY_JSON -->|成功| CHECK_KEY{包含目标字段?<br/>answer / keywords / relevant_parts}
    CHECK_KEY -->|是| EXTRACT[提取目标字段值]
    CHECK_KEY -->|否| RAW[返回原始文本]
    TRY_JSON -->|失败| SECTION[Section-marker 解析<br/>KEYWORDS: / CONTEXT: / TAGS:]
    SECTION --> RAW2[返回解析结果或原始文本]
    EXTRACT --> DONE([返回解析结果])
    RAW --> DONE
    RAW2 --> DONE
```

---

## 9. 评估流程参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `--dataset` | `data/locomo10.json` | 数据集文件路径 |
| `--model` | `gpt-4o-mini` | LLM 模型名称 |
| `--output` | None | 结果输出路径 |
| `--ratio` | 1.0 | 评估样本比例（0.0~1.0） |
| `--backend` | `openai` / `sglang` | LLM 后端 |
| `--temperature_c5` | 0.5 | 类别 5 问题的温度参数 |
| `--retrieve_k` | 10 | 检索记忆数量 |
| `--sglang_host` | `http://localhost` | SGLang 服务器地址 |
| `--sglang_port` | 30000 | SGLang 服务器端口 |

---

## 10. 输出结构

```mermaid
classDiagram
    class FinalResults {
        +str model
        +str dataset
        +str memory_layer
        +int total_questions
        +Dict category_distribution
        +Dict aggregate_metrics
        +List individual_results
    }

    class AggregateMetrics {
        +Dict overall
        +Dict category_1
        +Dict category_2
        +Dict category_3
        +Dict category_4
        +Dict category_5
    }

    class MetricStats {
        +float mean
        +float std
        +float median
        +float min
        +float max
        +int count
    }

    class IndividualResult {
        +int sample_id
        +str question
        +str prediction
        +str reference
        +int category
        +Dict metrics
    }

    FinalResults --> AggregateMetrics
    AggregateMetrics --> MetricStats
    FinalResults --> IndividualResult
```
