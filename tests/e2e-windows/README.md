# Windows 实时 E2E 测试框架

PowerShell 5.1，不依赖 Python，不依赖 POSIX shell。驱动真实的、已暂存的、带哈希校验的 `agent-desktop.exe`，针对真实的 WinForms 测试夹具（`tests/fixture-app-windows`），并通过独立重新观测来断言每个效果，绝不依赖命令自身的 `ok:true`。

## 模块

| 文件 | 职责 |
|---|---|
| `NativeTypes.psm1` | Win32 P/Invoke 类型接口（结构体、DllImport 声明），通过 `Add-Type` 一次性加载。 |
| `Native.psm1` | 基于 `NativeTypes.psm1` 构建的文件/句柄/作业对象/令牌 SID 原语。 |
| `NativeDesktop.psm1` | 原始前台窗口/光标/GUI 线程读取，同样基于 `NativeTypes.psm1` 构建。 |
| `BoundedProcess.psm1` | `Invoke-BoundedProcess` —— 受作业对象约束的子进程启动，带墙钟截止时间和捕获输出的字节上限。 |
| `Harness.psm1` | 隔离环境（`HOME`/`TEMP`/`CARGO_TARGET_DIR` 重定向）、可恢复删除、不可变二进制暂存 + 哈希重新校验、夹具进程身份，以及 `ConvertFrom-AgentJson`。 |
| `DesktopLease.psm1` | 桌面独占租约及其 CI 采纳协议，以及两个受保护的启动入口（`Invoke-Guarded`、`Invoke-GuardedAgent`）。 |
| `Lib.psm1` | 始终携带其快照的目标对象（`Find-Target`、`Get-Target`、`Test-Target`、`Wait-Target`、`Invoke-Target`）、独立观测断言（`Assert-Effect`、`Assert-NoEffect`、`Assert-Envelope`）、锁排序（`Enter-Stage`），以及判定/跳过账本（`Register-Legs`、`Add-Pass`/`Add-Fail`/`Add-Skip`、`Write-Verdict`）。 |
| `NativeToken.psm1` | 进程令牌和完整性级别读取。 |
| `StagedProcess.psm1` | 暂存进程启动和自身句柄报告辅助函数。 |
| `StagedProcessHandleStability.psm1` | 暂存进程的句柄计数稳定性检查。 |
| `LibEnvelope.psm1` | 由 `Lib.psm1` 重新导出的信封、批量和遮挡物断言。 |
| `LibVerdict.psm1` | 锁排序、leg 账本和 `Require-Target`，由 `Lib.psm1` 重新导出。 |
| `LibShell.psm1` | shell 场景使用的 shell 表面辅助函数。 |
| `ChromiumNative.psm1` | Chromium 场景的原生读取。 |
| `ChromiumStage.psm1` | Chromium 内容暂存。 |
| `CursorOverlaySupport.psm1` | 光标叠加场景的支持辅助函数。 |
| `Run-E2E.ps1` | 整个目录树中唯一终止进程的调用。场景文件（`scenarios/*.ps1`）返回；只有此脚本退出。 |
| `scenarios/*.ps1` | 每个文件一个场景，由 `Run-E2E.ps1` 按以下顺序 dot-source 并运行：Preconditions、Observation、NonInterference、Interaction、Acceptance、Reliability、Surfaces、ShellSurfaces、TracePerformance、Chromium、SplitIntegrity、CursorOverlay。不会自动发现：`Run-E2E.ps1` 未 dot-source 且未在 `Invoke-ScenarioSequence` 中列出的场景文件永远不会运行。 |
| `AcceptanceContendedFocus.ps1`、`InteractionHeaded.ps1`、`InteractionPress.ps1`、`InteractionScroll.ps1` | 由 `scenarios/Acceptance.ps1` 和 `scenarios/Interaction.ps1` dot-source 的 leg。它们位于 `scenarios/` 之外，因为契约门禁将那里的每个文件视为顶层场景入口点。 |
| `selftest/` | 测试框架自身的自测（见运行）。 |

每个文件都保持在 `scripts/check-e2e-windows-contract.ps1` 强制执行的 400 行上限以内（规则 13），因此大型模块和自测按职责拆分：每个 `selftest/Invoke-*SelfTests.ps1` 入口点负责进程退出，并 dot-source 其 `*SelfTestCases*.ps1` 文件。

