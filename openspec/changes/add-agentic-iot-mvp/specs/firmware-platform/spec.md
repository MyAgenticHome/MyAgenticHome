# Spec Delta

## Purpose

定义 BL616CL、BL618 以及后续芯片实现共同遵守的固件运行时和平台适配契约。发现、MQTT v5、Device Tree、RPC、去重及 Protocol 注册表只维护一份公共实现；芯片专属 capability 仅规定身份、SDK/板级映射与独立验收。本期交付 BL616CL 与 BL618 两个目标，未来其他芯片只预留接入边界，不宣称已经支持。

## ADDED Requirements

### Requirement: 跨芯片稳定设备身份

平台适配必须（SHALL）提供稳定唯一硬件身份，芯片 capability 规定确定性的 device_id 派生规则；ID 满足 device-discovery 的语法和 64 字符上限，作为 MQTT client ID 与主题路径。不得从 DHCP 地址、随机启动值、sdk_chip 编译宏或可随升级变化的 BOARD 名派生身份；factory_reset 不清除身份来源。缺少可靠唯一身份时必须明确报错、禁止上线，不生成所有设备共用的占位 ID。服务端按整个 device_id 区分设备，不解析芯片前缀决定协议行为。

#### Scenario: SDK 参数不改变设备身份

- **WHEN** 同一实板升级兼容固件或切换等价的 SDK 板级配置，唯一硬件标识未变化
- **THEN** device_id 保持不变，注册表不创建新设备

#### Scenario: 无法取得唯一身份

- **WHEN** 平台无法读取约定的唯一硬件身份
- **THEN** 固件报告诊断并停止 MQTT 上线，不用全零或随机临时 ID 注册设备

### Requirement: WiFi 入网

固件必须（SHALL）使用预置在非易失存储或构建配置中的 WiFi SSID 和密码以 STA 模式连接网络。连接失败时必须按 1 秒、2 秒、5 秒、10 秒、30 秒退避重试，之后保持每 30 秒重试。MVP 不提供 AP 配网页面或蓝牙配网。

#### Scenario: 启动后自动入网

- **WHEN** 设备上电且已保存有效 WiFi 凭据
- **THEN** 设备在 30 秒内尝试连接 WiFi，并在成功后进入发现或 broker 连接流程

#### Scenario: WiFi 凭据无效

- **WHEN** 设备使用保存的凭据无法在 60 秒内连接 WiFi
- **THEN** 设备保持每 30 秒重试，并通过串口日志报告连接失败原因

### Requirement: 发现与 broker 配置应用

WiFi 连接成功后，固件必须（SHALL）按 `device-discovery` 规范执行发现，除非本地配置未到期且网络指纹一致；跨重启使用还必须有可信时钟证明剩余有效期，否则重新发现。首次取得有效配置后，固件必须保存 broker 地址、端口、凭据和有效期，并立即尝试连接 broker；已有租约的后台续租遵循 device-discovery，相同配置续租不重连，不能每收到 offer 就断开当前连接。

#### Scenario: 首次上电完成配网

- **WHEN** 设备首次上电、WiFi 连接成功且本地无 broker 配置
- **THEN** 设备广播发现请求，收到有效响应后保存配置并在 2 秒内发起 MQTT 连接

#### Scenario: 已有配置跳过发现

- **WHEN** 设备重启、有可信跨重启时钟证明配置仍有效，且网络指纹未变化
- **THEN** 设备不广播发现请求，直接连接已保存的 broker

### Requirement: MQTT v5 客户端行为

固件必须（SHALL）使用 MQTT v5 连接 broker，启用 Response Topic 和 Correlation Data 属性，keepalive 为 60 秒。连接前必须注册状态主题的 retained `offline` Last Will；连接成功后必须订阅自己的请求主题并发布 retained 的 `online` 状态。连接断开后必须按 1 秒到 60 秒的指数退避自动重连。按 mqtt-messaging 每 30–35 秒重新公告同一连接的 retained online，PUBACK 等待最多 5 秒；下线前停止公告并隔离旧连接回调。周期公告不改变连接 ID 或描述版本，也不代替服务端 describe 确认。

#### Scenario: 连接成功后订阅并上报在线

