# Spec Delta

## Purpose

在服务端维护每台设备的发现记录、在线状态、Device Tree 缓存、Protocol 索引和状态变更事件，为 MCP 网关提供一致且可恢复的设备视图。

## ADDED Requirements

### Requirement: 设备发现记录

服务端必须（SHALL）在收到合法发现请求时创建设备记录，记录设备 ID、型号、来源 IP、首次发现时间和最后发现时间。同一设备 ID 的重复发现必须更新最后发现时间而不是创建重复记录。尚未确认 MQTT 在线的记录必须标记为 `discovered`，取得描述不是进入 online 的前提。连接状态使用 `discovered/online/offline/expired`；描述状态独立使用 `missing/fresh/stale/invalid`，不得把 stale 混作连接状态。服务端自身 broker 连接状态独立记录为 connected/disconnected/reconciling；恢复期间的缓存设备标记连接未确认，不对外宣称可调用。

#### Scenario: 首次发现创建设备记录

- **WHEN** 服务端收到设备 `dev-a` 的第一条合法发现请求
- **THEN** 服务端创建 `dev-a` 的 `discovered` 记录，包含型号、来源 IP 和首次发现时间

#### Scenario: 重复发现只更新时间

- **WHEN** 服务端再次收到 `dev-a` 的发现请求
- **THEN** 服务端更新 `dev-a` 的最后发现时间，不新增设备记录

### Requirement: 在线状态判定

服务端必须（SHALL）订阅所有设备的 `iot/v1/devices/+/status` 主题。当前已确认或待确认连接匹配的 `offline` 消息必须使对应连接不可调用；已退役或不匹配当前/候选连接的 offline 不得改写当前连接；`online` 消息只是在线线索，不能单凭 retained 消息确认存活。服务端启动、broker 重连或收到未被识别为退役的候选 connection_id 后，可以暂记报告状态为 online，但必须按“候选连接确认与旧连接隔离”规则设置 `connection_confirmed: false`、描述为 stale/missing；只有收到当前候选/恢复探测对应的 `device.describe` 响应、校验并提交快照成功才确认连接并设为 fresh。只有已确认且描述 fresh、版本一致的重复 online 才不触发拉取；状态被清除后，相同 connection_id 的下一次周期 online 必须允许建立新刷新代次，不能被普通重复公告去重规则吞掉。未经历 UDP 发现但收到合法 status 的设备也必须建档，因为设备可能使用缓存 broker 配置。

#### Scenario: 收到上线状态

- **WHEN** 服务端收到 dev-a 的有效 online，且它是新候选、启动恢复或状态清除后需要重新确认的连接
- **THEN** 服务端记录报告状态并调度 describe，在确认前不允许执行硬件操作；已确认 fresh 连接的相同周期公告仅更新最后公告观测时间，不改写首次上线时间

#### Scenario: 服务端重启后恢复在线集合

- **WHEN** 服务端重启并重新连接 broker
- **THEN** 服务端以 retained 消息寻找候选设备，通过当前连接的 describe 响应确认仍在线的设备；仅有历史 online 而无响应的设备不可调用

### Requirement: 自动获取设备描述

设备首次报告有效 online、出现非退役候选 connection_id、服务端/broker 恢复，或状态清除后再次收到有效 online 时，服务端必须（SHALL）在 2 秒内登记/合并刷新意图。若 RPC 槽空闲且不在失败退避期，必须在触发后 2 秒内发送 describe；若同一有效刷新代次的 describe 已在途则共享结果；若槽由 invoke 或已失效代次的 describe 占用，只保留一个待刷新意图，原 RPC 结束或原截止时间到达释放槽后，仍符合刷新条件且退避已结束时在 2 秒内发送；若尚在退避，则在退避结束且槽可用后 2 秒内发送。待刷新意图优先于后续新 invoke，避免饥饿；不得为了刷新取消、重试或并发绕过已提交的硬件操作。待刷新意图是本地调度状态，不是跨连接的 MQTT 请求队列，发送前必须重新核对 broker/连接/刷新代次。

