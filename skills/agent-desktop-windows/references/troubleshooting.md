# 故障排查

Windows 适配器的问题现象与原因对照表。每条先列可观察到的现象。

## 树为空或节点极少

| 现象 | 原因 | 修复 |
|---------|-------|-----|
| Chromium/Electron 应用返回的节点数远少于 UI 实际显示 | 首次读取发生在渲染器的无障碍树构建完成之前 | 运行 `snapshot --timeout-ms 30000`。冷启动时构建可能需要 10 到 25 秒。参见 `chromium-and-electron.md` |
| 无论等多久节点数都冻结在很小的数值 | 构建期间目标窗口被其他窗口完全遮挡 | 将目标窗口置于前台，或最小化遮挡窗口，然后重新快照 |
| WPF 应用：只有一个节点，`ClassName` 类似 `HwndWrapper[...]`，零子节点 | 在自动化对等体存在之前读取，永久绑定了通用 HWND 提供程序 | 重新快照。进程内重试永远无法恢复 |
| 自绘 UI 或游戏返回无意义内容 | 完全没有暴露无障碍信息 | 超出范围。仅用截图做验证 |

## PERM_DENIED 且详情为 E_ACCESSDENIED

```json
"code": "PERM_DENIED",
"platform_detail": "COM HRESULT 0x80070005 (E_ACCESSDENIED: Access is denied)"
```

原因：目标运行在严格更高的完整性级别，通常是从中完整性终端驱动的高权限应用。UIPI 会阻止跨该边界合成输入，但读取仍然成功。

修复：以与目标相同的权限级别运行终端。参见 `permissions-and-elevation.md`。

## COM 套间错误

```json
"message": "... COM multithreaded apartment ..."
```

原因：UIA 要求调用线程属于某个 COM 套间。CLI 自身会加入多线程套间，因此该错误意味着嵌入宿主的线程模型与之冲突。

修复：在 FFI 消费方中，通过库的引导入口点初始化，而不要从任意线程调用 UIA。错误的 `suggestion` 会提示加入多线程套间。缺陷在调用进程，不在目标应用。

## DPI 缩放坐标

现象：在缩放或多显示器设置下，有头坐标操作（`mouse-click --xy`、拖拽端点）落点偏离可见元素。

原因：没有逐显示器感知时，Windows 报告虚拟化坐标。适配器在启动时应用 `Per-Monitor V2`，因此边界和 `--xy` 值均为物理像素。

- 已设置不兼容感知上下文的宿主会被容忍（第二次调用返回 `ERROR_ACCESS_DENIED` 表示已设置）。
- 如果启动时报告 `INTERNAL` 并附带 `SetProcessDpiAwarenessContext` Win32 错误详情，说明宿主进程先锁定了上下文。请从普通终端运行，而非嵌入宿主。
- 优先使用 ref 而非原始坐标。

## 二进制无法启动

agent-desktop 在 Windows 上以未签名形式发布。三种执行控制对此处理不同：

- **SmartScreen：** 浏览器下载会携带 Mark-of-the-Web，双击可能显示"Windows 已保护你的电脑"。选择"更多信息"，然后"仍要运行"。npm 包捆绑了二进制文件，不下载任何内容，因此 npm 安装不会附加标记，也不会触发标记门控提示。
- **杀毒软件隔离：** Defender 或其他扫描器可能隔离新获得的未签名可执行文件，无论获取方式如何。验证发布的 SHA-256 校验和后，将其恢复或加入允许列表。
- **Smart App Control：** 在干净的 Windows 11 安装上，它会在进程创建时阻止未知的未签名可执行文件，包括命令行启动。任何下载路径或启动方式都无法绕过。请关闭 Smart App Control 或使用已签名构建。

## Shell 表面或通知命令返回 POLICY_DENIED

现象：`open-system-surface`、`list-notifications`、`wait --notification` 或通知变更操作返回 `POLICY_DENIED`，且未触及桌面。

原因：该命令会唤起 shell 界面元素，从而抢占前台，严格无头模式在任何内容被唤起之前就拒绝。

修复：传入全局 `--headed`，或先自行唤起该表面。`snapshot --surface <kind>` 可在无头模式下读取已存在的 shell 表面，`list-notifications` 和 `wait --notification` 会采用已打开的操作中心。

## wait --notification 在 Toast 发布期间超时

原因：操作中心仅在打开时收集 Toast，而等待在每次轮询时打开并关闭中心，因此轮询之间发布的 Toast 会丢失。

修复：在暂存窗口期间自行保持中心打开，使轮询采用它而不关闭它；或在中心打开时通过 `list-notifications` 确认 Toast。

## 托管（UWP）应用被读取为 ApplicationFrameHost

现象：`focused_window` 或 `list-windows` 将托管 UWP 应用（设置及其同类）报告为框架宿主的身份，或针对应用 pid 验证的身份停止解析。

原因：应用被挂起，其 `CoreWindow` 被丢弃而框架存活。参见 `shell-and-overlay.md` 中的"托管（UWP）窗口身份"。

修复：激活应用使其恢复，然后重新列出。

## INTERNAL："private file parent is owned by a foreign principal"

