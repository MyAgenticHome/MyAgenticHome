# Spec Delta

## Purpose

新增原始 BL618 芯片的固件实现描述，与 BL616CL 共享 firmware-platform 公共运行时。本文件的 BL618 不指 BL618DG/BL618M；后者需要独立的芯片/核/板型验证，不得混用构建目标或硬件能力。

## ADDED Requirements

### Requirement: BL618 公共运行时实现

BL618 固件必须（SHALL）实现与 BL616CL 同一版本的发现、后台续租、MQTT v5、Device Tree、RPC/去重与 Protocol 调用契约，复用 firmware-platform 公共代码；WiFi/lwIP/cJSON/非易失存储和 I2C 的 SDK 差异封装在平台/板级适配。SDK MQTT-C 的 3.1.1 实现不能替代 v5；客户端适配必须在 BL618 上验证而非仅复用 BL616CL 的结论。

#### Scenario: BL618 完整链路

- **WHEN** 已确认 BL618 实板上电，网络和服务端正常
- **THEN** 设备接入与描述确认成功，MCP 基础工具和动态工具可调用真实外设，不需要服务器选择 BL618 专用路径

### Requirement: BL618 稳定身份与硬件元数据

固件必须（SHALL）读取原厂稳定的 48 位唯一 MAC，生成 `bl618-` 加 12 位小写十六进制字符，并作为 MQTT client ID 和主题标识。不得从 SDK CHIP=bl616 推导 bl616 前缀，不使用随机/可配置 MAC，不截断未经验证的长 eFuse ID。device.hardware 的 chip 必须为 bl618，board 和 firmware_target 使用实际已确认 profile；device.model 是板级产品型号，不等同于 sdk_board。重启、烧录和 factory_reset 均保持身份。

#### Scenario: SDK 共用但身份不混淆

- **WHEN** BL618 固件使用 SDK 的 bl616 构建目标
- **THEN** device_id 仍为 bl618- 前缀，硬件 chip 为 bl618，服务端不会将其登记为 BL616CL

### Requirement: BL618 SDK 与板级构建映射

BL618 target 必须（SHALL）记录真实 chip=bl618、platform=bouffalo、sdk_chip=bl616，依据固定 SDK 提交 237ed937856bf435f4f3642ea4728294723e1d87 的 docs/zh/get_started/get_started.rst 中 BL618 使用 CHIP=bl616 的说明。SDK 中 bl616dk 是可评估的板级基线，不是已确认适用于用户实板的默认 BOARD；仅在核对引脚/flash/PSRAM/启动与分区后才能采用，否则在应用目录提供名称不与 SDK 内置 BOARD 冲突的独立 board profile，按 firmware-platform 校验 SDK 实际 Select board 路径，保持 SDK 子模块内容不变。不得设置未经支持的 CHIP=bl618，也不得用 bl618dg/bl618dgdk 替代原始 BL618。

必须分别验证串口、WiFi、I2C、flash/PSRAM 和烧录流程，锁定兼容工具链。SDK 文档/目录存在仅为静态证据，不等于实板已验证。板型未确认时保持 planned，不能把不完整 profile 加入可构建列表；make firmware TARGET=<已确认 BL618 target> 按 profile 转换 SDK 参数。

#### Scenario: 误选 DG 目标

- **WHEN** 真实 BL618 target 被配置成 sdk_chip=bl618dg 或使用未经确认的 DG 板型
- **THEN** profile 校验拒绝构建，不生成看似 BL618 的错误镜像

#### Scenario: 板级基线不匹配

- **WHEN** BL618 实板引脚或内存与 bl616dk 基线不同
- **THEN** 提供并验证独立名称的应用板级 profile，核对实际解析 BOARD 路径；不通过同名 bl616dk 覆盖、修改公共协议代码或默认引脚猜测规避差异

### Requirement: BL618 外设与资源验收

BL618 必须（SHALL）至少配置 I2C 与 OLED 或用户确认的等效低风险外设，以相同语义复用既有 Protocol UUID/Interface；实际 GPIO、地址、电压、时钟和驱动绑定由 BL618 板级 profile 决定。必须独立测量 WiFi/lwIP/任务栈/cJSON/MQTT 收发/64 KiB 描述/至少 8 项去重缓存的资源开销，不从芯片名称或 BL616CL 的测量推断 RAM/flash 足够。未配置外设不生成其能力。

#### Scenario: BL618 真实 OLED 调用

- **WHEN** 通过 MCP invoke_protocol 或动态工具调用 BL618 的 display_text
- **THEN** OLED Handler 经公共注册表调用 BL618 实例的 I2C 驱动，在真实屏幕显示结果，不误操作另一台 BL616CL

#### Scenario: BL618 预算不足

- **WHEN** 实测内存不足以实现公共报文或去重边界
- **THEN** BL618 放行被阻止并先修订契约，不静默降到 MQTT 3.1.1 或更小描述上限

### Requirement: BL618 独立与混合设备验收

BL618 必须（SHALL）独立通过构建/烧录、WiFi、UDP 发现/无缝续租、MQTT 订阅与重连、describe、真实调用、去重和 factory_reset，记录 SDK/工具链/板型/镜像及内存证据。必须同时连接一台 BL618 与一台 BL616CL，验证稳定身份、相同 Protocol UUID 的不同实例路由和状态隔离；重启其中一台不能撤下另一台工具。缺少 BL618 实板时明确标未验证，不以主机模拟或 BL616CL 成功代替。

#### Scenario: 双芯片混合运行

- **WHEN** BL618 与 BL616CL 同时在线并暴露相同 OLED Protocol UUID，随后仅 BL618 断电
- **THEN** 调用此前分别到达正确设备；掉线后只撤下 BL618 工具，BL616CL 仍可调用

#### Scenario: 只有编译通过

- **WHEN** BL618 镜像已生成但尚未烧录验证
- **THEN** 只记录 build-verified，不勾选硬件任务、不宣称 BL618 已交付