获取成功后必须原子地更新设备缓存和 profile_revision。失败保留报告的连接状态但描述不可调用，按该轮结束后 5/10/30/60 秒退避，之后每 60 秒；同一连接的周期公告不得重置失败退避或请求截止时间。新连接或状态清除后有效公告触发新的确认流程。后台 describe 的最多 15 秒传输预算从首次发送计起；MCP 调用的 15 秒刷新预算从调用开始计起，包含本地等槽/退避/合并等待，不因后台调度另加等待时间。各共享等待方独立遵守自己的截止时间，不因后来者加入重置已发送 RPC 的截止时间；一个等待方取消不取消其他有效等待方或独立自动刷新需求。无任何剩余需求时清理尚未发送的刷新意图，不为已取消手动调用额外启动请求。

#### Scenario: 设备上线后自动拉取描述

- **WHEN** 设备 dev-a 首次上线，RPC 槽空闲且不在退避期
- **THEN** 服务端在 2 秒内发送 device.describe，并在校验和提交成功后关联 Device Tree

#### Scenario: 描述请求失败后退避重试

- **WHEN** 服务端对在线设备发起 `device.describe` 但请求超时
- **THEN** 设备保持 `online` 状态，服务端按退避间隔重试，并在 MCP 中把该设备标记为描述不可用

#### Scenario: invoke 占槽时合并刷新

- **WHEN** 一个正常 invoke 尚占 RPC 槽，手动或自动刷新要求获取描述
- **THEN** 服务端只记录/合并刷新意图，不另发并行 RPC；原调用按原截止时间结束，槽释放且仍有有效刷新需求时至多 2 秒内发送 describe

#### Scenario: 周期公告不重置失败退避

- **WHEN** 当前连接描述拉取失败且处于 60 秒退避期，期间收到相同版本的周期 online
- **THEN** 原退避截止时间不变，不为每个公告开启新的 describe 循环

### Requirement: 同连接描述不可变与异常隔离

服务端必须（SHALL）为候选/已确认 connection_id 保存首次接受的 model、firmware_version 和 profile_revision 基线，并验证后续 status 与 describe 的对应字段一致；已经确认的同连接完整 Device Tree 也不得变化（以解析后的 JSON 值比较，不以空白或对象键序判定变化）。不同 connection_id 可以有不同版本，包括回滚，按候选确认流程处理。同一 connection_id 出现矛盾字段或描述时，不启动版本热更新，必须将该连接标为描述 invalid、禁止新操作、撤下其工具、使在途刷新代次失效并记录有界诊断，旧快照仅保留作只读展示。合法关联 describe 的矛盾结果返回 invalid_device_response；单纯矛盾 status 不得据此伪造另一个在途 invoke 的结果或宣称硬件操作已取消。

异常连接不能靠同 ID 的重复公告、手动刷新或清除状态后再公告恢复；必须使用新 connection_id 并重新确认。服务端在当前 broker 连接代次内保留该异常标记，遵守已有有界连接状态存储；服务端/broker 重启后的未知历史仍按未确认候选处理，不承诺无限跨重启隔离。此规则防止描述合并把矛盾版本的旧结果重新标为 fresh；它与正常状态清除后允许同 ID 周期公告恢复不同。

#### Scenario: 在途 describe 期间出现同连接版本冲突

- **WHEN** B 的 revision 1 describe 在途，又收到同一 B 的 revision 2 online
- **THEN** 服务端标记 B invalid、使旧刷新代次失效；随后到达的 revision 1 或 revision 2 响应均不得恢复 B 的工具，设备需换新 connection_id 后确认

#### Scenario: 新连接版本回滚合法

- **WHEN** 设备以新的 connection_id 公告较低 profile_revision，describe 与该新连接基线一致
- **THEN** 服务端允许经校验/事务提交替换旧描述，不因 revision 较低而拒绝

