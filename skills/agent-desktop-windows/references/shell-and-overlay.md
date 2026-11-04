# Shell 界面、通知与光标叠加层

打开本文件可唤起 Shell 界面、读取或驱动通知、使用光标叠加层，或处理托管（UWP）窗口标识与菜单检测。

## Shell 界面

`open-system-surface --surface <kind>` 唤起一个 Shell 界面，并返回其呈现窗口的 `w-<hwnd>` 标识。Windows 类型：`start-menu`、`taskbar`、`system-tray`、`system-tray-overflow`、`action-center`。用 `snapshot --surface <kind>` 读取该界面。`snapshot --window-id` 会以 `WINDOW_NOT_FOUND` 拒绝该句柄，因为窗口清单不包含 Shell 窗口。

- **该命令会抢占前台。** 严格无头模式会在唤起任何内容之前以 `POLICY_DENIED` 拒绝。需传入全局 `--headed`。已存在的界面会直接返回，不会再次唤起。
- **`snapshot --surface <kind>` 无需 `--app`，且可在无头模式下工作。** 界面已关闭时返回 `WINDOW_NOT_FOUND`，并给出指向 `open-system-surface` 的建议。
- **`quick-settings` 返回 `INVALID_ARGS`**，并指向 `action-center`，快捷操作位于其中。
- **`start-menu` 就是 Win 键唤起的任何内容。** Windows 11 之前，它是一个搜索叠加层，其根节点包含 `SearchTextBox`，而非磁贴网格。应按搜索界面来驱动它。
- **托盘项使用通用命令。** 没有 Windows 专用的托盘命令。`snapshot --surface system-tray` 返回提升后的通知区域项；在没有托盘图标的机器上，零项是正确答案。`snapshot --surface taskbar` 给出通知区域托盘 `button` 的引用。`snapshot --surface system-tray-overflow` 给出溢出项的引用：需先用 `open-system-surface --surface system-tray-overflow` 唤起浮出控件，否则点击会因遮挡检查而失败。点击 `system-tray` 项会报告 `delivered_unverified`，并附带 `retry: unsafe`，因为合成点击无法确认所属应用对其做了什么。点击溢出项未经测试。

## 通知

通知命令通过 UI Automation 读取和驱动操作中心。从不使用 WinRT 的 `UserNotificationListener`，因此其按机器设置的同意设置不会导致调用失败。读取器以标志性 `AutomationId` 为键：有通知时为 `MainListView`，无通知时为空状态形状，以及顶层 `ClearAllButton`。

- **读取时操作中心必须已打开。** `list-notifications` 在无头模式下读取已打开的操作中心。操作中心关闭时，严格无头模式会在唤起它之前返回 `POLICY_DENIED`，而 `--headed` 会唤起它并在之后恢复桌面。`dismiss-notification`、`dismiss-all-notifications` 和 `notification-action` 始终需要 `--headed`，`wait --notification` 在同一规则下会在首次轮询时拒绝。
- **空的操作中心列出零条条目。** 既不显示通知列表也不显示空状态标志的操作中心返回 `PLATFORM_NOT_SUPPORTED`，其 `platform_detail` 指明缺失的标志。无法识别的树是错误，绝不是空答案。
- **索引不是标识。** 每次变更都会重新读取操作中心，并将该索引处的条目与 `--expected-app` 和 `--expected-title` 比较。不匹配为 `NOTIFICATION_NOT_FOUND`。条目未提供的操作名称为 `ACTION_NOT_SUPPORTED`。
- **无效的关闭会失败。** Shell 可能接受条目的 `DismissButton` 却不据此执行。当目标是唯一条目时，改为运行已验证的全部清除控件。否则调用报告 `ACTION_FAILED`（`delivered_unverified`）。`dismiss-all-notifications` 以清除前捕获的条目为判定依据，因此运行期间到达的条目不算失败。
- **`wait --notification` 每次轮询都会打开并关闭操作中心**，与 macOS 相同。每次轮询会接管已打开的操作中心，并在之后恢复条目状态。实测成本：每次轮询最低 1243.5 ms，中位数 1254.2 ms。等待在轮询之间还会休眠最多 500 ms，因此 `--timeout` 需兼顾两者。在一个间隔内出现并被关闭的通知可能被漏掉。Toast 仅在操作中心打开时加入其中，因此轮询之间发布的 Toast 永远不会落入。要安排到达，请自行保持操作中心打开；轮询会接管它而不关闭它。
- **此输出敏感。** 通知区域界面会显示已安装后台代理的名称，包括安全和远程访问产品，且 `list-notifications` 会逐字返回标题和正文。不做任何脱敏。

