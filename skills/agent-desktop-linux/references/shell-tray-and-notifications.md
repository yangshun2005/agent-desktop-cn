# Linux 上的菜单、Shell、托盘与通知

在目标桌面会话内运行。`agent-desktop status` 列出的是快照能力，不能证明弹窗已打开或托盘监视器正在运行。Openbox 是没有 GNOME Shell 的 X11 窗口管理器。本文中的 Mutter 指 X11 上的 GNOME Shell 会话。GNOME Shell 提供 Wayland 表面。

## 应用菜单与 GTK4 弹出框

在三种后端上，只要无障碍总线健康，`window`、`sheet` 和 `menu` 都可用。打开器之后，读取其 `data.surfaces`：出现 `type: menu` 条目意味着下一步应快照该菜单。若没有，等待菜单，然后运行 `list-surfaces --app gnome-calculator`。GTK4 弹出框异步出现。扫描不完整时，窗口条目的 `unclassified` 列表会标记 `menu`，因此该结果中缺少菜单并不证明它已关闭。

此示例需要运行中的 Calculator 和 `jq`，并会找到新的打开器 ref。在 Mutter X11 上，先关闭 Overview 并用下面的恢复步骤聚焦 Calculator：当 Overview 覆盖应用时，点击可能报告成功但并未打开其菜单。检查每条命令的 JSON 结果，包括等待命令。

```bash
menu_ref=$(agent-desktop snapshot --app gnome-calculator -i | jq -r '.. | objects | select(.role? == "switch" and .name? == "Main Menu") | .ref_id // empty')
test -n "$menu_ref" && agent-desktop click "$menu_ref"
agent-desktop wait --menu --app gnome-calculator --timeout 5000
agent-desktop list-surfaces --app gnome-calculator
agent-desktop snapshot --surface menu --app gnome-calculator -i
agent-desktop press escape
agent-desktop wait --menu-closed --app gnome-calculator --timeout 5000
```

- `list-surfaces` 列出应用的窗口、sheet 和打开的菜单，菜单上带 `item_count`。
- `snapshot --surface menu` 以弹出框为根。`find --surface menu` 使用相同范围。
- 菜单 ref 标识新近验证、完整弹出框上下文（含标签和树形）中的实际 AT-SPI 对象。
- 具名的 GTK4 弹出框项（Calculator 的 Preferences）可在其弹出框有可证明屏幕位置时通过该身份找到并点击。在 Wayland 上它没有位置：解析时 ref 被拒绝，返回 `ACTION_FAILED`，`details.check` 为 `geometry`。
- 上下文不可读或匹配到多个目标时，在投递前拒绝。
- 关闭后同一 ref 返回 `STALE_REF`，`details.kind: menu_closed`。对已关闭菜单快照返回 `ELEMENT_NOT_FOUND`，消息为 "The window does not present an open menu"。重新打开并获取新 ref。
- 内容已变化的打开弹出框返回 `menu_context_changed`。上下文读取不完整返回可重试的 `menu_context_unknown`。保持打开，取新的菜单快照后再使用新 ref。
- 弹出框仅含一个标签的菜单按钮会验证 `expand` 和 `collapse`，对其再次 `expand` 无需投递即成功。两者各约 0.6 秒：GTK4 弹出框有动画，检查每 50 ms 轮询一次。
- 若打开弹出框改变了打开器的无障碍名称（Calculator 的 `Advanced` 变为 `Advanced Advanced`），折叠前取新快照；ref 仅对一次快照有效。

| 工具包 | 暴露的菜单形态与边界 |
|---------|--------------------------------|
| GTK3 | 菜单栏子项或独立上下文菜单窗口 |
| GTK4 / libadwaita | 框架内的弹出框；项名称可从其标签恢复 |
| Qt | `QMenu` 弹出框；`QLineEdit` 上下文菜单可能不通过 AT-SPI 暴露任何内容 |
| Chromium | 独立框架中的原生上下文/工具栏弹出框；网页内容不是通用菜单表面 API |

- 仅可见的菜单栏不是打开的弹出框。
- `wait --menu` 和 `wait --menu-closed` 使用与菜单快照相同的检测器。用 `--app` 限定范围。
- `press escape` 发送全局按键，不先激活父窗口。菜单仍打开时它可能报告 `delivered_unverified`，因此用 `wait --menu-closed` 确认：`TIMEOUT` 且 `predicate: menu`（"Menu did not close before the deadline"）表示菜单仍打开。

