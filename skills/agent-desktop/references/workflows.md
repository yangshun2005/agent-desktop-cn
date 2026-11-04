# 常见工作流

多步骤桌面自动化的模式。每个 ref 都来自最新的快照输出。

## 首次设置

在任何自动化之前，先检查平台和权限。

```bash
agent-desktop status
agent-desktop permissions
```

- 在 macOS 上，终端需要辅助功能权限，截图需要屏幕录制权限。如果 `permissions` 报告 `denied`，运行 `agent-desktop permissions --guide`。它会在系统设置中打开缺失权限的页面，并显示一个面板，告诉用户应将哪个应用拖入列表。
- 在 Windows 上，同完整性级别的目标没有权限对话框。向提权目标输入需要同样提权的终端。
- 在 Linux 上，没有权限对话框。会话需要运行中的 AT-SPI 总线，且 `status` 必须报告 `accessibility` 为 `granted`。在与目标应用相同的桌面会话中运行工具。如果 `status` 报告为拒绝，阅读 Linux 技能中的总线部分。

然后运行 `agent-desktop skills get platform` 获取你的操作系统的详细信息。

## 密集应用：骨架、下钻、操作、验证

适用于 Electron 应用以及任何交互元素超过约 50 个的应用。

```bash
agent-desktop snapshot --skeleton --app "Slack" -i --compact
# group "Channels" @s8f3k2p9:e2 (children_count: 42)
agent-desktop snapshot --root @s8f3k2p9:e2 -i --compact
agent-desktop click @s8f3k2p9:e18
agent-desktop snapshot --root @s8f3k2p9:e2 -i --compact
```

- 重新下钻某个根会替换该根的 ref 以及在其下进行的下钻的 ref。其他位置的骨架 ref 以及其他区域的下钻保持有效。
- 骨架深度内的交互元素已有普通 ref。
- 如果你知道角色或确切名称，跳过地图，直接运行带 `--role --name --exact` 的 `find`。

## 填写表单

```bash
agent-desktop snapshot --app "System Settings" -i --compact
agent-desktop set-value @s8f3k2p9:e3 "My MacBook Pro"
agent-desktop click @s8f3k2p9:e8
agent-desktop snapshot --app "System Settings" -i --compact
```

- 优先使用 `set-value`。当字段需要真实按键事件时，使用 `clear` 然后 `type`。
- 读取结果：`delivered_verified` 仅确认字段值，不确认应用已接受。在 UI 中验证结果。
- 要在字段间移动而不为每个字段获取 ref，使用 `press tab` 和 `type`。

## 导航菜单

```bash
agent-desktop find --app "TextEdit" --surface menubar --name "Save As…" --exact --first
agent-desktop click @s8f3k2p9:e5
agent-desktop wait --window "Save" --app "TextEdit"
agent-desktop snapshot --app "TextEdit" --surface sheet -i
```

- `find --surface menubar` 返回一个 ref，而不是整个菜单栏。
- 手动遍历菜单：点击菜单栏项，`wait --menu`，`snapshot --surface menu -i`，点击该项，`wait --menu-closed`。
- 无头 `press` 永远不会运行菜单项。找到该项并 `click` 它。

## 对话框、工作表和警报

```bash
agent-desktop wait --window "Save As" --app "TextEdit" --timeout 5000
agent-desktop snapshot --app "TextEdit" --surface sheet -i
agent-desktop set-value @s8f3k2p9:e2 "my-document.txt"
agent-desktop click @s8f3k2p9:e5
agent-desktop snapshot --app "TextEdit" -i
```

覆盖层打开时，快照表面（`sheet`、`alert`、`popover`），而不是窗口。永远不要在表面上使用 `--skeleton`。对话框关闭后，再次快照窗口以获取新的 ref。当某个操作使 `data.surfaces` 非空时，接下来处理该覆盖层。

## 右键上下文菜单

```bash
agent-desktop right-click @s8f3k2p9:e3
agent-desktop snapshot --app "Finder" --surface menu -i
agent-desktop click @s8f3k2p9:e7
agent-desktop wait --menu-closed --app "Finder" --timeout 2000
```

如果 `right-click` 返回 `APP_UNRESPONSIVE`，菜单可能已打开。先查看屏幕状态。不要盲目重试。某些操作系统需要 `--headed` 才能右键（平台技能）。

## 滚动查找项目

```bash
agent-desktop snapshot --skeleton --app "App" -i --compact
agent-desktop snapshot --root @s8f3k2p9:e2 -i --compact
agent-desktop scroll @s8f3k2p9:e8 --direction down --amount 5
agent-desktop find --app "App" --name "Target Item" --exact
```

重复 `scroll` 和 `find` 直到 `find` 返回 ref。然后 `click` 它。如果列表是虚拟化的且行始终不出现，滚动容器而不是 `scroll-to`。

## 等待异步 UI

