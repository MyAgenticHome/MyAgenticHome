# Tasks

SDK 同步任务 0.6 已完成；其他实现、构建和硬件验收任务仍待完成。规范审阅、依赖静态核查不等于实现或验收通过。按 design 中的 gate 放行，先跑通无硬件闭环，再接实板。新增 0.x 为前置验证，7.x 按无硬件闭环、实板闭环和交付重新编排。

## 0. 契约冻结与依赖验证

- [ ] 0.1 （G0a）建立共享 JSON 正反例（发现、status、describe、invoke、参数/结果 schema），冻结线上版本、字段、错误码和字节上限；验证示例自洽并覆盖重复键、非法 UTF-8、JSON 深度、Unicode 长度和非法 schema，供 Go/C 使用同一测试输入，包含字符串/键的 U+0000 拒绝与字面反斜杠保留，覆盖 BL616CL/BL618 两目标
- [ ] 0.2 （G0a）锁定 Go 最低版本、官方 MCP Go SDK、paho.golang/autopaho、bbolt 和 YAML 包的兼容版本，记录在 go.mod/go.sum；通过最小程序验证 MCP 初始化/动态增删/通知，以及 MQTT 订阅选项、断线不排队和重连不补发行为
- [ ] 0.3 （G0b）在已同步的本仓库 SDK 固定 commit `237ed937856bf435f4f3642ea4728294723e1d87` 上，分别验证 BL616CL 与原始 BL618 目标最小示例可构建（BL618 映射 CHIP=bl616，不是 bl618dg）；记录工具链/板型/命令与产物，不能以子模块下载成功或旁边 checkout 的构建代替
- [ ] 0.4 （G0b，配合 6.1）分别确认 BL616CL 与 BL618 实板、flash/PSRAM、引脚、OLED 型号/分辨率/I2C 地址/供电和按键，冻结首批 Protocol UUID、Interface 签名及外设边界；验证接线与板级配置一致，不假定 OLED 板载
- [ ] 0.5 （G0b）评估可复用 MQTT v5 实现的许可证、lwIP 移植和内存，并逐目标测量 WiFi/lwIP、任务栈、cJSON、最大描述、收发与去重的 RAM/flash 预算；确认复用或受限实现方案，预算不足先修订规范，不静默降级到 MQTT 3.1.1 或缩减上限

- [x] 0.6 已执行 `git submodule update --init --recursive -- bouffalo/bouffalo_sdk`，相对 URL 解析为 `git@github.com:MyAgenticHome/bouffalo_sdk.git`；核对本仓库 SDK HEAD 为 `237ed937856bf435f4f3642ea4728294723e1d87`、递归状态正常、SDK 工作区干净。仅代表同步与版本检查完成，不代表 0.3 构建或硬件验收通过

## 1. 工作区与 broker 基础

- [ ] 1.1 创建 `server/`、`firmware/`、`broker/`、`scripts/`、`docs/protocols/` 目录和根 `Makefile`（firmware 必须显式 TARGET，产物隔离），验证 `make help` 能列出 broker、server、firmware、test 目标；区分无硬件测试和硬件验收
- [ ] 1.2 添加 `broker/mosquitto.conf` 与 `docker-compose.yml`（固定测试镜像版本/digest、1883、匿名、持久化卷、payload 上限 131072 字节、显式可信 LAN 绑定），用 MQTT v5 pub/sub 验证 retained 消息在正常容器重启后仍存在；区分崩溃时持久化保证，不把历史 online 当作存活证明
- [ ] 1.3 在 `broker/README.md` 记录启动、停止、数据卷、端口、防火墙和异常退出的持久化边界，验证命令可从干净环境执行且不默认绑定公网；核对 Go broker_url 与 broker 实际绑定一致（LAN 与 loopback 不混淆）
- [ ] 1.4 初始化 `server/` Go module，实现配置加载（flag > env > yaml > 默认）、slog、`/healthz`、`/readyz` 和优雅退出；验证 build/test 通过，healthz 反映存活，readyz 仅在存储可用、MQTT 连接且 status/resp 订阅成功后为 200，broker 断连时为 503，无设备不影响就绪

## 2. 发现协议与服务端

