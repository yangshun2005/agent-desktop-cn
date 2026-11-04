# Linux 上的窗口与生命周期

窗口控制通过 X11 显示上的 EWMH 客户端消息实现：X11 会话的显示，或 Wayland 会话的 Xwayland 显示。不使用 `wmctrl` 和 `xdotool`。

## 为应用命名

- `--app` 接受 `list-apps` 打印的进程名，或桌面条目的 `Name` 或 id——其 `Exec` 解析为正在运行的可执行文件（`/proc/<pid>/exe`）且完整参数匹配。多个条目共享的桌面名称不获得别名。注册表标签和截断的 `comm` 名称永远不是别名。
- 作用域匹配精确且不区分大小写，进程必须属于本会话。`list-windows --app` 使用该精确规则。`list-apps --app` 将所列名称的子串匹配与这些精确适配器别名结合。
- 窗口 id 是不透明地址，长度 34 到 70 个字符，在窗口进程的整个生命周期内有效。
- `list-apps` 行携带 `name`、`pid`、`bundle_id`（已知时为桌面 id）和 `process_instance`。`list-windows` 行携带 `id`、`title`、`app_name`、`pid`、`is_focused`、`minimized`、`visible`、`accessible` 和 `process_instance`，当显示映射了边界时还携带 `bounds`。
- 同一应用的多个实例共享一个名称：用 `--window-id` 选择一个。`close-app` 和 `launch` 无法选择，返回 `AMBIGUOUS_TARGET`。
- 同一进程的两个同标题窗口，通过工具包报告为活动的那个来区分：`focus-window --window-id` 和窗口操作作用于活动窗口，且当恰好两个窗口共享标题时，作用于其孪生窗口。任何其他平局（无活动窗口、三个或更多孪生窗口中活动窗口不是其中之一、活动窗口属于另一标题）都以原因 `ambiguous_window` 拒绝。
- `list-windows` 先列出聚焦窗口，然后按 X11 堆叠顺序从上到下列出其余窗口。没有单一 X11 客户端的窗口（活动窗口无法区分的孪生窗口、Wayland 会话）随后按注册表顺序列出。

## `launch`

```bash
agent-desktop launch gnome-calculator
agent-desktop launch gnome-text-editor --arg=/tmp/note.txt
```

