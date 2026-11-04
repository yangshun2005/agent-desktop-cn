---
title: Xcode 项目逐层探测与展开状态一致性
module: macos adapter
problem_type: runtime_error
tags: [xcode, accessibility, headless, disclosure, scoped-refs]
date: 2026-09-20
---

# Xcode 项目探测

用户在 Xcode 中打开了 cursor-cam，PID 24741，窗口 w-19740。本次探测使用公开的 Agent Desktop JSON 和模型选定的 ref，未解析包装层。未编辑项目源码，未请求构建/运行。原始 AX 仅用于独立的只读诊断。这些试验不构成重复验收的分母。

## 已复现并修复

候选版本 `clipping-final-candidate` 在导航器展开控件上声明支持 Expand/Collapse，但 `expand @s2j42fiokeiy6k:e44` 失败，返回 ACTION_FAILED、not_delivered："All chain steps exhausted"。原始 AX 显示 AXDisclosureTriangle 的 AXValue 为 0 或 1，且缺少 AXExpanded/AXDisclosing。适配器的动作校验器和快照状态读取器忽略了该值。

共享状态读取器现在仅对 AXDisclosureTriangle 解释 AXValue，且仅在缺少显式 expanded/disclosing 证据时使用。快照状态、实时状态完整性和展开动作校验复用该规则。未知/混合值仍视为未知；其他角色永不继承该规则。现有的单次投递语义链不变。未添加焦点或输入回退。测试覆盖 true/false、显式优先、缺失/混合值及其他角色（`tree/state_reader_tests.rs` 中的 `disclosure_value_is_role_scoped_and_explicit_state_wins`；规则为 `tree/state_reader.rs` 中的 `expanded_state`，由 `actions/chain/chain_disclosure_steps.rs` 中的 `disclosed_state` 复用）。

修复后的不可变候选版本：`/tmp/ad-native-eval/disclosure-candidate`，SHA-256 `1d68e0d6d5f71e9bcc046f19a3b2c22ee31a3bf7dc49b2f6bad121513783b39c`。原展开 ref 成功展开，get 报告已展开，同一 ref 成功折叠。已查看截图 `/tmp/ad-native-eval/xcode-disclosure-expanded.png`，显示 MenuBar 下的 MenuBarManager 和 SettingsPopoverManager。45 秒监控报告无激活、零输入增量、指针未变；前台 PID 35000 未变，Xcode 保持运行。

## 列与命令检查

- 骨架 `s1a9jwgx9ts97p` 提供导航器 e2、编辑器 e5、调试 e7 和检查器 e9。检查器、编辑器和调试的逐层探测在修复前候选版本上成功；导航器逐层探测已在 `s2j42fiokeiy6k:e2` 下成功。
- 检查器精确查找 textfield Tab 返回一个匹配，值为 4；同一查询以编辑器 e5 为根返回零匹配。逐层探测编辑器和调试兄弟节点后，检查器子节点 e44 仍返回 4。
- 在修复后的候选版本上，编辑器作用域的 Source Editor 查找返回一个匹配，调试作用域的 VariablesViewFilter 查找返回一个匹配。
- 重新逐层探测检查器 e9 替换了其子 ref。旧 e44 失败，返回 STALE_REF，处置为安全的 not-delivered；替换后的 e100 返回 4。未触碰的编辑器子节点 e53 仍可解析。未使用过期 ref 尝试任何变更。
- 检查器 Text Settings e96 用同一 ref 折叠并展开；get 报告中间的空状态。编辑器 scrollarea e52 用同一 ref 向下和向上滚动。两个命令均报告语义校验投递。查看 `/tmp/ad-native-eval/` 下的截图 `xcode-editor-scrolled.png` 和 `xcode-editor-restored.png`，分别显示源码视口移动和回到第 1 行。第二次 45 秒监控同样报告零激活/输入增量，指针/前台 PID 35000 未变。

这些是单项支持性试验，不是三十次运行的资格验证。旧 ref 在候选版本变更时刻意保留；仍需对新候选版本执行完整的基线/重置系列。

## 其他保留的发现

在 Project Navigator 上对 CursorCamApp.swift 执行 `select` 返回 ELEMENT_NOT_FOUND，尽管嵌套 textfield 暴露了该值。规范子标签读取静态文本，而此导航器暴露可编辑 textfield 值。不要盲目将稳定标识扩展到所有可变值，也不要点击可编辑标签。集合选择现在匹配实时非安全 AXTextField 值，并仅按下其所属的可选行（`actions/select/select_name.rs`）。

点击文件行报告校验选择并可见地选中了它，但编辑器仍停留在 CursorCamAppDelegate.swift。选择不等于打开文件。Xcode 全程显示 Preparing Editor Functionality。未尝试重复激活以强制结果。

将 Project navigator 过滤器设为 CursorCam 可见地过滤了树。清除它报告 delivered_unverified，因为空值缺失（已由 empty_text 读取器修复，2026-09-20-xcode-empty-search-value.md）；最终查看的截图显示原始未过滤树和空过滤器。较早的监控包含外部会话输入和多个应用的激活；无法归因，因此那些试验不是干净的无头资格验证。文件行选择仍为 CursorCamApp；编辑器仍为 CursorCamAppDelegate。未更改项目内容。

## 验证与限制

- 五个聚焦展开测试通过。完整 `cargo test --workspace` 在 macOS 访问权限下通过（`disclosure-workspace-unsandboxed.log`）。沙箱尝试在该次重跑前有两个 CLI 事件等待批处理测试失败；保留在 `disclosure-workspace.log` 而非删除。
- Clippy 警告视为错误、格式和 diff 检查通过。
- 自有安全语义夹具在精确候选哈希上通过全部七项检查（`disclosure-semantic.log`）。
- 只读三轮性能对比：`/tmp/ad-native-eval/disclosure-performance/report.html`。Xcode 候选/基线 p50 ms：交互 261.7/265.7，骨架 175.3/188.9，深度 30 278.8/258.5，首按钮查找 202.8/205.1。全部 12 次候选和基线调用成功，各自树形匹配。深度 30 慢 20.3 ms；此小样本无法在累积分支 diff 中隔离其原因。本次修复未增加额外的逐节点 AX 读取。
- SF Symbols 候选调用全部成功；基线首按钮查找三轮全部失败。深层部分树形不同（937 对 930），因此这不是干净的聚合性能通过或完整性等价声明。

无头 InteractionPolicy 已将焦点窃取和光标移动默认设为 false。本次修复保持该策略，不添加物理路径。这不能证明任意应用中的任意 AX 动作无法激活其应用。菜单、文件打开、语义拖拽和重复跨应用验收仍未完成。