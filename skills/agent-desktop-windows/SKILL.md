---
name: agent-desktop-windows
tags: windows-automation, accessibility, uia, ai-agent, gui-automation, cli
requirements:
  - agent-desktop
description: >
  agent-desktop CLI 的 Windows 平台指南。按命令组列出哪些在 Windows 上可用、
  哪些会被拒绝，首次接触的坑（PowerShell 中 ref 的引号处理、最小化窗口、
  仅限有头模式的输入），以及在哪里查找 shell 界面、通知、光标叠加层、UIPI 提权、
  Chromium/Electron 的稳定行为与故障排查。
  触发词："windows desktop automation"、"UIA tree"、"automate Windows app"、
  "agent-desktop on Windows"、"elevated window input blocked"、"E_ACCESSDENIED"，
  或任何 Windows GUI 交互任务。
---

# agent-desktop-windows

本文件只讲 Windows 上的差异。观察-操作循环、ref、JSON 信封、会话和追踪在各平台一致。

要求 Windows 10 1809+ / Windows Server 2019+（x64 或 ARM64）。在已登录用户的交互式桌面会话中运行。不支持 Session 0（服务和 SYSTEM 任务）、Server Core、安全桌面（UAC 提示、登录界面）、锁定的桌面以及其他用户的会话：在这些环境中观察、操作和截图会失败或返回空。

## 能力表

| 组 | 命令 | Windows 上的状态 |
|-------|----------|-------------------|
| 观察 | `snapshot`、`find`、`get`、`is`、`screenshot` | 可用。界面为 `window`、`focused`、作为 `sheet` 的 Chromium 模态框，以及作为 `menu` 的已打开应用菜单 |
| 界面 | `list-surfaces` | 可用。按进程列出 `window`、`focused` 和 `sheet`，菜单打开时额外列出带 `item_count` 的 `menu` |
| Shell 界面 | `open-system-surface` | 支持 `start-menu`、`taskbar`、`system-tray`、`system-tray-overflow` 和 `action-center`，并返回 `snapshot --surface <kind>` 读取的窗口。`quick-settings` 返回 `INVALID_ARGS` 并提示使用 `action-center` |
| Ref 交互 | `click`、`right-click`、`type`、`set-value`、`clear`、`select`、`toggle`、`check`、`uncheck`、`expand`、`collapse`、`scroll`、`scroll-to` | 可用，无头模式和 `--headed` 模式下均为语义操作。`right-click` 和 `type` 需要 `--headed`，无头模式返回 `POLICY_DENIED` |
| 多击与聚焦 | `double-click`、`triple-click`、`focus` | 仅在全局 `--headed` 下可用 |
| 键盘与鼠标 | `press`、`hover`、`drag`、`mouse-click`、`mouse-move`、`mouse-wheel` | 可用。移动光标的输入需要 `--headed`。`press --app` 见首次接触 |
| 按住输入 | `key-down`、`key-up`、`mouse-down`、`mouse-up` | 在所有平台均以 `ACTION_NOT_SUPPORTED` 失败关闭。无头模式下 `mouse-down` 或 `mouse-up` 先返回 `POLICY_DENIED` |
| 应用与窗口 | `launch`、`close-app`、`list-windows`、`list-apps`、`focus-window`、`resize-window`、`move-window`、`minimize`、`maximize`、`restore` | 可用。`launch` 接受的名称见首次接触 |
| 显示器 | `list-displays` | 可用 |
| 剪贴板 | `clipboard-get`、`clipboard-set`、`clipboard-clear` | 可用；支持文本和图像内容 |
| 等待 | `wait` | 支持 ms、`--element`、`--window`、`--text`、`--menu`、`--menu-closed`、`--event` 和 `--notification` |
| 通知 | `list-notifications`、`dismiss-notification`、`dismiss-all-notifications`、`notification-action` | 通过操作中心可用。操作中心关闭时传 `--headed` |
| 光标叠加层 | `cursor-overlay` | 可用。`enable` 仅在渲染器通过管道确认后返回 `data.rendered: true`。见 `references/shell-and-overlay.md` |
| 系统 | `status`、`permissions`、`version`、`batch`、`skills`、`session`、`trace` | 可用 |

## 首次接触

