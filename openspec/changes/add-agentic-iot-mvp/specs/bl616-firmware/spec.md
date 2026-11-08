# Spec Delta

## Purpose

定义 BL616CL 的身份、Bouffalo SDK 构建/板级适配和独立验收。公共运行时要求集中在 firmware-platform；此 capability 保留原名称 bl616-firmware，但目标明确是 BL616CL，不代表 BL616、BL618 或 BL618DG 自动兼容。

## ADDED Requirements

### Requirement: BL616CL 公共运行时实现

BL616CL 固件必须（SHALL）完整实现 firmware-platform、device-discovery、mqtt-messaging 和 device-profile 的行为，复用公共发现/续租、MQTT v5、RPC 去重、Protocol 注册与描述生成代码。不得以固件公共条款移出本文件为由缩减原有范围；包括 WiFi 串口配置、原子存储、factory_reset、内存边界及真实 I2C/外设调用。

#### Scenario: BL616CL 完整链路

- **WHEN** 已确认 BL616CL 板型上电且 WiFi/broker/发现服务可达
- **THEN** 设备完成发现、MQTT 订阅、online、describe 确认，MCP 能调用其真实低风险外设

### Requirement: BL616CL 稳定身份

固件必须（SHALL）使用原厂稳定的 48 位唯一 MAC，或经验证可唯一映射为 48 位的硬件标识，生成 `bl616cl-` 加 12 位小写十六进制字符；不采用可配置/随机化运行时 MAC，不任意截断更长 eFuse ID。该规则保持已有 BL616CL ID 格式，重启、重新烧录、factory_reset 和 broker 配置变化不改变身份。ID 用作 MQTT client ID 和主题设备标识。

#### Scenario: 身份跨升级保持

- **WHEN** 同一 BL616CL 实板升级兼容固件并重置配置
- **THEN** device_id 不变，不与其他芯片实例混用身份

### Requirement: BL616CL 构建与板级配置

BL616CL target 必须（SHALL）记录真实 chip=bl616cl、platform=bouffalo、sdk_chip=bl616cl；SDK 固定提交为 237ed937856bf435f4f3642ea4728294723e1d87。候选 sdk_board 为 bl616cl_ave_v10 或 bl616cldk，实际 BOARD 必须按实板确认，不能根据芯片型号猜测引脚/flash/PSRAM/按键。I2C 引脚与 OLED 型号/地址、电压、分区和启动参数分别登记；多个板型分别建立 target，不复制公共状态机。构建经 firmware-platform 的 TARGET 入口选择并隔离产物。

#### Scenario: 两个 BL616CL 板型

- **WHEN** 同芯片两块板的 I2C 引脚或内存配置不同
- **THEN** 使用独立 target/profile，协议公共代码不变，不共用未经验证的烧录配置

### Requirement: BL616CL 独立验收

BL616CL 必须（SHALL）在至少一块确认板型上独立通过公共主机测试、目标构建、烧录启动、WiFi/发现/续租、MQTT v5、真实 Protocol 调用、恢复出厂和故障恢复测试，记录工具链、SDK、镜像大小、最大描述路径、最小剩余堆与栈测量。BL618 的测试结果不能替代 BL616CL 的证据。

#### Scenario: 最大描述与内存失败

- **WHEN** BL616CL 执行最大合法描述及资源分配失败测试
- **THEN** 保持既定字节/去重边界并安全失败，资源不足时先修订规范，不默默缩减能力或假报验收通过
