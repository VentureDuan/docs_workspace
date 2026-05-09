# 技术方案：CloudGameTrace

## 背景

云游戏 CallTrace 平台借鉴 5GC 通讯行业中的 CallTrace 思想，将一次云游戏串流过程建模成一条端到端故障排查链路。

核心类比：

| 5GC CallTrace | 云游戏 CloudGameTrace |
| --- | --- |
| 用户 IMSI / SUPI | userId / accountId / deviceId |
| 一次注册、上网、呼叫 | 一次云游戏启动、串流、操作、断连 |
| AMF / SMF / UPF | 调度服务、游戏实例、串流网关、边缘节点 |
| PDU Session | Game Session / Stream Session |
| TEID / Call-ID / Session-ID | sessionId / streamId / roomId / rtcConnectionId |
| cause code | errorCode / closeReason / networkReason / decoderError |
| CallTrace 时间线 | CloudGameTrace 会话时间线 |

一句话定义：

> 把一次云游戏会话当成“一次呼叫”，把启动、调度、实例、编码、传输、播放、输入回传都当成“信令和承载过程”，用 traceId、sessionId、connectionId 串成端到端排障链路。

## 问题

云游戏串流故障经常跨越客户端、调度、实例、编码器、网络传输、边缘节点、播放器和输入回传。单看播放器报错、服务端日志或局部监控，很难判断问题发生在哪一段。

典型问题包括：

- 首帧慢
- 黑屏
- 卡顿
- 断连
- 输入延迟高
- 编码异常
- 边缘链路异常
- 调度或实例启动异常

## 方案

建设一套 `CloudGameTrace / SessionTrace` 系统，以“玩家一次游戏会话”为中心，把一次串流过程拆成阶段化事件，并聚合成端到端时间线、瀑布图、拓扑图和根因候选。

核心能力：

- 全局 `traceId` 贯穿客户端、控制面、游戏运行时、边缘节点和 Trace 平台。
- 区分 `gameSessionId`、`streamSessionId`、`connectionId`，支持重连和多连接分析。
- 统一事件模型，所有事件包含阶段、事件类型、状态、耗时、错误码和扩展属性。
- 使用 MQ 承载 Trace 事件流，按会话聚合。
- 用状态机重组阶段，计算阶段耗时、超时、失败和缺失事件。
- 在 UI 中展示时间线、阶段耗时瀑布图、链路拓扑、关键指标和原始日志引用。

## 为什么这么做

云游戏 Trace 的关键不是收集更多日志，而是把分散事件放回同一条业务会话链路中。

这样做的价值：

- 从“局部报错”升级为“端到端链路定位”。
- 可以把服务端指标和客户端体验放在同一条时间线上对比。
- 可以通过分段指标判断根因候选，例如编码器、主机到边缘、边缘到客户端、客户端解码或渲染。
- 可以复用通信行业 CallTrace 的成熟思路，但字段、阶段和故障分类按云游戏业务重建。

## 链路阶段

一次云游戏会话建议拆成以下阶段：

```text
AUTH
MATCH / QUEUE
SCHEDULING
RESOURCE_ALLOC
INSTANCE_START
GAME_PROCESS_START
ENCODER_INIT
STREAM_SIGNALING
NETWORK_CONNECT
FIRST_FRAME
PLAYING
INPUT_LOOP
ADAPTIVE_BITRATE
RECONNECT
SESSION_RELEASE
```

每个阶段至少要有：

- 开始事件
- 成功事件
- 失败事件
- 超时判断
- 耗时指标
- 失败原因码

## 采集点

优先采集 8 类节点：

- 客户端 Client：点击开始、建连耗时、首帧耗时、播放器状态、解码耗时、帧率、卡顿、输入延迟、丢包、RTT、断连原因。
- API Gateway：startGame 请求、鉴权结果、用户区域、设备类型、客户端版本、调度请求耗时、返回的 sessionId、edgeId、streamUrl。
- 调度系统 Scheduler：机房、边缘节点、GPU 池、实例类型、排队时长、资源不足原因、亲和性策略、延迟评估。
- 游戏实例 / 容器 / 云主机：实例创建、镜像拉取、容器启动、GPU 绑定、游戏进程 ready、崩溃、退出码、CPU/GPU/显存/磁盘 IO。
- 编码器 Encoder：编码器初始化、编码格式、分辨率、码率、FPS、编码耗时、队列积压、关键帧、硬编错误、GPU encoder saturation。
- 串流传输层：WebRTC/QUIC/UDP 建连状态、ICE、DTLS/SRTP、connectionId、RTT、jitter、packet loss、NACK、PLI/FIR、带宽估计、拥塞控制。
- 边缘节点 / Relay / SFU / TURN：上下游连接、用户到边缘 RTT、边缘到云主机 RTT、上下行丢包、转发队列、buffer 水位、节点负载。
- 输入回传 Input Channel：按键产生时间、客户端发送时间、服务端接收时间、游戏进程消费时间、反馈帧编码时间、反馈帧到达客户端时间。

## 关联键设计

