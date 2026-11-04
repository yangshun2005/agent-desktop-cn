# 系统命令

应用生命周期、窗口、通知、剪贴板、等待、批处理、会话、健康检查。运行 `agent-desktop <command> --help` 查看参数。

## launch

```bash
agent-desktop launch "System Settings"       # macOS 显示名称
agent-desktop launch "TextEdit" --activate
agent-desktop launch "MyTool" --arg /tmp/notes.txt --env KEY=VALUE --no-attach
agent-desktop launch "Obsidian" --cdp
```

`launch` 在进程运行后返回。标识符因操作系统而异：示例使用 macOS 显示名称，平台技能会说明其操作系统的形式。默认附加到运行中的实例。`--no-attach` 改为启动新实例而非附加：在 Windows 上，若已有匹配应用在运行则返回 `ACTION_FAILED`；在 macOS 上，会在运行中的实例旁再启动一个实例。若 `--arg` 的值以 `-` 开头，请用 `--arg=<value>`。

进程运行与应用显示窗口是两回事：

```json
{ "app": "TextEdit", "pid": 611, "window": { "id": "w-110407", "title": "Open", "visible": true } }
```

- 应用已有窗口时 `window` 存在，没有时省略。缺失不算失败，`launch` 仍返回 `ok: true`。
- 基于文档的应用可能只在被置于前台时才打开首个窗口。用 `--activate`：它请求应用呈现窗口，并最多等待 `--timeout`（默认 30000）。这会把应用带到前台，因此不是无头操作。或者自行触发窗口，再用 `wait --event window-opened`。
- 在显示窗口前就退出的进程返回 `APP_UNRESPONSIVE`。
- 无窗口和仅菜单栏的应用不报告 `window`。用 `list-apps` 读取 `presentation`。
- 基于 Chromium 的应用可能返回 `renderer: "chromium"` 和 `suggestion`。这些是提示，无障碍路径仍然有效。缺少 `renderer` 不能证明应用是原生的。

### Chromium 应用的 Web 内容（`--cdp`）

对 Web 内容密集或遍历缓慢的 Electron 和 Chromium 应用（Slack、VS Code、Discord、Obsidian、Notion）使用 `--cdp`。它会以仅回环的调试端口全新启动应用，轮询直到端点响应，然后返回。可指定端口，或省略让操作系统选择空闲端口。

```json
{ "app": "Obsidian", "pid": 4821, "cdp": { "port": 9229, "http_endpoint": "http://127.0.0.1:9229",
  "websocket_url": "ws://127.0.0.1:9229/devtools/browser/<id>", "product": "Chrome/142.0.7444.265" } }
```

端口仅存在于新进程中。`launch` 不会替你退出运行中的应用，因为那会丢失用户状态。目标已在运行时返回 `ACTION_FAILED`，并带 `details.kind: "cdp_requires_fresh_launch"`。运行 `close-app`，确认退出，再用 `--cdp` 启动。

| `details.kind` | 含义 | 恢复方式 |
|----------------|---------|----------|
| `cdp_requires_fresh_launch` | 应用已在运行 | `close-app`，确认退出，重新启动 |
| `cdp_port_in_use`（`INVALID_ARGS`） | 指定端口已被占用 | 换一个端口，或省略 |
| `cdp_switch_conflict`（`INVALID_ARGS`） | `--arg` 携带了 `--remote-debugging-*` 或 `--remote-allow-origins` 开关 | 去掉该 `--arg` |
| `cdp_endpoint_unavailable` | 截止前无端点响应 | 应用仍在运行。改用无障碍路径 |

交接：agent-desktop 从不直接与端口通信。检查 `command -v agent-browser`。若存在，运行 `agent-browser connect <port>`，使用其快照、点击和输入工作流（`agent-browser skills get electron` 有指南）。Playwright、Puppeteer 及其他 CDP 客户端也可用。不要手写原始 CDP 或调用应用内部 API。若无客户端，请用户运行 `npm install -g agent-browser`，或留在无障碍路径。

