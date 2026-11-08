# Design

## Context

当前仓库尚无应用实现，已完成 Bouffalo SDK 子模块同步，保留 OpenSpec 和工作流配置。`bouffalo/bouffalo_sdk` 已通过 `git submodule update --init --recursive -- bouffalo/bouffalo_sdk` 初始化并检出主仓库锁定的 `237ed937856bf435f4f3642ea4728294723e1d87`；相对 URL 实际解析为 `git@github.com:MyAgenticHome/bouffalo_sdk.git`。本轮再次核对 HEAD、递归子模块状态及 SDK 工作区，固定提交一致、工作区干净，未升级 gitlink、未修改 SDK 源码。

已对照本仓库实际 SDK 静态核查：存在 `bl616cl_ave_v10`、`bl616cldk`、`bl616dk` 板级目录，以及 lwIP socket、cJSON、littlefs KV/easyflash 兼容头和 `examples/shell/wasm_cmd`；MQTT-C README 明确标为 MQTT v3.1.1。`docs/zh/get_started/get_started.rst:198` 明确原始 BL618 使用 `CHIP=bl616`，BL618DG/BL618M 不属于同一目标。`bsp/board/CMakeLists.txt` 优先选择 SDK 内置 BOARD，找不到才选择应用目录，因此自定义 BOARD 必须使用不冲突的名称并验证实际解析路径。

状态边界：**SDK 同步与静态内容核查已完成；BL616CL/BL618 实板选型、最小示例构建、应用固件构建、资源预算、烧录及硬件验收均未完成**。SDK 已下载不等于 target 已达到 build-verified 或 hardware-verified；G0b 仍需逐板放行。

关键约束：

- 服务端运行在 NAS 或树莓派上，需要长期在线、单进程、易部署；MVP 使用明文通信，仅限隔离或可信实验网络，不代表生产环境应禁止加密。
- BL616CL 与 BL618 均需按实际板型独立验证 RAM/flash 预算，不能因共享 SDK 推断资源相同；固定 SDK 中已核实的 MQTT-C 是 MQTT 3.1.1，不能直接满足 MQTT v5 Request-Reply，必须单独评估 v5 客户端实现。
- Device Tree 的具体字段由用户后续细化，因此 v0 固定信封、节点树、Protocol/Interface 声明和通用寻址，其余属性保持开放。
- 设备发现依赖二层广播，跨 VLAN、AP 隔离或访客网络可能不可用，MVP 假定设备与服务端在同一可广播子网。
- 设备能力采用参考 EDK2 的 Protocol/Interface/Handler 模型：Protocol 用 UUID 标识类型，Interface 声明函数，Handler 在固件内实现函数；节点树表达 I2C → OLED 这类嵌套实例关系。
- 本 change 不集成 WASM 运行时；SDK 的 WAMR 组件和 `wasm_cmd` 示例留给后续 change。当前协议注册表和分发函数必须能被将来的 WAMR host import 直接复用。

## Goals / Non-Goals

**Goals:**

- 一套可在单台 NAS/树莓派上运行的服务端二进制，同时承担发现响应、MQTT 客户端、设备注册表、Protocol 索引和 MCP 网关。
- 固件与服务端之间依赖两个明确契约：UDP 发现协议和 MQTT v5 RPC（`device.describe`、`protocol.invoke`）。
- 固件实现 EDK2 风格 Protocol/Interface/Handler 注册表，支持嵌套 Protocol 组合和通用 JSON 参数分发。
- Device Tree 从固件静态 Protocol 注册表生成，服务端校验后建立 Protocol 索引并动态映射为 MCP tools/resources。
- 端到端可本地复现：docker-compose 启动 broker，Go 服务端一条命令启动，固件分别构建并烧录到已确认的 BL616CL、BL618 开发板，并验证混合设备调用。

**Non-Goals:**

- 不实现 TLS、MQTT ACL、设备凭证签发、OTA 升级、云端接入和多服务端高可用。
- 本期只实现 BL616CL 和原始 BL618；其他芯片（包括 BL618DG/BL618M）的实际适配、多核调度和非 WiFi 网络接入留给后续 change。平台接口不等于所有芯片已受支持。
- 不实现 AP/蓝牙配网；WiFi 凭据在构建或首次烧录时预置。
- 不定义最终的 Device Tree 业务字段全集；v0 只保证结构可扩展。
- 不支持 MQTT 3.1.1 设备接入；broker 虽兼容旧客户端，但本项目设备固定使用 v5。
- 不实现设备到设备的消息路由或规则引擎。
- 不集成 WASM 虚拟机，不实现 WASM 模块编译、分块下发、安装、运行、停止和宿主 ABI；这些留给后续 change。本 change 不加载任何动态原生代码。

## Decisions

### D1. 单仓库多构建单元

仓库根目录按角色拆分：

```
server/      Go module，MCP 服务端
firmware/    跨芯片公共应用 + 平台适配 + 板级 profiles，CMake 引用 bouffalo/bouffalo_sdk
broker/      mosquitto.conf + docker-compose.yml
scripts/     本地联调、固件构建与烧录辅助脚本
docs/        部署、MCP、协议与 UUID 分配说明
```

根目录提供 `Makefile` 聚合 `make broker-up`、`make server`、`make firmware TARGET=<target_id>`、`make test`。三种构建系统互不侵入，避免把 Go 工具链塞进固件 CMake，也避免把 SDK 子模块变成 Go 模块的一部分。备选方案是单一 CMake 驱动全部组件，但 Go 服务端与嵌入式工具链差异过大，放弃。

