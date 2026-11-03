# Spec Delta

## Purpose

在服务端维护每台设备的发现记录、在线状态、Device Tree 缓存、Protocol 索引和状态变更事件，为 MCP 网关提供一致且可恢复的设备视图。

## ADDED Requirements

### Requirement: 设备发现记录

服务端必须（SHALL）在收到合法发现请求时创建设备记录，记录设备 ID、型号、来源 IP、首次发现时间和最后发现时间。同一设备 ID 的重复发现必须更新最后发现时间而不是创建重复记录。设备记录在设备成功上线并取得 Device Tree 之前必须标记为 `discovered` 状态。

#### Scenario: 首次发现创建设备记录

- **WHEN** 服务端收到设备 `dev-a` 的第一条合法发现请求
- **THEN** 服务端创建 `dev-a` 的 `discovered` 记录，包含型号、来源 IP 和首次发现时间

#### Scenario: 重复发现只更新时间

- **WHEN** 服务端再次收到 `dev-a` 的发现请求
- **THEN** 服务端更新 `dev-a` 的最后发现时间，不新增设备记录

### Requirement: 在线状态判定

服务端必须（SHALL）订阅所有设备的 `iot/v1/devices/+/status` 主题。收到 retained 或实时的 `online` 消息必须把设备标记为 `online`；收到 `offline` 消息必须标记为 `offline`。服务端启动或与 broker 重连后，必须通过 retained 状态消息重建在线设备集合。

#### Scenario: 收到上线状态

- **WHEN** 服务端收到设备 `dev-a` 状态主题的 `online` 消息
- **THEN** 服务端把 `dev-a` 标记为 `online`，并记录上线时间

#### Scenario: 服务端重启后恢复在线集合

- **WHEN** 服务端重启并重新连接 broker
- **THEN** 服务端根据 retained 状态消息恢复所有仍在线设备的 `online` 状态

### Requirement: 自动获取设备描述

设备首次进入 `online` 状态，或其 `profile_revision` 与缓存不一致时，服务端必须（SHALL）在 2 秒内发起 `device.describe` 请求。获取成功后必须原子地更新设备缓存和 `profile_revision`；失败时必须保留设备在线状态，并按 5 秒、10 秒、30 秒、60 秒的间隔退避重试，60 秒后维持每 60 秒重试。

#### Scenario: 设备上线后自动拉取描述

- **WHEN** 设备 `dev-a` 首次上线
- **THEN** 服务端在 2 秒内发送 `device.describe`，并在成功后把 Device Tree 关联到 `dev-a`

#### Scenario: 描述请求失败后退避重试

- **WHEN** 服务端对在线设备发起 `device.describe` 但请求超时
- **THEN** 设备保持 `online` 状态，服务端按退避间隔重试，并在 MCP 中把该设备标记为描述不可用

### Requirement: Protocol 索引

服务端必须（SHALL）在每次成功获取 Device Tree 后原子重建 Protocol 索引，索引键为 `device_id + node_id + protocol_uuid + interface + function`，并保存函数签名、描述和节点路径。索引必须与 Device Tree 缓存同一次更新生效；重建失败时必须保留旧索引并把设备描述标记为不可用。索引必须在 2 秒内对 MCP 网关可用。

#### Scenario: 描述更新后重建索引

- **WHEN** 设备 `dev-a` 上报包含 OLED Protocol 的新 Device Tree
- **THEN** 服务端在同一事务中更新缓存和 Protocol 索引，MCP 网关可在 2 秒内查询到 OLED 函数

#### Scenario: 索引重建失败保留旧索引

- **WHEN** 新 Device Tree 校验通过但 Protocol 索引写入失败
- **THEN** 服务端保留旧索引，把设备描述标记为不可用，并记录可诊断错误

### Requirement: 缓存持久化与恢复

服务端必须（SHALL）把每台设备最近一次成功获取的 Device Tree、`profile_revision` 和最后在线时间持久化到本地存储。服务端重启后必须先加载缓存，并把尚未重新确认的设备标记为 `stale`；设备重新上线并成功上报后必须清除 `stale` 标记。缓存损坏或版本不兼容时必须丢弃缓存并重新拉取。

#### Scenario: 重启后先展示缓存

- **WHEN** 服务端重启且本地存在设备 `dev-a` 的有效缓存
- **THEN** 服务端在重新确认前以 `stale` 状态展示缓存中的 Device Tree

#### Scenario: 缓存损坏时重新拉取

- **WHEN** 服务端启动时发现缓存文件损坏
- **THEN** 服务端丢弃该缓存，并在设备下次上线时重新发送 `device.describe`

### Requirement: 设备身份冲突处理

同一设备 ID 必须（SHALL）只对应一条活跃记录。收到不同来源 IP 的同 ID 发现请求时，服务端必须更新来源信息并保留已有 Device Tree。若新上线设备的型号与已有记录不同，服务端必须在日志中记录冲突，并在成功获取新 Device Tree 后替换旧缓存。

#### Scenario: IP 变化后更新来源

- **WHEN** 设备 `dev-a` 以新的 IP 地址再次发送发现请求
- **THEN** 服务端更新 `dev-a` 的来源 IP，保留其 Device Tree 和在线状态

#### Scenario: 型号冲突告警

- **WHEN** 已登记的 `dev-a` 以不同型号重新上线
- **THEN** 服务端记录型号冲突日志，并在获取新描述成功后替换旧 Device Tree

### Requirement: 状态变更事件

设备状态或 Device Tree 发生变化时，服务端必须（SHALL）在 2 秒内通知 MCP 网关，事件类型至少包括 `device_online`、`device_offline`、`device_profile_updated` 和 `device_stale`。通知失败必须重试，且不得阻塞后续状态处理。

#### Scenario: 上线事件触发工具注册

- **WHEN** 设备 `dev-a` 上线并成功获取 Device Tree
- **THEN** 服务端发出 `device_profile_updated` 事件，MCP 网关在 2 秒内完成工具注册

#### Scenario: 通知失败不阻塞

- **WHEN** MCP 网关暂时不可用导致状态事件处理失败
- **THEN** 服务端保留状态变更并重试通知，同时继续处理其他设备的状态消息

### Requirement: 发现超时清理

处于 `discovered` 状态但连续 5 分钟未上线的设备必须（SHALL）被标记为 `expired` 并从可发现设备列表中移除；保留其最后发现时间用于诊断。之后若再次收到该设备的发现请求，必须重新创建 `discovered` 记录。

#### Scenario: 长期未上线设备过期

- **WHEN** 设备 `dev-a` 被发现后 5 分钟内未进入 `online` 状态
- **THEN** 服务端把 `dev-a` 标记为 `expired` 并从列表移除

#### Scenario: 过期设备重新出现

- **WHEN** 已过期设备再次发送合法发现请求
- **THEN** 服务端重新创建该设备的 `discovered` 记录
