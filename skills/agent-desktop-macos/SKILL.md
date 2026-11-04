---
name: agent-desktop-macos
tags: macos-automation, accessibility, ax, tcc, ai-agent, gui-automation, cli
requirements:
  - agent-desktop
description: >
  agent-desktop CLI 的 macOS 平台指南。用于 macOS 上的桌面自动化：
  授予辅助功能与屏幕录制权限（TCC）、macOS 上有头模式的行为（物理优先的点击、输入和滚动）、
  各类界面（菜单、菜单栏、表单、弹出框、警告框、程序坞、菜单栏附加项）、
  无窗口的菜单栏应用与系统面板、通知中心、
  Chromium 与 Electron 应用（包括 launch --cdp），以及修复 PERM_DENIED、
  辅助功能树为空、ACTION_FAILED 和 APP_UNRESPONSIVE。
---

# macOS 上的 agent-desktop

先阅读核心技能（`agent-desktop skills get desktop`）了解观察、操作、验证循环。本技能只介绍 macOS 上不同之处。命令失败时，在 [references/troubleshooting.md](references/troubleshooting.md) 中查找对应症状。

## 要求

- macOS 13 或更高版本，Intel 或 Apple Silicon。
- 为启动 agent-desktop 的应用（Terminal、iTerm2、Warp、VS Code、Codex）授予辅助功能权限。macOS 27 及更高版本将此列表命名为"设备控制与数据访问"。如果系统设置中单独列出了 `agent-desktop` 二进制文件，也要为它授权。
- 屏幕录制权限，仅用于 `screenshot` 和 `session start --screenshots` 产物。
- 系统事件（System Events）的自动化权限，仅在命令报告缺失时需要。

## 首次使用

1. 运行 `agent-desktop permissions`。
2. 如果某项权限被拒绝，运行 `agent-desktop permissions --guide`。它会打开正确的系统设置页面，并显示一个面板，告诉用户应将哪个应用拖入列表。用户通过 Touch ID 或密码确认后，面板会自动跳到下一个缺失的权限。该命令立即返回，值为 `"guide": "started"`、`"already_running"` 或 `"not_needed"`。`agent-desktop permissions --request` 则显示系统提示。
3. 如果引导无法运行，打开系统设置 > 隐私与安全性 > 辅助功能（macOS 27 及更高版本为"设备控制与数据访问"）。添加启动应用并开启。
4. 运行 `agent-desktop permissions` 确认状态为 `granted`。下一次 agent-desktop 调用即可看到新授权，启动应用无需重启。每条命令在各自进程中读取权限，因此在 `batch` 期间授予的权限从下一次 CLI 调用起生效，不影响该批次中后续的条目。
5. 运行 `agent-desktop snapshot -i` 确认。它读取当前聚焦的应用。当 Finder 没有打开的窗口时，`snapshot --app Finder -i` 返回 `WINDOW_NOT_FOUND`。

`permissions` 不会弹出提示。`permissions --request` 可以弹出提示，包括自动化权限。`permissions --guide` 不弹出提示，仅覆盖辅助功能和屏幕录制。当 macOS 需要弹出提示时，自动化权限报告为 `unknown`。

## macOS 上的不同之处

**有头模式对 `click`、`right-click`、`type`、`clear` 和 `scroll` 采用物理优先。** 核心会聚焦 ref 对应的确切窗口；来自后台应用（菜单栏应用、程序坞、系统面板）或状态项的 ref 没有窗口，不会被聚焦。适配器随后向已验证的坐标点发送真实的点击、按键或滚轮事件。在 `--headed` 下，`expand`、`collapse`、`set-value`、`select`、`toggle`、`check`、`uncheck`、`focus` 和 `scroll-to` 仍保持语义化。双击、三击、`hover` 和 `drag` 需要 `--headed`。原始 `--xy` 输入从不改变焦点。

**通过观察判断是否生效。** AX 返回码不能证明产生了效果。Finder 对不执行任何操作的 `AXConfirm` 也返回成功。再次快照，或用 `get` 和 `is` 检查结果。