### D2. Go 服务端采用单进程、按职责分包

`server/` 内部分为：

- `cmd/agentic-iotd/`：启动、配置加载、信号处理。
- `internal/discovery/`：UDP 37020 监听、请求校验、单播响应。
- `internal/mqtt/`：MQTT v5 客户端、请求关联、超时重试、状态订阅。
- `internal/registry/`：设备状态机、Device Tree 缓存、Protocol 索引、事件分发。
- `internal/profile/`：Device Tree 校验、节点/Protocol/Interface 模型。
- `internal/mcp/`：MCP HTTP 端点、资源与工具注册、调用路由。
- `internal/store/`：bbolt 持久化。

各组件通过接口和事件通道解耦，`registry` 是唯一的状态权威，`mcp` 只读注册表快照并通过 `mqtt` 发起调用。备选方案是拆成多个微服务，但 MVP 部署目标是单机，拆分会增加发现、配置和运维成本。

### D3. MCP 使用官方 Go SDK + Streamable HTTP

选择 `github.com/modelcontextprotocol/go-sdk` 的 `mcp` 包实现 Streamable HTTP，端点固定为 `/mcp`，默认监听 `127.0.0.1:8080`。实现前锁定 SDK 版本和最低 Go 版本，通过初始化、动态工具增删、资源读取和 listChanged 通知的最小验证；不在规范中假定某个尚未验证的 SDK 方法签名。非回环访问必须显式配置，并限制可信来源；校验 Host/Origin，默认不允许通配 CORS。匿名接口不得暴露公网。MVP 不增加自研鉴权系统；远程访问使用可信 LAN 或已鉴权反向代理。

不使用 stdio 作为主传输：服务端部署在 NAS，需要被多台机器上的 LLM 客户端访问。stdio 可在后续作为调试模式补充。

### D4. MQTT 客户端使用 paho.golang

服务端使用 `github.com/eclipse/paho.golang/autopaho`（MQTT v5）而不是 `eclipse/paho.mqtt.golang`，因为后者只支持 3.1.1，无法携带 Response Topic 和 Correlation Data。`autopaho` 负责重连和会话管理，请求层在其上维护 `correlation_id -> pending request` 映射。

待处理请求以设备、connection_id 和请求 ID 联合关联，并校验响应主题和 JSON id。Correlation Data 为请求 UUID 的 36 字节 UTF-8 文本。单次等待 5 秒；describe 最多 3 次总尝试（首次 + 2 次重试），复用同一 ID 和完全相同负载，总截止时间 15 秒；invoke 不进行应用层自动重试。操作已交给传输层但无响应时返回 `outcome_unknown`，不能推断未执行。请求/响应均不 retained，不跨断线排队，不在重连时补发旧操作；需通过 autopaho 集成测试验证这些约束，而非假设库的默认行为恰好满足。每设备最多一个在途 RPC，额外直接 invoke 返回 busy；刷新只合并一个本地意图，不形成第二个在途请求。空槽时触发后 2 秒内发送，已有 RPC 时等其完成或原截止时间到达释放槽，仍有效的刷新优先于新 invoke 并在释放后 2 秒内发送。后台 describe 从首发计最多 15 秒；MCP 刷新从工具调用起计 15 秒，包含等槽/退避/合并/校验提交，不重置预算，取消方不影响其他有效等待方。响应主题统一为 `iot/v1/devices/{device_id}/resp`。服务端同样使用 clean_start=true、Session Expiry=0；resp 订阅要求 RAP=1、Retain Handling=2，status 要求 QoS 1、RAP=1、Retain Handling=0，接收 retained 候选和实时清除事件。仅检查默认订阅收到的 RETAIN 位不足以拒绝实时 retained 发布，必须在真实 broker 上验证订阅选项。

### D5. 持久化使用 bbolt

设备记录、Device Tree JSON、Protocol 索引、`profile_revision`、最后在线时间写入单个 bbolt 文件（默认 `data/registry.db`）。选择 bbolt 的原因：纯 Go、无 cgo、单文件、事务原子性足够；设备数量和写入频率都很低。备选 SQLite 功能更强但引入 SQL 迁移和驱动选择成本，MVP 不需要。

Device Tree 以 JSON blob 存储，旁边存校验和与 schema 版本；单条缓存损坏时忽略该记录并重新拉取。整个 DB 无法打开时应保留损坏文件作诊断，在能够安全隔离并新建存储时重建，否则明确报错，不静默删除用户文件。树与索引同一事务提交，提交成功后才发布新内存快照；失败保留旧快照并标记不可调用。

注册表把连接状态（discovered/online/offline/expired）、`connection_confirmed` 和描述状态（missing/fresh/stale/invalid）分开。online status 含每次 MQTT CONNECT 的新 connection_id；retained online 只是线索。服务端启动、broker 重连或设备连接代次变化后必须 describe 确认，才能恢复动态工具；broker 断连立即撤下动态工具。即便没有 UDP 发现记录，也要根据合法 status 建档。旧连接、旧刷新代次的响应不能覆盖新快照。

