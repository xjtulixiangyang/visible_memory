# LLM 控制器模块设计文档

> 源文件：`memory_layer.py`（标准版）、`memory_layer_robust.py`（鲁棒版）、`llm_text_parsers.py`（鲁棒版解析器）

## 1. 模块架构总览

LLM 控制器模块是 AgenticMemory 系统的 LLM 调用抽象层，负责屏蔽不同 LLM 后端的通信差异，为上层记忆系统提供统一的调用接口。模块提供两套并行实现：

- **标准版**（`memory_layer.py`）：依赖 `response_format` / JSON Schema 强制结构化输出
- **鲁棒版**（`memory_layer_robust.py`）：纯文本 prompt + section-marker 解析，兼容性更强

```mermaid
graph TB
    subgraph 上层调用
        MS[AgenticMemorySystem]
        RMS[RobustAgenticMemorySystem]
    end

    subgraph LLM控制器层
        LC[LLMController 工厂]
        RLC[RobustLLMController 工厂]
    end

    subgraph 标准版控制器
        OAI[OpenAIController]
        OLL[OllamaController]
        SGL[SGLangController]
        LIT[LiteLLMController]
    end

    subgraph 鲁棒版控制器
        ROAI[RobustOpenAIController]
        ROLL[RobustOllamaController]
        RSGL[RobustSGLangController]
        RVLLM[RobustVLLMController]
        RLIT[RobustLiteLLMController]
    end

    subgraph 辅助模块
        PARSER[llm_text_parsers<br/>提示模板 + 解析器]
        RETRY[retry_llm_call<br/>重试装饰器]
    end

    MS --> LC
    RMS --> RLC
    LC --> OAI & OLL & SGL & LIT
    RLC --> ROAI & ROLL & RSGL & RVLLM & RLIT
    ROAI & ROLL & RSGL & RVLLM & RLIT -.-> RETRY
    RMS --> PARSER
```

## 2. 类继承关系

### 2.1 标准版类图

```mermaid
classDiagram
    class BaseLLMController {
        <<abstract>>
        +get_completion(prompt: str)* str
    }

    class OpenAIController {
        -model: str
        -client: OpenAI
        +get_completion(prompt, response_format, temperature) str
    }

    class OllamaController {
        -model: str
        +get_completion(prompt, response_format, temperature) str
        -_generate_empty_value(schema_type, schema_items) Any
        -_generate_empty_response(response_format) dict
    }

    class SGLangController {
        -model: str
        -sglang_host: str
        -sglang_port: int
        -base_url: str
        +get_completion(prompt, response_format, temperature) str
        -_generate_empty_value(schema_type, schema_items) Any
        -_generate_empty_response(response_format) dict
    }

    class LiteLLMController {
        -model: str
        -api_base: str
        -api_key: str
        +get_completion(prompt, response_format, temperature) str
        -_generate_empty_value(schema_type, schema_items) Any
        -_generate_empty_response(response_format) dict
    }

    class LLMController {
        -llm: BaseLLMController
        +__init__(backend, model, api_key, api_base, sglang_host, sglang_port)
    }

    BaseLLMController <|-- OpenAIController
    BaseLLMController <|-- OllamaController
    BaseLLMController <|-- SGLangController
    BaseLLMController <|-- LiteLLMController
    LLMController *-- BaseLLMController : llm
```

### 2.2 鲁棒版类图

```mermaid
classDiagram
    class RobustBaseLLMController {
        <<abstract>>
        +SYSTEM_MESSAGE: str
        +get_completion(prompt: str, temperature: float)* str
        +check_connectivity() void
    }

    class RobustOpenAIController {
        -model: str
        -client: OpenAI
        +get_completion(prompt, temperature) str
    }

    class RobustOllamaController {
        -model: str
        +get_completion(prompt, temperature) str
    }

    class RobustSGLangController {
        -model: str
        -base_url: str
        -_requests: module
        +get_completion(prompt, temperature) str
    }

    class RobustVLLMController {
        -model: str
        -base_url: str
        -_requests: module
        +get_completion(prompt, temperature) str
    }

    class RobustLiteLLMController {
        -model: str
        -api_base: str
        -api_key: str
        -_completion: function
        +get_completion(prompt, temperature) str
    }

    class RobustLLMController {
        -llm: RobustBaseLLMController
        +__init__(backend, model, api_key, api_base, sglang_host, sglang_port, check_connection)
    }

    RobustBaseLLMController <|-- RobustOpenAIController
    RobustBaseLLMController <|-- RobustOllamaController
    RobustBaseLLMController <|-- RobustSGLangController
    RobustBaseLLMController <|-- RobustVLLMController
    RobustBaseLLMController <|-- RobustLiteLLMController
    RobustLLMController *-- RobustBaseLLMController : llm
```

