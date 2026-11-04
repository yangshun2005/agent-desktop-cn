# JSON 输出契约

每条命令都返回结构化 JSON：

```json
{
  "version": "2.6",
  "ok": true,
  "command": "click",
  "data": {
    "action": "click",
    "disposition": { "delivery": "delivered_unverified", "retry": "unsafe" }
  }
}
```

错误包含机器可读的错误码和恢复提示：

```json
{
  "version": "2.6",
  "ok": false,
  "command": "click",
  "error": {
    "code": "STALE_REF",
    "message": "@s8f3k2p9:e7 not found in current RefMap",
    "suggestion": "Re-run a snapshot to obtain fresh refs, then retry with the new ref (CLI: snapshot [--skeleton]; FFI: ad_snapshot then pass the printed qualified ref to ad_execute_by_ref)",
    "recovery": {
      "strategy": "refresh_snapshot_then_retry_original",
      "retryable": true,
      "requires_fresh_snapshot": true
    },
    "disposition": {
      "delivery": "not_delivered",
      "retry": "safe"
    }
  }
}
```

`error.code`、`error.message` 和 `error.disposition` 始终存在。`suggestion`、`recovery`、`platform_detail` 和 `details` 为空时省略。`recovery` 仅在 `disposition.retry` 为 `safe` 时出现，其 `retry_after_ms` 仅对 `TIMEOUT` 和 `APP_UNRESPONSIVE` 设置。

版本 2.2 将预算耗尽的快照原本返回的 2.1 `TIMEOUT` 错误替换为成功但明确标记为部分结果的响应，并新增 `data.complete`。快照数据始终携带 `complete`。预算耗尽的快照现在返回 `ok: true`、`"complete": false`、实际观察到的树、`"truncated": true` 和 `nodes_observed`，而不再返回 `TIMEOUT` 错误且无树。每个后代被截断的节点都携带 `"subtree_truncated": true`，读者可从根节点走到每个边界。消费者必须读取 `data.complete` 来判断树是否完整；`TIMEOUT` 错误不再表示树过大。`--root` 下钻不受影响：它替换现有快照内的 ref，因此不完整的观察仍返回 `TIMEOUT` 而非部分树。

版本 2.1 将 2.0 的 `error.retry_command` 字符串替换为结构化的 `error.recovery` 对象，并新增 `error.disposition`。消费者仅在 `disposition.retry` 为 `safe` 时才能从 `recovery.strategy` 选择恢复策略；不得盲目执行旧信封中的命令字符串。被移除的 `retry_command` 字段没有兼容别名。

## Disposition 取值

每个错误都携带 `disposition`，作用于目标的命令也可在结果中携带它。`delivery` 取值为 `unknown`、`not_delivered`、`delivery_uncertain`、`delivered_unverified` 或 `delivered_verified`。`retry` 取值为 `unknown`、`safe` 或 `unsafe`，由 `delivery` 推导：`not_delivered` 为 `safe`，其他所有非 `unknown` 值均为 `unsafe`。

`unknown` 表示命令未对投递做出任何判定。不携带投递证据的变更命令失败时保留该值，例如可能已生效的操作上的 `TIMEOUT`。只读命令不发送任何内容，无效参数不会到达应用，因此它们的错误以及 `APP_NOT_FOUND` 等查找失败携带 `not_delivered`。将 `unknown` 视为不可自动重试：读取 `error.code` 和消息，仅在 `disposition.retry` 为 `safe` 时选择恢复策略。

## 版本 2.4 迁移

有状态的 ref 操作（`set-value`、`type`、`clear`、`check`、`uncheck`、`toggle`、`expand` 和 `collapse`）现在在一次投递尝试后拒绝观察到的后置条件矛盾。回读失败保留适配器的错误码，并携带 `details.kind: "post_action_verification"`。仅缺少证据仍为成功的 `delivered_unverified` 结果，携带 `details.verification_scope: "unavailable"`；该结果和已验证的元素状态均不能证明应用层持久化。