- **WHEN** 设备成功完成 MQTT v5 CONNECT
- **THEN** 设备按 mqtt-messaging 的订阅选项订阅 `iot/v1/devices/{device_id}/req`，收到成功 SUBACK 后才向状态主题发布 retained 的 `online`

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
- **THEN** 设备分发到对应 Handler，Handler 在 3 秒内完成或以有界驱动超时结束，并在服务端 5 秒等待窗口内返回响应；MVP 不接纳长时间阻塞操作

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

固件必须（SHALL）把 broker 配置保存到非易失存储，使用格式版本、CRC 和双槽/等价原子提交机制，不能把 CRC 当作掉电原子性保证。串口 `factory_reset` 经确认后清除 NVS WiFi 覆盖值和 broker 配置并重启；有可用且已确认引脚的板载按键时支持长按 5 秒，无此按键的板型只要求串口入口。构建期 WiFi 默认值不会被此命令擦除：存在默认值则回退使用，否则进入等待串口配置的状态，不得宣称能自动入网。必须提供本地串口设置 WiFi SSID/密码并持久化的方式，日志不回显密码。此恢复语义不是安全擦除设备中所有秘密。配置写入失败不得破坏已有有效副本；读取到损坏配置时必须视为无配置。

#### Scenario: 恢复出厂后重新配网

- **WHEN** 用户在串口输入 `factory_reset` 并确认，设备重启
- **THEN** 设备清除 NVS WiFi 覆盖值与 broker 配置；有构建期 WiFi 默认值则重新入网发现，否则等待串口重新配置，不陷入无凭据重试

#### Scenario: 配置损坏时自愈

- **WHEN** 设备启动时读取到校验失败的 broker 配置
- **THEN** 设备把配置视为不存在，WiFi 连接成功后重新执行发现流程

### Requirement: MQTT v5 子集的可互操作性

固件必须（SHALL）正确处理 TCP 分片、粘包、部分读写和关闭连接，不把一次 socket read 当作一帧 MQTT；验证 Remaining Length、属性长度、包标识符及所有长度边界。支持本项目使用的 Response Topic、Correlation Data、Maximum Packet Size、Receive Maximum，以及 broker 正常发送且可按 MQTT v5 类型安全跳过的属性。协议非法、无法安全跳过的属性或长度越界必须关闭连接并有界重连，而不是越界访问。CONNACK、SUBACK、PUBACK 失败 reason code 不能当作成功；QoS 1 DUP 重投递与 RPC 去重是不同层次，都须测试。不得把 PUBACK 当作 Handler 执行成功的证明。

网络处理与硬件 Handler 必须分离，Handler 执行不得阻塞 PUBACK、keepalive 或重连处理；单设备执行队列有界，无容量返回 busy。内存分配失败和响应过大必须返回有界错误或安全断连，不能截断 JSON 或继续执行尚未获得缓存资源的操作。大 describe 可复用启动时生成的不可变描述或流式发送，去重缓存可引用该描述，不要求保存 8 份 64 KiB 副本。

#### Scenario: TCP 分片和粘包

- **WHEN** 一个 PUBLISH 被拆成多个 TCP 片段，或多个 MQTT 包出现在一次 read 中
- **THEN** 固件完整重组并逐包处理，不丢包、不越界、不重复执行 Handler

#### Scenario: 订阅被 broker 拒绝

- **WHEN** SUBACK 返回失败 reason code
- **THEN** 固件不上报 online，记录可诊断原因并进入有界重连流程

### Requirement: Handler 实例上下文与资源预算

Protocol 注册表必须（SHALL）同时描述函数签名元数据和可执行 Handler，并传入对应节点的实例上下文（node_id、父节点、板级连接参数及实例私有数据）。不得仅用全局函数指针区分同 UUID 的多个 OLED 实例。接口以名称寻址，MVP 不额外要求 Interface UUID。参数 cJSON 由调用方拥有、Handler 仅借用；成功返回的 result 由分发层释放，失败路径同样明确清理。内部组合调用设深度上限 8 并检测循环依赖，不持有会导致递归死锁的非重入全局锁。

在实现客户端前必须完成目标板 RAM/flash 预算，包含 WiFi/lwIP、任务栈、cJSON 树、64 KiB 描述、MQTT 收发、去重和最坏情况下的调用结果。集成验收必须记录目标板、SDK commit、工具链版本、固件大小和最小剩余堆；最大合法描述路径与分配失败路径必须测试。若预算不满足现有契约，必须先修订 spec，不得静默把上限改成 2 KiB/8 KiB。

