# phantun-dkms 设计

**语言**: [English](DESIGN.md) | 简体中文

本文档介绍 **内部设计决策** 与 **协议行为**。
关于安装、日常配置、示例、统计、MTU 指导和运行说明，请参阅 [**`README.md`**](./README.zh-CN.md)。

## 1. 范围

### 目标

- 直接在 Linux 内核中运行 **Phantun 风格的 fake TCP**。
- 透明地与现有 UDP 应用协同工作，尤其是：
  - 内核 WireGuard
  - `wireguard-go`
- 避免使用 **TUN** 设备以及 TUN 侧 NAT 拓扑。
- 保留 fake-TCP 线上模型的核心：
  - 严格的三次握手
  - payload 位于 `ACK` 包中
  - 字节级精确的 `seq` / `ack`
  - 协议错误时发送 `RST`
  - 没有 FIN 关闭状态机
- 支持可选的 **首 payload shaping hint**，但不把它们变成必需且需验证的子协议。
- 使用**对称节点模型**：每条流只有 initiator 和 responder。

### 非目标

- 去掉 packet-boundary helper 中独立的 IPv4/IPv6 地址族拆分
- 提供用户态 Phantun 互操作性保证
- 将 eBPF 作为主要实现
- 将 xtables target 作为核心数据平面
- 提供真实的内核 TCP listener/service 实现

## 2. 核心协议契约

### 2.1 保留的线上语义

模块保持以下 fake-TCP 不变量：

- 严格握手：
  1. initiator 发送 `SYN`
  2. responder 回复 `SYN|ACK`
  3. initiator 以 `ACK` 完成握手
- responder 仅在 `seq % 4095 == 0` 时接受 `SYN`
- 数据包是携带 payload 的 `ACK` 包
- 发送方 `seq` 按 payload 长度推进
- 接收方 `ack` 跟踪 `peer_seq + payload_len`
- 没有 FIN/CLOSE 状态机
- `RST` 是拆除和错误信号
- 格式错误或不可能的数据包会以 `RST` 拒绝，除非被明确列入静默丢弃情形

### 2.2 可选 shaping hint

`handshake_request` 和 `handshake_response` 是 **best-effort shaping hint**。
它们**不会**变成必需的握手子协议。

规则：

- `handshake_request` 可以选择性占用 initiator 的第一个 payload 槽位。
- `handshake_response` 可以选择性占用 responder 的第一个 payload 槽位，但前提是同时配置了 `handshake_request`。
- 需要忽略的 payload 是通过该流世代的**保留最低 payload 序列号**来识别的，**而不是**通过到达顺序识别。
- 该保留槽位的身份被限定在接收方已确认进度的有符号半空间内：一旦接收方的 `ack` 相对保留序列向前推进了**至少 2^31 字节**，该槽位就会被解除武装，随后从该（回绕后）序列开始的 payload 会被当作普通数据投递。这样可为延迟 shaping 抑制设置上界；明确不承诺抑制那些延迟超过一半序列空间后才到达的控制 payload。
- shaping payload 的缺失、延迟、重复或乱序，**不会**单独导致建立失败。
- 只有被 shaping 逻辑有意抑制的 payload，才会对本地 UDP socket 隐藏。

理想路径：

1. initiator：`SYN`
2. responder：`SYN|ACK`
3. initiator：若已配置则发送 `ACK + handshake_request`；否则若存在已排队 UDP payload，则发送 `ACK + first queued UDP payload`；否则发送纯 `ACK`
4. responder：若两个 hint 都已配置，则发送 `ACK + handshake_response`；否则正常 responder 数据可立即开始发送
5. 之后继续正常数据传输

实现还必须接受“纯最终 `ACK`，随后再发送 payload”的情况，因为 shaping 仍然是可选的。

## 3. 选定的实现载体

### 决策

使用基于 **raw netfilter hook** 构建的 **out-of-tree C 内核模块**。

### 原因

难点不在于过滤，而在于完整的有状态转换：

- 截获出站 UDP
- 发起 fake-TCP 数据包
- 在真实 TCP 栈看到入站 fake TCP 之前拦截它
- 将 payload 解封装回 UDP
- 重新注入以供本地投递
- 管理定时器、重试、存活性、冲突处理和拆除