macOS 上无法证明未投递的变更错误以不安全的 retry disposition 终止回退链。这影响激活、选择、展开、滚动和窗口操作；状态未变或不可读不能证明变更从未投递。值写入链在未验证投递后也停止，不再应用第二种机制。预备滚动后的失败携带 `details.preparatory_scroll`，记录视图可能已改变。其 retry disposition 随失败而定：未发送任何内容的拒绝为 `not_delivered`，可安全重试；其他失败保留各自的 disposition。

重试或选择替代操作前，无论 `error.code` 为何，都要检查 `disposition.retry`。错误可包含 `post_state` 和 `postcondition_satisfied`，但不抹去原始执行失败。相同的验证规则适用于 FFI ref 操作；C ABI 不变。Rust 的 `ActionResult::from_execution` 构造函数现在只接收 action 和 steps，返回不可失败的结果；实时验证属于 `execute_verified_action`。

## 版本 2.5 迁移

版本 2.5 对派发指针、拖拽、剪贴板、关闭应用和窗口操作的命令是破坏性载荷变更：`mouse-click`、`mouse-move`、`mouse-wheel`、`hover`、`drag`、`clipboard-set`、`clipboard-clear`、`close-app`、`move-window`、`resize-window`、`minimize`、`maximize` 和 `restore`。

无条件成功标志被移除，无兼容别名：`clicked`、`moved`、`scrolled`、`hovered`、`dragged`、`resized`、`minimized`、`maximized`、`restored`、`closed`、`cleared`、`close-app` 的 `requested` 字段，以及 `clipboard-set` 的内层 `ok`。每次成功调用都携带的标志不能证明任何操作。

已验证的窗口报告在 `details.verification_scope` 中指明确切的回读（`window_size`、`window_position`、`window_minimized_state` 或 `window_maximized_state`）。几何范围还声明 `details.tolerance` 和 `details.units`：macOS 为 2 点，Windows 为 8 px，Linux 为 2 像素。写入后的窗口失败保留其错误码、details 和 disposition，并新增 `details.target` 和已完成的 `details.steps`。

这些命令现在都在 `data` 内返回适配器自身的报告：

```json
{
  "action": "drag",
  "disposition": { "delivery": "delivered_unverified", "retry": "unsafe" },
  "steps": [ { "label": "pointer_drag", "outcome": "succeeded", "mechanism": "physical_synthetic" } ],
  "target": { "kind": "drag", "from": { "x": 10, "y": 20 }, "to": { "x": 80, "y": 20 } },
  "from": { "x": 10, "y": 20 },
  "to": { "x": 80, "y": 20 }
}
```

- `disposition.delivery` 仅在适配器回读了其声明的后置条件后为 `delivered_verified`；完成派发但无此证明为 `delivered_unverified`；仅当适配器观察到操作已满足（例如对已退出的应用执行 `close-app`）时为 `not_delivered`。`ok: true` 表示命令返回了已投递或已满足的结果，不表示应用效果已验证。
- `target.kind` 为 `window`（`id`、`title`、`process`）、`app`（`name`、`process`）、`pointer`（`point`）、`drag`（`from`、`to`）或 `clipboard`。适配器提供其验证并使用过的身份。指针目标是派发坐标，不能证明哪个应用接收了输入。
- 请求上下文（`x`、`y`、`count`、`dx`、`dy`、`width`、`height`、拖拽端点、`drop_delay_ms`、`type`、`app`、`method` 以及确认的 `focused`）保持原有含义。这些字段回显请求，不是观察到的后置状态。
- 错误保留 `error.disposition`。已投递操作后 `--wait-for` 失败时，在 `error.disposition` 中保留操作自身的投递状态（已验证、未验证或未投递），并在 `details.after_action` 下携带报告。
- Rust 的 `ActionResult` 和 `OperationReport` 读取器拒绝没有显式 `disposition` 的载荷；缺失 disposition 不再回读为 delivered-unverified。
- C ABI 不变。`ad_close_app` 和 `ad_drag` 等仅返回状态的函数仍无法暴露成功调用的 disposition、steps 或 target。

## 版本 2.6 迁移