## 3. 工厂模式流程

### 3.1 标准版工厂 `LLMController`

```mermaid
flowchart TD
    A["LLMController(backend, model, ...)"] --> B{backend 类型?}

    B -->|openai| C["OpenAIController(model, api_key)"]
    B -->|ollama| D["LiteLLMController(model='ollama/{model}',<br/>api_base='http://localhost:11434',<br/>api_key='EMPTY')"]
    B -->|sglang| E["SGLangController(model,<br/>sglang_host, sglang_port)"]
    B -->|其他| F["raise ValueError"]

    C --> G["self.llm = 实例"]
    D --> G
    E --> G
```

> **注意**：标准版 `backend="ollama"` 实际创建的是 `LiteLLMController`（使用 `ollama/` 前缀通过 LiteLLM 代理），而非 `OllamaController`（使用 `ollama_chat/` 前缀）。

### 3.2 鲁棒版工厂 `RobustLLMController`

```mermaid
flowchart TD
    A["RobustLLMController(backend, model, ...,<br/>check_connection)"] --> B{backend 类型?}

    B -->|openai| C["RobustOpenAIController(model, api_key)"]
    B -->|ollama| D["RobustOllamaController(model)"]
    B -->|sglang| E["RobustSGLangController(model,<br/>sglang_host, sglang_port)"]
    B -->|vllm| F["RobustVLLMController(model,<br/>sglang_host, sglang_port)"]
    B -->|其他| G["raise ValueError"]

    C --> H["self.llm = 实例"]
    D --> H
    E --> H
    F --> H

    H --> I{check_connection?}
    I -->|True| J["self.llm.check_connectivity()"]
    I -->|False| K["初始化完成"]
    J --> K
```

## 4. 重试机制流程

鲁棒版通过 `retry_llm_call` 装饰器为所有控制器的 `get_completion` 方法提供指数退避重试：

```mermaid
flowchart TD
    A["调用 get_completion()"] --> B["执行 LLM 请求"]
    B --> C{请求成功?}
    C -->|是| D["返回结果"]
    C -->|否| E["捕获异常"]

    E --> F{attempt < max_retries?}
    F -->|是| G["计算退避延迟<br/>delay = base_delay × 2^attempt"]
    G --> H["logger.warning 记录重试"]
    H --> I["time.sleep(delay)"]
    I --> J["attempt += 1"]
    J --> B

    F -->|否| K["logger.error 记录最终失败"]
    K --> L["raise last_exc"]
```

**参数说明**：

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `max_retries` | 2 | 最大重试次数（不含首次调用，即总共最多 3 次调用） |
| `base_delay` | 1.0s | 基础退避延迟（秒） |

**退避时间表**（默认参数）：

| 尝试序号 | 延迟 |
|----------|------|
| 第 1 次失败后 | 1.0s |
| 第 2 次失败后 | 2.0s |
| 第 3 次失败后 | 抛出异常 |

## 5. 连接检查流程

鲁棒版 `RobustBaseLLMController.check_connectivity()` 在工厂创建控制器后可选执行：

```mermaid
sequenceDiagram
    participant Factory as RobustLLMController
    participant Controller as RobustXxxController
    participant Backend as LLM 后端

    Factory->>Controller: 创建实例
    Factory->>Controller: check_connectivity()

    Controller->>Backend: get_completion("Reply with exactly one word: READY", temperature=0.0)
    
    alt 后端可达
        Backend-->>Controller: 响应文本
        Controller->>Controller: 检查响应非空
        Controller-->>Factory: 检查通过 (logger.info)
    else 后端不可达 / 响应为空
        Backend-->>Controller: 异常或空响应
        Controller-->>Factory: raise ConnectionError("Cannot reach LLM backend: ...")
    end
```

**连接检查特点**：
- 使用 `temperature=0.0` 确保确定性响应
- 发送极简 prompt 降低 token 消耗
- 失败时抛出 `ConnectionError` 并附带原始异常信息
- 仅在 `check_connection=True` 时触发，默认关闭

## 6. 标准版 vs 鲁棒版调用时序对比

### 6.1 标准版调用时序（单次 LLM 调用）

标准版在内容分析和记忆进化中均依赖 `response_format` 参数强制 JSON Schema 输出：

