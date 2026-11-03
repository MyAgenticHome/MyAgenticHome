# Spec Delta

## Purpose

定义设备自描述的 Device Tree 文档格式，使服务端和大模型能够发现设备上的 Protocol、Interface、函数签名以及 Protocol 之间的嵌套关系，并通过统一寻址调用这些函数。

## ADDED Requirements

### Requirement: Device Tree 文档结构

Device Tree 必须（SHALL）是一个 JSON 对象，包含 `schema_version`、`device` 和 `root` 三个顶层字段。`device` 必须包含稳定的 `id`、`model`、`firmware_version` 和单调递增的 `profile_revision`；`root` 是描述整个设备的根节点。根节点及每个子节点必须包含 `id`、`type`、`name`，并可包含 `properties`、`protocols` 和 `children`。MVP 的 `schema_version` 为 `"1.0"`。

#### Scenario: 最小合法文档

- **WHEN** 设备返回仅包含根节点、且根节点包含 id、type、name 的 Device Tree
- **THEN** 服务端接受该文档，并把该设备识别为没有可操作 Protocol 的设备

#### Scenario: 缺少必需字段

- **WHEN** Device Tree 缺少 `schema_version`、`device.id` 或根节点 `id`
- **THEN** 服务端拒绝该文档，不更新设备能力，并记录校验错误

### Requirement: 节点标识与树状关系

同一 Device Tree 内所有节点的 `id` 必须（SHALL）唯一且在同一次固件运行期间保持稳定。节点代表一个设备实例或 Protocol 挂载点，总线、外设和嵌套 Protocol 之间的关系必须通过父子关系表达：父节点暴露总线 Protocol，子节点暴露挂载其上的驱动 Protocol，并在子节点 `properties` 中声明地址、引脚或接口等连接信息。

#### Scenario: I2C 上的 OLED 驱动 Protocol

- **WHEN** 设备通过 I2C 总线连接一个 OLED 屏幕
- **THEN** Device Tree 中包含一个暴露 I2C Protocol 的父节点，OLED 节点是其子节点，OLED 节点暴露自己的驱动 Protocol，并在属性中包含 I2C 地址

#### Scenario: 重复节点 ID

- **WHEN** 同一 Device Tree 中两个节点使用相同的 `id`
- **THEN** 服务端判定文档非法并拒绝注册该设备能力

### Requirement: Protocol 与 Interface 声明

节点可以（MAY）通过 `protocols` 数组声明一个或多个 Protocol。每个 Protocol 必须包含 `uuid`、`name` 和 `interfaces`；每个 Interface 必须包含 `name` 和 `functions`；每个函数必须包含 `name`、`description`，并可选包含 `parameters` 和 `result`。`parameters` 使用 JSON Schema 子集描述参数名称、类型和是否必填。UUID 标识 Protocol 类型，节点 ID 标识具体实例；同一 UUID 可以出现在多个节点上。服务端必须（SHALL）根据声明生成工具，而不是依赖内置的 UUID 语义。

#### Scenario: OLED 节点声明驱动 Protocol

- **WHEN** OLED 节点声明 UUID 为 `0f5d2f6e-...` 的 Protocol，并在 Interface 中声明 `init` 和 `display_text` 函数
- **THEN** 服务端为该实例生成对应的动态工具，并在工具描述中包含节点路径和函数说明

#### Scenario: 一个节点暴露多个 Protocol

- **WHEN** 同一节点同时声明传感器 Protocol 和电源管理 Protocol
- **THEN** 服务端分别索引两个 Protocol，并生成两组函数工具

#### Scenario: Protocol 缺少 UUID

- **WHEN** `protocols` 中的某项缺少 `uuid` 字段
- **THEN** 服务端拒绝该 Device Tree，并报告缺失字段位置

### Requirement: 协议实例寻址与名称唯一性

协议调用必须（SHALL）使用 `node_id + protocol_uuid + interface + function` 唯一寻址。同一节点内 Protocol UUID 必须唯一；同一 Protocol 内 Interface 名称必须唯一；同一 Interface 内函数名称必须唯一。节点 ID 必须在设备运行期间稳定，服务端不得仅凭 Protocol UUID 调用实例。

#### Scenario: 同一 UUID 的多个实例

- **WHEN** 设备有两个 OLED 节点，且都暴露同一个 OLED Protocol UUID
- **THEN** 服务端分别使用两个节点的 ID 寻址，调用不会串到另一个实例

#### Scenario: 函数名重复

- **WHEN** 同一 Interface 中声明两个同名函数
- **THEN** 服务端判定文档非法并拒绝注册该设备能力

### Requirement: 上报时机与版本

设备必须（SHALL）在收到 `device.describe` 请求时返回完整 Device Tree。树内容在一次启动周期内必须保持稳定；当板级配置、Protocol 集合或 Interface 函数变化时，设备必须递增 `profile_revision`。服务端发现 `profile_revision` 变化后必须重新获取完整树并重建 Protocol 索引。

#### Scenario: 首次请求返回完整树

- **WHEN** 服务端向设备发送 `device.describe` 请求
- **THEN** 设备在响应中包含完整的 Device Tree、Protocol 声明和当前 `profile_revision`

#### Scenario: Protocol 变化后递增版本

- **WHEN** 固件新增一个 OLED Protocol 并重新构建烧录
- **THEN** 新固件上报的 `profile_revision` 大于旧固件上报的值

### Requirement: 文档校验与限制

服务端必须（SHALL）在注册设备能力前校验 Device Tree：文档不超过 64 KiB、节点数不超过 256、树深度不超过 8、单节点 Protocol 不超过 32 个、单 Protocol Interface 不超过 16 个、单 Interface 函数不超过 64 个。未知节点类型和未知字段必须被保留并原样暴露，但只有包含完整 Protocol/Interface/函数声明的条目才生成可调用工具。

#### Scenario: 超限文档被拒绝

- **WHEN** 设备返回的 Device Tree 超过 64 KiB、节点数超过 256 或深度超过 8
- **THEN** 服务端拒绝该文档，保留设备在线状态但不注册其协议能力

#### Scenario: 未知字段被保留

- **WHEN** Device Tree 包含服务端不认识的节点属性或 Protocol 扩展字段
- **THEN** 服务端保留这些字段并在设备树资源中展示，不影响合法函数的工具生成

### Requirement: Protocol UUID 与安全校验

Protocol 的 `uuid` 必须（SHALL）使用 RFC 4122 小写带连字符格式。服务端必须拒绝格式非法的 UUID、同一节点内重复的 Protocol UUID，以及含密码、密钥或令牌的节点属性。服务端不要求 UUID 对应已知类型；未知 UUID 按声明正常索引和调用。

#### Scenario: 非法 UUID 被拒绝

- **WHEN** Protocol 的 `uuid` 不是 RFC 4122 格式
- **THEN** 服务端拒绝该 Device Tree，并报告具体 Protocol 路径

#### Scenario: 文档包含敏感信息

- **WHEN** 服务端在校验时发现节点属性包含密码或密钥字段
- **THEN** 服务端拒绝该文档并记录安全告警，不把内容暴露给 MCP 客户端
