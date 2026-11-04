# Linux 故障排查

先运行 `agent-desktop status` 和 `agent-desktop list-apps`。

## 症状与原因

| 症状 | 原因与处理 |
|---------|------------------|
| `PLATFORM_NOT_SUPPORTED` "The AT-SPI accessibility bus is not available in this session"，或 `status` 显示 `supported_surfaces` 为空 | 会话总线或无障碍总线不可达：总线未响应。阅读 `references/bus-and-permissions.md`；在桌面会话内运行 |
| `list-apps` 为空，`status` 正常 | 总线已启动，但没有应用注册。启动一个应用，或重启在总线之前启动的应用 |
| 应用在运行，`--app` 却找不到 | 不在总线上，或名称错误。使用 `list-apps` 中的名称；参见"不在总线上的应用" |
| Chromium 显示总线条目但没有窗口 | 渲染器无障碍已关闭。添加 `--force-renderer-accessibility` |
| 窗口出现，`snapshot` 内容很少 | 内容尚未就绪。先 `wait --window`，再 `wait --text`；调大 wait 的 `--timeout` 或 snapshot 的 `--timeout-ms` |
| `snapshot` 返回 `TIMEOUT` | 截止时间前没有可用的观察结果，或 `--root` 下钻不完整。调大 `--timeout-ms`；参见"观察细节" |
| `ACTION_FAILED` `activation_not_granted` | 窗口管理器从未激活该窗口：`references/windows-and-lifecycle.md`，Focus |
| `ACTION_FAILED` `overview_open` | GNOME Activities Overview 处于打开状态。此时 `focus-window` 会拒绝；对未弹出弹窗的菜单开启项执行 `click` 或 `expand`（`details.kind: popup_not_observed`）时，若 Overview 打开也会报此错误。关闭后重试 |
| `ACTION_FAILED` `frame_not_active` 或 `check: window_active` | 窗口管理器已将窗口设为活动，但应用未报告。对该应用运行一次 `snapshot` 后重试（Electron 上为间歇性问题） |
| `focus @ref` 返回 `ACTION_FAILED` `grab_focus_rejected` | 工具包拒绝了 `Component.GrabFocus`（GTK4 会如此），焦点未移动。用 `focus-window` 聚焦窗口；`type` 和 `press` 需要目标已聚焦，参见 `references/display-servers-and-input.md` |
| `click` 或 `type` 返回 `POLICY_DENIED` | 缺少所需的策略或能力。阅读失败的检查项。Headed 策略仅在能证明接收者时才发送物理输入 |
| 托盘项没有 `Activate` 或 `ContextMenu` | 没有主协议动作。使用已公布的虚拟菜单项。Headed 输入无法从项标题证明原生接收者 |
| `press` 返回 `POLICY_DENIED` | 组合键被阻止。除非确实需要，否则不要使用 `--force` |
| `ACTION_NOT_SUPPORTED` `wayland_session` | Wayland 焦点、headed 作用域 `press` 或控制原生 Wayland 窗口不可用。自行将目标置于前台；使用全局 `press` 或应用自身的 UI |
| `ACTION_NOT_SUPPORTED` 且 `physical_delivery_started: false` | Wayland 上的普通应用指针输入。使用语义动作、`set-value` 或全局 `press` |
| `PLATFORM_NOT_SUPPORTED`、`daemon: absent` 或 `unsupported` | 通知协议缺失，或其所有者不是检测到的 shell 后端。在支持的桌面会话内运行；参见 shell 参考 |
| `PLATFORM_NOT_SUPPORTED`、`watcher: absent` | 不存在 SNI watcher。启动会话的托盘后端以提供 `org.kde.StatusNotifierWatcher`；扩展建议需要检测到后端 |
| `ACTION_NOT_SUPPORTED`、`shell_container_unproven` | 角色和父链无法确定 shell 容器或开启项。使用应用表面或逻辑 SNI 托盘。英文标签、包装层深度和屏幕位置无法确定弹窗身份 |
| `ACTION_NOT_SUPPORTED`、`native_window_unproven` | 列表运行期间显示服务器未响应原生窗口读取。`launch` 在返回此结果前会重试读取直到超时。显示稳定后再次运行 `list-windows`。另一进程的窗口若原生表面无法分类，仍会列出，且不会导致列表被拒绝 |
| `STALE_REF` 且 `kind: resolution_identity_unknown`，或点击超时且 `last_report` 为该值 | 严格解析无法证明菜单 ref 的实时身份。不要因菜单可读就推断应用卡死；参见 shell 参考 |
| `APP_UNRESPONSIVE` | 应用已停止（`SIGSTOP`）或主循环阻塞。恢复它；`close-app --force` 可结束它 |
| `STALE_REF` | 元素已消失或不再匹配。`details.kind: saved_ref_absent` 表示没有实时元素匹配存储的角色和稳定字段，需重新快照；`retained_object_stale` 在 Linux 上不会出现。`menu_closed` 需重新打开其开启项后再做新的菜单快照；`resolve_target_destroyed` 需替换目标；菜单或托盘的 `resolve_no_candidate` 可重试 |
| `ACTION_NOT_SUPPORTED`、`selection_index_unproven` | 工具包的选择索引无法安全标识可访问行。无头 select 无法映射该行，刷新也无济于事。在 X11 上使用相同 ref 运行 `--headed select`；参见"工具包限制" |
| `INTERNAL` "private file parent is accessible by group or other users" | 状态根目录可被组或其他用户写入。执行 `chmod 700`；参见 `references/bus-and-permissions.md` |

