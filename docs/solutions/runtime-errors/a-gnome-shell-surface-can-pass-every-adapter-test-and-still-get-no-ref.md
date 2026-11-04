---
title: GNOME Shell 表面能通过所有适配器测试，却仍然拿不到 ref
date: 2026-10-19
category: runtime-errors
module: crates/linux/src/system/tray, crates/linux/src/system/shell, crates/linux/src/notifications, scripts/notification-oracle.sh
problem_type: runtime_error
component: gnome-shell-surfaces
symptoms:
  - "snapshot --surface system-tray 报告的 ref_count 等于 shell 按钮数量，且没有任何托盘项或菜单项拿到 ref。"
  - "通过面板的直接子节点查找 shell 节点会找不到，于是它被判定为离屏，点击失败并报 'visible (live offscreen state is true)'。"
  - "点击通知时，gdbus monitor --dest org.freedesktop.Notifications 不打印任何信号。"
  - "有头点击打开托盘项的弹出菜单后，snapshot --surface menu 返回 WINDOW_NOT_FOUND。"
root_cause: wrong_assumption
resolution_type: code_fix
severity: high
tags: [linux, gnome-shell, status-notifier-item, dbusmenu, notifications, at-spi, refs, d-bus]
---

# GNOME Shell 表面能通过所有适配器测试，却仍然拿不到 ref

## 问题

Shell 表面（托盘、通知、快速设置）由虚拟节点和 shell 控件构建。它们有四个属性与应用不同。每一个都破坏了端到端路径，而适配器自身的测试却一直通过（`probes/linux/FINDINGS.md` L32-34、L32-35、L32-28、L32-37）。

## 实际情况

1. **核心会丢弃标识符类型为 `Unknown` 的条目。** `validate_ref_entry` 会拒绝它，分配返回 `SkippedInvalidEntry` 且不报错。没有原生 id 的平台节点必须携带带类型的 kind。托盘使用 `RuntimeId`（`crates/linux/src/system/tray/node_evidence.rs:20-28`），它通过了 `crates/core/src/refs/refs_validate.rs:38-45` 的校验。适配器测试中手工构建的 `RefEntry` 值永远到不了这道校验，所以只有 CLI 的 `ref_count` 能看出这次丢弃。
2. **Shell 地标位于你预期容器的下好几层。** 托盘面板在 `System` 菜单按钮上方两层。面板的直接子节点是无名分组，项按钮是它们下面的 `menu` 节点。按直接子节点查找什么也找不到，于是每个项都被判定为离屏且未绑定（L32-35）。托盘不再把项绑定到面板节点。托盘节点自身不携带边界，离屏状态未知（`crates/linux/src/system/tray/live.rs:21-61`）。
3. **Shell 用自己的唯一名发出通知信号。** 按目标 `org.freedesktop.Notifications` 过滤的监视器看不到任何信号。改为按接口匹配，正如 `scripts/notification-oracle.sh` 用 `dbus-monitor` 所做的那样。守护进程不做重放，所以点击之后才添加的规则收不到任何东西（L32-15）。
4. **托盘弹出菜单不是应用菜单表面。** 有头点击某个项会打开 shell 的弹出菜单，但 shell 树中没有弹出节点，`list-surfaces` 也不列出任何弹出表面。它的条目是该项的虚拟子节点，通过 dbusmenu 读取，并用 `Event(id, "clicked")` 选取（L32-36、L32-37）。托盘点击后不要等待 `menu` 表面。

## 预防

- 用 CLI JSON 中的 `ref_count` 和 `ref_id` 检查新虚拟节点是否通过核心校验，不要在测试中构建条目来检查。
- 依赖直接子节点之前，先对照真实 shell 树测量查找深度。
- 按发送方的唯一名选择 D-Bus oracle，并在操作之前订阅。
- 通过数据源读取 shell 弹出菜单，绝不要通过 shell 未暴露的表面读取。