版本 2.6 改变了 Windows 和 Linux 上哪些 ref 能解析，以及 macOS 上不产生地址的元素。此类 ref 仅当其窗口的一次完整读取在捕获时证明其唯一，且当前恰好有一个活动元素匹配其角色和稳定字段时才解析。唯一新增的载荷字段是 `data.identity_unproven`，以及在拒绝后预算耗尽的 ref 操作上的 `error.details.refusal`。

- 指向两个相同控件之一的 ref，此前按位置或几何解析，现在在读取任何内容前以 `AMBIGUOUS_TARGET` 失败，`details.kind: "identity_not_unique"`。从快照超时内无法完整读取的窗口捕获的 ref 同样如此。
- 2.6 之前写入的 refmap 不记录捕获证明，因此这些平台上其所有 ref 都以相同方式失败。请重新拍摄快照。
- 捕获后出现的相同孪生元素以 `AMBIGUOUS_TARGET` 失败，`details.kind: "saved_ref_ambiguous"`。
- 所有这些都携带 `details.retryable: false` 和 `not_delivered` disposition。`disposition.retry` 仍为 `safe`，因为未发送任何内容，但重复同一 ref 会返回相同拒绝。
- 当 ref 操作在租约下的完整解析被以可能清除的方式拒绝时，操作在其 `--timeout-ms` 内再次等待。若该预算随后耗尽，错误在 `details.refusal` 中携带拒绝信息：其 `code`、`message`，以及拒绝具有的任何 `details`。
- 操作系统不具备的 surface 类型现在返回 `INVALID_ARGS` 而非 `PLATFORM_NOT_SUPPORTED`，disposition 为 `not_delivered`，`retry: "safe"`。FFI 调用者看到匹配的 `AdResult` 错误码变化。完全没有 surface 的适配器仍返回 `PLATFORM_NOT_SUPPORTED`。
- 在 macOS 上，`open-system-surface --surface dock` 和 `--surface menu-bar-extras` 现在可无头成功。其 `data` 没有 `window`；读取前请检查该字段。Windows 和 Linux 仍返回 `window`，需要 `--headed`。
- 在 macOS 上，对不列出窗口的后台应用（菜单栏应用、Dock、控制中心）拍摄快照，以及 `--surface dock` 或 `--surface menu-bar-extras`，现在成功并通过其进程读取应用。此前它们失败 `WINDOW_NOT_FOUND`。此类快照省略 `data.window`，`data.app` 指明进程；读取 `data.window.id` 前请检查该字段。

需要一次完整窗口读取来证明其 ref 但无法获得的快照，在 `data` 中以 `"identity_unproven": true` 表明。`complete` 仍描述呈现的树。在 Windows 和 Linux 上，以及 macOS 上无地址的元素，该快照的每个 ref 都以 `AMBIGUOUS_TARGET` 失败，`details.kind: "identity_not_unique"`；请用更大的 `--timeout-ms` 重新拍摄快照。其他情况省略该字段。

## 有头操作验证

- 有头 ref 交互在最终指针命中测试前激活确切的源应用/窗口。只读就绪轮询不得阻止后台目标到达该聚焦阶段；投递前的最终命中测试仍为强制。
- 在 macOS 上，通过后台应用进程读取的 ref 没有可激活的窗口，后台应用（菜单栏应用、Dock、系统面板）中的目标或状态项跳过最前台应用和主窗口检查，因为此类应用永远不会成为最前台。最终命中测试仍把守投递。任何其他没有已验证所属窗口的目标都被拒绝。
- 聚焦和预备滚动可能改变坐标。稳定性针对新的实时观察验证，而非要求坐标与原始快照保持相等。
- 在 macOS 上，激活后的聚焦确认使用新的系统级 `AXFocusedApplication` 和带进程代际检查的确切窗口证据。在刚刚激活了另一应用的进程中，缓存的 `NSWorkspace` 前台快照不够。
- 聚焦非活动文本字段可能改变其选择。聚焦前范围无法确定对非空字段的预期插入；此类结果明确为未验证，而非被错误地判定为矛盾。
- 稳定循环的节奏预算不替代调用者对原生读取的截止时间。测试断言截止时间同一性，而非任意剩余时间阈值。

