# Linux 实时端到端测试装置

在私有的临时桌面会话中，驱动构建好的 `agent-desktop` 二进制文件对真实的 GNOME 和 Qt 应用执行操作。每个效果都通过独立读取（原始 AT-SPI、X11、GSettings、进程、像素、剪贴板或 D-Bus）验证，绝不依赖命令自身的 `ok:true`。

## 运行

构建 CLI 及其相邻的剪贴板所有者，并在一次调用中运行一个层级：

```sh
bash tests/e2e-linux/run.sh --tier x11-openbox
```

每次运行都会构建（`cargo build --locked --release`）到该套件自己的临时 Cargo target 中，将两个可执行文件暂存到 `<suite>/bin/`，并将其 SHA256 记录在相邻的 `.sha256` 文件和启动时的 `image:` 行中。随后删除构建中间产物。每个 leg 都使用这对固定版本。暂存哈希不匹配会导致运行失败，其他工作树的构建也无法替换任一镜像。外部提供的 `AGENT_DESKTOP_E2E_BIN`（或 `AGENT_DESKTOP_E2E_BIN_SRC`）会跳过构建，并在获取交互租约之前冻结该镜像及其相邻辅助程序。

省略 `--tier` 以顺序运行所有层级。将 `AGENT_DESKTOP_E2E_SUITE` 设为一个新目录以选择证据位置。默认位置为：当 `$AGENT_DESKTOP_SESSION_BASE` 非空时使用其下的新目录；否则当 `XDG_RUNTIME_DIR` 非空时使用 `$XDG_RUNTIME_DIR/agent-desktop-sessions`；否则使用 `/tmp/agent-desktop-sessions-<uid>`（数字用户 ID）。基础目录以 0700 模式创建。运行器启动自己的私有会话并获取交互租约。

退出码：`0` 表示每个 leg 都通过或断言了其拒绝行为；`1` 表示其他任何结果（见结果）；`2` 表示门禁拒绝启动（缺少前置条件、未知 leg、leg 未注册到该层级、暂存失败或交互租约已被持有）。退出码 2 时未运行任何内容。

保留失败的套件用于诊断；运行器会删除成功的套件。使用另一个新套件进行唯一的诊断重跑。不要用不同的二进制镜像复用套件。

### 前置条件

`runner/gate.py` 是权威列表；缺少任何一项都会在任何 leg 运行前以退出码 2 退出。

- Python 3 及 PyGObject（`import gi`）。
- 基础工具：`Xvfb`、`dbus-daemon`、`dbus-update-activation-environment`、`gdbus`、`busctl`、`gsettings`、`setsid`、`xprop`、`xwininfo`、`xwd`、`fusermount`、`timeout`。
- `x11-openbox`：`xdotool`、`wmctrl`、`xclip`，以及位于 `$AGENT_DESKTOP_WM_ROOT/usr/bin/openbox` 的 openbox 二进制文件（`AGENT_DESKTOP_WM_ROOT` 默认为 `/`）。
- `x11-mutter`：`gnome-shell`、`xdotool`、`wmctrl`、`xclip`。
- `wayland`：`gnome-shell`、`wl-copy`、`wl-paste`。
- 这些 leg 还会驱动 GNOME 应用 Calculator、Text Editor、Files、Terminal 和 Settings，以及一个 PyQt6 测试窗口。私有会话中必须提供 AT-SPI。

运行器继承调用者的工具路径和库：通过 `PATH` 提供工具，通过 `LD_LIBRARY_PATH` 提供共享库。Qt leg 在 `python3` 下运行；当 PyQt6 位于单独的解释器中时，设置 `AGENT_DESKTOP_QT_PYTHON`。这不会改变测试装置的解释器。

## 层级

- `--tier x11-openbox`：私有 Xvfb 和 openbox。
- `--tier x11-mutter`：私有 Xvfb 和私有 mutter shell 包装器。
- `--tier wayland`：私有总线和一个经过健康检查的无头 GNOME Wayland shell。

每个层级运行注册到该 leg 的无头和有头模式。无头合成器上的有头模式会在该私有会话内发送合成输入，而非发送到用户桌面。

