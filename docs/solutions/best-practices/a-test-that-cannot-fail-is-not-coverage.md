---
title: 无法失败的测试不是测试覆盖率
date: 2026-08-01
category: best-practices
module: crates/core/src/tree/accname.rs, crates/windows/src/tree, crates/windows/examples/uia_tree_dump
problem_type: process_gap
component: tooling
symptoms:
  - "一个回归测试断言两个入口点“一致”，而其中一个入口点的定义就是调用另一个。"
  - "一个 states 生产者的单元测试通过了某个角色/属性组合，但角色映射器永远不可能同时产生该组合。"
  - "一个安全脱敏测试位于 crates/windows/examples/ 下，而 CI 流水线运行的是 cargo test -p agent-desktop-windows --lib。"
  - "当某个调用点被改回逐字渲染文本时，普查工具的脱敏防护仍然通过，因为该防护测试的是辅助函数，而不是调用点。"
  - "第二份手工维护的列表（VALUE_BEARING）可能漏掉第一份列表（WALK_SET）新增的条目，而没有任何测试失败。"
  - "一个 #[cfg(test)] 辅助函数重新敲了一遍生产函数内联的同一个布尔表达式，因此反转生产代码后，镜像测试仍然通过。"
  - "一个实时测试以 `if !stage_precondition() { return; }` 开头，因此无法搭建前置条件的运行会通过，却什么也没断言。"
root_cause: process_gap
resolution_type: label_relocate_extract_seam_or_exhaustive_match
severity: high
tags: [testing, tautology, ci, exhaustiveness, com-free-seam, windows, coverage]
---

# 无法失败的测试不是测试覆盖率

## 问题

Windows 词汇表工作交付后，两轮评审加上实现者自己的一轮评审，发现了八种形态的测试，它们通过的原因与声称要证明的内容无关：

1. **同义反复。** `accname_tests.rs` 中有一个
   `the_uncertainty_aware_and_plain_entry_points_agree_on_every_input`，它
   断言 `resolve_name(&value, &NameSlotStatus::default()).known().cloned() ==
   compute_name(&value)`。`compute_name`（`accname.rs:84-88`）的定义正是这次调用，
   所以该断言就是 `x == x`，披着“一致性”的外衣。
   该测试在 `e549527c` 中被删除，该提交移除了重复更强测试的测试；`accname_tests.rs` 中不再有 `agree` 测试。
2. **不可能的输入。** `states_tests.rs` 向 states 生产者喂入
   `role == "button"` 且 `ToggleAvailable` 为 true。`roles.rs` 的 `button_role`
   会在 states 解析之前，把任何 toggle-available 的 `Button` 重新归类为 `Role::Switch`，
   因此到达生产者的 `button` 角色从未声明过 `ToggleAvailable`——被测分支在生产中不可能触发。
3. **运行器从不执行的门禁。** 普查脱敏防护——证明
   `Name`、`HelpText`、`FullDescription`、`LegacyDefaultAction` 永远不会
   序列化进已提交的捕获文件——位于
   `crates/windows/examples/uia_tree_dump/render_node_tests.rs`。Windows CI 流水线运行的是
   `cargo test -p agent-desktop-windows --lib`，它从不构建 `examples/`。
4. **防护与被防护对象隔了一层。** 即使修复了 #3，脱敏*规则*也只在
   `render_slots.rs` 中针对 `text_presence`/`field_presence` 本身做了固定。
   决定每个属性使用哪个渲染器的调用点——`render.rs` 中的 `node()`——只能通过实时
   `UIAElement` 到达。将一个调用点从 `text_presence(...)` 改回逐字的 `slot(...)`，
   仍能通过 `--lib`、`--examples` 和新的 CI 步骤。
5. **手工维护的平行列表。** `is_value_bearing()` 检查成员是否在
   `VALUE_BEARING` 数组中。没有任何机制强制该数组与遍历中实际携带目标文本的属性保持同步——
   向 `WALK_SET` 添加一个文本属性却没有对应的 `VALUE_BEARING` 条目，会静默泄漏。
