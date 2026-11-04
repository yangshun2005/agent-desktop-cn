# Linux 上的截图与剪贴板

X11 无需权限，也无需外部工具。Wayland 会就 `screenshot` 向用户请求一次授权，并需要 `wl-clipboard`。`agent-desktop permissions` 会显示具体情况：`screen_recording` 在 X11 上为 `not_required`，在 Wayland 上为 `unknown`。`screenshot` 没有超时标志。

## X11 上的 `screenshot`

```bash
agent-desktop screenshot --app gnome-calculator /tmp/calc.png
agent-desktop screenshot --screen 0 /tmp/screen0.png
agent-desktop screenshot /tmp/everything.png
```

- 无需权限，也无提示。目标可以是：窗口（`--app`、`--window-id`）、按 `list-displays` 索引指定的显示器（`--screen`），或不指定选择器时为主显示器（若无标记为主显示器，则为第一个）。不提供输出路径时，图像以 base64 形式在 `data` 中返回。
- 窗口从其自身的可绘制对象读取。在合成窗口管理器（mutter）下，被遮挡和最小化的窗口也能捕获。
- 在非合成窗口管理器（openbox）下无法这样读取。被遮挡的窗口返回 `ACTION_FAILED`，`details.reason: "covered"`，并建议用 `--headed focus-window` 将其置前；聚焦后再次捕获即可返回该窗口。最小化的窗口返回 `INVALID_ARGS`（"The window is not viewable and the window manager does not composite, so it has no pixels to capture"），并建议对其执行 `restore`。两者均为 `not_delivered`，`retry: safe`。在非合成管理器下，全屏和 `--screen` 捕获是精确的。
- 在 X11 上的 GNOME 等合成器下，根窗口只包含管理器自身的舞台，因此全屏和 `--screen` 捕获返回 `PLATFORM_NOT_SUPPORTED`，`details.reason: "compositing"`，并建议用 `--window-id` 捕获单个窗口。合成器下无法读取不属于任何 X 窗口的区域（壁纸、shell 装饰）。
- 同一进程的两个同名窗口：`--window-id` 捕获工具包报告为活动的那个；当恰好两个窗口共享标题时，捕获其孪生窗口。其他并列情况返回 `AMBIGUOUS_TARGET`（"More than one X11 window matches this window"），附带 `candidates`，并建议为窗口设置不同标题。
- 工具在捕获期间隐藏自身的光标叠加层并在之后恢复；它们既不阻挡截图，也不出现在像素中。
- 显示服务器或调用方截止时间超时仍为 `TIMEOUT`，包括捕获被遮挡窗口时。它不能证明遮挡，也不建议聚焦。
- 资源、连接和可见性失败保留各自的错误。只有无法证明其绘制的窗口才会得到 `covered` 建议。

## Wayland 上的 `screenshot`

Wayland 没有窗口位置，也没有逐窗口捕获。门户的 `Screenshot` 捕获桌面；CLI 默认返回主显示器，在多显示器桌面上会裁剪图像。`--screen N` 选择另一台显示器。

```bash
agent-desktop screenshot /tmp/desktop.png
```

- 首次调用会显示一个同意对话框（"Allow Applications to Take Screenshots?"，焦点在 Deny 上），它不在无障碍总线上。必须由人回答。CLI 等待门户的答复，以命令自身预算与 30 秒中较长者为准。
- 允许则返回所选显示器的 PNG。拒绝或关闭对话框则返回 `PERM_DENIED`。
- 无人回答：门户约 25 秒后放弃，调用返回 `TIMEOUT`，建议中会指明该对话框。对话框保持打开，后续调用立即返回 `PERM_DENIED`，不再弹出第二个对话框，直到有人将其关闭。不要循环重试：请用户回答它。
- 桌面按用户为主机进程记住答案。允许后，后续调用约 50 毫秒返回，无对话框。拒绝后，每次调用都是 `PERM_DENIED`，直到用户在桌面的隐私设置中重置。
- `--screen N` 和显示器目标会将整个桌面图像裁剪到该显示器。裁剪需要能确认显示器布局的尺寸；否则调用返回 `PLATFORM_NOT_SUPPORTED`，原因为 `display_mapping`。
- 已知的显示器布局会在同意后重新检查，包括原本只有一台显示器的情况。布局变化返回 `ACTION_FAILED`，不返回像素。
- 窗口目标（`--app`、`--window-id`）在 Wayland 上永久为 `PLATFORM_NOT_SUPPORTED`。请捕获显示器，或用 `snapshot` 读取窗口。
- 门户将 PNG 写入用户主目录下。CLI 仅接受位于临时、运行时或主目录内的文件，检查它是用户为此请求创建的正规文件，读取后删除。

