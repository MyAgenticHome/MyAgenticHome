# Spec Delta

## Purpose

定义设备自描述的 Device Tree 文档格式，使服务端和大模型能够发现设备上的 Protocol、Interface、函数签名以及 Protocol 之间的嵌套关系，并通过统一寻址调用这些函数。

## ADDED Requirements

### Requirement: Device Tree 文档结构

Device Tree 必须（SHALL）是一个 JSON 对象，包含 `schema_version`、`device` 和 `root` 三个顶层字段。`device` 必须包含稳定的 `id`、`model`、`firmware_version` 和非负整数 `profile_revision`（不超过 2147483647）；`root` 是描述整个设备的根节点。根节点及每个子节点必须包含 `id`、`type`、`name`，并可包含 `properties`、`protocols` 和 `children`。MVP 的 `schema_version` 为 `"1.0"`；文中的“v0”仅表示首期产品范围，不是另一种线上版本。服务端必须拒绝未知 schema 主版本，对相同主版本的未知元数据字段保持兼容。`device.id` 必须与本次 MQTT 请求的目标设备及响应主题一致，不能用响应内容改写其他设备记录。

#### Scenario: 最小合法文档

- **WHEN** 设备返回仅包含根节点、且根节点包含 id、type、name 的 Device Tree
- **THEN** 服务端接受该文档，并把该设备识别为没有可操作 Protocol 的设备

#### Scenario: 缺少必需字段

- **WHEN** Device Tree 缺少 `schema_version`、`device.id` 或根节点 `id`
- **THEN** 服务端拒绝该文档，不更新设备能力，并记录校验错误

### Requirement: 节点标识与树状关系

同一 Device Tree 内所有节点的 `id` 必须（SHALL）唯一；相同板级实例在重启和兼容固件升级后必须保持稳定。节点 ID 匹配 `[A-Za-z0-9][A-Za-z0-9_.-]{0,63}`，Interface/函数名匹配 `[A-Za-z_][A-Za-z0-9_]{0,63}`，Protocol UUID 使用规范化小写格式。节点代表一个设备实例或 Protocol 挂载点，总线、外设和嵌套 Protocol 之间的关系必须通过父子关系表达：父节点暴露总线 Protocol，子节点暴露挂载其上的驱动 Protocol，并在子节点 `properties` 中声明地址、引脚或接口等连接信息。

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

设备必须（SHALL）在收到 `device.describe` 请求时返回完整 Device Tree。树内容在一次启动周期内必须保持稳定；同一 connection_id 的 model、firmware_version、profile_revision 与完整描述不得变化，describe 中这些字段必须与该连接首次接受的 online 基线一致。板级配置、Protocol 集合或 Interface 函数变更通过新固件/配置启动生效，正常发布递增 profile_revision，并以新的 connection_id 公告；服务端对新连接重新获取完整树并重建索引，含显式回滚。MVP 不支持同连接热更新描述，收到矛盾版本按 device-registry 的异常连接规则处理，不将其解释为正常刷新。

#### Scenario: 首次请求返回完整树

- **WHEN** 服务端向设备发送 `device.describe` 请求
- **THEN** 设备在响应中包含完整的 Device Tree、Protocol 声明和当前 `profile_revision`

#### Scenario: Protocol 变化后递增版本

- **WHEN** 固件新增一个 OLED Protocol 并重新构建烧录
- **THEN** 新固件上报的 `profile_revision` 大于旧固件上报的值

### Requirement: 文档校验与限制

服务端必须（SHALL）在注册设备能力前校验 Device Tree：文档不超过 64 KiB、节点数不超过 256、树深度不超过 8（根节点为第 1 层）、单节点 Protocol 不超过 32 个、单 Protocol Interface 不超过 16 个、单 Interface 函数不超过 64 个。未知节点类型和未知字段必须被保留并原样暴露，但只有包含完整 Protocol/Interface/函数声明的条目才生成可调用工具。

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

### Requirement: 参数与返回值的可执行 Schema 子集

函数 `parameters` 必须（SHALL）使用 JSON Schema 2020-12 的有界子集，根为 `type: object`；省略时等价于无参数对象（`properties: {}`、`additionalProperties: false`）。`args` 省略时等价于 `{}`。支持 `type`（object、array、string、integer、number、boolean）、`properties`、`required`、`additionalProperties: false`、`items`、`enum`、`minimum`、`maximum`、`minLength`、`maxLength`、`minItems`、`maxItems`、`description`。所有对象必须禁止额外属性，数组必须声明 `items` 和 `maxItems`，字符串必须声明 `maxLength`；schema 嵌套深度最多 8（根 schema 为 1，进入 properties 的子 schema 或 items 加 1，不按 JSON 对象层数计）。每个 schema 必须明确单一 type，不接受 type 数组；约束关键字的类型和值必须合法，min 不大于 max，enum 为非空且无重复的同类型值数组。minLength/maxLength 按 Unicode 码点数计算，而不是 UTF-8 字节数；消息大小上限始终按序列化 UTF-8 字节计算。整数限制为绝对值不超过 9007199254740991，数值必须有限，不进行字符串到数字等隐式转换。`required` 只能引用已声明属性。未知校验关键字、`$ref`、组合 schema 和远程引用必须拒绝，不得静默忽略约束；此规则不同于 Device Tree 元数据未知字段的保留规则。

