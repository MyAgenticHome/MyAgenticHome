# Spec Delta

## Purpose

将服务端发现的设备、Device Tree 和 Protocol/Interface 函数通过 MCP 暴露给大模型，使模型能够列出设备、读取设备描述、发现 Protocol 并通用调用设备声明的函数。

## ADDED Requirements

### Requirement: MCP 服务端点

服务端必须（SHALL）以 Streamable HTTP 方式提供 MCP 服务，默认监听 `127.0.0.1:8080`，MCP 端点路径为 `/mcp`，监听地址和端口必须可配置。服务端必须在 MCP 初始化时声明 `tools.listChanged` 和 `resources.listChanged` 能力。MVP 不启用 MCP 层认证；显式监听非回环地址时必须输出安全警告，并要求部署者用可信 LAN、防火墙或已鉴权反向代理限制访问。HTTP 必须校验 Host 和已提供的 Origin（使用显式允许列表，拒绝非法来源）；不得默认允许通配 CORS，不得将匿名硬件控制端点暴露到公网。

#### Scenario: MCP 客户端初始化

- **WHEN** MCP 客户端向 `/mcp` 发起初始化请求
- **THEN** 服务端返回协议版本、服务器信息和 `tools`、`resources` 能力声明

#### Scenario: 无设备时正常启动

- **WHEN** 服务端启动时没有任何设备记录
- **THEN** MCP 服务正常可用，`tools/list` 返回基础工具，`resources/list` 至少返回 `iot://devices`，读取该资源得到空数组

### Requirement: 设备资源

服务端必须（SHALL）提供 `iot://devices` 资源列出所有未过期的已知设备（discovered/online/offline，默认排除 expired），并为每台设备提供 `iot://devices/{device_id}` 资源返回该设备的 Device Tree 和元数据（在线状态、是否 stale、`profile_revision`、最后上线时间）。资源内容必须随设备状态变化及时更新。

#### Scenario: 读取设备列表资源

- **WHEN** MCP 客户端读取 `iot://devices`
- **THEN** 服务端返回所有未过期已知设备的 ID、型号、连接状态、connection_confirmed、描述状态和 `profile_revision`

#### Scenario: 读取单台设备资源

- **WHEN** MCP 客户端读取 `iot://devices/dev-a`
- **THEN** 服务端返回 `dev-a` 的 Device Tree、在线状态和缓存新鲜度信息

#### Scenario: 未知设备资源

- **WHEN** MCP 客户端读取不存在的 `iot://devices/dev-x`
- **THEN** 服务端返回资源不存在的错误，不创建空资源

### Requirement: 基础工具

服务端必须（SHALL）提供以下基础 MCP 工具：`list_devices` 返回未过期设备摘要列表；`get_device_tree` 接收 `device_id` 返回完整 Device Tree；`list_protocols` 接收 `device_id` 返回节点、Protocol UUID、Interface 和函数签名摘要；`invoke_protocol` 接收 `device_id`、`node_id`、`protocol_uuid`、`interface`、`function` 和可选 `args` 执行通用协议调用；`refresh_device_profile` 接收 `device_id` 强制重新获取设备描述。工具参数必须经过校验，未知或不存在的 `device_id` 必须返回结构化错误，未声明的协议函数必须返回 `unsupported_operation`。

#### Scenario: 列出设备

- **WHEN** 大模型调用 `list_devices`
- **THEN** 服务端返回所有未过期已知设备的 ID、型号、连接状态、connection_confirmed、描述状态和可调用函数数量；只有连接已确认且描述 fresh 才计为可调用

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

设备连接已确认且成功建立 fresh Protocol 索引后，服务端必须（SHALL）在发布预算内为 Protocol Interface 函数生成 MCP 工具。默认每设备最多 64 个动态工具、全局最多 256 个（均可配置）；超额函数仍可通过 `list_protocols` 查看并通过同等校验的 `invoke_protocol` 调用。预算内选择按完整寻址元组排序，不依赖设备发现顺序，列表资源必须注明总函数数、已发布数和截断情况。工具名由设备 ID、节点 ID、Protocol UUID、Interface 和函数名确定性生成，只包含小写字母、数字和下划线，总长度不超过 64 字符，并保证全局唯一。所有工具名均附加完整寻址元组的 SHA-256 前 16 位十六进制摘要；可读前缀最多 45 字符，再加 `__` 与摘要。摘要输入为五个字符串的紧凑 UTF-8 JSON 数组，不能用可能歧义的字符串拼接。极低概率摘要碰撞时拒绝发布碰撞项并报警，两项仍能通过通用工具访问，不能覆盖已有路由。设备离线、broker 断连或描述失效时，对应工具必须移除。

#### Scenario: 上线后 OLED 工具出现

- **WHEN** 设备 `dev-a` 上线且其 OLED 节点声明 `display_text` 函数
- **THEN** `tools/list` 中出现由设备、OLED 节点、Protocol、Interface 和 `display_text` 生成的工具，且参数与 Device Tree 声明一致

#### Scenario: 嵌套节点工具使用子节点 ID

- **WHEN** OLED 节点是 I2C 节点的子节点且声明 `display_text`
- **THEN** 生成的工具以 OLED 节点的 `node_id` 寻址，而不是 I2C 父节点

#### Scenario: 离线后工具移除

