---
title: Xcode 搜索清空成功但未暴露 AXValue
module: macos adapter
problem_type: runtime_error
tags: [xcode, value, verification, headless]
date: 2026-09-20
---

# 空搜索框的证据缺口

候选版本 `/tmp/ad-native-eval/descendant-stop-candidate`，SHA-256
`5f2bf71761548cd6cbb58b8161b0fbc2dc4bfee033f9d30fec603f93613a2115`。
Xcode PID 24741，窗口 w-19740。使用公开 CLI 并保留原始响应，命名空间
`/tmp/ad-native-eval/xcode-state`；未提供响应解析器生成的操作引用。

## 观察到的行为

调试过滤器 `@s287cvyqznmcqt:e75` 初始返回空字符串。对 ad_probe_93c217 设置值已验证，get 返回该确切字符串，查看截图显示其出现在调试过滤器中。对同一引用执行清空已验证，get 再次返回空字符串。

项目导航器过滤器 `@s287cvyqznmcqt:e3` 接受了 CursorCamAppDelegate。随后限定范围的导航器快照仅包含其父分组和匹配文件。查看截图确认实际列表已改变。对同一引用执行清空返回 delivered_unverified，原因为 insufficient_evidence。未重复写入。后续 get 返回 null，而限定范围快照显示完整的原始导航器树和原始选中行已恢复。最终截图也已查看，显示两个过滤器均为空，原始编辑器保留。

产物：
- `/tmp/ad-native-eval/xcode-debug-filter-filled.png`
- `/tmp/ad-native-eval/xcode-navigator-filter-filled.png`
- `/tmp/ad-native-eval/xcode-filter-restored.png`

60 秒监控记录到零激活、零输入事件增量、指针无变化；前台 PID 35000 保持不变。其时间区间覆盖了调试过滤器往返和导航器过滤。后续的恢复观察发生在完成之后；不要将监控结论扩展到这些观察。

## 原始 AX 诊断

只读有界辅助程序 `/tmp/ad-native-eval/xcode-filter-read.swift` 报告：

```
AXRole: success, AXTextField
AXSubrole: success, AXSearchField
AXDescription: success, Project navigator filter
AXValue: -25212, nil
AXNumberOfCharacters: success, 0
AXPlaceholderValue: success, Filter
```

这不是写入丢失。Xcode 对空搜索控件不暴露 AXValue，而 Agent Desktop 没有共享规则来使用原生零字符证据。该规则必须区分未知与空：仅限非安全文本角色，要求成功的数值零字符计数，且绝不从超时、不支持的计数、正计数或安全内容推断为空。快照、get 和操作后回读共享该规则，同时通过 node_attribute_fetch 的批量值路径和 text_attributes 的直接值路径；仅修改验证会导致读取结果矛盾。

## 文件激活线索已排除

原始 AX 行操作仍为 AXShowDefaultUI 和 AXShowAlternateUI。Apple 的
[showDefaultUI 文档](https://developer.apple.com/documentation/appkit/nsaccessibility-swift.struct/action/showdefaultui)
描述的是恢复原始/默认 UI，而非打开文件。未添加新的操作映射，也未尝试原始变更。文件打开问题仍未解决。

更正前次修复的影响范围说明：activate_descendant 被点击和语义切换/勾选路径使用。双击目前是物理操作，不使用该辅助程序。Xcode 项目中未编辑任何源文件。

## 修复

新增共享的 macOS empty_text 读取器，供批量节点水合和直接类型化值读取使用。仅非安全 AXTextField/AXTextArea 元素上缺失的值可触发它。成功的数值零 AXNumberOfCharacters 产生空字符串；正数、负数、小数、布尔、字符串、缺失和失败的计数均不能证明为空。真实读取错误会传播。现有截止时间仍约束额外读取；普通非空值不增加 IPC。批量指标记录回退读取。

负向测试发现了最初复用宽泛 ax_absence 分类器的问题，该分类器将通用 kAXErrorFailure 视为缺失。新的计数读取器仅允许 NoValue 和 AttributeUnsupported 作为缺失。失败的测试运行保留在 `/tmp/ad-native-eval/empty-text-workspace.log`。

第一个候选版本（4e34fdbe5b1e760d9db2f41da95dc4b477ca7f18be660bed1dc55211e3bf12ff）在原始过滤器引用上通过了实时 set/filter/clear/get 往返。清空从 delivered_unverified 变为 delivered_verified，get 返回空字符串。限定范围快照仍通过现有核心 observed_tree 投影省略空值；它不会暴露矛盾的非空值。查看 `/tmp/ad-native-eval/empty-text-restored.png` 显示完整的原始导航器和空过滤器。60 秒监控记录到零输入事件、零激活，指针/前台不变。最终候选检查遵循上述更严格的 native-error 分类变更。

最终候选版本 `/tmp/ad-native-eval/empty-text-final-candidate`，SHA-256
`6e9626edf059424869caba2519b8bbb2d749c7443512e5c8045ca9c8809789c1`。
分类器修正后，完整工作区、clippy、格式化和 diff 空白检查均通过。最终二进制对替换过滤器引用 e246 的 get 返回空字符串。最终三轮 merge-base 性能检查对所有命令均成功，报告的形状一致。候选/基线 p50 毫秒：交互式快照 300.5/315.2，骨架 201.2/205.2，深度 30 309.5/283.9，find 236.6/227.9。回退仅为符合条件的缺失文本值增加一次字符计数读取；在此小样本中深度 30 中位数高出 25.6 ms，候选 p95 314.4 对比基线 325.4 ms。这不是性能相等的统计声明。产物位于
`/tmp/ad-native-eval/empty-text-final-performance/`。

确切的最终二进制通过了全部七项后台语义夹具断言
（`/tmp/ad-native-eval/empty-text-final-semantic.log`）。这关闭了已诊断的空搜索值验证缺口；它不构成对文件打开或完整重复应用验收套件的合格证明。读取器位于
`crates/macos/src/tree/properties/empty_text.rs`；其单元测试固定了仅 NoValue 和
AttributeUnsupported 的缺失规则。