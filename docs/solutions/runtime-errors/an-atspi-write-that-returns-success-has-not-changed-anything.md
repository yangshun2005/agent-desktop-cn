---
title: 返回成功的 AT-SPI 写入并未改变任何内容
date: 2026-10-19
category: runtime-errors
module: crates/linux/src/actions/write/write_call.rs, crates/linux/src/actions/write/mutation.rs, crates/linux/src/actions/write/text_write.rs, crates/linux/src/actions/write/numeric_write.rs, crates/linux/src/actions/write/secure_write.rs, crates/linux/src/tree/capabilities/action_read.rs
problem_type: runtime_error
component: atspi-write-path
symptoms:
  - "set-value 或 type 报告成功，但字段仍保留旧文本。"
  - "对已销毁的控件、只读字段或 Gecko 输入框的写入返回 true，且无错误。"
  - "超时的写入在停止的应用程序恢复后才落地。"
  - "密码值出现在错误消息或回读结果中。"
tags: [linux, at-spi, d-bus, write-path, read-back, delivery, secure-fields, toolkits]
---

## 问题

`EditableText.SetTextContents`、`InsertText`、`Value` 写入和 `Action.DoAction` 返回布尔值或不返回任何内容。在 Linux 上，该返回值只说明调用到达了一个作出应答的对象，并不说明该对象执行了操作。在真实运行的 GTK3、GTK4、Qt 6、Firefox 和 Chromium 上实测（`probes/linux/FINDINGS.md` L30-2 至 L30-16），每个工具包都存在写入返回成功却毫无变化的情况，也存在写入返回失败但效果仍然落地的情况。

## 工具包的实际行为

- **成功但无效果。** 已销毁的 GTK3 控件对 `DoAction`、`SetTextContents`、`InsertText` 和 `Properties.Set` 作出应答且不报错，而 `GetState` 返回两个零字，名称为空（L30-2）。Gecko 对每个输入框的 `SetTextContents` 和 `InsertText` 都返回 `true`，却不做任何改变，尽管几乎每个 Gecko 节点上都存在 `EditableText`（L30-14）。只读的 GTK3 字段和 Qt 只读字段会静默接受写入；Qt 在此路径上不强制执行 `READ_ONLY`（L30-6、L30-11）。Gecko 忽略越界的 `Value` 写入，既不截断也不报错（L30-16）。
- **失败但效果已送达。** 停止的应用程序在调用方超时后对每次写入都返回 `NoReply`，而总线在 `SIGCONT` 时投递排队的调用，因此每次写入都会落地（L30-3）。`NoReply` 附带 "recipient disconnected" 表示应用程序在调用期间死亡，结果未知（L30-4）。
- **失败仅因接口缺失。** Chromium 输入框没有 `EditableText`，因此两种文本写入都以 `UnknownMethod` 失败（L30-15）。已销毁的 GTK4 对象以相同名称失败，消息为 "Object does not exist"（L30-2）；Qt 对已销毁对象使用 `UnknownObject`，对已销毁路径或缺失接口使用 `UnknownInterface`（L30-10）。仅凭名称无法区分这些情况（见 `an-atspi-error-name-is-not-proof-an-object-is-gone.md`）。
- **秘密藏在回读中。** 安全字段通过原始角色编号 40 识别，而非名称：GTK 对角色 40 报告的名称是 `text box`（L30-17、L30-19）。可见性关闭、`InputPurpose.PASSWORD` 或 `Adw.PasswordEntryRow` 的 GTK4 输入框角色为 61，完全不带安全信号，因此其 `GetText` 会返回明文秘密。

## 解决方案

每次写入都以写入后读取节点所显示的结果来判定，绝不依据应答。

1. 节点已知不可写时在调用前拒绝：`READ_ONLY` 置位或 `EDITABLE` 清除（`text_write::writable`）。工具包会接受写入并忽略它，因此在 Qt 上调用后没有任何手段能可靠捕获这种情况。
2. 写入前读取值，发送写入，然后再次读取（`text_write::judge`）。等于请求值为 `Matched`，结果携带 `verified`。等于写入前的值为 `write_ignored`：`ACTION_FAILED`，`details.cause = "write_ignored"`，处置为未送达。其他任何值，或无可读值，结果保持 `delivered_unverified`。`numeric_write.rs` 应用相同规则。`select/mod.rs` 没有写入前值比较；它通过轮询选择状态来验证，否则保持 `delivered_unverified`。数值回读通过核心的 `value_matches` 比较。仅当值为整数、控件不是微调按钮，或其 `MinimumIncrement` 已知时，才计为证明，因为工具包会取整（`numeric_write.rs:43-46`、L30-12）。
3. 在 `mutation::classify_write` 中对失败的调用分类。可能表示"已消失"或"未实现"的错误名称（`ObjectUnknown`、已声明接口上的 `InterfaceMissing`）通过对节点的一次 `GetRole` 读取来判定（`action_read::corroborate`）：存活表示方法缺失，死亡表示 `STALE_REF`，不可读表示未确认。`NoReply` 和已耗尽的截止时间属于 `delivery_uncertain`，绝不是可重试的未送达。
4. 当字段的角色编号为 40、状态标志指示其为安全字段，或其令牌包含 `secure` 时，将其视为安全字段（`secure_write::is_secure`）。安全写入从不回读，因此不携带 `verified`，`scrub` 会在错误的消息、建议和平台详情中将写入的文本替换为 `[withheld]`，并丢弃回显该文本的 `details`（`secure_write.rs:18-38`）。GTK4 角色 61 的字段无法被识别，因此其回读不得复制到信封中。

## 预防

- 不要仅凭应答就为 AT-SPI 写入报告 `ok`。应答最多证明送达了一个存活对象，在 GTK3 上连这一点都不成立。
- 在 `NoReply` 之后，未读取节点前绝不重发写入：第一次调用可能在恢复时落地，第二次会使其翻倍。
- 不要根据接口是否存在（Gecko）或 `read-only` 是否缺失（GTK3）来判断可编辑性；应同时使用 `EDITABLE` 和 `READ_ONLY` 位。
- 因保密而跳过的回读会放弃 `verified`。应如实报告为 `delivered_unverified`；不要为了恢复判定而读取秘密。
- 在编码写入规则前先探测每个工具包。Gecko、Chromium、Qt、GTK3 和 GTK4 在上述至少一种情况下彼此不一致。