- 标识符依次尝试：绝对路径、桌面 id、裸命令或桌面条目 `Name`（`launch "Text Editor"`）。相对路径，或含分隔符或 NUL 的名称，为 `INVALID_ARGS`、`not_delivered`；未知名称为 `APP_NOT_FOUND`；`Terminal=true` 或无 `Exec` 的条目为 `ACTION_NOT_SUPPORTED`。
- 应用以空 stdio 在其自己的进程组中启动，因此通过管道读取的调用者在退出时看到文件结束。每次启动都会向子进程环境添加 `AGENT_DESKTOP_LAUNCHED=1`；`close-app` 读取它以区分你启动的应用和会话服务。
- 绝对 `.desktop` 路径在受信任的 XDG 数据目录内启动该确切文件。桌面 id 使用 XDG 目录优先级。
- 仅当解析后的可执行文件等于其 `/proc/<pid>/exe` 且完整参数和调用名称匹配时，`launch` 才附加到正在运行的进程。缺少证据则启动新进程；仅名称永不附加。按名称不跟随移交到另一可执行文件的启动器：若无验证的所有者出现，`launch` 报告未解析的移交。
- 一个无参数的已验证匹配返回所有者；有参数则启动命令以转发它们。`--no-attach` 以 `ACTION_FAILED` 拒绝任何已验证的运行中匹配并列出其 pid；没有它时，多个已验证匹配为 `AMBIGUOUS_TARGET`。
- `--arg` 传递一个参数并可重复；`--arg --info` 按原样工作，`--arg=-x` 是以连字符开头的值的安全形式。移交到运行中实例会应用参数但不应用 `--env` 或 `--cwd`，结果会说明这一点。
- 负数需要 `=` 形式：`move-window --x=-100 --y=-20`。
- 窗口结果命名其注册表所有者，通过进程身份验证。命令等待窗口：注册表所有者获得 2 秒的首窗口宽限，若窗口已打开则为新窗口 5 秒。`--timeout`（默认 30000）限制两者；`--activate` 将剩余完整超时用于窗口等待。
- 被拒绝的 `--activate` 不会使 `launch` 失败：它返回 `ok: true`，`data.suggestion` 以 `The window could not be brought forward:` 开头并给出原因，且 `window.is_focused` 为 `false`。读取 `data.suggestion`：`window.is_focused` 是请求前的状态，因此仅 `false` 不表示拒绝。对该窗口运行 `focus-window` 会报告原因。
- 无法证明帧的原生窗口类型的窗口读取（`details.kind: native_window_unproven`，窗口在读取期间发生变化时会给出）视为尚无窗口，并重新读取直到超时，附加和新启动均如此。若届时仍无窗口被证明，`launch` 以该 kind 返回 `ACTION_NOT_SUPPORTED`：未启动任何进程时为 `not_delivered`，启动后为 `delivered_unverified`。
- 若启动的进程从未在无障碍总线上注册，所有者查找可能消耗完整超时，结果则无 `window`。仍在注册表外运行的已启动进程可无窗口返回，且可能是包装器。无窗口出现时结果可为 `ok` 且无 `window`。
- 无所有者退出的启动器为 `ACTION_FAILED`、`details.kind: exited`（信号后为 `crashed`）、`delivered_unverified`。所有者在其窗口出现前退出为 `ACTION_FAILED`、`delivered_unverified`，无该 kind。转发启动器可在同一可执行文件的现有实例打开新窗口时附加到该实例，即使其 argv 不同。若无法证明所有者，包括仅在现有窗口中打开标签页的移交，已退出的启动器可返回 `ACTION_FAILED`、`delivered_unverified`；重试前重新列出窗口。
- 结果携带 `app`、`pid`、`process_instance`，以及出现时的 `window`。无参数附加返回运行中应用的现有窗口。新启动或参数移交仅报告新窗口，因此在现有窗口中打开标签页的移交可返回无 `window`。
- Chromium 和 Qt 应用需要标志或环境变量才能出现在总线上；用 `--arg` 或 `--env` 传递。Electron 应用不需要。参见 `references/troubleshooting.md`。

## `close-app`

`close-app` 关闭应用，包括其所有窗口，仅当进程消失时才成功。核心 `commands-system` 参考涵盖关闭单个窗口。

单实例应用（如 GNOME Text Editor）可将带参数的 `launch` 移交给已运行的实例。当恰好一个匹配实例打开新窗口时，`launch` 报告该实例及其新窗口。无法归属的移交即使请求已生效，仍可返回 `ACTION_FAILED`（`details.kind: exited`、`status: 0`）。`close-app` 关闭其目标实例的每个窗口，包括你未打开的窗口。
要启动自己的实例，用 `--arg` 传递应用自身的标志（Text Editor：`--arg=--standalone`）。

示例：打开 Text Editor 窗口 `a.txt` 和 `b.txt` 时，此 Bash 配方用 `jq` 选择唯一的 `b.txt` 窗口及其 Close 按钮，点击它并等待该窗口关闭。窗口标题为 `b.txt (~/dir) - Text Editor`，因此过滤器匹配标题开头。任一选择有歧义时配方停止，不选第一个匹配。

```bash
(
  set -euo pipefail
  window_id=$(agent-desktop list-windows --app gnome-text-editor |
    jq -er '[.data[] | select(.title | startswith("b.txt"))] |
      if length == 1 then .[0].id else error("choose one window") end')
  close_ref=$(agent-desktop snapshot --app gnome-text-editor --window-id "$window_id" -i --compact |
    jq -er '[.. | objects | select(.role? == "button" and .name? == "Close" and .ref_id?) | .ref_id] |
      if length == 1 then .[0] else error("choose the window Close button") end')
  batch=$(jq -nc --arg ref "$close_ref" --arg id "$window_id" \
    '[{command:"click",args:{ref_id:$ref}},
      {command:"wait",args:{event:"window-closed",app:"gnome-text-editor",window_id:$id,timeout:10000}}]')
  agent-desktop batch "$batch" --stop-on-error |
    jq -e '.data.results[1].data.event // error("the window did not close")'
)
```

