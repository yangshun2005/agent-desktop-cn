---
name: agent-desktop-linux
tags: linux-automation, accessibility, at-spi, x11, wayland, ai-agent, gui-automation, cli
requirements:
  - agent-desktop
description: >
  agent-desktop CLI 的 Linux 平台指南。按命令组列出在 X11 和 Wayland 上
  哪些功能可用、如何检查 AT-SPI 无障碍总线、首次接触的注意事项（应用名、
  Chromium 和 Electron 标志、有头焦点、被阻止的按键组合），以及在哪里查找
  生命周期、截图、剪贴板、托盘、通知和故障排查的详细信息。
  触发场景："linux desktop automation"、"AT-SPI tree"、"automate GNOME app"、
  "agent-desktop on Linux"、"accessibility bus not available"、"empty
  supported_surfaces"、"wayland screenshot consent"，或任何 Linux GUI 交互任务。
---

# agent-desktop-linux

本文件只涵盖 Linux 上不同的部分。观察-操作循环、ref、JSON 信封、会话和追踪在各平台上都相同。

需要 Ubuntu 24.04+（x86_64 或 aarch64）、已登录用户的图形会话以及运行中的 AT-SPI 总线。其他发行版需要 at-spi2-core 桥接修复 dc0dc33；Ubuntu 22.04 的桥接在 `Action.GetActions` 上会中止。归档加载器的最低要求是 glibc 2.35。在目标应用的桌面会话内运行 CLI：会话总线、显示和运行时目录来自其环境。TTY 登录、没有会话变量的 SSH shell、没有总线的容器或其他用户的会话都看不到应用。GNOME 是实测桌面；X11 和 Wayland 会话都可用，但在输入、捕获和窗口控制上有所不同。观察和语义操作不需要辅助工具。

## 能力表