与 eBPF、xtables-target 核心逻辑，或者假装该协议就是普通 TCP 相比，这些需求更适合 netfilter 加直接 `sk_buff` 所有权的方式。

## 4. 顶层架构

### 4.1 对称节点

每台主机都运行同一个模块。
**不存在节点级的 client/server 模式位**。

对每条流而言，角色只有：

- **initiator**：由于本地出现出站 UDP 而创建该流
- **responder**：接受一个入站 fake-TCP `SYN`

### 4.2 命名空间附着

`managed_netns=init|all` 是最外层的附着边界。

| 值 | 效果 |
|---|---|
| `init` | 默认值。只附着到 `init_net`。 |
| `all` | 通过 pernet init 附着到每个网络命名空间。 |

只有被选中的命名空间会收到流表、每 net netdevice notifier、保留本地 TCP socket，以及 IPv4/IPv6 netfilter hook。下文的 selector 规则仍会在每个被选中命名空间内部决定流量所有权。被跳过的命名空间对全局地址 notifier 必须不可见，并且在退出时必须成为 no-op，因为它们的 pernet 存储没有初始化好的流表。

### 4.3 拦截 selector

translator 基于两个可选 selector 列表取得流量所有权。
一个数据包必须满足**所有已配置的 selector**。

| Selector | 目的 |
|---|---|
| `managed_local_ports` | translator 接管的本地 UDP/TCP 端口 |
| `managed_remote_peers` | translator 接管的精确远端 `IPv4:port` 或 `[IPv6]:port` peer |

Selector 模式：

| 模式 | 出站匹配 | 入站 fake-TCP 匹配 |
|---|---|---|
| 仅本地端口 | 本地源端口 | 本地目的端口 |
| 仅 peer | 远端目的 `IPv4/IPv6:UDP port` | 远端源 `IPv4/IPv6:TCP port` |
| 交集 | 两者都必须匹配 | 两者都必须匹配 |

约束：

- 至少一个 selector 列表必须非空
- selector 所有权仅适用于**非 loopback** 流量
- 只有在确认目的地址会被本地主机/当前 netns 本地投递后，入站 selector 所有权才会生效；转发流量永远不属于 translator
- 路由到 loopback 的出站 UDP 保持为 UDP
- 到达 loopback 的入站 fake TCP 会被模块忽略
- 到达 loopback 的原始入站 UDP 不受 selector-owned drop 影响

- `ip_families=both|ipv4|ipv6` 控制会注册哪些 netfilter 地址族；默认 `both` 会在内核具备 IPv6 支持时同时注册两者
- IPv6 `managed_remote_peers` 条目必须使用带方括号的 `[IPv6]:port` 语法；未加方括号的 IPv6 会被拒绝
- `managed_remote_peers` 是精确地址匹配；远端隐私地址轮换会被视作新的远端端点，需要更新配置，除非改用本地端口选择
- 在为带 scope 的链路本地流标识、校验和失效处理实现一致契约之前，IPv6 链路本地端点地址会被有意拒绝且不受支持

仅 peer 模式的注意事项：

- 对该远端 `IPv4:port` 或 `[IPv6]:port` 而言，入站 TCP 所有权会变得宽泛
- 仅当该远端 peer 专门用于此 translator 时才应使用仅 peer 模式

### 4.4 可选的本地 TCP 预留保护

仅本地端口模式会按目的端口选择入站 fake TCP，但 selector 所有权本身并不会让模块成为该端口的真实 TCP 所有者。希望由内核拒绝竞争性 TCP listener 的操作者可以配置 `reserved_local_ports`。

规则：

- 仅在设置了 `managed_local_ports` 且 `managed_remote_peers` 为空时生效
- 在 `phantun_net_init()` 期间，模块会在被选中的 netns 中，对每个生效的保留端口和已启用地址族尝试执行通配 TCP bind（`0.0.0.0:port`、`[::]:port`）
- 这些 socket 会一直保持绑定，直到 `phantun_net_exit()`
- bind 失败会被记录日志，但不会在该命名空间中禁用拦截
- 通配 bind 会有意阻止相同端口上的 loopback listener

这只是一个防御性的所有权保护。模块**不会**调用 `listen()`，**不会**接受连接，也**不会**表现得像一个真实 TCP 服务端点。

### 4.5 默认的入站原始 UDP 丢弃

