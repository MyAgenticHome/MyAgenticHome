# Proposal

## Why

当前局域网 IoT 设备的接入依赖手工配置 MQTT broker 地址、手工维护设备能力描述，大模型无法直接感知设备上有哪些传感器和外设。本项目要构建一个端到端的 agentic IoT 平台：设备上电入网后自动发现服务端并完成 broker 配置，服务端通过 MQTT v5 Request-Reply 获取设备自描述的 Device Tree，再通过 MCP 把这些能力暴露给大模型，使自然语言可以查询和操控硬件。

设备能力需要一套可扩展的抽象，而不是为每种外设硬编码 RPC。本 change 引入参考 EDK2 的 Protocol/Interface/Handler 模型：Device Tree 用 UUID 描述板子支持哪些 Protocol、每个 Protocol 的 Interface 函数，以及 I2C 总线 → OLED 驱动这类嵌套关系；服务端通过通用 `protocol.invoke` 找到并调用这些 Interface。复杂控制流程所需的设备端 WASM 虚拟机、WASM 编译与下发不在本 change 范围，留到后续 change；本 change 先把 WASM 将来要复用的协议调用面打好。

## What Changes

- 新增 Go 实现的 MCP 服务端，部署在 NAS 或树莓派上：监听设备发现广播、下发 broker 配置、维护设备在线状态与 Device Tree 缓存、索引 Protocol，并将协议函数映射为 MCP tools/resources。
- 新增基于 docker-compose 的 mosquitto broker 编排，启用 MQTT v5，作为局域网内设备与服务端的消息总线。
- 新增 BL616CL 与 BL618 两种嵌入式固件目标（基于固定 Bouffalo SDK，原始 BL618 不等同于 BL618DG）：WiFi STA 入网、UDP 广播发现服务端、接收并持久化 broker 信息、建立 MQTT v5 连接、响应 Request-Reply 请求、上报 Device Tree。
- 提取跨芯片固件公共运行时与平台适配契约：公共协议/注册表、平台后端和板级 target 分层；实际芯片标识与 SDK CHIP/BOARD 分开，构建产物按 TARGET 隔离，后续芯片复用线上协议而非增加服务端芯片分支。
- 定义 v0 协议契约：局域网发现协议（UDP 广播/单播响应）、MQTT v5 主题与关联 ID 约定、Device Tree 描述格式（可扩展、带版本号）。
- Device Tree 节点使用 EDK2 风格的 `protocols` 声明：每个 Protocol 包含 UUID、名称、Interface 和函数签名；节点可以作为 Protocol 实例，也可以作为父 Protocol 的挂载点（例如 I2C Protocol 节点下挂 OLED 驱动 Protocol 节点）。
- 新增通用 `protocol.invoke` MQTT 方法：服务端按 `node_id + protocol_uuid + interface + function + args` 调用设备 Protocol；设备通过静态 Handler 注册表分发到原生处理函数。
- MCP 网关新增 `list_protocols`、`invoke_protocol` 工具，并为每个协议函数生成动态工具；服务端只允许调用 Device Tree 中声明的函数。
- 服务端在设备上线时自动请求 Device Tree，并将 Protocol 索引动态注册到 MCP；设备下线后从可用能力中移除。
- MVP 安全边界：局域网明文通信、broker 匿名访问；协议与配置预留认证字段，后续 change 再引入凭证与 TLS。
- **Non-Goal（本 change 不做）**：设备端 WASM 虚拟机集成、WASM 模块编译/分块下发/执行、WASM 宿主 ABI。这些作为后续独立 change；当前固件协议注册表和 `protocol.invoke` 设计需为将来的 WASM host import 留出复用接口。

## Capabilities

### New Capabilities

- `device-discovery`: 嵌入式端通过 UDP 广播宣告自身，服务端监听并单播回传 MQTT broker 连接信息，完成设备首次 broker 配置（不负责 WiFi 配网）。
- `mqtt-messaging`: mosquitto MQTT v5 broker 的部署配置，以及服务端与设备之间 Request-Reply 的主题、关联 ID、通用 `protocol.invoke` 方法和错误语义。
- `device-profile`: Device Tree 的节点树、Protocol/Interface 声明、Protocol UUID、嵌套协议关系、版本协商、上报时机与校验规则。
- `device-registry`: 服务端维护设备生命周期（发现、上线、离线、超时）、Device Tree 缓存和 Protocol 索引。
- `mcp-device-gateway`: 将已上线设备的 Protocol 索引和 Interface 函数暴露为 MCP resources 与 tools，供大模型查询和通用调用。
- `firmware-platform`: 跨芯片公共固件运行时、平台接口、板级 target、构建隔离及逐芯片验收契约。
- `bl616-firmware`: BL616CL 的稳定身份、SDK/板级映射和独立验收，继承公共运行时。
- `bl618-firmware`: BL618 的稳定身份、真实 chip=bl618 / sdk_chip=bl616 映射、板级适配与双芯片混合验收。

### Modified Capabilities

无。项目当前没有已存在的 spec。

## Impact

- 新增代码目录：`server/`（Go MCP 服务端）、`broker/`（mosquitto 配置与 compose）、`firmware/`（BL616CL/BL618 共用运行时、平台后端与板级 profiles，链接 `bouffalo/bouffalo_sdk` 子模块）。
- 新增 Go 依赖：MCP Go SDK、MQTT v5 客户端、bbolt 和选定的 YAML 解析包；日志使用标准库 slog，具体兼容版本在实现前验证并锁定。
- 嵌入式侧依赖固定提交的 Bouffalo SDK WiFi、lwIP、cJSON、littlefs KV/flash 与板级 I2C 驱动；该提交中的 MQTT-C 为 MQTT 3.1.1，另需评估并移植 MQTT v5 客户端，必要时实现受限子集。本 change 不启用 WAMR；未来 WASM 可复用原生协议调用面，但宿主 ABI 另行设计。
- 新增 OpenSpec specs：上述 8 个 capability。
- 不改变现有 Bouffalo SDK 子模块内容；固件应用作为独立工程引用 SDK，必要的自定义 BOARD 放在应用目录。BL618 的 CHIP=bl616 映射有固定提交文档依据，但实板 BOARD、引脚、存储和资源预算仍需逐目标验证。其他芯片的实际适配不在本 change。
