---
title: 引用的测量必须与其采集数据一致
date: 2026-08-09
category: best-practices
module: probes/windows/FINDINGS.md, crates/windows/src/system/window/window_activate.rs
problem_type: process_gap
component: tooling
symptoms:
  - "台账行的观测单元格所述内容与其所指的 JSON 采集数据相反。"
  - "已发布的文档注释引用探针行来支撑探针从未测量过的事实。"
  - "设计本身站得住脚，但其下的证据句是假的，下游每位读者都继承了这句假话。"
applies_when:
  - "根据采集数据编写探针台账行"
  - "在已发布源码或设计文档中引用探针行"
  - "审查一项以测量为依据的变更"
root_cause: process_gap
resolution_type: reread_the_artifact_not_the_expectation
severity: high
tags: [evidence, probes, ledger, citations, review, windows]
---

# 引用的测量必须与其采集数据一致

## 问题

探针行 `A21-6` 记录称，无争用的 `SetForegroundWindow` 每次都能在第一次尝试时成功。该行所指的采集数据就提交在该行旁边，记录的却是相反结果：第一次尝试未取得所有权，需要第二次尝试。

这句假话扩散开来。一份估算重试预算的设计文档重复了它，已发布常量上的文档注释也引用了该行 id。产品自身源码声称了一项测量，而其证据与之矛盾。没有任何门禁失败，因为没有门禁将台账行的文字与其所指的 JSON 进行比对。

## 根因

该行是依据促使编写探针的预期写成的，而非依据采集数据。设计并不依赖该答案，因为无论哪种情况重试预算为二都正确，所以错误的数字没有改变任何行为，也未引起注意。下游读者引用的是台账行，而非采集数据。这使该行成为无人重新推导的单点故障。

## 解决方案

在同一次变更中，依据采集数据修正该行，并修正所有重复它的位置。若证据仍支持设计，则保留设计：首次尝试成功率为零比完美成功率更有力地支持有界重试。

检查台账行时，读取它所指的字段，而非你预期采集数据应具有的字段：

```bash
python -c "import json,io; d=json.load(io.open('<capture>.json',encoding='utf-8-sig')); print(d)"
```

两个属性使台账行可审计：

- 引用采集数据所用的字段名和值。写 `first_attempt_success_rate: 0` 的行可与 JSON 比对；写“第一次尝试可靠”的行则不能。
- 记录探针写下的判定（`branch`、`measurable`），而非转述它。

## 预防

- 编写台账行时，把采集数据打开放在旁边，绝不要依据探针本要回答的问题来写。未重新阅读其采集数据而写成的行是未经验证的。
- 当发现正是作者期望的答案时，再读一遍采集数据。确认性结果比矛盾性结果受到的审视更少，这是本末倒置。
- 在已发布源码或设计文档中引用台账行之前，先打开采集数据。重复某行并不证明它为真。
- 修正台账行时，在同一次变更中修正所有下游重复。
- 设计正确但证据不支持其理由，仍然是没有辩护的。即使代码不变，也要修正那句话。
- 探针测量的二进制必须是产品运行的二进制。测量绝对路径的探针不覆盖按裸名称发起的调用，测试辅助程序必须使用与产品相同的解析方式。在 Windows 上，裸 `tar` 解析到了 Git for Windows 的 GNU tar，后者将盘符归档路径读作网络主机。
- 重复消除的是噪声，不是环境。N 次取最小值给出的是其运行机器上的稳定数字。单次未重复的计时样本不是证据。计时结论还必须在第二个环境（如 CI 运行器）上成立，才能作为事实写下。
- 报告分布，而非最佳情况。最小值回答的是“这能多快”，而非“这会花多少”。同时报告中位数，分布宽时报告最大值。
- 台账门禁 `probes/windows/13-ledger-check.ps1`（辅助脚本在 `probes/windows/13-ledger-content.ps1`，由 `.github/workflows/windows-capability-probe.yml` 运行）会使引用的 `field: value` 对或计数数字与其采集数据不一致的行失败。定性断言不在其覆盖范围内，因此上述规则仍约束作者。

## 相关

- [无法失败的测试不是测试覆盖率](a-test-that-cannot-fail-is-not-coverage.md)