## 数据文件

- `skip-allowlist.psd1` —— 场景可传给 `Add-Skip` 的每个能力令牌，附带原因。未声明的令牌会导致运行失败（跳过绝不能读作通过）。
- `latency-baseline.psd1` —— 已提交的 p99 存储。`Initialize-NoEffectWindow` 将固定的 `Assert-NoEffect` 窗口与其比对：当该窗口低于最大记录 p99 的 1.5 倍时，运行拒绝启动。它包含四个取自 `probes/windows/24-fixture-e2e/captures/cost-baseline-devbox.json` 的整 CLI 成本条目；尚未测量任何 per-leg 条目，且 `BootstrapP99Ms` 仅在 `Legs` 为空时适用。
- `fixture-inventory.psd1` —— 夹具必须暴露的 `Resident` 和 `Staged` `AutomationId`。Preconditions 场景将其与夹具写入的清单（`tests/fixture-app-windows/README.md`）进行双向比对。

## 运行

```powershell
# Lib.psm1 自测（目标、断言、锁排序、判定账本），针对
# Stub-AgentDesktop.ps1 运行：无桌面、无夹具、无暂存二进制。
# CI 通过 scripts\check-e2e-windows-contract.ps1 -SelfTest 运行它们。
powershell -NoProfile -File tests\e2e-windows\selftest\Invoke-U7SelfTests.ps1

# 测试框架核心自测（隔离、不可变暂存、有界启动、租约
# 采纳）：需要真实桌面和已构建的 target\release\agent-desktop.exe
# （或 -AgentDesktopBinary <path>）；会获取真实桌面租约，因此
# 无法在托管运行器上运行。
powershell -NoProfile -File tests\e2e-windows\selftest\Invoke-U6SelfTests.ps1

# 种子失败入口路径（证明失败的 leg 会到达非零
# 进程退出码，在每个 PR 的托管 CI 通道上运行）：
powershell -NoProfile -File tests\e2e-windows\Run-E2E.ps1 -SelfTestSeedFailure

# 实时套件（仅本地机器 —— 获取桌面独占租约）。
# 先构建两个产物；-AgentDesktopBinary 和 -FixtureBinary 覆盖
# 默认路径 target\release\agent-desktop.exe 和
# tests\fixture-app-windows\build\AgentDeskFixture.exe。
cargo build --locked --release -p agent-desktop
powershell -NoProfile -File tests\fixture-app-windows\build.ps1
powershell -NoProfile -File tests\e2e-windows\Run-E2E.ps1
```

`scripts\run-windows-e2e-ci.ps1` 是操作员入口点，在单个桌面租约下完成上述所有工作：构建 release 二进制和夹具，运行光标叠加拆除测试和帧预算、测试框架核心自测，然后运行 `Run-E2E.ps1`。它拒绝在非 Windows 上运行，或缺少 `AGENT_DESKTOP_NATIVE_E2E_RUNNER=1` 时运行。没有 CI 工作流运行该套件；`ci.yml` 中的拒绝自测仅运行它以证明该守卫以退出码 2 退出。

## 场景文件必须遵循的约定

- 每个 ref 都是来自 `Find-Target`/`Find-TargetById` 的目标对象，同时携带 `RefId` 和 `SnapshotId`。没有辅助函数接受或返回裸 `@eN`。
- 效果断言通过 `Assert-Effect`/`Assert-NoEffect`；信封断言通过 `Assert-Envelope`。只有 `Lib.psm1` 和 `LibEnvelope.psm1` 可以对命令结果使用点号访问 `.ok`/`.error`/`.disposition`/`.data`（规则 5 仅门禁点号成员访问；方括号索引不受门禁）。
- 每个触及桌面的 leg 都在词法上包裹在 `Enter-Stage` 中，按固定顺序声明锁 `DesktopLease` -> `ForegroundStage` -> `MenuStage`。
- 每个场景都预先用其 leg 名称调用 `Register-Legs`；每个注册的 leg 在运行结束时都有一个处置（`Add-Pass`/`Add-Fail`/`Add-Skip`）。