6. **仅测试用的镜像。** `crates/windows/src/tree/hit_test/mod.rs` 的探测前
   防护阶梯直接内联在生产代码中，其 `#[cfg(test)]`
   覆盖（`guard_zero_area`、`guard_point_outside_bounds`、
   `guard_outside_virtual_screen`）重新敲了相同的布尔表达式，而不是调用它们。测试驱动的是自己的镜像，
   因此反转生产防护后，它们全都仍然通过。

7. **生产函数的 `#[cfg(test)]` 重新实现。** Windows 物理 `type` 分支防护了两件事：
   第一块之前的焦点检查，以及逐块验证器，它在块之间重新检查焦点，
   以免长时间输入继续进入抢走焦点的窗口。每个命名该分支的测试
   都调用 `type_text_from_gate`，它被声明为
   `#[cfg(test)]`，省略了策略门禁，省略了载荷预检，
   并在生产传入真实验证器的位置传入了 `|_| Ok(())`。
   从发布路径中删除任一防护，整个测试套件仍然通过。
   这比形态 6 更糟。镜像重新敲的是*表达式*；仅测试用的入口点重新敲的是*函数*，
   因此它会在所有维度上同时漂移，并以自己的名义冒充覆盖率。
   修复方法不是更好的测试，而是更好的接缝——让生产函数把变化的部分作为参数（这里是验证器），
   由测试提供自己的参数，这样只有一个门禁，测试驱动它。

8. **一个返回而非断言的搭建防护。** 四个实时
   `press_key_for_app` 测试以 `if !stage_as_foreground(&fixture) {
   return; }` 开头，其中搭建会执行真实的 OS 激活，而 Windows
   激活捕获（`A21-6`）记录它可能不会成功。在任何未成功的运行中，每个
   测试都在第一个断言之前返回，libtest 报告通过——而且
   四个测试中有一个是唯一证明该方法通过 trait 接通的测试。
   这是评审中最难发现的形态，因为提前返回读起来像防御性卫生，而不是漏洞：
   测试诚实地承认无法搭建前置条件，然后仍然静默地声称成功。
   修复方法不是删除防护，而是在其两侧都做断言——从未到达前台的目标必须以失败关闭且不合成任何内容，
   这是一个值得固定的真实契约，因此降级环境会行使一个属性，而不是跳过它。

## 根本原因

这些测试每一个都是照着实现已有的样子写的，
而不是照着测试要捕获的失败写的。

形态 1 把委托重述为断言——按构造为真，
委托可能重新引入的任何 bug 都不会改变它。形态 2 是孤立地针对 states 生产者写的，
没有检查角色映射器实际交给它什么，也从未注意到映射器早已堵上了这个缺口。
形态 3 和 4 是同一缺陷在不同距离上的表现：运行器从不构建的测试，以及能到达运行器但到不了真实调用点的测试，
都无法与其声称覆盖的代码路径相交。形态 5 是重复事实来源的 bug：
一个测试拿一份手工维护的列表去核对另一份，只能证明副本彼此匹配，永远不能证明任何一份是完整的。
形态 6 与形态 1 的委托伎俩相邻但更尖锐：测试根本没有调用生产表达式，
而是独立重新推导它，因此生产和测试可能静默分叉，而双方从未调用对方——
反转生产什么也证明不了，因为测试自己的镜像从未移动。

## 解决方案

这些修复没有一个是“删除测试”：

1. **移除它。** 委托防护没有被保留。`e549527c` 将其作为更强测试的重复而删除，
   而 `compute_name`（`accname.rs:84-88`）仍然委托，因此 `accname_tests.rs` 中的优先级测试
   自行承担覆盖率。
2. **覆盖可达形态，并将不可达形态保留为具名回归测试。**
   `button_role_with_toggle_state_but_no_pressed_bit_emits_no_pressed`
   保留，并附文档注释说明其输入不可达，并指向
   `toggle_state_on_a_switch_role_emits_checked_and_indeterminate`，后者
   以断言 `is_toggleable_role("switch")` 开头，因此可达情况先被固定。
3. **添加构建目标的流水线。** `.github/workflows/ci.yml` 新增了
   `Windows example tests` 步骤（`cargo test --locked -p agent-desktop-windows
   --examples`），并在 `src/cli/contract_tests.rs` 中固定为字面字符串，
   使工作流无法悄悄缩回 `--lib`。
