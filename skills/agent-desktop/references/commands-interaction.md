# 交互命令

改变 UI 状态的命令。运行 `agent-desktop <command> --help` 查看标志。

## 无头与有头

`--headed` 是全局标志（`agent-desktop --headed click @s8f3k2p9:e5`），对每个 `batch` 条目生效。有头操作可能先聚焦确切的源窗口，指针操作需要已验证的目标点。

`press` 始终是物理键盘输入。`hover`、`drag`、`mouse-move`、`mouse-click` 和 `mouse-wheel` 是物理光标输入，需要 `--headed`。原始坐标不携带窗口标识，因此从不聚焦应用。`key-down`、`key-up`、`mouse-down` 和 `mouse-up` 为保留命令，从不投递输入。它们返回 `ACTION_NOT_SUPPORTED`，但无头下的 `mouse-down` 或 `mouse-up` 会先返回 `POLICY_DENIED`。请使用 `press`、`mouse-click` 或 `drag`。

哪些手势有无头路径：

| 命令 | 无头 |
|---------|----------|
| `click`、`set-value`、`clear`、`select`、`check`、`uncheck`、`toggle`、`expand`、`collapse`、`scroll`、`scroll-to` | 是，前提是元素声明了语义操作（Linux 上的 `scroll` 除外） |
| `type`、`right-click`、`focus` | 取决于操作系统。阅读平台技能了解哪些需要 `--headed` |
| `double-click`、`triple-click`、`hover`、`drag`、`mouse-*` | 否。需要 `--headed` |

原生跨应用拖放可能无法从合成事件启动。同视图拖放和 Web 拖放可以工作。

## 读取结果

成功的操作会报告发生了什么，而不仅仅是它运行了。

| 字段 | 含义 |
|-------|---------|
| `data.steps` | 按顺序尝试的每种机制：`label`、`outcome`（`attempted`、`skipped`、`succeeded`）、`mechanism`（`semantic_api` 或 `physical_synthetic`）、`verified` |
| `data.disposition.delivery` | 观察到效果时为 `delivered_verified`；应用声称成功但无可观察变化时为 `delivered_unverified` |
| `data.post_state` | 操作后目标的状态（如果有） |
| `data.surfaces` | 操作稳定后打开的覆盖层 |

有状态操作（`set-value`、`type`、`clear`、`check`、`uncheck`、`toggle`、`expand`、`collapse`）在运行后重新读取状态。可读的矛盾返回 `ACTION_FAILED`，并带 `details.kind: "post_action_verification"`。读取失败保留自己的错误码，因此即使 `STALE_REF` 或 `TIMEOUT` 也可能在投递后出现。不可读的后置条件不会使操作失败。仅缺少证据时返回成功，并带 `delivered_unverified`、`details.verification_scope: "unavailable"` 和 `verification_reason: "insufficient_evidence"`。

- 安全文本会被脱敏。已投递的写入可能返回 `delivered_unverified`，并带 `verification_reason: "secure_field"`。这绝不会把原生投递错误变成成功。
- `set-value` 和 `type` 只验证元素值（`verification_scope: element_value`、`application_commit: not_verified`）。文档级效果（例如本应调整大小的表格）需用独立读取确认。
- 如果 `data.surfaces` 列出了工作表、菜单或警告框，应用正在等待它。使用 `find --surface sheet` 并响应它。带有打开覆盖层的 `delivered_unverified` 结果通常意味着有待确认的操作。

## 错误

退出码：`0` 成功，`1` 结构化错误，`2` 参数错误。`error` 对象可携带 `details`、`recovery` 和 `disposition`。忽略未知键。