- [ ] 2.1 在 `server/internal/discovery` 实现请求/响应模型和严格 JSON 校验；单元测试覆盖字段/类型、设备 ID、主版本、未知字段、512 字节上限、nonce、host/port/ttl、重复键、超长或截断数据报
- [ ] 2.2 实现 UDP 37020 监听、nonce 回显、源端口单播和 advertise_host；验证显式覆盖生效、唯一 LAN 地址自动探测、多地址歧义拒绝启用发现、生成的 offer 不超 512 字节，不把 Docker bridge 端口映射当广播可达保证
- [ ] 2.3 在 `docs/protocols/discovery.md` 记录 v1 报文、2 秒响应窗口、新 nonce、非负 0–250ms 抖动/降频、80%–85% ttl 后台续租/失败不延长旧租约、无 RTC 冷启动重发现和 SSID/子网/掩码指纹；文档样例使用 0.1 的共享输入
- [ ] 2.4 实现每来源 IP 5/s（突发 10）、全局 100/s（突发 200）的限流及有界来源桶与采样日志；验证畸形/超额报文不导致退出或无界增长，日志仅含来源和原因、不输出秘密

## 3. 服务端 MQTT v5 消息层

- [ ] 3.1 用 paho.golang/autopaho 实现 v5 连接、重连、status/resp 订阅；使用 clean_start=true、Session Expiry=0，resp 为 QoS 1/RAP=1/Retain Handling=2，status 为 QoS 1/RAP=1/Retain Handling=0，接收 retained 线索与实时空 retained 清除；真实 broker 测试验证 SUBACK/失败 reason code、断连重连、retained 响应拒绝和无离线队列；验证周期 online 接收、状态清除后同连接重新确认、正常重复公告不重复 describe
- [ ] 3.2 实现 pending 关联（设备主题 + connection_id + JSON id + 36 字节 UUID Correlation Data）；describe 单次 5 秒、最多 3 次总尝试/15 秒且 ID/负载不变，invoke 单次 5 秒无应用重试；测试取消/断连清理、迟到/重复/错误连接响应丢弃、每设备一个 RPC/busy，以及提交前 broker_unavailable、提交后无响应 outcome_unknown；区分直接 invoke 的 busy 与注册表单一待刷新意图，不把等槽转成跨连接 MQTT 发布队列
- [ ] 3.3 实现 describe/invoke 信封、状态字段与结果解析；单元测试覆盖 unsupported_method、各 unknown_*、invalid_params、execution_failed、request_id_conflict、busy、payload_too_large、畸形 JSON、result/error 互斥、身份冲突；已关联合法信封的 result schema 不符立即 invalid_device_response，身份不匹配仍等待；PUBACK 不视为执行成功
- [ ] 3.4 在 `docs/protocols/mqtt.md` 记录主题、状态/Will/连接代次、QoS/订阅选项、重试/去重、错误与副作用语义、报文大小分层；用 0.1 样例验证文档与实现一致
- [ ] 3.5 用真实 broker + 假设备验证响应丢失但操作已执行、实时/历史 retained 控制报文、TCP/客户端断连及服务端重连不补发；测试 4096 字节请求/invoke 响应、65536 字节树、69632 字节 describe 响应和 73728 字节 MQTT 报文边界

## 4. Device Tree、Protocol 校验与注册表

- [ ] 4.1 在 `server/internal/profile` 实现树/节点/Protocol/Interface/函数模型、身份与版本校验、UUID/名称唯一性、64 KiB/256 节点/深度 8/32 Protocol/16 Interface/64 函数限制、全树敏感键检测和未知元数据保留；实现有界 schema 子集/参数/结果验证，测试未知 UUID、未知校验关键字拒绝、可选 hardware 元数据和每个边界；旧设备无 hardware 仍合法，真实 chip 与 sdk_chip 分离，不增加芯片白名单
- [ ] 4.2 实现独立连接状态 discovered/online/offline/expired、connection_confirmed、描述状态 missing/fresh/stale/invalid 与 broker 状态；可注入时钟测试重复发现、无发现直接 status、IP/型号变化、5 分钟过期及重激活，默认列表排除 expired
- [ ] 4.3 实现有效上线/恢复后 2 秒内登记刷新意图：空槽时触发后 2 秒内发送，invoke/旧代次 describe 占槽时合并意图，槽释放后 2 秒内优先刷新，不取消既有硬件调用；后台 describe 首发起最多 15 秒，失败后 5/10/30/60 秒退避，周期 online 不重置退避；候选/确认连接分离、最近 64 项退役集合及未知候选失败后后台重确认 B，回退不延长 MCP 15 秒刷新预算；测试同连接基线不变、矛盾版本 invalid/旧结果失效、新连接版本变化和回滚正常、清除后同 ID 重公告恢复与异常连接隔离的区别
- [ ] 4.4 在 bbolt 同一事务提交 Device Tree 和 `(device_id,node_id,protocol_uuid,interface,function)` 索引，提交后发布内存快照；测试重启恢复、索引重建、验证/索引/提交失败保留完整旧快照但禁止调用、单条坏缓存忽略、整个 DB 损坏保留原文件并明确恢复/报错
- [ ] 4.5 在 `docs/protocols/device-tree.md` 记录线上 schema_version 1.0（v0 仅为产品阶段）、节点/Protocol/签名、Unicode 长度和字节/深度限制；验证合法与非法样例通过共享测试
- [ ] 4.6 在 `docs/protocols/uuids.md` 正式分配首批 Protocol UUID 并记录接口版本/参数/结果与板级实例；验证 UUID 格式合法且树通过 4.1，不把 design 的样例 UUID 自动视为正式分配
- [ ] 4.7 实现有界事件合并与快照对账，2 秒内通知网关；测试 broker 断连立即禁用动态能力、历史 retained online 不可直接调用、A 连接迟到 online/offline 不影响已确认 B，空 retained status 清除使旧刷新代次失效，以及通知暂时失败后能恢复且不阻塞状态处理；验证清除后下一个周期公告可以建立新刷新代次，正常 fresh 公告仅更新观测信息