`events.open_and_close_inside_one_wait` 和 `ref_actions.drag` 仅在 openbox 上注册。在 Mutter 上，第一个 leg 因 AT-SPI 节点消失而拆除。第二个在编辑器注册表设置期间超时，尚未派发任何产品命令。两者都不算 Mutter 通过。未注册到某层级的 leg 在该层级不做实时声明。

## Leg

使用 `--leg <id>` 配合相同层级来复现单个已注册的 leg：

```sh
bash tests/e2e-linux/run.sh --tier x11-openbox --leg clipboard.text_roundtrip
```

`runner/legs.py` 是 ID、层级、模式、预言机和处置的清单。传入 `--leg id1,id2,id3` 以在一个私有层级会话中运行有界批次。每个请求的 ID 都必须注册到所选层级；不匹配则以退出码 2 退出。单个 ID 会执行其所有已注册模式；它不是模式选择器。

通过要求独立的操作前后观察、状态变更处的恢复，以及清理后相同的归属应用进程代。拒绝 leg 必须断言其预期错误且无效果。

每个 leg 的清理会停止每个归属应用进程代。私有后台服务可能独立启动或退出；其成员关系不显示应用泄漏。私有会话拆除会清扫所有运行时成员，若有任何存活则失败。

列出某层级已注册的 ID、模式和处置而不启动应用：

```sh
python3 -I - <<'PYLIST'
import sys
sys.path.insert(0, "tests/e2e-linux")
from runner.legs import REGISTRY
for leg in REGISTRY:
    if "x11-openbox" in leg.tiers:
        print(leg.id, ",".join(leg.modes), leg.disposition, leg.reason)
PYLIST
```

将层级字符串改为 `x11-mutter` 或 `wayland` 以获取那些清单。

### 结果

报告按层级和模式对结果分组。只有 `pass` 和 `asserted-refusal` 以退出码 0 退出。

| 结果 | 含义 |
| --- | --- |
| `pass` | 每个操作前、操作后和恢复后的读取都一致，且归属进程集未变。 |
| `asserted-refusal` | 该 leg 的预期结果是产品以命名错误码拒绝且无效果；已观察到。 |
| `fail` | 断言失败、预言机读取缺失或乱序、状态漂移且未恢复，或归属进程集改变。 |
| `error` | 意外异常、证据写入失败、暂存二进制文件改变或拆除失败。 |
| `blocked` | 该 leg 抛出 `NotImplementedError`。 |
| `unimplemented` | 该 leg 已注册但无实现。 |
| `outstanding`、`owner-decision` | 已注册但故意不运行；原因会被打印。 |
| `tier-unavailable` | 该层级的私有会话从未变为健康，或其桌面预热失败，因此其 leg 未运行。 |

`launch.recovery_strategy_on_error` leg 是一个 `asserted-refusal`。它在快照后停止 Calculator，然后要求点击已死所有者的 ref 以返回 `STALE_REF`，并带有 `details.process_state=exited`、`details.retryable=false` 和处置 `not_delivered`/`safe`，且无 `recovery` 对象且不重新启动所有者。

`error_contract.recoverable_error_envelopes` leg 在两个 X11 层级上都是 `asserted-refusal`。针对活跃的 Calculator，它要求裸 `@e1` 返回 `INVALID_ARGS` 及快照限定的建议，`@sdeadbeef:e1` 返回 `SNAPSHOT_NOT_FOUND` 及 `refresh_snapshot_then_retry_original` 恢复，未知 `--app` 返回 `APP_NOT_FOUND` 及 `list-apps` 建议。每个都必须是 `not_delivered` 并以退出码 1 退出，且 Calculator 显示和归属进程集必须不变。

`observation.stale_ref_after_ui_change` leg 在两个 X11 层级上都是 `asserted-refusal`。它为 Calculator 的 About 菜单项生成一个 ref，通过原始切换关闭菜单，并要求对该 ref 的 `click` 返回 `STALE_REF` 及 `refresh_snapshot_then_retry_original` 恢复、`not_delivered` 和退出码 1，同时应用保持归属且显示不变；随后新快照必须生成不同的 ref。

`text_entry.press_combos_edit_document` leg 在两个 X11 层级上有头运行：在语义 `type` 之后，`press return` 必须向 AT-SPI 文档文本精确添加一个换行，`press ctrl+a` 然后 `press backspace` 必须清空它，且任何 X11 修饰键都不得保持按下。无头 `press` 会在发送任何按键之前被物理输入策略拒绝，因此只有有头运行能证明投递。