即使连接了 CDP，以下仍由 agent-desktop 处理：原生菜单栏、文件对话框和表单、窗口管理、通知、截图，以及任何非你启动的应用。端口开放期间，同一用户的任何本地进程都能控制应用的 Web 内容。仅在需要该步骤时请求 `--cdp`。`close-app` 结束暴露。

## close-app

`close-app` 关闭应用及其所有窗口。没有 `close-window` 命令。要关闭多个窗口中的一个，快照目标 `--window-id` 并点击其自身的关闭按钮 ref，然后用 `batch` 组合点击和 `wait --event window-closed` 确认关闭。通过其表单回答任何保存提示。对于最后一个窗口，若打算退出应用，用 `close-app`。Ctrl+W 可能只关闭标签页或文档，窗口仍开着。Linux 窗口参考包含一个 Bash 配方，用于选择新的关闭 ref。

```bash
agent-desktop close-app "TextEdit"
agent-desktop close-app "TextEdit" --force
```

仅在确认进程消失后才报告成功：`{ "method": "graceful" | "force", "disposition": { "delivery": "delivered_unverified" | "delivered_verified" | "not_delivered" }, "target": { "kind": "app", ... } }`。`not_delivered` 表示应用已退出。读取 `disposition` 而非 `ok` 来了解已证实的内容。若出现保存对话框，快照并回答它。`--force` 结束进程。受保护的系统进程返回 `INVALID_ARGS` 和 `not_delivered`，而非 `PERM_DENIED`。

## list-apps

以 `{ name, pid, bundle_id, presentation }` 列出运行中的 GUI 应用。`--app` 按名称子串过滤。`presentation` 为 `foreground`（普通窗口）、`background`（无 Dock 条目，如菜单栏项和热键浮层），或省略（辅助进程）。无 UI 的应用被排除。

## 窗口（操作系统窗口）

```bash
agent-desktop list-windows --app "Finder"
agent-desktop focus-window --window-id w-4521
agent-desktop resize-window --app "TextEdit" --width 800 --height 600
agent-desktop move-window --app "TextEdit" --x 0 --y 0
agent-desktop minimize --app "TextEdit"   # maximize、restore 同理
```

- `list-windows` 返回 `{ id, title, app_name, pid, bounds, is_focused, accessible }`。应用有多个窗口时，用 `w-<id>` 配合 `--window-id`。
- `accessible: false` 表示语义命令无法到达该窗口。定位它会返回 `ACTION_NOT_SUPPORTED`，带 `kind: "window_without_accessibility_element"`。它仍接受截图和坐标输入。请选择其他窗口。
- `focus-window` 至少需要一个标识符。它确认操作系统报告该窗口已聚焦，否则返回 `ACTION_FAILED`。
- `restore` 撤销最小化，将窗口恢复到先前位置。它不保证取消最大化。

## 通知

命令：`list-notifications`、`dismiss-notification`、`dismiss-all-notifications`、`notification-action`、`wait --notification`。它们驱动操作系统通知界面：macOS 的通知中心、Windows 的操作中心、Linux 的 GNOME Shell 消息列表。各平台的命令形式和 JSON 字段相同。平台技能说明其限制。

- 输出为原文。标题和正文不脱敏。将你转发的任何内容视为敏感。
- 在 macOS 和 Windows 上，无头列出仅在界面已打开时有效。界面关闭时，严格无头调用返回 `POLICY_DENIED`。用 `--headed`，它可能打开界面并恢复先前的前台应用。Linux 即使列表关闭也能无头列出。
- 每次变更都需要 `--headed`。单条通知变更还需要来自同一列表的指纹：`--expected-app` 或 `--expected-title`。通知在列表间会重排。当索引处的行不再匹配时，调用返回 `NOTIFICATION_NOT_FOUND` 且不点击任何东西。无指纹则 `INVALID_ARGS`。
- `list-notifications` 返回 `{ index, app_name, title, body, actions }`，索引从 1 开始。`--app`、`--text`、`--limit` 过滤。
- `notification-action INDEX "Reply"` 点击命名按钮。条目缺少该操作时返回 `ACTION_NOT_SUPPORTED` 且不做更改。
- `dismiss-all-notifications` 返回 `{ dismissed_count, failures, failed_count }`。之后关闭界面失败不报告为错误。
- `wait --notification` 以等待开始时取的基线检测新条目。可重试错误在 `--timeout` 内重试。在 macOS 和 Windows 上，无头等待只能看到已打开的界面。