```mermaid
sequenceDiagram
    participant Caller as 上层调用者
    participant LC as LLMController
    participant Ctrl as XxxController
    participant Backend as LLM 后端

    Caller->>LC: lc.llm.get_completion(prompt, response_format, temperature)
    LC->>Ctrl: get_completion(prompt, response_format, temperature)
    
    Note over Ctrl: 构造请求<br/>system: "You must respond with a JSON object."<br/>user: prompt<br/>response_format: JSON Schema

    Ctrl->>Backend: API 调用（含 response_format）
    
    alt 成功
        Backend-->>Ctrl: JSON 字符串
        Ctrl-->>LC: 返回 JSON 字符串
        LC-->>Caller: JSON 字符串
    else 失败（Ollama/SGLang/LiteLLM）
        Backend-->>Ctrl: 异常
        Ctrl->>Ctrl: _generate_empty_response(response_format)
        Note over Ctrl: 根据 Schema 生成空默认值
        Ctrl-->>LC: 返回空 JSON 字符串
        LC-->>Caller: 空 JSON 字符串
    end
```

### 6.2 鲁棒版调用时序（内容分析）

鲁棒版使用纯文本 prompt + section-marker 解析，并支持 JSON 回退：

```mermaid
sequenceDiagram
    participant Caller as RobustMemoryNote
    participant LC as RobustLLMController
    participant Ctrl as RobustXxxController
    participant Backend as LLM 后端
    participant Parser as llm_text_parsers

    Caller->>LC: lc.llm.get_completion(prompt)
    LC->>Ctrl: get_completion(prompt, temperature)
    
    Note over Ctrl: 构造请求<br/>system: "Follow the format specified..."<br/>user: prompt（含 section-marker 格式要求）<br/>无 response_format

    Ctrl->>Backend: API 调用（纯文本）
    
    alt 成功（含重试）
        Backend-->>Ctrl: 纯文本响应
        Ctrl-->>LC: 返回纯文本
        LC-->>Caller: 纯文本
    else 重试耗尽
        Backend-->>Ctrl: 异常
        Ctrl-->>LC: raise 异常
        LC-->>Caller: 异常向上传播
    end

    Caller->>Parser: parse_analyze_content(response)
    Parser->>Parser: parse_with_json_fallback()
    
    alt 尝试 JSON 解析
        Parser->>Parser: strip_markdown_fences → json.loads
        alt JSON 有效
            Parser-->>Caller: 返回 dict
        else JSON 无效
            Parser->>Parser: 回退到 section-marker 解析
        end
    end
    
    alt section-marker 解析
        Parser->>Parser: _extract_section("KEYWORDS") / _extract_section("CONTEXT") / _extract_section("TAGS")
        Parser-->>Caller: 返回 dict
    end

    Caller->>Parser: validate_analysis_result(result)
    Note over Parser: 空 keywords → 启发式提取<br/>空 context → 取首句<br/>空 tags → 从 keywords 派生
```

### 6.3 鲁棒版调用时序（记忆进化 — 三步条件调用）

鲁棒版将标准版的单次进化调用拆分为最多 3 次条件性调用：

```mermaid
sequenceDiagram
    participant Caller as RobustAgenticMemorySystem
    participant LC as RobustLLMController
    participant Parser as llm_text_parsers

    rect rgb(230, 245, 255)
        Note over Caller: 调用 1：进化决策（必须）
        Caller->>LC: get_completion(EVOLUTION_DECISION_PROMPT)
        LC-->>Caller: decision_response
        Caller->>Parser: parse_evolution_decision(response)
        Parser-->>Caller: {"decision": "...", "reason": "..."}
    end

    alt decision == "NO_EVOLUTION"
        Caller-->>Caller: return (False, note)
    end

    alt decision 包含 "STRENGTHEN"
        rect rgb(255, 245, 230)
            Note over Caller: 调用 2：强化详情（条件）
            Caller->>LC: get_completion(STRENGTHEN_DETAILS_PROMPT)
            LC-->>Caller: strengthen_response
            Caller->>Parser: parse_strengthen_details(response)
            Parser-->>Caller: {"connections": [...], "tags": [...]}
            Caller->>Caller: note.links.extend(connections)<br/>note.tags = tags
        end
    end

    alt decision 包含 "UPDATE"
        rect rgb(230, 255, 230)
            Note over Caller: 调用 3：邻居更新（条件）
            Caller->>LC: get_completion(UPDATE_NEIGHBORS_PROMPT)
            LC-->>Caller: update_response
            Caller->>Parser: parse_update_neighbors(response, num_neighbors)
            Parser-->>Caller: [{"context": "...", "tags": [...]}, ...]
            Caller->>Caller: 更新邻居记忆的 context 和 tags
        end
    end

    Caller-->>Caller: return (True, note)
```

## 7. 关键差异对比

