# Spec Delta

## Purpose

定义局域网 mosquitto MQTT v5 broker 的部署行为，以及服务端与 BL616CL 设备之间可关联、可超时、可报错的 Request-Reply 和通用协议调用消息契约。

## ADDED Requirements

### Requirement: MQTT v5 broker 可用

项目必须（SHALL）提供 docker-compose 编排的 mosquitto broker，默认监听 1883/TCP，启用 MQTT v5 协议、匿名访问和消息持久化。broker 容器重启后，retained 消息必须仍然可用。MVP 不启用 TLS 和 ACL。

#### Scenario: 设备以 MQTT v5 连接

- **WHEN** 设备使用 MQTT v5 CONNECT、任意用户名或空用户名连接到 broker 的 1883 端口
- **THEN** broker 接受连接并允许该客户端发布和订阅授权主题

#### Scenario: broker 重启后 retained 状态保留

- **WHEN** broker 容器重启，且重启前存在 retained 的设备状态消息
- **THEN** 新订阅者连接后仍能收到该 retained 消息

### Requirement: 主题命名空间

所有 MVP 消息必须（SHALL）使用 `iot/v1/devices/{device_id}/...` 主题前缀，其中 `{device_id}` 为设备的稳定唯一标识。设备请求主题为 `iot/v1/devices/{device_id}/req`，响应主题为 `iot/v1/devices/{device_id}/resp`，状态主题为 `iot/v1/devices/{device_id}/status`。设备上线后必须订阅自己的请求主题。

#### Scenario: 设备订阅自己的请求主题

- **WHEN** 设备成功建立 MQTT 连接
- **THEN** 设备以 QoS 1 订阅 `iot/v1/devices/{device_id}/req`，且不订阅其他设备的请求主题

#### Scenario: 主题与设备身份不匹配

- **WHEN** 服务端准备向设备 `dev-a` 发送请求
- **THEN** 服务端只向 `iot/v1/devices/dev-a/req` 发布消息，不向其他设备主题广播

### Requirement: MQTT v5 Request-Reply 关联

服务端发送请求时必须（SHALL）使用 MQTT v5 的 Response Topic 属性指定响应主题，并使用 Correlation Data 属性携带请求 ID。设备响应时必须把请求中的 Correlation Data 原样带回，并发布到请求指定的 Response Topic。MVP 只支持服务端发起请求、设备响应。

#### Scenario: 请求与响应正确关联

- **WHEN** 服务端向设备请求主题发布带 Response Topic 和 Correlation Data 的请求
- **THEN** 设备向该 Response Topic 发布响应，响应中的 Correlation Data 与服务端请求完全一致

#### Scenario: 缺少 Response Topic

- **WHEN** 设备收到没有 Response Topic 属性的请求
- **THEN** 设备不执行该请求，并通过日志报告协议错误

### Requirement: 请求超时与重试

服务端发起 `device.describe` 或 `protocol.invoke` 请求后必须（SHALL）等待响应，默认超时 5 秒；超时后最多重试 3 次，每次使用新的请求 ID 和 Correlation Data。全部尝试失败后，服务端必须向调用方返回明确的超时错误，并清理该请求的待处理状态。

#### Scenario: 设备及时响应

- **WHEN** 服务端在 5 秒内收到 Correlation Data 匹配的响应
- **THEN** 服务端取消超时计时器并把响应交给调用方，不再重试

#### Scenario: 设备无响应

- **WHEN** 服务端在 5 秒内未收到匹配响应，且 3 次重试均超时
- **THEN** 服务端返回 `timeout` 错误，错误信息包含设备 ID 和被调用的方法名

### Requirement: 统一的消息与错误语义

请求和响应负载必须（SHALL）使用 JSON 对象。请求包含 `id`、`method` 和可选 `params`；响应包含 `id`、`ok`，以及成功时的 `result` 或失败时的 `error`（含 `code` 和 `message`）。设备对未知方法、非法参数和内部执行失败必须返回错误响应而不是静默丢弃。

#### Scenario: 未知方法返回错误

- **WHEN** 设备收到 `method` 不在其支持列表中的请求
- **THEN** 设备返回 `ok: false` 且 `error.code` 为 `unsupported_method` 的响应

#### Scenario: 非法参数返回错误

- **WHEN** 设备收到参数缺失或类型错误的请求
- **THEN** 设备返回 `ok: false` 且 `error.code` 为 `invalid_params` 的响应

### Requirement: 协议调用方法

服务端必须（SHALL）通过 `protocol.invoke` 方法调用 Device Tree 中声明的协议函数。`params` 必须包含 `node_id`、`protocol_uuid`、`interface`、`function` 和可选 `args`。设备必须按 Device Tree 声明的函数签名校验 `args`，并按 `node_id + protocol_uuid + interface + function` 分发。服务端必须只发送已声明的组合；本地校验失败时返回 `unsupported_operation`，不发送 MQTT 请求。设备对未知节点、协议、接口、函数和参数错误必须分别返回 `unknown_node`、`unknown_protocol`、`unknown_interface`、`unknown_function`、`invalid_params`。

#### Scenario: 合法协议调用

- **WHEN** 服务端向设备发送 `protocol.invoke`，其中节点、Protocol UUID、Interface 和函数均在 Device Tree 中声明，且参数合法
- **THEN** 设备分发到对应 Handler 执行，并在响应中返回结果

#### Scenario: 服务端调用未声明函数

- **WHEN** 服务端尝试调用 Device Tree 中不存在的 Protocol 函数
- **THEN** 服务端返回 `unsupported_operation` 错误，不向设备发送 MQTT 请求

#### Scenario: 设备侧未知协议实例

- **WHEN** 设备收到 `protocol.invoke`，但 `node_id` 或 `protocol_uuid` 无法解析
- **THEN** 设备返回 `ok: false` 且 `error.code` 为 `unknown_node` 或 `unknown_protocol` 的响应

#### Scenario: 嵌套协议实例寻址

- **WHEN** 服务端调用挂在 I2C 节点下的 OLED 节点函数
- **THEN** 请求使用 OLED 节点的 `node_id`，设备分发到 OLED Handler，由其在内部复用父节点的 I2C Protocol

### Requirement: QoS 1 与幂等处理

MVP 所有请求、响应和状态消息必须（SHALL）使用 QoS 1。设备必须对相同请求 ID 的重复投递保持幂等：在 30 秒内收到重复请求时，必须返回缓存的结果而不是重复执行。服务端必须忽略 Correlation Data 无法匹配到待处理请求的响应。

#### Scenario: 重复请求不重复执行

- **WHEN** 设备在 30 秒内两次收到请求 ID 相同的请求
- **THEN** 设备只执行一次，并对第二次投递返回第一次的结果

#### Scenario: 迟到响应被忽略

- **WHEN** 服务端收到 Correlation Data 已过期或无法匹配的响应
- **THEN** 服务端丢弃该响应，不影响其他待处理请求

### Requirement: 设备在线状态主题

设备连接 broker 时必须（SHALL）在状态主题发布 retained 的 `online` 消息，并设置 Last Will 为同一主题的 retained `offline` 消息。设备正常断开前必须主动发布 `offline`。状态消息使用 QoS 1。

#### Scenario: 上线上报状态

- **WHEN** 设备成功连接 broker 并完成订阅
- **THEN** 设备向状态主题发布 retained 的 `online` 消息

#### Scenario: 异常掉线由 Last Will 标记

- **WHEN** 设备与 broker 的连接异常断开且未主动发布离线消息
- **THEN** broker 向状态主题发布设备预先注册的 retained `offline` 消息