默认情况下，凡是匹配已配置 selector、目的地会在当前主机/当前 netns 本地投递、并且从非 loopback 设备进入的原始入站 UDP，都会在 `PRE_ROUTING` 中被丢弃。

原因：

- 命中 selector 的流量必须只有一个所有者
- 如果允许原始 UDP 投递和已转换 fake-TCP 投递同时发生，会产生歧义性的混合投递
- 被转发的 UDP 不属于 translator 的流量，必须继续走正常路由路径
- 重新注入的已转换 UDP 会在 `PRE_ROUTING` 之后进入，因此不会被这条丢弃规则黑洞化

## 5. 流标识与冲突处理

### 5.1 面向本地端点的身份标识

流以 packet-boundary 的本地/远端端点对为键，包括地址族：

- `local` 始终是本地主机/当前 netns 的端点
- `remote` 始终是对端端点
- 直接按地址族 + 地址字节 + 端口匹配
- 因此，出站 UDP 和入站 fake TCP 会落到同一条流上，无需 canonical tuple 排序

- IPv4 辅助地址以及 IPv6 全局临时/弃用地址仍然是不同的端点身份；转换后的 fake-TCP 报头和路由查找会使用已存储的精确本地地址，而不会替换成别的本地地址
- IPv6 链路本地地址在本地和远端端点位置上都会被拒绝，因为当前的 scope 处理还没有形成完整的带 scope 链路本地契约

流仍然会存储有方向的本地/远端地址和角色。
canonical key 仅用于查找和防碰撞。

### 5.2 重复本地发起规则

在为某个 tuple 创建新的出站流之前：

- 如果存在 `ESTABLISHED` 流：复用它
- 如果存在正在握手的流：
  - 不创建第二条流
  - 如果还没有排队的出站 UDP skb，则最多排队一个
  - 否则丢弃，并依赖应用重传
- 如果只存在陈旧/死亡的本地状态：
  - 静默删除它
  - 在重新打开时最多保留一个已排队的出站 UDP skb
  - 创建一个新的 initiator 流

### 5.3 替换世代的 quarantine 与保护

如果一条已建立流在同一 tuple 上接受到一个有效的 bare replacement `SYN`：

- 销毁当前世代
- 只为紧前一个世代保留一条短暂的 quarantine 记录
- 在这段短窗口内，任何看起来仍属于旧世代的数据包都会被静默丢弃，而不是触发 `RST`
- 过期后恢复正常的 unknown-tuple 处理
- v1 默认 quarantine 窗口：`3000 ms`（`replacement_quarantine_ms`）

目的：避免 tuple 复用后，延迟到达的旧世代数据包污染恢复过程。

当 `SYN_SENT` 握手接受了一个干净的 `SYN|ACK` 时，已建立的 initiator 流还会启动一个不滑动的 bare-`SYN` replacement protection 截止时间。

在该截止时间内，一个 bare、对齐的 replacement `SYN` 会在通用替换处理之前被静默丢弃。
这覆盖了 simultaneous initiation 中延迟到达的失败方 `SYN` 包，而不改变 responder 对 duplicate-`SYN` 的处理。
`replacement_protect_ms = 0` 表示自动：使用 `min(replacement_quarantine_ms, handshake_timeout_ms * max(1, handshake_retries / 2))`。
非零的 `replacement_protect_ms` 会被直接使用；截止时间过后，替换行为恢复不变。

### 5.4 simultaneous initiation 策略

真正的 simultaneous open 会被拒绝。
该设计希望**每个 canonical tuple 只有一条幸存流**。

当 `SYN_SENT` 在同一 canonical tuple 上收到 bare `SYN` 时，使用以下 tie-break 规则：

- 更低的 ISN 获得 initiator 角色
- 更高的 ISN 失去 initiator 角色，并把入站 `SYN` 重新按 responder 处理
- ISN 完全相同：丢弃，并依赖重传

这样可以避免对 NAT 敏感的端点启发式判断，并保持 shaping 语义清晰。

### 5.5 一个已排队 UDP skb

half-open 流的缓冲会有意保持得很小：

- 每条正在握手的流**最多只排队一个**出站 UDP skb
- 为常见的 WireGuard 行为节省一个重传周期
- 控制内存和复杂度上界
- 超过第一个已排队 skb 的其余内容都会被丢弃

## 6. 每流状态机

