# Mem0 LLM 子模块深度解析

## 1. 模块概述

### 1.1 职责

`mem0/llms/` 子模块是 Mem0 的 **LLM 抽象层**，负责将不同大语言模型提供商的 API 差异封装在统一的接口背后。上层业务（如 `Memory.add()`、`Memory.search()`）只需调用 `generate_response()` 方法，无需关心底层是 OpenAI、Anthropic 还是 Ollama。

### 1.2 支持的 Provider（18 个）

| # | Provider | 类名 | 默认模型 | 客户端 SDK |
|---|----------|------|----------|-----------|
| 1 | OpenAI | `OpenAILLM` | `gpt-5-mini` | `openai.OpenAI` |
| 2 | Anthropic | `AnthropicLLM` | `claude-3-5-sonnet-20240620` | `anthropic.Anthropic` |
| 3 | Ollama | `OllamaLLM` | `llama3.1:70b` | `ollama.Client` |
| 4 | Gemini | `GeminiLLM` | `gemini-2.0-flash` | `google.genai.Client` |
| 5 | DeepSeek | `DeepSeekLLM` | `deepseek-chat` | `openai.OpenAI` (兼容) |
| 6 | Groq | `GroqLLM` | `llama-3.3-70b-versatile` | `groq.Groq` |
| 7 | Together | `TogetherLLM` | `mistralai/Mixtral-8x7B-Instruct-v0.1` | `together.Together` |
| 8 | LiteLLM | `LiteLLM` | `gpt-5-mini` | `litellm` |
| 9 | LangChain | `LangchainLLM` | (必须传入 `BaseChatModel`) | `langchain` |
| 10 | Azure OpenAI | `AzureOpenAILLM` | `gpt-5-mini` | `openai.AzureOpenAI` |
| 11 | AWS Bedrock | `AWSBedrockLLM` | `anthropic.claude-3-5-sonnet-20240620-v1:0` | `boto3` |
| 12 | vLLM | `VllmLLM` | `Qwen/Qwen2.5-32B-Instruct` | `openai.OpenAI` (兼容) |
| 13 | xAI | `XAILLM` | `grok-2-latest` | `openai.OpenAI` (兼容) |
| 14 | MiniMax | `MiniMaxLLM` | `MiniMax-M2.7` | `openai.OpenAI` (兼容) |
| 15 | LM Studio | `LMStudioLLM` | `Meta-Llama-3.1-70B-Instruct-IQ2_M.gguf` | `openai.OpenAI` (兼容) |
| 16 | Sarvam | `SarvamLLM` | `sarvam-m` | `requests` (原生 HTTP) |
| 17 | OpenAI Structured | `OpenAIStructuredLLM` | `gpt-5-mini` | `openai.OpenAI` (beta parse) |
| 18 | Azure Structured | `AzureOpenAIStructuredLLM` | `gpt-5-mini` | `openai.AzureOpenAI` |

---

## 2. 基类设计

```mermaid
classDiagram
    class LLMBase {
        <<abstract>>
        +config: BaseLlmConfig
        +__init__(config: Optional~Union~BaseLlmConfig, Dict~)
        +_validate_config()
        +_is_reasoning_model(model: str) bool
        +_get_supported_params(**kwargs) Dict
        +_get_common_params(**kwargs) Dict
        +generate_response(messages, tools, tool_choice, **kwargs)*
    }

    class OpenAILLM {
        +client: OpenAI
        +generate_response(messages, response_format, tools, tool_choice, **kwargs)
        -_parse_response(response, tools)
    }

    class AnthropicLLM {
        +client: Anthropic
        #_get_common_params(**kwargs) Dict
        +generate_response(messages, response_format, tools, tool_choice, **kwargs)
    }

    class OllamaLLM {
        +client: Client
        +generate_response(messages, response_format, tools, tool_choice, **kwargs)
    }

    class GeminiLLM {
        +client: genai.Client
        -_reformat_messages(messages)
        -_reformat_tools(tools)
        +generate_response(messages, response_format, tools, tool_choice)
    }

    class AWSBedrockLLM {
        +client: boto3.client
        +provider: str
        +supports_tools: bool
        -_format_messages_anthropic(messages)
        -_format_messages_cohere(messages)
        -_format_messages_amazon(messages)
        +generate_response(messages, response_format, tools, tool_choice, stream, **kwargs)
    }

    class LangchainLLM {
        +langchain_model: BaseChatModel
        +generate_response(messages, response_format, tools, tool_choice)
    }

    LLMBase <|-- OpenAILLM
    LLMBase <|-- AnthropicLLM
    LLMBase <|-- OllamaLLM
    LLMBase <|-- GeminiLLM
    LLMBase <|-- AWSBedrockLLM
    LLMBase <|-- LangchainLLM
```