| 组 | 命令 | Linux 上的状态 |
|-------|----------|-----------------|
| 观察 | `snapshot`、`find`、`get`、`is` | 通过 AT-SPI 在 X11 和 Wayland 上可用。支持 `window`、`sheet`、`alert` 和 `menu`；GNOME 额外提供 shell 表面。虚拟托盘 getter 有限制；参见 shell 参考并检查 `status` |
| 表面 | `list-surfaces` | 列出应用的窗口、工作表和打开的菜单；shell 读取使用 `snapshot --surface` |
| Ref 交互 | `click`、`right-click`、`type`、`set-value`、`clear`、`select`、`toggle`、`check`、`uncheck`、`expand`、`collapse`、`scroll-to` | 需要可读的实时标识和所需的 AT-SPI 能力。菜单 ref 解析限制见 shell 参考；X11 有头回退还需要可用的几何信息，且不绕过标识检查 |
| 滚轮滚动 | `scroll`、`mouse-wheel` | 物理滚轮输入；仅 X11 和 `--headed` |
| 指针手势 | `double-click`、`triple-click`、`hover`、`drag` | 仅 `--headed`、X11。在 Wayland 上指针无法瞄准，这些命令以 `ACTION_NOT_SUPPORTED` 拒绝 |
| 焦点 | `focus` | 焦点权限需要 `--headed`。对尚未 `focused` 的可聚焦节点调用 AT-SPI `Component.GrabFocus`，不使用指针，并验证状态；GTK4 应用拒绝该调用，因此 `focus` 在那里以 `ACTION_FAILED` `grab_focus_rejected` 结束，`focus-window` 是聚焦其窗口的方式 |
| 键盘和鼠标 | `press`、`mouse-click`、`mouse-move`、`mouse-wheel` | `press` 和文本键在 X11 和经能力检测的 Mutter Wayland 会话上可用；指针命令仅 X11 和 `--headed`。参见 `references/display-servers-and-input.md` |
| 按住输入 | `key-down`、`key-up`、`mouse-down`、`mouse-up` | 在所有平台上失败关闭：在 Linux 上，`key-down` 和 `key-up` 以 `ACTION_NOT_SUPPORTED` 拒绝；`mouse-down` 和 `mouse-up` 在无头时以 `POLICY_DENIED` 拒绝，`--headed` 时以 `ACTION_NOT_SUPPORTED` 拒绝 |
| 应用和窗口 | `launch`、`close-app`、`list-windows`、`list-apps`、`focus-window`、`move-window`、`resize-window`、`minimize`、`maximize`、`restore` | 启动和列出在 X11 和 Wayland 上可用；`focus-window` 需要 X11。其他窗口控制和优雅关闭需要 X11 客户端。参见 `references/windows-and-lifecycle.md` |
| 显示器 | `list-displays` | 可用 |
| 捕获 | `screenshot` | X11：无需权限即可截取窗口；仅在非合成管理器下可截取显示器。Wayland：门户同意，支持显示器裁剪。无选择器时返回主显示器。参见 `references/capture-and-clipboard.md` |
| 剪贴板 | `clipboard-get`、`clipboard-set`、`clipboard-clear` | 可用；支持文本、图像和文件列表。Wayland 需要 `wl-clipboard` |
| 等待 | `wait` | 支持毫秒、`--element`、`--window`、`--text`、`--menu`、`--menu-closed`、`--event`。菜单等待共用菜单表面检测器；`--notification` 在 GNOME 上等待新通知 |
| 通知 | `list-notifications`、`dismiss-notification`、`dismiss-all-notifications`、`notification-action` | 仅 GNOME Shell；列出无头，变更在 X11 上需 `--headed`，且 `dismiss-notification` 和 `notification-action` 还需要 `--expected-app` 或 `--expected-title`。字段从 shell 的控件布局读取，仅在 GNOME Shell 46.0 上验证，每个变更通过其效果证明。Wayland 在输入前拒绝 shell 指针。通知所有者不是 shell 或其自身辅助程序时返回 `PLATFORM_NOT_SUPPORTED`。参见 shell 参考 |
| Shell 表面 | `open-system-surface` | 只有 `system-tray` 和 `action-center` 可能可用。`system-tray` 需要 StatusNotifierWatcher（否则以 `watcher: absent` 拒绝）；`action-center` 需要已证明的角色祖先。`quick-settings` 以 `shell_container_unproven` 拒绝，`taskbar` 总是拒绝，`start-menu` 和 `system-tray-overflow` 不受支持，尽管 `--help` 列出了它们。参见 shell 参考 |
| 光标叠加 | `cursor-overlay` | 需要活动会话；见下方光标叠加。X11 启用返回 `data.rendered: true`；Wayland 保存设置并返回 `data.rendered: false` |
| 系统 | `status`、`permissions`、`version`、`batch`、`skills`、`session`、`trace` | 可用 |

## 首次接触