`batch_contract.entry_failure_and_stop_on_error` leg 在两个 X11 层级上都是 `asserted-refusal`。一个三条目批次（点击 7、点击不存在的 ref、点击 8）必须在 `ok` 批次信封内报告条目 1 的 `STALE_REF`，并仍运行条目 2，在 Calculator 显示上留下 `78`。同一批次使用 `--stop-on-error` 必须在索引 1 处报告 `stopped` 为 `stop_on_error`，两个已完成条目和一个未开始，留下 `7`。显示通过 AT-SPI 读取，而非信封，Undo 会恢复它。

### Ref 操作契约

`ref_actions.*` leg 观察并恢复原生应用状态：

| 操作 | 独立效果 |
| --- | --- |
| 聚焦 | Qt 文本输入获得 `FOCUSED`；恢复时先前的按钮重新获得它。 |
| 双击 / 三击 | Text Editor 选择一个词 / 行，通过 AT-SPI Text 选择偏移读取。 |
| 右键单击 | Text Editor 暴露一个显示中的 `Context menu` 及其菜单项子项。 |
| 悬停 | Qt 探针的原生状态提示出现，指针离开时清除。 |
| 展开 / 折叠 | Calculator 的 Main Menu 同时改变 `EXPANDED` 和显示中的菜单子项。 |
| 滚动 | Settings 的 Hot Corner 离开视口，Dynamic Workspaces 进入视口。 |
| 滚动到 | Settings 显示屏幕外的显示器复选框而不改变其勾选状态。 |
| 拖拽 | Text Editor 获得非空文本选择。 |

无头聚焦、双击、三击、悬停、滚动和拖拽是单独的 `asserted-refusal` leg，要求 `POLICY_DENIED`、无应用效果且指针和焦点不变。右键单击、展开/折叠和滚动到在两种模式下都有语义成功 leg。`pointer.*` leg 在已投递和未投递两种情况下，将产品的投递报告与应用状态的独立读取进行比较。伪造报告或仅指针移动不能通过。`mouse-click`、`drag` 和 `mouse-wheel` 投递 leg 读取实际效果：Calculator 数字、Text Editor 选择或滚动条位置。Text Editor 通过原始 `EditableText` 在内存中播种，选择清理仅在要求确切归属窗口处于活动状态后才发送 Right。

### 通知和托盘 leg

通知 leg（`notifications.*`）在 `x11-mutter` 上运行，其中私有 GNOME shell 拥有通知服务器；`oracle/notify.py` 检查服务器的进程属于私有运行时目录。一个单独的发送者进程发布通知，`dbus-monitor` 在 `notification-signals.log` 中记录守护进程的信号。leg 通过发送者的 ID、那些信号和列表回读，并在下一个 leg 之前关闭每个归属 ID 并检查恢复。

- 无头 `dismiss-notification`、`dismiss-all-notifications` 和 `notification-action` 必须返回 `POLICY_DENIED`，无守护进程信号且列表不变。
- `notifications.dismiss_notification_closes_only_target` 是有头的。它播种一个目标和旁观者，要求错误的 `--expected-app` 返回 `NOTIFICATION_NOT_FOUND` 且无信号且列表不变，然后要求仅目标的守护进程关闭（原因 2）、旁观者仍活跃，以及列表仅包含旁观者。

托盘 leg（`tray.*`、`x11-mutter`、无头）注册一个测试装置 StatusNotifierItem（`apps/sni_item.py`）并读取其调用日志：`Activate`、菜单 `Event(id, clicked)`、对没有 `Activate` 的项的拒绝，以及被杀死的所有者将旧 ref 变为 `STALE_REF`。

### 覆盖边界

注册表是清单；这些表面没有实时声明，其覆盖存在于命名的测试中。