在无头 Wayland 会话中，全局 `press` 在其临时 RemoteDesktop 会话结束后可能使 seat 失去键盘。GTK4 打开器随后可能变为展开状态却不呈现原生菜单，工具报告 `ACTION_FAILED`、`popup_not_observed`，投递未验证且重试不安全。应用启动时即有键盘并跨命令保持可避免此问题；失败后再添加键盘无法恢复弹出框。仅打开器展开从不证明菜单已打开：继续前检查原生菜单。

GTK4 原始菜单项可能无名，而快照显示其标签。可读标签不能确立安全的物理点击位置。Wayland 上普通应用指针操作被拒绝，返回 `ACTION_NOT_SUPPORTED`，cause 为 `wayland_session`。X11 上，仅在观察到可用弹出框位置后使用物理输入，并验证其效果。仅输入报告成功不能证明选中了某项。

GTK4 列表支撑的下拉框（如 Settings 的 Night Light Schedule）不是 menu 角色弹出框。其选项留在窗口树中：`wait --menu` 和 `snapshot --surface menu` 不覆盖它们。找到组合框，点击其新 ref，取窗口快照前用 `wait --text` 等待选项。

无弹出框状态、无选中、无操作的组合框（如 libadwaita 组合行）在 X11 上使用 `--headed select @ref "Choice"`。它点击组合框，在出现的列表中找到选项，通过该列表和弹出框默认操作选中它，并检查组合框显示的值。选项缺失时用 Escape 关闭弹出框并返回 `ELEMENT_NOT_FOUND`。无 `--headed` 时该 select 为 `POLICY_DENIED`，kind 为 `pointer_required`，除非组合框已显示该选项。`select` 仍拒绝时，加 `--headed` 不能补上缺失的语义操作。

X11 上，有头点击聚焦打开的下列表后，显式有头 `press down` / `press up` 和 `press return` 可选择项。先检查当前选项，再次变更前验证结果值。若 `get` 只返回组合框标签，对其新 ref 用 `snapshot --root` 读取显示值。按键报告成功不能证明选项已提交。

## GNOME Shell：Overview、顶栏与弹出框

Shell 身份来自 `org.gnome.Shell` 的会话总线所有者，并与其无障碍总线进程关联。进程名和翻译标签不能证明它。声明的表面需要可解析的结构容器。

| 表面 | 含义 | 可用性 |
|---------|---------|--------------|
| `system-tray` | 逻辑监视器项及其虚拟菜单 | 有 StatusNotifierWatcher 时的无头快照；无原生面板绑定 |
| `action-center` | 标签、开关和按钮的页脚行上方的容器 | 列表显示时有内容；`open-system-surface` 通过点击日期菜单打开它 |
| `quick-settings` | 系统菜单 | 拒绝：暴露的通用面板链无法证明其容器 |
| `taskbar` | Dock | 不声明或不可打开 |

- Shell 快照无需 `--app`。`list-surfaces` 列出用 `--app` 选中的应用。Shell 容器不出现在 `list-windows` 中。
- 角色形态和唯一性找到消息列表及打开它的日期菜单。英文标签、包装深度和屏幕位置不能证明任何事。持续丢失节点的 shell 读取以 `shell_container_unproven` 拒绝；shell 稳定后重试。
- 其他容器和系统按钮打开器在输入前返回 `ACTION_NOT_SUPPORTED`、`shell_container_unproven`。无头打开隐藏弹出框仍需有头权限。
- Overview 不是 `--surface` 值（`INVALID_ARGS`）。Mutter X11 上，Activities Overview 打开时有头应用聚焦返回 `ACTION_FAILED`，cause 为 `overview_open`。发送全局 `agent-desktop press escape`，然后 `agent-desktop --headed focus-window --app gnome-calculator`，再执行应用操作。此 X11 恢复不确立 Wayland 窗口控制支持。
- 原生 shell 点击需要有头权限，X11 上任何指针事件前需树命中测试。Shell 不回答比容器更细的点查询，因此适配器按绘制顺序读取每个显示的 shell 节点。瞄准点上方最后一个节点必须是目标或位于其下方，且目标范围必须两次读取一致，间隔至少 50 ms。
- 目标上方绘制的节点、仍在移动的目标、未显示或已离开树的目标以 `ACTION_FAILED`、`pointer_hit_unproven` 拒绝且不投递。shell 稳定后重试。读取期间离开树或不回答角色的另一节点使点击未证明，因为它可能覆盖了目标。每个效果都从 shell 读回。
- Wayland 上有头 shell ref 点击在任何输入前以 `ACTION_NOT_SUPPORTED`、cause 为 `wayland_session` 拒绝，弹出框打开器以 `pointer_hit_unproven` 拒绝，因为共享命中测试无法证明其接收者。

