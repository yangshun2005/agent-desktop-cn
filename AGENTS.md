# AGENTS.md

面向在本仓库中工作的 AI 编码代理的指引。

agent-desktop 是供 AI 代理调用的工具，本身不是代理。它是一个 Rust CLI 和 C 库，读取原生无障碍树，返回带基于 ref 的元素 ID 的结构化 JSON。观察-行动循环位于调用方代理中。它从不内嵌 LLM，也没有 GUI 或交互式提示。它不自动化网页（请用 agent-browser）。它无法看到不暴露无障碍树的 UI。

## 命令

每个平台适配器只在其自身操作系统上编译，而工作区同时命名了全部三个平台，因此未限定范围的 `cargo build`、`cargo test --workspace` 或 `cargo clippy --all-targets` 在任何主机上都会失败。请将 cargo 限定到主机的包集合，即 `.githooks/pre-commit` 使用的同一集合：

```bash
HOST_PKGS=(-p agent-desktop-core -p agent-desktop-macos -p agent-desktop -p agent-desktop-ffi)
# Windows：将 agent-desktop-macos 替换为 agent-desktop-windows。Linux：替换为 agent-desktop-linux。
# 数组在 bash 和 zsh 中都会展开为独立参数；普通字符串在 zsh 中不会。

cargo build "${HOST_PKGS[@]}"                                  # 调试构建
cargo build --release -p agent-desktop                         # 发布版 CLI；超过 15 MB 时 CI 失败
cargo test "${HOST_PKGS[@]}"                                   # 所有测试目标
cargo test "${HOST_PKGS[@]}" --lib                             # 仅库测试；跳过二进制 crate
cargo test -p agent-desktop                                    # 二进制 crate：CLI 契约、分发、批处理、策略
cargo test "${HOST_PKGS[@]}" <test_name>                       # 按名称运行单个测试
cargo clippy "${HOST_PKGS[@]}" --all-targets -- -D warnings    # 零警告
cargo fmt --all -- --check
cargo check -p agent-desktop-core --all-targets --target x86_64-pc-windows-msvc   # core 可交叉编译
cargo check -p agent-desktop-core --all-targets --target x86_64-unknown-linux-gnu
cargo tree -p agent-desktop-core                               # 不得出现任何平台 crate
AGENT_DESKTOP_E2E_EXCLUSIVE=1 bash tests/e2e/run.sh            # macOS E2E；自行构建产物
bash scripts/perf-baseline-compare.sh                          # macOS 性能 A/B，对比 merge-base
```

`scripts/cargo-test-isolated-home.sh test "${HOST_PKGS[@]}" --lib` 使用一次性 `HOME` 运行测试，使其不进入你真实的 `~/.agent-desktop`。pre-commit 钩子使用它。

每个克隆启用一次钩子：`git config core.hooksPath .githooks`。对于暂存的 Rust 或 TOML 变更，它会在主机包集合上运行源码规则脚本（`check-rust-file-size.sh`、`check-no-phase-references.sh`、`check-stale-ref-constructor-misuse.sh`）、fmt、clippy 和库测试。FFI 变更会额外运行 C ABI 透传测试和 cbindgen 头文件检查。仅在紧急情况下绕过：`git commit --no-verify` 或 `SKIP_PRECOMMIT=1 git commit ...`。

## 事实来源

- **已发布行为**是代码及其测试。与代码矛盾的文档是缺陷；在同一变更中修复它。
- **代理用法**是 `skills/` 下的技能。它们会编译进二进制：`agent-desktop skills get desktop` 和 `agent-desktop skills get platform`。
- **词汇表**是 `CONCEPTS.md`。**JSON 契约**是 `docs/json-output.md`。**过往经验**在 `docs/solutions/` 中，带 YAML frontmatter（`module`、`tags`、`problem_type`）；在有文档记录的领域工作前先搜索它。
- **用户文档**位于 `apps/docs`，在 agent-browser.dev/desktop 提供服务。撰写或编辑文字时使用 `technical-writer` 技能（`.agents/skills/technical-writer/SKILL.md`）：文档页面、README、技能、变更日志条目、提交信息、PR 描述和 Issue。文档服务于开发者及调用 agent-desktop 的代理，因此保持完整、简短，并与代码核对。
- **规划文档**（阶段路线图 `phases.md`、子阶段计划、头脑风暴）位于仓库根目录的 `.compound-engineering/`。该文件夹有意不纳入版本控制。规划时阅读它。绝不要在已提交文件中链接或引用它，因为它在其他任何检出中都不存在。
- 当计划与代码矛盾时，以代码为准说明已发布内容。就地更正计划，不加"此前说"的注释，并引用推翻它的证据：`probes/**/FINDINGS.md` 的行 ID 或源码行。