- Electron 和 Chromium：此注册表中没有 leg。
- 剪贴板 HTML 导入：设计上不支持。`clipboard-get` 仅读取文本和图像目标（`formats::TEXT_TARGETS`、`targets::choose_target`；`x11_read_tests::an_owner_without_the_wanted_target_reads_as_absent`）。剪贴板 leg 覆盖文本、图像、所有者生命周期、清除以及 CLI 退出后的外部读取。
- 显示值上的正向 `select`：仅 `select.refusal_without_semantic_route` 是实时的。`crates/linux/src/actions/select/select_tests.rs` 和 `select_search_tests.rs` 覆盖路由、身份、效果和拒绝。
- Wayland：私有 shell 没有键盘 seat，因此聚焦和窗口捕获仅作为失败关闭的拒绝（`wayland_actions.*`、`gaps.wayland_window_capture_refused`）实时存在。其他 Wayland leg 检查 `list-windows` 不报告位置、无 seat 的语义点击有效，以及不健康的 shell 以命名原因使层级失败。

## 阅读失败

打印的报告按层级和模式对每个结果分组。失败的运行在打印的套件目录中保留 `report.txt` 和每个 leg 的 JSON 证据。证据包括产品 argv、响应信封、预言机读取和进程成员关系。先读 `report.txt`，然后用 `python3 -m json.tool` 美化打印匹配的 `evidence/<tier>/<mode>/<leg-id>.json`。其 `exception_traceback`、`products`、`reads` 和 `owned_baseline`/`owned_final` 显示失败发生在派发之前、效果检查处还是恢复期间。匹配的确定性产品失败需要在诊断重跑中使用相同的命令、前置条件和独立的效果/无效果读取。产品派发之前的原始总线、启动、时序或服务成员关系失败是夹具/环境失败；退役时保留其证据。

最多重跑失败的 leg 一次，以区分确定性产品缺陷与计时、负载、会话设置或环境行为。删除不稳定的 leg 或用确定性假件或观察状态等待替换它，并在 `test:` 提交中记录原因。仅对确定性产品缺陷修复产品代码，尤其是错误目标输入或虚假投递证据。

### 阴性对照

`--lying-actions` 用一个总是返回 `ok:true` 但没有真实 ref 或效果的假件替换 CLI，因此任何选择都必须失败。它可能在操作之前的 ref 获取处失败；可移植的撒谎效果测试单独行使效果和恢复检查。使用新套件在本地运行此预期失败：

```sh
bash tests/e2e-linux/run.sh --tier x11-openbox --leg state.text_editor_semantic_type --lying-actions
```

前置条件门禁仍要求存在发布二进制文件路径（`AGENT_DESKTOP_E2E_BIN`、`AGENT_DESKTOP_E2E_BIN_SRC` 或 `target/release/agent-desktop`）；假件在门禁通过后替换它。阴性对照不证明清理；那是下面的取消检查。

### 取消检查

启动时，运行器打印 `suite:` 和两个 `image:` 哈希。当私有层级的可访问性健康门禁通过时，它原子地发布 `<suite>/<tier>.ready.json` 并打印 `cancel-ready:` 及该路径、运行时目录、`harness-owner` 标记、运行器 PID 和 Linux `/proc` 启动令牌（字段 22，自启动以来的滴答数）。标记包含 `<pid> <start-token>`。后台 shell/交互锁包装器 PID 不是运行器 PID。

此有界检查等待回执，而非猜测 PID 或在信号前休眠：

```sh
export CARGO_INCREMENTAL=0
base=${AGENT_DESKTOP_SESSION_BASE:-${XDG_RUNTIME_DIR:+$XDG_RUNTIME_DIR/agent-desktop-sessions}}
base=${base:-/tmp/agent-desktop-sessions-$(id -u)}
umask 077
mkdir -p "$base"
suite=$(mktemp -d "$base/e2e-cancel.XXXXXX")
AGENT_DESKTOP_E2E_SUITE="$suite" bash tests/e2e-linux/run.sh \
  --tier x11-mutter --leg ref_actions.right_click >"$suite/run.log" 2>&1 &
job=$!
python3 -I tests/e2e-linux/runner/cancel.py "$suite/x11-mutter.ready.json" --wait 600
signal_rc=$?
wait "$job"; rc=$?
cat "$suite/run.log"
test "$signal_rc" -eq 0 && test "$rc" -eq 143 || exit 1
python3 -I tests/e2e-linux/runner/cancel.py "$suite/x11-mutter.ready.json" \
  --check-cleanup --wait 60
```

`cancel.py` 在通过 Linux PID 描述符发送 SIGTERM 之前