- **先运行 `agent-desktop status`。** `supported_surfaces` 为空意味着无障碍总线没有响应，而不是桌面为空。阅读 `references/bus-and-permissions.md`；不要重启不是你启动的总线。
- **`--app` 接受 `list-apps` 打印的名称**（`gnome-calculator`、`nautilus`）或由已验证的可执行文件和参数标识支持的桌面条目名称或 id，绝不是窗口标题或任意注册表标签。桌面条目名称如 `Files` 在条目存在后即被接受。`list-windows --app` 是精确匹配（包括别名）；`list-apps --app` 将列出名称的子串与精确适配器别名结合，忽略大小写。
- **窗口 id 是不透明地址**，34 到 70 个字符，在窗口进程生命周期内有效；重启后重新运行 `list-windows`。`list-apps` 行携带 `bundle_id`（已知时为桌面 id）和 `process_instance`。
- **GTK4 应用嵌套很深。** `--max-depth` 和 `--skeleton` 计算语义层级，因此空的布局包装器会增加输出层级。`-i --compact` 仍显示嵌套的无名 `group` 节点：读过去即可。
- **空文件视图不能证明文件夹为空。** 如果图标视图没有暴露文件项，尝试可用的列表或详细视图，然后重新快照并验证应用暴露了哪些项。
- **窗口先于其内容出现。** 先运行 `wait --window "Text Editor" --app gnome-text-editor`，然后运行 `wait --text`，它匹配节点自身的名称、值或描述。对于 Web 引擎应用（从其 AT-SPI 工具包元数据检测），窗口观察等待三次相等的完整读取和 Web 文档角色，或请求的深度截断。尚无文档的 shell 在预算结束时保持不完整。`--timeout` 短于首次读取时返回 `TIMEOUT`，绝不返回部分树。
- **ref 必须是唯一的。** Linux 不存储元素地址，因此 ref 仅当其快照捕获时在其窗口中以其角色、原生 id、名称和描述是唯一元素，且恰好一个活动元素仍匹配时才解析。唯一性在整个窗口上计数，而非 `--skeleton` 或 `--max-depth` 快照呈现的浅树，因此浅树从未显示的孪生元素仍会拒绝该 ref。位置、大小和值从不计为标识，因此布局变化本身不会使 ref 失效，但孪生元素会：相同的按钮，或共享角色且无其他字段的视图，在任何内容到达应用之前以 `AMBIGUOUS_TARGET`、`details.kind: identity_not_unique` 拒绝，已消失的元素为 `STALE_REF`、`saved_ref_absent`。列表变化后重新快照。要在 X11 上选择两个相同控件之一，使用 `--headed mouse-click --xy`；Wayland 会话拒绝有头指针输入，因此那里孪生元素既无 ref 也无指针路径。ref 属于其被捕获的窗口，因此两个同标题窗口各自解析自己的 ref。
- **Chromium 和 Qt 应用需要启动标志或变量**才能出现在总线上；Electron 应用不需要，仅在 `chrome-sandbox` 未设置 setuid 时需要 `--no-sandbox`。参见 `references/troubleshooting.md`。
- **有头 ref 操作需要能获得焦点的窗口。** 窗口管理器从未激活的窗口以 `activation_not_granted` 结束。参见 `references/windows-and-lifecycle.md`。
- **GTK4 秘密字段可能显示其文本。** 只有密码角色被隐藏。将带隐藏字符的 GTK4 条目视为可被所有观察命令读取。
- **被阻止的按键组合在没有 `--force` 时拒绝。** 参见 `references/display-servers-and-input.md`。
- **打开菜单角色弹出框后，检查 `data.surfaces`。** `menu` 条目通向 `snapshot --surface menu --app <name> -i`。如果不存在，在假设打开器失败之前使用 `wait --menu` 和 `list-surfaces`。列表支持的下拉框是窗口的一部分，不是菜单表面；参见 shell 参考。
- **Shell 指针输入需要接收者证明。** 在 X11 上，有头 shell 点击从 shell 自身的树证明：接收者必须是目标且保持静止，否则点击在任何指针事件之前拒绝。Wayland shell 点击拒绝。通知需要 GNOME 通知服务；托盘快照还需要 StatusNotifierWatcher。
- **状态根必须私有。** 组可写或全局可访问的 `AGENT_DESKTOP_HOME` 或 `~/.agent-desktop` 会使 `snapshot` 以 `INTERNAL` 失败；对其运行 `chmod 700` 或让工具创建它。

## 等待窗口

等待应用窗口出现（`--window` 需要标题或标题子串）：

```bash
agent-desktop wait --window "Text Editor" --app gnome-text-editor --timeout 10000
```

要等待你引起的关闭，将其批处理以便在关闭前取得基线（Calculator 没有未保存文档提示来中断它）：

```bash
agent-desktop batch '[{"command":"close-app","args":{"app":"gnome-calculator"}},{"command":"wait","args":{"event":"window-closed","app":"gnome-calculator","timeout":10000}}]' --stop-on-error
```

要观察其他人关闭窗口，在他们关闭窗口之前启动
`agent-desktop wait --event window-closed --app gnome-text-editor --timeout 10000`。没有 `--window-closed` 标志。

## 光标叠加

启动会话并将其返回的 ID 传给每个叠加命令：

```bash
session_id=$(agent-desktop session start --name docs-example | jq -r '.data.session_id')
agent-desktop cursor-overlay enable --session "$session_id"
agent-desktop cursor-overlay disable --session "$session_id"
agent-desktop session end "$session_id"
```

没有会话时，`cursor-overlay enable` 和 `disable` 返回 `INVALID_ARGS`。
在 X11 上，启用返回 `data.rendered: true`；在 Wayland 上它保存设置并返回 `data.rendered: false`，不绘制叠加。