每条流会存储：

- 角色：`INITIATOR` 或 `RESPONDER`
- 状态
- 有方向的本地/远端地址
- 发送序列号
- 接收确认号
- 一个已排队 UDP skb 指针
- 保留的首 payload 忽略槽位
- responder 控制响应的 pending-ACK / pending-release 标志
- 重传定时器状态
- 空闲与入站存活时间戳
- 用于 ACK 抑制的最近一次成功发送已建立本地 payload 的时间戳
- initiator bare-`SYN` replacement-protection 截止时间
- 引用计数和锁

### 6.1 Initiator 状态

#### `SYN_SENT`

当出现受管出站 UDP 且不存在有效流时进入该状态。

动作：

- 选择随机 `u32` 初始序列号，并保证 `seq % 4095 == 0`
- 拒绝与上一世代之间距离违反 `reopen_guard_bytes` 的候选 ISN
- 发送 `SYN`
- 最多排队一个 UDP skb
- 启动重传定时器

可接受：

- `ack = syn_seq + 1` 精确匹配的有效 `SYN|ACK`
- 用于 tie-break 处理的 bare、对齐 collision `SYN`
- `RST` → 销毁流

收到有效 `SYN|ACK` 时：

- 设置 `ack = responder_seq + 1`
- 如果配置了 `handshake_request`：发送 `ACK + handshake_request`
- 否则如果存在已排队 UDP：发送 `ACK + first queued UDP payload`
- 否则：发送纯最终 `ACK`
- 如果同时配置了 `handshake_request` 和 `handshake_response`：为从 `responder_seq + 1` 开始的 payload 启动忽略槽位
- 启动不滑动的 established-initiator replacement-protection 截止时间
- 立即转换到 `ESTABLISHED`

#### `ESTABLISHED`

行为：

- 如果注入了 `handshake_request`，则在该注入请求之后刷新 initiator 拥有的已排队 UDP
- 如果 responder 的首 payload 忽略槽位已激活，则只抑制起始序列匹配保留 responder 控制序列的 payload
- 后续更高序列的 responder payload 正常投递
- 按常规执行 UDP ↔ fake-TCP 转换
- 被接受的入站数据包会刷新存活怀疑状态，包括纯 `ACK` 和 handshake-response 确认流量
- 被接受的入站 payload 通常会立即发送纯 `ACK`
- 仅当该端点在固定的 250 ms 抑制窗口内，已经在同一条流上发送过已建立 fake-TCP payload 数据时，才可以跳过这个立即 payload `ACK`
- 由保留首 payload 控制导致的丢弃仍然会立即发送纯 `ACK`；它们不适用于抑制
- 只接收不发送的流，以及窗口之外的流，会保持原先立即发送纯 `ACK` 的行为
- 在 `keepalive_interval_sec` 内未收到有效入站流量后：发送纯 `ACK` keepalive
- 抑制窗口不会改变这一由入站驱动的存活规则
- 在 `keepalive_misses * keepalive_interval_sec` 内未收到有效入站流量后：如果已存储的路由/源身份仍可发送，则尽力发送一个 `RST`，然后销毁本地状态
  - 如果 RST 发送失败，则静默销毁本地状态
  - 如果已经有一个出站 UDP skb 排队，则创建新的 `SYN_SENT`，携带该 skb，并发送 `SYN`
  - 否则等待未来的出站 UDP

已建立状态下的入站标志优先级：

1. `RST` → 静默销毁本地状态
2. 重复的当前世代 `SYN|ACK` → 发送纯 `ACK`，保留当前世代
3. 在 established-initiator replacement protection 截止时间仍有效时收到 bare、对齐的 `SYN` → 静默丢弃，保留当前世代
4. 收到无 payload、无 `ACK`、且无其他控制标志的 bare、对齐 `SYN` → 接受为世代替换，把旧世代移入 quarantine，创建新的 responder `SYN_RCVD`，发送 `SYN|ACK`
5. 任何其他带有 `SYN` 的数据包 → 发送 `RST|ACK`，销毁本地状态
6. 否则 → 正常数据处理

### 6.2 Responder 状态

#### `SYN_RCVD`

当一个命中 selector 的入站 `SYN` 到达，且没有现有流接管该 tuple 时进入该状态。

校验：