## 不在总线上的应用

只有当应用的工具包向无障碍总线注册后，应用才会出现在 `list-apps` 中。进程在运行但不在总线上时，CLI 会提示。

| 应用 | 需要 |
|-----|-------|
| GTK3、GTK4 | 当 `toolkit-accessibility` 开启或总线存在时默认注册。在总线之前启动的应用永远不会注册：重启它 |
| Qt | 进程环境中设置 `QT_ACCESSIBILITY=1` |
| Chromium 浏览器（Chrome、Chromium） | 命令行添加 `--force-renderer-accessibility`。仅 `ACCESSIBILITY_ENABLED=1` 会让浏览器上总线但没有窗口 |
| Electron | 无需无障碍标志。`chrome-sandbox` 未设置 setuid 时需 `--no-sandbox` |
| Flatpak、snap | 沙箱必须共享总线：`references/shell-tray-and-notifications.md` |

通过 `launch` 传递启动标志：

```bash
agent-desktop launch google-chrome --arg=--force-renderer-accessibility
```

在没有桌面设置守护进程的会话中（容器、CI、裸 Xvfb），`org.a11y.Status IsEnabled` 为 false，因此还需传 `launch --env ACCESSIBILITY_ENABLED=1`，否则浏览器永远不会加入总线，`launch` 会等待完整超时。

Chrome 在列表中显示为 `chrome`：之后使用 `--app chrome`。`launch` 结果带有 `renderer: chromium`。Chromium 和 Electron 在稳定前读取内容很少：给 `snapshot` 传 `--timeout-ms`，不要断定树为空。Chromium 菜单框架是无名顶层框架，不是窗口，因此不计为窗口事件。
Electron 限制：当命令是唯一的 AT-SPI 客户端时，其断开连接可能导致 Electron 无障碍桥崩溃（VS Code 在 `press` 时以 SIGSEGV 退出）。GNOME 桌面保持 shell 连接，因此不会发生。Electron 原生对话框（保存提示、拖放确认）不在 AT-SPI 树上：`close-app` 会计入它们，但任何读取命令都看不到。报告 `expandable` 的树项没有展开动作，菜单项无法通过 ref 点击：在其边界上使用 `--headed` 指针输入。

## 观察细节

