# 使用 virtme-ng 的端到端测试

**语言**: [English](TESTING.md) | 简体中文

集成测试使用 `pytest` 和 `virtme-ng`（vng）。`virtme-ng` 会启动一个 QEMU 虚拟机，使用宿主机内核或缓存的 Ubuntu mainline 内核，并在宿主机文件系统之上使用 Copy-on-Write（COW）覆盖层。

## 前置条件

- 宿主机上已安装 `virtme-ng`。
- `dpkg-deb`（供 `prepare.py` 解压内核使用）。
- `git`（测试框架会用它只把已跟踪文件复制到虚拟机中）。

> **重要：** `virtme-ng` 使用 **COW（Copy-on-Write）** 文件系统。这意味着，guest VM 看到的是其创建瞬间宿主机文件系统的一个“快照”。之后对宿主机文件的任何修改（例如编辑源代码）都**不会**反映到 guest 中，除非重启虚拟机。
>
> 测试框架会在启动虚拟机**之前**准备源码 tarball（`.dkms_copy.tar`）来处理这一点，从而确保最新的 git 已跟踪变更被打包进去。
## 准备内核

在针对某个特定 Ubuntu 内核版本进行测试之前，你必须先在宿主机上准备它：

```bash
# 准备一个或多个版本
python prepare-kernels.py v6.19.1 v6.19.2

# 列出所有已准备的内核并校验其完整性
python prepare-kernels.py
```

该脚本会：
1. 从 Ubuntu mainline 仓库下载 `.deb` 软件包到 `kernels/<version>/`。
2. 使用 `dpkg-deb` 解压它们。
3. 校验 `vmlinuz` 和内核头文件的完整性（会自动清理损坏的内容）。
4. 修复符号链接，以便 DKMS 能在虚拟机内正确找到头文件。

## 运行测试

### 针对宿主机内核测试（默认）
```bash
pytest
```
`pyproject.toml` 中的 pytest 配置会把默认收集范围限制在 `tests/`，因此直接执行 `pytest` 不会递归进入已准备的内核树或构建产物。

### 针对所有已准备的内核测试
```bash
pytest --all-kernels
```

### 针对某个特定缓存内核测试
```bash
pytest --kernel v6.19.1
```

### 针对多个内核测试（矩阵）
```bash
pytest --kernel host --kernel v6.19.1
```

### 调试
若要查看实时输出（包括模块加载日志和虚拟机初始化）：
```bash
pytest -s
```
日志会自动保存到 `~/.cache/logs/phantun_tests/YYYYMMDD_HHMMSS/`。

## 框架结构

- `tests/conftest.py`：核心框架。提供：
  - `vm`：管理 virtme-ng / QEMU 生命周期
  - `phantun_module`：每个 session 通过 DKMS 安装一次，并通过 `/etc/modprobe.d/phantun.conf` 重新加载模块参数
  - `dmesg`：等待新的内核日志行
- `tests/helpers.py`：共享辅助 API，用于命名空间、guest 场景执行、nft probe 和模块统计读取。
- `tests/guest/scenarios.py`：测试使用的、已签入仓库的 guest 侧小型 Python 场景。
- `tests/test_dkms.py`：覆盖 DKMS 安装/加载/重载和参数校验。
- `tests/test_config_stats.py`：覆盖 selector 配置、`/sys/module/phantun/stats/*` 和基础 selector 路径行为。
- `tests/test_handshakes.py`：覆盖 shaping 语义和控制 payload 可见性规则。
- `tests/test_netns_udp.py`：覆盖基本的命名空间 UDP 到 fake-TCP 行为以及多通道行为。
- `tests/test_packet_loss.py`：覆盖握手重试、payload 丢失行为以及丢包条件下的状态机行为。
- `tests/test_recovery.py`：覆盖冲突处理、相同 tuple 替换、quarantine 和 unknown-packet 恢复行为。
- `tests/test_wireguard.py`：覆盖内核 WireGuard 和 `wireguard-go` 的端到端行为。

### 最佳实践

1. **使用 `phantun_module` fixture 管理模块生命周期**
   - 对要测试的参数调用 `phantun_module.load(...)`，例如：
   - `managed_local_ports="51820"`
   - `managed_remote_peers="198.51.100.20:51820"`
   - 该辅助工具会在不同参数集之间干净地卸载/重载模块。

2. **记住虚拟机看到的是 COW 快照**
   - 如果你在虚拟机已经启动后修改了已跟踪文件，guest 看不到这些修改。
   - 对于必须在 guest 内可见的源码改动，请重启 pytest session / VM。

3. **使用命名空间辅助工具，而不是手写 shell**
   - `ensure_netns_topology(vm)` 和 `cleanup_netns_topology(vm)` 会创建并拆除标准的 `pht-a` / `pht-b` veth 拓扑。
   - `run_netns_scenario(...)` 用于同步的 guest 动作。
   - `spawn_netns_scenario(...)` 用于长时间运行的并发参与者，例如 server、延迟发送方或抓包辅助进程。

