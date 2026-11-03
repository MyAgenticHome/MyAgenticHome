# Tasks

## 1. 工作区与 broker 基础

- [ ] 1.1 创建 `server/`、`firmware/`、`broker/`、`scripts/`、`docs/protocols/` 目录和根 `Makefile`，验证 `make help` 能列出 broker、server、firmware、test 目标
- [ ] 1.2 添加 `broker/mosquitto.conf` 与 `docker-compose.yml`（1883、匿名、persistence、消息大小限制），验证 `docker compose -f broker/docker-compose.yml up -d` 后可用 `mosquitto_pub`/`mosquitto_sub` 以 MQTT v5 收发 retained 消息，且容器重启后 retained 消息仍存在
- [ ] 1.3 在 `broker/README.md` 记录 broker 启动、停止、数据卷和端口检查命令，验证文档中的命令可从干净环境执行成功
- [ ] 1.4 初始化 `server/` Go module，实现配置加载（flag > env > yaml > 默认）、slog 日志、`/healthz` 和优雅退出，验证 `go build ./...`、`go test ./...` 通过，启动后 `curl /healthz` 返回 200

## 2. 发现协议与服务端

- [ ] 2.1 在 `server/internal/discovery` 实现发现请求/响应结构、JSON 编解码、主版本与字段校验，验证单元测试覆盖合法请求、缺字段、非法设备 ID、主版本不兼容和未知字段
- [ ] 2.2 实现 UDP 37020 监听、nonce 回显、单播响应和 `advertise_host` 自动探测/配置覆盖，验证用 Go 测试客户端广播后收到指向源端口的响应，且配置覆盖时响应主机为配置值
- [ ] 2.3 在 `docs/protocols/discovery.md` 记录 v1 报文、端口、重试与兼容规则，验证文档中的 JSON 示例能通过 2.1 的编解码测试
- [ ] 2.4 为发现服务添加非法报文限流与日志，验证畸形报文不会导致进程退出，且日志包含来源 IP 和失败原因

## 3. 服务端 MQTT v5 消息层

- [ ] 3.1 使用 `paho.golang/autopaho` 实现 MQTT v5 连接、重连、订阅 `iot/v1/devices/+/status` 和 `iot/v1/devices/+/resp`，验证集成测试在 compose broker 上完成连接、断开重连和状态消息接收
- [ ] 3.2 实现 Request-Reply 关联：Response Topic、Correlation Data、5 秒超时、3 次重试、迟到响应丢弃，验证单元测试覆盖成功、超时、重试和重复响应；集成测试向假设备发布请求并匹配响应
- [ ] 3.3 实现 `device.describe` 和 `protocol.invoke` 请求封装与响应解析，覆盖 `unknown_node`、`unknown_protocol`、`unknown_interface`、`unknown_function`、`invalid_params`、`execution_failed` 和设备返回畸形 JSON，验证单元测试覆盖每个错误码
- [ ] 3.4 在 `docs/protocols/mqtt.md` 记录主题、负载、QoS、Last Will、超时、`protocol.invoke` 参数和错误码，验证文档示例与 3.1–3.3 的测试用报文一致

## 4. Device Tree、Protocol 校验与注册表

- [ ] 4.1 在 `server/internal/profile` 实现 Device Tree 模型与校验：必需字段、节点 ID 唯一、Protocol UUID 格式与节点内唯一、Interface/函数名唯一、64 KiB/256 节点/深度 8/32 Protocol/16 Interface/64 函数限制、敏感键拒绝、未知字段保留，验证单元测试覆盖每个拒绝条件和未知 UUID 保留
- [ ] 4.2 实现注册表状态机（discovered/online/offline/stale/expired）、重复发现、IP 变化、型号冲突和事件分发，验证使用可注入时钟的单元测试覆盖全部状态转换和 5 分钟过期
- [ ] 4.3 实现设备上线后 2 秒内自动拉取描述、5/10/30/60 秒退避重试和 `profile_revision` 变化刷新，验证与假设备的集成测试覆盖首次成功、超时重试和版本变化刷新
- [ ] 4.4 实现 `(device_id, node_id, protocol_uuid, interface, function)` Protocol 索引，并与 Device Tree 缓存同事务写入 bbolt；验证临时目录单元测试覆盖正常重启、索引重建、索引失败保留旧值、损坏缓存丢弃
- [ ] 4.5 在 `docs/protocols/device-tree.md` 记录 v0 信封、节点、Protocol/Interface/函数和校验限制，验证文档中的合法示例被 4.1 校验器接受、反例被拒绝
- [ ] 4.6 在 `docs/protocols/uuids.md` 建立 Protocol UUID 分配表并记录示例 I2C/OLED UUID，验证表中的每个 UUID 符合 RFC 4122 且能被 4.1 校验器接受

## 5. MCP 网关

