# AgentDeskFixture (Windows)

一个由 Windows 实时 E2E 测试框架（`tests/e2e-windows/`）构建和控制的 WinForms 应用。它的存在是为了让测试框架能够驱动真实的 `agent-desktop.exe` 二进制文件操作真实窗口，并通过独立重新观察 fixture 来验证每个效果，而不是依赖命令自身的 `ok:true`。

## 构建

```powershell
.\build.ps1 [-OutputDir <path>]
```

`csc.exe` 通过绝对路径解析（`%WINDIR%\Microsoft.NET\Framework64\v4.0.30319\csc.exe`），从不经过 `PATH`，因此构建不会静默选中另一个内置编译器。`-OutputDir` 默认是本脚本旁的 `build` 目录，`tests/e2e-windows/Run-E2E.ps1` 会在该目录查找 `AgentDeskFixture.exe`，除非传入了 `-FixtureBinary`。

构建是幂等的：先编译到临时文件，只有编译完全成功后才替换之前的 `AgentDeskFixture.exe`。

## 工具链约束：仅限 C# 5

该路径下的内置编译器（`csc.exe`，版本 `4.8.3761.0`，在实测机器 A24-1 上为 "NET48REL1"）是 Roslyn 之前的旧版 .NET Framework 编译器。它最多接受 `/langversion:5`；`6` 及以上会报 `CS1617` 失败。因此本目录下每个 `.cs` 文件都必须避免字符串插值、表达式体成员、`nameof` 和 null 条件运算符。无论 `/langversion` 如何设置，这些构造在此工具链下都无法编译。

## 身份标识

每个交互控件的 `AutomationId` 来自 WinForms 的 `Control.Name`，通过 `FixtureIdentity.Assign` 赋值而非直接设置，因此 fixture 创建的每个控件都有据可查。不附带也不需要 `.exe.config`：在此工具链上 `Control.Name` 会直接呈现为 UIA `AutomationId`，无需 `Switch.UseLegacyAccessibilityFeatures` 覆盖。

`FixtureIdentity.WriteManifest` 在每次启动时将目前已赋值的所有 id（去重并排序）写入运行中 exe 旁的 `AgentDeskFixture.ids.txt`。测试框架的 Preconditions 场景会将此文件与 `tests/e2e-windows/fixture-inventory.psd1` 做双向集合相等检查，因此从 fixture 中移除控件却未同步从清单中移除（或反之）会导致运行失败，而不是静默缩小覆盖范围。

`WriteManifest` 恰好执行一次，在 `Main` 中主窗体构造函数返回后立即调用。因此每个带身份标识的控件都必须在那之前**构造**完成（从而完成 `Assign`），即使尚未**显示**。若某控件的构造被推迟到后续用户操作（延迟构建的模态对话框、上下文菜单项、按需打开的重复窗口），它将不会出现在清单中，尽管它一旦存在就带有真实可用的 `AutomationId`。应在窗体自身的构造函数中预先构建每张卡片的控件，之后再显示，而不是按需构造。

## 状态读数

`FixtureStatus` 是后续所有效果断言所依赖的唯一原语：一个只读 `TextBox`（`ControlType.Edit`，带 `ValuePattern`），其 `AutomationId` 是身份标识，其实时 `.Text` 是值。两者是独立的槽位，因此值变化绝不会被误认为身份变化。每个状态都从哨兵值 `idle` 开始；每次通过 `FixtureStatus.SetValue` 写入都会在赋值后追加一个单调递增的每状态计数器。对同一状态的任意两次写入都不会产生相同文本，因此任何环节都不会因前一环节遗留的值而通过。

通过 `agent-desktop get --property value` 读取状态，绝不要用 `--property text` 或 `--property name`。`value` 是带 `ValuePattern` 的 `Edit` 能应答而 `Label` 不能应答的属性。这一差异让读取本身就能区分正确实现与它绝不能回退到的单槽 `Label` 形态。