```json
"code": "INTERNAL",
"message": "private file parent is owned by a foreign principal, not this
process's token user, token owner, or an owner-eligible token group; if a
different principal owns this path intentionally, relocate the state root with
AGENT_DESKTOP_HOME instead of reusing it"
```

原因：令牌所有者不同的进程创建了状态根目录（`~/.agent-desktop`）。在 Windows 上，这几乎总是高权限（以管理员身份运行）shell 复用了非高权限会话创建的目录，或反之。所有权检查保护 refmap 和 trace 段免受预置路径攻击。

修复：在同一权限级别运行，或为高权限会话将 `AGENT_DESKTOP_HOME` 设置为新的绝对目录。不要通过夺取现有目录所有权或开放其 ACL 来绕过该错误，因为这会破坏检查。

## 操作验证

每个有状态操作（`set-value`、`clear`、`type`、`toggle`、`check`、`uncheck`、`expand`、`collapse`）**恰好执行一次**，然后重新读取实时状态。稳定循环每 50 ms 轮询一次，上限 400 ms，一旦后置条件成立即停止。它绝不会再次执行该操作。

三种结果不可互换：

- 回读匹配：`disposition.delivery` 为 `delivered_verified`。值写入还携带 `details.verification_scope: "element_value"` 和 `application_commit: "not_verified"`：控件持有你的值，但无法说明应用是否已提交。
- 回读与请求矛盾：`ACTION_FAILED`，附带 `details.kind: "post_action_verification"` 和 `details.post_state`。**先读 `post_state`。** 操作已送达，因此 `disposition.retry` 为 `unsafe`。
- 证据从未被观察到：操作成功为 `delivered_unverified`，附带 `details.verification_scope: "unavailable"`。这不是失败，也不是什么都没变的证明。元素缺少检查所需的 UIA 属性，例如 TogglePattern 或 ExpandCollapsePattern，或读取失败。**仅当聚焦字段暴露可读的 Text 模式选择时，`type` 才会被验证。** 没有时，`type` 落在此处。如果重要，用新的 `get` 确认值。

**被应用重塑的 `set-value` 会读为失败。** 如果应用修剪空白、重新格式化日期或规范化数字，回读与你写入的不同，信封会显示 `ACTION_FAILED`，尽管写入已落地。`SetValue` 检查不捕获前置状态，因此无法区分"什么都没发生"和"已接受并重塑"。将 `details.post_state.value` 与你发送的值比较：你的值以应用自身格式出现即表示写入成功。数值容差按角色限定：只有 `slider`、`incrementor`、`scrollbar` 和 `handle` 进行数值比较，因此你写入 `42` 后存储 `42.0` 的 `textfield` 会以此方式失败。

## 无键盘输入、无头保存文档

不存在 `save` 命令。保存是菜单路径加对话框，整条链是语义的，因此不需要 `--headed`，也不需要按键。同样的形态适用于任何保存流程为"文件"然后"另存为"的应用。

脚本按原样针对打开的记事本运行。每个 ref 来自前一步的 JSON，因此树变化时序列仍然正确。

```powershell
$out = "$env:TEMP\notepad-save-demo.txt"

agent-desktop restore --app notepad.exe                 # only if minimized; behind is fine
$doc = (agent-desktop find --app notepad.exe --role textfield | ConvertFrom-Json).data.matches[0]
agent-desktop set-value $doc.ref_id "the document body"

$file = (agent-desktop find --app notepad.exe --role menuitem --name "File" | ConvertFrom-Json).data.matches[0]
agent-desktop expand $file.ref_id
agent-desktop wait --menu --app notepad.exe             # the open menu is its own surface
$menu = (agent-desktop snapshot --app notepad.exe --surface menu | ConvertFrom-Json).data.tree
agent-desktop click ($menu.children | Where-Object name -eq 'Save As...').ref_id

agent-desktop wait --window "Save As" --app notepad.exe # the dialog is a new window
$dialog = (agent-desktop list-windows --app notepad.exe | ConvertFrom-Json).data |
    Where-Object title -eq 'Save As'
$name = (agent-desktop find --name "File name" --window-id $dialog.id | ConvertFrom-Json).data.matches |
    Where-Object role -eq 'textfield'
agent-desktop set-value $name.ref_id $out               # full path goes in the name field

$save = (agent-desktop find --role button --name "Save" --window-id $dialog.id | ConvertFrom-Json).data.matches[0]
agent-desktop click $save.ref_id
agent-desktop wait --window "notepad-save-demo - Notepad" --app notepad.exe
Get-Content $out
```

三件让首次调用者意外的事：

- **`find --name "File name"` 返回三个匹配：** 一个 `statictext` 标签、一个 `combobox`，以及 combobox 内的 `textfield`。只有 textfield 接受 `set-value`，因此要同时按 `role` 和名称匹配。
- **名称字段中的完整路径会选择目录。** 没有单独的文件夹步骤，按 ref 导航文件列表要脆弱得多。
- **点击保存返回时文件尚不存在。** shell 在点击信封与磁盘文件之间耗时 10.9 秒。用 `wait --window` 等待标题变化。固定睡眠一两秒会读为文件缺失。

对话框的保存按钮返回 `delivered_unverified`，因为合成的 invoke 无法确认 shell 对话框对其做了什么。通过从磁盘读回文件来验证。