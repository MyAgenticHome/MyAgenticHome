# Spec Delta

## Purpose

定义局域网 mosquitto MQTT v5 broker 的部署行为，以及服务端与实现公共固件契约的设备（首期 BL616CL、BL618）之间可关联、可超时、可报错的 Request-Reply 和通用协议调用消息契约。

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

服务端发起请求后必须（SHALL）等待响应，单次等待上限 5 秒。`device.describe` 最多 3 次总尝试（首次 + 2 次重试，总截止时间 15 秒），同一逻辑请求的所有尝试必须复用同一请求 ID、Correlation Data 和完全相同的负载。`protocol.invoke` MVP 不进行应用层自动重试，包括声明为幂等的函数；MQTT QoS 1 的同一报文重投递仍必须由设备去重。调用已经交给传输层但没有有效响应时，返回 `outcome_unknown`，明确可能已经执行，调用者不得把它解释为失败并自动重试；describe 耗尽尝试返回 `timeout`。调用尚未提交即发现 broker 不可用时返回 `broker_unavailable`。取消、断连及超时均清理 pending，不排队跨连接补发操作。

#### Scenario: 设备及时响应

- **WHEN** 服务端在 5 秒内收到 Correlation Data 匹配的响应
- **THEN** 服务端取消超时计时器并把响应交给调用方，不再重试

#### Scenario: 设备无响应

- **WHEN** `device.describe` 的 3 次总尝试均在 5 秒内未收到有效响应
- **THEN** 服务端返回 `timeout` 错误，错误信息包含设备 ID 和被调用的方法名

#### Scenario: 操作已执行但响应丢失

- **WHEN** `protocol.invoke` 已交给传输层，但 5 秒内未收到有效响应
- **THEN** 服务端返回 `outcome_unknown`，不生成新 ID 重试，不宣称操作未执行

### Requirement: 统一的消息与错误语义

请求和响应负载必须（SHALL）使用 JSON 对象。请求包含 `id`、`connection_id`、`method` 和可选 `params`；响应包含 `id`、`connection_id`、布尔 `ok`，成功时必须有 `result` 且不得有 `error`，失败时必须有 `error`（含字符串 `code` 和 `message`）且不得有 `result`。`device.describe` 省略 params 或使用空对象，成功 result 直接为 Device Tree 对象，不额外包裹 device_tree 字段；`protocol.invoke` 的成功 result 为 Handler 的 JSON 返回值。设备对未知方法、非法参数和内部执行失败必须返回错误响应而不是静默丢弃。

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

MVP 所有请求、响应和状态消息必须（SHALL）使用 QoS 1。req/resp 必须 `retain=false`，禁止设备执行 retained 请求，禁止服务端接受 retained 响应。设备必须按 `(connection_id, id)` 对重复投递去重：首个请求先占位为 in-flight，完成后缓存结果至少 30 秒；相同 ID 不同负载返回 `request_id_conflict`，相同 ID 正在执行时不得再次执行。30 秒从完成时计起，未过期记录不可为了接收新请求而驱逐；容量不足返回 `busy`，在预留结果缓存之前不得执行 Handler。MVP 每设备最多一个在途 RPC（describe 也占用此槽），重试不占新槽，去重缓存至少 8 项；服务端遵守同一限制，直接 invoke 无空闲槽则返回 busy；注册表可合并一个待刷新意图并等槽释放后构造 describe，这不占用第二个在途槽、不保存待补发的 MQTT 操作，也不允许跨连接回放；调度与等待预算见 device-registry/MCP 规范。该保证只覆盖当前连接与缓存窗口，不声称跨重启 exactly-once。服务端必须忽略无法匹配到待处理请求的响应。

#### Scenario: 重复请求不重复执行

- **WHEN** 设备在 30 秒内两次收到请求 ID 相同的请求
- **THEN** 设备只执行一次，并对第二次投递返回第一次的结果

#### Scenario: 迟到响应被忽略

- **WHEN** 服务端收到 Correlation Data 已过期或无法匹配的响应
- **THEN** 服务端丢弃该响应，不影响其他待处理请求

### Requirement: 设备在线状态主题