| 代码 | 含义 | 恢复 |
|------|---------|----------|
| `PERM_DENIED` | 缺少所需的操作系统权限 | 阅读平台技能 |
| `ELEMENT_NOT_FOUND` | 无法在实时 UI 中解析 Ref | 重新快照 |
| `STALE_REF` | Ref 不再匹配实时 UI | 重新快照，使用新 Ref |
| `AMBIGUOUS_TARGET` | 多个实时元素匹配 Ref 的身份，或在捕获时未证明唯一 | 操作窗口中唯一的控件；完全相同的孪生元素需要 `--headed mouse-click --xy`（见 `commands-observation.md`） |
| `SNAPSHOT_NOT_FOUND` | 快照 ID 缺失或已过期 | 重新快照 |
| `APP_NOT_FOUND` | 应用未运行 | `launch` 它 |
| `WINDOW_NOT_FOUND` | 无匹配窗口 | 检查名称，使用 `list-windows` |
| `ACTION_FAILED` | 操作被拒绝、阻止或结果矛盾 | 阅读 `error.details`。`checks[]` 报告会指出阻止的检查。投递后，阅读 `disposition.retry` 和 `details.post_state` |
| `ACTION_NOT_SUPPORTED` | 元素无法执行此操作 | 使用其他命令 |
| `POLICY_DENIED` | 物理或有头路径被阻止 | 如果意图是物理输入，添加 `--headed`。危险按键组合需要 `--force` |
| `APP_UNRESPONSIVE` | 在不确定的变更后，存活探测也失败 | 获取新快照。等待恢复后再重试 |
| `TIMEOUT` | 等待或可操作性条件未满足 | 阅读 `error.details.kind` |
| `INVALID_ARGS` | 参数错误，或针对受保护的系统进程执行关闭 | 检查语法 |
| `PLATFORM_NOT_SUPPORTED` | 适配器缺少此方法 | 使用其他方法 |
| `NOTIFICATION_NOT_FOUND` | 通知索引或指纹不再匹配 | 重新列出 |
| `INTERNAL` | 意外的操作系统失败 | 阅读 `message`。仅当 `disposition.retry` 为 `safe` 时重试一次 |

`TIMEOUT` 的 `details.kind` 值：`actionability_timeout`（目标从未变为可操作）、`wait_timeout`（`wait` 条件；携带 `predicate`、`timeout_ms`、`last_observed` 或 `last_error`）、`chain_deadline`（激活链超时；携带 `value_before`、`value_at_timeout`、`target`、`mutated`）。对于 `chain_deadline`，`mutated: true` 或未知的展开状态意味着重试前需重新读取。`mutated: false` 意味着直接重试是安全的。

## 可操作性

在 Ref 操作派发前，核心在一个有界预算内检查目标。`--timeout-ms`（默认 5000）设置该预算。对于 `drag`，同一截止时间也覆盖拾取、移动、放下延迟和完成；手势时间不额外计入。检查项为 `visible`、`stable`、`enabled`、`supported_action`、`policy`、`editable`，以及指针投递的 `receives_events`。每项为 `pass`、`fail` 或 `unknown`。完整报告在 `error.details` 中：`{ "actionable": false, "checks": [ { "check", "status", "reason" } ] }`。

- 瞬态检查（`visible`、`stable`、`enabled`、`receives_events`）约每 100 ms 轮询一次，直到通过。到期时结果为 `TIMEOUT` 或 `ACTION_FAILED`。
- 禁用的目标会等完整个预算，最终以 `kind: actionability_timeout` 结束 `TIMEOUT`；其 `last_report` 携带失败的 `enabled` 检查。同时处于屏幕外的目标在保持禁用期间不会被滚动，因此结果不取决于页面滚动位置。
- 在完整实时读取中不匹配任何元素的 Ref 会等完整个预算，最终以 `STALE_REF` 结束（若多个匹配则为 `AMBIGUOUS_TARGET`），而非 `TIMEOUT`。以这种已稳定 Ref 结束的 `hover` 或 `drag` 等待也会指出该代码。
- 终态检查（`supported_action`、`policy`、`editable`）无法通过等待修复。它们立即以 `ACTION_NOT_SUPPORTED` 或 `POLICY_DENIED` 失败。
- `hover` 和 `drag` 的每个 Ref 端点使用指针解析器：实时可见性和边界、一次滚动到视图尝试、第二次等边界采样，然后 `receives_events`。它们不需要元素操作。
- `receives_events` 仅在操作通过指针到达目标时运行：有头操作、`hover`、`drag`，或无语义投递的点击。通过语义操作派发的无头点击会跳过它。
- 当另一个元素位于顶层时，`receives_events` 以 `reason: "occluded by <role>"` 失败，并带 `occluder` 为 `{ role, name, bounds }`。将目标窗口带到前台或关闭遮挡物，然后重试。盲目重试会以同样方式失败。
- `receives_events` 失败开放。不确定的命中测试报告 `unknown`，操作继续派发。通过的检查不能证明没有东西覆盖目标。
- `details.kind: "live_read_incomplete"` 且 `retryable: false` 意味着每次轮询的原生读取都失败。获取新快照并使用新 Ref。
- 根据失败检查的 `reason` 选择恢复方式：`wait --element <ref> --predicate actionable`、新快照，或当 `policy` 失败且意图是物理输入时使用 `--headed`。