连接状态机区分最后确认连接、候选和最近 64 项已退役 ID 集合；A 被 B 替代后，集合中的 A online/offline/响应全部忽略。UUID 无序，未识别的新 ID 只进入候选；describe 确认且快照提交成功才切换。候选失败可在后台用单 RPC 槽重新探测最后确认的 B，成功恢复，不能无休止重试 A。退役集合淘汰/重启后不假定历史全知，使用同一候选确认规则；后台回退不扩展 MCP 当前刷新 15 秒预算。status 按设备串行处理，旧 broker 连接代次的回调忽略。空 retained status 使所有刷新代次失效，迟到 describe 不能复活已清除状态。设备在首次 SUBACK 后公告 online，并每 30–35 秒按 mqtt-messaging 重发同连接状态（PUBACK 最多 5 秒，失败进入重连）；下线时先停公告。正常 fresh 的重复公告不刷新，清除后下一次公告建立新确认代次，describe 校验提交后恢复，所以删除状态不是永久禁用机制。

MVP 同一 connection_id 的 model/firmware_version/profile_revision 和完整树不变；新固件/配置启动以新 connection_id 公告，可包含回滚。服务端保存首次 online 的基线，发现同连接版本或描述矛盾就标为 invalid、撤工具并使在途刷新失效，不把旧结果提交成 fresh；该异常连接在当前 broker 代次内必须换新 connection_id 才可恢复，不与正常清除后同 ID 恢复混淆。周期公告不重置失败退避或请求截止时间。

### D6. UDP 发现协议使用 JSON 明文

发现端口固定 UDP 37020，服务端绑定 `0.0.0.0:37020`。设备向 `255.255.255.255:37020` 以及所在子网的定向广播地址发送请求；服务端单播回源地址和源端口。

请求：

```json
{"v":1,"type":"discover","device_id":"bl616cl-a1b2c3d4e5f6","model":"bl616cl-board","fw":"0.1.0","nonce":"9f2c5b8e1d7a4069b3e8c1f2056a7d90"}
```

响应：

```json
{"v":1,"type":"offer","nonce":"9f2c5b8e1d7a4069b3e8c1f2056a7d90","broker":{"host":"192.168.1.10","port":1883,"mqtt_version":5,"username":"","password":"","ttl":3600}}
```

选择 JSON 而不是 CBOR/Protobuf：报文小于 512 字节，JSON 可直接用 `tcpdump`/日志排查；固件侧复用 cJSON。nonce 只用于匹配请求与响应，不提供认证；MVP 的欺骗风险由“同一可信局域网”约束承担。

`discovery.advertise_host` 必须是设备可达的 broker 地址，可以与 Go 进程地址不同。未配置且只有一个候选 LAN IPv4 地址时允许自动探测，多网卡有歧义则禁用发现并报告配置错误。数据报上限 512 字节，nonce 每次发送更新，成功采用后不再接受该会话响应。前 30 次发送间隔为 2 秒 + [0,250]ms 非负抖动，此后 30 秒 + 同样抖动；旧 2 秒窗口关闭前不发新 nonce。完整字段约束和限流见 device-discovery spec。

ttl 运行期间使用单调时钟；没有可靠跨重启时钟时冷启动必须重发现，不能将重启视为配置续期。配置失效只标记不可使用，新配置原子落盘前不破坏旧副本；后台续租从每次接收有效 offer 后 ttl 的 80% 加 [0,5% ttl] 抖动开始，相同连接参数在旧截止前完成原子提交并在运行时生效才无缝更新租约，不重连、不改变 connection_id；配置变化才保存后有界排空/下线并切换。续租失败或写入失败不延长旧截止时间，旧租约有效期间继续服务；到期仍未生效则主动下线重发现。提交完成与到期在同一状态机裁决，每次接收操作/处理提交均检查单调时钟，now >= old_expires_at 时到期优先，即使到期 timer 延迟也不得放行；写任务不能阻塞到期处理。旧租约已到期/下线后，迟到提交不得复活旧连接；仅当前网络/配置代次且新租约仍有效时，关闭旧连接后用新 connection_id 连接。新截止始终为 offer 接收时刻 + ttl，不从提交完成时重新计时，迟到写也不能覆盖新配置或 factory_reset。对比连接参数包括 host/port/mqtt_version/凭据而不含 ttl；DNS host 比较大小写规范化，不以成功 MQTT 流量自动续期。合法字段组合还必须满足 offer 总长 512 字节，服务端启动校验，不能只检查各字段长度。服务端首期原生运行在设备同一可广播子网，broker 可以容器化；服务端 Docker bridge 的 UDP 端口映射不等于具备局域网广播接收能力。

### D7. 固件实现最小 MQTT v5 客户端

先核实固定 SDK 的 MQTT 能力，并对可复用的嵌入式 MQTT v5 实现做许可证、内存和 lwIP 接口评估。`firmware/common/mqtt5_client/` 保持公共协议层，并经 `firmware/platform/` 的网络后端连接 SDK；若无合适实现，再基于 lwIP BSD socket 实现受限 v5 子集，而不是未评估就承诺自研完整客户端：

- CONNECT/CONNACK，带 clean start、keepalive、Will（retained `offline`）。
- SUBSCRIBE/SUBACK，以 QoS 1、RAP=1、Retain Handling=2 订阅自己的请求主题，拒绝实时 retained 控制发布且不接收历史 retained 请求。
- PUBLISH/PUBACK，QoS 1；解析 Response Topic 和 Correlation Data 属性。
- PINGREQ/PINGRESP 与 DISCONNECT。

