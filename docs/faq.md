# agent-desktop 常见问题

## 什么是 agent-desktop？

agent-desktop 是一个 CLI 和 C 库。它读取 macOS、Windows 和 Linux 上桌面应用的无障碍树，并以 JSON 返回。每个 agent 可操作的元素都会获得一个 ref。调用方 agent 负责执行观察、操作、验证循环，因为 agent-desktop 内部没有模型。它不自动化网页。网页请使用 [agent-browser](https://agent-browser.dev)。其余内容见[简介](https://agent-browser.dev/desktop)。

## agent-desktop 需要截图或像素匹配吗？

不需要。接受 ref 的命令在无障碍树中查找元素，而非匹配像素。`screenshot` 是独立命令。不发布无障碍树的应用对 agent-desktop 不可见。

## agent-desktop 如何工作？

`snapshot` 读取应用的无障碍树并以 JSON 打印，每个 agent 可操作的元素都带一个 ref。操作命令接受来自 snapshot 的 ref，每条命令打印一个 JSON 信封。[简介](https://agent-browser.dev/desktop)展示了各组件，从 CLI 或 C 库到平台无障碍 API。

## 支持哪些平台？

macOS、Windows 和 Linux。[README 中的平台表](../README.md#platform-support)列出了各项要求。`agent-desktop skills get platform` 打印二进制所在操作系统的平台指南。

## 如何安装 agent-desktop？

```bash
npm install -g agent-desktop
```

[安装](https://agent-browser.dev/desktop/installation)涵盖其他包管理器、发布归档和从源码构建。[docs/install.md](install.md) 为 GitHub 读者提供相同细节。

## 如何从源码构建 C 库？

```bash
cargo build --profile release-ffi -p agent-desktop-ffi
```

库输出到 `target/release-ffi/`：macOS 和 Linux 上为 `libagent_desktop_ffi.dylib` 或 `.so`，Windows 上为 `agent_desktop_ffi.dll`。只有 `release-ffi` profile 会在边界捕获 panic 并返回 `AD_RESULT_ERR_INTERNAL`。普通 `--release` 构建不会。源码构建还需要库旁边的平台辅助程序。见[构建与链接](https://agent-browser.dev/desktop/ffi/build-and-link)。

## ref 系统是什么？

`snapshot` 按深度优先顺序分配 ref。ref 携带其来源 snapshot 的 ID，例如 `@s8f3k2p9:e1`。操作时传入完整形式。裸 `@e1` 会以 `INVALID_ARGS` 失败。

元素在角色属于以下之一，或声明了主操作时获得 ref：

`button`、`textfield`、`checkbox`、`link`、`menuitem`、`tab`、`slider`、`combobox`、`treeitem`、`cell`、`radiobutton`、`incrementor`、`menubutton`、`switch`、`colorwell`、`dockitem`、`listbox`、`option`。

Snapshot 查找限定在所选会话内。来自某会话的 ref 需要该会话的 `--session` 标志或 `AGENT_DESKTOP_SESSION` 值。当操作返回 `STALE_REF` 时，重新 snapshot 并用其中的 ref 重试。

## agent 如何了解该工具？

运行 `agent-desktop skills get desktop` 获取核心指南。运行 `agent-desktop skills get platform` 获取二进制所在操作系统的指南。运行 `agent-desktop <command> --help` 查看命令的标志。

## agent-desktop 是免费开源的吗？

是。采用 [Apache-2.0](../LICENSE) 许可。

## 哪些数据可能包含隐私信息？

Snapshot、窗口标题、剪贴板读取、通知、截图和命令结果包含你请求读取的桌面内容。`--debug --screenshot PATH.html` 会写入一个本地 HTML 页面，包含截图和无障碍数据。会话跟踪会脱敏敏感的结构化字段，但可能保留证据产物。请将这些文件和 agent 的对话记录保密。脱敏并不能让截图可以安全发布。

原生 CLI 不需要远程模型服务。可选的 [JEV 辅助程序](../skills/jev-desktop/SKILL.md)会将你的目标和屏幕描述发送给你配置的模型提供商。仅对提供商可看的数据使用它。

`probes/` 下的探测捕获记录 OS 版本和窗口几何等技术细节。发布前请审查，即使已脱敏身份字段。

## 在哪里获取帮助？

bug 报告中应包含的内容见 [SUPPORT.md](../SUPPORT.md)。安全问题的私下报告方式见 [SECURITY.md](../SECURITY.md)。

- [Issues](https://github.com/vercel-labs/agent-desktop/issues)
- [发布版本](https://github.com/vercel-labs/agent-desktop/releases)
- [npm 包](https://www.npmjs.com/package/agent-desktop)