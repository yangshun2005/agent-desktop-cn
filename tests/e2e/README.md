# 原生 macOS 端到端测试

本套件驱动发布版 `agent-desktop` 二进制，针对真实 macOS 应用运行，并通过独立的无障碍观测验证效果。当测试夹具暴露了可观测的状态、值、进程、窗口或界面变化时，命令返回 `ok: true` 永远不足以作为通过依据。

## 运行夹具套件

```bash
AGENT_DESKTOP_E2E_EXCLUSIVE=1 bash tests/e2e/run.sh
```

唯一的先决条件门禁要求：

- macOS；
- 终端或运行器已获得辅助功能权限；
- 可构建的 Swift 夹具应用。

`AGENT_DESKTOP_E2E_EXCLUSIVE=1` 是一项安全确认，表示调用方已独占桌面，包括停止用户输入和其他自动化。测试框架锁可阻止第二个原生测试框架启动，但无法阻止无关的 `agent-desktop` 进程在命令之间启动。桌面正在使用时不要设置该确认。

在获取桌面锁之前，`run.sh` 会用 `--locked` 从当前检出构建 CLI、macOS 辅助程序和 FFI 库，输出到规范的仓库 `target` 目录。随后对这些确切的构建产物计算哈希，并复制到私有套件目录下的只读路径。套件只使用这些副本，外加隔离的 `HOME`、ref/会话存储、夹具构建和 `TMPDIR`。每个 CLI 子进程都在自己的进程组中运行，带绝对超时和受限的 stdout/stderr 捕获。在第一个场景运行前，JSON `version` 和 Clap `--version` 输出都必须与 `Cargo.toml` 匹配；报告成功前会再次校验构建产物哈希。

在夹具套件启动前，`run.sh` 会运行 `safe-semantic.sh`，这是一个严格的无头非干扰门禁。它在后台启动夹具，并断言语义化的 click、type、set-value 和 check/uncheck/toggle 命令各自产生效果，同时夹具始终不会成为前台应用。其退出码（`1` 失败，`2` 先决条件受阻）会在任何有头用例之前终止运行。

随后测试框架构建并启动 `AgentDeskFixture.app`。夹具运行后，缺失控件、窗口、状态读数、边界记录或界面都视为测试失败。场景绝不会把缺失的先决条件变成成功的 `SKIP`。退出码 `0` 表示所有断言都在未受污染的运行中通过，`1` 表示行为失败，`2` 表示全局先决条件门禁失败。

有头用例会移动真实光标。请在安全的机器上运行。即使测试失败，清理陷阱也会强制关闭夹具进程。场景结束后，`run.sh` 关闭夹具，并在无头和有头模式下运行 `scripts/cursor-multi-agent-probe.py`。

对于专用的自托管运行器，`scripts/run-native-e2e-ci.sh` 会设置独占确认并调用 `run.sh`；除非设置了 `AGENT_DESKTOP_NATIVE_E2E_RUNNER=1`，否则拒绝运行。`native-e2e.yml` 工作流仅支持 `workflow_dispatch`。

## 精确的 ref 命名空间

每个非计数 `find` 返回两个不可分割的值：

```json
{"ref_id":"@s8f3k2p9:e12","snapshot_id":"s8f3k2p9"}
```

`lib.sh` 将该对作为一个目标携带，每个 ref 操作、`get`、`is` 和元素 `wait` 都传入限定 ref，其中内嵌了其快照。测试框架故意在 `find` 和 `get` 之间运行无关观测，以证明 ref 不依赖可变的 latest-snapshot 指针。不要添加返回裸 `@eN` 的辅助函数。

## 目录结构

| 路径 | 职责 |
|---|---|
| `run.sh` | 全局门禁、夹具生命周期、语义套件编排 |
| `harness.sh` | 独占确认、隔离的 `HOME`/`TMPDIR`、不可变二进制和 FFI 暂存 |
| `lib.sh` | 失败关闭断言和精确的 `{ref_id,snapshot_id}` 目标辅助函数 |
| `safe-semantic.sh`、`scenarios/silent-text.sh`、`safe_semantic_guard.py` | 在夹具套件之前运行的严格无头非干扰门禁 |
| `scenarios/observation.sh` | 快照、定位器、命名空间固定、严格孪生 |
| `scenarios/interaction.sh` | 无头/有头操作和精确一次效果 |
| `scenarios/acceptance.sh` | 命名的 AE1-AE7 验收契约 |
| `scenarios/reliability.sh` | 过期 ref、等待、下钻、会话 |
| `scenarios/surfaces.sh` | 表单、菜单、拖拽、展开 |
| `scenarios/error_contract.sh` | 智能体据此分支的退出码和 `error.code` 信封，`batch` 的 continue/stop/reject 语义 |
| `scenarios/keyboard.sh` | 通过夹具自身文本读数观测的 `press` 组合键、被阻止组合键和未知修饰键拒绝 |
| `scenarios/trace_performance.sh` | 跟踪产物、脱敏、计时、清理 |
| `scenarios/ref_churn.sh` | UI 真正变化后的 `STALE_REF`，在专用夹具进程上运行（`AGENT_DESKTOP_FIXTURE_REF_CHURN=1`） |
| `scenarios/notifications.sh` | 真实通知中心生命周期 |
| `scenarios/os_surfaces.sh` | Dock 启动、控制中心面板和音量滑块、按可执行文件名指定 `--app` |
| `permission-contract.sh` | 确定性的 AE5 映射测试，不修改 TCC |
| `electron-live.sh`、`electron_metrics*.py` | 可选的已安装 Electron/Chromium 应用测量 |
| `json_tool.py`、`bounded_process.py`、`guard-command.sh` | JSON 字段提取，以及带超时/输出上限、进程组隔离的命令运行器 |
| `interaction_lock.py` | 由 `run.sh`、`safe-semantic.sh` 和 `electron-live.sh` 持有的独占桌面租约 |
| `ffi_live.py` | AE2 使用的原生发布版 FFI 消费者 |
| `test_*.py` | 测试框架自身的无桌面单元测试 |

