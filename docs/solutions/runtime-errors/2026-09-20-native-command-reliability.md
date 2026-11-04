---
module: agent-desktop
date: 2026-09-20
problem_type: runtime_error
tags: [native, scrolling, refs, identity, headless, reliability]
---

# 原生命令可靠性：滚动、选择与菜单发现

该领域的试验仅限少数应用和运行次数，不能据此确立普遍可靠性。

## 滚动控件不能以其值作为标识

滚动条滚动时其值会变化，因此值不能作为稳定标识。Core 的可变值角色列表遗漏了 `scrollbar` 和 `handle`。旧逻辑把值当作标识，一次正常滚动就导致活动滚动条无法解析。修复在 `crates/core/src/tree/roles.rs` 的 `is_mutable_value_role` 中加入了这两个角色。未放宽任何超时、模糊匹配、路径信任或歧义规则。

仅凭值匹配仍不足以作为标识，不同的具名滚动控件仍应视为不同。几何变化但无稳定标识时，仍需安全拒绝。该修复不承诺恢复任意滑块。

证据：`crates/core/src/refs/ref_identity_scroll_tests.rs`，测试 `scroll_positions_neither_prove_nor_reject_element_identity`。

## Select 匹配规范集合标签

`select` 只搜索 AXTitle 和 AXDescription。快照从行的子内容推导行名，因此 `select` 对快照已命名的行返回 ELEMENT_NOT_FOUND。现在搜索读取与快照相同的有限名称证据。候选必须声明 Click，因为惰性文本不能仅因文本匹配就成为操作目标。证据未知时按失败关闭处理，条目缺失时报告 `not_delivered`。

菜单选择共用此搜索，尚无实时回归测试。

证据：`crates/macos/src/actions/select/select_name.rs`，测试 `selection_requires_a_known_canonical_label_and_an_actionable_target`。

## 不完整分支无法证明菜单不存在

应用级菜单搜索在第一个不完整分支处中止。在一个 6,984 项的网格中，某分支只加载了部分子项，掩盖了同级分支中打开的菜单。现在搜索改为广度优先，在检查同级分支时暂缓 `AppUnresponsive` 分支错误。观察到菜单则返回；否则保留不完整错误。权限错误和超时仍为终止性，2,048 节点上限仍然适用。

证据：`crates/macos/src/tree/surfaces.rs`，测试 `an_incomplete_branch_cannot_prove_menu_absence`。

## 展开状态不可读的展开控件

SF Symbols 中的一个 Weight 弹出按钮发布了 AXShowMenu 和 AXPress，但对 AXExpanded 和 AXDisclosing 返回 `kAXErrorAttributeUnsupported`。其展开操作拒绝且未交付。展开状态未知时，应测试原生菜单打开语义，而非盲目切换 AXPress。

## 受监控试验需要安静的前台

无头探测统计会话级输入并观察激活通知。用户或其他应用的活动会出现在这些计数中，因此事件无法归因于代理。存在无关前台活动的受监控试验不能证明无头成功，无法归因的事件不计入成绩。

证据：`tests/real-apps/headless_probe.swift`，只读诊断工具，不是驱动器。