- **WHEN** 设备 `dev-a` 变为 `offline`
- **THEN** 服务端从 `tools/list` 中移除 `dev-a` 的所有动态工具，其资源仍保留并标记为离线

#### Scenario: 可读名称规范化冲突

- **WHEN** 两个不同寻址元组的可读前缀经规范化或截断后相同，但摘要不同
- **THEN** 两项各自使用包含完整寻址元组摘要的名称，不因注册顺序改变名字；重启后名称保持一致

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

对标记为 `stale` 的描述，服务端必须（SHALL）继续在资源中展示缓存并明确标注 stale，但不在 tools/list 发布其动态工具。`invoke_protocol` 或客户端缓存的动态工具调用必须先检查 broker 连通性和设备状态：明确离线则返回 `device_offline`；broker 不可用返回 `broker_unavailable`；其余情况先合并执行 `refresh_device_profile`，刷新成功后按新索引重新校验再执行，失败返回 `device_profile_stale`。刷新后函数已移除则返回 `unsupported_operation`。调用方应优先使用通用工具处理失效动态工具名称；未知名称可按 MCP 协议返回工具不存在错误。刷新最多 15 秒，从本次工具调用开始计时，包含等 RPC 槽、退避、合并等待、describe 及校验提交；随后 invoke 最多 5 秒，总预算 20 秒（不含客户端与模型推理耗时）。不能在等待槽后重新获得完整 15 秒：剩余时间不足时减少尝试或截短等待；预算耗尽返回 device_profile_stale 且不启动 invoke，独立 refresh_device_profile 超时返回 timeout。若刷新后调用槽已被其他操作占用，invoke 返回 busy，不进入无界等待。客户端截止时间更短则服从客户端取消且不再为其启动新操作；共享后台刷新仅可因其他有效等待方或独立自动刷新需求继续，一个调用超时/取消不得重置或取消其他等待方的有效预算。描述 invalid 的异常连接不做同连接刷新恢复，返回 device_profile_stale，等待新 connection_id 确认。

#### Scenario: stale 设备刷新后调用

- **WHEN** 大模型调用 `stale` 设备 `dev-a` 的工具，且刷新描述成功
- **THEN** 服务端使用刷新后的 Device Tree 校验参数并完成工具调用

#### Scenario: stale 设备刷新失败

- **WHEN** 大模型调用 `stale` 设备 `dev-a` 的工具，但设备描述刷新超时
- **THEN** 服务端返回 `device_profile_stale` 错误，不执行设备操作

#### Scenario: 等待 RPC 槽计入刷新预算

- **WHEN** 某次 stale 工具调用在刷新阶段已等槽 4 秒，随后才开始 describe
- **THEN** 该调用刷新剩余预算至多 11 秒，不重新获得 15 秒；总刷新截止仍为调用开始后 15 秒，未成功则返回 device_profile_stale 且不启动 invoke

#### Scenario: 同连接异常不可手动恢复

- **WHEN** 当前 connection_id 已因矛盾版本被标为异常，客户端请求刷新或调用其函数
- **THEN** 服务端返回 device_profile_stale，不通过同连接 describe 或周期 online 重新发布工具，等待新 connection_id 确认

### Requirement: 错误、通知和声明内容的信任边界

工具执行失败必须（SHALL）返回 MCP `CallToolResult`，设置 `isError: true`，并以 JSON 文本提供稳定的 `code`、`message`、可用时的 device_id 和 request_id；不能把业务失败当作正常成功文本。JSON-RPC 的非法方法、无效协议参数和未知工具使用协议级错误。至少覆盖 `device_not_found`、`device_offline`、`broker_unavailable`、`device_profile_stale`、`unsupported_operation`、`invalid_params`、`invalid_device_response`、`timeout`、`outcome_unknown`、`busy`、`payload_too_large` 和合法设备错误。设备响应包含不可信文本，保留错误码/说明的语义但限制大小并安全编码，不渲染为可执行内容。执行后返回的 execution_failed、invalid_device_response 等错误同样不保证没有产生硬件副作用；客户端不得把任意 isError 当作可自动重试控制操作的信号。

工具集合或设备资源列表变化时必须向已协商相应能力的活动会话发送 list_changed；通知可以合并，但重新查询必须得到当前一致快照。缓存内容变化不等于资源列表变化；MVP 不声明 resources.subscribe，客户端通过重新读取获取最新资源。设备声明的描述不能成为系统提示或权限来源，默认标记工具有潜在副作用；LLM 客户端确认策略是集成验收的一部分，不能声称 MCP annotations 提供访问控制。MVP 仅在低风险外设演示，不纳入门锁、加热器或市电负载。

#### Scenario: 调用结果不确定

- **WHEN** 硬件操作可能已执行但响应丢失
- **THEN** MCP 返回 isError 的 `outcome_unknown` 结果，说明不得自动重试，不将其伪装成“设备操作失败”

#### Scenario: 发布预算耗尽

- **WHEN** 设备合法声明 80 个函数且每设备工具预算为 64
- **THEN** 发布确定性的 64 个动态工具，list_protocols 仍返回全部函数并说明截断，通用调用仍受同样的安全和参数校验

#### Scenario: 浏览器来源不可信

- **WHEN** `/mcp` 请求携带不在允许列表中的 Origin
- **THEN** 服务端拒绝该 HTTP 请求，不启动 MCP 工具调用
