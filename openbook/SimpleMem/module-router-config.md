# 核心模块设计: Router & Config (路由与配置)

> 源码路径: `simplemem/router.py`, `simplemem/config.py`, `simplemem/__init__.py`

## 1. 模块概述

Router & Config 是 SimpleMem 统一包的粘合层，提供：
1. **自动路由**: 根据首次调用方法自动选择后端引擎
2. **统一配置**: 跨所有后端的参数管理与覆盖
3. **包入口**: `from simplemem import SimpleMem` 的统一 API

## 2. AutoMemory 路由器

### 2.1 类设计

```mermaid
classDiagram
    class AutoMemory {
        -_mode: Optional~str~
        -_backend: Optional~object~
        -_config: Config
        +add_dialogue(speaker, content, timestamp)
        +add_dialogues(dialogues)
        +add_text(text, **kwargs) ProcessingResult
        +add_image(image, **kwargs) ProcessingResult
        +add_audio(audio, **kwargs) ProcessingResult
        +add_video(video_path, **kwargs) ProcessingResult
        +finalize()
        +ask(question) str
        +query(query, top_k, **kwargs) RetrievalResult
        +get_all_memories() List
        +close()
        -_ensure_text_backend()
        -_ensure_omni_backend()
        -_get_backend() object
    }

    class TextBackend {
        +add_dialogue()
        +add_dialogues()
        +finalize()
        +ask()
        +get_all_memories()
    }

    class OmniBackend {
        +add_text()
        +add_image()
        +add_audio()
        +add_video()
        +query()
    }

    AutoMemory --> TextBackend : _mode="text"
    AutoMemory --> OmniBackend : _mode="omni"
```

### 2.2 路由决策流程

```mermaid
flowchart TB
    Call["方法调用"] --> Check{"_mode 已设置?"}

    Check -->|"已设置: text"| Text["转发到 TextBackend"]
    Check -->|"已设置: omni"| Omni["转发到 OmniBackend"]
    Check -->|"未设置"| Classify{"方法分类"}

    Classify -->|"add_dialogue / add_dialogues"| SetText["_mode = 'text'<br/>初始化 TextBackend"]
    Classify -->|"finalize / ask"| SetText2["_mode = 'text'<br/>初始化 TextBackend"]
    Classify -->|"add_text / add_image /<br/>add_audio / add_video"| SetOmni["_mode = 'omni'<br/>初始化 OmniBackend"]
    Classify -->|"query"| SetOmni2["_mode = 'omni'<br/>初始化 OmniBackend"]

    SetText --> Text
    SetText2 --> Text
    SetOmni --> Omni
    SetOmni2 --> Omni

    Text --> Exec["执行后端方法"]
    Omni --> Exec
```

### 2.3 后端初始化时序

```mermaid
sequenceDiagram
    participant User as 用户
    participant AM as AutoMemory
    participant CFG as Config
    participant TB as TextBackend
    participant OB as OmniBackend

    User->>AM: SimpleMem(**kwargs)
    AM->>CFG: Config.from_kwargs(kwargs)
    CFG-->>AM: config

    Note over AM: _mode = None, _backend = None

    User->>AM: add_dialogue("Alice", "Hello")
    AM->>AM: _mode = "text"
    AM->>TB: TextBackend(config)
    TB-->>AM: _backend = TextBackend
    AM->>TB: add_dialogue("Alice", "Hello")

    Note over AM: 后续所有调用转发到 TextBackend

    User->>AM: finalize()
    AM->>TB: finalize()

    User->>AM: ask("What did Alice say?")
    AM->>TB: ask("What did Alice say?")
    TB-->>AM: "Alice said Hello"
    AM-->>User: "Alice said Hello"
```

## 3. 配置体系

### 3.1 Config 类

```mermaid
classDiagram
    class Config {
        +llm_api_key: Optional~str~
        +llm_model: str
        +llm_base_url: Optional~str~
        +embedding_model: str
        +embedding_dim: int
        +db_path: str
        +window_size: int
        +overlap_size: int
        +semantic_top_k: int
        +keyword_top_k: int
        +structured_top_k: int
        +context_budget: int
        +enable_planning: bool
        +enable_reflection: bool
        +max_reflection_rounds: int
        +enable_parallel_processing: bool
        +max_parallel_workers: int
        +use_json_format: bool
        +answer_style: str
        +omni_config: Dict
        +from_kwargs(kwargs) Config
        +to_dict() Dict
    }
```

### 3.2 配置优先级

```mermaid
flowchart LR
    subgraph "优先级 (高→低)"
        P1["构造参数<br/>(SimpleMem(k_sem=10))"]
        P2["环境变量<br/>(SIMPLEMEM_LLM_API_KEY)"]
        P3["配置文件<br/>(config.json)"]
        P4["默认值<br/>(settings.py)"]
    end

    P1 --> Override["最终配置"]
    P2 --> Override
    P3 --> Override
    P4 --> Override
```