## 5. MCP 网关

- [ ] 5.1 使用官方 MCP Go SDK 在 `/mcp` 提供 Streamable HTTP，默认 loopback；实现 Host/Origin 显式允许列表、非回环警告，声明 tools/resources 的 listChanged；用 Inspector 或官方客户端验证初始化、资源读取、空设备行为与不可信来源拒绝
- [ ] 5.2 实现五个基础工具及资源摘要（连接状态、connection_confirmed、描述状态、revision、总函数数/已发布数/截断）；测试未知设备、无描述/离线只读缓存、expired 列表排除/单资源诊断，以及刷新操作
- [ ] 5.3 动态工具仅发布已确认且 fresh 的函数；按完整寻址元组排序选取，默认每设备 64/全局 256；名称可读前缀最多 45 字符，加 `__` 和紧凑 UTF-8 JSON 元组 SHA-256 前 16 位，总长 <=64；测试规范化/截断、重启稳定、摘要碰撞拒绝发布、预算截断和通用调用可达
- [ ] 5.4 动态工具与 invoke_protocol 共用新鲜快照校验和路由，失败用 isError 的稳定 JSON 错误；与假设备测试 device_offline、broker_unavailable、unsupported_operation、invalid_params、invalid_device_response、timeout、outcome_unknown、busy、payload_too_large 和设备错误，不自动重试控制调用
- [ ] 5.5 stale 调用先合并刷新，调用起最多 15 秒包含等槽/退避/合并等待/describe/校验提交，随后 invoke 最多 5 秒，总预算 20 秒且服从客户端截止；测试等待扣减预算、单个等待方超时不影响其他有效等待方、取消后无需求不发新请求、刷新失败 device_profile_stale、独立 refresh 超时 timeout、刷新后函数移除、调用槽被抢返回 busy、异常连接不靠同连接刷新恢复，以及离线/broker 不可用拒绝
- [ ] 5.6 在 `docs/mcp.md` 记录端点、资源/基础/动态工具、匿名边界、可信客户端配置与副作用确认策略；用 Inspector/官方客户端完成无模型验收，明确模型账户/密钥只属于可选自然语言演示，不属于网关/固件依赖
- [ ] 5.7 实现活动 MCP 会话 list_changed 合并通知及快照对账；测试上下线、broker 重连和描述失效后的工具集合；资源内容变化不误发资源列表变化通知，MVP 不声明 resources.subscribe

## 6. 公共固件运行时与 BL616CL/BL618 适配

