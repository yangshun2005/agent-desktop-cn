# Numbers 面向模型的可靠性评估

直接使用公开的 Agent Desktop CLI。模型必须自行读取原始 JSON 并选择 ref。不得在评估的交互流程中插入 Python/jq 树提取、自定义 ref 选择器、原始 AX 变更或 AppleScript。

所需的任务结果和门禁已冻结在 [acceptance v1](acceptance-v1.md) 中，该版本将 Numbers 用例（N1-N8）排除在活跃资格之外；下方的 Numbers 协议仍是可复现的运行方式。Fixture 测试提供确定性的回归，而非 Numbers 任务完成的证明。

## 归属与证据

使用不可变的发布二进制、隔离的绝对路径 `AGENT_DESKTOP_HOME`，以及名为 `AD Reliability.numbers` 的自有文档。记录二进制/差异哈希、macOS/Numbers 版本、种子内容、权限以及进程/窗口标识。切勿激活应用或触碰其他文档。保留原始命令输出；shell `tee` 可以接受，但为模型过滤响应则不行。

设置与独立验证独立于任务执行。准备 Sheet 1 / Eval Grid，B2=`seed-target`、C2=`neighbor-guard`、B3=17、C3=25，八行五列。对于重复表格用例，准备 Eval Other / Eval Grid，B2=`other-sheet-guard`。原生文档脚本可在不激活的情况下准备这些种子，但不能算作完成的 CLI 任务。异步设置后等待窗口就绪。

## 模型任务

1. 通过 `list-windows`、`snapshot --window-id ... --skeleton -i --compact` 和 `snapshot --root ...` 发现网格。从这些输出中选择 ref。
2. 选中 B2，发现编辑器，替换其内容，并以语义方式提交。独立验证确切的单元格和未改变的邻居。
3. 进入编辑模式后读取原始单元格 ref。分别记录自动恢复、陈旧拒绝或错误定位。
4. 按 ref 切换工作表。将同一 ref 上的 `get --property states`、`get --property value` 和 `is --property checked` 与实际活动工作表进行比较。
5. 使用原始 ref 将行数从八改为九再改回。将 CLI 读取与独立查询的文档维度进行比较。
6. 输入公式、保存、关闭，并通过受支持的语义命令重新打开。如果较早的步骤被阻塞，报告依赖任务未完成。不要通过脚本填充或保存并计为 Agent Desktop 成功。

保留每次尝试、预期/实际效果、原始输出、延迟和无头证据。区分任务失败、安全拒绝、原生限制、环境干扰和未运行的情况。后续成功不能抹去失败。仅端点指针相等不足以作为副作用证据。

## 独立原始 AX 诊断

`ax_probe.swift` 是诊断辅助工具，不是模型的驱动器。它通过 ApplicationServices 读取原生属性/动作/可设置标志，不涉及 Agent Desktop 内部。它需要进程 ID 和以 `AD Reliability` 开头的确切自有窗口标题。它还会列出参数化属性名称并读取 AXIdentifier 和 AXDOMIdentifier。在断定编辑器不暴露写入路径之前，先检查参数化属性。普通 setter 可能是空操作，而元素仍声明支持诸如 `AXReplaceRangeWithText` 的操作（Numbers 的单元格编辑器声明支持 AXStringForRange、AXAttributedStringForRange、AXLineForIndex、AXReplaceRangeWithText、AXBoundsForRange 和 AXRangeForLine）。

使用 `swiftc tests/real-apps/ax_probe.swift -o /tmp/ad-ax-probe` 编译。示例：

```
/tmp/ad-ax-probe PID 'AD Reliability.numbers' inspect 'seed-target'
/tmp/ad-ax-probe PID 'AD Reliability.numbers' inspect editor
```

将 `press`、`select` 和 `set` 保留用于在等效干净起始状态上的单独原始 AX 试验。切勿用它们来挽救 CLI 工作流或盲目重复不确定的操作。仅原生成功不足以证明效果。这些操作按确切标题或描述定位一个 `AXCell` 或 `AXMenuButton`；`select` 是 Numbers 特有的，会在标题包含 `Eval Grid` 的 `AXTable` 上写入 `AXSelectedCells`。

