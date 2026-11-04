---
title: 用身份指纹守护被操作系统重排的资源
date: 2026-07-11
category: best-practices
module: notification commands and FFI, crates/windows/src/system/window/window_activate.rs, crates/windows/src/system/display.rs, crates/windows/src/system/capture/screenshot.rs, crates/core/src/refs/ref_identity.rs
problem_type: best_practice
component: tooling
severity: high
applies_when:
  - "对由操作系统管理的列表索引选中的条目执行操作"
  - "将 ref 解析为集合中的某一行，而该集合可能因过滤或重排而变化"
  - "为通知或其他可重排界面设计操作 API"
  - "通过 CLI 或 FFI 添加通知变更操作"
  - "恢复、置前或以其他方式变更由可回收操作系统句柄（HWND、PID 或类似值）寻址的资源，且检查与写入无法作为单个原子操作发出"
  - "截取由可回收显示器句柄寻址的显示器，且 list-displays 与 screenshot 无法作为单个原子操作发出"
tags: [notifications, identity, reordering, fail-closed, ffi, windows, macos, stale-ref, check-then-act, screenshot, display]
---

# 用身份指纹守护被操作系统重排的资源

## 背景

操作系统管理的资源可能在调用方观察到它之后、操作变更它之前被插入、移除、回收或重新分配。通知中心可能在列表调用与变更之间插入、移除、重新分组或重排条目。Windows 可能销毁一个窗口，并在下一次使用它的调用之前把其 `HWND` 交给无关进程。索引或句柄本身都不是身份：在使用时，二者都可能指向另一个资源；只要检查与操作无法作为单个原子操作发出，二者之间的间隙就是目标可能改变的地方。

## 指导

**通知身份。** 通知变更命令要求至少一个来自同一列表结果的身份字段：预期应用或预期标题。Core 通过 `commands::notification_identity::required_identity` 构造该身份；空身份为 `INVALID_ARGS`。macOS 适配器在操作前立即用 `NotificationIdentity` 校验实时条目，不匹配则失败关闭。FFI 遵循同一规则：无效或缺失的身份数据会被拒绝，而不是被强制转换为无约束操作；调用方通过重新列出并选择新条目来恢复。

**窗口句柄身份。** Windows `HWND` 是可回收句柄：窗口一旦销毁，操作系统就可将同一数值交给无关进程的下一个窗口。当后续写入与该检查不原子时，仅在入口处校验一次身份并不够。句柄可能在检查与原生写入之间的间隙中被销毁并回收。`focus_window`（`crates/windows/src/system/window/window_activate.rs`）在入口校验之后的每次原生写入前立即重新校验句柄所有权（`owned_for_write`）。`restore_if_iconic` 在调用 `ShowWindow` 前重新检查所有权，`bring_to_foreground` 在 `ShowWindow` / `SetForegroundWindow` 前重新检查。任一内部检查捕获到回收时，都会以 `WINDOW_NOT_FOUND` / 未送达失败关闭，绝不会向外部窗口执行置前写入。

在每次写入前重新校验前置条件必要但不充分：成功检查也必须带身份限定，否则在最后一次写入与结果检查之间赢得竞态的回收会报告对错误窗口的成功。`is_owned_foreground` 同时要求 `is_foreground_window(handle)` **和** `evidence.owns_handle_now()`。第二项检查确认该句柄的实时所有者仍是存储的 pid，且该 pid 仍是存储的进程实例。仅句柄相等不够，因为它会接受一个因无关原因恰好处于前台的回收 HWND。理由见 `window_activate.rs` 中 `focus_window` 自身的文档注释。

**显示器身份。** Windows 显示器 id 是可回收的 `monitor-{HMONITOR}` 句柄：热插拔、模式变更或操作系统重新分配后，同一数值可能指向另一台显示器。在 `list-displays` 与 `screenshot` 之间仅信任 id，与裸 HWND 一样存在 check-then-act 间隙。调用方传入列表结果中的完整 `DisplayInfo`；适配器在截取前重新列出并核对 id 以及边界、主显示器、缩放（`display.rs` 中的 `verify_display_identity`），截取后再校验一次，使截取中途的变化丢弃字节，而不是将其归因到错误表面（`screenshot.rs` 中的身份三明治）。精确窗口截图同样要求 `process_instance` 令牌：仅 HWND 不够。后端选择（Modern 与 Legacy）与此正交：无论哪条截取路径生成 PNG，指纹都适用。

**重复集合内的元素身份。** 重复的原生标识符不能标识某一行。过滤后，另一行可能以相同标识符和边界占据原位置；禁用路径优化也无济于事，因为宽泛搜索仍会匹配它们。不要为此冻结可变值，因为普通文本输入必须能通过同一 ref 保持可编辑。macOS 保留元素自身的辅助功能地址，因此 `crates/core/src/refs/ref_identity.rs` 中的 `ref_identity::identity_match` 会拒绝已移除的行或孪生行，而不是将其别名化。其他平台从捕获证明解析 ref：它在捕获时唯一，且现在恰好有一个实时元素匹配其完整谓词，因此孪生行会在读取窗口前以 `AMBIGUOUS_TARGET` 被拒绝；被销毁并重建的唯一元素则按其替代者执行操作。

## 预防

- 绝不在没有身份前置条件的情况下暴露会变更的列表索引或裸句柄 API。
- 平台提供稳定不透明句柄时优先使用；否则使用最小可靠指纹，并在变更前立即校验。
- 当检查与操作无法原子化时，仅在入口校验一次前置条件不够：在操作执行的*每次*原生写入前立即重新校验身份，而不只是第一次。
- 将身份纳入成功谓词本身，而不只是前置条件。仅由句柄相等（`GetForegroundWindow() == handle`）构建的成功检查无法区分真正成功与恰好满足条件的回收句柄；检查还必须确认资源仍是操作所针对的那个，而不只是预期地址上有东西应答。
- 测试列表与操作之间的插入、移除以及标题/应用不匹配；对可回收句柄，直接针对前置条件和成功谓词测试销毁并回收的竞态。
- 对显示器截取，在列表与操作之间分别变更每个指纹字段（id、边界、主显示器、缩放）进行测试，并断言截取后不匹配会丢弃缓冲区。
- 对重复的原生标识符，测试同一位置边界上的相同替代项，并检查未变更条目上的普通文本编辑仍能通过其 ref 正常工作。

## 相关

- `crates/core/src/notifications/notification_identity.rs`
- `crates/macos/src/notifications/actions.rs`
- `crates/windows/src/system/window/window_activate.rs`
- `crates/windows/src/system/display.rs`
- `crates/windows/src/system/capture/screenshot.rs`