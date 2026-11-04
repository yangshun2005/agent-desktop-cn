# 权限与提权

Windows 没有 TCC 式的 UI Automation 权限授予机制。控制访问的是**完整性级别**：Medium 完整性级别的 agent 可以观察并操作同完整性级别的应用，跨边界向上的操作由操作系统把关。

## 同完整性目标无需授权

- UIA 读取（snapshot、find、get、is、screenshot）和同完整性输入在任何终端都能工作，无需权限提示。
- `agent-desktop permissions` 实时探测 UIA，而非查询策略存储：UIA 有响应时 `accessibility` 报告 `granted`，`automation` 报告 `not_required`，`screen_recording` 反映截图是否可用。
- 没有 `--request` 对话框可触发；若 `accessibility` 报告 `denied`，原因是下文的完整性或策略边界，而非缺少勾选项。

## UIPI 边界

用户界面特权隔离（UIPI）阻止低完整性进程向高完整性进程注入合成输入（Medium 到 High，例如提权后的 Notepad）：

- **读取可跨边界。** 从 Medium 级别，UIA 解析提权目标的名称、类名、控件类型、pid、边界和节点数，与 High 调用方结果相同，`WM_GETTEXT` 读取也成功。跨边界的观察从不会被拒绝。
- **写入不行。** 适配器在合成任何内容前比较两个进程令牌的完整性级别，目标严格更高时返回 `PERM_DENIED`，其 `platform_detail` 为 `COM HRESULT 0x80070005 (E_ACCESSDENIED: Access is denied)`。
- **修复：** 从与目标同级别的提权终端运行 agent-desktop，或操作不跨边界的元素。

### 检测靠读取令牌，绝不靠 SendInput 的返回值

`SendInput` 的返回值证明不了任何事：无论事件是否落地，它都报告成功。只有上述令牌比较，或重新读取效果，才能区分已送达与静默丢弃。进程无法读取其完整性级别的目标，不假定为同级或更低：操作按尽力而为执行，并通过重新读取效果来判定。

## 窗口激活可跨边界

`focus-window` 对更高完整性目标也能成功。`SetForegroundWindow` 受 Windows 前台锁定启发式（近期输入、`AllowSetForegroundWindow`）限制，而非完整性。预算耗尽前无法确认前台时，严格更高的目标返回带激活措辞的 `PERM_DENIED`，同完整性目标返回 `ACTION_FAILED`。

## 被阻止的按键组合

`press` 在合成前拒绝危险快捷键。所有修饰键顺序、键别名（`meta`/`cmd`/`super` 映射为 `win`）及修饰键超集都会被捕获：

| 被阻止 | 原因 |
|---------|-----|
| `alt+f4` | 关闭活动窗口 |
| `win+l` | 锁定会话 |
| `win+d` | 显示桌面 |
| `alt+tab` | 运行中抢占前台 |
| `win+r` | 在目标上方打开“运行” |
| `win+x` | 在目标上方打开快速链接菜单 |
| `win+e` | 在目标上方打开文件资源管理器 |
| `ctrl+shift+esc` | 在目标上方打开任务管理器 |

`alt+shift+tab` 等超集也会被阻止。`--force` 可覆盖此防护。`ctrl+alt+delete` 不在列表中，因为 `SendInput` 根本无法合成安全注意序列。

## 受保护的进程

`close-app` 按精确、不区分大小写的 `.exe` 名称拒绝会话和 shell 关键映像：`csrss.exe`、`wininit.exe`、`winlogon.exe`、`services.exe`、`lsass.exe`、`smss.exe`、`lsaiso.exe`、`dwm.exe`、`explorer.exe`。其他名称正常关闭，包括 `explorer++.exe` 这类近似名。拒绝时返回 `INVALID_ARGS`。

## 跨进程交互租约

交互式工作持有一个独占的、由令牌派生的锁文件，因此两个并发的 agent-desktop 进程无法在同一桌面上交错焦点切换、点击和输入。继承句柄模式（`AGENT_DESKTOP_INTERACTION_LEASE_HANDLE`）允许父进程在重新验证身份和独占性后，将租约传给子进程。若获取失败，等待另一进程完成。不要绕过租约。

## 光标叠加层的控制管道

光标叠加层是一个分离的渲染进程，通过命名管道驱动。

**受保护的部分：**

- 拒绝远程客户端（`PIPE_REJECT_REMOTE_CLIENTS`）。
- 双向拒绝其他用户的进程：每个连接的令牌用户 SID 必须匹配。
- 客户端在发送控制前，检查管道服务器的可执行文件名为 `agent-desktop`（不区分大小写，带不带 `.exe` 均可）。这是文件名检查，不是签名检查。不匹配时 `enable` 报告 `rendered: false`。
- 每个会话一个渲染器（`FILE_FLAG_FIRST_PIPE_INSTANCE`）。第二个渲染器在创建窗口前退出。

**不受保护的部分：同机器上的同用户进程。** 管道没有安全描述符，其名称源自状态根、会话 id 和协议代次。同用户进程可以：

- 先占用名称，在此期间 `enable` 返回 `rendered: false`（叠加层缺失，绝不会是伪造的）；
- 连接到活跃渲染器，移动绘制的光标或向标签卡片写入任意文本。

叠加层仅用于呈现。它从不执行或导致操作失败，也不携带凭据。在他人以你的账户运行进程的主机上，不要信任叠加层绘制的内容，并保持其禁用。