不主动使用 QoS 2、共享订阅、AUTH、主题别名，但必须正确解析或安全跳过 broker 合法属性，检查 reason code、长度和包标识符，覆盖 TCP 分片/粘包/部分写。请求和 invoke 响应 JSON 上限均为 4096 字节；树上限 65536 字节、describe 响应含信封上限 69632 字节、完整 MQTT 控制报文上限 73728 字节。不能把树大小当成整个报文大小，也不固定宣称 2 KiB/8 KiB 缓冲足够。

先测量 WiFi/lwIP、任务栈、cJSON、描述、收发和去重的总内存，再决定分片传输或复用不可变描述缓存；不引入新的应用层分块 RPC。网络任务不能被 Handler 阻塞。Handler 内部驱动截止时间不超过 3 秒，为 5 秒 RPC 等待留出余量。去重缓存至少 8 项，先预留再执行，不驱逐未过期项，内存不足返回 busy；describe 缓存可引用同一份不可变树。

MQTT 客户端 ID 直接使用设备 ID，`clean_start=true`、Session Expiry 为 0；设备离线期间不排队命令，重连后由服务端重新拉取描述。

### D8. EDK2 风格 Protocol/Interface/Handler 注册表

固件用静态表把每个 Protocol 实现注册为：

```c
typedef struct {
    const char *node_id;
    const char *parent_node_id;
    const void *board_config;
    void *instance_data;
} iot_instance_ctx_t;

typedef struct {
    const char *name;
    const char *description;
    const char *parameters_schema_json;
    const char *result_schema_json; /* optional */
    bool read_only;
    bool idempotent;
    int (*handler)(const iot_instance_ctx_t *ctx,
                   const cJSON *args, cJSON **result);
} iot_interface_fn_t;

typedef struct {
    const char *name; /* Interface 以名称寻址，不另设 UUID */
    const iot_interface_fn_t *functions;
    size_t function_count;
} iot_interface_t;

typedef struct {
    const char *uuid;
    const char *name;
    const iot_interface_t *interfaces;
    size_t interface_count;
} iot_protocol_t;
```

- 每个节点是一个实例，节点可以绑定一个或多个 Protocol；同一 Protocol UUID 可以有多个实例，用 `node_id` 区分。
- Handler 获得当前节点实例上下文，不能用全局变量混淆同 UUID 的多个实例。args 由调用方拥有、仅借用；result 由分发层释放。公共 helper 与服务端采用同一 schema 子集；内部组合调用检测环并限制深度 8，避免锁重入死锁。
- Handler 可通过注册表按 `node_id + uuid` 查找其他 Protocol。OLED Handler 在内部查找父 I2C 节点并调用 I2C 函数，实现嵌套组合。
- 注册是编译期静态的；MVP 不加载动态原生代码，也不允许运行时新增 Protocol。
- UUID 使用 RFC 4122 小写格式，项目维护 `docs/protocols/uuids.md` 作为 UUID → Protocol 的分配表；服务端不要求认识 UUID 语义，只按声明索引。
- 协议表同时驱动 Device Tree 生成和 MQTT 分发，避免“文档声明了但代码没实现”的漂移；主机侧测试对比生成的树和注册表。

备选方案是为每个函数生成独立的 C RPC stub，但会增加代码生成器和构建复杂度；统一 cJSON 参数在 MVP 的函数数量和参数规模下更简单。

### D9. Device Tree 生成与 Protocol 索引

`firmware/common/device_model/` 用静态节点数组描述板级实例树，节点 ID 在同一板级实例重启和兼容升级后稳定。Device Tree 在线版本为 `schema_version: "1.0"`，“v0”仅为产品阶段称呼。profile_revision 正常发布递增，新连接的任意版本变化（含回滚）都需确认，版本不变的重连也重新确认；同连接的版本变化不是热更新，按异常隔离规则处理。

参数和结果使用 device-profile spec 明确定义的 JSON Schema 2020-12 子集；对象禁止额外属性，参数省略等价于空对象，禁止远程 `$ref` 和不支持的校验关键字。未知元数据保留不等于忽略未知 schema 约束。整棵树递归检测常见敏感键，日志不打印秘密值；未知 UUID 按声明处理，但任何描述文本都是不可信数据。

服务端校验通过后建立索引：

```
(device_id, node_id, protocol_uuid, interface, function) -> signature + description + path
```

索引与 Device Tree 缓存同事务写入 bbolt，并触发 `device_profile_updated` 事件。MCP 网关只从索引生成工具，避免每次调用都遍历整棵树。未知节点类型、未知字段和未知 Protocol UUID 都保留在资源中，但只有包含完整 Interface/函数声明的条目才生成工具。

### D10. MQTT RPC 契约：describe 与 protocol.invoke

`device.describe` 保持不变，返回完整 Device Tree。新增：

```json
{
  "id": "968f5213-1640-4e2e-a74e-a536084d91b1",
  "connection_id": "a2a6c5a8-61df-49d5-b5ac-325b3a10df04",
  "method": "protocol.invoke",
  "params": {
    "node_id": "i2c0.oled",
    "protocol_uuid": "0f5d2f6e-3c7b-4c82-a2ad-bcaec2ea3f12",
    "interface": "oled",
    "function": "display_text",
    "args": {"text": "hello"}
  }
}
```