必须从客户端点击“开始游戏”时生成全局 `traceId`。

建议核心 ID：

```text
traceId              // 一次完整启动和串流链路
gameSessionId        // 一次游戏业务会话
streamSessionId      // 一次音视频串流会话
connectionId         // 一次底层网络连接
instanceId           // 云端游戏实例
containerId          // 容器 ID
hostId               // 物理机 / GPU 机器
edgeNodeId           // 边缘节点
playerId / userId    // 用户
deviceId             // 设备
requestId            // 单个 API 请求
```

关键判断：

- `gameSessionId` 表示整局游戏生命周期。
- `streamSessionId` 表示一次串流连接生命周期。
- `connectionId` 表示一次底层网络连接生命周期。
- 重连、切边缘、切码率时不能把这些 ID 混用。

## 事件模型

统一事件模型建议命名为 `CloudGameTraceEvent`。

关键字段：

| 字段 | 作用 |
| --- | --- |
| traceId | 端到端主线 |
| gameSessionId | 游戏业务会话 |
| streamSessionId | 串流会话 |
| connectionId | 网络连接 |
| eventTime | 事件真实发生时间 |
| receiveTime | Trace 平台收到事件时间 |
| service | 事件来源服务 |
| stage | 链路阶段 |
| eventType | 具体事件 |
| status | 成功、失败、超时 |
| errorCode | 统一错误码 |
| latencyMs | 阶段或事件耗时 |
| attributes | 协议或业务特有字段 |

事件必须阶段化，不要只上报普通日志。

示例：

```text
stage = FIRST_FRAME
eventType = FIRST_FRAME_RENDERED
latencyMs = 2460
status = SUCCESS
```

## 总体架构

推荐链路：

```text
Trace SDK / Agent / Probe
    -> Kafka / Pulsar
    -> Normalizer
    -> Aggregator / FSM
    -> ClickHouse / OpenSearch
    -> Trace UI / Alert / Root Cause Engine
```

Topic 初步设计：

```text
cloudgame.trace.raw.client
cloudgame.trace.raw.gateway
cloudgame.trace.raw.scheduler
cloudgame.trace.raw.instance
cloudgame.trace.raw.encoder
cloudgame.trace.raw.stream
cloudgame.trace.raw.edge
cloudgame.trace.normalized
cloudgame.trace.session
cloudgame.trace.alert
```

Kafka message key 建议优先使用 `traceId` 或 `gameSessionId`，让同一会话事件尽量进入同一分区，降低聚合复杂度。

## 实时聚合

聚合逻辑：

```text
消费 normalized event
 -> 按 traceId 分组
 -> 放入状态缓存
 -> 按 eventTime 排序
 -> 根据状态机推进阶段
 -> 计算阶段耗时
 -> 判断超时 / 缺失 / 错误码
 -> 输出 session trace 结果
```

状态机聚合要重点处理：

- 事件乱序
- 事件缺失
- 客户端和服务端时钟偏差
- 会话长时间未完成
- 重连导致多个 streamSessionId
- 同一个 gameSessionId 下多条 connectionId

## 故障分类

初始故障分类：

- 启动类：`START_GAME_API_TIMEOUT`、`AUTH_FAILED`、`QUEUE_TIMEOUT`、`SCHEDULER_NO_RESOURCE`、`INSTANCE_ALLOC_FAILED`、`CONTAINER_START_TIMEOUT`、`GAME_PROCESS_CRASH`。
- 编码类：`ENCODER_INIT_FAILED`、`ENCODER_FPS_ZERO`、`ENCODER_QUEUE_BLOCKED`、`GPU_ENCODER_SATURATED`、`CAPTURE_NO_FRAME`。
- 建连类：`SIGNALING_FAILED`、`ICE_FAILED`、`TURN_ALLOC_FAILED`、`QUIC_HANDSHAKE_FAILED`、`DTLS_FAILED`。
- 播放类：`FIRST_FRAME_TIMEOUT`、`DECODER_INIT_FAILED`、`DECODE_ERROR`、`RENDER_ERROR`、`BLACK_SCREEN`、`FREEZE`。
- 网络类：`HIGH_RTT`、`HIGH_JITTER`、`HIGH_PACKET_LOSS`、`BANDWIDTH_DROP`、`EDGE_TO_CLIENT_LOSS`、`HOST_TO_EDGE_LOSS`。
- 输入类：`INPUT_CHANNEL_DISCONNECTED`、`INPUT_RTT_HIGH`、`INPUT_EVENT_DROPPED`、`GAME_NOT_CONSUMING_INPUT`、`INPUT_TO_DISPLAY_LATENCY_HIGH`。

## 根因判断方法

核心方法是“分段对比”：

```text
Game Render
 -> Capture
 -> Encode
 -> Host Send
 -> Edge Receive
 -> Edge Send
 -> Client Receive
 -> Decode
 -> Render Display
```

黑屏示例：