## 无需重写响应即可重复工作表切换

自行读取 `find --window-id ... --role radiobutton`，并从其原始响应中选择两个工作表 ref。在拥有文档且状态命名空间正确的情况下，运行：

```
bash tests/real-apps/numbers-sheets.sh /absolute/candidate w-123 @snapshot:e1 @snapshot:e2 5
```

参数为 `BINARY`（绝对路径）、`WINDOW_ID`、`SHEET_1_REF`、`OTHER_SHEET_REF` 和可选的 `CYCLES`（默认 5，最多 15）。环境必须将 `AGENT_DESKTOP_HOME` 设置为绝对路径。使用实际合格的 ref，而非上述占位符。以 Eval Other 为活动工作表开始，使第一次点击即为切换。脚本每周期复用相同 ref 进行两次切换，打印未更改的 JSON 并将其与截图一起保存在其公布的 `/tmp/ad-numbers-sheets.*` 证据目录下，捕获每次切换，并等待选中和未选中的值。任何命令失败都会停止运行；失败的点击会获得诊断截图，但绝不重试。它不执行设置、响应解析、应用激活或物理输入。

完成不等于通过评分：检查每一对 `get`/`is` 和截图，并独立验证活动工作表和未改变的守卫单元格。在验收批次之间重置并重新发现 ref。保留失败的尝试。运行器不验证持续的前台/指针稳定性或文档持久性。其失败停止检查为 `python3 tests/real-apps/numbers_sheets_test.py`。

## 参数化替换差分

`replace_contract_fixture.m` 和 `replace_range_probe.swift` 是独立的原始 AX 诊断工具，不是 Agent Desktop 验收驱动器。使用以下命令编译：

```
clang -framework AppKit tests/real-apps/replace_contract_fixture.m -o /tmp/ad-replace-fixture
swiftc -module-cache-path /tmp/ad-swift-cache tests/real-apps/replace_range_probe.swift -o /tmp/ad-replace-probe
```

运行 `/tmp/ad-replace-fixture` 以使用自定义接收器，或添加 `--standard` 以使用普通 NSTextView。`--standard-focused` 还会将文本视图指定为窗口的内部第一响应者，并记录输入上下文状态，而不激活应用或使窗口成为 key。这些模式会创建一个自有后台窗口并打印其 PID。
在单独的进程中，使用打印的 PID：

```
/tmp/ad-replace-probe PID 'AD Reliability Parameter Contract' seed-target replacement-token
```

探针要求确切的自有窗口和一个具有预期种子值的 AXTextArea，检查声明的支持，并拒绝前台目标。它分发一次，当 AX 失败或读回与请求的替换不同时退出 1。它不重试、不提交，也不证明持续的无头操作。测试后仅使用 `kill -TERM PID` 关闭打印的 fixture PID。成功的自定义接收器证明参数传输；它不证明 AppKit 的默认接收器或 Numbers 能在后台执行该操作。跨进程参数是一个字典：`AXReplacementRange`（以 UTF-16 单位表示的 AXValue CFRange）和 `AXReplacementText`（一个 CFString），由 AppKit 的 `_NSAccessibilityResultForReplaceRangeWithText` 读取。原生成功加 `true` 并不意味着文本已更改。普通后台 NSTextView（`--standard`）和 Numbers 都返回它们并保持值不变；只有自定义接收器会更改它。替换 `A😀B` 的代理对检查记录了 UTF-16 范围 `{0, 4}`。AppKit 的默认路径经过当前输入上下文。对于非 key 窗口，该上下文为 nil，即使文本视图成为第一响应者（`--standard-focused`）也是如此。没有生产写入路径使用此操作：未证明任何 Numbers 后台写入可行。

## 嵌套滚动回归