示例 UUID 仅为协议样例，不是已经确定的硬件 Protocol 分配。成功响应包含相同 `id`、`connection_id`、`ok: true` 和 `result`；失败响应为 `ok: false` 与 `error.code`、`error.message`，result/error 不得同时存在。状态 JSON 必须包含 v、device_id、state、model、firmware_version、profile_revision、connection_id；每次 CONNECT 生成新的 connection_id，Will 与该连接保持一致。状态最大 1024 字节，字段/主题不一致不得确认在线；空 retained status 只表示状态被清除，不删除设备缓存。设备只接受自己的响应主题和当前连接请求。错误码：

- `unknown_node`、`unknown_protocol`、`unknown_interface`、`unknown_function`
- `invalid_params`
- `execution_failed`

服务端发送前用索引验证组合和参数：组合不存在为 `unsupported_operation`，参数不合法为 `invalid_params`，均不发送 MQTT。设备再次校验。超时、重试和去重遵循 D4 与 mqtt-messaging spec，尤其禁止以新 ID 自动重试有副作用操作。固件只提供同步短调用，不引入异步任务模型；超时/取消、执行失败或返回结果校验失败均不保证已经发生的副作用被撤销。

`device.describe` 的 params 省略或为空对象，成功 result **直接是 Device Tree**，不是 `{ "device_tree": ... }`。最小无外设设备的完整交互样例（只说明线上格式，不代表已实现）：

```json
{
  "id": "968f5213-1640-4e2e-a74e-a536084d91b1",
  "connection_id": "a2a6c5a8-61df-49d5-b5ac-325b3a10df04",
  "method": "device.describe"
}
```

```json
{
  "id": "968f5213-1640-4e2e-a74e-a536084d91b1",
  "connection_id": "a2a6c5a8-61df-49d5-b5ac-325b3a10df04",
  "ok": true,
  "result": {
    "schema_version": "1.0",
    "device": {
      "id": "bl616cl-a1b2c3d4e5f6",
      "model": "bl616cl-board",
      "firmware_version": "0.1.0",
      "profile_revision": 1
    },
    "root": {"id": "board", "type": "board", "name": "Demo board"}
  }
}
```

对应 status（发布前已经收到请求订阅的成功 SUBACK）：

```json
{"v":1,"device_id":"bl616cl-a1b2c3d4e5f6","state":"online","model":"bl616cl-board","firmware_version":"0.1.0","profile_revision":1,"connection_id":"a2a6c5a8-61df-49d5-b5ac-325b3a10df04"}
```

能可靠关联且信封合法的响应若 result 不符合方法/声明 schema，立即结束为 invalid_device_response；身份不匹配、retained 或无法解析合法信封的响应丢弃继续原等待。不能把 result schema 失败与不可关联消息混为一谈。

所有消息使用严格 UTF-8 JSON，拒绝重复键/尾随非空白内容以及对象键和字符串中解码后的 U+0000（在 cJSON 转换前检查，正确区分字面反斜杠文本）；JSON 容器深度最多 64，树与 schema 深度各自最多 8，不能混用。schema 字符串长度按 Unicode 码点，传输大小按 UTF-8 字节。Go/C 共享同一正反例输入，特别验证重复键时两端都不执行操作。

### D11. MCP 工具映射

基础工具：

- `list_devices`、`get_device_tree`、`list_protocols`、`invoke_protocol`、`refresh_device_profile`。

动态工具按协议函数生成，名称格式：

```
iot__<device>__<node>__<protocol>__<interface>__<function>
```

各段规范化为小写字母、数字和下划线，可读前缀最多 45 字符；所有名称均追加 `__` 和寻址元组 SHA-256 前 16 位十六进制摘要，总长不超过 64。摘要输入为 `[device_id,node_id,protocol_uuid,interface,function]` 紧凑 UTF-8 JSON，不依赖显示名称或注册顺序。完整名称碰撞则拒绝发布碰撞项并报警，不能覆盖路由。工具描述由“设备型号 + 节点路径 + Protocol 名称 + 函数描述”组成，输入 schema 来自已验证的 parameters。

默认每设备最多发布 64 个、全局最多 256 个动态工具，按完整寻址元组排序选取；超额条目仍可经 list_protocols 和 invoke_protocol 使用，并在摘要里标明截断。离线、broker 断连或描述失效时撤下动态工具；stale 的通用调用先合并刷新，再按新索引校验，整个调用预算最多 20 秒。执行失败用 MCP isError 结果传递结构化错误，不能伪装成成功文本。

`invoke_protocol` 是通用逃生口，供大模型在动态工具过多或需要按 UUID 调用时使用。两者最终都走同一校验和 MQTT 路由，避免行为分叉。设备上下线、描述失效或新连接的描述快照提交导致工具集合变化时，网关增删工具并发送 `notifications/tools/list_changed`；资源列表变化发送 `notifications/resources/list_changed`。

### D12. 配置与部署

服务端配置优先级：命令行 flag > 环境变量 > `server/config.yaml` > 默认值。关键配置项：

```yaml
discovery:
  listen: "0.0.0.0:37020"
  advertise_host: ""        # 仅单一候选 LAN 地址时自动探测；否则必须显式配置
  broker_port: 1883
  broker_ttl_seconds: 3600
mqtt:
  broker_url: "tcp://127.0.0.1:1883"
  client_id: "agentic-iot-server"
mcp:
  listen: "127.0.0.1:8080"
  path: "/mcp"
  allowed_hosts: ["127.0.0.1", "localhost"]
  allowed_origins: []        # 无 Origin 的本地非浏览器客户端不受此项限制
  dynamic_tools_per_device: 64
  dynamic_tools_total: 256
store:
  path: "data/registry.db"
log:
  level: "info"
```

