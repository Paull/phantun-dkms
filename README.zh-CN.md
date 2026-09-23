# phantun-dkms

**语言**: [English](README.md) | 简体中文

[![Latest Release](https://img.shields.io/github/v/release/bjin/phantun-dkms.svg?display_name=release)](https://github.com/bjin/phantun-dkms/releases/latest)
[![GitHub branch status](https://github.com/bjin/phantun-dkms/actions/workflows/ci.yml/badge.svg)](https://github.com/bjin/phantun-dkms/actions/workflows/ci.yml)

如果你已经了解 [**Phantun**](https://github.com/dndx/phantun/)：这是一份 **Phantun fake-TCP 思路的 Linux 内核模块实现**。

Phantun 以围绕 **TUN 接口** 的 **用户态客户端/服务端** 形式运行。`phantun-dkms` 则将转换保留在 **内核** 中，直接拦截 **现有 UDP socket**，并避免了 TUN 拓扑。

## Phantun 兼容性

`phantun-dkms` **并未宣称与 Phantun 端点兼容**。

关键细节在于：**基础线上的数据包形态被有意保持一致**，因此 TCP/UDP 头部开销的计算方式也相同。就 MTU 预算而言，Phantun 的文档仍然是合适的思维模型。

变化的是围绕该线格式的**行为契约**。我没有尝试让混合的 `phantun` / `phantun-dkms` 端点互通，目前我会直接假定它们**很可能无法无缝工作**。

主要原因：

- **互操作性未经测试**：尚未验证混合部署。
- **发起方 ISN 随机化**：`phantun-dkms` 会随机化初始序列号；Phantun 历史上使用 `0`，更容易被 DPI 指纹识别。
- **对称角色模型**：`phantun-dkms` 去掉了固定的客户端/服务端节点角色，改为每条流仅区分 **initiator** / **responder**。
- **类 TCP 的 keepalive 行为**：`phantun-dkms` 会发送 keepalive ACK，并在连续多次未收到响应后拆除流；这也是一种可能破坏混合端点预期的行为差异。
- **其他协议级改进**：若干行为变化进一步降低了意外兼容的可能性，即使 fake-TCP 的数据包形态仍然比较接近。

简而言之：**大体上线路形态相同，但行为差异已足够大；除非经过验证，否则应将其视为仅适用于内核到内核的通信**。

## 如果你了解 Phantun，这里是实际差异

| 主题 | Phantun | `phantun-dkms` |
|---|---|---|
| **运行位置** | 用户态 | Linux 内核模块 |
| **流量模型** | TUN 接口，以及围绕它的路由/NAT | 直接拦截选定的现有 UDP socket |
| **进程模型** | 独立的客户端/服务端二进制程序 | 没有独立守护进程；两端主机都运行同一个模块 |
| **节点角色** | 固定的客户端与服务端 | 没有节点级客户端/服务端划分；每条流只有 initiator/responder |
| **应用集成** | 应用通过 Phantun 的 TUN 拓扑通信 | 应用继续使用自己现有的 UDP socket |
| **流量选择** | 围绕 TUN 端点的 bind/listen 拓扑 | 通过 `managed_local_ports` 和/或 `managed_remote_peers` 基于 selector 取得流量所有权 |
| **全网状组网** | 每对节点都要配置一条隧道，因此 mesh 会按 O(n²) 增长 | 每个节点只需配置一次 selector；无需按节点对建立隧道 |
| **防火墙/NAT 布线** | 需要 TUN 侧 DNAT/SNAT/masquerade 配置 | 不需要 Phantun 专用的 TUN DNAT/masquerade 布线；模块停留在正常主机路径上并与 conntrack 集成 |
| **入站原始 UDP 策略** | 所有权由 TUN 拓扑处理 | 命中 selector 的原始入站 UDP 会在非 loopback 入接口上被丢弃 |
| **存活性 / keepalive** | 这里没有对应的内核模块 keepalive 行为 | 类 TCP 的 keepalive ACK，以及多次丢失后的拆除 |
| **协议兼容性** | Phantun 契约 | 混合使用 Phantun / `phantun-dkms` **未经测试且很可能无法无缝工作** |
| **地址族** | IPv4 和 IPv6 | IPv4 和 IPv6，可通过 `ip_families=both/ipv4/ipv6` 选择 |

## 哪些地方仍然像 Phantun

核心 fake-TCP 形态仍然很熟悉：

- 严格的三次握手
- 数据承载在带 payload 的 TCP `ACK` 包中
- `seq` / `ack` 跟踪 payload 字节数
- 没有 FIN 关闭状态机
- `RST` 是拆除/错误信号

因此，如果你已经理解 Phantun 在线路上为什么能工作，那么这里绝大多数数据包级思路你也已经理解了。

## 能否与 Phantun 共存？

可以。

`phantun-dkms` 会明确忽略 **loopback 设备** 上的流量，包括本地 loopback 端点之间的 UDP。这意味着，使用 `127.0.0.1` 的本地 Phantun 部署可以共存，而内核模块不会试图接管它。

## 如何理解这个项目

`phantun-dkms` 不是创建 TUN 接口并通过它路由流量，而是这样做：

> 选定要接管的 UDP 流量，在内核中将其截获，在线上发送 fake TCP，在本地应用看到入站数据之前，再把入站 fake TCP 还原成 UDP。

这就是完整模型。

## 何时适合使用

当你想要类似 Phantun 的 fake TCP，但又希望：

- **没有 TUN 设备**
- **没有用户态转发守护进程**
- 应用继续使用它的**真实 UDP socket**
- 能显式控制**哪些本地端口或远端 peer**会被转换

典型目标：**WireGuard** 或 **`wireguard-go`**。

## 快速开始

### 从源码构建并安装

```bash
make
sudo make modules_install
```

静态 `Makefile` 仅会在其生成文件缺失或过期时引导执行 `./autogen.sh` 和 `./configure`。

如果内核自动检测失败，请向 `make` 传入 `KDIR=/path/to/kernel/build`。

### 在本地构建发布产物

```bash
make dkms
make dkms-deb
make dkms-rpm
```

### 从 DKMS 软件包安装（最新发布版）

* Arch Linux：安装 AUR 软件包 [phantun-dkms](https://aur.archlinux.org/packages/phantun-dkms)
* Debian/Ubuntu：从 [latest release](https://github.com/bjin/phantun-dkms/releases/latest) 下载并安装 `.deb` 文件
* Fedora/RHEL/Rocky：从 [latest release](https://github.com/bjin/phantun-dkms/releases/latest) 下载并安装 `.rpm` 文件
* 其他发行版：从 [latest release](https://github.com/bjin/phantun-dkms/releases/latest) 解压 `.tar.gz` dkms tarball 到 `/usr/src/phantun-0.x.y`，然后手动运行 `dkms add`

### 在 NixOS 上通过 flake 安装

添加 flake 输入：

```nix
inputs.phantun-dkms = {
  url = "github:bjin/phantun-dkms";
  inputs.nixpkgs.follows = "nixpkgs";
};
```

然后导入 NixOS 模块，并通过类型化选项配置内核参数：

```nix
{
  imports = [ inputs.phantun-dkms.nixosModules.default ];

  services.phantun = {
    enable = true;
    loadOnBoot = true;
    managedRemotePeers = [
      "198.51.100.20:51820"
      "[2001:db8::20]:51820"
    ];
    handshakeRequest = "hello";
    handshakeResponse = "base64:d29ybGQ=";
    keepaliveIntervalSec = 15;
    keepaliveMisses = 3;
  };
}
```

`loadOnBoot = false` 会保留软件包和已生成的 modprobe 配置，以便手动执行 `modprobe phantun`，但不会把 `phantun` 加入 `boot.kernelModules`。
当受管应用有意运行在初始网络命名空间之外时，请设置 `services.phantun.managedNetns = "all";`。

`nixpkgs.follows` 这一行使该模块使用使用者 flake 的 `nixpkgs` 输入，而不是继续保留本仓库所固定的第二份传递性 `nixpkgs`。

### 最简单的加载方式：接管一个本地 UDP 端口

```bash
sudo modprobe phantun managed_local_ports=51820
```

### 改为接管一个精确的远端 peer

```bash
sudo modprobe phantun managed_remote_peers=198.51.100.20:51820
```

```bash
sudo modprobe phantun managed_remote_peers='[2001:db8::20]:51820'
```

### 同时要求本地端口和远端 peer 匹配

```bash
sudo modprobe phantun \
  managed_local_ports=51820 \
  managed_remote_peers=198.51.100.20:51820
```

## 流量所有权模型

模块只会处理匹配一个或两个 selector 列表的流量。

| Selector | 含义 | 典型用途 |
|---|---|---|
| `managed_local_ports` | 接管这个本地 UDP/TCP 端口 | “转换我的本地 WireGuard 监听端口” |
| `managed_remote_peers` | 接管这个精确的远端 `IPv4:port` 或 `[IPv6]:port` | “只转换发往这个 peer 的流量” |

规则：

- **至少一个** selector 列表必须非空。
- 如果你同时配置了**两者**，则**两者都必须匹配**。
- selector 所有权只适用于**非 loopback** 流量。
- 命中 selector 的原始入站 UDP 会在**非 loopback 入接口**上被丢弃，以避免同一流量同时以原始 UDP 和已转换 UDP 的形式投递。
- `ip_families=both|ipv4|ipv6` 用于选择要转换的 IP 地址族；默认值为 `both`。在不支持 IPv6 的内核上，默认 `both` 会带警告地退化为仅 IPv4，而显式的 `ipv6` 会被拒绝。
- IPv6 `managed_remote_peers` 条目必须使用方括号，例如 `[2001:db8::20]:51820`；未加方括号的 IPv6 端点会因歧义而被拒绝。
- `managed_remote_peers` 采用精确地址匹配。如果远端主机会轮换其公网源地址（包括 IPv6 隐私地址轮换），请更新配置的 peer 地址；或者对应该在已接管本地端口上接受任意远端地址的类服务端端点，使用 `managed_local_ports`。
- 不支持 IPv6 链路本地端点地址。命中 selector 且本地或远端端点为链路本地地址的流量会被拒绝，而不是被转换。

### 命名空间附着

`managed_netns=init|all` 控制在评估 selector 之前，模块会附着到哪些位置：

| 值 | 含义 |
|---|---|
| `init` | 默认值。只附着到初始网络命名空间（`init_net`），也就是通常的主机命名空间。 |
| `all` | 兼容模式。通过 Linux pernet hook 附着到每个新建的网络命名空间。 |

这个边界决定了哪些命名空间会获得流表、拓扑 notifier、可选的保留 TCP socket，以及 IPv4/IPv6 netfilter hook。在每个已附着命名空间内，`managed_local_ports` 和 `managed_remote_peers` 仍然决定流量所有权。如果你的 UDP 应用有意运行在独立网络命名空间、容器命名空间或测试命名空间中，请设置 `managed_netns=all`。

### Selector 模式

| 模式 | 你设置的内容 | 取舍 |
|---|---|---|
| **仅本地端口** | `managed_local_ports` | 最简单，也最接近 Phantun 的**服务端 selector 模型**：按本地服务端口接管流量。 |
| **仅 peer** | `managed_remote_peers` | 最接近 Phantun 的**客户端 selector 模型**：按选定的远端 peer 接管流量。对该远端 `IPv4:port` 或 `[IPv6]:port` 而言，入站 TCP 所有权会变得很宽泛，因此仅应在该 peer 专门用于此 translator 时使用。 |
| **交集** | 两者都设置 | 最显式，通常也最安全。 |

### 仅本地端口模式下的可选本地 TCP 预留

`managed_local_ports` 告诉 `phantun-dkms` 要拦截哪些流量。它**不会**自动让模块成为该端口的真实内核 TCP 所有者。

这一点在**仅本地端口**模式（设置了 `managed_local_ports`，而 `managed_remote_peers` 为空）下很重要：发往所选端口的入站 fake TCP 会在正常 TCP 栈看到它之前被拦截，但另一个应用仍然可以成功在同一个 TCP 端口上 `bind()` / `listen()`。结果会产生误导性的所有权：应用看起来拥有该端口，但你原本期望它收到的 fake-TCP 流量会先被 `phantun-dkms` 截走。

对 WireGuard 服务端风格的部署而言，这意味着仅设置 `managed_local_ports=51820` 就可能遮蔽端口 `51820` 上的另一个 TCP listener，即使该 listener 成功启动。如果同一主机或命名空间还在 `127.0.0.1:51820` 上运行用户态 Phantun，那么默认行为仍然是安全的，因为 loopback 流量会被忽略。

`managed_remote_peers` 会把所有权缩小到一个精确的远端 `IPv4:port` 或 `[IPv6]:port`，因此同一本地端口上的无关 listener 不会以同样的方式被广泛遮蔽。这也是为什么 `reserved_local_ports` 只会在纯本地端口模式下生效。

`reserved_local_ports` 是可选参数，默认值为空。特殊值 `off` 和空输入都会禁用它。

| 值 | 含义 |
|---|---|
| 空 / 省略 | 禁用（默认） |
| `off` | 禁用 |
| 逗号分隔的端口列表（最多 64 个） | 只预留同时出现在 `managed_local_ports` 中的列出端口 |
| `all` | 预留所有生效的 `managed_local_ports` 条目 |

规则与取舍：

- 只有在设置了 `managed_local_ports` 且 `managed_remote_peers` 为空时才会生效
- 只会在选定命名空间中尝试预留
- 预留会对已启用地址族使用内核通配 bind：IPv4 为 `0.0.0.0:<port>`，IPv6 为 `[::]:<port>`
- 通配预留也会阻止该命名空间中同一端口上的 loopback listener
- 如果某个端口已被占用，模块仍会保持激活，并如实记录日志，而不是加载失败

只有当你希望 `phantun-dkms` 主动声明这些 fake-TCP 端口，并且愿意放弃在选定命名空间中这些相同端口上的 loopback TCP 共存时，才应使用 `reserved_local_ports`。如果你想让本地 WireGuard 服务端与使用 `127.0.0.1:<port>` 的用户态 Phantun 进程共享同一主机，请不要设置该参数。

## 日常参数

### Selector 与附着

| 参数 | 类型 | 默认值 | 含义 |
|---|---|---:|---|
| `managed_local_ports` | integer array，最多 64 个 | 空 | 模块接管的本地端口。对 WireGuard 而言，通常是本地监听端口。 |
| `managed_remote_peers` | string array，最多 64 个 | 空 | 采用 `x.y.z.w:p` 或 `[IPv6]:p` 形式的精确 peer。 |
| `ip_families` | string | `both` | 启用的转换地址族：`both`、`ipv4` 或 `ipv6`。 |
| `managed_netns` | string | `init` | 命名空间附着范围：`init` 表示仅初始网络命名空间，`all` 表示每个网络命名空间。 |

校验规则：

- `managed_local_ports`：`1..65535`
- `managed_netns`：`init` 或 `all`
- `managed_remote_peers`：合法的 `IPv4:port` 或带方括号的 `[IPv6]:port`
- 至少需要一个 selector 列表

### 可选本地 TCP 预留

| 参数 | 类型 | 默认值 | 含义 |
|---|---|---:|---|
| `reserved_local_ports` | string | 空 | 可选的仅本地端口 TCP 预留集合。空或 `off` 表示禁用，`all` 预留每个 `managed_local_ports` 条目，逗号分隔且最多 64 个端口的列表则只预留所列出的 `managed_local_ports` 子集。除非设置了 `managed_local_ports` 且 `managed_remote_peers` 为空，否则会被忽略。 |
### 可选时序与行为

| 参数 | 默认值 | 含义 |
|---|---:|---|
| `handshake_request` | 空 | 可选的发起方 payload，作为第一个 fake-TCP payload 发送。 |
| `handshake_response` | 空 | 可选的响应方 payload；仅在同时设置 `handshake_request` 时生效。 |
| `handshake_timeout_ms` | `1000` | 握手重传超时。 |
| `handshake_retries` | `6` | 在 `RST` 拆除之前允许的最大握手重试次数。 |
| `keepalive_interval_sec` | `30` | 在发送 keepalive ACK 之前允许的空闲时长。 |
| `keepalive_misses` | `3` | 在拆除之前允许未得到响应的 keepalive 次数。 |
| `hard_idle_timeout_sec` | `300` | 空闲流生命周期的硬上限。 |
| `reopen_guard_bytes` | `4194304` | 同一 tuple 重新打开前要求的最小序列空间距离；接受 `0..1073741823`，拒绝 `>= 1073741824` 的值。 |
| `half_open_limit` | `4096` | 每个网络命名空间允许的最大并发 half-open 流数。超过该限制后，新建的 `SYN` 创建或出站 half-open 流会被拒绝，直到现有 half-open 流建立或超时。 |
| `replacement_quarantine_ms` | `3000` | tuple 替换后的上一代 quarantine 窗口。匹配旧世代的数据包会在该窗口内被静默丢弃。 |
| `replacement_protect_ms` | `0`（自动） | 已建立的 initiator bare-SYN 替换保护窗口。在此窗口期间，发往已建立 initiator 的对齐 bare replacement SYN 会被静默丢弃，以抑制陈旧的 simultaneous-initiation 失败方 SYN。窗口过期后恢复正常替换处理。 |

### shaping payload 格式

`handshake_request` 和 `handshake_response` 接受：

| 形式 | 示例 | 含义 |
|---|---|---|
| 普通字符串 | `HELLO` | 按原样发送字节 |
| Hex | `hex:deadbeef` | 将后续文本按 hex 解码 |
| Base64 | `base64:YWJj` | 当目标内核支持时，将后续文本按 Base64 解码 |

格式错误的 `hex:` payload 会导致模块加载被拒绝。当目标内核提供 Base64 解码时，格式错误的 `base64:` payload 也会导致模块加载被拒绝；而在不支持内核内 Base64 的旧内核上，Base64 payload 会被忽略，并打印内核警告。

payload 长度会依据已启用地址族中最小的 fake-TCP payload 上限进行校验。当启用 IPv6 时，即使操作者预计只会有 IPv4 流，也会使用 IPv6 上限。

## 常见配置

### 接管一个 WireGuard 端口

```bash
sudo modprobe phantun managed_local_ports=51820
```

### 接管一个 WireGuard 端口，但只针对一个 peer

```bash
sudo modprobe phantun \
  managed_local_ports=51820 \
  managed_remote_peers=198.51.100.20:51820
```

### 添加可选的请求/响应 shaping hint

```bash
sudo modprobe phantun \
  managed_local_ports=51820 \
  managed_remote_peers=198.51.100.20:51820 \
  handshake_request=HELLO \
  handshake_response=WORLD
```

### 跨重启持久化

创建 `/etc/modprobe.d/phantun.conf`：

```text
options phantun \
  managed_local_ports=51820 \
  managed_remote_peers=198.51.100.20:51820
```

然后：

```bash
sudo modprobe phantun
```

## 面向 Phantun 用户的运行说明

| 主题 | 这里有什么变化 |
|---|---|
| **没有 TUN 布线** | 不要把 Phantun 的 TUN DNAT/SNAT/masquerade 配置照搬到这个项目中。 |
| **Conntrack** | 模块停留在正常主机路径上，并与 conntrack 集成，而不是创建独立的 TUN 路由拓扑。 |
| **Loopback** | 数据路径仍然不会碰 loopback 流量，因此默认的本地 loopback 上运行 Phantun 的方案可以共存。只有在纯本地端口模式下生效的 `reserved_local_ports` 通配 bind 才会阻止选定命名空间中相同端口上的 loopback listener；被忽略或失败的预留不会这样做。 |
| **MTU** | 与 Phantun 具有相同的基本 fake-TCP 头部开销；Phantun 的 MTU 指导仍然适用。 |
| **握手缓冲** | 在握手期间，模块每条流最多只排队 **一个** 出站 UDP 数据包；后续数据包可能被丢弃，并依赖应用的正常重传。 |
| **零长度 UDP** | 命中受管 tuple 的零长度出站 UDP 数据报会被消费并计为丢弃；它不会创建流，也不会被投递，因为 fake-TCP 数据承载在 ACK payload 字节中，而线上协议没有空数据报的表示。 |
| **Shaping 语义** | `handshake_request` / `handshake_response` 是 hint，而不是一个经过验证的子协议。 |
| **Keepalive** | `phantun-dkms` 具有类 TCP 的 keepalive 行为；这也是不应假定混合 Phantun / `phantun-dkms` 端点可以互通的另一个原因。 |

### IPv4 反向路径过滤

解封装后的 UDP 会在 fake-TCP 的入接口上重新注入。对于使用策略路由的 WireGuard 客户端，如果 peer 的反向路由选择的是 WireGuard 接口而不是物理 underlay，那么严格的 IPv4 反向路径过滤（`rp_filter=1`）会将其丢弃。IPv6 没有内核级的 `rp_filter` 等价物，尽管基于防火墙的反向路径检查也可能施加同样的约束。

可采用以下方法之一：

1. **在 underlay 接口上使用 loose RPF。** 这是对非对称或动态客户端路由最推荐的通用方案：

   ```text
   net.ipv4.conf.<underlay-interface>.rp_filter = 2
   ```

   Linux 会取 `all` 与接口设置中的最大值，因此当 `all` 保持为 `0` 或 `1` 时，这样可以让其他接口继续保持严格模式。请通过主机的网络或 sysctl 配置持久化该设置。

2. **保持 strict RPF，并添加显式的 WireGuard peer 规则。** 在 full-tunnel 策略规则之前，让 UDP 的反向查询先通过 underlay：

   ```bash
   ip -4 rule add pref <priority> \
     to <peer-ipv4> ipproto udp dport <peer-port> lookup main
   ```

   `main` 路由表必须把该 peer 路由到物理 underlay。当端点变化时请更新这条规则。如果 phantun 缺失，这也会让匹配的原始 WireGuard UDP 通过 `main`；如果这种回退不可接受，请添加 fail-closed 的 OUTPUT 规则。

phantun 不会修改 `rp_filter`；源地址校验策略仍由操作者自行控制。

## 运行时统计

模块会在以下位置导出计数器：

```text
/sys/module/phantun/stats/
```

`/sys/module/phantun/stats/*` 下的计数器是模块全局的，会聚合所有受管网络命名空间；在 `managed_netns=all` 时，它们并不是按 netns 分开的计数器。

示例：

```bash
cat /sys/module/phantun/stats/flows_created
cat /sys/module/phantun/stats/rst_sent
```

### 有用的计数器

计数器是单调递增的。有些计数器是聚合值，单个数据包/事件
可能同时增加一个聚合计数器和一个更具体原因的计数器：

- `udp_queue_full_dropped`、`udp_raw_inbound_dropped`、
  `udp_translation_failed_dropped` 和 `udp_reinject_failed_dropped` 也会
  计入 `udp_packets_dropped`。
- `tcp_misaligned_syn_rejected` 和 `tcp_unknown_tuple_rejected` 也会
  计入 `tcp_protocol_rejected`。
- `oversized_payloads_dropped` 同时覆盖两个方向：过大的出站 UDP 或
  已建立连接中生成的 fake TCP 超过当前路由 MTU 的发送，也会
  计入 `udp_packets_dropped`；过大的入站 fake-TCP 在模块以 RST 拒绝时，
  也会计入 `tcp_protocol_rejected`。
- 这些原因计数器是有用的标签，而不是完整的划分。它们的总和
  不一定总能等于对应的聚合计数器。

| Stat file | 含义 |
|---|---|
| **流生命周期与容量** | |
| `flows_created` | 成功插入流表的 flow 对象。 |
| `flows_established` | 到达 `ESTABLISHED` 状态的流。 |
| `flows_current` | 当前仍存在于流表中的 flow 对象。 |
| `half_open_rejected` | 因每个 netns 的 half-open 限制已满而被拒绝的有效 half-open opener。 |
| `handshake_retries_exhausted` | half-open 流在握手重传预算耗尽后被拆除。 |
| `established_liveness_timeouts` | 已建立流因错过过多 keepalive 而被拆除。 |
| **替换与 simultaneous-init 恢复** | |
| `replacements_accepted` | 已建立流在同一 tuple 上接受了有效的 bare、对齐 replacement SYN。 |
| `replacement_quarantine_dropped` | 在 replacement quarantine 窗口期间被静默丢弃的延迟上一代数据包。Bare SYN 不属于 quarantine drop。 |
| `replacement_protect_dropped` | 在 replacement-protect 窗口处于活动状态时被静默丢弃的已建立 initiator replacement SYN。 |
| `retired_evicted` | 由于哈希桶达到上限而被驱逐的、尽力保留的 retired flow 元数据记录。 |
| `collisions_won` | simultaneous-initiation 冲突中本地一侧保留 initiator 角色的次数。 |
| `collisions_lost` | simultaneous-initiation 冲突中本地一侧切换为 responder 角色的次数。 |
| **Shaping 与控制 payload** | |
| `request_payloads_injected` | 由模块注入的 `handshake_request` payload。 |
| `response_payloads_injected` | 由模块注入的 `handshake_response` payload。 |
| `shaping_payloads_dropped` | 因握手 shaping 而对 UDP 应用隐藏的保留首 payload 槽位。 |
| **RST 与 UDP 数据包统计** | |
| `rst_sent` | 成功发送的 fake-TCP RST 数据包。RST 往往是对拒绝或拆除的反应；要判断发送原因，请查看拒绝/丢弃计数器。 |
| `idle_acks_suppressed` | 对入站已建立 payload，本应立即发送的纯 ACK 因该流最近在短暂的 idle-ACK 抑制窗口内发送过本地 UDP payload 而被跳过。 |
| `route_cache_hits` | 已建立的 fake-TCP payload 发送在 `dst_check()` 通过后复用了每流 dst 缓存。 |
| `route_cache_misses` | 已建立的 fake-TCP payload 发送由于没有可用且有效的匹配缓存 dst，而需要重新执行路由查找。 |
| `udp_packets_queued` | 被流的一 skb 队列接受的已选出站 UDP 数据包数。队列已满导致的丢弃会单独计数。 |
| `udp_packets_dropped` | 被模块消费、但未能成功排队、转换或重新注入的 UDP 数据报聚合计数。 |
| `udp_queue_full_dropped` | 因该流的一 skb 队列已占用而被丢弃的已选出站 UDP。也会增加 `udp_packets_dropped`。 |
| `udp_raw_inbound_dropped` | 为避免原始与已转换的重复投递，在 `PRE_ROUTING` 中被有意丢弃的 selector-owned 原始入站 UDP。也会增加 `udp_packets_dropped`。 |
| `udp_translation_failed_dropped` | 在本地转换/打开/发送路径失败后被消费的已选出站 UDP。也会增加 `udp_packets_dropped`；half-open 准入限制拒绝则由 `half_open_rejected` 统计。 |
| `udp_reinject_failed_dropped` | 因本地 UDP 重新注入在临时压力下失败而被丢弃的入站 fake-TCP payload。也会增加 `udp_packets_dropped`。 |
| **Fake-TCP 拒绝与校验** | |
| `tcp_protocol_rejected` | 作为当前协议/状态机上下文中无效数据包而被拒绝的 owned fake-TCP 数据包聚合计数。常见原因请看下方更细的计数器。 |
| `tcp_misaligned_syn_rejected` | 因 `seq % 4095 != 0` 而被拒绝的 owned bare SYN。也会增加 `tcp_protocol_rejected`。 |
| `tcp_unknown_tuple_rejected` | 没有存活 flow 与之匹配的 owned fake-TCP 数据包，因其不是有效的新 opener 而被拒绝。也会增加 `tcp_protocol_rejected`；未对齐的 bare SYN 则使用 `tcp_misaligned_syn_rejected`。 |
| `bad_checksum_dropped` | 因 TCP 校验和无效而被丢弃的 fake-TCP 数据包。这些不计入协议拒绝。 |
| `oversized_payloads_dropped` | 超出模块限制的 payload，以及已建立连接中生成的 fake TCP 超过当前路由 MTU 的发送。过大的出站 UDP 也会增加 `udp_packets_dropped`；过大的入站 fake-TCP 在以 RST 拒绝时也会增加 `tcp_protocol_rejected`。 |

## 构建、重新加载、卸载

| 任务 | 命令 |
|---|---|
| 构建 | `make` |
| 刷新编译数据库 | `make compile_commands` |
| 安装模块 | `sudo make modules_install` |
| 加载模块 | `sudo modprobe phantun ...` |
| 卸载模块 | `sudo rmmod phantun` |

## 限制与当前状态

- 支持 IPv4 和 IPv6，并通过 `ip_families=both|ipv4|ipv6` 选择实际启用的转换地址族。
- 默认只在初始网络命名空间中安装 hook。对于需要在 `init_net` 之外进行转换的命名空间/容器工作负载，请使用 `managed_netns=all`。
- Linux 内核兼容性：从 `5.10` 到 `7.2`。
- 混合 **Phantun** / **`phantun-dkms`** 部署**未经测试**，应视为**很可能无法无缝工作**。
- **没有 FIN 关闭状态机**。
- 实际上这是一个**内核到内核的协议变体**，尽管基础数据包形态仍与 Phantun 很接近。
- 为 shaping 预留的首个 payload 可能会被有意对 UDP 应用隐藏。
- shaping payload 的缺失、延迟、重复或丢失，本身不会导致连接失败。

关于协议内部机制和状态机细节，请参阅 [**`DESIGN.md`**](./DESIGN.zh-CN.md)。

## 许可证

项目许可证：**GPL-2.0-or-later**，详见 `LICENSE` 文件