#### Scenario: 同协议多个实例

- **WHEN** 两个 OLED 节点使用同一 Protocol UUID 但 I2C 地址不同
- **THEN** 调用各自节点时 Handler 收到各自实例上下文，且只访问正确外设

#### Scenario: 嵌套调用成环

- **WHEN** Protocol A 间接再次调用自身并形成环
- **THEN** 分发层返回 execution_failed，不死锁、不耗尽栈

#### Scenario: 分配失败

- **WHEN** 请求无法预留执行及响应缓存所需的内存
- **THEN** 固件在产生硬件副作用前返回 busy 或安全断连，后续合法请求仍可恢复处理

### Requirement: 公共运行时与平台边界

固件必须（SHALL）分离公共运行时、平台适配和板级 profile。公共层包含严格 JSON/schema 验证、发现及续租状态机、MQTT/RPC 关联和去重、Protocol 分发、Device Tree 生成；不得包含按芯片名复制的业务状态机。平台层提供启动/网络就绪与断线事件、IPv4 UDP/TCP/DNS（含超时/部分 I/O）、单调时钟与可选可信跨重启时钟、随机数、唯一身份、任务/互斥/队列、非易失存储、串口/重启及总线驱动绑定；这些适配定义行为契约，不要求自研通用 RTOS 或完整 HAL。Bouffalo 平台可以直接复用已验证的 lwIP/RTOS 公共后端，只把 SDK 差异保留在平台层。

板级 profile 决定芯片封装、引脚复用、总线频率、外设地址、flash/PSRAM、分区、可用按键和静态节点。通用 OLED Handler 通过注册表调用 I2C Protocol，I2C Handler 再使用板级实例和平台驱动；不可由公共 OLED 代码根据芯片宏访问不同 GPIO。平台不得在阻塞 I/O 时持有公共全局锁，单调时钟回绕、随机源失败、存储掉电与任务退出必须有测试。随机 nonce/connection_id 不得使用固定 seed 或仅以启动时间生成。

#### Scenario: 第二个芯片接入

- **WHEN** BL618 适配完成并使用与 BL616CL 相同的 OLED 接口签名
- **THEN** 两者复用公共发现/MQTT/RPC/Protocol 代码与 OLED Handler，只有平台驱动和板级实例不同

#### Scenario: 主机替身替代硬件

- **WHEN** 主机测试注入假时钟、网络、存储、随机数和 I2C 实现
- **THEN** 公共状态机和相同的 Go/C 契约样例可无硬件运行，不依赖 SoC 寄存器头文件

### Requirement: 显式构建目标与输出隔离

固件必须（SHALL）用显式 `TARGET` 选择一个受版本控制的 target profile，至少记录 target_id、真实 chip、board、platform、sdk_chip、sdk_board、SDK 固定 commit、工具链、内存/分区配置和 Protocol 节点配置。真实 chip 与 SDK 的 CHIP 参数是不同字段；目标可额外定义 SDK 所需 CPU_ID，但不得推断所有芯片都需要或使用相同 CPU 核。target_id 匹配 `[a-z0-9][a-z0-9_-]{0,63}`。`make firmware TARGET=<target_id>` 只构建已登记且配置完整的目标，未指定、多义或未知目标必须报错并列出可用项，不静默落到 BL616CL。

输出目录必须按 TARGET 隔离，产物附带目标/真实芯片/板型/SDK/工具链/分区信息；目标切换不能复用另一目标的 CMake cache、链接脚本或烧录配置。烧录命令显式选择 TARGET 与产物并验证元数据一致；工具若无法自动识别实物芯片，应要求操作者核对板型，不宣称能阻止一切误烧。板型未确认的条目只能记为 planned，不得伪装成可构建 profile。

使用 Bouffalo SDK 的应用自定义 BOARD 时，必须采用与该固定 SDK 的 bsp/board 内置目录不冲突的独立名称（例如 agentic_bl618_<board>），不可在应用目录用同名 bl616dk 期待覆盖内置配置。target profile 必须区分内置/应用板级来源并记录期望目录；构建配置阶段校验实际解析的规范化路径与期望一致，证据包含 SDK 的 Select board 路径和 SDK 内置或项目相对路径。名称冲突或路径不符必须停止构建，不等烧录时再发现。镜像 manifest 记录实际选中的板级来源和路径，不能只复述 BOARD 字符串。