可选 `result` 使用相同子集，但根类型不限于 object；声明时设备和服务端必须验证结果，服务端收到不符合声明的结果返回 `invalid_device_response`。设备在 Handler 已产生副作用后发现结果非法，必须清理结果、缓存并返回 execution_failed，不得重新执行 Handler；该错误不代表硬件状态已回滚。未声明 `result` 时仅接受有界合法 JSON。函数可声明 `read_only` 和 `idempotent` 布尔元数据，默认均为 false；它们不构成鉴权，也不解除 MQTT 层对有副作用操作的重试限制。设备和服务端必须共享正反例契约测试，不能各自定义不同的参数规则。

#### Scenario: 两端一致拒绝非法参数

- **WHEN** `display_text` 要求长度不超过 128 的字符串 `text`，请求缺少 text、传数字、超长字符串或携带未声明参数
- **THEN** 服务端拒绝并返回 `invalid_params`；绕过服务端直接调用设备时设备也拒绝，Handler 不执行

#### Scenario: 不支持的校验关键字

- **WHEN** 函数 parameters 包含 `$ref` 或 `oneOf`
- **THEN** 服务端拒绝该描述并报告 schema 路径，而不是生成看似受约束但实际上未校验的工具

### Requirement: 描述身份、版本与敏感信息边界

服务端必须（SHALL）把 `profile_revision` 作为变化标识而非只接受更大的值：正常发布递增，但显式回滚到较低 revision 时也必须刷新。新连接的固件版本变化、设备重新连接和服务端恢复连接后均须重新确认描述，不能仅凭相同 revision 信任旧缓存；同一 connection_id 的版本或完整描述变化属于异常，不能借本条触发正常热刷新。描述中的名称、说明、属性和未知扩展都是不可信数据，不得作为系统指令或授权依据。

固件不得在任何描述字段中输出凭据。服务端对整个文档递归检测敏感键：键名转小写并移除 `-`、`_` 后，拒绝 `password`、`passwd`、`secret`、`token`、`apikey`、`privatekey`、`wifipassword`、`mqttpassword`；日志只记录字段路径和原因，不记录原值或完整文档。该检查只能防止常见误泄露，不保证识别任意文本中藏匿的秘密。

#### Scenario: 响应冒用另一台设备

- **WHEN** 对 `dev-a` 发起的 describe 返回 `device.id: dev-b`
- **THEN** 服务端拒绝响应，不更新任一设备的描述或索引

#### Scenario: 固件回滚

- **WHEN** 设备从 revision 4 回滚到 revision 3 并重新上线
- **THEN** 服务端重新获取并验证 revision 3 的完整描述，成功后原子替换缓存

#### Scenario: 扩展字段泄露凭据

- **WHEN** 任意嵌套扩展对象包含 `api_key` 字段
- **THEN** 服务端拒绝描述，日志不含该字段的值

### Requirement: 可选硬件元数据与芯片无关寻址

Device Tree 的 device 可以（MAY）包含 hardware 对象；出现时必须包含 vendor、chip、board、firmware_target，可选 board_revision，各已知字段为 1–64 字节字符串。未知 hardware 扩展字段按普通元数据保留并执行全树敏感键检查；hardware 缺失不影响既有 schema_version 1.0 描述合法性。芯片固件 capability 可进一步要求其自身必须生成 hardware，但服务端不得据此拒绝其他兼容旧设备。chip 表示真实硬件而不是 SDK CHIP 别名；硬件元数据不得参与隐式授权、能力推断或协议寻址。

#### Scenario: BL618 SDK 别名不影响协议

- **WHEN** Device Tree hardware.chip 为 bl618，而固件构建使用 SDK CHIP=bl616
- **THEN** 服务端按实际硬件元数据显示 BL618，并仅按 device_id/node_id/Protocol/Interface/function 路由

#### Scenario: 兼容旧描述

- **WHEN** 合法 schema_version 1.0 描述未包含 hardware
- **THEN** 服务端仍接受其已声明能力，不强制升级协议或猜测芯片型号
