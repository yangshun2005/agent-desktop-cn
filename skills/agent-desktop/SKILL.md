---
name: agent-desktop
tags: computer-use, desktop-automation, accessibility, ai-agent, gui-automation, cli
requirements:
  - agent-desktop
description: >
  通过原生无障碍树，使用 agent-desktop CLI 在 macOS、Windows 和 Linux 上操作桌面应用。
  适用于 AI agent 需要查看或操作桌面 GUI 的场景：点击按钮、填写表单、读取 UI 状态、导航菜单、
  切换复选框、滚动、拖拽、输入文本、按下快捷键、截图、管理窗口、使用剪贴板、启动或关闭应用、
  读取通知。
  触发词："click button"、"fill form"、"open app"、"read UI"、"computer use"、
  "operate desktop"、"accessibility tree"、"snapshot app"、"type into field"、"navigate menu"、
  "toggle checkbox"、"take screenshot"、"desktop automation"、"agent-desktop"，
  或 macOS、Windows、Linux 上的任何桌面 GUI 任务。
---

# agent-desktop

agent-desktop 是一个 CLI，通过操作系统无障碍树读取和操作桌面应用。每条命令输出一个 JSON 信封（`ok`、`data` 或 `error`）。它是工具而非 agent：观察、操作、验证的循环由你执行。

## Shell 语法

示例使用 POSIX shell。在 PowerShell 中，每个 ref 都要加引号（`'@s8f3k2p9:e1'`），因为裸 `@token` 是 splatting 运算符。用 `$env:NAME = 'value'` 代替 `export`，用 `| ConvertFrom-Json` 代替 `$(...)` 和 `jq`。Windows 技能中有 PowerShell 写法。

## 首次运行

1. 运行 `agent-desktop status`。它报告平台、权限和 `supported_surfaces`。
2. 运行 `agent-desktop skills get platform`。它加载二进制所在操作系统（macOS、Windows 或 Linux）的技能。
3. 查看参数和语法，运行 `agent-desktop <command> --help`。本技能不重复参数列表。

## 循环

知道角色或确切名称时用 `find`。否则先映射应用，再深入某个区域。

```
agent-desktop snapshot --skeleton --app "Notes" -i --compact   # 浅层映射，被截断的分支带 children_count
agent-desktop snapshot --root @s8f3k2p9:e3 -i --compact        # 深入一个区域
agent-desktop click @s8f3k2p9:e12                              # 对限定 ref 执行操作
agent-desktop snapshot --root @s8f3k2p9:e3 -i --compact        # 验证：重读同一区域
```

- 小应用无需 skeleton。直接做完整 `snapshot -i`。
- `-i` 只保留可交互节点，因此行内的静态文本标签会被丢弃。要判断重复按钮属于哪一行，用 `--root` 深入该行的容器并去掉 `-i`。在 Windows 和 Linux 上，这样能读到该行，但不会让其按钮可操作：在其他行重复出现的按钮是窗口中的孪生元素，其 ref 会被拒绝（见 Refs）。
- `--root` 深入会把其 ref 加入同一 snapshot id。被深入子树之外的 ref 以及被深入的根节点本身仍然有效；原本就在其中的 ref 会被替换，因此那里的旧 ref 会返回 `STALE_REF`。
- 绝不对 surface（`--surface menu`、`sheet`、`alert`）做 skeleton。surface 本身已聚焦。
- 打开菜单、sheet 或 alert 的操作会在 `data.surfaces` 中列出它。接下来以该浮层为目标。
- 异步步骤之后，用 `wait`（见 `commands-system.md`）。固定 sleep 是竞态。

## Refs

- 输出 ref 是限定的：`@<snapshot_id>:e<N>`，例如 `@s8f3k2p9:e1`。原样传给后续命令。
- ref 仅对一次 snapshot 有效。UI 变化后，重新深入受影响区域或重新 snapshot。
- 只有可寻址元素才有 ref。静态文本和非交互容器没有。
- 命令在执行前会重新识别元素。在 macOS 上，ref 携带元素自身地址，因此相同的孪生元素各自保留 ref。在 Windows 和 Linux 上，只有当该 ref 是其 snapshot 以该角色、原生 id、名称和描述在窗口中捕获的唯一元素（即使位于 skeleton 或深度截断之下也计入），且仍恰好有一个活动元素匹配时，ref 才能解析。那里的相同孪生元素没有 ref 路径：只有 `--headed mouse-click --xy X,Y` 能选中一个，它会抢占前台，而 Linux Wayland 会话会拒绝它。
- `STALE_REF`：元素已消失或不再匹配。`details.kind` 为 `saved_ref_absent`，或在 macOS 上为 `retained_object_stale`。重新 snapshot 并使用新 ref。
- `AMBIGUOUS_TARGET`：出现了相同元素（`saved_ref_ambiguous`），或 ref 在捕获时未被证明唯一（`identity_not_unique`）。重复该 ref 或对相同孪生元素重新 snapshot 会得到同样的拒绝；请操作窗口中唯一的控件。`identity_not_unique` 也可能出现在未能及时读取整个窗口的 snapshot 之后：用更大的 `--timeout-ms` 重新 snapshot。
- `TIMEOUT` 且 `details.kind: resolution_identity_unknown`：实时读取未响应，身份未获证明。未发送任何内容；重试。
- `SNAPSHOT_NOT_FOUND`：snapshot id 已过期。重新 snapshot。

## 投递与重试

操作结果携带 `data.disposition`，错误携带 `error.disposition`，各自包含 `delivery` 和 `retry`。`snapshot` 等观察命令成功时没有这些字段；它们的错误以及所有 `INVALID_ARGS` 拒绝都携带 `not_delivered` 和 `retry: safe`。

- 仅当 `disposition.retry` 为 `safe` 时才重试或替换操作。只有 `not_delivered` 是安全的。
- 其他任何 delivery 值都意味着操作可能已生效。重复执行可能点击、输入或提交两次，`toggle` 可能自我撤销。
- 结果不安全时，读取实时状态（`get`、`is` 或新 snapshot）再据此决定。存在 `details.post_state` 时可供参考。
- 依据 `error.code` 和 `error.disposition` 分支。消息和建议文本可能变化。
- `details.retryable: false` 表示等待无法改变结果。换一种方法。

## 无头与有头

ref 操作默认无头。它们使用语义无障碍调用，不抢占焦点、不移动光标、不合成按键、不使用剪贴板。若不存在语义路径，操作会失败关闭。仅当用户明确允许物理输入（如双击、悬停、拖拽或聚焦）时，才使用全局 `--headed` 标志。绝不要为了从无头失败中恢复而切换到有头模式。原始坐标（`--xy`）从不意味着聚焦。哪些操作需要 `--headed` 因操作系统而异：阅读平台技能。

## 参考

仅在需要时加载：`agent-desktop skills get desktop <reference>`。

| 参考 | 何时打开 |
|-----------|--------------|
| `commands-observation` | 需要 `snapshot`、`find`、`get`、`is`、`screenshot`、`list-surfaces` 语义、部分树或 `--root` 规则 |
| `commands-interaction` | 在 click、type、set-value、select、toggle、scroll、press 和鼠标命令之间选择，或调试操作错误 |
| `commands-system` | 启动或关闭应用、管理窗口、使用剪贴板、`wait`、`batch`、会话、多 agent 光标、追踪，或加载技能 |
| `workflows` | 需要实战模式（表单、菜单、对话框、等待新窗口、滚动查找、拖拽）和反模式列表 |