# Windows 平台探索发现（子阶段 2.0）

phases.md 2.0 要求的证据台账。每一行将一个实验映射到观察到的行为和一个文档对齐判定，每一行都注明产生它的客户端栈以及其结论所承载的范围。

请配合 `probes/windows/README.md`（如何运行语料库、脱敏门、安全边界）和 `docs/phases.md` 第 2 阶段（这些行所依据的契约）一起阅读。

## 如何阅读一行

| 列 | 含义 |
| --- | --- |
| `id` | 稳定的行 id，由底部的 hunk 索引引用 |
| `script` | 产生该行的探针，或——对于会话证据行——精确的重跑命令 |
| `stack` | `managed`（.NET `System.Windows.Automation`）、`uia3-com`（手工声明的 `CUIAutomation8` 垫片），或 `n/a`（工具链和会话证据行） |
| `scope` | `api-contract` = 在 KTD2 的前向兼容规则下适用于整个产品。`app/provider` = 特定于此环境和这些应用版本。下游子阶段读取此列以了解哪些结论可推广 |
| `verdict` | `CONFIRMS`、`CONTRADICTS`、`NEW-EDGE`、`DEFERRED`。没有 `UNKNOWN` 行（R5） |
| `action` | 该发现所要求的行动。`DEFERRED` 行带有 `closure: 2.<n>`——关闭它们的第 2 阶段子阶段——或 `closure: post-phase-2`，表示超出第 2 阶段范围的工作 |

KTD1 管辖栈的权威性：当 `managed` 行和 `uia3-com` 行不一致时，COM 行是与产品相关的，因为 Rust 适配器封装的是 UIA3 COM 客户端。重复 COM 测量事实的 managed 行在其自身捕获中被标记为非权威交叉检查。

KTD7 管辖对文档的权威性：探针行只有在 `scope` 为 `api-contract`，或其环境依赖在 `observed` 单元格中明确记录时，才能优先于 `docs/phases.md`。环境产物永远不会自我洗白为产品契约。

## 环境头

此台账中的每一行都在同一台机器上测量，完整记录于 `captures/00-environment/environment.json`（重跑：`powershell -NoProfile -ExecutionPolicy Bypass -File .\00-environment.ps1`）。

| 事实 | 值 |
| --- | --- |
| OS | Windows Server 2019 Datacenter，`InstallationType` Server，ReleaseId 1809 |
| 构建 | 17763，UBR 7434（`Microsoft Windows NT 10.0.17763.0`） |
| 会话 | `SESSIONNAME=Console`，`SessionId` 1，`UserInteractive` true，`isRemoteSession` false，窗口站 `Console`，单显示器 1639x732 |
| 完整性 | 强制标签 `S-1-16-12288`（High）；`EnableLUA` 1，`FilterAdministratorToken` 未设置，`ConsentPromptBehaviorAdmin` 5，因此管理员批准模式**未**激活，`-Verb RunAs` 产生 High-vs-High |
| 身份 | 用户 SID RID 500（内置 Administrator）；`TokenOwner` `S-1-5-32-544`；新文件通过 **TokenOwner 而非 TokenUser** 归属于 `S-1-5-32-544` |
| 区域设置 | 分裂：`InstalledUICulture` es-ES，`CurrentCulture` en-US，ANSI 代码页 1252，OEM 850。已知账户渲染为 `BUILTIN\Administradores`。没有探针匹配本地化的显示名称 |
| UIA | GAC `UIAutomationClient` 4.0.0.0 导出 57 个类型且**没有 `IUIAutomation`**；`CUIAutomation` 和 `CUIAutomation8` 都注册到 `C:\Windows\System32\uiautomationcore.dll` 7.2.17763.1，线程模型 Both |
| 工具链 | Windows PowerShell 5.1.17763.7434（`pwsh` 缺失）；`csc.exe` 4.8.3761.0，C# 5 上限；.NET Framework 4.8.03761 |
| 应用版本 | Obsidian 1.12.7.0 捆绑 Chromium 142.0.7444.265 和 Electron 39.8.3；经典 Win32 记事本（`%WINDIR%\System32\notepad.exe`），非 Store/MSIX RichEdit 变体 |