`broker/` 提供 mosquitto 2.x 配置：`listener 1883`、`allow_anonymous true`、`persistence true`、`persistence_location /mosquitto/data/`、`message_size_limit 131072`。实现时固定经过测试的镜像版本或 digest，不使用 latest。compose 对外只绑定部署者指定的可信 LAN 地址；若使用 Go 默认 `tcp://127.0.0.1:1883`，还需显式提供 loopback 端口映射，否则必须将 mqtt.broker_url 覆盖为该 LAN broker 地址，不能假定仅绑定 LAN 的端口也能从 loopback 访问。发现下发的始终是设备可达 LAN 地址而非 loopback。挂载持久化卷，设置 `restart: unless-stopped`；文档记录防火墙规则和持久化正常重启/崩溃的区别。服务端不管理 broker 生命周期。

`/healthz` 只反映进程存活；`/readyz` 在存储可用、MQTT 已连接且 status/resp 订阅成功后返回 200，否则 503，并提供不含凭据的原因；不以“至少有一台设备在线”为就绪条件。首次配置和依赖版本记录进部署文档，SSID/密码不得提交到仓库。factory_reset 只清 NVS 覆盖和 broker 配置，不能清除编译进固件的默认值；无默认值时用串口恢复 WiFi。

### D13. 测试策略

- Go 单元测试：发现报文编解码与校验、MQTT 关联/超时状态机、注册表状态转换、Device Tree 与 Protocol/Interface 校验、Protocol 索引、工具名生成与冲突处理、`protocol.invoke` 参数校验。
- Go 集成测试：testcontainers 或本地 compose 启动 mosquitto，用假设备跑通“发现 → 上线 → describe → list_protocols → invoke_protocol”。
- 固件主机侧测试：MQTT v5 编解码、Protocol 注册表分发、cJSON 参数提取、Device Tree 生成、嵌套 Protocol 组合、请求去重缓存。
- 硬件联调：BL616CL 与 BL618 各至少一块已确认板型 + 各自真实 I2C/OLED 或等效外设 + mosquitto + 服务端 + MCP Inspector/LLM 客户端，分别验证并同时在线测试身份/路由/状态隔离。

MCP 网关本身不实现模型推理。先用 Inspector/官方 MCP 客户端完成无模型验收；自然语言演示另依赖支持 Streamable HTTP 的模型客户端及其账户/凭据/网络，不是 Go 网关或固件的依赖，也不把任何模型密钥放入固件。

测试分层：无硬件单元测试 → 真实 broker + 假设备 → 固件主机侧协议测试 → 实板联调。测试矩阵必须包含响应丢失但操作已执行、重复请求执行中、不同负载同 ID、retained 假在线、broker 断连、旧连接迟到响应、固件回滚、64 KiB 树、存储事务失败、无 RTC 冷启动和无 WiFi 默认值恢复出厂。主机 C 测试使用内存/未定义行为检查和有界畸形报文测试；Go 使用 race 检测。真实硬件不在场时，相关验收必须标为未验证而非通过。第二轮回归增加状态清除后周期公告恢复、同连接版本冲突使旧刷新失效、invoke 占槽时刷新调度和端到端预算、续租提交跨越/恰逢截止及旧写代次失效、自定义 BOARD 同名/解析路径错误。新增回归覆盖已退役 online/候选失败恢复、实时清除 status 与迟到 describe、已关联坏结果立即失败、U+0000 与字面反斜杠、无缝续租/写失败/到期以及 1.9 秒响应窗口；两芯片独立跑相应硬件子集。共享样例由 tasks 0.1 建立，包含完整成功/失败信封、status、边界与畸形 JSON、Unicode schema 语义；规范中的 JSON fenced block 必须可解析，但仅有语法检查不等于 Go/C 实现已符合契约。

### D14. 为后续 WASM 预留的调用面（本 change 不实现）

协议分发函数在固件内部保持单一入口：

```c
int iot_protocol_call(const char *node_id, const char *uuid,
                      const char *interface, const char *function,
                      const cJSON *args, cJSON **result);
```

MQTT protocol.invoke 和内部 Handler 组合调用同一入口，由分发层解析节点实例上下文并管理返回值生命周期。后续 WAMR host import 可复用该原生调用面，但 WASM 线性内存与 cJSON 指针不可直接互传，仍需单独设计 ABI 编解码、能力权限、执行配额与模块生命周期；不能承诺“增加两个方法即完成 WASM”。本 change 不链接 WAMR、不定义 WASM 模块格式、不实现模块存储或执行。

### D15. 公共固件、芯片适配与板级 target

规范按三层组织：`firmware-platform` 是所有固件共享行为和移植契约的唯一来源；`bl616-firmware` 保留名称但只规定 BL616CL 差异；`bl618-firmware` 规定原始 BL618 差异。原 BL616CL 的运行时要求整体移入公共规范，没有删除 WiFi、配置恢复、MQTT、Protocol、去重或内存验收要求。目标板待确认不阻塞本次规范完善，也不阻塞无硬件 G1。

规划目录（实现任务尚未开始）：