## 查找结果形状

`find` 匹配对象报告元素的实际无障碍 `name`。没有无障碍名称的元素完全省略 `name`；其内容保留在 `value` 中。早期输出合成显示名称，回退到 `value` 或 `description`，再回退到 `(unnamed <role>)` 占位符，因此将 `name` 视为始终存在的消费者必须容忍其缺失，并对 `role` 或 `value` 应用自己的回退。

## Ref 重新识别详情

每个 ref 都从其存储条目重新识别或被拒绝，`details.kind` 指明是哪种。核心解析器以以下四种之一拒绝保存的 ref：

| `details.kind` | 错误码 | 含义 |
|------|------|---------|
| `retained_object_stale` | `STALE_REF` | macOS：元素记录的地址已失效，或其进程、角色或记录的标识符不再匹配。`details.reason` 为 `dead`、`pid`、`role` 或 `native_identifier`。不运行搜索，因此 ref 不会落到相似元素上。请重新拍摄快照。 |
| `saved_ref_absent` | `STALE_REF` | 在完整证据下没有活动元素匹配存储的角色和稳定字段，元素已消失。请重新拍摄快照。 |
| `saved_ref_ambiguous` | `AMBIGUOUS_TARGET` | 出现了与捕获元素相同的元素，因此两个或更多活动元素匹配存储的角色和稳定字段。新快照会将两者都作为孪生元素拒绝；请操作窗口中唯一的控件。 |
| `identity_not_unique` | `AMBIGUOUS_TARGET` | 捕获时未证明该 ref 是唯一匹配其角色和稳定字段的元素，因此不搜索任何内容。要么存在相同孪生元素，要么窗口无法在快照超时内完整读取（请用更大的 `--timeout-ms` 重新拍摄快照）。在 Windows、Linux 以及 macOS 上元素无地址时会出现。孪生元素被每个快照拒绝；只有显式坐标点击（`--headed mouse-click --xy`）能选中一个，它会占据前台，且在 Linux Wayland 会话中不可用。 |

第五种结果是 `resolution_identity_unknown`，携带 `TIMEOUT`：无法读取某个活动字段，身份计数保持未证明。它可重试。Linux 菜单和托盘 ref 报告与 `STALE_REF` 相同的 kind。

核心解析器选定元素后，平台适配器落地它，落地失败保留适配器自身的 `details.kind`（例如 Linux 和 Windows 上的 `resolve_no_candidate`，或 macOS 上的 `resolution_complete_absence`）。平台技能列出这些 kind。`saved_ref_absent`、`saved_ref_ambiguous` 和 `identity_not_unique` 携带 `details.retryable: false`，因为等待无法改变答案。所有这些均为 `not_delivered`。

## 错误码

| 错误码 | 含义 |
|------|---------|
| `PERM_DENIED` | 未授予无障碍权限 |
| `ELEMENT_NOT_FOUND` | 没有元素匹配 ref 或查询 |
| `APP_NOT_FOUND` | 应用未运行或无窗口；查找在发送任何内容前失败，因此 disposition 为 `not_delivered` |
| `STALE_REF` | 无法在活动 UI 中重新识别 ref |
| `AMBIGUOUS_TARGET` | ref 恢复匹配到多个可能目标 |
| `SNAPSHOT_NOT_FOUND` | 快照 ID 缺失或已过期 |
| `POLICY_DENIED` | 物理/有头路径被策略阻止 |
| `ACTION_FAILED` | 操作被拒绝，或观察到的结果与请求矛盾 |
| `ACTION_NOT_SUPPORTED` | 目标不暴露请求的操作 |
| `APP_UNRESPONSIVE` | 匹配的应用停止响应 |
| `PLATFORM_NOT_SUPPORTED` | 适配器方法在此平台未实现 |
| `TIMEOUT` | 等待条件过期，或活动身份读取直到截止时间仍不完整 |
| `INVALID_ARGS` | 参数值无效 |
| `WINDOW_NOT_FOUND` | 请求的窗口不存在，包括不匹配任何窗口的