4. **提取无 COM 接缝并测试接缝。** `render_node()` 移入
   自己的文件，接收普通 `NodeFields` 结构体而不是实时
   `UIAElement`。`render_node_tests.rs` 在每个
   文本承载字段中植入唯一标记，并断言它不出现在渲染后的 JSON 中，
   同时 `present`/`chars` 仍然报告——它现在驱动的是决定什么进入已提交捕获的确切函数，
   并且能对真实的调用点回退失败。
5. **用编译器强制的 match 替换平行列表。**
   `TreeProperty::carries_target_text()` 是一个没有兜底分支的穷尽 match——
   添加变体而不扩展它会是编译错误，而不是静默的 `false`。
   `property_ids_tests.rs` 断言它在整个枚举上蕴含
   `is_value_bearing()`，而不仅仅是 `WALK_SET`。
6. **提取一个纯函数，让双方驱动同一份副本。**
   `pre_probe_decision`（`crates/windows/src/tree/hit_test/hit_test_classify.rs:59-78`）
   现在是防护阶梯唯一表达的地方。生产代码从 `pre_probe_guard`
   （`crates/windows/src/tree/hit_test/mod.rs`）调用它；`hit_test_guard_tests.rs`
   直接调用它。生产和测试读取相同的逻辑，而不是两份独立维护的副本。

## 预防

能廉价捕获所有这些问题的唯一检查是**反转被测对象并确认测试变红**：
将委托回退为两个独立实现，向生产者喂入角色映射器永远不会产生的输入，
将 `text_presence` 调用点回退为 `slot()`，从穷尽 match 中删除一个枚举变体，
触发探测前防护阶梯的一个分支，并检查只有该分支自己的测试失败。如果没有任何东西变红，该测试就不是覆盖率。

形态 6 表明，仅靠反转仍然可能说谎。防护阶梯的第一次提取返回裸
`Option<HitTestResult>`，每次防护触发都产生相同的 `Some(Unknown)`。
在该形态下反转零面积分支仍然让一个测试变红——但原因错了：
几何落到了*另一个*防护，后者代替它回答了 `Some(Unknown)`，
因此所谓被测的分支从未真正被行使。可证伪性需要每个分支都能被单独归责的结果：
`PreProbeGuard`（`hit_test_classify.rs:40-45`）是四个具名变体——
`ZeroArea`、`IconicRoot`、`OutsideVirtualScreen`、`OutsideTargetBounds`——而
`hit_test_guard_tests.rs` 按名称断言每一个，因此被兄弟分支静默吸收的分支现在会产生错误的变体，
而不是其兄弟分支也会产生的相同 `Unknown`。

- 当多个分支坍缩为同一输出值时，“反转它并确认测试变红”是必要但不充分的：
  检查它变红是因为*该*分支自己的断言失败，而不是因为兄弟分支代替它捕获了该情况。
  给每个分支一个可区分的结果，否则反转会被兄弟分支吸收。
- 每次运行只反转**一个**生产位置。同时进行两个变异会混淆上一条所依赖的归因：
  同时破坏一个驻留防护和一个步进钳制，会让一个测试因*另一个*变异的算术在 `Duration` 内部 panic 而变红，
  并让本应失败的测试通过。每次反转都是独立的构建-运行-恢复循环，
  而一个什么都不变红的变异本身就是发现——它意味着该行是死代码。
  删除循环防护已经覆盖的零驻留提前返回，正是由此而来。
- 一个能在第一个断言之前 `return` 的测试，也需要在该路径上有断言。在测试套件中搜索测试体内的提前返回：
  每一个都是在声称当搭建不成立时没有什么值得检查，而这个声称通常是假的。
- 在信任“这两个一致”的测试之前，检查其中一方是否以另一方定义。
- 在信任 states/角色测试之前，检查角色映射器是否真的能产生被测组合。
- 无法到达生产调用点的测试，与运行器从不执行的测试是同一缺陷——检查 CI 流水线的标志，
  以及测试驱动的是被评审的函数，而不是它的邻居。
- 当某物必须被分类时，优先使用没有兜底分支的穷尽 match，而不是第二份手工维护的列表；
  编译器是一个永远不会过时的测试。