```mermaid
graph LR
    subgraph 标准版
        direction TB
        S1["response_format / JSON Schema<br/>强制结构化输出"]
        S2["单次 LLM 调用完成进化决策"]
        S3["失败时返回空 JSON 默认值"]
        S4["无重试机制"]
        S5["无连接检查"]
        S6["print() 调试输出"]
    end

    subgraph 鲁棒版
        direction TB
        R1["纯文本 prompt +<br/>section-marker 解析"]
        R2["最多 3 次条件性 LLM 调用<br/>（决策 → 强化 → 更新）"]
        R3["失败时启发式降级<br/>+ 优雅存储"]
        R4["retry_llm_call 指数退避重试"]
        R5["check_connectivity() 连接检查"]
        R6["logging 结构化日志"]
    end

    S1 -.->|对比| R1
    S2 -.->|对比| R2
    S3 -.->|对比| R3
    S4 -.->|对比| R4
    S5 -.->|对比| R5
    S6 -.->|对比| R6
```

### 详细对比表

| 维度 | 标准版 | 鲁棒版 |
|------|--------|--------|
| **结构化输出方式** | `response_format` JSON Schema | 纯文本 prompt + section-marker 解析 |
| **`get_completion` 签名** | `(prompt, response_format, temperature)` | `(prompt, temperature)` |
| **System Message** | `"You must respond with a JSON object."` | `"Follow the format specified in the prompt exactly. Do not add extra commentary."` |
| **支持后端** | openai, ollama, sglang | openai, ollama, sglang, vllm |
| **Ollama 实现** | LiteLLM 代理 (`ollama/` 前缀) | 直接 Ollama chat API |
| **进化调用次数** | 1 次（单 prompt 含全部逻辑） | 最多 3 次（决策 → 强化 → 更新） |
| **失败处理** | 返回空 JSON 默认值 | 启发式降级 + 优雅存储（不丢失记忆） |
| **重试机制** | 无 | `retry_llm_call` 指数退避（默认 2 次重试） |
| **连接检查** | 无 | `check_connectivity()` 可选 |
| **日志** | `print()` | `logging` 结构化日志 |
| **解析兼容** | 仅 JSON | JSON 优先 + section-marker 回退 |

## 8. 各后端通信协议

```mermaid
graph TB
    subgraph OpenAI
        OAI_REQ["OpenAI SDK<br/>client.chat.completions.create()"]
    end

    subgraph Ollama
        OLL_STD["标准版: LiteLLM 代理<br/>model='ollama_chat/{model}'"]
        OLL_ROB["鲁棒版: Ollama SDK<br/>ollama.chat()"]
    end

    subgraph SGLang
        SGL_REQ["HTTP POST<br/>{host}:{port}/generate"]
        SGL_STD["标准版: 含 json_schema 参数"]
        SGL_ROB["鲁棒版: 无 json_schema 参数"]
    end

    subgraph vLLM
        VLLM_REQ["HTTP POST<br/>{host}:{port}/v1/chat/completions<br/>OpenAI 兼容 API"]
    end

    subgraph LiteLLM
        LIT_REQ["litellm.completion()"]
    end
```

### 各后端请求参数对比

| 后端 | 标准版关键参数 | 鲁棒版关键参数 |
|------|---------------|---------------|
| **OpenAI** | `model`, `messages`, `response_format`, `temperature`, `max_tokens=1000` | `model`, `messages`, `temperature`, `max_tokens=1000` |
| **Ollama** | LiteLLM: `model="ollama_chat/{model}"`, `response_format` | Ollama SDK: `model`, `messages`, `options={"temperature": ...}` |
| **SGLang** | `text`, `sampling_params={temperature, max_new_tokens, json_schema}` | `text`, `sampling_params={temperature, max_new_tokens}` |
| **vLLM** | — | `model`, `messages`, `temperature`, `max_tokens=1000` |
| **LiteLLM** | `model`, `messages`, `response_format`, `temperature`, `api_base`, `api_key` | `model`, `messages`, `temperature`, `api_base`, `api_key` |

## 9. 优雅降级策略

鲁棒版在 LLM 调用失败时采用多层降级策略，确保记忆不丢失：

```mermaid
flowchart TD
    A["LLM 调用失败"] --> B{调用场景?}

    B -->|内容分析| C["启发式降级"]
    C --> C1["_heuristic_keywords(content)<br/>提取大写词/非停用词"]
    C --> C2["_heuristic_context(content)<br/>取首句或前 200 字符"]
    C --> C3["tags 从 keywords 派生"]
    C1 & C2 & C3 --> D["返回降级分析结果<br/>记忆正常存储"]

    B -->|进化决策| E["捕获异常"]
    E --> F["logger.error 记录"]
    F --> G["return (False, note)<br/>记忆不进化但正常存储"]

    B -->|关键词提取为空| H["FOCUSED_KEYWORDS_PROMPT 重试"]
    H --> I{重试成功?}
    I -->|是| J["使用重试结果"]
    I -->|否| K["validate_analysis_result<br/>启发式填充"]
```
