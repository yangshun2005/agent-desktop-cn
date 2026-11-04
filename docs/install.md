# 安装 agent-desktop

使用 npm、pnpm、Yarn 或 Bun 安装 `agent-desktop` npm 包。它需要 Node 18 或更高版本，并捆绑了适用于 macOS、Windows 和 Linux（x64 和 ARM64）的原生可执行文件和平台辅助程序：

```bash
npm install -g agent-desktop
agent-desktop version
```

不加 `-g` 时，命令会安装到 `./node_modules/.bin` 而非 PATH。用 `npx agent-desktop` 运行，或全局安装。

## Homebrew

在 macOS 和 Linux 上，从 Homebrew 默认目录安装：

```bash
brew install agent-desktop
agent-desktop version
```

Formula 位于 `Homebrew/homebrew-core`。它从带标签的源码构建，将平台辅助程序安装在 CLI 旁边，由 Homebrew 管理自己的 bottle。Homebrew 的自动化会拾取新的发布标签，因此新版本可能在 npm 和 GitHub Releases 之后才到达 Homebrew。用 `brew upgrade agent-desktop` 升级。

## Scoop 和旧版安装

目前没有官方 Scoop 包。本仓库不再维护自定义 Homebrew tap 或 Scoop bucket；现有的自定义安装仍固定在其已安装的版本上。

如果旧版安装遮蔽了该命令，请通过其原始包管理器卸载。在 macOS/Linux 上用 `command -v agent-desktop` 检查解析结果，在 Windows 上用 `Get-Command agent-desktop`。

## 安装脚本被阻止

npm 包的 `postinstall` 脚本会用指向原生二进制的链接替换 Node 启动器。某些包管理器会阻止安装脚本，npm 12 默认如此。此时每次调用都通过 Node 启动器运行，增加约 125 毫秒。启动器会向 stderr 打印一次性提示。允许该脚本并重新安装：

```bash
npm install -g agent-desktop --allow-scripts=agent-desktop
```

对于项目内安装，在项目内运行 `npm install-scripts approve agent-desktop`，然后运行 `npm rebuild agent-desktop`。approve 命令只编辑项目的 `package.json`，对全局安装会失败。

## 发布归档

从 [GitHub Releases](https://github.com/vercel-labs/agent-desktop/releases) 下载适用于你的操作系统和架构的归档。将可执行文件及其平台辅助程序解压到 PATH 上的同一目录。升级时，用同一发布版本中的文件替换这两个文件。

发布版本的 `checksums.txt` 能发现损坏的下载，但不能证明归档的来源。用构建证明验证其来源：

```bash
gh attestation verify <downloaded-archive> --repo vercel-labs/agent-desktop
```

## 从源码构建

你需要 `rust-toolchain.toml` 中固定的 Rust 工具链（1.89.0）。`rustup` 会在首次使用时安装它。你还需要 C 工具链：macOS 13 或更高版本上的 Xcode 命令行工具、Windows 上的 MSVC 构建工具，或 Linux 上的 `build-essential`。

```bash
git clone https://github.com/vercel-labs/agent-desktop && cd agent-desktop
cargo build --release -p agent-desktop
```

每个平台 crate 只能在其自身的操作系统上构建。为你的平台构建辅助程序。源码构建会将其放在 `target/release/` 中，与 CLI 相邻。Windows 没有辅助程序。

```bash
cargo build --release -p agent-desktop-macos --bin agent-desktop-macos-helper   # macOS
cargo build --release -p agent-desktop-linux --bins                             # Linux
```

CLI 会在其自身目录中查找辅助程序，因此请将两者一起移动。

## 权限

**macOS** 需要辅助功能权限。截图还需要屏幕录制权限。某些命令需要针对 System Events 的自动化权限，并在缺失时报告。`agent-desktop permissions` 报告状态而不弹出提示。`permissions --request` 显示系统提示。`permissions --guide` 打开系统设置，并显示辅助功能和屏幕录制的屏幕指南。

**Windows** 对与终端完整性级别相同的应用无需授权。以管理员身份运行的应用可以被读取，但只有提权的 agent 才能向其发送输入。该二进制文件未签名。从浏览器下载时 SmartScreen 可能发出警告，杀毒软件可能将其隔离。在干净的 Windows 11 安装上，Smart App Control 会阻止它，任何下载方式或启动模式都无法避免。关闭 Smart App Control 才能运行它。npm 安装不会触发 SmartScreen。要恢复被隔离的文件，请先对照发布版本检查其 SHA-256 校验和。

**Linux** 需要 Ubuntu 24.04 或更高版本，运行于 x86_64 或 aarch64，glibc 最低 2.35。Ubuntu 22.04 的 AT-SPI 桥接在 `Action.GetActions` 上会中止。在用户的图形会话内运行 CLI，并安装 `at-spi2-core`。TTY、SSH shell 或缺少该会话总线、显示和运行时环境的容器看不到任何应用。`agent-desktop status` 会激活并检查辅助功能总线。`supported_surfaces` 列表为空表示总线未响应。

按键在 X11 上需要 XTEST，在 Wayland 上需要 GNOME Mutter RemoteDesktop。Wayland 剪贴板需要 `wl-clipboard`。其余部分，请在 Linux 机器上运行 `agent-desktop skills get platform`。

## 平台说明

**Windows。** `list-surfaces` 列出每个进程的 `window`、`focused`、`sheet` 和 `menu` 表面。`snapshot --surface` 在无 `--app` 时还会读取 `taskbar`、`system-tray`、`system-tray-overflow`、`start-menu` 和 `action-center`。`quick-settings` 返回 `PLATFORM_NOT_SUPPORTED`，并指出 `action-center` 是承载这些控件的表面。

`launch` 接受绝对路径，或 System32 或 Windows 目录下的裸名称，带或不带 `.exe`。它不会解析 "Google Chrome" 之类的显示名称。用 `list-apps` 查找正在运行的应用。

**Linux。** 语义操作在 X11 和 Wayland 上均可用。在 Wayland 上，应用指针输入返回 `ACTION_NOT_SUPPORTED`，原生窗口的窗口控制会被拒绝。