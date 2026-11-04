# 定位器解析基准测试

一个确定性测试框架，用于对比旧版全快照定位器路径与无句柄的观测树求值器。两者都运行在合成的 Chromium 和 Electron 无障碍树上。这些树覆盖深层匿名包装链、重复的角色和名称候选、移动边界、同时设置的 `AXIdentifier` 和 `AXDOMIdentifier`、包含谓词以及大型树。

无需网络访问即可运行：

```bash
cargo run -p agent-desktop-core --release --example locator_benchmark \
  > /tmp/agent-desktop-locator-synthetic.json
```

JSON 报告包含：

- 31 次测量运行的 p50 和 p95 墙钟延迟
- 候选节点、谓词工作量以及请求的合成属性值
- 基数与正确性

三种实时模式测量不同内容：

- `live_find_selected_refs` 使用与默认 CLI `find` 路径相同的仅选中匹配物化方式。
- `live_arena_direct` 是开销更低的直接目标指标，单独报告。
- `live_count_no_refmap` 检查仅计数请求不执行任何动作或可设置证据读取。

该框架不报告原生 AX 调用的绝对次数，因为依赖角色的动作和可设置探测会使该计数不准确。原生 Slack 和 Electron 测量需要辅助功能权限。请使用需主动启用的实时框架 `tests/e2e/electron-live.sh` 运行它们，而不是使用此基准测试。

要生成全仓库报告，请运行 `bash scripts/perf-baseline-compare.sh`，可选带 `--apps "Slack,Google Chrome"`。它会写出 `report.html`。该报告涵盖 HEAD 与 merge-base 的夹具 A/B、可选的只读应用探测以及此基准测试。

## Electron 和 Chromium 激活（macOS）

对于 Electron 和 Chromium 应用，macOS 适配器在读取应用之前会先开启无障碍树。当应用支持时设置 `AXManualAccessibility`，否则设置 `AXEnhancedUserInterface`。如果值已经是 `true`，则跳过设置。然后在请求截止时间内等待，直到该属性报告就绪。不暴露该属性的原生应用保持不变。这遵循 [Electron 关于第三方辅助软件的无障碍指南](https://github.com/electron/electron/blob/main/docs/tutorial/accessibility.md)。基准测试在设置之后读取就绪状态，因为设置成功并不意味着树已就绪。