- 只能是 bare `SYN`（设置了 `SYN`，无 `ACK`、无 payload、无其他控制标志）
- `seq % 4095 == 0`
- tuple 通过 selector 策略

动作：

- 选择 responder 序列号
- 设置 `ack = initiator_seq + 1`
- 发送 `SYN|ACK`
- 启动重传定时器

在 half-open 期间可接受：

- 重复的入站 bare `SYN` 重传 → 重新发送 `SYN|ACK`
- 有效的最终 `ACK`

收到有效最终 `ACK` 时：

- 将本地 `seq` 推进到 `responder_seq + 1`
- 如果配置了 `handshake_request` 且最终 `ACK` 已携带 payload：立即抑制该 payload，因为它占用了保留的 initiator 首 payload 序列
- 如果配置了 `handshake_request` 且最终 `ACK` 不携带 payload：为从 `initiator_seq + 1` 开始的入站 payload 启动忽略槽位
- 如果两个 shaping hint 都已配置：
  - 发送 `ACK + handshake_response`
  - 将 `seq` 按 `handshake_response.len()` 推进
  - 继续阻塞 responder 拥有的已排队 UDP，直到后续 initiator `ACK` 覆盖注入的 response，或后续 initiator 流量确认保留的 responder 序列已被跳过
- 否则直接转入 `ESTABLISHED`

#### `ESTABLISHED`

行为：

- 出站 UDP 变成 `ACK + payload`
- 入站 fake-TCP payload 会成为本地 UDP，除非其 payload 起始序列匹配某个已激活的忽略槽位
- 大于 translator 所支持最大 UDP reinjection 尺寸的 payload 属于无效，并会以 `RST|ACK` 拒绝
- `seq` 随出站 payload 长度增长
- `ack` 跟踪 peer 的 `seq + payload_len`
- 如果注入的 responder `handshake_response` 仍在等待确认，则 responder UDP 会遵循“一 skb 排队/丢弃”规则，直到解除阻塞
- 被接受的入站数据包会刷新存活怀疑状态
- 被接受的入站 payload 通常会立即发送纯 `ACK`
- 仅当该端点在固定的 250 ms 抑制窗口内，已经在同一条流上发送过已建立 fake-TCP payload 数据时，才可以跳过这个立即 payload `ACK`
- 保留首 payload 控制导致的丢弃仍然会立即发送纯 `ACK`；它们不适用于抑制
- 只接收不发送的流，以及窗口之外的流，会保持原先立即发送纯 `ACK` 的行为
- 如果一个携带 payload 的最终 `ACK` 让 responder 转入 established，并且还会先刷新已排队的 responder UDP，那么被刷新的数据可能携带的是 payload 之前的 `ack`；若随后抑制纯 `ACK`，则在后续流量出现前，这个确认值会暂时落后，因为该协议没有数据重传
- keepalive、存活失败以及硬空闲拆除使用与 initiator-established 流相同的策略

入站标志优先级：

1. `RST` → 静默销毁流
2. 重复的当前世代 bare `SYN` → 重新发出 `SYN|ACK`，保留当前世代
3. 无 payload、无 `ACK`、且无其他控制标志的 bare、对齐 replacement `SYN` → 替换世代，对旧世代执行 quarantine，创建新的 `SYN_RCVD`，发送 `SYN|ACK`
4. 任何其他带有 `SYN` 的数据包 → 发送 `RST|ACK`，销毁流
5. 否则 → 正常数据处理

## 7. 失败策略

### 7.1 立即 `RST` + 销毁流

- 错误的 `SYN` 对齐
- 握手期间错误的最终 `ACK`
- 不可能的标志/状态组合
- 超出 translator 支持的 UDP reinjection 尺寸的过大入站 payload
- 指向 unknown tuple 的非 `RST` 数据包

仅 peer 模式仍保持这条规则：如果一个来自受管远端 peer 的数据包与本地流状态不匹配，且也不是有效的新 bare `SYN`，则以 `RST` 拒绝，而不是静默丢弃。

### 7.2 静默情形

- 指向 unknown tuple 的杂散入站 `RST`
- 指向已知 tuple 的入站 `RST`：销毁本地状态，不回复
- TCP 校验和校验失败的入站数据包
- 处于 quarantine 活动期间、来自紧前一世代的数据包
- shaping-payload 的丢失、重复、延迟或乱序
- established 存活失败仅在尽力发送 `RST` 无法路由或传输时才退回静默拆除
- 由拓扑驱动的失效与硬空闲过期：本地拆除，不发送 `RST`