4. **优先使用已签入仓库的 guest 场景，而不是内联 Python**
   - 把可复用的 guest 行为添加到 `tests/guest/scenarios.py`，而不是在测试中嵌入 heredoc Python。
   - 这样可以让这些场景通过已跟踪文件 tarball 对 guest 可见，也能避免重复的测试逻辑。

5. **针对你的问题使用正确的 nft probe**
   - `make_netns_output_probe(...)`：验证命名空间 `output` 上看到的是原始 UDP 还是已转换 TCP。
   - `make_netns_output_flag_probe(...)`：验证特定 TCP 标志模式（`SYN`、`SYN|ACK`、`RST|ACK`、keepalive ACK 等）。
   - `make_netns_tcp_payload_probe(...)`：验证特定 TCP payload，例如 shaping/control payload 或已排队的 responder 数据。
   - `make_netns_ingress_flag_drop_probe(...)`：在 veth 入接口上丢弃数据包，用于丢包测试。
   - `make_netns_ingress_payload_drop_probe(...)`：在 veth 入接口上丢弃特定 TCP payload。

6. **对于丢包测试，在 veth 入接口丢弃，而不是在发送方 output 丢弃**
   - 使用 `VETH_A` / `VETH_B` 上的 `netdev` 入接口 probe 来模拟路径上的丢失。
   - 当你的目的是模拟网络丢失时，不要在发送方 `OUTPUT` 上丢弃；那样测试到的是本地发送失败，而不是路径丢失。

7. **通过辅助工具读取统计和日志**
   - 对 `/sys/module/phantun/stats/*` 使用 `read_module_stats(vm)` / `read_module_stat(vm, name)`。
   - 当可观察结果是内核日志行而不是数据包或统计计数器时，请使用 `dmesg` fixture。

8. **显式处理预期失败**
   - 当测试故意期望某个 guest 命令或 `modprobe` 失败时，传入 `check=False`。
   - 对于成功的 guest 场景，使用本地 `assert_completed(...)` helper 或显式 `pytest.fail(...)` 检查，以获得更清晰的错误信息。

9. **让断言精确针对被测行为**
   - 对 selector 测试，检查是否有原始 UDP 逃逸，还是出现了已转换 TCP。
   - 对 shaping 测试，同时检查线上 payload probe 与 UDP 应用实际收到的内容。
   - 对 recovery 测试，同时检查数据平面是否成功，以及控制平面副作用，例如 `RST`、collision 统计、已排队数据包或 quarantine 行为。

10. **先运行最小且有用的子集**
    - 在开发期间，优先执行如下更聚焦的命令：
    - `pytest tests/test_packet_loss.py -q`
    - `pytest tests/test_recovery.py::test_established_bare_syn_replacement -q -vv`
    - 待聚焦场景通过后，再扩大到更广的回归测试集。

11. **控制时序，而不是靠运气**
    - 如果某个测试需要特定事件在飞行途中交错发生（例如同时发起连接），不要依赖 Python 的顺序执行或很小的 `time.sleep()` 调用。在负载下，CPU 调度器会破坏你的假设，导致测试变得脆弱。
    - 相反，应通过 `tc netem` 在数据平面上增加延迟来强制时序：
      ```python
      vm.run(["ip", "netns", "exec", NS_A, "tc", "qdisc", "add", "dev", VETH_A, "root", "netem", "delay", "150ms"])
      ```
    - 这能确保数据包在队列中停留足够久，从而让测试场景触发必要的重叠状态转换。记得在 `finally` 块中或拆除拓扑时清理 `qdisc`。

## GitHub Actions 变慢警告

> **重要：** GitHub 的托管 runner 本身已经运行在虚拟化环境中。我们的测试框架随后又通过 `virtme-ng` 启动另一个 QEMU guest，因此 CI 实际上是在嵌套 QEMU 中运行。那些在本地机器上很快完成的测试，在 GitHub Actions 上可能会慢得多，而且短暂状态可能会在宿主机侧轮询之间出现又消失。

在新增或调试测试时，应把 GitHub CI 视为最差情况的调度与时序环境：

- 优先使用能够保持足够久、可以跨越慢速轮询的可观测量：nft 计数器、累积模块统计、guest 可见结果或 dmesg 行。不要让测试只依赖捕捉一个短暂的中间值，例如瞬时的 `flows_current` 峰值。
- 如果测试必须轮询 guest 状态，就尽量用最少的 guest 往返读取它。在需要时扩展共享 helper，而不是在循环里手写许多小型 SSH 命令。
- 使用 `tc netem` 或其他数据平面控制来强制顺序/重叠。不要依赖很小的 `time.sleep()` 间隔来制造只会在快速笔记本上出现的竞争。
- 在合并时序敏感测试之前，请先问自己：如果 runner 慢 5-10 倍，并且 host/guest 时间都带噪声，这个断言仍然成立吗？如果不能，那么这个测试很可能检查的是错误的东西。