## 服务要求与精确拒绝

Shell 无障碍树、通知守护进程和托盘监视器是独立要求。健康的 app 总线不能确立其中任何一个。

| 缺失要求 | 结果 |
|---------------------|--------|
| Shell 会话所有者或匹配的无障碍树 | 特定原因的缺失树拒绝；不猜测 shell 身份 |
| 未证明的 shell 容器或打开器 | `ACTION_NOT_SUPPORTED`、`shell_container_unproven` |
| 无通知服务 | `PLATFORM_NOT_SUPPORTED`、`daemon: absent` |
| 通知服务所有者既非 shell 也非同一用户且 argv 命名 shell 自身通知模块的进程 | `PLATFORM_NOT_SUPPORTED`、`daemon: unsupported`；服务器品牌不是身份 |
| 无 `org.kde.StatusNotifierWatcher` | `PLATFORM_NOT_SUPPORTED`、`watcher: absent`；指引命名所需协议后端 |
| 通知服务不可读 | `TIMEOUT`、`daemon: unreadable` |
| 托盘监视器不可读 | 调用超时时 `TIMEOUT`，否则 `INTERNAL` |

守护进程不可用不等于通知列表为空。监视器缺失不等于托盘为空。不要重启你不拥有的 shell 或无障碍总线。

## 通知

**每个通知字段都来自 shell 的控件布局。** 通知协议和 GNOME 的无障碍树都不命名通知的应用、标题、正文、时间或控件，因此适配器在 shell 放置它们的位置读取。已在 GNOME Shell 46.0（Ubuntu 24.04）的私有 Mutter X11 和原生 Wayland 会话验证，未在其他版本验证。以不同方式布局条目的 shell 会被拒绝（`INTERNAL`，"no readable title"）或误读，绝不通过猜测修复。适配器不使用文本、英文单词或页脚按钮位置来查找列表、条目或其部件。`notification-action` 只匹配你传入的操作名。

- `list-notifications` 无头读取 GNOME Shell 的消息列表，包括关闭时，并排除横幅副本。
- 索引是未过滤列表中从 1 开始的位置，最新在前。过滤不重新编号。
- `--app` 为不区分大小写的包含匹配。`--text` 搜索应用、标题和正文。`--limit N` 保留过滤后前 N 个条目，各带自己的索引。
- 空列表是成功的空结果。
- `wait --notification` 建立基线并等待该基线之后新增的匹配条目。

条目如何读取：

- 列表是页脚行正上方的容器，页脚行子项依次为标签、开关和按钮。空列表是该容器无条目。
- 头部是容纳条目无名按钮的容器。其第一个标签是应用。后续标签是时间，被丢弃。
- 头部外的标签是标题，然后是正文。头部外的具名按钮是操作。
- 无此头部的条目无法读取，命令失败。

协议无历史查询，GNOME 不暴露通知 ID，因此索引是位置而非身份。单条命令内，条目是其无障碍对象加其文字。每次单项变更读取列表两次，检查索引处的预期应用和标题，检查两次读取找到同一对象，并通过对象离开列表证明效果。

示例假设已存在恰好一条来自 `Skill Example`、标题为 `Beta` 的一次性通知。`jq` 选出实际索引，变更同时校验两个身份字段：

```bash
agent-desktop list-notifications
notice_index=$(agent-desktop list-notifications --app "Skill Example" --text Beta | jq -r '.data.notifications[] | select(.app_name == "Skill Example" and .title == "Beta") | .index')
test -n "$notice_index" && agent-desktop --headed dismiss-notification "$notice_index" --expected-app "Skill Example" --expected-title Beta
agent-desktop list-notifications --app "Skill Example"
```

单项 `dismiss-notification` 和 `notification-action` 需要 `--headed` 及至少一个 `--expected-app` / `--expected-title`。`dismiss-notification` 以 `data.dismissed` 返回移除的条目。