- **在 PowerShell 中给每个 ref 加引号。** PowerShell 会把裸 `@token` 当作 splatting 运算符，在二进制看到参数之前就将其删除，因此 `set-value @s8f3k2p9:e1 hi` 会报 `INVALID_ARGS`。应写成 `set-value '@s8f3k2p9:e1' hi`。cmd.exe 和 bash 无需加引号。
- **被其他窗口遮挡的窗口可操作。最小化的窗口不行。** 最小化窗口的每个元素都报告 `offscreen`，ref 操作会失败。运行 `restore --app <image>` 后重新 snapshot。不要反射性地使用 `focus-window`：它会抢占用户的前台。
- **不存在权限对话框。** `permissions` 将 `automation` 报告为 `not_required`。提权边界是唯一的访问控制。
- **Chromium 和 Electron 应用在稳定前读取结果偏少。** 给 `snapshot` 传更长的 `--timeout-ms`，不要断定树为空。
- **被阻止的快捷键。** `alt+f4`、`win+l`、`win+d`、`alt+tab`、`win+r`、`win+x`、`win+e`、`ctrl+shift+esc` 及其修饰键超集在无 `--force` 时被拒绝。组合键中 `cmd` 和 `meta` 是 Windows 键，所以 `meta+c` 是 shell 快捷键。复制请写 `ctrl+c`。
- **`type` 需要 `--headed`。** 它通过 `SendInput` 合成击键，因此严格无头模式下 `type` 返回 `POLICY_DENIED`。改用 `set-value`，或传 `--headed`。当字段没有可读的 Text 选区时，它以 `delivered_unverified`（`verification_scope: "unavailable"`）结束；有可读选区时可验证：用新的 `get` 确认。
- **`press --app` 写入前台队列**，没有按进程定向。无头模式下向非最前台应用发送 press 会失败关闭。非交互调用方（服务、计划任务、CI 任务）会得到 `delivered_unverified`。
- **`launch` 接受绝对路径或裸名称**，裸名称需位于 System32 或 Windows 目录下，带不带 `.exe` 均可（`notepad` 或 `notepad.exe`），不接受显示名称。用 `list-apps` 查找运行中的应用。
- **通知：** 操作中心关闭时 `list-notifications` 需要 `--headed`。单项 `dismiss-notification` 和 `notification-action` 需要 `--headed`，且需用 `--expected-app` 或 `--expected-title` 指定 INDEX 处的条目。
- **界面：** 打开的菜单或上下文菜单是 `menu` 界面。Windows 不提供 `menubar` 界面。
- **会唤起 shell 界面元素的命令会占用前台。** 严格无头模式在唤起任何内容之前以 `POLICY_DENIED` 拒绝。
- **有状态操作执行一次，然后通过重新读取验证。** `delivered_verified` 表示回读匹配。`ACTION_FAILED` 且 `details.kind: "post_action_verification"` 表示操作已生效但回读不同：读取 `details.post_state`，不要重试。应用重新格式化过的 `set-value` 会读作失败。见 `references/troubleshooting.md`。
- **未签名二进制警告是预期行为。** 见 `references/troubleshooting.md`。

## Windows 上的差异

- 无头模式和 `--headed` 模式下，ref 操作都是语义 UI Automation 调用，但 `right-click`、`double-click`、`triple-click`、`type` 和 `focus` 例外，它们是物理操作或需要获取焦点，需要 `--headed`。
- 只有当 ref 是其快照捕获时窗口中唯一具有该角色、原生 id、名称和描述的元素，且仍恰好有一个活动元素匹配时，ref 才能解析。唯一性按整个窗口计算，而非 `--skeleton` 或 `--max-depth` 快照呈现的浅层树，因此截断范围下的孪生元素仍会拒绝该 ref。完全相同的孪生元素（例如多个相同的 `Delete` 按钮）在触及应用之前以 `AMBIGUOUS_TARGET`、`details.kind: identity_not_unique` 拒绝；存在活动的第二个匹配为 `saved_ref_ambiguous`，元素被移除为 `STALE_REF`、`saved_ref_absent`。列表变化后重新 snapshot；必须在两个相同控件中选一个时，使用 `--headed mouse-click --xy`。
- 光标叠加层仅对无头语义操作播放逐操作移动动画。`--headed` 操作移动真实光标。

## 参考

- `references/shell-and-overlay.md`：涉及 `open-system-surface`、托盘和任务栏界面、通知、光标叠加层、托管（UWP）窗口标识和菜单检测覆盖范围时打开。
- `references/permissions-and-elevation.md`：完整性级别、UIPI、被阻止的组合键、受保护进程、跨进程交互租约。
- `references/chromium-and-electron.md`：稳定时序、被遮挡窗口的风险、WPF 错误提供程序陷阱、未命名 Web 内容。
- `references/troubleshooting.md`：症状到原因映射、操作验证结果、无头保存文档、安装警告。

从二进制读取任意文件：
`agent-desktop skills get agent-desktop-windows references/troubleshooting.md`。