## 光标叠加层

`cursor-overlay enable` 在 Windows 上渲染：一个分离的渲染器进程在点击穿透、置顶、非激活窗口中绘制仅用于呈现的光标。

```powershell
$env:AGENT_DESKTOP_HOME = "$env:TEMP\ad-scratch"
$start = agent-desktop session start --cursor | ConvertFrom-Json
$env:AGENT_DESKTOP_SESSION = $start.data.session_id
agent-desktop cursor-overlay enable --label "Opening menu" --accent "#FF3B7B"
agent-desktop cursor-overlay disable
```

- **先启动会话再快照。** 叠加层需要会话，在会话之外拍摄的快照会落入全局命名空间，会话作用域的操作无法看到它（`SNAPSHOT_NOT_FOUND`）。依次运行 `session start`、`cursor-overlay enable`、`snapshot`，然后执行操作。
- **`data.rendered` 是渲染器的确认。** 仅当渲染器连接到会话的控制管道并确认控制后，`enable` 才返回 `true`。当无法及时联系、启动或听到渲染器时返回 `false`。`disable` 不带 `rendered` 字段。`session start --cursor` 也会启用叠加层，且仅回显 `cursor_overlay`。
- **有头操作跳过移动动画。** `--headed` 操作会移动真实光标，因此叠加层不会为其动画化移动或点击。已启用的叠加层仍会绘制其静止光标。
- **每次操作约 355 ms 开销。** 无头 `click` 在无叠加层时耗时 427 ms，有叠加层时 782 ms。启用一次性约 50 ms。一百次叠加点击约增加 35 秒。
- **它从主显示器工作区的中心开始**，而非你所驱动的应用之上。首帧在 `enable` 返回后不久出现，因此截图前请轮询像素。
- **它绘制在任务栏及其他置顶 Shell 界面之上。**
- **它忽略"在 Windows 中显示动画"设置。** 在标准 Windows Server 主机上，Windows 默认报告动画关闭，因此显式 `enable` 即视为选择加入。macOS 则遵循"减少动态效果"。
- **标签是黑色胶囊配白色 Segoe UI 文本**，尺寸随文本。`--fill` 和 `--rim` 仅给光标着色，不给标签着色。
- **仅在有事可说时显示标签。** 不带 `--label` 的 `enable` 会问候一次，带 `--image` 则不问候。之后，仅携带描述的操作才绘制标签。每次 `enable` 和每次操作都会替换标签文本。
- **无指令 6 秒后淡出**，下一条命令时立即恢复。
- **插入符来自 Text 模式。** 文本输入后，代理的插入符位于字段 Text 模式放置插入点的位置，处于其自身窗口中，并在光标前往别处时消失。没有 Text 模式的字段保留元素轮廓。
- **它以物理像素绘制**，即元素边界报告所用的空间，因此不会按显示器 DPI 再次缩放。多显示器放置仅经单元测试，未在真实多显示器桌面上验证。
- **测试框架作业对象会带走叠加层。** 带有 `JOB_OBJECT_LIMIT_KILL_ON_JOB_CLOSE` 的作业（CI 运行器和某些代理测试框架使用）会在作业关闭时结束分离的渲染器。此处叠加层消失不是渲染器缺陷，操作不受影响。
- **其控制管道信任任何以你身份运行的进程。** 参见 `references/permissions-and-elevation.md` 中的"The Cursor Overlay's Control Pipe"。
- **清理。** `cursor-overlay disable` 结束其会话的渲染器，`session end` 也会关闭它。当会话以其他方式结束（崩溃、`session gc`）时，渲染器会在 3000 ms 内自行回收。

## 托管（UWP）窗口标识

托管在 `ApplicationFrameHost` 中的应用通过其框架报告：`focused_window` 和 `list-windows` 给出框架的句柄作为窗口 `id`，所有窗口操作都以此为靶，而 `app` 和 `pid` 命名托管应用，从其 `Windows.UI.Core.CoreWindow` 读取。挂起的托管应用会丢弃其 `CoreWindow`，因此在恢复之前，条目读作其框架宿主，且针对托管 pid 检查的标识会失败关闭。

## 菜单检测覆盖范围

`menu` 界面检测器覆盖经典 Win32 菜单、WPF 上下文菜单和菜单栏下拉菜单，以及绘制在应用自身窗口内的 Chromium/Electron DOM 上下文菜单。WinUI3 和 MSIX 宿主未评估。对于这些家族之外的应用，请将"没有菜单打开"读作"未检测到"，而非菜单已关闭的证据。