设备连接 broker 时必须（SHALL）设置 Last Will 为状态主题的 retained `offline` 消息；收到请求主题 SUBACK 成功后才发布 retained `online`。设备正常断开前必须主动发布 offline 并等待 PUBACK 或至多 1 秒后关闭。状态消息使用 QoS 1，最大 1024 字节，其 JSON 对象必须包含 `v: 1`、`device_id`、`state`（online/offline）、`model`、`firmware_version`、`profile_revision` 和 `connection_id`。`connection_id` 是每次 MQTT CONNECT 前生成的新 UUID，同次连接的 online、offline 和 Will 必须一致；同 connection_id 的 model、firmware_version 和 profile_revision 固定且与 describe 一致，版本/板级配置变化须通过新启动和新 connection_id 生效；device_id 必须与主题一致，model/firmware_version 为 1–64 字节，profile_revision 与 device-profile 的整数范围一致。字段非法、主题不匹配或状态 payload 为空时不得创建设备或确认在线；空 retained payload 仅视为状态记录被清除，使已登记设备不可调用，不删除其缓存。不依赖设备墙上时钟。MVP keepalive 为 60 秒、Will Delay 为 0、clean_start 为 true、Session Expiry 为 0。retained online 只是历史线索，不能作为存活证明，详见 device-registry。

#### Scenario: 上线上报状态

- **WHEN** 设备成功连接 broker 并完成订阅
- **THEN** 设备向状态主题发布 retained 的 `online` 消息

#### Scenario: 异常掉线由 Last Will 标记

- **WHEN** 设备与 broker 的连接异常断开且未主动发布离线消息
- **THEN** broker 向状态主题发布设备预先注册的 retained `offline` 消息

### Requirement: 在线状态周期重新公告

设备必须（SHALL）在请求主题订阅成功后首次发布 retained online，此后在同一健康 MQTT 连接中每 30 秒加 [0,5] 秒非负抖动重新发布一次 QoS 1 retained online；connection_id、model、firmware_version 和 profile_revision 与该连接首次公告保持一致，不增加新 RPC 方法或额外 topic。周期从上次 online 提交发送时计起，不因普通 RPC 流量或续租而无限推迟。每次 online 的 PUBACK 最多等待 5 秒，失败/超时进入有界断连重连，不累计无界公告队列。网络任务负责公告，Handler 不得阻塞它。

主动下线、租约到期、网络断开或进入重连前必须停止旧连接公告定时器，丢弃旧连接代次的回调，不得在 offline 后再向旧连接提交 online。状态清除只移除存活线索，不代表永久撤销或禁止设备；健康设备无需重连，在后续周期公告后由服务端发起新的 describe 确认。正常 fresh 设备的相同重复公告不重复触发 describe，也不以公告本身代替连接确认。MVP 不定义基于删除 retained status 的永久禁用功能。

#### Scenario: 状态清除后不重连恢复

- **WHEN** 健康连接的 retained status 被清除，设备保持 MQTT 连接
- **THEN** 在无发送故障时设备至多 35 秒内重新发送相同 connection_id 的 online；服务端收到后按新刷新代次确认，只有 describe 校验提交成功才恢复工具

#### Scenario: 周期公告不重复刷新

- **WHEN** 已确认且描述 fresh 的设备再次发布相同身份与版本的 online
- **THEN** 服务端只更新公告观测信息，不重复拉取描述、不撤下工具

#### Scenario: 下线取消旧公告

- **WHEN** 主动下线与周期公告定时器同时到期
- **THEN** 下线状态优先，旧定时器不得在 offline 之后提交 online；新连接仅在其成功 SUBACK 后重新公告

### Requirement: 关联身份与报文边界

请求 `id` 必须（SHALL）为小写带连字符 UUID，Correlation Data 必须为同一 UUID 的 36 字节 UTF-8 文本（不是 16 字节二进制）。请求和响应 JSON 都必须包含 `connection_id`；服务端从已收到的 status 中取得该值，设备必须拒绝非当前连接的请求且不执行 Handler。设备只接受精确等于自身 `iot/v1/devices/{device_id}/resp` 的 Response Topic；缺少关联属性、属性重复或 payload id 不匹配时丢弃并记录有界诊断，不向任意请求指定地址发送数据。服务端先匹配响应主题设备 ID、Correlation Data、JSON id、connection_id，再校验信封和结果。无法可靠关联、retained、旧连接或无法解析合法信封的响应丢弃，仍在原等待窗口内等待合法响应；能够可靠关联且信封合法（id/connection_id/ok/result/error 满足约束）的响应，其 result 不符合 pending 方法的结果结构或已声明 schema 时，必须立即结束请求并返回 invalid_device_response，不继续等待到 timeout/outcome_unknown。合法错误信封按设备错误返回。invalid_device_response 不保证没有硬件副作用，不得自动重试 invoke。

请求 JSON 最大 4096 字节；`protocol.invoke` 响应 JSON 最大 4096 字节；`device.describe` 内部 Device Tree 最大 65536 字节，含信封后的响应 JSON 最大 69632 字节；完整 MQTT 控制报文上限 73728 字节。这些都是 UTF-8 序列化字节数，不是字符数。固件通过 CONNECT Maximum Packet Size 宣告上限；服务端与设备在发送前检查对端限额，无法容纳时返回 `payload_too_large`（响应本身可安全发送时），否则关闭连接并记录原因。broker 的 payload 上限 131072 字节不代表固件需要一次性分配同等大小缓冲。