并非每个 `TextBox` 读数都经过 `FixtureStatus.SetValue` 的计数器。存在两种类型，场景作者在编写相等断言前必须分清：

- **计数器后缀标记**（`click-status`、`text-status`、`menu-status` 及其他所有 `*-status` id）：一个不透明的、单调编号的标记（`"clicked#1"`、`"changed#2"`），证明*某个*效果已落地，而非动作的字面内容。环节断言"与之前不同"，绝不断言特定字符串。
- **字面值读数**（`delayed-text`、`appeared-text`、`scroll-offset`）：通过 `FixtureStatus.Create` 构建以获得身份形态，但直接写入（`.Text = "ready"`），绕过计数器，因为确切字符串*正是*被测对象。自动等待环节轮询 `delayed-text` 的字面值 `ready`，而非"与之前不同的任意值"。

## 非 Control 身份不会呈现为 AutomationId

`Control.Name` 在无 `.exe.config` 时呈现为 UIA `AutomationId`（A24-2），但这是 `Control` 的属性，而非所有 WinForms 项类型的属性。针对本 fixture 运行中的构建实测（`agent-desktop find --native-id <id>`，以下每种情况均零匹配）：

- **`ToolStripItem.Name` 不会呈现为 `AutomationId`。** 一个为 `menu-disclosure` 携带 `Name` 的 `ToolStripSplitButton` 和一个为 `menu-fire-item` 携带 `Name` 的顶层 `ToolStripMenuItem` 均无法通过 `--native-id` 解析，尽管*容器* `ToolStrip`/`MenuStrip`（真正的 `Control`）能正常解析。
- **`TreeNode.Name` 同样不会呈现为 `AutomationId`**，原因相同：`TreeNode` 不是 `Control`。标准 `TreeView` 自身的 `AutomationId`（真正的 `Control`）能正常解析；其子节点则不能。

每个需要按原生 id 解析且需要标准控件无法产生的 `ControlType` 的目标，改为真正的 `Control`（`Button`、`Label` 或 `Panel`），并挂载 `WM_GETOBJECT` 覆盖提供程序（`FixtureProviderHost.cs` 中的 `UiaProviderHost` 及其子类、`FixtureToggleHost.cs`、`FixtureExpandCollapseHost.cs` 和 `FixtureControlTypeOverrideHost.cs`）：`menu-disclosure`、`menu-fire-item`、`outline-tree`、`outline-parent`、`outline-child-a`、`outline-child-b`、`cell-griditem`、`cell-custom`、`row-plain`。因此 `outline-tree` 不是真正的 `System.Windows.Forms.TreeView`：它是一个 `Panel`，内含三个按缩进排列的 `Label`，各自独立覆盖自己的 `ControlType`。

注册的自定义原始提供程序也**不会**像边界/启用/离屏那样通过 `HostRawElementProvider` 合并，从而透明继承宿主的 `InvokePattern` 可用性。针对 `menu-fire-item` 在无显式 `IInvokeProvider` 时实测，`InvokePattern.Invoke` 返回 `skipped`，点击链的 `LegacyIAccessible.DoDefaultAction` 回退报告成功，但 Click 处理程序从未运行。因此 `ControlTypeOverrideHost` 接受一个可选的 invoke 回调，并在被包装目标必须通过 CLI 保持可点击时自行实现 `IInvokeProvider`。

## 非激活启动

在启动环境中设置 `AGENT_DESKTOP_FIXTURE_NO_ACTIVATE=1`，主窗口将通过 `SW_SHOWNOACTIVATE` 显示而非抢占前台；`Run-E2E.ps1` 会设置它。未设置（或值不为 `1`）时，窗口正常激活。

## 进程生命周期

`Main()` 调用 `Application.Run(form)` 时传入主 `AgentDeskFixture` 实例而非无参重载，因此消息循环和进程会在该特定窗体关闭时退出，无论当时有多少次要窗口（重复标题目标、模态对话框、覆盖层）处于打开状态。