## 剪贴板

```bash
agent-desktop clipboard-set "hello from agent-desktop"
agent-desktop clipboard-get
agent-desktop clipboard-clear
```

格式和输出遵循核心 `commands-system` 参考。文本、图像和文件列表在 Linux 上均可使用。

### X11

- `clipboard-set` 将选区交给一个分离的所有者进程——辅助程序 `agent-desktop-linux-clipboard-owner`，它提供内容服务，直到另一个客户端取得选区或 `clipboard-clear` 结束它。CLI 先退出；所有者存活期间内容保持可用。
- 辅助程序必须与 `agent-desktop` 二进制位于同一目录，且是正规可执行文件，不是符号链接，由用户或 root 拥有。它及其目录不得对组可写，即使该组看似私有；文件不得对所有人可写。在使用以组可写 umask 构建的版本前，请从辅助程序及其目录移除组写权限。对所有人可写的目录仅在具有粘滞位且无组写权限时才被接受。777 副本会被拒绝，返回 `PLATFORM_NOT_SUPPORTED`，"writable by others"。npm 安装和发布归档会将其随附到位。单独复制的二进制无法设置剪贴板：`PLATFORM_NOT_SUPPORTED` 会指明缺少辅助程序。两种拒绝均为 `not_delivered`、`safe`。
- 没有剪贴板管理器时，内容随所有者结束。GNOME 的 shell 在所有者死亡后仅保留文本和 PNG，文本上限 4 MiB，且不接管文件复制或自定义目标：不要依赖内容比所有者存活更久。
- 文件复制提供 `x-special/gnome-copied-files` 和 `text/uri-list`。纯文本路径在文件管理器中粘贴不出任何内容。
- 超过 X 服务器单请求限制的内容通过 `INCR` 传输，因此大内容可用。
- "没有内容"和"目标不可用"是不同的答案：缺失的格式为 `absent`，不是错误。

### Wayland

- 剪贴板通过 `wl-clipboard`（`wl-copy`、`wl-paste`）：安装该包。文本、图像和文件列表在 GNOME 下与外部读写方往返正常。`clipboard-clear` 运行 `wl-copy --clear`。
- 路由遵循显示服务器检测器：空的 `WAYLAND_DISPLAY` 视为未设置，过期的套接字会落到可达的 `DISPLAY`。一旦选择 Wayland，剪贴板失败绝不回退到 X11。缺少 `wl-copy` 或 `wl-paste` 返回 `PLATFORM_NOT_SUPPORTED`，附带 `wl-clipboard` 安装提示，即使设置了 `DISPLAY`。
- 选择 Wayland 后，连接失败（"Failed to connect to a Wayland server"）、无键盘的 seat（"This seat has no keyboard"）以及已停止服务客户端的 shell 的超时，都是不同的错误，绝不会是空剪贴板。

### 通过 FFI 使用剪贴板

C 库到达同一代码；其剪贴板 API 仅支持文本。设置和清除获取跨进程交互租约；读取有截止时间但不获取该变更租约。匹配的所有者辅助程序可与 `libagent_desktop_ffi.so` 位于同一目录；它无需位于宿主的可执行文件旁。该库没有稳定的 SONAME。