因为只有一个环境（KTD2），`scope` 列在做实际工作：`app/provider` 行陈述此 VM 和这些应用版本的行为，仅此而已。

## 区域 1 - 全树转储

| id | script | stack | scope | phases.md 期望 | 观察 | verdict | action |
| --- | --- | --- | --- | --- | --- | --- | --- |
| A1-1 | `01-tree-dump.ps1` | managed | app/provider | 2.0 区域 1 要求对记事本、资源管理器、设置和一个 Electron 应用进行全树转储，每个节点读取所有属性，并注明捕获的是哪个记事本变体 | 四次转储，ControlView，每个节点都携带 ControlType：记事本 3 个节点，100% 非空边界；设置 49 个，95.92%；资源管理器 196 个，100%；obsidian 119 个，99.16%。焦点身份（ControlType、ClassName、ProcessId、NativeWindowHandle）在所有七次遍历前后断言相同，零焦点干扰行。目标记录为经典 Win32 记事本 | CONFIRMS | 区域满足；记事本变体条款作为 phases.md 替换落地 |
| A1-2 | `01-tree-dump.ps1` | managed | app/provider | Windows 工程不变量 10：`ElementFromHandle` 对可见**和最小化**窗口是无头安全的 | 最小化经典记事本不改变树完整性：恢复 3 个节点，最小化 3 个，差值 0。它以两种不同形状退化几何：只有顶层 Window 报告 `Rect.Empty`，而其子代报告非空矩形，具有真实宽度和高度，锚定在 -32000 屏幕外原点。`IsOffscreen` 在最小化遍历的每个节点上都是 false，包括空矩形 Window | NEW-EDGE | 不变量 10 成立，无需更改。2.7 不得仅从 `IsOffscreen` 获取屏幕外证据，也不得将 `Rect.IsEmpty` 视为唯一的退化几何：-32000 矩形按任一属性既非空也非屏幕外 |
| A1-3 | `01-tree-dump.ps1` | managed | app/provider | 2.4 解析应用和窗口目标；2.0 区域 1 将设置命名为转储目标 | 设置解析为 `ApplicationFrameWindow`（pid 8712，`ApplicationFrameHost`），包含 `Windows.UI.Core.CoreWindow`（pid 9052，`SystemSettings`）。UWP 应用呈现的顶层窗口不属于应用 pid。选择仅按 ClassName 和 ProcessId；任何地方都没有比较本地化名称 | NEW-EDGE | 2.4 UWP 目标必须下降到 `CoreWindow`，而不是匹配顶层窗口的 ProcessId。区域设置规则是承重的而非装饰性的：此机器在 es-ES 安装上以英语渲染设置界面，因此标题匹配在这里会通过而在别处会失败 |
| A1-4 | `11-electron-activation.ps1` | uia3-com | app/provider | P2-O15：Chromium 138+ 向任何 UIA 客户端暴露 UIA 树，无需标志 | Obsidian 1.12.7.0 捆绑 Chromium 142.0.7444.265 和 Electron 39.8.3，版本从 `ELECTRON_RUN_AS_NODE` 和 DLL 文件版本都失败后对发布二进制进行用户代理扫描读取。每个栈×标志单元格一个全新实例：稳定 RawView 在所有四个单元格中都是 172，`managedActivatesWithoutFlag` 和 `comActivatesWithoutFlag` 都为 true，`clientStackDivergence` false | CONFIRMS | 计划的候选 CONTRADICTS 被**取代**：8 个稳定 managed 后代的预发现是通过被遮挡窗口和不受控的稳定窗口测量的。此声明不存在客户端栈分歧 |
| A1-5 | `11-electron-activation.ps1` | uia3-com | app/provider | 同样的行规定当树仍为空或极小时使用 `--force-renderer-accessibility`，无稳定要求 | 首次接触在两个栈上都是 13 个 RawView 节点，无标志，对比稳定的 172：13.2 倍的低估，跨运行确定性。使用标志时，首次接触是一场竞赛，落在从稳定计数的一小部分到几乎全部的任意位置，141 COM 和 164 managed。该标志对最终暴露是冗余的，对首次接触就绪**不是**冗余的；两个分支都不能消除稳定的需要 | CONTRADICTS | 从单次读取得出稀疏结论的指导为仅仅读取过早的树规定了该标志。已在目标、2.4 实现要点、文档检查清单和 R2 风险行中就地更正 |
| A1-6 | `01-tree-dump.ps1` | managed | app/provider | 无 - 这是探针方法学事实 | 早期修订版将记事本、设置和资源管理器恢复在 Obsidian 之上同一矩形，使 Obsidian 在完整 8 秒稳定和 16 秒仪器化保持中保持首次接触节点数，跨三次连续运行。最小化覆盖窗口修复了它。因果机制**未**被隔离：Chromium 原生窗口遮挡跟踪是明显候选，未经证实 | NEW-EDGE | 记录为探针放置危险，明确**不是**产品契约声明。后续测量另一窗口后 Electron 树的子阶段可能什么都没测到 |
| A1-7 | `03-pattern-census.ps1` | managed | app/provider | 无 - WPF 读取过早失败模式的首次观察 | 在其自动化对等体构建之前读取 WPF 窗口的 UIA 客户端**永久**绑定通用 HWND 提供程序：窗口报告 `ClassName HwndWrapper[appdomain;thread;guid]` 且零子代，30 秒进程内轮询包括 `FromHandle` 和强制 `FindAll(Descendants, TrueCondition)` 从未恢复它。同一夹具在启动后 8 秒读取立即报告 `ClassName=Window` 和 8 个子代 | NEW-EDGE | 启动 WPF 应用后立即拍摄的 Windows 快照可能返回永久空的一节点树，进程内重试无法修复——适配器必须从新 HWND 或新客户端重新解析。与 Electron 不同：Electron 低报后增长，WPF 绑定错误提供程序并保持错误 |
| A1-8 | `03-pattern-census.ps1` | managed | app/provider | 无 - A1-7 的推论 | `ClassName` 在同一 WPF 窗口上根据对等体绑定呈现两种形状，未绑定形状嵌入每次启动的 GUID | NEW-EDGE | 2.5 不得在 WPF 上将 `ClassName` 用作元素身份组件 |

