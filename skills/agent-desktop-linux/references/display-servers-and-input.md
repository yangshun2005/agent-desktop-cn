# Linux 上的显示服务器与输入

观察和语义 ref 操作使用 AT-SPI，在 X11 和 Wayland 上行为一致。输入、窗口控制和捕获则不同。

## 显示服务器如何选择

CLI 读取 `WAYLAND_DISPLAY`、`DISPLAY` 和 `XDG_RUNTIME_DIR`（空值视为未设置）。能接受连接的 Wayland socket 优先，因为 Xwayland 客户端运行在 Wayland 桌面上，而 X11 调用无法列出其窗口。来自其他会话的过期 `WAYLAND_DISPLAY` 会回退到可用的 `DISPLAY`。两者都不可达时，依赖显示的命令失败，原因为 `not_configured`、`missing` 或 `refused`，并列出尝试过的变量。

## X11 与 Wayland 对比

| 能力 | X11 | Wayland（GNOME） |
|------------|-----|-----------------|
| 观察、语义 ref 操作 | AT-SPI，无需工具 | AT-SPI，无需工具 |
| 按键、`press`、文本按键 | 进程内 XTEST，实时键盘映射 | Mutter RemoteDesktop，keysyms |
| 应用指针：`hover`、`drag`、`mouse-*`、物理点击、滚轮 | XTEST，`--headed` | 不支持：`ACTION_NOT_SUPPORTED`，`physical_delivery_started: false` |
| 窗口操作 | X11 显示上的 EWMH | 仅 Xwayland 客户端；原生窗口和取消最小化被拒绝 |
| `focus-window`、headed 作用域 `press` | EWMH 激活，然后为 `press` 发送按键 | 拒绝，`ACTION_NOT_SUPPORTED`，原因 `wayland_session`，包括 Xwayland 客户端 |
| 无头作用域 `press` | 实时显示和帧焦点守卫，无激活 | 实时帧焦点守卫，无激活 |
| `screenshot` | 窗口或显示；默认主显示 | Portal 授权；默认主显示，`--screen` 选择显示器 |
| 剪贴板 | 原生选区所有者 | `wl-clipboard` |
| GNOME Shell ref 点击、弹出菜单打开器 | `--headed`，接收方证明来自 shell 自身的树 | ref 点击拒绝，`wayland_session`；弹出菜单打开器因未证明被拒绝；见 shell 参考 |

X11 除二进制及其剪贴板辅助工具外无需其他工具。不使用 `xdotool`、`ydotool`、`xclip` 和 `wmctrl`。`wl-clipboard` 是唯一的外部工具，仅在 Wayland 上使用。AT-SPI 操作在两种服务器上都不需要输入设备。

## Headed 与无头

ref 操作默认无头且语义化：指针和活动窗口从不改变，无头操作报告 `semantic_api`。指针命令（`hover`、`drag`、`double-click`、`triple-click`、`mouse-move`、`mouse-click`、`mouse-wheel`）需要全局 `--headed`，因为每条 X11 合成指针路径都会移动用户的真实光标。

- 在 `--headed` 下，ref `click`、`right-click` 和 `type` 先尝试物理输入。物理成功为 `physical_synthetic`，`verified: false`（XTEST 无回读）。
- 仅当物理尝试返回 `ACTION_NOT_SUPPORTED` 或带 `not_delivered` 的几何拒绝（`check: geometry`）时，适配器才回退到语义操作；`details.physical_fallback` 记录跳过的 `physical_input` 步骤和拒绝。焦点、策略、超时、不确定和部分投递拒绝从不回退。
- 在 Wayland 上，CLI 的指针门控在该回退之前运行：普通应用的 headed 指针命令立即被拒绝，`ACTION_NOT_SUPPORTED`，原因 `wayland_session` 和 `physical_delivery_started: false`（包括 `click`、`hover`、`double-click`、`mouse-wheel`、`mouse-down` 和 `select`）。headed `type` 在 Mutter 能力探测成功后通过 `RemoteDesktop.type_text` 发送按键。
- Windows 相反（语义优先）；不要将该假设跨平台套用。