- [ ] 6.1 与用户完成 0.4 的两种实板确认（BL616CL 的 bl616cl_ave_v10/bl616cldk、BL618 的 bl616dk 均仅为候选；后者需按实板决定应用 BOARD）、首批 I2C + OLED 或等效低风险外设清单，记录到 firmware README/UUID 表；验证构建板型、引脚、实例配置与实物一致
- [ ] 6.2 创建 `firmware/` CMake 应用并引用本仓库子模块，在 0.3/0.5 放行后实现最小启动与串口日志；验证 `make firmware TARGET=<已确认目标>` 为 BL616CL/BL618 分别产出隔离镜像、烧录后打印启动横幅，记录工具链/SDK/镜像大小
- [ ] 6.3 实现 WiFi STA、NVS 覆盖优先于构建默认值、无凭据等待串口配置、不回显密码和 1/2/5/10/30 秒重连；主机测试状态/退避，硬件验证入网与断网恢复
- [ ] 6.4 实现版本/CRC/双槽或等价原子配置提交、本地串口设置 WiFi、经确认的 factory_reset 和可用板型按键长按 5 秒；测试掉电/坏 CRC/写失败保留副本，无默认凭据恢复出厂后能串口重新配置，不声称擦除编译期凭据
- [ ] 6.5 按 0.5 选型实现 MQTT v5 适配层/必要受限子集；主机测试覆盖 CONNECT/CONNACK、SUBSCRIBE/SUBACK（含 RAP/RH）、PUBLISH/PUBACK、PING、DISCONNECT、属性/失败 reason code、Remaining Length、TCP 分片/粘包/部分写/关闭和畸形输入
- [ ] 6.6 实现 MQTT 连接状态机、新 connection_id、Will、成功 SUBACK 后 online、有界主动 offline、60 秒 keepalive 和 1–60 秒指数退避；真实 broker 测试 QoS 1 重投递、session 不排队、broker 重启恢复与 retained 控制拒绝，网络任务不被 Handler 阻塞；实现首次公告后每 30–35 秒重新发布相同 connection_id/版本的 retained online、PUBACK 5 秒失败重连、下线优先取消公告和旧回调隔离，验证无需重连恢复清除后的状态
- [ ] 6.7 实现静态 Protocol/Interface/Handler 注册表、节点实例上下文、参数/结果校验和明确 cJSON 所有权；主机测试未知寻址、非法参数、同 UUID 多实例、组合调用深度 8/检测环/无递归锁死
- [ ] 6.8 由同一注册表生成不可变 Device Tree，节点 ID/顺序稳定，revision 发布递增但允许显式回滚；主机生成结果通过服务端校验器，两种硬件 describe 返回完整树，带真实 chip/board/target 元数据，验证无凭据泄露；验证同启动周期/同连接完整描述和版本不变，新固件/配置以新 connection_id 公告，describe 对齐首次 online 基线
- [ ] 6.9 实现 OLED Handler 经注册表调用父 I2C Protocol；mock 测试 init/display_text 调用链、驱动超时 <=3 秒及多实例地址不串用，实板显示真实结果
- [ ] 6.10 实现 UDP 发现、新 nonce/完整 2 秒窗口/非负抖动/降频/首个有效响应、网络指纹和单调 ttl；在 80%–85% ttl 后台续租，相同配置仅在旧截止前提交并生效才保持 MQTT/connection_id，变化配置有界切换；测试续租写失败不延长租约与 flash 磨损预算；主机测试无 RTC 冷启动、同网段 DHCP 改址、网络变化、续租失败直至过期才主动下线重发现、10 次连接失败、迟到/重复响应；可用网络条件下实板首次启动 60 秒内连 broker；假时钟/慢存储测试提交跨截止、恰好截止到期优先、timer 延迟不绕过检查、迟到提交只启新连接且截止为接收时刻+ttl、factory_reset 淘汰旧写任务
- [ ] 6.11 实现 describe/invoke RPC、主题/属性/连接身份/严格 JSON 检查；每设备一个执行槽，至少 8 项且完成起 >=30 秒的去重缓存，执行前占位/预留、不驱逐未过期项；测试执行中重复、同 ID 不同负载、缓存满 busy、响应丢失、全部错误码和实际外设调用
- [ ] 6.12 在 firmware README 记录构建/烧录/串口/恢复/UUID 扩展；主机测试启用 ASan/UBSan 和有界畸形输入，实板记录最大描述路径、分配失败恢复及最小剩余堆，验证干净 checkout 可逐目标构建；不满足内存契约先修订 spec