### 实现它；不要推迟

- 实现者负责让工作落地。"比预期更难"、"PR 已经很大"和"后续阶段会涉及"都不是推迟的理由。
- 仅在工作被阻塞时推迟：平台上不可能、依赖不存在的基础设施，或只有破坏既定不变量或安全属性才可能。指明是哪一种。
- 推迟前先测量。用探针解决未知；基于假设的推迟只是猜测。
- 缩减范围是负责人的决定。如果工作无法落地，说明并停止。绝不悄悄缩小范围。
- 在 PR 开启前将每项推迟写入本地路线图的接收阶段，附上迫使其推迟的证据，并在 PR 描述中指明。如果它改变了当前阶段交付的内容，也要更正该阶段。

## 分支与提交

- `main` 承载 macOS、Windows 和 Linux。修复和功能从 `main` 分支，以 squash merge 合并回去；`main` 要求线性历史。
- 绝不添加 `Co-Authored-By` 行、AI 署名或"Generated with"页脚。
- 使用 Conventional Commits：`feat:`、`fix:`、`feat!:` 或 `BREAKING CHANGE:` 页脚、`docs:`、`style:`、`refactor:`、`perf:`、`chore:`、`ci:`、`test:`。冒号后写小写的 `type: 祈使句摘要`，并说明原因。
- 版本从 1.0.0 起遵循 semver。一次发布是一个 PR，提升 `Cargo.toml`、`Cargo.lock`、`npm/package.json` 和 `CHANGELOG.md` 中的工作区版本。合并它会启动 `.github/workflows/release.yml`，发布 npm 和 GitHub release；`docs/releasing.md` 有步骤。

## 架构

```
crates/core/     agent-desktop-core：adapter/ 中的 PlatformAdapter trait，领域文件夹中的共享类型，
                 commands/ 中每个命令一个文件
crates/macos/    agent-desktop-macos：AX API
crates/windows/  agent-desktop-windows：UI Automation
crates/linux/    agent-desktop-linux：AT-SPI
crates/ffi/      agent-desktop-ffi：cdylib 及已提交的 C 头文件 include/agent_desktop.h
src/             agent-desktop 二进制：cli/、cli_args/、dispatch/、batch/、command_policy/、
                 diagnostic/、visual_debug/、mcp/
skills/          代理技能，编译进二进制
tests/           黄金夹具、各 OS 的 E2E 测试框架、FFI 冒烟测试
probes/          原始平台测量；FINDINGS.md 行是可引用的证据
apps/docs/       Geistdocs 站点，作为 Vercel Microfrontends 子应用在 agent-browser.dev/desktop 提供服务
```

**依赖倒置（不可协商）。**
- Core 定义 `PlatformAdapter` 和所有共享类型。平台 crate 实现该 trait。
- Core 绝不导入平台 crate，平台 crate 之间绝不互相导入。如果 `cargo tree -p agent-desktop-core` 出现平台 crate，CI 失败。
- 只有两个 crate 将平台连接到 core：二进制（`src/`）和 FFI crate。
- 平台在编译时由 `build_adapter()`（`src/main.rs`、`crates/ffi/src/adapter.rs`）中的 `#[cfg(target_os)]` 选择，依赖在 `src/Cargo.toml` 中按目标门控（`[target.'cfg(target_os = "macos")'.dependencies]`），绝不在源码 `cfg` 后使用无条件依赖。
- `PlatformAdapter` 拆分在 `crates/core/src/adapter/` 中（`mod.rs` 加上 `actions`、`input`、`observation` 和 `system` 片段）。每个方法默认返回 `AdapterError::not_supported`，因此适配器只实现它支持的方法。严格解析（`resolve_element_strict*`）在零匹配时返回 `STALE_REF`，在两个或更多匹配时返回 `AMBIGUOUS_TARGET`。
- Core 必须在每个 OS 上运行。core 中没有 CI 通道执行的 `#[cfg]` 分支是假设，不是已发布代码：添加通道或将其移到 `PlatformAdapter` 之后。