### Requirement: Protocol 索引

服务端必须（SHALL）在每次成功获取 Device Tree 后原子重建 Protocol 索引，索引键为 `device_id + node_id + protocol_uuid + interface + function`，并保存函数签名、描述和节点路径。索引必须与 Device Tree 缓存同一次更新生效；重建失败时必须保留旧索引并把设备描述标记为不可用。索引必须在 2 秒内对 MCP 网关可用。

#### Scenario: 描述更新后重建索引

- **WHEN** 设备 `dev-a` 上报包含 OLED Protocol 的新 Device Tree
- **THEN** 服务端在同一事务中更新缓存和 Protocol 索引，MCP 网关可在 2 秒内查询到 OLED 函数

#### Scenario: 索引重建失败保留旧索引

- **WHEN** 新 Device Tree 校验通过但 Protocol 索引写入失败
- **THEN** 服务端保留旧索引，把设备描述标记为不可用，并记录可诊断错误

### Requirement: 缓存持久化与恢复

服务端必须（SHALL）把每台设备最近一次成功获取的 Device Tree、`profile_revision` 和最后在线时间持久化到本地存储。服务端重启后必须先加载缓存，并把尚未重新确认设备的描述状态设为 `stale`、connection_confirmed 设为 false，而非引入 stale 连接状态；当前连接 describe 确认且新快照提交成功后才能清除描述 stale。单条缓存损坏或版本不兼容时只忽略该缓存并重新拉取，不删除整个数据库。整个 DB 无法打开时保留原文件，能安全隔离并新建存储才重建，否则启动失败并报错，不静默删除或覆盖。

#### Scenario: 重启后先展示缓存

- **WHEN** 服务端重启且本地存在设备 `dev-a` 的有效缓存
- **THEN** 服务端在重新确认前以描述状态 `stale` 展示缓存中的 Device Tree，并标记 connection_confirmed 为 false，不发布动态工具

#### Scenario: 缓存损坏时重新拉取

- **WHEN** 服务端启动时发现某台设备的缓存条目损坏，而数据库仍可读取
- **THEN** 服务端忽略该条目、不影响其他合法缓存，在该设备报告 online 时重新发送 `device.describe`

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

处于 `discovered` 状态但连续 5 分钟未上线的设备必须（SHALL）被标记为 `expired` 并从可发现设备列表中移除；保留其最后发现时间用于诊断。MCP 默认设备列表与资源列表排除 expired 记录；已知 ID 的单设备资源可只读返回 expired 元数据。之后若再次收到该设备的发现请求，必须重新激活为 `discovered` 记录，而非制造重复身份。

#### Scenario: 长期未上线设备过期

- **WHEN** 设备 `dev-a` 被发现后 5 分钟内未进入 `online` 状态
- **THEN** 服务端把 `dev-a` 标记为 `expired` 并从列表移除

#### Scenario: 过期设备重新出现

- **WHEN** 已过期设备再次发送合法发现请求
- **THEN** 服务端将该设备重新激活为 `discovered`，保持同一设备 ID 只有一条记录

### Requirement: 重连隔离与刷新合并

服务端必须（SHALL）在 broker 断连时立即禁止下发操作、移除动态工具并标记缓存 stale；请求必须在原截止时间内以 `broker_unavailable` 或 `outcome_unknown` 结束，不得在重连后自动补发旧操作。同一设备的上线拉取、退避拉取、手动刷新和 stale 调用前刷新必须合并为一个在途 describe。当前/候选连接的有效 offline、清除事件或新候选建立后，旧刷新代次的响应不得更新缓存；最后确认连接的恢复必须通过新的探测代次，而非接受旧响应；当前连接已确认为 B 时，已知被替代的 A 的 online/offline/响应均不得使 B 失效、重新进入 A 的刷新循环或改写快照；每次刷新还必须绑定本地刷新代次，较早响应不能覆盖较新快照。

