# Windows 平台探测脚本（子阶段 2.0 及之后）

Windows 适配器的证据收集脚本。它们观察真实操作系统，写出有界的、已脱敏的捕获文件，供 `FINDINGS.md` 台账引用。这里没有产品代码，也没有工作区成员。

子阶段 2.0（`00-*.ps1` 到 `13-*.ps1`）仅使用 PowerShell。后续子阶段在以下情况扩展语料库：某个问题只能针对适配器实际发布的 crate 回答，或只能在本机不具备的运行器上回答：`14-ci-capability/` 增加一个独立的 Rust 探测程序，在工作区之外的临时目录中构建，由 `.github/workflows/windows-capability-probe.yml` 在 `windows-latest` 上运行。该工作流按路径过滤到探测目录及其自身文件，因此不会给必需流水线增加时间。

## 前置条件

- Windows 及 Windows PowerShell 5.1（不需要也不使用 `pwsh`）。
- .NET Framework 4.8 — 提供托管 UIA（`System.Windows.Automation`）以及内置编译器 `%WINDIR%\Microsoft.NET\Framework64\v4.0.30319\csc.exe`。
- Obsidian 安装在 `%LOCALAPPDATA%\Programs\Obsidian\Obsidian.exe` — Electron 探测目标。其版本在探测时读取并记录到台账。
- 交互式控制台会话。输入合成和命中测试探测需要真实桌面；在无头/断开连接的会话中无效。

## 运行

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File .\run-all.ps1
powershell -NoProfile -ExecutionPolicy Bypass -File .\run-all.ps1 -Compare
```

`run-all.ps1` 按字典序执行每个顶层 `NN-*.ps1`，各自在独立的 PowerShell 进程中运行，并为每个探测打印一行摘要。位于各自 `NN-*/` 目录中的子阶段探测**不在**该集合中 — 它们的脚本名为 `probe.ps1`/`census.ps1`，由 `.github/workflows/windows-capability-probe.yml` 驱动或手动运行。下面的脱敏门禁是例外：它也会扫描这些目录，因此"干净"意味着整个语料库中的每个捕获都干净。退出码：

- `0` — 每个探测都产生了捕获，脱敏门禁干净，没有残留的派生进程。
- `1` — 探测失败、捕获缺失、脱敏残留、进程存活，或（在 `-Compare` 模式下）规范化差异非空。
- `2` — 测试框架错误（缺少 `common.ps1`、捕获目录不可写、探测无法启动）。框架错误会中止运行；探测级失败会被记录，运行继续。

单个探测可独立运行：`powershell -NoProfile -File .\05-interactions.ps1`。

## 捕获布局

```
captures/
└── <probe-name>/
    ├── <capture>.json              # 已脱敏捕获，无 BOM UTF-8
    └── <capture>.json.normalized   # KTD9 规范化副本，写在捕获文件旁边