标准 Ref 操作会先尝试滚动到视图。如果不存在可滚动祖先，结果为 `ACTION_FAILED`，并带 `details.kind: "scroll_into_view_unsupported"` 和 `retryable: false`。改用 `scroll` 滚动包含的视口。可能已移动但无法确认的滚动携带 `delivered_unverified`。在假设无效果前重新读取元素。

## 点击

```bash
agent-desktop click @s8f3k2p9:e5
agent-desktop --headed double-click @s8f3k2p9:e3
agent-desktop --headed triple-click @s8f3k2p9:e2
agent-desktop right-click @s8f3k2p9:e5
```

- `click` 是主要激活方式。无头使用元素发布的激活方式。`--headed` 在可能时先物理点击，并在 `data.steps` 中报告 `physical_synthetic`。通过观察应用来判断投递，而非可访问性返回码。
- `double-click` 和 `triple-click` 是物理手势，无头下失败关闭。对于原始坐标，使用 `--headed mouse-click --xy X,Y --count 2`（或 `3`）。
- `right-click` 打开上下文菜单。然后执行 `snapshot --surface menu`。对于组合框和菜单按钮，改用 `select`。
- 右键点击在模态菜单打开后可能返回 `APP_UNRESPONSIVE` 且投递不确定。先查看效果。不要重试。

## 文本输入

```bash
agent-desktop type @s8f3k2p9:e2 "hello@example.com"
agent-desktop type @s8f3k2p9:e2 "hello world" --interval-ms 60
agent-desktop set-value @s8f3k2p9:e2 "new value"
agent-desktop clear @s8f3k2p9:e2
agent-desktop focus @s8f3k2p9:e2
```

- `type` 插入文本，并在值和选择可读时检查结果。可读的不匹配返回 `ACTION_FAILED`，不可读的结果以 `delivered_unverified` 成功。作为按键发送的文本通过应用的事件队列到达字段，因此检查在 `--timeout-ms` 预算内于字段变化时重新读取。如果它稳定在其他文本上，或从未稳定，结果为 `delivered_unverified`，并带 `verification_reason: postcondition_mismatch`。有头输入聚焦非活动字段可能改变选择，结果随后保持未验证。再次写入前用 `get` 读取字段。
- `type` 的默认 `--timeout-ms` 为 5000 加上文本每字符 30 ms。显式 `--timeout-ms` 优先。
- `type --interval-ms N`（1 到 2000）在一个命令内每 N ms 输入一个按键；`--timeout-ms` 随后应用于每个按键，`0` 使每个按键只尝试一次。一个按键保持组合标记、emoji 序列和 CR LF 在一起。第一个按键是完整 Ref 操作（有头时还会聚焦窗口），其余发送到同一元素，一次一个按键地获取交互租约。有头按键仅在目标进程和聚焦元素未变时投递；如果焦点移动，突发以错误停止。结果添加 `keystrokes` 和 `interval_ms`，`post_state` 是最后一个按键后的字段。仅当该读取显示完整文本时才保持 `delivered_verified`；否则为 `delivered_unverified`，并带 `verification_reason` 为 `postcondition_mismatch`、`insufficient_evidence`（在字段中途输入）或 `final_read_failed`（无 `post_state`）。`--wait-for` 在最后一个按键后等待一次。中途失败会添加 `details.typed_keystrokes` 和 `details.keystrokes`，且为 `delivered_unverified`：再次输入前读取字段。
- `set-value` 通过语义值路径直接写入值。它比 `type` 快，但可能不会触发每个 UI 回调。用于文本字段、文本区域和滑块。
- 不暴露值写入的单元格不能直接编辑。激活它，获取新快照，并定位出现的编辑器。某些应用不暴露编辑器，只接受键盘输入。
- `clear` 清空元素。`focus` 设置键盘焦点而不点击。

