# OpenSquilla Channels 模块深度分析

## 一、模块概述

`opensquilla.channels` 是多平台消息适配层，将 Slack、Telegram、Discord、飞书、钉钉、企业微信、QQ、Matrix、MS Teams 等平台统一抽象为标准化的 `Channel` / `ManagedChannel` 协议。

## 二、Channel 适配器架构

```mermaid
classDiagram
    direction TB

    class Channel {
        <<Protocol>>
        +receive() IncomingMessage
        +send(message: OutgoingMessage) None
        +edit(message_id: str, content: str) None
        +delete(message_id: str) None
    }

    class ManagedChannel {
        <<Protocol>>
        +start() None
        +stop() None
        +health_check() ChannelHealth
    }

    class ChannelCapabilityProfile {
        +channel_type: str
        +group_chat: bool
        +mentions: bool
        +native_file_upload: bool
        +threads: bool
        +reactions: bool
        +capability_tags() frozenset
    }

    class ChannelManager {
        -_channels: dict
        -_tasks: dict
        -_dispatch_states: dict
        +from_config() ChannelManager
        +start_all() dict
        +stop_all() None
        +restart_channel(name) None
        +health() dict
    }

    class SlackChannel
    class TelegramChannel
    class DiscordChannel
    class FeishuChannel
    class DingTalkChannel
    class WeComChannel
    class QQChannel
    class MatrixChannel
    class MSTeamsChannel

    Channel <|-- ManagedChannel
    ManagedChannel <|.. SlackChannel
    ManagedChannel <|.. TelegramChannel
    ManagedChannel <|.. DiscordChannel
    ManagedChannel <|.. FeishuChannel
    ManagedChannel <|.. DingTalkChannel
    ManagedChannel <|.. WeComChannel
    ManagedChannel <|.. QQChannel
    ManagedChannel <|.. MatrixChannel
    ManagedChannel <|.. MSTeamsChannel
    ChannelManager o-- ManagedChannel : manages
```

## 三、消息流转流程

### 3.1 入站消息（Webhook 模式）

```mermaid
sequenceDiagram
    participant Platform as Slack Platform
    participant Webhook as Webhook Handler
    participant Adapter as SlackChannel
    participant Queue as asyncio.Queue
    participant Dispatch as Channel Dispatch
    participant Runner as TurnRunner

    Platform->>Webhook: HTTP POST /slack/events
    Webhook->>Webhook: 验证签名 + 去重
    Webhook->>Adapter: parse_event → IncomingMessage
    Adapter->>Queue: enqueue(IncomingMessage)
    Dispatch->>Adapter: receive() → IncomingMessage
    Dispatch->>Runner: run_turn(msg, session)
    Runner-->>Dispatch: OutgoingMessage
    Dispatch->>Adapter: send(OutgoingMessage)
```

### 3.2 流式消息发送

```mermaid
sequenceDiagram
    participant Runner as TurnRunner
    participant Policy as StreamPolicy
    participant Adapter as SlackChannel
    participant Throttle as StreamThrottle
    participant API as Slack API

    Runner->>Policy: resolve_channel_stream_policy()
    Policy-->>Runner: mode=adapter_stream
    Runner->>Adapter: send_streaming(chunks)
    loop For each chunk
        Adapter->>Throttle: add(chunk) + maybe_flush()
        alt First flush
            Throttle->>API: chat.postMessage
        else Subsequent flush
            Throttle->>API: chat.update
        end
    end
```

## 四、平台适配器一览

| 适配器 | 传输方式 | 流式策略 | 文件上传 |
|---|---|---|---|
| SlackChannel | Webhook / Socket Mode | adapter_stream | 外部上传 |
| TelegramChannel | Polling / Webhook | adapter_stream | sendDocument |
| DiscordChannel | Gateway WebSocket | adapter_stream | multipart |
| FeishuChannel | Webhook / WebSocket | final_only | 图片/文件上传 |
| DingTalkChannel | Stream Mode | adapter_stream (Card) | 不支持 |
| WeComChannel | Webhook (AES) | final_only | media/upload |
| QQChannel | WebSocket (botpy) | final_only | 不支持 |
| MatrixChannel | WebSocket (nio) | adapter_stream | media.upload |
| MSTeamsChannel | Webhook (Bot Framework) | adapter_stream + 降级 | 不支持 |

## 五、调度状态机

```mermaid
stateDiagram-v2
    [*] --> Running: start()
    Running --> Exhausted: 重试耗尽
    Exhausted --> Restarting: restart_count < 3
    Restarting --> Running: 重启成功
    Exhausted --> Dead: restart_count >= 3
    Dead --> Running: 管理员手动 restart_channel()
```

## 六、设计模式

| 模式 | 应用 |
|---|---|
| **适配器模式** | 每个平台适配器适配为统一 Channel 协议 |
| **协议驱动设计** | `Channel` / `ManagedChannel` 为 Protocol，非抽象基类 |
| **策略模式** | 流式策略、访问策略、能力声明 |
| **注册表模式** | `ChannelRegistration` + `discover_all()` + `entry_points` |
| **生产者-消费者** | `asyncio.Queue` 解耦入站/出站 |
| **熔断器** | `FloodStrikeBackoff` 429 洪泛退避 |
| **节流器** | `StreamThrottle` 序列化并发编辑 |
