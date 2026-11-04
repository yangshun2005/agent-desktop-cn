# 为 agent-desktop 做贡献

agent-desktop 是一个 Rust CLI 和 C 库。它读取原生无障碍树，返回带基于 ref 的元素 ID 的结构化 JSON，让 AI agent 能够观察并操作桌面应用。

## 适合的贡献

- 修复 Bug，例如 JSON 输出错误、ref 解析不正确，或稳定目标上出现 `STALE_REF`。
- 新增命令。遵循 [AGENTS.md](AGENTS.md#architecture) 架构章节中的步骤。
- macOS、Windows 和 Linux 的平台适配器修复与功能。每个适配器实现 `PlatformAdapter`。
- 应用特定的特殊情况，例如 Electron 或游戏引擎的边界情况，记录在 `skills/` 下。
- 文档和 skill 文件。行为变化时保持 `skills/agent-desktop*/` 准确。

不在范围内：嵌入 LLM、GUI 或 TUI 界面、浏览器自动化（请使用 agent-browser），以及宏录制。

## 行为准则

遵循[行为准则](CODE_OF_CONDUCT.md)。

## 前置条件

| 要求 | 说明 |
|---|---|
| Rust 工具链 | 1.89.0，固定在 `rust-toolchain.toml`。`rust-version` 1.89 是 MSRV。`rustup` 会安装固定的通道。 |
| macOS 13 或更高版本 | 构建和运行 macOS 适配器（`crates/macos/`）所需。需要 Xcode 命令行工具。 |
| Windows | 构建 Windows 适配器（`crates/windows/`）。仅限 Windows。需要 MSVC 构建工具。 |
| Linux | 构建 Linux 适配器（`crates/linux/`）。仅限 Linux。实时测试需要 AT-SPI 会话。参见 `tests/e2e-linux/README.md`。 |
| Python 3 | `scripts/check-rust-file-size.sh` 会运行一个 Python 辅助脚本，`PATH` 中没有 Python 3 时会失败。 |
| 无障碍权限 | macOS，用于针对真实应用的集成测试和 E2E 测试。在**系统设置 > 隐私与安全性 > 辅助功能**中添加你测试所用的终端。 |
| 屏幕录制权限 | macOS，用于 `screenshot` 测试。在**系统设置 > 隐私与安全性 > 屏幕录制**中添加终端。 |

## 入门

克隆仓库并启用 pre-commit hook。每次克隆运行一次 `git config` 命令。

```bash
git clone https://github.com/vercel-labs/agent-desktop
cd agent-desktop
git config core.hooksPath .githooks
```

每个平台适配器只在其自身的操作系统上编译，而 workspace 列出了全部三个。未限定范围的 `cargo build` 或 `cargo test --workspace` 在每个主机上都会失败，因此要将 cargo 限定到主机的包。示例使用 macOS。在 Windows 上，将 `agent-desktop-macos` 替换为 `agent-desktop-windows`。在 Linux 上，使用 `agent-desktop-linux`。

```bash
HOST_PKGS=(-p agent-desktop-core -p agent-desktop-macos -p agent-desktop -p agent-desktop-ffi)

cargo build "${HOST_PKGS[@]}"                  # debug build
cargo build --release -p agent-desktop         # release CLI; CI fails it above 15 MB
./target/release/agent-desktop snapshot --app Finder -i
```

`HOST_PKGS` 是一个数组，因此在 bash 和 zsh 中每个条目会展开为单独的参数。zsh 是 macOS 的默认 shell。

## 质量门禁

以下每项门禁都必须在 PR 合并前通过。按上文所示设置 `HOST_PKGS`。

```bash
cargo fmt --all -- --check
cargo clippy "${HOST_PKGS[@]}" --all-targets -- -D warnings
cargo test "${HOST_PKGS[@]}" --lib             # adapter stubs and golden fixtures; skips the binary crate
cargo test -p agent-desktop                    # CLI contract, dispatch, batch and policy tests
cargo test -p agent-desktop-ffi --tests        # C ABI, enum fuzzing and out-param zeroing
cargo tree -p agent-desktop-core               # must name no platform crate
AGENT_DESKTOP_E2E_EXCLUSIVE=1 bash tests/e2e/run.sh   # macOS only; needs an exclusive desktop and Accessibility permission
```

CI 通过 `scripts/cargo-test-isolated-home.sh` 运行其测试步骤，该脚本为 cargo 提供一个一次性的 `HOME`，因此测试永远不会触碰 `~/.agent-desktop`。pre-commit hook 对其 `--lib` 步骤也做同样处理。在本地运行该脚本以获得相同的隔离：

```bash
scripts/cargo-test-isolated-home.sh test "${HOST_PKGS[@]}" --lib
```

`tests/e2e/run.sh` 会自行构建 release CLI、macOS 辅助程序和 FFI 库。没有 `AGENT_DESKTOP_E2E_EXCLUSIVE=1` 时它会拒绝运行，因为有头用例会移动真实光标。它断言什么以及如何添加场景，参见 `tests/e2e/README.md`。

CI 还会检查本地运行可能遗漏的内容：

- 每个交付可执行文件的 15 MB 限制。
- Rust 1.89 MSRV 构建。
- 已提交的头文件 `crates/ffi/include/agent_desktop.h` 与 Rust ABI 之间的漂移。用 `scripts/update-ffi-header.sh` 重新生成，该脚本需要 cbindgen 0.29.4。
- `deny.toml` 中的 `cargo-deny` 策略。
- 密钥扫描、CodeQL 和工作流 lint。

## Pre-commit hook

`.githooks/pre-commit` 镜像 CI 门禁。脚本是权威列表。对于暂存了 `.rs` 或 `.toml` 文件的提交，hook 会在主机的包集合上运行以下内容：

- `scripts/check-rust-file-size.sh`、`scripts/check-architecture.sh`、`scripts/check-no-phase-references.sh` 和 `scripts/check-stale-ref-constructor-misuse.sh`
- `cargo fmt --all -- --check`
- 带 `-D warnings` 的 Clippy
- `--lib` 测试

当提交涉及 `crates/ffi/` 时，hook 还会运行 stub 适配器透传测试和 cbindgen 头文件漂移检查。未安装 cbindgen 0.29.4 时会跳过漂移检查。CI 仍会运行它。

没有 Rust 或 FFI 变更、但涉及 `scripts/`、`.githooks/`、`.github/workflows/` 或 `tests/e2e-linux/` 的提交，会运行 Bash 3 兼容性和 no-phase-reference 检查。

仅在紧急情况下绕过 hook：

```bash
git commit --no-verify
# or
SKIP_PRECOMMIT=1 git commit -m "..."
```

## 工程规则

[AGENTS.md](AGENTS.md) 包含面向人员和编码 agent 的工程规则。它们涵盖 Conventional Commits、每个 Rust 文件 400 行上限、禁止内联注释、交付源码中不得引用交付计划、每个文件一个领域类型、错误处理、核心隔离以及添加命令的步骤。hook 和 CI 强制执行其中大部分规则。

Skill 保持同步。新增命令或更改 CLI 标志必须更新对应的 skill。跨平台行为放在 `skills/agent-desktop/`。平台行为放在 `skills/agent-desktop-macos/`、`skills/agent-desktop-windows/` 或 `skills/agent-desktop-linux/`。C ABI 变更放在 `skills/agent-desktop-ffi/`。二进制文件会嵌入这些 skill，agent 会读取它们，因此它们必须与代码一致。

## 提交 Pull Request

1. 从 `main` 拉出分支，并向 `main` 发起 PR。这适用于所有平台、核心、FFI 和 Linux 工作。
2. 每个 PR 只做一项逻辑变更。将重构与功能分开。
3. 推送前在本地通过每项质量门禁。
4. 使用 Conventional Commit 标题。squash 合并会将其用作提交信息。
5. 在描述中用 `Fixes #N` 或 `Refs #N` 关联相关 Issue。
6. 如果更改了 crate 依赖，检查 `cargo tree -p agent-desktop-core` 是否仍然干净。
7. CI 在每次推送时运行。CI 变红会阻止合并。

## 发布源码和诊断证据

从经过审查的 Git 修订或通过发布工作流发布。切勿上传工作目录或本地内部文档归档。被忽略的文件可能包含 agent 设置、对话记录、截图和个人文件路径。在将浏览器截图和演示录制加入 Git 之前先审查。忽略规则只能阻止意外暂存。它们不会清理文件，也不会阻止强制添加。

发布前运行密钥和隐私扫描。它是使用 `.gitleaks.toml` 的 gitleaks。其回归检查是 `node scripts/check-gitleaks-policy.mjs ./gitleaks`。凭据在 lockfile、fixture 以及同时包含安全示例值的行中必须保持可检测。用肉眼审查媒体。文本扫描器无法证明视频、截图或音轨是安全的。

`.gitleaksignore` 包含旧 E2E 计划中两个 Windows 机器 SID 的精确指纹。这些 SID 不在当前树中，也不是凭据。这些条目记录了 SID 存在于已发布历史中。它们不会抹除该历史，也不覆盖新的 SID、其他提交或真实凭据。不要用宽泛的路径或规则排除来替换它们。

提交会保留作者和提交者的姓名与邮箱。要将个人地址排除在未来的提交之外，请将 Git 邮箱设置为已验证的 GitHub noreply 地址。更改本地 Git 配置不会重写历史，现有归属也不是凭据泄露。不要为了清除审计警告而重写共享历史。

## 报告 Bug 和安全问题

Bug 和功能请求：使用 [Issue 模板](https://github.com/vercel-labs/agent-desktop/issues/new/choose)。[SUPPORT.md](SUPPORT.md) 列出了应包含的内容。

安全漏洞：不要公开开 Issue。遵循 [SECURITY.md](SECURITY.md)。

## 使用 AI 编码 agent 写作

如果 AI 编码 agent 在你的贡献中撰写任何文字（文档、README、skill、变更日志条目、提交信息、PR 描述或 Issue），它必须使用 [`technical-writer` skill](.agents/skills/technical-writer/SKILL.md)。该 skill 设定仓库的语气，并要求 agent 对照源码核实技术声明。AGENTS.md 要求 agent 使用它。

## 许可证

贡献按覆盖本项目的 [Apache-2.0 许可证](LICENSE)授权。