## 区域 2 - 按 ControlType 的模式可用性普查

| id | script | stack | scope | phases.md 期望 | 观察 | verdict | action |
| --- | --- | --- | --- | --- | --- | --- | --- |
| A2-1 | `08-uia3-com.ps1` 然后 `03-pattern-census.ps1` | uia3-com | app/provider | 2.0 区域 2 要求按 ControlType 进行模式可用性普查，覆盖 Invoke、Toggle、Value、RangeValue、ExpandCollapse、SelectionItem、Scroll、ScrollItem、Text、Window 和 LegacyIAccessible | 4 个目标，141 个 RawView 元素，48 个 ControlType 行，无截断。观察到所有 11 个命名模式。运行时发现 35 个模式可用性属性；只有 16 个不同模式出现。每个目标上的健全性锚点成立：每个 Button 携带 Invoke，没有 Text 元素携带 Invoke | CONFIRMS | 区域满足。2.3 的角色和可操作性表由此矩阵构建 |
| A2-2 | `03-pattern-census.ps1` | uia3-com | app/provider | KTD1 的前提：managed 普查会记录虚假缺失 | `LegacyIAccessible` 由 141 个 COM 遍历元素中的 141 个通告，而 managed 扫描中为 0。排除它后，每目标模式覆盖率降至 11/35、37/46、33/34 和 18/26，因此它不是装饰性的 | CONFIRMS | 第 2 阶段中每个模式可用性行都必须携带 `stack=uia3-com`。从 managed 派生模式表决定可操作性的适配器会恰好低报那些 `LegacyIAccessible.DoDefaultAction` 是唯一可供性的遗留 Win32 表面 |
| A2-3 | `03-pattern-census.ps1` | uia3-com | api-contract | KTD1 预测一组 managed 栈结构上无法命名的 UIA3 独有模式 | 诚实的差异是**两个**模式，而非 KTD1 预期的六个：`LegacyIAccessible` 141 COM 对 0 managed，`Text2` 1 COM 对 0 managed。其他 14 个观察到的模式由两个栈都报告。Drag、DropTarget、Annotation、Styles 和 TextChild 从未出现在任何元素上，因此它们的 managed 缺失在此未测试而非已证明 | NEW-EDGE | KTD1 的结论建立在两个测量模式上。此子阶段早期报告的更大差异数字是 WPF 对等体竞赛产物（A1-7），已撤回 |
| A2-4 | `03-pattern-census.ps1` | uia3-com | app/provider | 2.3 将 UIA ControlType 整数映射到统一角色枚举 | 经典记事本的编辑表面对 COM 客户端是 `ControlType.Document`，携带 Value、Text、Text2、Scroll 和 LegacyIAccessible，由 `Microsoft: EDIT Proxy (unmanaged:uiautomationcore.dll)` 提供。同一表面对 managed 客户端是 `ControlType.Pane`，`ClassName Edit` 和 `AutomationId 15`。全窗口计数：26 个 COM 节点对 3 个 managed | NEW-EDGE | 2.3 的词汇表映射不能仅凭 ControlType 键控可编辑文本表面，Win32 目标上的低 managed 节点计数是客户端栈分歧的证据，而非缺失树的证据 |
| A2-5 | `08-uia3-com.ps1` | uia3-com | api-contract | 无 - 普查必须发现属性 id 才能运行 | 模式可用性属性 id 是构建特定的：`IsAnnotationPatternAvailable` 在此构建上是 30118，而 30113（记忆表会携带的值）是不同属性。从实时运行时枚举了 175 个属性和 35 个模式可用性属性 | NEW-EDGE | 2.2 和 2.3 必须从 UIA 运行时或 crate 生成的常量获取模式可用性属性 id，绝不从手写表获取。这是 Rust 模式可用性检查中最可能的单一静默失败 |
| A2-6 | `03-pattern-census.ps1` | managed | app/provider | 无 - 此语料库的内部一致性 | `divergence.json` 在散文中声明 managed 栈暴露 22 个模式类，而同一文件中机器生成的字段在运行时反映 `UIAutomationTypes` 中的 21 个并恰好列出 21 个名称 | NEW-EDGE | 浮出而非静默调和：反映的 21 是可信数字，散文的 22 是手写的。此台账中没有结论依赖于哪个正确 |