- 在将缺失的控件视为不存在之前，先检查 `data.complete`（部分树参见核心 `commands-observation` 参考）。
- `--root @ref` 从该 ref 的快照中获取窗口和选项。
- `status` 显示最近一次 `find` 写入的快照。
- `find --first` 返回 `data.match`，而非 `data.matches`。
- `find --text` 匹配子树的文本，因此文本的祖先也会匹配。`wait --text` 匹配节点自身的名称、值或描述。
- 单元格内的单元格（列表行中）是两个 ref。
- 没有可用位置的节点没有 `bounds`，不参与命中测试。
- 仅当工具包报告密码角色时才隐藏密码字段。可见性关闭的 GTK4 `Gtk.Entry`、带 `InputPurpose.PASSWORD` 的条目和 `Adw.PasswordEntryRow` 都会显示文本：观察命令能读取它们。
- 没有 ref 的 `scrollarea` 无法接受 `scroll @ref`。
- 实时 `offscreen` 来自正面证据：最小化的窗口、滚出视口的 GTK3 哨兵行，或位于其框架、滚动容器或所有显示器之外的矩形。仅有 `VISIBLE` 而无 `SHOWING` 既不能证明隐藏也不能证明离屏。非最小化窗口中已证明的相交可能报告为 false；缺少位置证据则保持未知。快照不携带实时 `offscreen` 或 `hidden`。`offscreen` 不是受支持的 `get`/`is` 属性：在可操作性拒绝中使用实时证据。

## GTK 编辑器标签页

GNOME Text Editor 可能在两个标签组下都暴露聚焦文档的编辑器：`alpha.txt` 和 `beta.txt` 都可能包含 `BETA FILE`，且都处于聚焦和可见状态。原生 AT-SPI 也暴露同样的重复。组标签可能命名一个文件，而其子编辑器暴露另一个文件的内容，因此标签和子项的聚焦状态无法证明 ref 将编辑哪个缓冲区。

切换到目标标签页，重新快照并在编辑前验证活动内容。例如，`beta.txt` 处于活动状态，`alpha.txt` 紧邻其左侧于聚焦的 Text Editor 窗口中，已知不同内容为 `BETA FILE` 和 `ALPHA FILE`：

```bash
agent-desktop press ctrl+pageup --app gnome-text-editor
agent-desktop snapshot --app gnome-text-editor -i --compact
agent-desktop press ctrl+a --app gnome-text-editor
agent-desktop press ctrl+c --app gnome-text-editor
agent-desktop clipboard-get
```

检查复制的文本是否为 `ALPHA FILE`。这会替换剪贴板并选中缓冲区。使用 Ctrl+PageDown 向右切换。若身份仍不明确，在写入前停止。对于独立目标，在各自窗口中打开每个文档（`agent-desktop launch gnome-text-editor --arg=--new-window --arg=/tmp/note.txt`），并使用 `--window-id` 配合新快照。若 Text Editor 已打开该文件，该 launch 会打开一个空的 `New Document (Draft)` 窗口，并在其现有窗口中将该文件显示为标签页，因此运行 `list-windows` 查看哪个窗口显示它。

## 工具包限制

| 工具包 | 限制 |
|---------|-------|
| GTK4 | 边界是窗口坐标矩形转换到窗口的 X11 客户端原点：即其进程和标题的唯一客户端，或对于共享标题的两个窗口，窗口管理器为工具包报告的活动窗口所报告的活动客户端，以及另一窗口的另一客户端。窗口本身（在 `list-windows` 中及作为快照根）是该客户端绘制的不含阴影的矩形。其他情况下不报告位置：无头语义动作仍在其自身窗口中执行，需要指针位置的请求返回 `AMBIGUOUS_TARGET` 及 `details.kind: window_position_ambiguous`。工具包的 `[0,0,w,h]` 屏幕范围不确立位置。菜单项标签来自标注节点。秘密条目显示文本 |
| GTK3 | `scroll-to` 通过滚动条工作。持有的句柄在其对象销毁后 16 到 17 秒内返回过期值，因此等待在每次轮询时重新解析 |
| Qt | `scroll-to` 不工作：使用 `scroll --headed`。`QLineEdit` 上下文菜单不可观察。组合框弹窗不接受语义选择。Qt 进程注册两次；应用列表每个进程显示一个条目 |
| Chromium、Electron | `scroll-to` 通过 `ScrollTo` 工作。`hit_test` 对 Electron 宿主返回未知。Chromium 需要上述标志；Electron 不需要 |

GNOME 设置分类行不暴露语义激活动作，且列表有可访问索引看不到的隐藏行，因此对 `Settings categories` 的无头 `select` 会拒绝且不改变选择。在 X11 上，`--headed select` 点击匹配行，并在该行读取为选中时返回 `delivered_verified`；`details.semantic_refusal` 记录拒绝。也要验证窗格。不要用刷新后的 ref 重试无头 `select`。

