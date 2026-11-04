---
title: 顶层响应契约变更时递增 envelope 版本
date: 2026-05-13
category: best-practices
module: crates/core, src
problem_type: best_practice
component: output-contract
severity: high
applies_when:
  - 顶层 JSON 响应字段变更
  - 权限或状态 envelope 结构变更
  - 现有命令的错误码语义变更
  - 原本返回结构化错误的路径现在返回成功
  - 成功载荷新增调用方必须读取的始终存在字段
  - CLI 与批量输出契约统一
tags:
  - json-contract
  - cli
  - batch
  - compatibility
---

# 顶层响应契约变更时递增 envelope 版本

## 指导

`crates/core/src/response/output.rs` 中的 `ENVELOPE_VERSION` 是唯一事实来源。客户端靠 envelope 选择解析分支。若顶层字段或错误语义变了而版本不变，锁定版本的客户端无法区分新旧响应。

只要已能解析成功 JSON 的客户端对同一命令名需要换解析分支，就递增版本。包括：

- 顶层响应结构变更，以及权限、状态等始终存在字段的嵌套结构变更
- 原本成功的路径现在返回结构化错误
- 结构化错误变为成功，其条件需从字段读取。快照耗尽预算时返回 `ok: true` 和 `complete: false` 而非 `TIMEOUT`，靠错误码判断树过大的调用方会静默失效。`docs/json-output.md` 展示了此情况。
- 成功载荷新增必填字段，包括仅在某状态下有意义的始终存在字段（如 `complete`）
- 调用方合理依赖的错误码变更

新增可选字段、新命令、新标志或更具体的人类可读消息不递增版本。

## 影响范围

`ENVELOPE_VERSION` 持有该值，但值及其语义在非测试界面中被手工重述，且均非由该常量生成。同一次变更中逐一检查并修正错误之处：`docs/json-output.md`、`.github/ISSUE_TEMPLATE/bug_report.yml` 中的 JSON 占位符、`AGENTS.md` 中的 envelope 示例与错误说明，以及 `crates/ffi/src/commands/snapshot.rs` 中的 rustdoc 示例。头文件 `crates/ffi/include/agent_desktop.h` 是重新生成的，不手工编辑。

`src/cli/doc_literal_drift_tests.rs` 会在上述文件中的版本字面量与 `ENVELOPE_VERSION` 不一致时失败。它不检查字面量周围的文字，仍需人工核对。界面已更新为新版本号却仍在解释旧版本，正是本清单要捕获的失败。

## 发布说明

每次 envelope 递增都需在提交正文中加 `BREAKING CHANGE:` 脚注。不要在功能提交中手工编辑 `CHANGELOG.md`。发布 Pull Request（`docs/releasing.md`）负责撰写条目，因此脚注要足够精确以便据此撰写。

## 测试

通过 `ENVELOPE_VERSION` 断言版本，不要用字符串字面量，这样递增后 `main`、`batch`、FFI JSON 命令测试和单元级 envelope 测试能保持一致。