## 剪贴板

```bash
agent-desktop clipboard-get --format auto
agent-desktop clipboard-get --format image --out /tmp/clip.png
agent-desktop clipboard-set "Hello"
agent-desktop clipboard-set --image /tmp/a.png
agent-desktop clipboard-set --file-url /tmp/a.txt --file-url /tmp/b.txt
agent-desktop clipboard-clear
```

- 格式：`text`（默认）、`auto`（文件引用，然后图像，然后文本）、`image`、`file-urls`。输出为 `{ "type": "text", "text": ... }`、`{ "type": "file_urls", ... }` 或 `{ "type": "image", "path", "width", "height" }`。剪贴板无该形式内容时：`{ "type": "<format>", "found": false }`。
- 图像字节写入 `--out`，或活动会话目录下的私有文件（或 `~/.agent-desktop/tmp`）。
- `clipboard-set` 每次调用写入一种表示。`--file-url` 和 `--image` 优先于位置文本。每个 `--file-url` 路径必须存在，否则 `INVALID_ARGS`。
- 中途失去剪贴板所有权的写入报告 `delivered_unverified`，而非虚假成功。

## wait

```bash
agent-desktop wait 1000
agent-desktop wait --element @s8f3k2p9:e5 --predicate actionable --action type --timeout 5000
agent-desktop wait --element @s8f3k2p9:e5 --predicate value --value "Done"
agent-desktop wait --window "Save As" --app "TextEdit"
agent-desktop wait --text "Loading complete" --app "Safari"
agent-desktop wait --menu --app "Finder"
agent-desktop wait --event window-opened --app "Finder"
```

用 `wait`，不要固定睡眠。`--timeout` 默认 30000 毫秒。模式：

- **时间。** `wait <ms>` 暂停。
- **元素。** `--predicate` 为 `exists`（默认）、`enabled`、`visible`、`actionable` 或 `value`（配合 `--value`）。`actionable` 检查对 `--action`（默认 `click`，还有 `type`、`set-value`、`clear`）的就绪状态。在等待后输入流程前用 `--action type`。可编辑性检查仅对编辑操作运行。`--element` 接受限定 ref（`@<snapshot_id>:eN`）；等待轮询 ref 中嵌入的快照。
- **窗口。** 等待标题包含该文本的窗口。`--app` 缩小范围。超时时，`details.last_observed` 列出范围内的窗口数和最多八个标题。
- **文本。** 选择一个应用窗口，然后等待该树中的文本。`--count N` 等待恰好 N 个节点匹配，结果带 `count`。多个符合条件的窗口（如文档及其保存对话框）在任何文本检查前返回 `AMBIGUOUS_TARGET`。`--window-id` 不缩小文本等待。恢复方法：用 `list-windows --app <app>` 找到目标，用 `snapshot --app <app> --window-id <id> -i` 检查。对于对话框，用 `wait --window <title> --app <app>`，然后 `snapshot --surface sheet`（见对话框工作流）。`--window` 和 `--text` 是不同模式，不能组合。
- **菜单。** `--menu` 等待菜单界面打开。`--menu-closed` 等待其关闭。
- **事件。** `--event` 为 `window-opened`、`window-closed`、`app-launched`、`app-terminated`、`focus-changed`、`surface-appeared`、`surface-dismissed` 之一。它将等待开始时取的基线与新读取做差异比较，因此无需预先提供窗口 id。`--window-id` 和 `--window` 缩小范围。

仅对 `wait --event`，`--app` 在等待开始时解析一次为一个进程实例。（`wait --window` 每次轮询按应用名过滤。）之后同名的进程对该等待不可见。两个运行实例返回 `AMBIGUOUS_TARGET`。无应用匹配时：