**界面。** 使用 `--surface menubar` 访问应用的"文件"、"编辑"和"显示"菜单。使用 `--surface menu-bar-extras` 访问状态项：不带 `--app` 时读取系统项（Wi-Fi、电池、时钟），带 `--app` 时读取该应用自己的项。两者都适用于无窗口的菜单栏应用。使用 `--surface dock` 访问程序坞。两者始终在屏幕上，因此 `open-system-surface` 会立即返回它们，无头模式即可，无需 `--headed`；`action-center` 和 `start-menu` 是 Windows 类型。没有普通窗口的后台应用（菜单栏应用、程序坞、控制中心和通知中心面板）通过其进程观察：点击其状态项后运行 `snapshot --app "Control Center"`。这适用于 `window`、`menubar` 和 `menu-bar-extras`，不适用于 `focused`，也不适用于尚未显示窗口的常规应用（后者返回 `WINDOW_NOT_FOUND`：等待窗口出现）。控制中心在点击磁贴后会立即重建其树，因此要用 `snapshot --app "Control Center" --wait-for "<role>:<name>"` 深入查看，而不是立即快照。其 ref 在无头模式和 `--headed` 下都可用。当菜单、表单、警告框或弹出框打开时，其后方窗口树中的 ref 是错误的目标。

**菜单。** 无头模式下的 `right-click` 使用 `AXShowMenu`。如果菜单已打开但应用返回 `APP_UNRESPONSIVE` 且 `retry: unsafe`，不要重试。用 `--surface menu` 检查菜单。对于组合框和菜单按钮，使用 `select @ref "Option"`。`select` 需要菜单从关闭到打开的转换，因此先关闭任何已打开的菜单。

**通知中心。** 通知命令通过辅助功能驱动通知中心，规则见核心 `commands-system` 参考。当 agent-desktop 打开通知中心后，它会在之后关闭并恢复之前的应用焦点。在单次关闭或操作时同时传入 `--expected-app` 和 `--expected-title`，因为通知堆栈会移动。对于在你列出之前就消失的横幅，使用 `wait --notification --app "App"`。

**Chromium 与 Electron。** agent-desktop 在需要时会自动启用 Chromium 渲染器的 Web 辅助功能。对于内容密集的运行中应用，使用 `snapshot --skeleton -i --compact`，然后用 `--root @ref` 深入查看。

**标识。** `launch` 接受显示名称（`"System Settings"`）或 bundle ID（`com.apple.systempreferences`）。其他命令通过 `--app` 接受显示名称，或当没有运行中的应用使用该显示名称时，接受 bundle ID 或可执行文件名（`ControlCenter`）。窗口 ID（如 `w-4521`）在窗口生命周期内保持有效。用 `list-windows` 查找它们。

**Ref 携带每个元素自身的地址。** ref 会还原为它被捕获时的确切元素，因此相同的孪生元素（例如多个相同的 `Delete` 按钮）各自作用于自己的行。如果元素已被移除或应用已重启，该 ref 为 `STALE_REF`，`details.kind: retained_object_stale`，它绝不会回退为搜索相似元素。当应用未提供元素的地址时，该 ref 遵循跨平台规则，且必须是其窗口中唯一具有该角色、原生 id、名称和描述的元素。列表变化后请重新快照。

**按键。** 主修饰键是 `cmd`：`press cmd+c`、`press cmd+s`。退出快捷键（如 `cmd+q`、`cmd+shift+q`、`cmd+alt+esc`、`ctrl+cmd+q` 和 `cmd+shift+delete`）会返回 `POLICY_DENIED`，除非传入 `--force`。要退出应用，请使用 `close-app`。标点键可用于组合键，以符号形式（`,` `.` `/` `;` `'` `[` `]` `\` `-` `=` `` ` ``）或名称形式（`comma`、`period`、`slash`、`semicolon`、`quote`、`leftbracket`、`rightbracket`、`backslash`、`minus`、`equal`、`grave`）。每个键都使用当前键盘布局中能打出该字符的键发送，因此 `press cmd+,` 在 ANSI 和 JIS 键盘上都能打开设置。

**光标叠加层。** `cursor-overlay enable` 在点击穿透窗口中绘制代理光标。它从不移动操作系统指针或激活应用。macOS 的运动和图像限制见核心 `commands-system` 参考。