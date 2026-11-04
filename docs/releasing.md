# 发布

在一个 Pull Request 中更新 Cargo 工作区版本、锁文件、npm 源版本和变更日志。
合并到 `main` 后，`Cargo.toml` 的变更会触发已有的 Release 工作流。
已发布版本未变则跳过发布。仍可手动触发以重试。

该工作流构建并检查原生 CLI 和 FFI 归档，将 CLI 可执行文件打包进 npm，
通过可信发布（OIDC）发布 npm，并使用内置的 `GITHUB_TOKEN` 发布 GitHub 归档、校验和与来源证明。
无需额外的发布令牌或包更新提交。
npm 任务在 `Release` 环境中运行，因此发布前需由审核者在工作流运行中批准。

Homebrew formula 位于 `Homebrew/homebrew-core`，从发布标签的源码归档构建。
Homebrew 的 autobump 自动化会检测新标签并更新 formula，因此发布无需 Homebrew 步骤。
Homebrew 上架可能晚于 npm 和 GitHub 发布。
没有官方 Scoop 包；在 `ScoopInstaller/Main` 接受之前，不要宣传 Scoop 安装方式。