## 区域 3 - 每个交互原始演练

| id | script | stack | scope | phases.md 期望 | 观察 | verdict | action |
| --- | --- | --- | --- | --- | --- | --- | --- |
| A3-1 | `05-interactions.ps1` | managed | app/provider | 2.0 区域 3 要求 invoke、toggle、set value、select、expand 和 collapse、按模式和按滚轮滚动、text get、selection、caret 和 insert、以及 focus，全部原始演练 | 每一个都演练了。在 WPF 夹具上全部成功：Invoke 驱动状态接收器从 `status:ready` 到 `action:1`；Toggle Off 到 On；ValuePattern 往返 ASCII、CJK、4 个 UTF-16 单元、3 个码点和 1 个代理对的星体平面载荷，以及混合载荷，全部精确；ExpandCollapse 往返；SelectionItem 选择计数 0 到 1；ScrollPattern 垂直百分比 0 到 80。TextPattern 跨模式读取精确读回 ValuePattern 写入的字符串，选择 5 单元范围，并在偏移 4 处折叠退化 caret 范围。TextPattern 按契约只读，因此插入通过 `ValuePattern.SetValue` 完成并通过 TextPattern 读取验证 | CONFIRMS | 2.6 的动作集有根据。此捕获中的每个否定都归档为 verdict `pattern-unavailable`，绝不作为脚本错误 |
| A3-2 | `05-interactions.ps1` | managed | api-contract | 2.6 需要一个可观察量来证明动作落地 | `TogglePattern.Toggle` 翻转 `ToggleState` Off 到 On 而不引发 Click，因此夹具的 Click 驱动状态接收器保持在 `action:1` 而元素状态改变。后状态通过按 AutomationId 重新查找元素并重新获取模式读取，而非从调用返回的对象读取 | NEW-EDGE | 2.6