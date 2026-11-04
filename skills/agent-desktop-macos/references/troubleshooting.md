# macOS 故障排查

每条包含症状、原因和修复方法。先读取 JSON 信封中的 `error.code`、`error.suggestion` 和 `error.disposition`。

## PERM_DENIED

**症状。** 命令返回 `PERM_DENIED`，或 `permissions` 显示 `denied`。

**原因。** 启动 agent-desktop 的应用没有辅助功能权限（`screenshot` 和 `session start --screenshots` 还需要屏幕录制权限）。授权属于启动应用，不属于 shell。新的启动器（如其他终端或 IDE）需要单独授权。

**修复。** 按 macOS 技能中的"首次接触"操作：运行 `permissions --guide`，在系统设置 > 隐私与安全性 > 辅助功能或屏幕录制中开启启动应用（如果 `agent-desktop` 二进制单独列出，也一并开启），然后运行 `agent-desktop permissions` 确认 `granted`。启动应用无需重启。

指南会提供 agent-desktop 上方最近进程的应用包。在 tmux 服务器或 SSH 会话下不存在此类应用，因此指南提供 `agent-desktop` 二进制。此时需手动添加启动 tmux 的终端应用。macOS 27 及更高版本将辅助功能列表命名为"设备控制与数据访问"。

对于自动化，打开隐私与安全性 > 自动化，允许启动应用控制 System Events。

## 树为空或稀疏

**症状。** `snapshot` 返回少量节点，或无 ref。

**原因。** 部分应用暴露的辅助功能数据很少：自定义渲染的 UI、游戏、部分创意工具以及 Web 视图的部分区域。Chromium 和 Electron 应用需要在渲染器中启用 Web 辅助功能。

**修复。**
1. 先检查权限（见 PERM_DENIED）。
2. 去掉 `-i` 查看非交互节点。提高 `--max-depth`。
3. 运行 `list-surfaces --app "App"`。内容可能位于表单、弹出框或菜单界面中。
4. 重试一次快照。启用辅助功能后渲染器需要短暂稳定时间。
5. 用 `screenshot` 作为视觉后备（需要屏幕录制权限）。
6. 对于新的 Chromium 应用，使用 `launch "App" --cdp` 和 CDP 客户端访问 Web 内容。

## 快照缓慢或被截断

**症状。** 快照缓慢，或返回 `"complete": false` 和 `"truncated": true`。

**原因。** 大型应用（Xcode、打开多个标签页的 Safari）树很深。被截断的快照耗尽了观察预算。

**修复。** 知道目标时用 `find`。否则用 `snapshot --skeleton -i --compact`，再用 `snapshot --root @<snapshot_id>:eN` 下钻。加 `--window-id` 限定窗口。

## ACTION_FAILED

**症状。** ref 操作返回 `ACTION_FAILED`。

**原因。** 元素已禁用、应用繁忙，或元素不支持该操作。语义链只尝试元素声明的操作，然后停止。

**修复。**
1. 运行 `is @ref --property enabled`。若为 false，等待或修复状态。
2. 运行 `get @ref --property bounds`，确认元素在屏幕上。
3. 若物理输入适合该任务，加 `--headed` 重试。
4. 最后手段用键盘：`focus @ref`，然后 `press return`。

## POLICY_DENIED

**症状。** 命令返回 `POLICY_DENIED`。

**原因。** 语义路径无法完成，且命令以无头模式运行导致物理路径被阻止。悬停、拖拽、双击、三击和移动光标的鼠标命令始终需要 `--headed`。坐标 `right-click` 在无头模式下也被阻止。

**修复。** 仅当确实需要物理输入时，加全局 `--headed` 标志。示例：`agent-desktop --headed mouse-click --xy 640,400 --button right`。坐标从 `get @ref --property bounds` 获取。

## 右键后 APP_UNRESPONSIVE

**症状。** `right-click` 返回 `APP_UNRESPONSIVE`，带 `delivery_uncertain` 和 `retry: unsafe`。

**原因。** `AXShowMenu` 可能进入模态菜单跟踪，并在菜单打开后返回 `kAXErrorCannotComplete`。

**修复。** 不要重试。运行 `snapshot --app "App" --surface menu -i`。若菜单存在则使用它。否则运行 `list-surfaces --app "App"`。对于组合框或菜单按钮，用 `select @ref "Option"`。

## select 找不到菜单

**症状。** `select` 无法确认选择。

**原因。** `select` 通过菜单从关闭到打开的变化来证明成功。已打开的菜单会隐藏该变化。

**修复。** 用 `press escape` 关闭已打开的菜单，然后再次运行 `select`。

## APP_NOT_FOUND 或 WINDOW_NOT_FOUND

**修复。** 运行 `agent-desktop launch "App Name"`（显示名称或 bundle ID）。然后运行 `list-windows --app "App"`。全屏应用仍会出现在 `list-windows` 中，但其边界等于屏幕，状态可能滞后于动画，因此用 `wait`。

## 通知中心

| 症状 | 原因 | 修复 |
|---|---|---|
| 列出时 `POLICY_DENIED`：中心已关闭 | 命令为无头模式，无法打开中心 | 加 `--headed`，或自行打开中心 |
| `dismiss-notification` 返回 `ACTION_FAILED` | 该条目无可用关闭路径 | 见下方链路，然后重新列出 |
| `NOTIFICATION_NOT_FOUND` | 关闭后索引已移动 | 再次运行 `list-notifications` |
| 列出前横幅已消失 | 横幅是临时的 | 用 `wait --notification --app "App"` |
| 已关闭的条目被另一条替换 | 同一应用的通知会堆叠 | 再次关闭。`dismiss-all-notifications` 可能需要多轮 |
| 日历小组件仍在 | 它是系统小组件，不是通知 | 预期行为 |

关闭链路按顺序执行：`AXDismiss` 和 `AXRemoveFromParent` 操作，然后是名为 close、clear 或 dismiss 的关闭按钮，再是悬停显示隐藏的关闭按钮并按下。从不使用 `AXPress`，因为它会打开源应用而不会关闭通知。

## 键盘快捷键无效

**原因。** 快捷键用了 `ctrl`，而 macOS 用 `cmd`；或按键发到了错误的应用。

**修复。** 使用 `cmd+...`。要在不聚焦某应用的情况下向其发送按键，用 `press cmd+s --app "App"`。