### 7.3 握手丢包容忍

translator 必须在重试预算范围内容忍握手路径数据包丢失：

- initiator `SYN` 丢失 → 保持 `SYN_SENT`，重传 `SYN`，继续最多保留一个已排队 UDP skb
- responder `SYN|ACK` 丢失 → 保持 `SYN_RCVD`，在定时器到期和收到重复 `SYN` 时重传 `SYN|ACK`
- `handshake_request` 或 `handshake_response` 丢失 → 连接仍视为已建立；后续更高序列的 payload 可以继续
- 三次握手完成前重试耗尽 → 拆除 half-open 流，并以 `RST` 发出信号

### 7.4 本地 I/O 压力

临时的本地队列或内存压力（`NET_XMIT_DROP`、`-ENOBUFS` 或
`-ENOMEM`）以及 path-MTU 拒绝（`-EMSGSIZE`）只会丢弃受影响的 payload 或
控制包，同时保持该流世代存活。Half-open 握手数据包仍保持定时器重试，
已建立 payload 的序列空间不会复用。已建立连接上的 `-EMSGSIZE` 发送
会被计为 oversized drop，而不是 translation failure。诸如不可达路由、
不支持的地址族、访问被拒或无效数据包构造等终结性路由或结构错误，
仍然会拆除受影响的世代。

### 7.5 抗伪造姿态

指向已知 tuple 的入站 `RST`，以及指向已建立 tuple 的入站、对齐 bare replacement `SYN`，都会在**不进行当前世代序列窗口校验**的情况下被接受；数据路径的窗口校验被有意省略，因为 payload 本质上是 UDP。

后果：只要一个路径外发送者知道四元组，并发来一个校验和正确的数据包，就可以拆除或替换一个世代。replacement-protect 只保护处于 initiator 角色的已建立流；responder 上与当前 opener 完全相同的 duplicate `SYN` 不在其保护范围内；活动中的 replacement quarantine 会顺带过滤被归入上一世代窗口的 `RST`。

结论：这是 v1 接受的取舍（peer 可通过重新握手恢复）。如果未来确有需要进行加固，一种方向是把 `RST` 路径受限于现有的当前世代窗口检查（`remote_seq_window_start..ack` /
`local_seq_window_start..seq`），而不触碰数据路径语义。

## 8. 内核中的数据包路径

### 8.1 出站 UDP 拦截

| Item | Value |
|---|---|
| Hook | `NF_INET_LOCAL_OUT` |
| 目标优先级 | 初始 `LOCAL_OUT` conntrack 分类之后（当前设计目标中为 `-199`） |
| 匹配 | IPv4 或 IPv6 UDP、非 loopback 出接口、命中 selector 的 tuple |

行为：

- 已建立流 → 消费 UDP skb，发出 fake-TCP skb
- 正在握手的流 → 排队一个 skb 或丢弃
- 没有流 → 创建 initiator 流，排队一个 skb，发送 `SYN`
- 在受管 tuple 上，零 payload UDP 会被消费/丢弃而不是转换，因为 fake-TCP payload 数据位于 ACK payload 中，没有空数据报表示
- 出站 UDP GSO superframe 会在转换前由软件分段；每个分段都会独立转换，half-open 的一 skb 排队规则对每个分段分别适用
- 如果 skb 已携带 conntrack 状态，则在截获数据包前先确认原始 UDP 项，以便转换后的入站回复能够匹配已建立的主机防火墙策略
- 把出站 UDP 数据包的传输元数据复制到生成的 fake-TCP 数据包上
- 原始 UDP skb 会从协议栈中被截走

### 8.2 入站 fake-TCP 拦截

| Item | Value |
|---|---|
| Hook | `NF_INET_PRE_ROUTING` |
| 目标优先级 | 在 conntrack 和真实 TCP 处理之前（`PHANTUN_PRE_ROUTING_PRIORITY`，`-399`） |
| 匹配 | IPv4 或 IPv6 TCP、命中 selector 的现有流或可接受的新 responder `SYN`、会在当前主机/当前 netns 本地投递、非 loopback 入接口 |

行为：