```

规范化副本将运行间变化的值规范化，使重跑差异为空：进程 ID、线程 ID、窗口句柄、UIA `RuntimeId` 数组、耗时/持续时间/计时数字、GUID、临时文件路径和时间戳。矩形分量（`X`/`Y`/`Left`/`Top`/`Right`/`Bottom`/`Width`/`Height` 及 `Bounds` 数组）舍入到 **8 像素桶**以吸收布局抖动。`run-all.ps1 -Compare` 从每个已提交捕获重新生成副本并报告任何差异 — 非空差异是真实的平台漂移，正是后续重跑者需要看到的。

## 脱敏门禁（R11）

每个捕获都通过 `Protect-ProbeText` 写出，它替换操作员的用户名、机器名和 DNS 主机名、用户配置文件路径（`%USERPROFILE%`、`%LOCALAPPDATA%`、`%APPDATA%`，以及任意斜杠形式的 `C:\Users\<name>`），以及 SID 子权威值为稳定占位符。`S-1-5-21-a-b-c-RID` 变为 `S-1-5-21-<redacted>-RID`：RID 是承重证据，机器唯一部分不是。无域部分的知名 SID（`S-1-5-18`、`S-1-5-32-544`、`S-1-16-12288` 等）原样通过 — 完整性级别 SID 正是多个探测的目的。

`Protect-ProbeName` 将文档/内容节点的 `Name` 缩减为 `<redacted:N chars>`。`ControlType`、`AutomationId`、`ClassName`、边界、模式和状态保持原样，应用外壳名称由调用方直接不调用它来保留。

**`Protect-ProbeName` 是调用点缩减器，不属于门禁，Rust 探测无法调用它。** `Protect-ProbeText` 和 `Test-CaptureRedaction` 检查操作员身份 — 用户名、机器名和 DNS 名、配置文件路径、SID — 仅此而已；两者都没有针对内容节点 `Name` 的规则，无论何种探测语言。读取内容 `Name` 的 `.ps1` 探测自行调用 `Protect-ProbeName`；Rust 探测输出自己的 JSON，编排器只将其传给 `Protect-ProbeText`，因此 **Rust 探测必须在记录点缩减内容 `Name`** — 长度、存在标志或稳定摘要 — 因为下游不会做，下游也不会察觉。`probe_survival.rs` 是示例：标记通过名称的 FNV-1a 摘要跨捕获配对，绝不通过名称本身。

没有绕过门禁的写入器。`run-all.ps1` 在退出前对每个捕获重新断言门禁，并指出任何违规文件 — 针对上述身份类别以及下面的回显名称残留，这就是此处"干净"的含义。

### 容器从自身内容继承的名称

没有自身标签的容器从下方内容获取可访问名称，因此文档标题会落到 `Group` 或 `Button` 上，任何内容控件类型测试都看不到，任何窗口标题比较也抓不到 — 泄漏的名称属于窗口标题未命名的某个工作区分割。在调用方声明携带用户内容的目标上（`-RedactRootName`），`01-tree-dump.ps1` 因此还会缩减任何 `Name`，只要它作为 >=4 字符子串包含此规则已缩减的后代 `Name`。

该范围正是此规则与证据破坏的分界，且是依据测量而非猜测选定的。若不限定范围，同一规则会缩减 Settings 框架的窗口标题，因为系统应用的标题和其主页标题都是 `Settings` 一词 — 它会在框架窗口和核心窗口上缩减，却让标题栏窗口原样保留，拆解了这三个节点存在所要展示的框架/宿主分割。携带内容的应用中的外壳因结构性原因而非运气得以保留：显式标签不欠其下方子树任何东西，而可点击图标的子树是未命名的 `Image`。Electron 捕获保留其所有外壳名称 — `Close`、`Bookmarks`、`New tab`、功能区操作、视图操作 — 原样。

`Test-CaptureNameEcho` 在已提交捕获上重新断言该规则，此时原始字符串已消失，只剩长度：在根 `Name` 本身已缩减的 pass 内 — 捕获自身关于目标被声明携带用户内容的记录 — 若某个保留的 `Name` 的子树持有足够短、能容纳于其中的已缩减 `Name`，则该保留 `Name` 是残留。长度包含弱于缩减器应用的字符串包含，因此门禁标记的是超集，不会漏掉缩减器本会捕获的内容。残留行只命名两个节点索引及其长度，绝不命名值。

**此处没有谓词能决定的事。** 此规则捕获容器从语料库已缩减内容中*回显*的标题。若容器将文档标题作为自身显式标签携带，下方没有已缩减内容可比较，也不匹配窗口标题，则任何结构测试都无法将其与外壳区分，会被原样发布。没有自动机制能关闭这一点，因此 Electron 目标的安全性由操作员决定，而非谓词。

> 已提交捕获会发布此 VM 的软件指纹：OS 构建号、区域设置、已安装应用版本、窗口类名和自动化 ID。这是刻意选择 — 这是一次性探测机，若脱敏到无法验证的程度，证据就毫无价值。不要在软件清单敏感机器上运行此语料库，并将 Electron 探测指向笔记标题不敏感的 vault：语料库会发布遍历该 vault 所持任何内容的树。

## 安全边界（KTD5）

交互、输入合成和提权探测只作用于探测自身启动的临时窗口。

- `Start-ScratchProcess` 将每个派生的 pid 记录到运行作用域台账（由 `run-all.ps1` 设置的 `AGENT_DESKTOP_PROBE_PIDS` 命名的临时文件）。
- `Stop-ScratchProcess` 终止进程，然后通过重新读取进程列表**确认**终止，若 pid 存活则抛出异常。
- `run-all.ps1` 在运行结束时若有任何被跟踪 pid 仍存活则非零退出。
- `Assert-Foreground` 必须包围每次 `SendInput` 调用。它读取 `GetForegroundWindow` + `GetWindowThreadProcessId`，当前台进程不是临时应用时抛出 `PROBE-INTERFERENCE:` 错误。捕获到它的探测会记录一行干扰台账 — 绝不重新注入。
- `Show-WindowNoActivate` 用 `ShowWindow(SW_SHOWNOACTIVATE)` 加可选的 `SetWindowPos(SWP_NOACTIVATE|SWP_NOZORDER)` 将临时窗口放到可见桌面。绝不调用 `SetForegroundWindow`。
- 剪贴板在任何组合键探测前快照，之后恢复。

### 中完整性进程

此机以内置 Administrator 在 High 完整性下运行且 AAM 关闭，因此 `Start-Process -Verb RunAs` 产生 High-vs-High，不产生完整性边界。`Start-MediumIntegrityProcess` 通过复制当前主令牌、用 `SetTokenInformation(TokenIntegrityLevel)` 将其强制标签降至 `S-1-16-8192`、并通过 `CreateProcessAsUser` 启动，制造真实的 Medium-IL 进程。然后用 `GetTokenInformation(TokenIntegrityLevel)` 读回派生进程的令牌标签，若不是恰好 `S-1-16-8192` 则抛出异常。没有真实边界时取得的 UIPI 观察是环境产物，绝不能进入产品契约，因此此断言不是可选的。

`runas.exe /trustlevel:0x20000`（`SAFER_LEVELID_NORMALUSER`）先被尝试，在此**不**充分：在此 VM 上测量，它限制令牌（BUILTIN\Administrators 变为仅拒绝），但将强制标签留在 `S-1-16-12288`（High）。这本身就是一个探测发现，不只是测试框架细节 — 基于 `runas /trustlevel` 构建的 UIPI 探测会测量 High-vs-High 并将其报告为完整性边界。

## 源编码（R12）

- **探测源（`.ps1`、`.cs`、`.md`）保存为带 BOM 的 UTF-8。** PowerShell 5.1 将无 BOM 文件解析为 ANSI，此 VM 代码页为 1252 — 非 ASCII 字面量在输入前就已损坏。
- **捕获写为无 BOM UTF-8**，仅通过 `Write-ProbeCapture` / `Write-ProbeJson`，它们使用
  `[IO.File]::WriteAllText($path, $text, (New-Object System.Text.UTF8Encoding $false))`。
  捕获绝不用 `Out-File`、`Set-Content` 或 `>`：5.1 默认 UTF-16LE，git 将其视为二进制，使已提交证据无法审查。
- 非 ASCII 输入载荷用 `[char]::ConvertFromUtf32(...)` 构建，使载荷完整性完全不依赖源编码。

用会剥离 BOM 的工具编辑后，重新保存带 BOM 的源：

```powershell
$p = '.\common.ps1'
$c = [IO.File]::ReadAllText($p)
[IO.File]::WriteAllText($p, $c, (New-Object System.Text.UTF8Encoding $true))
```

## C# 5 上限

内置编译器是 pre-Roslyn 的 .NET Framework 4.8 `csc.exe`（4.8.3761.0）。此语料库中每个 C# 源 — `csc.exe` 垫片、临时应用和 `Add-Type` 源 — 都限于 C# 5：无字符串插值（`$"..."`）、无 null 条件运算符（`?.`）、无表达式体成员、无 `nameof`。平实编写，处处可编译。

## `common.ps1` API

| 函数 | 用途 |
| --- | --- |
| `Get-ProbeRoot` | `probes/windows` 的绝对路径 |
| `Get-CaptureDir -Probe` | `captures/<Probe>`，不存在则创建 |
| `Protect-ProbeText -Text` | 对自由文本执行 R11 脱敏 |
| `Protect-ProbeName -Name` | `<redacted:N chars>` 内容节点缩减器 |
| `Write-ProbeCapture -Probe -Name -Content` | 已脱敏无 BOM UTF-8 捕获 + 规范化副本 |
| `Write-ProbeJson -Probe -Name -InputObject` | `ConvertTo-Json -Depth 25`，然后 `Write-ProbeCapture` |
| `Get-NormalizedCapture -Text` | KTD9 规范化 |
| `Test-CaptureRedaction -Path` | 干净时 `$true`；记录每个残留原因 |
| `Test-CaptureNameEcho -Path` | 无保留 `Name` 回显其下已缩减内容时 `$true` |
| `Read-ProbeNodeRecords -Path` | 捕获中的节点记录，按所属 pass 键控 |
| `ConvertFrom-ProbeJsonEscape -Text` | 撤销已提交捕获应用的 JSON 转义 |
| `Start-ScratchProcess -FilePath [-ArgumentList] [-NoActivate] [-TimeoutSec]` | 受跟踪启动 |
| `Stop-ScratchProcess -ProcessId` | 终止并确认消失 |
| `Get-ScratchProcessIds` | 仍存活的被跟踪 pid |
| `Assert-Foreground -ExpectedProcessId -Stage` | KTD5 注入守卫 |
| `Show-WindowNoActivate -WindowHandle [-X -Y -Width -Height]` | 放置而不抢焦点 |
| `Start-MediumIntegrityProcess -FilePath [-ArgumentList]` | Medium-IL 进程，标签已断言 |
| `Write-ProbeResult -Probe -Status -Message [-Data]` | `PROBE-RESULT\|...` 测试框架摘要行 |
| `Write-ProbeLog -Message [-Level]` | `PROBE-LOG [level] ...` |

探测脚本 dot-source `common.ps1`，在 `try` 中完成工作，在 `finally` 中拆除临时进程，写出至少一个捕获，并以恰好一个 `Write-ProbeResult` 结束，其状态为 `ok`、`fail` 或 `skip`。


## 阅读 Windows 实时测试套件（A28-6，A30-4 再次确认）



`cargo test -p agent-desktop-windows --lib` 驱动真实桌面，其

实时分支会布置窗口、菜单和 toast，存活时间超过创建它们的测试。

由此得出三条规则，搞反任何一条代价相同 — 真实回归被当作偶发忽略，或干净分支被一个偶发阻塞。



- **先让桌面静默。** 并发构建、探测进程和第二次测试运行都会争抢同一批窗口。

- **单次红色运行不是回归，单次绿色运行不是证明。** 相信任一之前先重跑。失败会在不相关模块间游走而非重复：落地此子阶段时它们出现在 `surface_inventory`，然后 `frame_identity`，然后 `notifications` 和 `tree::observe` — 五次运行中四组不同集合，每个单独运行都通过，同一棵树上套件两次绿色。

- **绝不要通过 stash A/B 将单测试差异归因于某变更。** 在此 stash 变更无论变更是否引起任何事都会读作单测试回归；A28-6 直接测量了这一点，此子阶段再次遇到，差点去追一个不存在的缺陷。



若运行在负载下失败，也检查磁盘余量：A28-6 记录满卷表现为游走的测试失败而非写入错误。