**分发。** `src/dispatch/mod.rs` 中的 `dispatch()` 是一个直接 `match`，调用 `crates/core/src/commands/` 中每个命令的 `execute()`。没有命令 trait 或注册表。批处理不是第二个分发器：它将 JSON 条目反序列化为同一个 `Commands` 枚举，运行相同的 `CommandPolicy` 预检并调用相同的 `dispatch()`。

**MCP。** `agent-desktop mcp`（`src/mcp/`）不是第二个命令面。它在运行时读取 clap 命令树：每个叶子命令是一个工具，其 schema 来自命令的 clap 参数和帮助，调用会重建 argv 并作为子进程运行此二进制。在 `Commands` 中添加或移除的命令会自动出现在 MCP 工具列表中或从中消失，无需改动 MCP。`mcp` 本身在 `main.rs` 中挂载到根命令，位于 `Commands` 之外，因此它永远不是工具、批处理条目或计入的命令。

**添加命令。**
1. 创建 `crates/core/src/commands/<name>.rs`，含 `execute()` 函数，测试放在 `<name>_tests.rs`，并在 `commands/mod.rs` 中注册。
2. 在 `src/cli/mod.rs` 中添加 CLI 变体，参数放在 `src/cli_args/` 下。
3. 将其分支添加到 `dispatch()` 和 `src/command_policy/mod.rs` 中的 `policy_for()`。两个 match 都是穷尽的。
4. 将其名称添加到 `src/batch/mod.rs` 中的 `parse_command()`；否则 `every_cli_subcommand_is_known_to_batch_parser` 失败。将其添加到 `src/cli/contract_command_surface_tests.rs` 中的表面列表。
5. 更新 `README.md` 和本文件中的命令计数；`src/cli/doc_literal_drift_tests.rs` 会检查它们。
6. 新的 `Action` 变体放在 `crates/core/src/actions/action.rs`。新的适配器方法放在匹配的 `PlatformAdapter` 片段中，带 `not_supported` 默认值。
7. 在同一变更中更新匹配的技能：跨平台行为用 `skills/agent-desktop/`，平台差异用平台技能，C ABI 用 `skills/agent-desktop-ffi/`。技能编译进二进制，是代理读取的内容。

**Crate 布局。** Core 根目录只放 `lib.rs` 和以 `CONCEPTS.md` 词汇命名的领域文件夹：`actionability`、`actions`、`adapter`、`commands`、`context`、`cursor_overlay`、`input`、`locator`、`notifications`、`ref_action`、`refs`、`response`、`session`、`snapshot`、`storage`、`system`、`trace`、`tree` 和 `wait`。
- 平台 crate 的根目录放 `lib.rs`、`adapter.rs`（trait 实现）和五个层文件夹，存在辅助二进制时加 `bin/`。`tree/` 读取、遍历和解析 UI；`actions/` 对元素执行操作；`input/` 合成原始键盘、鼠标和剪贴板输入；`system/` 处理应用生命周期、窗口、权限和截图；`notifications/` 处理 OS 通知面。
- 在层内，有四个或更多源文件的概念获得以共享词汇命名的子文件夹（`resolve`、`walk`、`properties`、`select`、`scroll`、`clipboard`、`keyboard`、`mouse`、`window`、`capture` 等）。文件名保留其前缀（`system/window/window_ops.rs`）。与文件夹同名的文件成为其 `mod.rs`。
- 测试与代码并列，命名为 `foo_tests.rs`，从 `foo.rs` 用 `#[cfg(test)] #[path = "foo_tests.rs"] mod tests;` 声明。
- 能力实现放在 `adapter.rs` 或匹配层的 `adapter.rs` 中。

