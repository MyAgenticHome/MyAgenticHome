# Spec Delta

## Purpose

将服务端发现的设备、Device Tree 和 Protocol/Interface 函数通过 MCP 暴露给大模型，使模型能够列出设备、读取设备描述、发现 Protocol 并通用调用设备声明的函数。

## ADDED Requirements

### Requirement: MCP 服务端点

服务端必须（SHALL）以 Streamable HTTP 方式提供 MCP 服务，默认监听 `0.0.0.0:8080`，MCP 端点路径为 `/mcp`，监听地址和端口必须可配置。服务端必须在 MCP 初始化时声明 `tools` 和 `resources` 能力。MVP 不启用 MCP 层认证，但监听非回环地址时必须输出安全警告日志。

#### Scenario: MCP 客户端初始化

- **WHEN** MCP 客户端向 `/mcp` 发起初始化请求
- **THEN** 服务端返回协议版本、服务器信息和 `tools`、`resources` 能力声明

#### Scenario: 无设备时正常启动

- **WHEN** 服务端启动时没有任何设备记录
- **THEN** MCP 服务正常可用，`tools/list` 返回基础工具，`resources/list` 返回空设备列表

### Requirement: 设备资源

服务端必须（SHALL）提供 `iot://devices` 资源列出所有已知设备，并为每台设备提供 `iot://devices/{device_id}` 资源返回该设备的 Device Tree 和元数据（在线状态、是否 stale、`profile_revision`、最后上线时间）。资源内容必须随设备状态变化及时更新。

#### Scenario: 读取设备列表资源

- **WHEN** MCP 客户端读取 `iot://devices`
- **THEN** 服务端返回所有已知设备的 ID、型号、在线状态和 `profile_revision`

#### Scenario: 读取单台设备资源

- **WHEN** MCP 客户端读取 `iot://devices/dev-a`
- **THEN** 服务端返回 `dev-a` 的 Device Tree、在线状态和缓存新鲜度信息

#### Scenario: 未知设备资源

- **WHEN** MCP 客户端读取不存在的 `iot://devices/dev-x`
- **THEN** 服务端返回资源不存在的错误，不创建空资源

### Requirement: 基础工具

服务端必须（SHALL）提供以下基础 MCP 工具：`list_devices` 返回设备摘要列表；`get_device_tree` 接收 `device_id` 返回完整 Device Tree；`list_protocols` 接收 `device_id` 返回节点、Protocol UUID、Interface 和函数签名摘要；`invoke_protocol` 接收 `device_id`、`node_id`、`protocol_uuid`、`interface`、`function` 和可选 `args` 执行通用协议调用；`refresh_device_profile` 接收 `device_id` 强制重新获取设备描述。工具参数必须经过校验，未知或不存在的 `device_id` 必须返回结构化错误，未声明的协议函数必须返回 `unsupported_operation`。

#### Scenario: 列出设备

- **WHEN** 大模型调用 `list_devices`
- **THEN** 服务端返回所有已知设备的 ID、型号、在线状态和可用函数数量

#### Scenario: 列出设备 Protocol

- **WHEN** 大模型对在线设备 `dev-a` 调用 `list_protocols`
- **THEN** 服务端返回 `dev-a` 的节点路径、Protocol UUID、Interface 和函数签名摘要

#### Scenario: 通用协议调用

- **WHEN** 大模型调用 `invoke_protocol`，参数组合在 Device Tree 中已声明且 `args` 合法
- **THEN** 服务端向设备发送 `protocol.invoke` 并返回函数结果

#### Scenario: 刷新设备描述

- **WHEN** 大模型对在线设备 `dev-a` 调用 `refresh_device_profile`
- **THEN** 服务端向设备发送 `device.describe`，成功后更新资源、Protocol 索引并返回新的 `profile_revision`

#### Scenario: 设备不存在

- **WHEN** 大模型对不存在的设备调用 `get_device_tree` 或 `list_protocols`
- **THEN** 服务端返回 `device_not_found` 错误，不发送 MQTT 请求

### Requirement: 动态协议函数工具

设备上线并成功建立 Protocol 索引后，服务端必须（SHALL）为每个 Protocol Interface 函数生成一个 MCP 工具。工具名必须由设备 ID、节点 ID、Protocol、Interface 和函数名确定性地生成，只包含小写字母、数字和下划线，总长度不超过 64 字符，并保证全局唯一。设备离线或描述被移除时，对应工具必须从 `tools/list` 中移除。

#### Scenario: 上线后 OLED 工具出现

- **WHEN** 设备 `dev-a` 上线且其 OLED 节点声明 `display_text` 函数
- **THEN** `tools/list` 中出现由设备、OLED 节点、Protocol、Interface 和 `display_text` 生成的工具，且参数与 Device Tree 声明一致

#### Scenario: 嵌套节点工具使用子节点 ID

- **WHEN** OLED 节点是 I2C 节点的子节点且声明 `display_text`
- **THEN** 生成的工具以 OLED 节点的 `node_id` 寻址，而不是 I2C 父节点

#### Scenario: 离线后工具移除

- **WHEN** 设备 `dev-a` 变为 `offline`
- **THEN** 服务端从 `tools/list` 中移除 `dev-a` 的所有动态工具，其资源仍保留并标记为离线

#### Scenario: 工具名冲突

- **WHEN** 两台设备或两个协议函数的组合生成相同工具名
- **THEN** 服务端对后注册项追加确定性哈希后缀，保证工具名唯一且重启后保持一致

### Requirement: 工具调用路由

动态工具和 `invoke_protocol` 调用必须（SHALL）先按 Device Tree 校验节点、Protocol、Interface、函数和参数，再映射为 `protocol.invoke` MQTT 请求发送给目标设备，并把设备响应转换为 MCP 工具结果。设备离线、请求超时、函数未声明或参数非法时，必须返回包含错误码和可读说明的 MCP 错误结果，不得抛出未处理异常。

#### Scenario: 成功调用协议函数

- **WHEN** 大模型调用 `dev-a` 的 OLED `display_text` 工具且参数合法
- **THEN** 服务端向设备发送 `protocol.invoke`，并把 `result` 作为工具结果返回

#### Scenario: 调用离线设备

- **WHEN** 大模型调用某离线设备的动态工具或 `invoke_protocol`
- **THEN** 服务端返回 `device_offline` 错误，不发送 MQTT 请求

#### Scenario: 设备返回协议错误

- **WHEN** 设备对 `protocol.invoke` 返回 `ok: false`
- **THEN** 服务端把设备的 `error.code` 和 `error.message` 原样映射为 MCP 工具错误结果

#### Scenario: 参数不符合函数签名

- **WHEN** 大模型传入的 `args` 不符合 Device Tree 中声明的函数参数
- **THEN** 服务端返回 `invalid_params` 错误，不发送 MQTT 请求

### Requirement: stale 缓存处理

对标记为 `stale` 的设备，服务端必须（SHALL）继续在资源中展示缓存的 Device Tree 并明确标注 stale。调用该设备的动态工具前，服务端必须先尝试 `refresh_device_profile`；刷新成功则继续执行调用，刷新失败则返回 `device_profile_stale` 错误。

#### Scenario: stale 设备刷新后调用

- **WHEN** 大模型调用 `stale` 设备 `dev-a` 的工具，且刷新描述成功
- **THEN** 服务端使用刷新后的 Device Tree 校验参数并完成工具调用

#### Scenario: stale 设备刷新失败

- **WHEN** 大模型调用 `stale` 设备 `dev-a` 的工具，但设备描述刷新超时
- **THEN** 服务端返回 `device_profile_stale` 错误，不执行设备操作