描述验证失败、索引失败或持久化失败时，必须保留旧树和旧索引作为一个完整快照，标记 invalid/stale，撤下其动态工具并禁止执行，不能“树已更新、索引仍旧”。下一次成功提交才恢复工具。设备状态事件可以合并，但最终必须由注册表快照对账恢复，不使用无限增长的重试队列。

#### Scenario: broker 崩溃留下历史 online

- **WHEN** broker 重启后保留 online 消息，但设备实际已断电
- **THEN** 服务端未通过 describe 确认之前不发布该设备的动态工具，也不执行协议调用

#### Scenario: 多个客户端同时刷新

- **WHEN** 自动拉取和多个 MCP 客户端同时刷新同一设备
- **THEN** 它们共享一次 describe 请求及结果，而不是各自启动重试循环

#### Scenario: 设备换连接后旧响应到达

- **WHEN** 设备连接从 A 变为 B，但 A 的 describe 响应此时才到达
- **THEN** 服务端忽略旧响应，不替换 B 的描述或可调用能力

#### Scenario: 持久化事务失败

- **WHEN** 新描述有效但 bbolt 提交失败
- **THEN** 新树和新索引都不生效，旧快照仅用于带 stale 标记的只读展示

### Requirement: 候选连接确认与旧连接隔离

注册表必须（SHALL）区分最后已确认连接与待确认候选连接；connection_id 是不透明 UUID，不能按字典序或到达时间推断设备启动先后。单设备 status 事件串行处理，所有异步工作绑定服务端 broker 连接代次和本地刷新代次。收到不同且未被识别为已退役的 online 后可暂停调用并撤下工具，但只建立候选，不立即销毁最后确认连接的身份；候选仅在对应 describe 确认且完整快照提交成功后才能替换当前连接。

每设备保留最近 64 个已被替代连接 ID 的有界退役集合（同一服务端 broker 连接代次内），对集合内所有 online/offline/响应直接忽略。候选最多一个、describe 最多一个，重复 online 合并且不重置请求截止时间；新候选使旧刷新代次失效，不并行探测。候选确认失败时，若存在尚未收到有效 offline/清除事件的最后确认连接，则先对该连接进行一次合并 describe 重新确认，成功后恢复调用，不能无限只向失败候选重试。该回退与候选探测共用单 RPC 槽，不属于单次 MCP 刷新的 15 秒预算扩展；当前 MCP 刷新可失败，回退在后台完成。

退役集合淘汰的 UUID 或服务端/broker 重启后的 UUID 均重新视为未确认候选，不假定无限历史已知；即便迟到 online 无法被集合识别，也不能不经 describe 直接替换已确认快照。broker 重连时旧回调代次失效，缓存可作只读展示，任何恢复都需当前 broker 连接中的有效 status 线索和 describe。空 retained status 清除事件必须使候选/刷新代次失效并禁用该设备，直到新的合法 online 触发确认；健康设备按 mqtt-messaging 周期重新公告，同 ID 可重新确认，但已被标为异常的连接除外。迟到 describe 不能撤销清除事件，删除 retained 状态不是永久禁用机制。

#### Scenario: 已退役 online 迟到

- **WHEN** A 已被 B 替代且仍在退役集合内，B 已确认，随后收到 A 的 online
- **THEN** 服务端忽略 A，不撤下 B 的工具、不发起 A 的 describe

#### Scenario: 未识别的旧候选失败后恢复

- **WHEN** B 已确认，收到不在退役集合的旧 A online，A 的 describe 失败而 B 未报告 offline
- **THEN** 服务端重新探测 B，成功后恢复 B 的 fresh 快照和工具，不因反复等待 A 永久失去 B

#### Scenario: 状态清除后迟到描述

- **WHEN** 设备 status 被清除，清除前发起的 describe 随后成功返回
- **THEN** 服务端忽略旧刷新结果，设备保持不可调用，直到新的 online 和确认流程成功