每个 shell 文件保持在 400 行以内（仓库的文件大小约定；`scripts/check-rust-file-size.sh` 仅对 `.rs` 文件强制执行上限），并兼容 Bash 3（macOS 自带 Bash 3）；`scripts/check-bash3-compat.sh` 门禁顶层 `tests/e2e/*.sh` 文件。

测试框架自测不需要桌面，在 `fmt` CI 通道中运行：

```bash
python3 -m unittest discover -s tests/e2e -p 'test_*.py'
```

## AE1-AE7 验收映射

`scenarios/acceptance.sh` 为每个验收示例命名并失败关闭：

| 示例 | 原生夹具断言 |
|---|---|
| AE1 | 一个可寻址按钮的实时 AX 帧为零时报告 `visible=false`。 |
| AE2 | CLI 和原生发布版 FFI 消费者等待一个 800 ms 后启用的按钮，精确派发一次，并在超时为零时执行一次立即检查且无延迟效果。 |
| AE3 | 一个永久禁用的按钮在 `--timeout-ms 2000` 下返回 `TIMEOUT`、`details.kind=actionability_timeout`，且 `details.last_report` 接近 2 秒。 |
| AE4 | 两个标题相同的窗口获得不同 id；聚焦第二个 id 会聚焦该确切窗口。 |
| AE5 | 确定性测试证明提示隔离和非提示的 Automation 探测；当运行器已有被拒绝的 Automation TCC 时，原生有头通知中心操作必须返回 `PERM_DENIED`。其他 TCC 状态会明确记录为不可用，而不是声称已执行。 |
| AE6 | 一个 batch 捕获操作前基线，点击打开表单，并报告 `surface_appeared` 而不命名表单标题。 |
| AE7 | 同一禁用操作不带超时标志时，返回接近未改动的 5 秒默认值的结构化超时。 |

AE2、AE3 和 AE7 在无头和有头策略模式下都运行。耗时容差考虑了进程启动和调度器噪声，但足够窄，可检测缺失等待、错误默认值或无界超时。延迟操作状态在稳定间隔后观测，因此重复的延迟派发无法通过。

`permission-contract.sh` 有意保持确定性。它测试过期截止时间、辅助程序结果和 Automation 非必需契约，不调用原生提示 API、不重置 TCC，也不假设开发者的同意状态。

## 已安装 Electron/Chromium 测量

实时测试框架为可选，绝不由 `run.sh` 或无特权 CI 调用：

```bash
AGENT_DESKTOP_E2E_EXCLUSIVE=1 bash tests/e2e/electron-live.sh --app Slack
AGENT_DESKTOP_E2E_EXCLUSIVE=1 bash tests/e2e/electron-live.sh \
  --app "Visual Studio Code" \
  --baseline-binary /path/to/previous/agent-desktop \
  --out /tmp/vscode-ax.json
```

应用必须已安装、正在运行，并至少暴露一个窗口。测试框架不会启动、聚焦、点击、输入或关闭用户的应用。任何先决条件不可用时，调用会失败，而不是输出成功的部分基准测试。

它精确执行五对预热和 31 对测量的发布版二进制 `find --role button --first`。提供 `--baseline-binary` 时，不可变基线（A）和当前（B）二进制以平衡的 AB/BA 顺序，针对同一确切进程代和窗口清单运行，使用独立的 `HOME` 目录。每对前后都检查状态。v3 JSON Schema 包含最近秩墙钟和 CPU p50/p95、配对的当前减基线差值、成功率和正确率、二进制输出时的 AX 遍历/读取统计、渲染器激活计数、SHA-256/版本标识、原始样本，以及精确快照重解析检查。它有意省略 RSS：`RUSAGE_CHILDREN.ru_maxrss` 是测试框架的累积最大值，不是每命令测量值。

报告标注其局限：它描述一台机器、一种应用状态和一种 TCC 状态；它测量观测和 ref 重解析，而非操作投递；未明确提供基线时绝不虚构基线。默认输出为 `/tmp` 下带时间戳的文件；传入 `--out` 可保留到其他位置。

## 添加场景

1. 为夹具添加稳定的无障碍标签和可独立观测的状态。
2. 使用 `require_target` 解析操作目标；绝不保留裸 ref。
3. 使用 `act_target`、`get_target`、`is_target` 或 `wait_target`，使源快照 id 始终明确。
4. 设置完成后，将缺失控件和不支持的行为视为失败。
5. 对于计时行为，同时断言结构化 JSON 契约和耗时边界，然后验证稳定后不出现重复效果。
6. 从 `run.sh` 中 source 新场景文件，放在现有场景之后，并由 `check_fixture_contention` 保护。