- `app-launched`、`window-opened`、`surface-appeared`：持续轮询直到超时。在与启动竞争时使用。
- `app-terminated`、`window-closed`、`surface-dismissed`：立即返回 `"found": true` 和 `"target_unresolved": true`。拼错 `--app` 也给出相同答案，因此请检查名称。
- `focus-changed`：返回 `APP_NOT_FOUND`。

在 macOS 和 Windows 上，应用无响应时现有窗口仍可满足 `--window`。等待可能在超时前返回顶层 `APP_UNRESPONSIVE`；直接处理该错误。`wait --event` 在可重试读取失败后继续轮询，若超时则在 `details.last_error` 中包含最新失败。不可重试错误立即结束事件等待。

平台技能说明因操作系统而异的事件和菜单等待规则。

`wait` 的 `TIMEOUT` 带 `details.kind: "wait_timeout"`、`last_observed` 或 `last_error`，以及事件的 `baseline_counts`。

## batch

```bash
agent-desktop batch '[
  {"command":"click","args":{"ref_id":"@s8f3k2p9:e1"}},
  {"command":"wait","args":{"ms":200}},
  {"command":"type","args":{"ref_id":"@s8f3k2p9:e2","text":"hello"}}
]' --stop-on-error

agent-desktop batch '[
  {"command":"click","args":{"ref_id":"@s8f3k2p9:e1"}},
  {"command":"wait","args":{"event":"window-opened","app":"TextEdit","timeout":5000}}
]' --stop-on-error
```

批处理在一个进程中按顺序运行条目。它不是事务。条目使用与 CLI 相同的类型化命令、策略检查和分发。用 `args`，不是 `params`。未知字段被拒绝，嵌套 `batch` 被拒绝。传 `--stop-on-error` 在首次失败时停止。每个条目可带 `"session": "id"`，否则继承顶层会话。分发前会话已结束的条目为 `not_started`，原因 `session_ended`。已结束的顶层会话在任何条目运行前以 `INVALID_ARGS`（`details.kind: session_ended`）拒绝整个批处理。快照参数用 `skeleton`、`root`、`interactive_only`。

等待条目使用与等待参数字段匹配的扁平 JSON 键。对于事件等待，用 `event` 和可选的 `app`、`timeout`（毫秒；默认 30000）、`window`、`window_id`。例如，上面的批处理最多等待五秒以获取新的 TextEdit 窗口。`window` 和 `window_id` 仅缩小窗口生命周期事件（`window-opened`、`window-closed`、`focus-changed`）；它们不限定文本等待。在动作后事件序列中，批处理在动作前捕获事件基线，若基线读取失败则不启动动作。

键是命令自身的字段名，因此位置参数以其自身而非占位符命名：`press COMBO` 是 `{"combo":"cmd+s"}`，`click REF` 是 `{"ref_id":"@s8f3k2p9:e1"}`，`type REF TEXT` 接受 `ref_id` 和 `text`。标志去掉前导 `--` 并将 `-` 写为 `_`，因此 `--timeout-ms` 是 `timeout_ms`。

## 会话与跟踪

会话是 `<state root>/sessions/<id>/` 下的磁盘容器，含清单、快照 refmap，以及跟踪开启时的 `trace/` 目录。状态根为 `~/.agent-desktop`。`AGENT_DESKTOP_HOME` 可重定位它。它必须是绝对路径，否则命令以 `INVALID_ARGS` 失败。`status` 将其报告为 `state_root`。

```bash
export AGENT_DESKTOP_SESSION="$(agent-desktop session start --name "invoice-bot" | jq -r '.data.session_id')"
agent-desktop session end "$AGENT_DESKTOP_SESSION"
```

- `session start` 创建会话并打印 `{ session_id, name, trace, created_at }`。`--name` 必须为 1 到 128 字节且无控制字符，否则 `session start` 返回 `INVALID_ARGS`。`session start` 不激活后续进程。用全局 `--session <id>` 或 `AGENT_DESKTOP_SESSION` 传递 id。`--