# Design

## Context

参见 `proposal.md` 的 Why 与 What Changes。当前仓库是空项目，只有 Bouffalo SDK 子模块（`bouffalo/bouffalo_sdk`）和 OpenSpec 目录；BL616CL 板级支持已存在于 SDK 的 `bsp/board/bl616cl_ave_v10` 与 `bsp/board/bl616cldk`，SDK 自带 WiFi、lwIP、cJSON 和 MQTT-C 组件，并自带 WAMR 2.4.4 与 `examples/shell/wasm_cmd` 示例。

关键约束：

- 服务端运行在 NAS 或树莓派上，需要长期在线、单进程、易部署；局域网内只允许明文通信。
- BL616CL 资源受限，固件需要控制静态内存和堆分配，且 SDK 自带的 MQTT-C 仅支持 MQTT 3.1.1，不满足 MQTT v5 Request-Reply 要求。
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
- 端到端可本地复现：docker-compose 启动 broker，Go 服务端一条命令启动，固件可烧录到 BL616CL 开发板。

**Non-Goals:**

- 不实现 TLS、MQTT ACL、设备凭证签发、OTA 升级、云端接入和多服务端高可用。
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
firmware/    BL616CL 应用工程，CMake 引用 bouffalo/bouffalo_sdk
broker/      mosquitto.conf + docker-compose.yml
scripts/     本地联调、固件构建与烧录辅助脚本
docs/        部署、MCP、协议与 UUID 分配说明
```

根目录提供 `Makefile` 聚合 `make broker-up`、`make server`、`make firmware`、`make test`。三种构建系统互不侵入，避免把 Go 工具链塞进固件 CMake，也避免把 SDK 子模块变成 Go 模块的一部分。备选方案是单一 CMake 驱动全部组件，但 Go 服务端与嵌入式工具链差异过大，放弃。

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

选择 `github.com/modelcontextprotocol/go-sdk` 的 `mcp` 包实现 Streamable HTTP，端点固定为 `/mcp`，默认监听 `0.0.0.0:8080`。选择官方 SDK 是为了跟随 MCP 协议演进；若实现期发现版本不兼容，抽象层只暴露 `AddTool`、`RemoveTool`、`AddResource` 和通知接口，可替换为 `mark3labs/mcp-go`。

不使用 stdio 作为主传输：服务端部署在 NAS，需要被多台机器上的 LLM 客户端访问。stdio 可在后续作为调试模式补充。

### D4. MQTT 客户端使用 paho.golang

服务端使用 `github.com/eclipse/paho.golang/autopaho`（MQTT v5）而不是 `eclipse/paho.mqtt.golang`，因为后者只支持 3.1.1，无法携带 Response Topic 和 Correlation Data。`autopaho` 负责重连和会话管理，请求层在其上维护 `correlation_id -> pending request` 映射。

待处理请求使用带超时的 `sync.Map` 或互斥保护的 map；每个请求 5 秒超时、最多 3 次尝试，每次生成新的 UUID 作为请求 ID 和 Correlation Data。响应主题统一为 `iot/v1/devices/{device_id}/resp`，服务端订阅 `iot/v1/devices/+/resp`。

### D5. 持久化使用 bbolt

设备记录、Device Tree JSON、Protocol 索引、`profile_revision`、最后在线时间写入单个 bbolt 文件（默认 `data/registry.db`）。选择 bbolt 的原因：纯 Go、无 cgo、单文件、事务原子性足够；设备数量和写入频率都很低。备选 SQLite 功能更强但引入 SQL 迁移和驱动选择成本，MVP 不需要。

Device Tree 以 JSON blob 存储，旁边存校验和与 schema 版本；读取时校验失败直接丢弃，符合 `device-registry` 的缓存恢复要求。Protocol 索引与 Device Tree 在同一事务更新，避免 MCP 看到半更新状态。

### D6. UDP 发现协议使用 JSON 明文

发现端口固定 UDP 37020，服务端绑定 `0.0.0.0:37020`。设备向 `255.255.255.255:37020` 以及所在子网的定向广播地址发送请求；服务端单播回源地址和源端口。

请求：

```json
{"v":1,"type":"discover","device_id":"bl616cl-a1b2c3d4e5f6","model":"bl616cl-board","fw":"0.1.0","nonce":"9f2c..."}
```

响应：

```json
{"v":1,"type":"offer","nonce":"9f2c...","broker":{"host":"192.168.1.10","port":1883,"mqtt_version":5,"username":"","password":"","ttl":3600}}
```

选择 JSON 而不是 CBOR/Protobuf：报文小于 512 字节，JSON 可直接用 `tcpdump`/日志排查；固件侧复用 cJSON。nonce 只用于匹配请求与响应，不提供认证；MVP 的欺骗风险由“同一可信局域网”约束承担。

服务端的 `broker.host` 不是自动猜测，而是配置项 `discovery.advertise_host`；未配置时取服务端默认路由所在网卡的 IPv4 地址，并在启动日志中打印实际值。这样在多网卡 NAS 上不会下发错误地址。

### D7. 固件实现最小 MQTT v5 客户端

SDK 的 `components/net/lib/mqtt` 是 MQTT-C（3.1.1），没有 MQTT v5 属性。固件新增 `firmware/components/mqtt5_client/`，基于 lwIP BSD socket 实现 MVP 所需的 v5 子集：

- CONNECT/CONNACK，带 clean start、keepalive、Will（retained `offline`）。
- SUBSCRIBE/SUBACK，订阅自己的请求主题。
- PUBLISH/PUBACK，QoS 1；解析 Response Topic 和 Correlation Data 属性。
- PINGREQ/PINGRESP 与 DISCONNECT。

不实现 QoS 2、共享订阅、AUTH、主题别名。固定发送/接收缓冲（建议 2 KiB 发送、8 KiB 接收，Device Tree 响应更大时单独流式构造或提高上限），避免运行期大块动态分配。选择自实现而不是移植 Paho C：Paho C 体积和内存开销对 BL616CL 偏高，且 MVP 只需要很小子集；自实现便于单元测试和裁剪。风险是协议细节出错，因此用主机侧单元测试覆盖编解码，并与真实 mosquitto 联调。

MQTT 客户端 ID 直接使用设备 ID，`clean_start=true`、Session Expiry 为 0；设备离线期间不排队命令，重连后由服务端重新拉取描述。

### D8. EDK2 风格 Protocol/Interface/Handler 注册表

固件用静态表把每个 Protocol 实现注册为：

```c
typedef struct {
    const char *name;
    int (*handler)(const cJSON *args, cJSON **result);
} iot_interface_fn_t;