**公共能力：**

| 方法 | 职责 |
|------|------|
| `_validate_config()` | 配置校验，检查 `model` 是否存在 |
| `_is_reasoning_model(model)` | 推理模型检测（o1/o3/gpt-5 系列），支持 `is_reasoning_model` 配置项显式覆盖 |
| `_get_supported_params(**kwargs)` | 推理模型仅保留 messages/response_format/tools/tool_choice/reasoning_effort |
| `_get_common_params(**kwargs)` | 构建通用参数字典（temperature/max_tokens/top_p），子类可覆写 |

---

## 3. 配置体系

### 3.1 三层配置架构

```mermaid
classDiagram
    class LlmConfig {
        +provider: str = "openai"
        +config: Optional~dict~ = {}
    }

    class BaseLlmConfig {
        <<abstract>>
        +model: Optional~Union~str, Dict~
        +temperature: float = 0.1
        +api_key: Optional~str~
        +max_tokens: int = 2000
        +top_p: float = 0.1
        +top_k: int = 1
        +enable_vision: bool = False
        +vision_details: str = "auto"
        +reasoning_effort: Optional~str~
        +is_reasoning_model: Optional~bool~
    }

    class OpenAIConfig {
        +openai_base_url: Optional~str~
        +models: Optional~List~str~~
        +route: str = "fallback"
        +openrouter_base_url: Optional~str~
        +store: Optional~bool~
        +response_callback: Optional~Callable~
    }

    class AnthropicConfig {
        +anthropic_base_url: Optional~str~
    }

    class OllamaConfig {
        +ollama_base_url: Optional~str~
    }

    class DeepSeekConfig {
        +deepseek_base_url: Optional~str~
    }

    class AzureOpenAIConfig {
        +azure_kwargs: AzureConfig
    }

    class AWSBedrockConfig {
        +aws_access_key_id: Optional~str~
        +aws_secret_access_key: Optional~str~
        +aws_region: str
        +provider: str
    }

    LlmConfig --> BaseLlmConfig : config dict 传入
    BaseLlmConfig <|-- OpenAIConfig
    BaseLlmConfig <|-- AnthropicConfig
    BaseLlmConfig <|-- OllamaConfig
    BaseLlmConfig <|-- DeepSeekConfig
    BaseLlmConfig <|-- AzureOpenAIConfig
    BaseLlmConfig <|-- AWSBedrockConfig
```

---

## 4. Provider 实现差异分析

### 4.1 核心能力对比

| Provider | Tools 支持 | response_format | 推理模型感知 | 专属 Config |
|----------|-----------|----------------|-------------|------------|
| OpenAI | Yes | Yes | Yes | `OpenAIConfig` |
| Anthropic | Partial | No | No | `AnthropicConfig` |
| Ollama | Yes | JSON only | No | `OllamaConfig` |
| Gemini | Yes | JSON+Schema | No | `BaseLlmConfig` |
| DeepSeek | Yes | Yes | Yes | `DeepSeekConfig` |
| Groq | Yes | Yes | No | `BaseLlmConfig` |
| Together | Yes | Yes | No | `BaseLlmConfig` |
| LiteLLM | Yes | Yes | No | `BaseLlmConfig` |
| LangChain | Yes | No | No | `BaseLlmConfig` |
| Azure OpenAI | Yes | Yes | Yes | `AzureOpenAIConfig` |
| AWS Bedrock | Partial | No | Partial | `AWSBedrockConfig` |
| vLLM | Yes | Yes | Yes | `VllmConfig` |
| xAI | No | Yes | No | `BaseLlmConfig` |
| MiniMax | Yes | Yes | Yes | `MinimaxConfig` |
| LM Studio | Yes | Yes (默认JSON) | Yes | `LMStudioConfig` |
| Sarvam | No | No | No | `BaseLlmConfig` |

### 4.2 消息格式差异

| Provider | system 消息处理 | 特殊处理 |
|----------|---------------|---------|
| OpenAI | 原样传入 | - |
| Anthropic | 提取为独立 `system` 参数 | - |
| Ollama | 原样传入 | JSON 提示追加 |
| Gemini | 提取为 `system_instruction` | 角色映射为 `types.Content` |
| LangChain | 转为 `("system", content)` 元组 | role 映射 |
| Azure OpenAI | 原样传入 | "assistant" -> "ai" 替换 |
| AWS Bedrock | Provider 依赖 | 6 种格式化方法 |