**基础规则。** `scripts/check-architecture.sh`（规则在 `scripts/check_architecture.py`）在 pre-commit 钩子和 CI 中与源码规则一起运行。扫描前，它会植入每条规则的违规，因此停止检测的规则会使门禁失败。当 core 依赖平台或 OS crate、平台 crate 触及另一个平台 crate、core 在其允许列表之外获得 OS 条件代码、crate 的顶层布局漂移、能力在错误的层实现、core 领域触及 `commands/`，或测试文件未在其代码旁声明时，它会失败。其列表（core 依赖、core 领域、二进制文件夹、OS 允许列表）仅在评审中变更。CI 的 `cargo tree` 隔离步骤仍是传递性检查。

## 编码标准

- **每个手写 `.rs` 文件最多 400 行**（`scripts/check-rust-file-size.sh`）。按职责拆分。前五行带 `@generated` 标记的生成 `.rs` 文件豁免，修复应在生成器中，而非输出中。探针 `.ps1` 脚本豁免。C 头文件由 cbindgen 生成，CI 检查漂移：更改 Rust ABI 或 `crates/ffi/cbindgen.toml` 后用 `scripts/update-ffi-header.sh` 重新生成，绝不手工编辑。
- **无内联注释。** 名称承载含义。独立的 `//` 行、行尾 `//` 注释和非文档 `/* */` 块都被拒绝。允许的形式是公共项上的 `///`（仅当名称不够时）和 `//!` 模块文档。`scripts/check-rust-file-size.sh` 通过 `scripts/check_rust_comments.py` 与大小上限一起强制执行。
- **`crates/**` 或 `src/**` 中无交付计划引用**：无阶段、子阶段、计划决策 ID（`KTD<n>`）或计划单元 ID（`U<n>`）。写关于代码的真实内容和原因。探针台账行 ID 如 `A15-7` 允许并鼓励。`scripts/check-no-phase-references.sh` 强制执行。
- **每个文件一个领域结构体或枚举**（`node.rs` 持有 `AccessibilityNode`）。**每个文件一个命令**，以命令命名。
- **每个结构体最多 7 个字段，每个函数最多 5 个参数。** 使用配置结构体或构建器。
- **显式 `pub` 边界。** 只有 `lib.rs` 重新导出。内部模块为 `pub(crate)`。无通配符重新导出。
- **命名。** Crate 为 `agent-desktop-<name>`；模块文件为单数 `snake_case`；类型为 `PascalCase`；函数为动词优先的 `snake_case`；常量为 `SCREAMING_SNAKE_CASE`；CLI 标志为 kebab-case。

**错误。**
- 测试外无 `unwrap()`、`expect()` 或 `panic!`；panic 仅限测试。用 `?` 或 match 传播。
- 适配器函数返回 `Result<T, AdapterError>`。命令处理器返回 `Result<serde_json::Value, AppError>`。`main()` 将 `AppError` 转为 JSON 信封和退出码。
- 错误携带 `code`（`ErrorCode`）、`message`，以及可选的 `suggestion`、`platform_detail` 和 `details`，加上其投递处置。
- 代码：`PERM_DENIED`、`ELEMENT_NOT_FOUND`、`APP_NOT_FOUND`、`ACTION_FAILED`、`ACTION_NOT_SUPPORTED`、`STALE_REF`、`AMBIGUOUS_TARGET`、`WINDOW_NOT_FOUND`、`PLATFORM_NOT_SUPPORTED`、`TIMEOUT`、`INVALID_ARGS`、`NOTIFICATION_NOT_FOUND`、`SNAPSHOT_NOT_FOUND`、`POLICY_DENIED`、`APP_UNRESPONSIVE`、`INTERNAL`。
- 退出码：`0` 成功，`1` 结构化错误，`2` 参数或解析错误。

## 输出契约

每个命令打印一个信封。`docs/json-output.md` 持有完整契约。

```json
{ "version": "2.6", "ok": true, "command": "snapshot",
  "data": { "app": "Finder", "snapshot_id": "s8f3k2p9", "ref_count": 14, "complete": true, "tree": { } } }
```

```json
{ "version": "2.6", "ok": false, "command": "click",
  "error": { "code": "STALE_REF", "message": "...", "suggestion": "...",
             "re
```