- 在模块状态机中处理握手和已建立数据
- 在仅 peer 模式下，来自受管远端 peer 的 bare、对齐 `SYN` 可以在任意本地目的端口上创建 responder 流，但前提是该数据包会在本地主机/当前 netns 被本地投递
- 如果没有流匹配，且该数据包也不是有效的新 bare `SYN`，则按 unknown tuple 拒绝，而不是传给真实 TCP 栈
- 在真实 TCP 栈生成自己的 reset 之前先消费该数据包
- 入站 fake-TCP 元数据只可复制到由同一个入站数据包触发的 fake-TCP 回复上

### 8.3 入站原始 UDP 丢弃

| Item | Value |
|---|---|
| Hook | `NF_INET_PRE_ROUTING` |
| 目标优先级 | 原始 UDP 丢弃运行在 `PHANTUN_PRE_ROUTING_PRIORITY`（`-399`），位于 conntrack 和本地 UDP 处理之前、IPv4/IPv6 defrag（`-400`）之后 |
| 匹配 | IPv4 或 IPv6 UDP、命中 selector 的 tuple、会在当前主机/当前 netns 本地投递、非 loopback 入接口 |

行为：

- 默认丢弃命中 selector 的原始入站 UDP
- 未命中或仅被转发的入站 UDP 允许正常通过
- 该丢弃规则不作用于模块重新注入的已转换 UDP

原始 UDP 丢弃和 fake-TCP 拦截都使用 `-399` 优先级。Linux 会把相同优先级的 hook 插入到现有条目前面，因此 ops 数组会先注册 `phantun_pre_routing`，再注册 `phantun_pre_routing_udp_drop`，所以命中 selector 的原始 UDP 会先执行。

### 8.4 解封装 UDP 重新注入

对于入站的已建立 fake-TCP 数据：

- 使用有方向的 tuple 构造一个新的 UDP skb
- 保留原始 UDP 源/目的 IP 和端口
- 通过 `netif_rx()` 使用原始入接口注入，从而让接收处理使用该设备所属的网络命名空间
- 在重新注入前要求原始入接口设备命名空间与 netfilter hook 命名空间匹配
- 给重新注入的 UDP 打上标记，使模块的原始 UDP 丢弃 hook 在其第二次经过 `PRE_ROUTING` 时豁免这个人工构造的 skb

结果：

- 本地 UDP socket（包括内核 WireGuard 和 `wireguard-go`）会像接收普通 UDP 一样接收数据
- 后续的入站防火墙和投递 hook 仍在与被拦截 fake-TCP 数据包相同的 netns 中运行
- 已转换 UDP 会绕过原始 UDP 丢弃 hook，因为该 hook 会消费掉重新注入标记

#### IPv4 反向路径过滤

重新注入的 UDP 会保留 fake-TCP 数据包的入接口设备。在严格 IPv4 `rp_filter=1` 下，策略路由可能会把该 UDP peer 解析到一个 tunnel，而不是这个入接口设备，并在本地投递前将数据包丢弃。IPv6 没有内核级的 `rp_filter` 等价物，尽管基于防火墙的反向路径检查也可能施加相同约束。

模块不会改变主机的源地址校验策略；部署方必须使用 loose RPF 或显式的 WireGuard peer 规则。

### 8.5 生成的 fake-TCP 发送

对于模块生成的 fake-TCP 数据包：

- 构造一个新的 TCP skb
- 显式设置 IPv4/TCP 或 IPv6/TCP 头部；将 TCP 校验和作为带有伪首部 seed 的 `CHECKSUM_PARTIAL` 输出，由设备或 `skb_checksum_help` 负责最终完成
- 在本地输出之前，把生成的 fake TCP 作为 conntrack-untracked（`IP_CT_UNTRACKED`）发出，以便 conntrack 永远不会跟踪这段半可见的 fake-TCP 交换
- 通过正常的、按地址族区分的本地输出路径发送（类似 `ip_local_out` / `ip6_local_out`）
- 在路由和本地输出之前应用所选择的逐包传输元数据

由于 `LOCAL_OUT` 截获的是 UDP 而不是 TCP，因此模块生成的 fake TCP 不需要复杂的自旁路路径。

### 8.6 传输元数据传播

元数据被视为**逐包传输上下文**，而不是流身份的一部分。

对于由当前出站 UDP skb 生成的 fake-TCP 数据包：