对于共享编辑器（如电子表格单元格），编辑是一个序列：激活单元格、快照、定位编辑器、写入、然后提交。从单个写入者运行整个序列。交互租约序列化单个操作，而非你的序列。

## 选择、切换、展开

```bash
agent-desktop select @s8f3k2p9:e4 "Option B"
agent-desktop check @s8f3k2p9:e6
agent-desktop toggle @s8f3k2p9:e6
agent-desktop expand @s8f3k2p9:e7
```

- `select` 按显示文本在列表、下拉框或组合框中选择选项。对于菜单支持的控件，它打开菜单、激活项并检查控件值。缺少项或值未变返回结构化错误。
- `check` 和 `uncheck` 是幂等的，并验证最终状态。优先使用它们而非 `toggle`。仅在当前状态未知时使用 `toggle`，它需要可读的前后状态。
- `expand` 和 `collapse` 作用于展开三角形、树项和手风琴，并验证最终状态。

## 滚动

```bash
agent-desktop scroll @s8f3k2p9:e1 --direction down --amount 3
agent-desktop scroll-to @s8f3k2p9:e8
```

- `scroll` 移动滚动区域。无头使用原生滚动路径，但在 Linux 上 `scroll` 是物理滚轮输入，需要 `--headed` 和 X11（见 Linux 技能）。有头在可能时发送物理滚轮手势。没有安全机制时返回结构化错误。
- `scroll-to` 将元素带入视图。成功意味着重新读取发现它可见。它需要 Ref。当只有 `bounds` 时，用 `scroll` 滚动容器或使用 `--headed mouse-wheel`。
- `mouse-wheel --x --y [--dx] [--dy]` 在屏幕点投递原始滚轮事件。用于没有可访问性滚动操作的画布和自定义滚动表面。

## 键盘

```bash
agent-desktop press return
agent-desktop press ctrl+shift+z
agent-desktop press cmd+s --app "TextEdit" --wait-for "button:Save" --wait-timeout 5000
```

- 键名：`return`、`escape`、`tab`、`space`、`delete`、`up`、`down`、`left`、`right`、`f1` 到 `f12`。修饰键：`cmd`/`meta`、`ctrl`、`alt`、`shift`，用 `+` 连接。修饰键名称和被阻止的组合因操作系统而异。阅读平台技能。
- 单独的大写字母会添加 Shift：`press H` 发送 `shift+h`。
- `--app` 或 `--window-id` 定位一个应用。当多个实例共享名称且仅 `--app` 为 `AMBIGUOUS_TARGET` 时，使用 `--window-id`。未限定范围的 `press` 发送到前台应用。
- `press` 报告 `delivered_unverified`，因为投递不能证明快捷键完成了工作。用 `--wait-for` 或 `--wait-for-gone` 等待预期效果一次。超时保留已投递结果，盲目重试不安全。
- 要用快捷键打开窗口，将按键和等待放在一个 `batch` 中，以便在投递前获取基线：`press`，然后 `wait --event window-opened`。等待失败不会使按键可安全重复。
- 危险快捷键被适配器以 `POLICY_DENIED` 拒绝。`--force` 仍会发送一个。给被阻止的快捷键添加修饰键通常会得到另一个被阻止的快捷键。

## `--wait-for` 和 `--wait-for-gone`

全局标志，轮询树直到选择器匹配（或停止