- [ ] 5.1 使用官方 MCP Go SDK 在 `/mcp` 提供 Streamable HTTP，声明 tools/resources，实现 `iot://devices` 与 `iot://devices/{device_id}` 资源，验证 MCP Inspector 或官方客户端能初始化、列出资源并读取单设备资源
- [ ] 5.2 实现 `list_devices`、`get_device_tree`、`list_protocols`、`invoke_protocol`、`refresh_device_profile` 基础工具及参数校验，验证集成测试覆盖正常返回、未知设备 `device_not_found`、未声明函数 `unsupported_operation` 和刷新超时
- [ ] 5.3 实现从 Protocol 索引生成动态函数工具、确定性命名、64 字符截断、冲突哈希后缀、离线移除和 list_changed 通知，验证单元测试覆盖命名规范化、嵌套节点子 ID、冲突、设备上下线后工具列表变化
- [ ] 5.4 实现动态工具和 `invoke_protocol` 路由到 `protocol.invoke`，校验节点/Protocol/Interface/函数/参数并映射 `device_offline`、`timeout`、`unsupported_operation`、`invalid_params` 和设备错误，验证与假设备的集成测试覆盖成功和每类错误
- [ ] 5.5 实现 stale 设备调用前刷新与 `device_profile_stale` 错误，验证集成测试覆盖刷新成功继续调用、刷新失败拒绝调用
- [ ] 5.6 在 `docs/mcp.md` 记录端点、认证边界、资源 URI、基础工具、动态工具和客户端配置示例，验证按文档配置 MCP Inspector 可完成 5.1 的初始化并调用 `list_protocols`

## 6. BL616CL 固件

- [ ] 6.1 与用户确认目标板型（`bl616cl_ave_v10` 或 `bl616cldk`）、板载外设与首批 Protocol 清单（至少 I2C 与 OLED 或等效外设），记录到 `firmware/README.md` 和 `docs/protocols/uuids.md`，验证构建使用的板型和 README/Protocol 表一致
- [ ] 6.2 创建 `firmware/` CMake 应用并引用 `bouffalo/bouffalo_sdk`，实现最小启动与串口日志，验证 `make firmware` 生成 BL616CL 可烧录镜像且串口打印启动横幅
- [ ] 6.3 实现 WiFi STA 连接、Kconfig/NVS 凭据读取和 1/2/5/10/30 秒退避重连，验证主机侧退避逻辑单元测试通过，硬件串口日志显示成功入网和断网重连
- [ ] 6.4 实现配置存储模块（broker 配置、CRC、`factory_reset` 串口命令、按键长按），验证主机侧序列化/CRC 单元测试通过，硬件执行 `factory_reset` 后重新进入发现流程
- [ ] 6.5 实现最小 MQTT v5 客户端编解码：CONNECT/CONNACK、SUBSCRIBE/SUBACK、PUBLISH/PUBACK、PINGREQ/PINGRESP、DISCONNECT、Response Topic/Correlation Data 属性和 Last Will，验证主机侧单元测试覆盖每个报文的编码与解码
- [ ] 6.6 实现 MQTT v5 客户端状态机：连接、订阅请求主题、发布 retained `online`、断线指数退避重连，验证与真实 mosquitto 联调时设备上下线状态正确，broker 重启后自动恢复
- [ ] 6.7 实现 EDK2 风格 Protocol/Interface/Handler 注册表：静态 Protocol 表、Interface 函数表、cJSON 参数提取辅助函数和统一分发入口，验证主机侧单元测试覆盖注册、查找、参数类型错误和未知 UUID
- [ ] 6.8 实现 Device Tree 由 Protocol 注册表生成：静态节点树、稳定节点顺序、Protocol/Interface 声明和 `profile_revision`，验证主机侧生成结果通过服务端 4.1 校验器，硬件 `device.describe` 返回完整树
- [ ] 6.9 实现嵌套 Protocol 组合：OLED Handler 通过注册表查找父 I2C Protocol 并调用其函数，验证主机侧用 mock I2C 接口测试 OLED `init` 和 `display_text` 的调用链
- [ ] 6.10 实现 UDP 发现客户端：广播格式、nonce 校验、首个有效响应采用、配置持久化、过期/连接失败重发现，验证主机侧协议单元测试通过，硬件首次上电 60 秒内取得 broker 配置并连接
- [ ] 6.11 实现 RPC 处理：`device.describe`、`protocol.invoke`、30 秒请求去重缓存、未知节点/协议/接口/函数/参数错误映射，验证主机侧单元测试覆盖全部错误码，硬件调用真实 OLED 或传感器函数返回结果
- [ ] 6.12 在 `firmware/README.md` 记录构建、烧录、串口日志、factory reset、Protocol 注册和 UUID/Interface 扩展方法，验证文档命令从干净 checkout 可构建出固件

## 7. 端到端集成与交付

- [ ] 7.1 按 `broker` → `server` → 真实 BL616CL 的顺序跑通完整链路：发现、获取 broker 配置、上线、自动 describe、`list_protocols` 返回 I2C/OLED Protocol，验证 MCP 客户端通过动态工具和 `invoke_protocol` 调用 OLED `display_text` 返回真实结果
- [ ] 7.2 验证故障恢复：broker 重启、设备断电重启、UDP 37020 被阻断、服务端重启，确认设备自动恢复或返回明确错误，服务端日志包含可诊断信息
- [ ] 7.3 在 `docs/deployment.md` 记录 NAS/树莓派部署、端口、配置示例、固件烧录和排障步骤，验证从干净环境按文档启动后 7.1 的检查可复现
- [ ] 7.4 运行 `openspec validate add-agentic-iot-mvp --strict`、`go test ./...`、`make firmware`，验证全部通过并把结果记录到交付说明

## Workflow follow-up

- 完成评审后归档本 change，并确认 `openspec/specs/` 中生成的主 spec 内容正确。
- 后续独立 change：集成 WAMR、WASM 模块编译/分块下发/安装/运行/停止与宿主 ABI；本 change 只交付 Protocol/Interface/Handler 直调。