- 复制 UDP skb 的 mark 和 priority
- 复制 IPv4 TOS，或 IPv6 traffic-class / flow-label
- 在可用时复制 socket UID 和显式绑定的输出接口
- 用这些元数据来构造生成的 fake-TCP skb 以及执行路由查找

对于直接由入站 fake-TCP 数据包生成的 fake-TCP 回复，例如 responder `SYN|ACK` 或注入的 `handshake_response`：

- 仅对这个即时回复复制入站 fake-TCP 元数据
- 不在流中持久保存入站元数据
- 不让入站 mark、TOS、traffic-class、flow-label 或 priority 影响后续出站数据包

一条流只存储 `local_tx_meta`：最近一次已知的本地出站 UDP 传输策略上下文。之所以需要它，仅仅是因为某些出站生成的 fake-TCP 数据包没有原始 UDP skb 可供复制：

- 握手重传（`SYN`、`SYN|ACK`）
- 在没有排队 UDP payload 发出的情况下，已配置的握手控制 payload
- keepalive `ACK`
- 本地存活/拆除控制包，例如尽力发送的 `RST`

`local_tx_meta` 只用于出站生成的 fake-TCP 数据包。它绝不能根据入站 fake-TCP 数据包更新，也绝不能影响入站 UDP 重新注入。解封装后的 UDP 重新注入会使用它自己的接收路径 skb，并且只使用绕过模块原始 UDP 丢弃 hook 所需的私有重新注入标记。

## 9. 尽力而为的本地流失效处理

某些本地拓扑变化会让一个现有世代变得不再安全可复用。
选定策略：

- 缓存上一次成功用于 fake-TCP 发送的路由出接口设备
- 如果该设备进入 `GOING_DOWN`、`DOWN` 或被注销：立即使该流失效
- 如果绑定到流 tuple 中的精确本地 IPv4 或 IPv6 地址被移除：立即使该流失效
- 失效是静默的本地拆除；不要从一个已不存在的路径或源身份伪造 `RST`
- 这比已建立连接的存活失败更严格：拓扑失效绝不能从一个已知陈旧的路径或源身份伪造 `RST`
- 下一次出站 UDP 可以正常创建一个新世代

v1 中**有意不做**的事情：

- 不会因通用 FIB/默认网关变化而失效
- 不会因为该设备上的其他地址变化而失效

- 不会因 IPv6 地址弃用或临时地址标志变化而失效；只有在精确本地地址被移除时才会失效
原因：每次出站发送都会基于固定 flow tuple 执行一次新的路由查找；宽泛的路由变化会引入误报，却没有明确收益。

## 10. 配置表面

v1 优先使用**简单的模块参数**。
用户可见的参数文档由 README 负责。

设计约束：

- 最多 64 个 `managed_local_ports`
- 最多 64 个 `managed_remote_peers`
- 至少一个 selector 列表必须非空
- `ip_families` 必须是 `both`、`ipv4`、`ipv6` 之一；默认 `both`
- `managed_netns` 必须是 `init`、`all` 之一；默认 `init`
- `reopen_guard_bytes < 2^30`
- 格式错误的显式 `hex:`/`base64:` shaping payload 会导致加载错误；在较老内核上不支持 Base64 解码时，则退化为带警告且无 payload 的回退行为

未来控制平面方向：

- 结构化运行时配置优先考虑 generic netlink
- 之后也许会提供 xtables/nftables 集成，但它们只是 selector 表面，而不是核心引擎

## 11. 值得坚持的实现选择

### Netfilter 核心，而不是 xtables-target 核心

转换、状态所有权、定时器、重新注入和协议语义应当属于真实模块，而不是主要为策略布线设计的 target callback 抽象。

### 基于 selector 的拦截，而不是 fake TCP listener socket

这种设计可以直接与现有 UDP 应用协作，支持本地端口和精确 peer 所有权，也不会通过假装 fake TCP 是普通 TCP 来欺骗内核。

### 每条 half-open 流只排队一个 UDP skb

一个 skb 足以为常见的 WireGuard 行为节省一次重传周期，同时保持内存和复杂度可控。

### 用确定性的 tie-break，而不是 simultaneous open

只保留一对 initiator/responder，可以让流所有权保持稳定，也让 shaping 语义不产生歧义。