- 无身份标志：`INVALID_ARGS`。无头变更：`POLICY_DENIED`，包括空列表上的 `dismiss-all-notifications`。
- 索引缺失或身份变化：`NOTIFICATION_NOT_FOUND`。`--app` 不重新编号列表：INDEX 仍是未过滤列表中的位置，除非该条目来自指定应用，否则命令返回 `NOTIFICATION_NOT_FOUND`。
- 每次变更后重新列出：索引之后可能指向另一条通知。
- 变更用指针按 shell 控件，需要 X11。Wayland 上变更在任何输入前以 `ACTION_FAILED`、`pointer_hit_unproven` 失败。
- 命令通过点击时钟菜单打开关闭的消息列表，调用后再关闭。若无法打开列表，错误建议会说明：手动打开列表并重试，因为已显示的列表按原样使用并保持显示。
- 关闭按条目的头部按钮，最后一个先按。两者外观相似，因此由效果决定：之后条目自身对象必须从列表消失，且其他条目的对象不得消失。仅展开条目的按钮会使其保留，因此接下来按另一个。
- 两个按钮后条目仍存在为 `ACTION_FAILED`，`delivery: delivered_unverified`，绝不成功。点击后读取不完整给出相同代码，消息为 "The click was delivered but the message list could not be read afterwards"，`retry: unsafe`。目标可能已消失。决定剩余内容前重新列出。不要盲目重复旧索引。
- `dismiss-all-notifications` 通过各条目自己的头部按钮逐个关闭范围内条目，仅当条目自身对象离开列表时计为已关闭。它绝不按页脚按钮，该按钮除位置外无属性。`--app` 限制范围。有头空 dismiss-all 返回零且不打开消息列表。`dismiss-all-notifications` 返回 `data.dismissed_count`，以及无法关闭条目的 `data.failures` 和 `data.failed_count`。
- 对 `notification-action`，传入行 `actions` 列表中的名称，例如 `Open`。匹配忽略大小写。操作缺失为 `ACTION_NOT_SUPPORTED`，带 `available_actions`。若条目按钮未布局，命令先用头部按钮展开它，再按最后一个。若该按压反而移除条目，结果为 `ACTION_FAILED`。
- 仅当条目对象离开列表时操作才算数，GNOME 在操作后如此，除非通知常驻。`details.evidence` 在也见到守护进程自身 `ActionInvoked` 信号一次时为 `action_invoked`，否则为 `entry_removed`。结果始终为 `delivered_unverified`，绝不声称接收应用的效果。条目仍在列表时按下的按钮为 `ACTION_FAILED`，带 `delivered_unverified`。

## `open-system-surface` 接受的表面

`open-system-surface --help` 显示每个平台的值。Linux 上只有两个可用：

| 值 | 结果 |
|-------|--------|
| `system-tray` | 需要 `org.kde.StatusNotifierWatcher`。没有时：`PLATFORM_NOT_SUPPORTED`、`watcher: absent`，与 `snapshot --surface system-tray` 相同的拒绝。不返回窗口 |
| `action-center` | 容器和打开器被证明时打开 GNOME 消息列表；否则 `ACTION_NOT_SUPPORTED`、`shell_container_unproven` |
| `quick-settings` | 任何输入前 `ACTION_NOT_SUPPORTED`、`shell_container_unproven` |
| `taskbar` | `INVALID_ARGS`：无标准 shell 显示 dock |
| `start-menu`、`system-tray-overflow`、其他值 | `INVALID_ARGS`，命名 `system-tray`、`action-center` 和 `quick-settings` |

重试不支持的值给出相同拒绝。

## StatusNotifierItem 与 AppIndicator 托盘

托盘快照需要 `org.kde.StatusNotifierWatcher`。适配器读取其注册项和 `org.kde.StatusNotifierItem` 属性。任意 XEmbed 图标不是 SNI 项。仅为你拥有的会话安装或启用后端。GNOME 通常通过 AppIndicator 获得。

`snapshot --surface system-tray -i` 在逻辑根下暴露监视器项，`com.canonical.dbusmenu` 条目作为虚拟子项。标题（包括 `System` 和重复标题）绝不绑定原生面板节点。`Passive` 项保留在快照中，`states: [hidden]`，点击它在发送任何调用前以 `ACTION_FAILED` 拒绝。

- 这些 ref 是虚拟协议身份，不是 AT-SPI 元素。原生文本获取器可返回 `INVALID_ARGS`、