物理 ref 寻址按键需要在短等待内目标帧为 `active`，否则 `check: window_active`。Headed `type` 和 ref 寻址按键按下还需要目标或其已证明的内部编辑器为 `focused`：在每个 X11 文本计划之前，以及在 Wayland 上，在引导 Shift 和每次后续 key-down 之前。内部编辑器证明需要可编辑目标、同所有者命中、回溯到目标的父链，以及声明 `EditableText` 的聚焦编辑器。

关闭 GTK4 菜单可能使文档失去焦点，即使其窗口仍为活动。GNOME 46 的 GTK4 桥接拒绝 AT-SPI `Component.GrabFocus` 为不支持。如果目标和其已证明的内部编辑器都未聚焦，工具无法自动恢复该文档的焦点：请自行聚焦文档，或通过 `EditableText` 使用无头 `type`。带 `physical_delivery_started: false` 的 `target_focus` 拒绝未发送任何按键，等待不会提供缺失的焦点操作。

无头语义操作不需要活动帧：向后台窗口的无头 `type` 可用。在 X11 上向后台应用的无头 `press --app` 失败，`display_active_window`。当无头 `press --window-id` 无法确认命名窗口的精确焦点时，返回 `STALE_REF`、`check: exact_window_focus`、`cause: exact_window_focus_unconfirmed`。将该窗口置于前台，刷新 `list-windows`，然后重试。活动帧门控是必要但不充分的：当弹出菜单持有抓取时，即使帧读取为活动，按键也会到达菜单，因此按键步骤保持未验证。

指针形式瞄准 ref 边界的中心，适配器在位置未证明处不提供该中心。用新的 `get` 读取结果。

ref 寻址的双击和三击在每次按下前证明指针接收方，因此第一次点击抬起的窗口在第二次之前可见。检查共享 300 ms 的按下到按下预算，在 GTK、Qt 和 GNOME 将两次按下读作一个手势的 400 ms 内。接收方改变，或检查超出预算（`details.kind: multi_click_window`），以 `ACTION_FAILED` 停止手势；`details.delivered_clicks` 计数已发送的点击，处置为 `delivered_unverified`。重试前检查应用。

`hover @ref` 在移动前以相同方式证明接收方；被覆盖的目标或未证明的接收方被拒绝（`ACTION_FAILED`、`not_delivered`，光标未移动）。`hover --xy` 没有可证明的目标元素。

## 按键与 `press`

```bash
agent-desktop press ctrl+a
agent-desktop press return --app gnome-text-editor
```

- 键名：macOS 和 Windows 名称（`return`、`escape`、`tab`、`space`、`home`、`pageup`、`f1`-`f24`）、标点名称（`comma`、`period`、`greater`、`leftbracket`）以及 X keysym 名称如 `Page_Up`。`delete` 是 Backspace；`forwarddelete` 是前向 Delete 键。单个字符是其自身的键。未知名称为 `INVALID_ARGS`、`not_delivered`，并列出接受的形式。
- 裸大写字母（`H`）在所有平台上都是 `shift+h`；与其他修饰键一起时大小写被忽略（`ctrl+A` 即 `ctrl+a`）。
- `meta`、`cmd`、`command`、`super` 和 `win` 在组合中命名 Super 修饰键；裸键使用 `super` 或 `meta`，不要用 `cmd`、`command` 或 `win`。
- 不带 `--app` 或 `--window-id` 的 `press` 发送到聚焦窗口。无头 `press --app` 要求目标进程拥有活动窗口；`press --window-id` 还要求精确窗口的焦点。后台或最小化目标被拒绝。见 `references/windows-and-lifecycle.md`。
- 按键携带实时键盘映射所需的 Shift 级别。在 X11 上，`press` 只接受活动布局映射的键和上述命名 keysyms；在 Wayland 上，它接受可打印 ASCII、功能键和修饰键 keysyms，不读取布局。布局无法产生的字符（`é`、`你`）被拒绝且不投递（X11 上 `INVALID_ARGS`，Wayland 上 `ACTION_NOT_SUPPORTED` 带 `character_outside_layout`）。在 X11 上，`--headed type` 通过备用键码发送此类字符；在 Wayland 上拒绝，因此重音和非拉丁文本请使用无头 `type` 或 `set-value`。