- [ ] 6.13 提取 firmware-platform 公共运行时和平台接口（网络/时钟/随机/身份/存储/任务/串口/I2C），用主机替身测试超时/回绕/掉电/随机源失败；公共层不包含芯片业务分支，不引入完整通用 RTOS/HAL
- [ ] 6.14 建立 target/profile 校验与 manifest：真实 chip、SDK CHIP/BOARD、工具链、SDK commit、内存/分区/节点；缺省或非法 TARGET 拒绝、CMake cache/产物隔离、烧录目标一致性检查，未确认板型仅 planned；应用自定义 BOARD 名不得与 SDK 内置同名，profile 记录期望来源/目录，校验 SDK 实际 Select board 规范化路径并写入 manifest，同名或路径不符必须在配置阶段失败
- [ ] 6.15 实现并验证 BL616CL 平台/板级绑定和 bl616cl- 稳定身份，保持既有格式；逐项满足 bl616-firmware，独立记录构建、实板与资源证据
- [ ] 6.16 实现并验证原始 BL618 平台/板级绑定（真实 chip=bl618、sdk_chip=bl616）、bl618- 稳定身份、I2C/OLED 实例、WiFi/存储/启动/烧录；拒绝误用 BL618DG profile，独立满足 bl618-firmware，不复制公共状态机；需自定义板型时采用独立 BOARD 名并核对实际解析路径，禁止同名 bl616dk 覆盖
- [ ] 6.17 在 `docs/firmware-porting.md` 记录后续芯片适配清单、接口所有权/时限/错误约束、身份规则、板型/分区/工具链选择和 planned/build-verified/hardware-verified 状态；验证新增主机测试 target 不要求修改服务端芯片逻辑，不宣称已实现第三种芯片

## 7. 端到端集成与交付

- [ ] 7.1 （G1，无硬件）真实 broker + Go + 假设备跑通发现/上线/describe/Protocol 列表/动态及通用调用；验证 MQTT/注册表/MCP 的集成错误矩阵，不要求 G0b、实板或模型账户
- [ ] 7.2 （G2/G3，实板）分别按 broker → server → BL616CL/BL618 跑通首次发现、确认在线、fresh 描述和真实 I2C/OLED 或已确认等效外设调用；对照动态工具与通用工具的寻址/结果，记录实物可观察证据
- [ ] 7.3 验证 broker 正常/异常重启、设备断电、UDP 阻断、服务端重启、历史 online、旧连接迟到消息、丢响应与固件回滚；确认不重复执行、不回放旧操作、仅有效 offer 原子提交成功才续租，健康连接无缝续租、错误可诊断
- [ ] 7.4 在 `docs/deployment.md` 记录 NAS/树莓派原生部署、端口/网络限制、依赖锁定清单、固件接线/烧录、持久化/备份/非破坏性恢复和排障；验证干净环境可复现 G1/G2/G3
- [ ] 7.5 （G4）运行 `openspec validate add-agentic-iot-mvp --strict`、Go 单元/集成测试与 `go test -race ./...`、固件主机测试、`make firmware TARGET=<目标>` 的双目标构建和实板矩阵；记录命令、版本、通过/失败/跳过，硬件未到场标未验证，不得勾选硬件任务或声称完成交付
- [ ] 7.6 如需自然语言演示，另配支持 Streamable HTTP 的模型客户端、账户/凭据与网络，启用控制确认策略并验证 display_text 等低风险外设；不把 API 密钥放进固件，不用此演示替代协议验收；未选择演示时明确记录不适用，不因此阻塞 G4

- [ ] 7.7 同时连接至少一台 BL616CL 与一台 BL618，验证相同 Protocol UUID 的独立寻址、动态工具、稳定身份与状态隔离；重启任一设备不影响另一台，分别记录硬件证据，缺板不宣称双芯片交付完成
- [ ] 7.8 回归六项 review 场景：已退役 online、候选失败后恢复 B、NUL/字面反斜杠、已关联坏 result 立即失败、实时清空 status（含迟到 describe）、续租与非负抖动完整窗口；共享 Go/C fixture + broker 集成测试 + 双芯片实板相关子集均记录结果

- [ ] 7.9 第二轮 review 回归：保持 MQTT 连接时清空 status → 至多 35 秒内设备重新公告 → 新 describe 确认；同连接矛盾版本时旧刷新不能复活；invoke 占槽与 MCP 15 秒含等待预算；59.9 秒 offer/60 秒到期/60.2 秒提交、恰好截止到期优先及旧配置写淘汰；SDK 内置与自定义 BOARD 同名和实际路径不符均构建失败。记录 Go/C/真实 broker 及逐板相关子集证据

## Workflow follow-up

- 仅在实现、测试与交付任务完成并经评审后归档本 change，再确认 `openspec/specs/` 主规范同步正确；规范评审完成不构成归档条件。
- 后续独立 change：WAMR、WASM 编译/分块下发/安装/运行/停止、宿主 ABI 与资源/权限边界；当前只交付 Protocol/Interface/Handler 直调。