`session start --cursor --multi-agent` 返回 `data.rendered: false`，因为在代理操作之前不绘制任何内容。在 X11 上每个代理 id 随后获得自己的光标；在 Wayland 上不绘制任何光标。

## Linux 上的差异

- Firefox/Gecko 的 `select` 可以改变选中值而不触发 `change` 或 `input` 事件，因此依赖的页面处理器不会运行。即使值回读匹配，结果仍为 `delivered_unverified`。继续之前检查依赖的应用效果：仅值回读不是提交证明。
- 对同一个 GTK4 按钮的两次 `click` 调用间隔小于约 250 ms 时只生效一次。两者都回答 `ok: true` 和 `delivered_unverified`。要多次点击一个按钮，在点击之间等待效果：对显示结果的元素使用 `wait --element <ref> --predicate value --value <expected>`，或 `wait 300`。
- Ref 操作是语义 AT-SPI 调用，从不移动指针或改变活动窗口。在 `--headed` 下，ref `click`、`right-click` 和 `type` 先尝试物理输入，仅在 `not_delivered` 拒绝后回退到语义操作（Windows 相反）。在 Wayland 上，有头指针命令和有头 `select` 立即以 `ACTION_NOT_SUPPORTED` 拒绝，原因 `wayland_session`；有头 `type` 需要 Mutter RemoteDesktop 能力。`references/display-servers-and-input.md` 有完整规则。
- `select` 先尝试语义路径，其次指针；指针路径在 X11 上需要显式 `--headed` 请求。Qt 组合框的语义选择只高亮其弹出框行，因此无头选择在改变该高亮之前以 `POLICY_DENIED`、kind `pointer_required` 拒绝。有头时，在工具包索引无法映射到行的列表（GTK4 列表）上，它点击该行并检查其 `selected` 状态；在没有语义打开器的组合框（libadwaita 组合框行）上，它点击组合框打开，通过弹出框自身的列表选择选项并检查组合框显示的值。无头时，列表以 `selection_index_unproven` 拒绝，没有语义打开器的组合框以 `POLICY_DENIED`、kind `pointer_required` 拒绝，除非组合框已显示该选项。通过自身切换打开的 GTK4 下拉框无需 `--headed` 即可接受 `select` 并检查其显示的值。
- Ref 可能不暴露安全的语义变更路径。无头请求随后返回不支持的操作错误，不虚构 `hidden` 或 `disabled` 状态，也不切换到物理输入。GTK4 条目和密码的 `Activate` 操作就是这种情况：使用 `type` 编辑字段，或在意图指针点击时显式请求 `--headed`。
- 打开原生 Wayland Qt 组合框在调用其操作之前返回 `ACTION_NOT_SUPPORTED`、kind `native_popup_requires_input`。Qt 需要用户输入序列来显示该弹出框。已打开的组合框可以折叠；已证明使用 Xwayland 的 Qt 窗口保留其语义打开器。
- 无头 `type` 通过 `EditableText` 写入。终端的文本节点是名为 `Terminal` 的 `textfield`，其唯一操作是 `RightClick`；对其无头 `type` 是 `POLICY_DENIED`，检查 `supported_action`，其值为 null。它仅在 `--headed` 下接收按键。
- 按键组合中 `cmd` 和 `meta` 是 Super 键。键名是 macOS 和 Windows 名称加上 X keysym 名称。`press` 只接受已映射的键；`type` 处理布局无法产生的字符。
- 窗口操作作用于 X11 客户端；`focus-window` 和有头作用域 `press` 需要 X11 会话。无头作用域 `press` 在任一显示服务器上都需要实时目标焦点。聚焦另一工作区的窗口可能切换用户当前桌面。
- 精确 FFI ref 调用（`ad_resolve_element_exact` 和 `ad_execute_ref_action_exact_with_policy`）在所有平台上以 `ACTION_NOT_SUPPORTED` 失败；其边界规则先适用：边界必须相对于窗口。
- Ref 重新标识唯一元素，而非控件实例。销毁并重建其行的工具包（GTK4、Qt6）将具有唯一名称的 ref 解析到替换元素上，因此操作落在新元素上。相同控件则以 `AMBIGUOUS_TARGET` 拒绝。
- 保持已移除控件列出的工具包（GTK4 未设置父级、Qt6 `set