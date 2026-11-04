---
title: AT-SPI 错误名不能证明对象已消失
date: 2026-10-18
category: runtime-errors
module: crates/linux/src/tree/properties/absence.rs, crates/linux/src/tree/resolve/resolve_errors.rs, crates/linux/src/tree/observe/node_evidence.rs, crates/linux/src/tree/resolve/mod.rs, crates/linux/src/tree/properties/toolkit_trust.rs
problem_type: runtime_error
component: atspi-resolver
symptoms:
  - "一个存活元素被报告为已销毁，因为某次读取返回了 UnknownObject、UnknownMethod 或 UnknownInterface。"
  - "一个已销毁元素仍能解析成功，因为对其存储对象路径的某次读取成功了。"
  - "一个已退出的应用被报告为已销毁元素，或一个已停止的应用被报告为已退出。"
tags: [linux, at-spi, d-bus, stale-ref, resolution, error-names, toolkits]
---

## 问题

在 Linux 上重新识别存储的引用，需要判断元素是否仍然存在。显而易见的规则是读取存储的对象路径，并把 D-Bus 错误名 `UnknownObject` / `UnknownMethod` 视为“已消失”。但对照实际运行的 GTK3、GTK4、Qt 6 和 Chromium 实测（`probes/linux/FINDINGS.md` L17-1 至 L17-12），该规则在两个方向上都是错的。

## 工具包的实际行为

- **同样的错误名在存活对象上表示“未实现”**（L17-2）。Qt 对 48 个存活节点中的 46 个，在 `Text.GetText` 上返回 `UnknownObject`，对 `Value` 的 `Properties.Get` 返回 `UnknownInterface`；GTK4 返回 `UnknownMethod`；GTK3 返回 `Failed`。Qt 还对每个存活节点在 `Properties.GetAll(Accessible)` 上返回 `UnknownInterface`（L17-3），因此 `GetAll` 失败不是缺失信号。
  应用接口上同样如此：GTK3 和 Qt 拒绝 `Properties.GetAll(org.a11y.atspi.Application)`，却能应答 `Get(ToolkitName)` 和 `Get(Version)`（L27-20、L27-32），因此 `toolkit_trust.rs` 逐个属性读取工具包信息，被拒绝的组读取不产生任何结论。
- **批量读取可能导致桥接崩溃。** 后续一次原始 Qt 探测用 `Properties.GetAll(Accessible)` 杀死了应用，而逐个 `Properties.Get` 调用和带接口守卫的遍历则存活下来（L41-7）。因此读取器使用并发的逐个属性读取，并在可选读取前检查已声明的接口。此前的拒绝结果并不能让批量读取变得安全。
- **已销毁对象仍会应答。** GTK3 和 Chromium 对象在销毁后约 16 秒内仍返回旧值，之后才返回 `UnknownObject`（L17-6 至 L17-8）。在该窗口期内，对存储路径的成功读取不能证明任何事。GTK4 和 Qt 则立即静默，分别返回 `UnknownMethod` 和 `UnknownObject`/`UnknownInterface`（L17-1、L17-4）。
- **对已销毁 GTK3 控件的写入会返回成功**，伴随归零的 `GetState` 和空名称（L30-2），因此成功写入与成功读取一样，都不能证明存在。`an-atspi-write-that-returns-success-has-not-changed-anything.md` 覆盖了写入路径。
- **已退出的应用是另一种错误。** SIGKILL 和正常退出都会在调用上返回 `org.freedesktop.DBus.Error.ServiceUnknown`，在总线守护进程查询上返回 `NameHasNoOwner`；绝不会返回 `UnknownObject`（L17-9）。
- **应用被杀时正在进行的调用返回 `NoReply`，而非 `ServiceUnknown`**，持续 27 至 45 毫秒，直到下一次调用发现应用已消失（L33-12）。`NoReply` 不是 `AppGone` 名称，因此跨越杀进程时刻的根 `GetChildren` 是瞬态的非判定：重试它，让下一次读取返回 `ServiceUnknown`。
- **已关闭或隐藏的 Qt 对话框仍留在树中。** 它保留 `ENABLED`、`RESIZABLE` 和 `SENSITIVE`，失去 `SHOWING` 和 `VISIBLE`（L33-15），因此从注册表构建的窗口列表会报告一个已消失的窗口。最小化窗口不是这种形态：GTK 3、GTK 4、Qt 和 Chromium 保留 `SHOWING` 或获得 `ICONIFIED`（L33-55）。两者皆无的窗口是空壳，`crates/linux/src/system/window/ghost_window.rs` 会将其排除。按标志过滤的状态普查可能隐藏空壳或隐藏真正的最小化窗口，因此应按完整状态集计数。
- **超时不携带 D-Bus 错误名**（L17-10）：已停止的进程在客户端限制后每次读取都以本地 `TIMED_OUT` 失败，而注册表仍列出它。注册表存在不是存活证据。
- **隐藏的 GTK4 标签页**离开树，但其对象仍应答每次通用读取（L17-5），因此直接路径读取能找到下降遍历找不到的东西。