- `Game Render FPS > 0` 且 `Capture FPS = 0`：采集画面失败、游戏窗口句柄异常、显卡输出异常。
- `Capture FPS > 0` 且 `Encoder FPS = 0`：编码器异常、GPU 编码资源不足、编码器阻塞。
- `Encoder FPS > 0` 且 `Host Send Bitrate > 0` 且 `Edge Receive Bitrate = 0`：云主机到边缘链路异常、防火墙或路由异常。
- `Edge Send Bitrate > 0` 且 `Client Receive Bitrate = 0`：边缘到用户网络异常、NAT、UDP 阻断、运营商链路异常。
- `Client Receive Bitrate > 0` 且 `Client Decode FPS = 0`：客户端解码失败、编码格式不兼容、硬解异常。
- `Client Decode FPS > 0` 且 `Client Render FPS = 0`：播放器渲染异常、Surface / Texture / GPU 渲染问题。

## MVP

第一阶段：统一 `traceId`。

- 接入 Client、API Gateway、Scheduler、Instance Manager、Streaming Server、Edge Relay。
- 所有服务打出 `traceId`、`gameSessionId`、`streamSessionId`、`userId`、`stage`、`eventType`、`timestamp`、`status`、`errorCode`、`latencyMs`。

第二阶段：事件进 Kafka。

- 各模块通过 SDK 上报到 `cloudgame.trace.normalized`。
- 不追求所有原始日志入库，先保证关键事件完整。

第三阶段：ClickHouse 存储。

- 明细表按 `event_time` 分区。
- 查询优先支持 `traceId`、`userId`、`gameSessionId`、时间范围。

第四阶段：Trace UI。

- 时间线 Timeline。
- 阶段耗时瀑布图 Waterfall。
- 链路拓扑图 Topology。

## 关键设计原则

- `traceId` 必须从客户端开始生成。
- 必须区分 `gameSessionId`、`streamSessionId`、`connectionId`。
- 事件必须阶段化，包含阶段、事件、状态、耗时、失败码。
- 服务端指标和客户端指标要放在同一条时间线。
- Trace 事件必须保留原始日志、指标、报文或录屏引用，例如 `logRef`、`metricRef`、`pcapRef`、`recordingRef`。

## 风险和迁移成本

- 接入成本：需要改造客户端、控制面、实例侧、边缘节点和串流服务的埋点。
- 数据治理成本：必须统一错误码、阶段枚举、事件模型和 ID 传播规则。
- 存储成本：高频客户端指标、网络指标和播放指标可能带来较大写入量，需要采样和分层存储。
- 时间一致性风险：多端时钟不一致会影响时间线，需要同时记录 `eventTime` 和 `receiveTime`。
- 聚合复杂度：乱序、缺失、重连、多连接会让状态机复杂化。
- 隐私和合规风险：用户 ID、设备 ID、IP、网络信息需要脱敏和权限控制。
- 迁移风险：如果已有日志、指标、Trace 体系字段不一致，需要设计兼容层，而不是一次性推翻重做。

## 踩坑

- 只看服务端指标可能误判客户端或网络问题。
- 只看播放器报错无法定位调度、实例、编码和边缘链路问题。
- 只用一个 sessionId 会在重连、切边缘、切码率时混淆会话。
- 不保留原始日志引用，会导致 Trace UI 只能看摘要，无法继续深挖。

## 下次怎么复用

设计或评审云游戏 Trace 时，按以下顺序检查：

1. 是否从客户端生成并贯穿 `traceId`。
2. 是否区分 `gameSessionId`、`streamSessionId`、`connectionId`。
3. 是否按阶段定义事件和错误码。
4. 是否覆盖客户端、调度、实例、编码器、传输、边缘、输入回传。
5. 是否能把服务端指标和客户端指标放在同一条时间线。
6. 是否能通过分段对比给出根因候选。
7. 是否保留原始日志、指标、报文和录屏引用。

## 相关链接

- [[01_Projects/云游戏CallTrace平台/项目概览|云游戏CallTrace平台 - 项目概览]]
- [[00_Inbox/agent-drafts/云游戏 Trace |云游戏 Trace 草稿]]

## 来源

- 用户在 2026-05-09 的当前对话中说明：“这是云游戏CallTrace平台的技术方案，借鉴5GC通讯行业中的CallTrace技术实现的云游戏Trace，解决一次串流过程中端到端的故障排查链路。”
- 草稿文件：`00_Inbox/agent-drafts/云游戏 Trace .md`。

## 假设

- 当前目标是沉淀技术方案入口，而不是生成最终 PRD 或详细实施排期。
- 云游戏链路中存在客户端、调度、实例管理、串流服务、边缘节点和 Trace 平台等组件。
- MQ、ClickHouse、OpenSearch、Trace UI 等选型仍可根据现有基础设施调整。

## 待确认项

- 现有云游戏链路的真实组件名称和边界。
- 当前是否已有 traceId、sessionId、streamId、connectionId。
- 现有日志、指标、Trace、告警系统分别是什么。
- 首期 MVP 要覆盖哪些故障场景。
- 采样策略、数据保留周期、脱敏规则和权限模型。