```
firmware/
  common/                 发现/续租、MQTT v5、RPC/去重、schema、Protocol、Device Tree
  platform/include/       时钟/随机/身份/网络/存储/任务/串口/总线的行为接口
  platform/bouffalo/      SDK 公共后端和必要的 bl616cl / bl616 系列适配
  platform/host/          主机测试替身，不是第三种芯片实现
  boards/                 应用 BOARD 的引脚/分区/驱动绑定（按 SDK 搜索规则暴露）
  targets/                已确认板型的 target manifests 与节点配置
  tests/                  公共契约与平台故障测试
```

不为每颗芯片复制 MQTT/发现/Protocol 状态机；若 lwIP/RTOS 后端已兼容就直接复用，不另造通用 RTOS。OLED 复用公共 Handler，I2C Protocol Handler 通过板级实例绑定 SDK 驱动。实例上下文和错误/超时/所有权规则不随芯片变化；SDK 类型和寄存器头留在平台层，公共模块可用 host 后端单独测试。

| 真实芯片 chip | device_id 前缀 | SDK CHIP | BOARD 状态 |
| --- | --- | --- | --- |
| BL616CL (`bl616cl`) | `bl616cl-` | `bl616cl` | `bl616cl_ave_v10`/`bl616cldk` 为候选，按实板确认 |
| 原始 BL618 (`bl618`) | `bl618-` | `bl616` | `bl616dk` 为可评估基线，或应用自定义 BOARD；尚未确认实板 |
| BL618DG/BL618M | 未分配 | 不属于本期 BL618 target | 后续独立 change，不从名字相近推断兼容 |

两种首期设备使用原厂稳定 48 位 MAC（BL616CL 也可用经过唯一性验证的等价硬件映射），不使用随机化 MAC、不随 BOARD/SDK 别名变化；服务端不解析前缀推断能力。target manifest 分开记录真实 chip/board 和 sdk_chip/sdk_board，另记 SDK commit、工具链、内存、分区、引脚/节点配置。构建由 `make firmware TARGET=<已确认 target_id>` 选择，未给 TARGET、未知或 planned 条目拒绝；每 target 独立构建目录/缓存/镜像及 manifest，烧录也核对目标。SDK 的 bsp/board/CMakeLists.txt 先找内置 BOARD，未命中才查应用目录；应用自定义 BOARD 必须采用独立名称（如 agentic_bl618_<board>），不得以同名 bl616dk 覆盖。应用构建映射自定义目录到 SDK 搜索路径，在配置阶段比较实际 Select board 规范化路径与 profile 期望；同名冲突或路径不符立即失败。镜像 manifest 记录实际内置/应用来源和项目/SDK 相对路径，不只记录 BOARD 字符串，不直接改 SDK 子模块。

Device Tree 可选扩展 `device.hardware`（vendor/chip/board/firmware_target，可选 board_revision）在本期两种固件中必填，服务端仍接受没有此字段的既有 schema 1.0 设备。SDK 参数不是硬件事实，BL618 不能上报为 BL616。相同接口语义跨芯片复用 Protocol UUID，不兼容接口使用新协议身份；实际可调用能力始终由描述决定，没有外设不生成对应节点。

后续芯片清单：硬件身份 → SDK/工具链锁定 → 平台适配 → 板级 profile → 资源预算 → 共享契约/构建 → 实板与混合测试。遵守已有线上契约不需要改 Go 网关；若缺少广播/WiFi 或达不到当前最大报文/去重要求，先另立变更，不能给弱目标悄悄降低上限。逐目标记录 planned/build-verified/hardware-verified，不将同系列、同 SDK 或单纯编译成功等同实板可用。

## Risks / Trade-offs

- [Protocol 表与 Device Tree 漂移] → 两者由同一静态表生成，主机侧测试对比声明与生成结果。
- [UUID 冲突或语义不明] → 维护 `docs/protocols/uuids.md` 分配表；服务端只按声明索引，不依赖内置语义。
- [嵌套 Protocol 组合复杂] → 用 OLED → I2C 作为端到端样板；Handler 只通过注册表查找依赖，不直接持有全局变量。
- [MCP 工具数量爆炸] → 结构上限不等于可用工具预算；默认每设备 64、全局 256，超额函数走同等校验的 invoke_protocol。
- [JSON 参数开销] → 函数参数使用有界缓冲和小型 cJSON 文档；Device Tree 仍限制 64 KiB。
- [WASM 不在本 change] → 复杂流程暂不能下发；本 change 只保证简单控制可直调，WASM 作为独立后续 change，避免范围失控。
- [SDK 或 MQTT v5 移植不满足要求] → 先做固定 commit 与客户端的可行性验证，再选择复用或受限实现；不得用未经测试的库大小推断资源适配。
- [UDP 广播被网络隔离] → 文档明确要求设备与服务端处于同一可广播子网；提供 `advertise_host` 和后续手动配置 broker 的扩展点。MVP 不做跨网段发现。
- [明文协议可被局域网内伪造] → 明确 MVP 仅用于可信网络；nonce 只防串话；后续 change 增加 PSK/HMAC、MQTT 用户名密码和 TLS。
- [Device Tree 格式后续细化导致返工] → 信封、节点 ID、Protocol/Interface 声明先固定；未知字段和未知 UUID 必须被容忍。
- [bbolt 单写者或提交失败] → 非 MQTT 回调线程执行持久化；提交成功才发布树与索引快照，失败保留旧快照但禁止操作。
- [固件内存不足] → 区分树、RPC 信封与 MQTT 报文上限，测量峰值堆/栈；不满足预算时先修订 spec，不静默截断。
- [待处理操作中途丢失] → 不恢复旧 invoke；进程仍可应答时返回 outcome_unknown，进程直接崩溃时客户端只能观察连接失败，文档必须警告不要自动重试。
- [匿名控制边界] → 仅做低风险 OLED/传感器演示；身份防伪、授权和高风险执行器属于后续安全设计，不能用 nonce 或 MCP annotations 替代。