点击在窗口消失前返回，因此紧随其后的 `list-windows` 仍可列出该窗口。批处理等待 `window-closed` 并打印已关闭窗口的事件。若出现保存提示，等待超时：快照 `--surface sheet`，回答它，然后运行 `list-windows`。

- 优雅关闭向该 pid 的每个顶层窗口发送 `_NET_CLOSE_WINDOW`，不发送信号。比其窗口存活更久的文件管理器以 `TIMEOUT`、`delivered_unverified` 结束；`--force` 结束它。终端服务器像任何其他应用一样以 `list-apps` 打印的名称关闭：`close-app gnome-terminal-server` 结束它，`close-app gnome-terminal` 为 `APP_NOT_FOUND`。
- 未确认的发送是不确定的，重试不安全，即使剩余扇出耗尽时间：先观察应用。
- 保存提示使应用保持打开。对话框是同一 pid 的新顶层窗口，或绘制在窗口内的 `dialog` 或 `alert` 节点（libadwaita 如此），计入 `details.dialogs`：用普通 `snapshot --app <name>` 或 `--surface sheet` 读取它，回答它，再次关闭。已存在的对话框：`ACTION_FAILED`、原因 `dialog_pending`、`not_delivered`、`retry: safe`，未发送任何内容。为响应请求而打开的对话框以相同原因结束等待，标题在 `details.dialog` 中，`delivered_unverified`、`retry: unsafe`。一个请求可为每个有未保存工作的窗口打开提示，因此 `dialogs`、`windows_remaining` 和 `dialog_titles` 均来自窗口停止变化后的一次读取，且彼此一致。
- 多个提示是独立的模态窗口，可按任意顺序回答：此时普通 `snapshot --app <name>` 为 `AMBIGUOUS_TARGET` 并列出窗口 id，`snapshot --window-id <id>` 以其中一个为根。提示后方窗口中的 ref 为 `ACTION_FAILED`、原因 `blocked_by_modal_dialog`、`not_delivered`。共享标题的提示在显示上可能看起来相同；其中一个的 `--headed` ref 操作则为 `AMBIGUOUS_TARGET`、`window_position_ambiguous`、`not_delivered`：不用 `--headed` 回答它。
- `--force` 通过 pidfd 发送 `SIGTERM`，2 秒后发送 `SIGKILL`；已停止的进程立即获得 `SIGKILL`。
- 无 X11 窗口的 Wayland 会话进程为 `ACTION_NOT_SUPPORTED`、原因 `wayland_session`。
- 保护跟随目标进程实例，绝不跟随其名称。以下情况为 `INVALID_ARGS`、`not_delivered`：
  - 活动显示套接字的所有者，或 X 根的支持 WM 窗口的所有者；
  - 在无障碍总线上应答、不显示窗口（窗口管理器管理的窗口，或其无障碍树列出的窗口：原生 Wayland 窗口仅在此处）且由会话启动的应用（它拥有会话总线名称、作为用户服务单元运行，或是显示所有者的子进程：门户后端、shell 辅助程序、装饰宿主），除非本工具启动了它。

  不在无障碍总线上应答的进程（已停止或挂起）在拥有会话总线名称且窗口管理器未管理其任何窗口时受保护，除非本工具的启动启动了它。有窗口的已停止应用不受保护，无论谁启动：`--force` 结束它。
  缺少所有权证据（如不可读的 `/proc` 条目或无会话总线）在发送任何内容前为 `ACTION_NOT_SUPPORTED`；不可达的显示为 `PLATFORM_NOT_SUPPORTED`。X11 管理器所有权需要 XRes 进程 ID；缺少服务器支持则拒绝。

## 窗口操作

`move-window`、`resize-window`、`minimize`、`maximize` 和 `restore` 使用 EWMH。移动和调整大小在 250 毫秒内以 2 像素容差读回几何；最小化和最大化读回其状态。其成功报告命名 `details.verification_scope`。`restore` 在其各分支间没有统一的最终读回，报告 `delivered_unverified`：之后查看窗口。报告携带 `action`、`target`、`steps` 和 `disposition`，没有 `moved`、`resized` 或其他无条件成功标志。