使用 `clang -framework AppKit tests/real-apps/nested_scroll_fixture.m -o /tmp/ad-nested-scroll-fixture` 编译 `nested_scroll_fixture.m` 并直接启动生成的可执行文件。它仅创建一个名为 AD Reliability Nested Scroll 的自有后台窗口。通过公开 CLI 发现该窗口和 Clipped 目标 ref。在发现的 ref 上执行 `scroll-to` 前后各捕获一张窗口截图。之前，按钮在其滚动视口之外但在其窗口之内；之后，按钮必须实际可见。仅返回 `verified: true` 或读取窗口相对离屏状态不是有效的判定依据：此回归防范的失败是已验证的跳过步骤没有移动任何东西。记录原始命令输出并检查两张截图。试验后仅使用 `kill -TERM PID` 关闭 fixture 打印的 PID。

## 额外的 Xcode 覆盖

第二应用探针仅使用后台 Xcode 中自有的 `AD Reliability.swift` 文件（原始 AX 辅助工具拒绝标题不以 `AD Reliability` 开头的窗口）。通过 skeleton/root 发现 Source Editor，替换其值，在同一 ref 上插入文本，并验证截图和确切的磁盘内容。通过原始 Close ref 关闭并作为单独设置重新打开以检查持久性；不要将设置计为 CLI 任务完成。将拒绝的旧窗口 ref 和原生菜单失败与成功编辑一并保留。

原始 AX 辅助工具可按确切标题或描述检查控件，并在自有的 AXMenuButton 上调用 `press` 或 `show-menu` 以进行单独诊断试验。它记录前台端点，而非持续监控。不确定的原生返回绝不授权回退操作或自动重放。

## 在试验期间观察无头副作用

使用 `swiftc tests/real-apps/headless_probe.swift -o /tmp/ad-headless-probe` 编译，然后在单独的终端/进程中运行 `/tmp/ad-headless-probe PID 60`（运行目标 PID，然后是最多 300 秒的持续时间）。仅在其原始 JSON 报告 `event: ready` 后启动评估命令。它观察工作区激活通知、会话级输入计数器和指针采样，直到有界持续时间到期。它不执行任何操作，也不记录按键内容。`/tmp/ad-headless-probe --self-test` 验证计数器算术，包括回绕。

将其最终 JSON 与操作输出一并检查。目标激活会使无头声明无效。其他应用激活和输入增量是归因未知的环境活动，既不能证明代理输入，也不能证明输入隔离干净。指针采样可能错过采样之间的移动并返回。未知前台（`-1`）、缺失指针采样或目标已退出均不能通过干净的监控门禁。此观察器改善证据，但不是自动评分。

## 针对 fixture 的保留身份

`retained-identity-probe.swift` 是用于 SwiftUI fixture 的 ref 变动视图（`tests/fixture-app/RefChurnView.swift`）的只读原始 AX 观察器。使用 `swiftc tests/real-apps/retained-identity-probe.swift -o /tmp/ad-retained-probe` 编译，并针对运行中的 `AgentDeskFixture` 运行 `/tmp/ad-retained-probe PID`。它要求恰好一个窗口，其中有一个读取 `Alpha` 的 `shared-title` 字段和一个 `stable-editor`，保留原生 `Alpha` 元素，打印 `ready`，然后从 stdin 读取阶段标签。在 `initial`、`filtered`、`restored`、`edited` 和 `cleared` 每个阶段之后，它报告保留的元素是否仍是活动窗口树的成员，并断言它仅在 `initial` 时位于树中；`quit` 退出。在阶段之间使用原始 Agent Desktop 命令驱动 fixture；探针从不修改任何内容。

## 选择传递 fixture

`selection_delivery_fixture.swift` 创建一个名为 `AD Reliability Selection Delivery` 的自有后台窗口。使用 `swiftc tests/real-apps/selection_delivery_fixture.swift -o /tmp/ad-selection-fixture` 编译并直接运行；它会记录 `READY pid=<PID>`。其行接受直接选中状态写入，但始终读回未选中，其表格容器独立接受选中行写入。每次写入记录为 `MUTATION direct-selection <bool>` 或 `MUTATION container-selection <count>`，因此试验可以显示对行的 `click` 走了哪条传递路径，以及是否尝试了回退。仅使用 `kill -TERM PID` 关闭打印的 PID。