## Migration Plan

绿地项目，无存量数据迁移。

1. 启动 `broker/` 的 mosquitto，确认 1883 可连接且 retained 消息跨重启保留。
2. 配置 `discovery.advertise_host` 为 NAS/树莓派局域网 IP，启动 Go 服务端，确认 `/mcp` 可初始化。
3. 按已确认 TARGET 分别构建并烧录 BL616CL 和 BL618，核对 SDK/分区/板型，不复用错误镜像；确认两种设备完成发现、上线并出现在 MCP `tools/list`。
4. 用 MCP Inspector 调用 `list_devices`、`list_protocols`、`get_device_tree`，再通过 `invoke_protocol` 调用 OLED 或传感器函数。

回滚：停止服务端和 broker 容器，保留数据卷和 registry.db；按已验证烧录流程恢复旧固件，factory_reset 本身不负责回滚镜像。备份后方可由操作者选择清理缓存，不能把删除数据库作为默认回滚步骤。固件 revision 降低也会触发服务端刷新。

## Dependencies and Delivery Gates

| 阶段 | 交付物 | 前置依赖 / 放行条件 |
| --- | --- | --- |
| G0a 服务端契约 | 共享 JSON 正反例、Go/MCP/MQTT 依赖最小验证 | tasks 0.1–0.2；固定兼容版本与线上契约，不依赖硬件 |
| G0b 固件可行性（逐目标） | BL616CL/BL618 各自板级配置、SDK 构建、客户端选型与预算 | SDK 同步 task 0.6 已完成；tasks 0.3–0.5 仍按芯片/板型分别放行，固定 SDK 可构建、外设/引脚确认、v5 和资源预算通过；可与 G1 并行 |
| G1 无硬件闭环 | broker + Go + 假设备 + MCP 调用 | G0a；Docker/Compose、mosquitto 客户端、Go 可用；不依赖 G0b、实板或模型账户 |
| G2 固件接入（逐目标） | 两种芯片分别 WiFi → UDP → MQTT → describe | G0b + G1；CMake、匹配 SDK 的 RISC-V 工具链、烧录工具、串口权限、2.4 GHz 网络（按板卡能力确认） |
| G3 真外设闭环 | 双芯片分别及混合运行 I2C → OLED 或等效外设，动态/通用调用一致 | G2 + 正确接线/电压/地址和可观察真实结果；完成副作用、掉线和重复消息测试 |
| G4 交付 | 部署手册、版本与测试证据、自然语言演示（若选择） | G1 及 BL616CL/BL618 各自 G2/G3 和混合验收通过，缺板标未完成；模型客户端与凭据仅供最后演示；严格 OpenSpec 校验后再考虑归档 |

Go 模块固定 `github.com/modelcontextprotocol/go-sdk`、`github.com/eclipse/paho.golang`、`go.etcd.io/bbolt` 和选定 YAML 解析包的兼容版本，日志采用标准库 slog，不额外引入日志框架。版本落在 go.mod/go.sum，不在规划阶段虚构版本号。固件依赖 SDK WiFi、lwIP、cJSON、非易失配置存储/flash、板级 I2C 驱动以及最终选择的 MQTT v5 实现。本文 NVS 指非易失配置抽象，不假定 SDK 存在同名组件；优先评估已确认存在的 littlefs KV/easyflash 兼容接口，原子提交仍由配置层验证。WAMR 不是当前依赖。

只完成规范评审不能归档。SDK 同步已完成，但应用实现、目标构建、协议测试和硬件交付任务仍未完成；实施时应按上述 gate 推进，而不是按目录一次性铺开全部代码。

## Open Questions

- BL616CL 实板是 `bl616cl_ave_v10` 还是 `bl616cldk`（或应用自定义 BOARD）？BL618 的具体开发板/模块及 board revision 是什么，能否采用 `bl616dk` 基线？分别确认引脚、供电、flash/PSRAM、分区、按键与外设，未确认前不生成可构建 target。
- WiFi 默认凭据是否编译进固件由部署选择；规范固定为 NVS 覆盖优先，并必须有无默认凭据时的串口恢复入口，禁止将真实凭据提交仓库。
- 首批 Protocol UUID、Interface 函数签名与外设地址需在 G0b 冻结；文中的 UUID 是格式示例，不替代正式分配表。
- 首个验收客户端默认 MCP Inspector 或官方客户端；是否需要跨机器访问及采用何种已鉴权代理，部署时确认，不能因此跳过 Host/Origin 校验。
- SDK 子模块同步、固定提交核对和组件静态核查已完成；实际最小构建、目标板 RAM/flash 预算及 MQTT v5 实现选型仍待验证，是固件实现的前置 gate。
- WASM 后续 change 需要决定 WAMR 解释器/AOT 取舍、模块格式、分块传输、宿主 ABI 和资源限制；本 change 不实现这些内容。
