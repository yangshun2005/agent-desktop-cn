---
title: 可读的 Chromium AX 树不能证明其操作已激活
date: 2026-09-19
category: runtime-errors
module: macos renderer accessibility activation
problem_type: integration_issue
component: platform_adapter
severity: high
tags: [macos, chromium, accessibility, headless, renderer-activation]
---

# 可读的 Chromium AX 树不能证明其操作已激活

## 机制

读取 AX 角色时，Chromium 会以受限的无障碍模式启动。其 `AXEnhancedUserInterface` setter 会在两秒防抖后请求完整模式。将标志读回为 true 并不代表防抖已结束，因此可用状态可能在写入约两秒后才到达。

setter 应用标志后会返回 `kAXErrorNotImplemented`。Chromium 在将 setter 转发给父类之前先处理该属性。因此，读回是确认变更已生效的唯一可靠信号。

Electron 的 `AXManualAccessibility` getter 只比较是否为完整模式，而其 setter 会附加平台和屏幕阅读器标志。已应用的写入可能读回 false。

主要来源：

- [Chromium 应用无障碍激活与两秒防抖](https://chromium.googlesource.com/chromium/src/+/refs/heads/main/chrome/browser/chrome_browser_application_mac.mm)
- [Chromium 激活回归测试](https://chromium.googlesource.com/chromium/src/+/refs/tags/133.0.6917.1/chrome/browser/chrome_browser_application_mac_browsertest.mm)
- [Chromium 异步无障碍操作语义](https://chromium.googlesource.com/chromium/src/+/main/docs/accessibility/browser/how_a11y_works_3.md)

## 修复

修复复用了现有的渲染器激活信号、截止时间和观察重试。策略与点击实现保持不变。

- 即使 Web 内容已可读，也要检查增强模式。可读的树不能证明其操作可用。
- 写入后读回标志。对于增强模式，匹配的读回可在返回码为不支持的情况下确立激活；矛盾的读回则不能。
- 在现有截止时间内等待两秒防抖结束。写入前拒绝过小的预算。已启用的模式无需承担此延迟。
- 当旧版手动模式渲染器的树可读但其手动标志没有可读值时，保持其正常工作。

该修复不添加浏览器名称检查、DOM 自动化或物理点击回退。点击仍可能返回 `delivered_unverified`，因此调用方必须检查请求的效果。

证据：`crates/macos/src/system/renderer_activation.rs` 中的测试 `observed_activation_overrides_a_setters_not_implemented_response`。