搜索控件可能不暴露语义 SetValue 动作，且按 ref 寻址的 headed 输入可能拒绝 `target_focus`。通过新观察到的控件或应用文档化的快捷键打开搜索，然后重新快照。通过角色、可编辑能力和实时聚焦状态识别目标编辑器；翻译后的标签不是就绪条件。若工具包省略编辑器或无法证明焦点，在输入或粘贴前停止。一旦独立确立焦点，剪贴板粘贴会替换剪贴板：验证结果查询和结果。仅粘贴报告不能证明字段已接收。

`hit_test` 在 Wayland 上返回 `Unknown`，无窗口管理器时、GTK4 的 X11 映射仍模糊时，以及 Electron 宿主上也是如此。按 ref 寻址的物理点击和滚轮对未证明的命中结果返回 `ACTION_FAILED`、`details.kind: pointer_hit_unproven`。被遮挡的按钮仍为 `visible` 且可无头操作：遮挡仅在指针投递前检查。

同窗口兄弟重叠对物理输入返回 `ACTION_FAILED`、`details.kind: hit_test_unproven`、`reason: sibling_overlap` 和 `physical_delivery_started: false`。GTK3 和 Qt 可在其原生点查询中命名底层子项，但其子项顺序、组件层和 z 序无法确定哪个重叠兄弟接收输入。同样的保守检查适用于 GTK4。移除重叠，然后重新快照再重试。检查跟随实时子项列表；GTK4 的 `Parent` 属性可能与之不一致。缺少子项边界或分支不可读也会以未证明拒绝。仅靠快照无法修复重叠控件。

## 错误与 `platform_detail`

错误保持标准信封。Linux 用原始 D-Bus 或 X11 错误填充 `platform_detail`。

| `error.code` | 典型 `platform_detail` | 含义 |
|--------------|---------------------------|---------|
| `PLATFORM_NOT_SUPPORTED` | `org.freedesktop.DBus.Error.ServiceUnknown` | AT-SPI 未安装或不可见 |
| `PLATFORM_NOT_SUPPORTED` | `org.freedesktop.DBus.Error.Spawn.*` | 总线启动器无法启动 |
| `PLATFORM_NOT_SUPPORTED` | ``Failed to connect to address `unix:path=/run/user/1000/at-spi/bus_0,guid=...`: No such file or directory (os error 2)`` | 无障碍总线套接字缺失（建议：重启总线启动器）。相同文本但为会话总线地址表示无会话总线 |
| `TIMEOUT` | "The accessibility bus did not answer in time" | 总线挂起或缓慢：未获得信息 |
| `INTERNAL` | "The accessibility bus returned an unexpected error" | 未分类的总线回复：重试一次，然后报告 |
| `INVALID_ARGS` | "Native handle does not contain a Linux accessibility element" | ref 类型与命令接受的不符 |

## 按设计拒绝的命令

- `key-down` 和 `key-up`：`ACTION_NOT_SUPPORTED`、`delivery: unknown`。`mouse-down` 和 `mouse-up` 无头时为 `POLICY_DENIED`、`not_delivered`，带 `--headed` 时为 `ACTION_NOT_SUPPORTED`。不发送任何事件。使用带修饰键的 `press` 和 `drag`。
- Wayland 上的 `cursor-overlay`：仅作为设置成功。`enable` 返回 `data.rendered: false` 且不绘制覆盖层，因此不要等待。
- `open-system-surface --surface taskbar`：`INVALID_ARGS`。使用已公布的 GNOME 表面；参见 `references/shell-tray-and-notifications.md`。
- Wayland 上的普通应用指针命令和窗口截图。Shell ref 点击在 Wayland 上也拒绝，在 X11 上除非 shell 自身的树证明接收者。

## 缺失工具

X11 上二进制不需要外部工具。Wayland 剪贴板需要 `wl-clipboard`。X11 剪贴板需要二进制旁的辅助程序 `agent-desktop-linux-clipboard-owner`。总线需要 `at-spi2-core`。缺失工具会按名称及其包报告，绝不作为空结果。

## 共享主机

`TIMEOUT` 带 `lock_timeout` 和高 `contention_count` 表示同一用户的另一 agent 持有交互锁。不要重启用户的无障碍总线。仅在私有运行时目录中运行第二个总线（参见 `references/bus-and-permissions.md`）。