### 3.3 配置流转

```mermaid
sequenceDiagram
    participant User as 用户
    participant AM as AutoMemory
    participant CFG as Config
    participant TB as TextBackend
    participant Settings as settings.py

    User->>AM: SimpleMem(k_sem=10, api_key="sk-...")
    AM->>Settings: 加载默认配置
    Settings-->>CFG: 默认值
    AM->>CFG: 覆盖用户参数
    CFG-->>AM: 最终配置

    AM->>TB: TextBackend(config)
    Note over TB: 从 config 读取所有参数

    User->>AM: optimize(dev_questions)
    AM->>AM: EvolveMem 优化
    AM->>CFG: 更新 evolved 配置
    CFG-->>AM: RetrievalConfig
```

## 4. 包入口设计

### 4.1 公开 API

```mermaid
flowchart TB
    subgraph "simplemem 包"
        INIT["__init__.py"]
        INIT --> SM["SimpleMem<br/>(= AutoMemory)"]
        INIT --> OPT["optimize()<br/>(= evolver.optimize)"]
        INIT --> CFG["Config<br/>(= config.Config)"]
    end

    subgraph "子模块"
        TEXT["simplemem.text<br/>(SimpleMemSystem)"]
        OMNI["simplemem.multimodal<br/>(OmniMemoryOrchestrator)"]
        EVOL["simplemem.evolver<br/>(EvolutionEngine)"]
    end

    SM -->|"add_dialogue"| TEXT
    SM -->|"add_text/image/audio/video"| OMNI
    OPT --> EVOL
```

### 4.2 导入映射

```python
# simplemem/__init__.py
from simplemem.router import AutoMemory as SimpleMem
from simplemem.config import Config
from simplemem.evolver import optimize

__all__ = ["SimpleMem", "Config", "optimize"]
```

## 5. 后端注册机制

### 5.1 注册表

```mermaid
flowchart LR
    subgraph "注册表 (_registry)"
        R1["'text' → simplemem.text.main:SimpleMemSystem"]
        R2["'omni' → simplemem.multimodal.orchestrator:OmniMemoryOrchestrator"]
    end

    subgraph "自定义后端"
        Custom["register('custom',<br/>'my_module', 'MyClass')"]
    end

    Custom --> R3["'custom' → my_module:MyClass"]

    R1 & R2 & R3 --> Router["AutoMemory 路由"]
```

### 5.2 延迟导入

```mermaid
sequenceDiagram
    participant AM as AutoMemory
    participant Reg as Registry
    participant Import as importlib

    AM->>AM: _ensure_text_backend()
    AM->>Reg: lookup('text')
    Reg-->>AM: 'simplemem.text.main:SimpleMemSystem'

    AM->>Import: import_module('simplemem.text.main')
    Import-->>AM: module
    AM->>AM: getattr(module, 'SimpleMemSystem')
    AM->>AM: SimpleMemSystem(config)
    AM->>AM: _backend = instance
```

## 6. 错误处理

### 6.1 路由错误

| 场景 | 处理 |
|:--|:--|
| 混合调用 (text + omni) | 抛出 ValueError，提示模式已锁定 |
| 未知方法 | 抛出 AttributeError |
| 后端初始化失败 | 抛出原始异常，附带模式信息 |

### 6.2 配置错误

| 场景 | 处理 |
|:--|:--|
| 缺少 API Key | 抛出 ValueError，提示设置环境变量 |
| 无效参数值 | 抛出 TypeError/ValueError |
| 未知参数 | 忽略并打印警告 |

## 7. 使用示例

### 7.1 文本模式

```python
from simplemem import SimpleMem

mem = SimpleMem(api_key="sk-...", model="gpt-4o")
mem.add_dialogue("Alice", "Let's meet at 2pm", "2025-11-15T14:30:00")
mem.add_dialogue("Bob", "Sure, see you then!", "2025-11-15T14:31:00")
mem.finalize()
answer = mem.ask("When will they meet?")
# → "16 November 2025 at 2:00 PM"
```

### 7.2 多模态模式

```python
from simplemem import SimpleMem

mem = SimpleMem(api_key="sk-...", model="gpt-4o")
mem.add_text("User loves hiking.", tags=["session_id:D1"])
mem.add_image("photo.jpg", tags=["session_id:D1"])
result = mem.query("What does the user enjoy?", top_k=5)
```

### 7.3 检索优化

```python
import simplemem

mem = simplemem.SimpleMem(api_key="sk-...")
# ... 添加对话 ...
config = simplemem.optimize(mem, dev_questions, max_rounds=3)
config.save("my_config.json")
```