### 被阻止的组合

`press` 在发送任何按键前以 `POLICY_DENIED`、`not_delivered` 拒绝这些快捷键，因为它们会夺取前台、关闭窗口或切换会话。`--force` 发送它们：`alt+f4`、`alt+tab`、`alt+f2`、`meta+l`、`meta+d`、`meta+tab`、裸 `super`、`ctrl+alt+t`、`ctrl+alt+delete`、`ctrl+alt+forwarddelete`、`ctrl+alt+backspace`、`ctrl+alt+l`、`ctrl+alt+f1` 到 `ctrl+alt+f12`。应用快捷键如 `ctrl+q` 会发送。列表在适配器中，不在 CLI 帮助里；`agent-desktop press <combo>` 按组合显示拒绝。

## `type`

对于 Bash 中的字面换行，将 `TEXT_REF` 设为最新快照中多行文本字段的限定 ref，然后使用无头输入：

```bash
agent-desktop type "$TEXT_REF" $'First line\nSecond line'
```

空文本不发送按键，不请求目标焦点，报告 `not_delivered`；`--headed` 仍可在空插入派发前激活源窗口。

- 无头 `type @ref text` 通过 `EditableText.InsertText` 写入。无 `--timeout-ms` 时，`type` 等待 5000 ms 加每字符 30 ms；显式 `--timeout-ms` 替换该值。
- 在 `--headed` 下先发送物理按键（X11 上步骤 `XTEST.type_text`，Wayland 上 `RemoteDesktop.type_text`，`physical_synthetic`；结果仍尽可能回读字段），仅在无法发送按键处回退到 `EditableText`。它证明字段或其已证明内部编辑器的焦点；窗口其他位置的焦点不授权 ref 输入。之后验证值或应用效果。
- 终端的文本节点（名为 `Terminal` 的 `textfield`，仅 `RightClick`）仅在 `--headed` 下接受按键；此处的无头 `type` 为 `POLICY_DENIED`，检查 `supported_action`。其值为 null，因此 ref 无法读取终端输出：通过文件或其他应用验证终端命令做了什么。
- X11 物理 `type` 在发送任何内容前拒绝控制字符，包括 Tab 和换行，因为这些键可能在文本突发中移动焦点或提交表单。对这些字符的字面文本使用无头 `type` 或 `set-value`，导航或提交使用显式 `press tab` 或 `press return`。在 Wayland 上，`--headed type` 将换行作为 `Return` 发送，制表符作为 `Tab` 发送，这可能提交表单或移动焦点；其他控制字符以 `character_outside_layout` 拒绝。
- X11 物理文本在每个块之前检查实时键盘状态。如果该状态不受支持，物理路径在该块的按键前拒绝；仅当未发送任何按键时才应用语义回退，因为较早的块报告 `delivered_unverified`。
- 首次 `type` 到新文档会改变窗口标题，但快照中的 ref 仍可解析：对其 `get` 读取输入的值。仅当你寻址的 UI 因其他原因改变时才取新快照。

## Wayland 按键

Wayland 按键使用一个检测到的后端，会话总线上的 `org.gnome.Mutter.RemoteDesktop`。在创建会话或发出事件前，适配器探测接口的类型化 `Version` 属性。缺失服务或能力为 `PLATFORM_NOT_SUPPORTED`、`not_delivered`、`physical_delivery_started: false`；其他合成器从不回退到 X11。探测超时为 `TIMEOUT`。调用发送 keysyms 并等待 150 ms 让合成器排空。长驻宿主可为后续投递保留键盘会话；新的 CLI 进程打开自己的会话。结束会话释放其持有的按键。