### 4.3 OpenAI 兼容层 Provider

```mermaid
flowchart TD
    SDK["openai.OpenAI SDK"] --> P1[OpenAI]
    SDK --> P2[DeepSeek]
    SDK --> P3[vLLM]
    SDK --> P4[xAI]
    SDK --> P5[MiniMax]
    SDK --> P6[LM Studio]

    AzureSDK["openai.AzureOpenAI SDK"] --> P7[Azure OpenAI]
    AzureSDK --> P8[Azure OpenAI Structured]

    style SDK fill:#e1f5fe
    style AzureSDK fill:#e1f5fe
```

---

## 5. 推理模型处理

```mermaid
flowchart TD
    A[_is_reasoning_model 调用] --> B{config.is_reasoning_model<br/>显式配置?}
    B -->|非 None| C[返回显式配置值]
    B -->|None| D{模型名称匹配}
    D --> E{精确匹配 o1/o3/gpt-5 系列?}
    E -->|是| F[返回 True]
    E -->|否| G{前缀匹配 o1-/o3-?}
    G -->|是| F
    G -->|否| H[返回 False]
```

推理模型参数过滤：仅保留 `messages`、`response_format`、`tools`、`tool_choice`、`reasoning_effort`，过滤掉 `temperature`、`max_tokens`、`top_p`。

---

## 6. Provider 注册与发现

```mermaid
flowchart TD
    A[LlmFactory.create] --> B{provider_name 在注册表?}
    B -->|否| C[抛出 ValueError]
    B -->|是| D[获取 class_type 和 config_class]
    D --> E[load_class 动态导入]
    E --> F{config 类型?}
    F -->|None| G[config_class 创建默认配置]
    F -->|dict| H[合并 kwargs 后创建配置]
    F -->|BaseLlmConfig| I{需要转换?}
    I -->|是| J[提取公共字段转 ProviderConfig]
    I -->|否| K[直接使用]
    G --> M[llm_class 实例化]
    H --> M
    J --> M
    K --> M
```

---

## 7. LLM 调用时序图

```mermaid
sequenceDiagram
    participant User as 用户代码
    participant Memory as Memory.add()
    participant Factory as LlmFactory
    participant LLM as Provider LLM
    participant API as Provider API

    User->>Memory: add(messages, user_id=...)
    Memory->>Factory: create("openai", config)
    Factory->>Factory: 查找 provider_to_class
    Factory->>Factory: load_class 动态导入
    Factory->>LLM: OpenAILLM(config)
    LLM->>LLM: _validate_config()
    LLM->>LLM: 初始化 OpenAI client
    LLM-->>Memory: 返回 LLM 实例

    Memory->>Memory: 构建 prompt messages
    Memory->>LLM: generate_response(messages, response_format=json)

    LLM->>LLM: _get_supported_params()
    alt 推理模型
        LLM->>LLM: 过滤 temperature/max_tokens/top_p
    else 普通模型
        LLM->>LLM: _get_common_params()
    end

    LLM->>API: client.chat.completions.create(**params)
    API-->>LLM: response
    LLM->>LLM: _parse_response(response, tools)
    LLM-->>Memory: str 或 dict

    Memory->>Memory: 解析结果，提取记忆
    Memory-->>User: 返回操作结果
```

---

## 8. API Key 环境变量映射

| Provider | 环境变量 | Base URL 环境变量 |
|----------|---------|-----------------|
| OpenAI | `OPENAI_API_KEY` | `OPENAI_BASE_URL` |
| OpenRouter | `OPENROUTER_API_KEY` | `OPENROUTER_API_BASE` |
| Anthropic | `ANTHROPIC_API_KEY` | - |
| Gemini | `GOOGLE_API_KEY` | - |
| DeepSeek | `DEEPSEEK_API_KEY` | `DEEPSEEK_API_BASE` |
| Groq | `GROQ_API_KEY` | - |
| Together | `TOGETHER_API_KEY` | - |
| Azure OpenAI | `LLM_AZURE_OPENAI_API_KEY` | `LLM_AZURE_ENDPOINT` |
| AWS Bedrock | `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` | `AWS_REGION` |
| vLLM | `VLLM_API_KEY` | `VLLM_BASE_URL` |
| xAI | `XAI_API_KEY` | `XAI_API_BASE` |
| MiniMax | `MINIMAX_API_KEY` | `MINIMAX_API_BASE` |
