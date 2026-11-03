# Spec Delta

## Purpose

定义 BL616CL 设备端固件从 WiFi 入网、发现服务端、连接 MQTT broker 到响应描述与协议调用请求的完整运行时行为，包括 EDK2 风格 Protocol/Interface/Handler 注册表。

## ADDED Requirements

### Requirement: 稳定设备身份

固件必须（SHALL）从芯片唯一标识（MAC 或 eFuse ID）派生设备 ID，格式为 `bl616cl-` 加 12 位小写十六进制字符。设备 ID 必须在重启、重新烧录应用固件和 broker 配置变化后保持不变，并用作 MQTT client ID 和所有主题中的 `{device_id}`。

#### Scenario: 重启后身份不变

- **WHEN** 设备断电重启
- **THEN** 设备使用与重启前相同的设备 ID 连接 broker，并在发现请求中上报同一 ID

#### Scenario: 不同设备身份不同

- **WHEN** 两台 BL616CL 设备同时运行
- **THEN** 它们生成不同的设备 ID，且不会覆盖彼此的 MQTT 状态主题

### Requirement: WiFi 入网

固件必须（SHALL）使用预置在非易失存储或构建配置中的 WiFi SSID 和密码以 STA 模式连接网络。连接失败时必须按 1 秒、2 秒、5 秒、10 秒、30 秒退避重试，之后保持每 30 秒重试。MVP 不提供 AP 配网页面或蓝牙配网。

#### Scenario: 启动后自动入网

- **WHEN** 设备上电且已保存有效 WiFi 凭据
- **THEN** 设备在 30 秒内尝试连接 WiFi，并在成功后进入发现或 broker 连接流程

#### Scenario: WiFi 凭据无效

- **WHEN** 设备使用保存的凭据无法在 60 秒内连接 WiFi
- **THEN** 设备保持每 30 秒重试，并通过串口日志报告连接失败原因

### Requirement: 发现与 broker 配置应用

WiFi 连接成功后，固件必须（SHALL）按 `device-discovery` 规范执行发现，除非本地存在有效期内的 broker 配置。收到有效配置后，固件必须保存 broker 地址、端口、凭据和有效期，并立即尝试连接 broker。

#### Scenario: 首次上电完成配网

- **WHEN** 设备首次上电、WiFi 连接成功且本地无 broker 配置
- **THEN** 设备广播发现请求，收到有效响应后保存配置并在 2 秒内发起 MQTT 连接

#### Scenario: 已有配置跳过发现

- **WHEN** 设备重启且本地 broker 配置仍在有效期内
- **THEN** 设备不广播发现请求，直接连接已保存的 broker

### Requirement: MQTT v5 客户端行为

固件必须（SHALL）使用 MQTT v5 连接 broker，启用 Response Topic 和 Correlation Data 属性，keepalive 为 60 秒。连接前必须注册状态主题的 retained `offline` Last Will；连接成功后必须订阅自己的请求主题并发布 retained 的 `online` 状态。连接断开后必须按 1 秒到 60 秒的指数退避自动重连。

#### Scenario: 连接成功后订阅并上报在线

- **WHEN** 设备成功完成 MQTT v5 CONNECT
- **THEN** 设备订阅 `iot/v1/devices/{device_id}/req`，并向状态主题发布 retained 的 `online`

#### Scenario: broker 重启后自动恢复

- **WHEN** broker 重启导致设备连接断开
- **THEN** 设备按指数退避重连，恢复后重新订阅请求主题并发布 `online`

### Requirement: 描述与协议调用处理

固件必须（SHALL）处理 `device.describe` 和 `protocol.invoke` 两个方法。`device.describe` 必须返回完整 Device Tree；`protocol.invoke` 必须校验 `node_id`、`protocol_uuid`、`interface`、`function` 和参数后分发到对应 Handler，并返回结果或错误。未知方法、未知节点、未知协议、未知接口、未知函数、参数错误和执行失败都必须返回符合 `mqtt-messaging` 错误语义的响应。