typedef struct {
    const char *uuid;          // RFC 4122
    const char *name;
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
- Handler 接收解析后的 cJSON 参数并返回 cJSON 结果；公共 helper 负责按 Device Tree 声明的类型提取字符串、整数、布尔和对象参数。
- Handler 可通过注册表按 `node_id + uuid` 查找其他 Protocol。OLED Handler 在内部查找父 I2C 节点并调用 I2C 函数，实现嵌套组合。
- 注册是编译期静态的；MVP 不加载动态原生代码，也不允许运行时新增 Protocol。
- UUID 使用 RFC 4122 小写格式，项目维护 `docs/protocols/uuids.md` 作为 UUID → Protocol 的分配表；服务端不要求认识 UUID 语义，只按声明索引。
- 协议表同时驱动 Device Tree 生成和 MQTT 分发，避免“文档声明了但代码没实现”的漂移；主机侧测试对比生成的树和注册表。

备选方案是为每个函数生成独立的 C RPC stub，但会增加代码生成器和构建复杂度；统一 cJSON 参数在 MVP 的函数数量和参数规模下更简单。

### D9. Device Tree 生成与 Protocol 索引

`firmware/components/device_model/` 用静态节点数组描述板级实例树：根节点、总线节点、外设节点、节点绑定的 Protocol 和节点属性。启动时由 cJSON 生成 Device Tree，节点顺序由静态数组顺序保证稳定；`profile_revision` 是板级配置头文件中的单调版本号，协议集合变化时递增。

服务端校验通过后建立索引：

```
(device_id, node_id, protocol_uuid, interface, function) -> signature + description + path
```

索引与 Device Tree 缓存同事务写入 bbolt，并触发 `device_profile_updated` 事件。MCP 网关只从索引生成工具，避免每次调用都遍历整棵树。未知节点类型、未知字段和未知 Protocol UUID 都保留在资源中，但只有包含完整 Interface/函数声明的条目才生成工具。

### D10. MQTT RPC 契约：describe 与 protocol.invoke

`device.describe` 保持不变，返回完整 Device Tree。新增：

```json
{
  "id": "uuid",
  "method": "protocol.invoke",
  "params": {
    "node_id": "i2c0.oled",
    "protocol_uuid": "0f5d2f6e-...",
    "interface": "oled",
    "function": "display_text",
    "args": {"text": "hello"}
  }
}
```

成功响应返回 `result`；失败响应返回 `error.code` 和 `error.message`。错误码：

- `unknown_node`、`unknown_protocol`、`unknown_interface`、`unknown_function`
- `invalid_params`
- `execution_failed`

服务端在发送前用 Protocol 索引校验组合和参数；本地校验失败返回 `unsupported_operation`，不发送 MQTT。设备端再次校验，防止服务端索引过期。超时与重试沿用 MQTT 请求层（5 秒、3 次）；本 change 不引入异步任务模型。

### D11. MCP 工具映射

基础工具：

- `list_devices`、`get_device_tree`、`list_protocols`、`invoke_protocol`、`refresh_device_profile`。

动态工具按协议函数生成，名称格式：

```
iot__<device>__<node>__<protocol>__<interface>__<function>
```

各段规范化为小写字母、数字和下划线；总长度超过 64 字符时截断并追加 8 位 SHA-256 前缀，仍冲突时追加完整短哈希。工具描述由“设备型号 + 节点路径 + Protocol 名称 + 函数描述”组成，输入 schema 由 `parameters` 转换。

`invoke_protocol` 是通用逃生口，供大模型在动态工具过多或需要按 UUID 调用时使用。两者最终都走同一校验和 MQTT 路由，避免行为分叉。设备上下线或 `profile_revision` 变化时，网关增删工具并发送 `notifications/tools/list_changed`；资源列表变化发送 `notifications/resources/list_changed`。

### D12. 配置与部署

服务端配置优先级：命令行 flag > 环境变量 > `server/config.yaml` > 默认值。关键配置项：

```yaml
discovery:
  listen: "0.0.0.0:37020"
  advertise_host: ""        # 空则自动探测
  broker_port: 1883
  broker_ttl_seconds: 3600
mqtt:
  broker_url: "tcp://127.0.0.1:1883"
  client_id: "agentic-iot-server"
mcp:
  listen: "0.0.0.0:8080"
  path: "/mcp"
store:
  path: "data/registry.db"
log:
  level: "info"
```

`broker/` 提供 mosquitto 2.x 配置：`listener 1883`、`allow_anonymous true`、`persistence true`、`persistence_location /mosquitto/data/`、`message_size_limit 131072`。compose 暴露 1883，挂载持久化卷，设置 `restart: unless-stopped`。服务端不管理 broker 生命周期。

### D13. 测试策略

- Go 单元测试：发现报文编解码与校验、MQTT 关联/超时状态机、注册表状态转换、Device Tree 与 Protocol/Interface 校验、Protocol 索引、工具名生成与冲突处理、`protocol.invoke` 参数校验。
- Go 集成测试：testcontainers 或本地 compose 启动 mosquitto，用假设备跑通“发现 → 上线 → describe → list_protocols → invoke_protocol”。
- 固件主机侧测试：MQTT v5 编解码、Protocol 注册表分发、cJSON 参数提取、Device Tree 生成、嵌套 Protocol 组合、请求去重缓存。
- 硬件联调：BL616CL 开发板 + I2C OLED（或等效外设）+ mosquitto + 服务端 + MCP Inspector/LLM 客户端，验证真实协议调用。

### D14. 为后续 WASM 预留的调用面（本 change 不实现）

协议分发函数在固件内部保持单一入口：

```c
int iot_protocol_call(const char *node_id, const char *uuid,
                      const char *interface, const char *function,
                      const cJSON *args, cJSON **result);
```

MQTT `protocol.invoke`、固件内部 Handler 组合和将来的 WAMR host import 都调用同一入口。后续 change 只需在该入口上增加 `wasm.install` / `wasm.run` 等 MQTT 方法和 WAMR 宿主函数，不需要改动协议模型。本 change 不链接 WAMR、不定义 WASM 模块格式、不实现模块存储或执行。

## Risks / Trade-offs

- [Protocol 表与 Device Tree 漂移] → 两者由同一静态表生成，主机侧测试对比声明与生成结果。
- [UUID 冲突或语义不明] → 维护 `docs/protocols/uuids.md` 分配表；服务端只按声明索引，不依赖内置语义。
- [嵌套 Protocol 组合复杂] → 用 OLED → I2C 作为端到端样板；Handler 只通过注册表查找依赖，不直接持有全局变量。
- [MCP 工具数量爆炸] → 单设备限制 256 节点、32 Protocol/节点、16 Interface/Protocol、64 函数/Interface；同时提供 `invoke_protocol` 通用工具。
- [JSON 参数开销] → 函数参数使用有界缓冲和小型 cJSON 文档；Device Tree 仍限制 64 KiB。
- [WASM 不在本 change] → 复杂流程暂不能下发；本 change 只保证简单控制可直调，WASM 作为独立后续 change，避免范围失控。
- [SDK 无 MQTT v5] → 自实现最小 v5 客户端，先做编解码单元测试，再用 mosquitto 联调；若工期超预期，退路是移植 Paho C 的 v5 子集并接受更大 flash/RAM 占用。
- [UDP 广播被网络隔离] → 文档明确要求设备与服务端处于同一可广播子网；提供 `advertise_host` 和后续手动配置 broker 的扩展点。MVP 不做跨网段发现。
- [明文协议可被局域网内伪造] → 明确 MVP 仅用于可信网络；nonce 只防串话；后续 change 增加 PSK/HMAC、MQTT 用户名密码和 TLS。
- [Device Tree 格式后续细化导致返工] → 信封、节点 ID、Protocol/Interface 声明先固定；未知字段和未知 UUID 必须被容忍。
- [bbolt 单写者] → 状态变更串行写入，MVP 设备量小可接受；写入失败只记录日志并保留内存状态，不阻塞 MQTT 回调。
- [固件内存不足] → Device Tree 和协议响应使用固定缓冲并限制 64 KiB；超限时返回错误而不是崩溃；在真实板子上测量峰值堆使用。
- [服务端重启丢失待处理请求] → 待处理请求只在内存中，重启后由注册表重新发起 describe；工具调用返回超时，不尝试恢复。

## Migration Plan

绿地项目，无存量数据迁移。

1. 启动 `broker/` 的 mosquitto，确认 1883 可连接且 retained 消息跨重启保留。
2. 配置 `discovery.advertise_host` 为 NAS/树莓派局域网 IP，启动 Go 服务端，确认 `/mcp` 可初始化。
3. 烧录固件到 BL616CL，确认设备完成发现、上线并出现在 MCP `tools/list`。
4. 用 MCP Inspector 调用 `list_devices`、`list_protocols`、`get_device_tree`，再通过 `invoke_protocol` 调用 OLED 或传感器函数。

回滚：停止服务端和 broker 容器；固件串口执行 `factory_reset` 清除 broker 配置并回退到上一个固件镜像。bbolt 数据文件可直接删除，服务端会重建空注册表。

## Open Questions

- 首个目标板型是 `bl616cl_ave_v10` 还是 `bl616cldk`，以及板载实际传感器/执行器/OLED 清单，需在固件任务开始前确认。
- WiFi 凭据的预置方式（Kconfig 编译期常量、烧录时写入 NVS、还是串口命令），MVP 默认按 Kconfig + NVS 覆盖处理。
- Protocol UUID 的具体分配和 Interface 函数签名由用户后续细化；当前按 spec 中的信封、节点和 Protocol 结构落地。
- 首个 MCP 客户端（Claude Desktop、MCP Inspector、自研 Agent）会决定是否需要 HTTP 鉴权和 CORS；MVP 先不实现。
- WASM 后续 change 需要决定 WAMR 解释器/AOT 取舍、模块格式、分块传输、宿主 ABI 和资源限制；本 change 不实现这些内容。