#### Scenario: BL618 不等于 SDK 参数 bl618

- **WHEN** 构建真实芯片为 BL618、sdk_chip 为 bl616 的已确认 profile
- **THEN** 构建传给 SDK 的 CHIP 为 bl616，但产物元数据、设备硬件描述和身份前缀仍标识 BL618

#### Scenario: 未指定目标或混用产物

- **WHEN** make firmware 未给 TARGET，或烧录的目标与产物 manifest 不一致
- **THEN** 构建或烧录入口明确失败，不猜测目标，不覆写其他目标产物

#### Scenario: 应用 BOARD 与内置同名

- **WHEN** target 声明使用应用自定义 BOARD=bl616dk，而固定 SDK 已有 bsp/board/bl616dk
- **THEN** 配置校验拒绝，要求独立名称，不让 SDK 优先选中内置目录后继续生成镜像

#### Scenario: 实际 BOARD 路径不符

- **WHEN** target 期望应用板级目录，但 SDK Select board 解析到另一个规范化路径
- **THEN** 构建立即失败并报告期望/实际路径，不发布带错误板级 manifest 的产物

### Requirement: 能力由声明而非芯片分支决定

两种首期固件必须（SHALL）在 Device Tree 的 device.hardware 中输出 vendor、chip、board、firmware_target，已知时可输出 board_revision；均为 1–64 字节字符串，值来自已确认 profile，不包含凭据。device-profile 将其作为可选兼容元数据，旧设备缺少该对象仍合法。服务端不得依据 chip/model/device_id 前缀为函数添加隐式能力或改变寻址，所有路由只依赖经过验证的 Protocol 声明。

相同接口语义在不同芯片复用同一 Protocol UUID；不兼容参数/结果或行为变更须按项目分配表分配新协议身份，不能因芯片不同就复制语义相同的 UUID，也不能同 UUID 偷改签名。未接入或未编译的外设不生成可调用节点。更小内存的未来芯片仍须满足既有报文、去重与时间边界，做不到就通过独立 change 协商新契约，不能按芯片名静默缩减上限。

#### Scenario: 不同芯片同协议调用

- **WHEN** BL616CL 与 BL618 分别声明相同 UUID/Interface 的 OLED 函数
- **THEN** 服务端无需芯片专属分支，通过不同 device_id/node_id 正确路由，工具参数语义一致

#### Scenario: 板型没有 OLED

- **WHEN** 某目标没有配置 OLED 节点
- **THEN** 描述不宣称 OLED 能力，对其调用被拒绝；不能用假成功替代真实驱动

### Requirement: 新芯片接入与逐目标验收

后续芯片接入必须（SHALL）新增平台适配或复用已验证后端、登记板级 target、定义稳定身份和内存预算，并执行公共契约测试、独立构建、实板接入/外设/故障恢复矩阵；无需修改 MQTT 主题版本、MCP 基础工具或服务端芯片白名单。首期 BL616CL 和 BL618 分别记录 planned、build-verified、hardware-verified 的证据状态，不能用一种芯片通过代替另一种。至少一块已确认板型代表每种芯片进行硬件验收，不承诺同芯片所有板型已支持。

本期公共网络基线为 WiFi STA + IPv4 UDP 发现 + MQTT v5 TCP。未来非 Bouffalo、不同 RTOS 或不同联网方式可复用线上契约，但实际适配不在本期；不支持广播、WiFi 或当前资源约束的芯片不能仅登记名字就称为兼容。真实身份、schema 和存储格式跨版本迁移需在新适配变更中说明。

#### Scenario: 一种芯片缺少实板

- **WHEN** BL616CL 已验证，但 BL618 只有成功构建、尚无实板测试
- **THEN** BL618 只标记 build-verified，其硬件任务保持未完成，不宣称双芯片交付完成

#### Scenario: 新芯片保持服务端不变

- **WHEN** 新适配满足公共网络和协议契约并通过同一测试矩阵
- **THEN** 现有服务端直接识别其设备描述和函数，不要求新增芯片分支或硬编码 UUID