## 解决方案

按层分类，如 `crates/linux/src/tree/properties/absence.rs` 所做。

1. `Failure` 对单次失败调用分类：`AppGone`（`ServiceUnknown`、`NameHasNoOwner`）、`Suspect`（上述三个名称）、`TimedOut`、`Other`。`Suspect` 本身绝不是判定。
2. `Liveness` 从节点自身的 `Accessible.GetRole` 读取，每个工具包的每个存活节点都会应答它（L17-11）。以 `Suspect` 失败的字段读取，仅当 `GetRole` 也在可疑族中失败时才变为“对象已消失”；否则失败保留在其字段上（`read_failure`）。`GetAll` 和 `GetChildAtIndex` 不是有效佐证：Qt 在第一个上失败，GTK4 无子节点在第二个上失败。
3. 存在性由父节点的子列表决定，而非对象自身的读取（L17-12）：解析器从已验证的窗口框架沿存储的子索引路径下降（`path_descent.rs`），仅当落点的证据仍匹配存储的身份时才接受。仅仅应答的节点不是正证据。绝不向上爬取 `Parent` 属性（L2-2）。
4. 在 `resolve_errors.rs` 中映射为确定的答案：离开总线的应用，仅当 `/proc` 显示进程已消失时才是 `STALE_REF` 类型 `resolve_owning_process_exited`，否则是 `resolve_owner_unreachable`（`owner_unreachable`），因为总线所有者缺失不能证明退出；已销毁对象是 `resolve_target_destroyed`；完整搜索无匹配是 `resolve_no_candidate`。映射之后，当错误是超时或不完整的身份观察（`resolution_identity_unknown`）时，`resolve/mod.rs` 检查 `/proc` 是否显示所属进程已停止。有停止进程证据时，两者都变为 `APP_UNRESPONSIVE`（`stopped_aware`）。没有时，超时保持不变，不完整观察变为可重试的 `STALE_REF`。其他错误原样传递。

### 进程状态需要独立证据

`crates/linux/src/system/process/process_state.rs` 中的生命周期路径与元素解析问的是不同问题。`system/process/liveness.rs` 在无障碍探测前后将预期的进程实例令牌与 `/proc` 核对：缺失、僵尸、死亡或被替换的进程是 `Exited`；不可读的 stat 保守视为存活。`kill(pid, 0)` 在僵尸进程上会成功，不能替代该检查。

`process_state_classify.rs` 将已应答探测、超时探测和不可用探测区分开。已停止的进程是 `Unresponsive`，除非它已应答。存活进程在连续两次无障碍探测超时后变为 `Unresponsive`（`process_state.rs::observe`）；一次超时或不可用证据则保持 `Running`。该规则是对照阻塞的主循环和负载下的健康应用实测的（台账 L31-17、L31-18），而非从注册表存在性推断。对不可用探测返回错误会让核心把不确定性误认为退出。

## 预防

- 不要仅凭 D-Bus 错误名分支；问同一名称在存活对象上还意味着什么，并按工具包实测。
- 对存储路径的成功读取不是存在性的正证据。正证据是从已验证的祖先遍历到达节点。
- 将“应用消失”、“对象消失”、“超时”和“未找到”保持为独立答案；合并它们会把原生失败变成自信的判定。
- 在根据一个工具包编码规则前，先探测每个工具包：Qt、GTK3、GTK4 和 Chromium 在此处的每个名称上都不一致。