```bash
agent-desktop click @s8f3k2p9:e5
agent-desktop wait --text "Download complete" --app "App" --timeout 30000
agent-desktop wait --element @s8f3k2p9:e10 --predicate actionable --timeout 10000
```

使用与信号匹配的等待原语：文本、元素、窗口、菜单或事件。在 `launch`、对话框触发或菜单点击之后，先等待再快照。

## 操作前检查

```bash
agent-desktop is @s8f3k2p9:e6 --property checked
agent-desktop check @s8f3k2p9:e6
```

`check` 和 `uncheck` 是幂等的。使用它们而不是 `toggle`。如果切换结果为 `delivered_unverified`，在再次操作前用 `is` 读取状态。

## 从元素复制文本

直接读取：`agent-desktop get @s8f3k2p9:e5 --property value`。如果元素不暴露其文本，聚焦它，全选，通过 `press` 使用操作系统快捷键复制，然后运行 `clipboard-get`。

## 拖放

```bash
agent-desktop --headed drag --from @s8f3k2p9:e3 --to @s8f3k2p9:e8
agent-desktop --headed drag --from-xy 100,200 --to-xy 500,400 --drop-delay 800
```

拖放是物理操作，需要 `--headed`。通过读取目标状态来确认放置，而不是命令的 `ok`。

## 多窗口

```bash
agent-desktop list-windows --app "Finder"
agent-desktop focus-window --window-id w-5678
agent-desktop snapshot --app "Finder" --window-id w-5678 -i
```

## 启动、自动化、关闭

```bash
agent-desktop launch "Calculator"       # macOS name; on Windows use "calc.exe" or an absolute path
agent-desktop snapshot --app "Calculator" -i
agent-desktop close-app "Calculator"
```

读取启动结果中的 `data.window`。如果它不存在且你需要窗口，使用 `launch --activate` 或 `wait --event window-opened`。启动标识符规则因操作系统而异（平台技能）。

## Chromium 应用：两个工具

Slack、VS Code、Discord 及类似应用基于 Chromium。使用 agent-desktop 处理原生表面，使用 CDP 客户端处理密集的 Web 内容。

```bash
agent-desktop launch "Slack"       # macOS name; on Windows use the absolute path to slack.exe
agent-desktop close-app "Slack"
agent-desktop wait --event app-terminated --app "Slack" --timeout 10000
agent-desktop launch "Slack" --cdp 9222
agent-browser connect 9222
```

`--cdp` 需要全新启动，所以先关闭正在运行的应用。用 CDP 客户端驱动 Web 内容。菜单、文件对话框、窗口、通知和截图保留在 agent-desktop 上。如果没有 CDP 客户端，跳过 `--cdp`，在辅助功能路径上使用骨架工作流。它始终有效。

## 通知

```bash
agent-desktop --headed list-notifications --app "Slack"
agent-desktop --headed notification-action 1 "Reply" --expected-app Slack --expected-title "#general"
agent-desktop --headed dismiss-notification 1 --expected-app Slack
```

macOS（通知中心）、Windows（操作中心）和 Linux（GNOME Shell）支持这些命令。先列出并从列表中复制指纹。通知会重新排序，所以过期的索引会以 `NOTIFICATION_NOT_FOUND` 失败，而不是按错行。

## 批处理

```bash
agent-desktop batch '[
  {"command":"click","args":{"ref_id":"@s8f3k2p9:e1"}},
  {"command":"wait","args":{"ms":200}},
  {"command":"press","args":{"combo":"return"}}
]' --stop-on-error
```

批处理节省进程启动。它不是事务。除非传入 `--stop-on-error`，否则失败后的条目仍会运行。

## 追踪多步骤运行

此示例使用 `jq` 读取 `session start` 返回的会话 id。

```bash
session_start=$(agent-desktop session start --name "invoice-bot")
AGENT_DESKTOP_SESSION=$(printf '%s\n' "$session_start" | jq -er '.data.session_id')
export AGENT_DESKTOP_SESSION
agent-desktop status
agent-desktop trace show --limit 50
agent-desktop session end "$AGENT_DESKTOP_SESSION"
```

`status` 确认 `session_id` 和 `tracing`。每次运行使用一个会话。并发代理设置相同的变量，并使用各自快照中的限定 ref。

## 反模式

核心技能已说明 ref、等待、有头模式和重试的规则。以下是其他常见错误：

1. **对密集应用做完整快照。** 使用 `--skeleton` 和 `--root`，只重新下钻变化的区域。完整树消耗的 token 多得多。
2. **有 ref 时使用坐标。** 语义操作比坐标点击更可靠。
3. **跳过 `status` 和 `permissions`。** 缺失权限看起来像空树。
4. **覆盖层打开时快照窗口。** 改为快照表面。
5. **手写 CDP。** 使用真正的 CDP 客户端。原始 WebSocket 脚本和应用内部 API 会静默失败且不验证任何内容。
6. **单个编辑器上的并行写入。** 点击编辑、输入和提交不是原子的。序列化该序列，在不确定的投递后先读取状态再操作。