#### Scenario: 返回设备描述

- **WHEN** 设备收到 `device.describe` 请求
- **THEN** 设备在 1 秒内返回包含完整 Device Tree 和当前 `profile_revision` 的成功响应

#### Scenario: 执行声明的协议函数

- **WHEN** 设备收到针对已声明节点、Protocol、Interface 和函数的合法 `protocol.invoke` 请求
- **THEN** 设备分发到对应 Handler 执行，并在 5 秒内返回函数结果

#### Scenario: 协议执行失败

- **WHEN** Handler 因外设错误无法完成操作
- **THEN** 设备返回 `ok: false` 且 `error.code` 为 `execution_failed` 的响应，设备保持运行

### Requirement: Protocol/Interface/Handler 注册表

固件必须（SHALL）实现参考 EDK2 的 Protocol 注册表：每个 Protocol 实现以静态表注册 `uuid`、名称、Interface 函数表和 Handler 函数。Handler 接收解析后的 JSON 参数并返回 JSON 结果；Handler 可以通过注册表查找并调用其他 Protocol，从而实现 I2C 节点上的 OLED 驱动 Protocol 复用父节点 I2C Protocol 的嵌套组合。MVP 只支持编译期静态注册，不加载动态原生代码。

#### Scenario: 注册 I2C Protocol

- **WHEN** 固件启动并初始化 I2C 总线
- **THEN** I2C Protocol 以其 UUID 和 Interface 函数表出现在注册表中，并可通过 `protocol.invoke` 调用

#### Scenario: OLED Handler 复用 I2C Protocol

- **WHEN** 服务端调用 OLED 节点的 `display_text` 函数
- **THEN** OLED Handler 在内部通过注册表调用父节点的 I2C Protocol 完成写屏，并返回执行结果

#### Scenario: 未注册的 Protocol UUID

- **WHEN** `protocol.invoke` 请求中的 `protocol_uuid` 不在注册表中
- **THEN** 设备返回 `unknown_protocol` 错误，不执行任何硬件操作

### Requirement: Device Tree 由 Protocol 注册表生成

固件必须（SHALL）在启动时根据编译期 Protocol 注册表和板级节点配置生成 Device Tree，描述板载总线、外设、嵌套 Protocol、Interface 和函数签名。生成的文档必须符合 `device-profile` 规范，节点 ID 在固件生命周期内稳定，`profile_revision` 在板级配置或 Protocol 集合改变时递增。设备树生成不得依赖运行时动态扫描产生不稳定的节点顺序。

#### Scenario: 板级配置决定设备树内容

- **WHEN** 使用包含 I2C 总线和 OLED 屏幕的板级配置构建固件
- **THEN** 启动后生成的 Device Tree 包含 I2C Protocol 父节点、OLED Protocol 子节点及两者的 Interface 函数

#### Scenario: 配置变化递增描述版本

- **WHEN** 板级配置新增一个外设 Protocol 并重新构建烧录
- **THEN** 新固件上报的 `profile_revision` 大于旧固件上报的值

### Requirement: 配置持久化与恢复出厂

固件必须（SHALL）把 broker 配置保存到非易失存储，并在收到串口 `factory_reset` 命令或板载按键长按 5 秒时清除 WiFi 和 broker 配置后重启。配置写入失败时不得破坏已有的有效配置；读取到损坏配置时必须视为无配置并重新执行发现。

#### Scenario: 恢复出厂后重新配网

- **WHEN** 用户在串口输入 `factory_reset` 并确认，设备重启
- **THEN** 设备清除 WiFi 与 broker 配置，并按首次上电流程重新入网和发现

#### Scenario: 配置损坏时自愈

- **WHEN** 设备启动时读取到校验失败的 broker 配置
- **THEN** 设备把配置视为不存在，WiFi 连接成功后重新执行发现流程
