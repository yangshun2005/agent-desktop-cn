---
module: agent-desktop-macos
date: 2026-09-20
problem_type: runtime_error
tags: [visibility, clipping, command-consistency]
---

# 观察命令间共享的裁剪状态

此前快照遍历对每个后代都复用窗口边界。实时状态观察和 ref 根观察也只使用所属窗口。因此，被更小滚动区域裁剪的原生按钮即使单独修复了 scroll-to，仍缺少离屏状态。

更早之前，元素根观察用根元素自身的边界作为视口来初始化构建上下文。一个零尺寸的 SF Symbols 按钮被自身裁剪，导致 find 报告其离屏，而读取 AXWindow 的 get-states 返回空列表。问题出在观察上下文，而非元素变化或 ref 过期。所属窗口查找现在位于 `tree/element_bounds.rs`。它不会回退到 AXTopLevelUIElement：对于无窗口的菜单，该值解析为菜单栏，即一条 29 点的条带，会把每个打开的菜单项都报告为离屏。AXWindow 读取失败仍会向上传播。

遍历现在沿每个分支向下传递有效裁剪矩形，并在访问兄弟节点前恢复先前的矩形，包括错误路径。Ref 根观察和实时观察在现有截止时间内读取裁剪祖先。矩形求交和裁剪角色定义与指针命中测试实现共享。完整遍历未增加逐节点祖先 IPC。空交集仍视为已知为空。未知祖先几何不会仅因更深层容器有边界而重新变为已知。

自有 fixture 现在包含一个位于滚动区域外的可见兄弟节点。候选 9dedd81c6561a354a6550b328877a623f73f019306cbfdb33f62a1ea7d06c6c9 通过 snapshot、get、find 和根快照一致报告为离屏；兄弟节点保持可见。在同一未变的目标/ref 上，前一个 nested-final 候选返回了空状态列表。语义 scroll-to 后，同一目标 ref 变为可见，兄弟节点保持不变，已由检查的 `/tmp/ad-native-eval/clipping-after.png` 确认。观察器记录到无目标激活但有无关输入，因此该次试验不是干净的无头合格通过。

最终候选 `/tmp/ad-native-eval/clipping-final-candidate`，哈希 f0ed9f26f9bf74b4d75e4004d3d773df91772e12301cd44c2013ae06fafadd5d，增加了未知祖先保留保护。工作区测试、clippy 和格式检查均通过。一个先前保存的匿名 scrollarea ref 超时，带有安全的未投递 STALE_REF 证据；它未被静默计为恢复。新的快照 ref 成功重置了 fixture 视口。保存的按钮 ref 和新的精确 find 随后在最终候选上均报告离屏。最终语义试验 1 在所有权检查点经过五轮后停止：fixture 窗口身份发生变化。原因未确定，该尝试保留为失败。另一次全新 fixture 试验在同一最终哈希上通过了全部七项断言。最终性能测试完成：候选 SF Symbols 观察成功 12/12；基线首按钮 find 失败 3/3。快照交互中位数为 3063.9/3064.5 ms，骨架 305.8/292.9 ms，深度 30 为 3062.1/3072.1 ms，候选 find 为 4099.2 ms。部分深度 30 形状不同；未声称完全等价或干净的聚合基线通过。

测试覆盖非裁剪组、不相交嵌套视口和未知几何（`tree/element_bounds.rs` 中的 `only_clipping_containers_narrow_the_viewport`、`disjoint_viewports_remain_known_empty_through_descendants`、`missing_clipping_geometry_is_unknown_not_empty` 和 `a_window_less_element_reports_no_clipping_viewport`）。较早的候选通过了七项语义断言和 SF Symbols 只读性能观察。Xcode 缺失并被跳过，而非通过。Numbers 按用户明确指示排除在主动测试之外。这些修复未确立重复的真实应用验收。

## 最终候选上的真实应用 S3 基线

SF Symbols PID 57617，窗口 w-17404。模型从限定 find 中选择了 outline @s1pw1hb4whof0b:e1，从精确 find 中选择了 Math @swkwt9hq8o9kg:e1。在同一 outline 上依次选择 Math 和 All，两次转换均返回 delivered_verified。保存的 Math/All ref 按预期报告为选中；返回 All 后 Math 变为未选中。检查的截图 `/tmp/ad-native-eval/s3-final-math.png` 和 `s3-final-all.png` 显示了预期的 78 和 6,984 个符号集合。六十秒观察器报告无应用激活、零输入事件增量和无指针移动，目标仍在运行且前台 PID 860 未变。结果：一次成功的 S3 基线往返，而非三个重置批次共三十次运行的要求。

最终候选上的 S4 尝试 1 使用了保存的离屏 All ref @ssvl7e1srna75:e4。Scroll-to 返回 delivered_verified，get-states 返回 selected 且无 offscreen。请求的截图因 TIMEOUT 失败：NSWorkspace 应用清单超时。三十秒监视器在其接近结束时记录到输入活动和目标激活。归因未知。该尝试保留为不合格/受污染且视觉验证不完整，而非通过；其处于前台期间 SF Symbols 变更停止。

随后 Xcode 设置启动了 PID 24741，未带激活标志，并使用后台设置仅打开了自有 AD Reliability.swift 文件。List-windows 将自有文件暴露为 w-19712，另有一个独立的 Welcome 窗口。限定到 w-19712 的精确 find 返回 Source Editor @s1j4qlrp4icwd:e1，带有预期的受保护源码。设置观察器记录到 Xcode 激活和输入活动；其原因未归因。发现已在功能上观察到，但未声称干净的背景设置或 X2 编辑通过。Xcode 处于前台期间文档编辑被暂停。