#### Scenario: 响应主题错误

- **WHEN** 请求要求设备向其他设备的 resp 或任意 status 主题响应
- **THEN** 设备不执行操作且不向该主题发布消息

#### Scenario: 连接过期请求

- **WHEN** 设备重连后收到前一次 connection_id 的请求
- **THEN** 设备拒绝请求，不执行旧操作

#### Scenario: 64 KiB 描述含 RPC 信封

- **WHEN** Device Tree 序列化为 65536 字节且完整响应不超过 69632 字节
- **THEN** 传输层能够完整发送和接收，不因仅为树预留缓冲而截断响应


#### Scenario: 已关联响应的结果类型错误

- **WHEN** invoke 的非 retained 响应身份和信封合法，但声明为 integer 的 result 实际为字符串
- **THEN** 服务端立即返回 invalid_device_response、清理 pending 且不重试，而不是等待到 outcome_unknown

### Requirement: 非排队会话与 retained 控制隔离

服务端和设备必须（SHALL）使用 clean_start=true、Session Expiry=0，不通过持久会话、离线发布队列或重连补发恢复旧 RPC。设备 req 订阅及服务端 resp 订阅必须设置 QoS 1、Retain As Published=1、Retain Handling=2：既不接收订阅前存储的 retained 消息，也能识别并拒绝实时发布的 retain=true 请求/响应。服务端 status 订阅必须设置 QoS 1、Retain As Published=1、Retain Handling=0，接收历史 retained 状态作为待确认线索，并保留实时空 retained status 的 RETAIN 标记以识别清除事件。必须验证所选客户端实际发出的订阅选项与离线发送行为，不能仅靠 payload 中的字段代替 MQTT 属性检查。

#### Scenario: 实时 retained 请求不能触发外设

- **WHEN** 设备已经订阅 req，某客户端向该主题实时发布 retain=true 的合法格式请求
- **THEN** broker 向设备保留 RETAIN 标记，设备拒绝执行；设备重新订阅时也不接收存储的该请求

#### Scenario: 重连不补发控制操作

- **WHEN** 服务端在 invoke 发送期间断开 broker，随后恢复连接
- **THEN** 原调用按提交情况返回 broker_unavailable 或 outcome_unknown，新连接不补发该请求

#### Scenario: 实时清除设备状态

- **WHEN** 服务端已经订阅 status，broker 转发空 payload 的 retained 状态清除消息
- **THEN** 服务端收到 RETAIN=1，使该设备不可调用、撤下动态工具并保留缓存；不能凭清除前的在途 describe 恢复能力，须等待新的合法 online 后重新确认

### Requirement: JSON 跨语言解析一致性

发现报文、状态、RPC 及 Device Tree 必须（SHALL）使用严格 UTF-8 JSON；拒绝重复对象键、非法 UTF-8、非有限数、超出范围的数值和 JSON 文档后的非空白内容，不允许 Go 与 cJSON 采用不同键值。所有对象键及字符串值禁止解码后的 U+0000（包括 JSON 转义 \u0000），必须在转换为 cJSON 的 NUL 结尾字符串之前检查；仅扫描原始文本中是否含有字面 \u0000 不够，必须区分转义反斜杠。由字符反斜杠、u、0、0、0、0 组成的普通字符串允许使用。完整 JSON 容器嵌套深度最多 64（最外层对象为 1），在递归解析前或解析时检查；这与 Device Tree 节点深度 8、schema 深度 8 是不同限制。缺少可安全关联身份的请求直接丢弃并采样记录原因；已具备合法身份且 params 非法时返回 invalid_params。未知普通元数据可忽略或按 device-profile 保留，但不得绕过 schema 关键字与敏感键校验。

#### Scenario: 重复寻址字段

- **WHEN** 请求 params 对象重复声明 node_id 且两次值不同
- **THEN** 设备不执行任何 Handler，服务端与固件的共享反例测试都拒绝该报文

#### Scenario: JSON 递归深度超限

- **WHEN** 未知扩展字段内嵌超过 64 层 JSON 容器
- **THEN** 接收方有界拒绝，不因保留未知字段而耗尽栈

#### Scenario: NUL 与字面反斜杠区分

- **WHEN** 对象键或字符串值含转义后 U+0000，或仅含表示字面反斜杠的合法转义文本
- **THEN** Go 与固件对前者一致拒绝且不执行 Handler，对不含 U+0000 的后者不因其外观类似转义序列而拒绝