- **引导 Shift。** 新会话发送一次单独的 Shift 轻击，并在真实按键前等待 100 ms，以便合成器附加其键盘。它不插入文本，但监视裸 Shift 的处理器可观察到它。复用仍存活的会话跳过引导轻击。
- **字符。** 仅可打印 ASCII、功能键、修饰键和厂商 keysyms；其他为 `ACTION_NOT_SUPPORTED`，原因 `character_outside_layout`。改用 `set-value` 直接设置值。在非美国布局上，键盘映射持有的字符仍可能被拒绝。拒绝是安全的：未发送任何内容。
- **应用指针。** AT-SPI 在 Wayland 上不提供全局窗口原点，因此普通应用指针命令无法瞄准（见上文 Headed 与无头）。GNOME Shell ref 点击和弹出菜单打开器也拒绝；见 `references/shell-tray-and-notifications.md`。
- **无活动窗口。** 在无 seat 键盘的 Wayland 会话上，没有窗口携带 `ACTIVE`，因此 `focus-changed` 等待超时。

## 滚轮与滚动

```bash
agent-desktop --headed mouse-wheel --x 640 --y 400 --dy 5
```

- 正 `--dy` 向上滚动，负值向下；正 `--dx` 向右，负值向左，在三个平台上均如此。负值写作 `--dy -5` 或 `--dy=-5`。
- 无 `--dy` 时，`--dx` 为零（包括显式 `--dx 0`）默认向下三行，`--dx` 非零时无垂直滚动。两轴均为零，或某轴超过 1000 次点击，为 `INVALID_ARGS`、`not_delivered`。
- 结果携带 `dy`、`dx`、`target`、`steps` 和 `disposition`，无 `scrolled` 成功标志。滚轮投递为 `delivered_unverified`：之后读取内容。
- `scroll @ref --headed` 在目标边界中心发送滚轮点击，仅对具有 `Scroll` 能力的节点。滚轮点击依赖页面大小（GTK 上约两行，Qt 上恰好三项），因此 `--amount` 计数点击，不是行。无 ref 的 `scrollarea` 不能接受 `scroll @ref`；对子项使用 `scroll-to` 或在坐标处使用 `mouse-wheel`。每次滚轮突发在该点使用 click 的指针门控：命中测试结果命名拦截器则拒绝突发；未知接收方以 `pointer_hit_unproven` 拒绝。如果较早的突发已发送，错误报告其 `delivered_clicks`、`delivered_unverified` 和不安全重试。下一步前检查内容。
- `scroll-to` 在 GTK3 和 GTK4 上通过滚动条滚动，在 Chromium 上通过 `ScrollTo`。在 Qt 上不工作：使用 `scroll --headed`。无可滚动轴的容器没有 ref。如果目标可见性未知，`scroll-to` 在滚动前拒绝；同等排名的滚动条在值写入前以 `AMBIGUOUS_TARGET` 拒绝。移动了视口但未显示目标的滚动不安全重试：先检查目标和处置。

## 拖拽

`drag` 拾取、沿插值路径移动、静止停留，然后释放。停留是故意的：悬停的文件夹或放置目标可能在指针下打开，任何后续移动都会落入其中并复制而非移动。默认持续时间为 300 ms；最小 100 ms，更短的 `--duration` 为 `INVALID_ARGS`、`not_delivered`，因为 GTK 等工具包需要多个移动事件才能开始拖拽。Linux 上默认放置延迟为 150 ms；仅当给慢速放置目标时间准备时才设置 `--drop-delay`。结果报告手势，不是结果：用新 `snapshot` 检查效果（文件移动、列表改变）。

## 中断的输入

客户端死亡时 X 服务器保留 XTEST 按下的按钮。CLI 在 `$XDG_RUNTIME_DIR/agent-desktop-input` 下记录其持有内容，该显示上的下一次调用在发送自己的事件前释放被杀进程留下的内容。如果杀死命令后按钮似乎卡住，运行任何带有效参数的输入命令，如 `agent-desktop --headed mouse-move --xy 640,400`，在新输入前释放它。