- 钳制或被忽略的请求为 `ACTION_FAILED`、`delivered_unverified`，附带请求、观察和显示边界；窗口可能仍部分改变：读取边界。
- 调整大小仅按其尺寸判断。窗口管理器为保持窗口在显示上而移动窗口会改变原点，不改变判定，报告的 `details.observed` 保存读回值，含原点。
- 窗口管理器应答但未显示窗口的 `restore` 为 `ACTION_NOT_SUPPORTED`、原因 `restore_refused`、`delivered_unverified`，附带 `details.manager`。GNOME Activities Overview 打开时原因为 `overview_open`（`ACTION_FAILED`）：按 Escape 关闭它，然后在重试前查看窗口。
- 无 `--app` 或 `--window-id` 时，五者均作用于聚焦窗口，`data.target` 命名窗口 id、标题和进程身份，即使未提供选择器。除非你指聚焦窗口，否则传递一个。
- Wayland 原生窗口，以及 Wayland 会话上的取消最小化，在发送任何内容前以原因 `wayland_session` 拒绝。
- 无法映射到进程的窗口在任何写入前拒绝：无客户端为 `ACTION_NOT_SUPPORTED`、原因 `window_not_mapped`；两个客户端为 `AMBIGUOUS_TARGET`、原因 `ambiguous_window`。
- 窗口管理器可能忽略最小化后 1 到 2 秒发送的 restore，或 restore 后约 250 毫秒内发送的 minimize：重试前读取窗口状态。未确认的 restore 发送是不确定的，重试不安全；它可能已恢复窗口。
- `move-window` 需要 `--x` 和 `--y` 两者（单独一个在解析时为 `INVALID_ARGS`，退出码 2）。负坐标需要 `=` 形式：`move-window --x=-100 --y=20`。

## 聚焦

```bash
agent-desktop focus-window --app gnome-calculator
```

`focus-window` 仅当 `_NET_ACTIVE_WINDOW` 命名该窗口且其应用报告其为活动时才成功：其帧或应用的另一帧携带 `ACTIVE`、其某个菜单打开，或它是持有输入焦点但无 `ACTIVE` 的模态或对话框帧。请求发送后，读回失败或模态拒绝为 `delivered_unverified`，重试不安全。未确认的发送是不确定的投递，后续超时不会使先前的请求可安全重复。

- 最小化或隐藏的目标先被提升。已停止的目标在发送任何内容前为 `APP_UNRESPONSIVE`。
- 模态对话框持有焦点时，`focus-window --app <name>` 成功，对话框作为 `data.focused`。读取 `data.focused.title` 并对对话框操作，而非你命名的窗口。
- 管理器从未激活的窗口可以 `ACTION_FAILED` 结束，原因 `activation_not_granted`（在 mutter 下，直到窗口曾持有焦点一次）。其他超时以 `activation_refused`（另一窗口保持活动）或 `frame_not_active`（管理器命名了窗口，但其帧从未报告 `ACTIVE`）结束。
- GNOME Activities Overview 打开时原因为 `overview_open`；先关闭概览。新 GNOME shell 会打开它，且它阻止所有窗口的激活。发送全局 `--headed press escape`（无 `--app`），然后再次调用 `focus-window --app <name>`。headed ref 操作会为你运行 `focus-window` 并以相同方式结束。
- Wayland 会话为 `ACTION_NOT_SUPPORTED`、原因 `wayland_session`。
- 对最小化窗口的无头 ref `click` 以实时 `offscreen` 证据拒绝并保持其最小化。在 X11 上，`--headed click`、`--headed hover` 和 `--headed drag` 先聚焦 ref 的确切源窗口，若最小化则恢复它，然后投递手势；若窗口管理器无法显示并激活该窗口，或目标在聚焦后仍隐藏、离屏或被遮挡，则拒绝。不支持的物理路径或几何拒绝仅在证明未发送任何内容时才回退到语义点击。Wayland 上普通应用的 headed 指针操作在投递前以原因 `wayland_session` 拒绝。用 `restore` 恢复窗口而不点击它。

## 工作区

另一工作区上的窗口列出 `visible: false`，不能接受无头 `press`。`focus-window` 通过切换用户工作区将其带到前面：调用前决定这是否可接受。

## `press --app` 与后台目标

无头 `press --app <name>` 仅在目标进程拥有