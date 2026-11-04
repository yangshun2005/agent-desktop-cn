# 更新日志

## [1.1.0](https://github.com/vercel-labs/agent-desktop/compare/v1.0.2...v1.1.0) (2026-11-02)

### ⚠ 破坏性变更

* C ABI 主版本号现为 7。`ad_resolve_element_exact` 和 `ad_execute_ref_action_exact_with_policy` 一律以 `ACTION_NOT_SUPPORTED`（`details.kind: exact_entry_unsupported`）拒绝：C 调用方无法提供 ref 现在解析所需的捕获证明，因此请改用快照 ref 执行操作。([#442](https://github.com/vercel-labs/agent-desktop/pull/442)) by @lahfir
* 响应信封升级至 2.6。在 Windows 和 Linux 上，指向多个相同控件之一的 ref、来自快照无法完整读取的窗口的 ref，以及来自更早快照的 ref，现在返回 `AMBIGUOUS_TARGET` 并附带 `details.kind: identity_not_unique`，而此前其中部分会执行操作。无法完整读取窗口以证明其 ref 的快照现在通过 `data.identity_unproven: true` 表明这一点。参见 `docs/json-output.md` 中的 2.6 版本迁移说明。([#442](https://github.com/vercel-labs/agent-desktop/pull/442)) by @lahfir
* 操作系统不存在的 surface 类型现在返回 `INVALID_ARGS` 而非 `PLATFORM_NOT_SUPPORTED`，并指明应使用的类型。FFI 调用方会看到对应的 `AdResult` 代码。完全没有快照 surface 的适配器仍返回 `PLATFORM_NOT_SUPPORTED`。([#480](https://github.com/vercel-labs/agent-desktop/pull/480)) by @lahfir

### 新功能

* 通过 `snapshot --surface dock` 和 `--surface menu-bar-extras` 访问 macOS Dock 和菜单栏状态项。配合 `--app` 时，`menu-bar-extras` 读取该应用自身的状态项。`open-system-surface` 立即返回任一 surface，无头运行，其数据中不含 `window`。([#480](https://github.com/vercel-labs/agent-desktop/pull/480)) by @lahfir
* 通过进程观察没有窗口的 macOS 后台应用，例如菜单栏应用、Dock，或控制中心与通知中心面板。其快照省略 `data.window`。`--app` 也匹配应用的可执行文件名或 bundle ID，因此 `--app ControlCenter` 可用。([#480](https://github.com/vercel-labs/agent-desktop/pull/480)) by @lahfir
* 新增 `type --interval-ms`，每 N 毫秒（1 至 2000）输入一次按键，并在最后一次按键后校验字段。([#348](https://github.com/vercel-labs/agent-desktop/pull/348)) by @ctate
* 在 macOS 和 Windows 上，`type` 和 `set-value` 之后在光标叠加层中显示插入符。([#349](https://github.com/vercel-labs/agent-desktop/pull/349), [#350](https://github.com/vercel-labs/agent-desktop/pull/350)) by @ctate
* 在 macOS 和 Windows 上，将光标标签绘制为按文本大小适配的黑色胶囊。([#357](https://github.com/vercel-labs/agent-desktop/pull/357)) by @ctate
* 光标主体默认使用强调色。`--fill` 可设置不同的主体颜色。([#361](https://github.com/vercel-labs/agent-desktop/pull/361)) by @ctate
* 在 Windows 上绘制 `--image` 和 `--pointer-image` 光标。([#345](https://github.com/vercel-labs/agent-desktop/pull/345)) by @ctate
* 无头操作在获取交互租约之前开始光标移动，使并行代理的光标同时移动。([#347](https://github.com/vercel-labs/agent-desktop/pull/347)) by @ctate

### 问题修复

* 移除 macOS 命令宿主。每次 CLI 调用都在自己的进程中运行，与 Windows 和 Linux 一致。npm 安装和每次 CLI 调用都会停止早期版本遗留的命令宿主。在跳过 npm 脚本的安装或手动安装之后，运行一次 `agent-desktop version`。([#414](https://github.com/vercel-labs/agent-desktop/pull/414)) by @lahfir
* 本版本之前拍摄的快照中的 ref 不携带捕获证明，因此请重新拍摄快照。在 Windows 和 Linux 上它们返回 `AMBIGUOUS_TARGET` 并附带 `details.kind: identity_not_unique`；在 macOS 上，只要其令牌所指的元素仍存活，ref 仍可解析。([#442](https://github.com/vercel-labs/agent-desktop/pull/442)) by @lahfir
* UI 变化后，ref 不会作用于不同的元素。在 macOS 上，ref 跟随其自身元素：指向已移除行的 ref 返回 `STALE_REF`，相同控件可各自通过 ref 使用。在 Windows 和 Linux 上，仅当 ref 在捕获时唯一且当前恰好有一个元素匹配时才解析；指向多个相同控件之一的 ref 返回 `AMBIGUOUS_TARGET`。身份是元素的角色和稳定字段，而非其实例：替换原元素的相同元素会被作用，而仍列出已移除控件的工具包（Linux 上的 GTK4、Qt6）仍可能收到该操作。([#441](https://github.com/vercel-labs/agent-desktop/pull/441), [#442](https://github.com/vercel-labs/agent-desktop/pull/442)) by @lahfir
* 在 macOS 上，对状态项或后台应用控件的 `--headed` 点击发送真实点击，而不会先聚焦应用没有的窗口。([#480](https://github.com/vercel-labs/agent-desktop/pull/480)) by @lahfir
* 在 macOS 上，当焦点在被按元素窗口内移动或聚焦元素发生变化时，返回码无法证明任何结果的按压计为已送达。Notes 的 New Note 在创建笔记后不再报告 `ACTION_FAILED`。([#480](https://github.com/vercel-labs/agent-desktop/pull/480)) by @lahfir
* 在 macOS 上，对步进控件的 `set-value` 在每一步后重新读取值再放弃，并在步骤无法达到目标时说明。([#480](https://github.com/vercel-labs/agent-desktop/pull/480)) by @lahfir
* 按键后保持叠加层插入符位于插入点。在 macOS 上，将其置于缩进的新行。([#353](https://github.com/vercel-labs/agent-desktop/pull/353), [#354](https://github.com/vercel-labs/agent-desktop/pull/354)) by @ctate
* 在 macOS 上，叠加层控件到达时即绘制，隐藏已清除的标签，并停止裁剪点击高亮的辉光。([#346](https://github.com/vercel-labs/agent-desktop/pull/346), [#352](https://github.com/vercel-labs/agent-desktop/pull/352)) by @ctate
* 在 Windows 上，停止每条命令以崩溃报告结尾以及退出时约 3 秒的错误报告。([#342](https://github.com/vercel-labs/agent-desktop/pull/342)) by @ctate
* 在 Windows 上，将 CR LF 作为单个换行输入。([#368](https://github.com/vercel-labs/agent-desktop/pull/368)) by @Railly
* 在 Windows 上，停止对叠加层坐标的二次缩放（该问题在 200% 缩放下使光标移出屏幕），并停止叠加层启动时闪现黑方块。([#343](https://github.com/vercel-labs/agent-desktop/pull/343), [#344](https://github.com/vercel-labs/agent-desktop/pull/344)) by @ctate

## [1.0.2](https://github.com/vercel-labs/agent-desktop/compare/v1.0.1...v1.0.2) (2026-10-30)

### 新功能

* 当 Node 启动器拖慢每次调用时告知 npm 用户。当 npm 阻止安装脚本时，启动器向 stderr 打印一次性通知，并附上链接原生二进制的命令。

### 问题修复

* 使 macOS 上的 `permissions --guide` 提供启动 agent-desktop 的应用（例如 LaunchServices 不认识的终端），而非 agent-desktop 二进制。
* 命令宿主报告缺少权限后重启它，使用户授予访问后命令成功。
* 将权限提示和技能指向 `permissions --guide` 而非 `--request`。
* 将 npm 包和仓库链接指向 vercel-labs/agent-desktop。

## [1.0.1](https://github.com/vercel-labs/agent-desktop/compare/v1.0.0...v1.0.1) (2026-10-27)

### 问题修复

* 从技能中移除 `version` 行。1.0.0 中它仍写着 0.11.1；`agent-desktop version` 报告版本。
* 当 main 上的 Cargo 工作区版本变化时自动发布 npm 和 GitHub 发布版本。
* 移除自定义 Homebrew tap、Scoop bucket 及其包更新 Pull Request。

## [1.0.0](https://github.com/vercel-labs/agent-desktop/compare/v0.11.1...v1.0.0) (2026-10-27)

### 发布

* 新增 Linux 支持：CLI、C 库和 npm 包现通过 AT-SPI 在 x86_64 和 aarch64 Linux（glibc 2.35 或更高）上运行。
* 将集成的 macOS、Windows 和 Linux CLI 及 C 库作为 1.0.0 发布。
* 在 npm 安装中捆绑原生 CLI 可执行文件和平台辅助程序，包括禁用生命周期脚本的安装。
* 通过手动发布工作流验证发布归档、校验和及构建来源。

## [0.10.1](https://github.com/vercel-labs/agent-desktop/compare/v0.10.0...v0.10.1) (2026-10-18)


### 问题修复

* **windows:** 允许 launch 接受不带 .exe 的裸名称 ([#276](https://github.com/vercel-labs/agent-desktop/issues/276)) ([483427e](https://github.com/vercel-labs/agent-desktop/commit/483427eb84821a9d0ebc79eb56dd689c04cb15be))

## [0.10.0](https://github.com/vercel-labs/agent-desktop/compare/v0.9.4...v0.10.0) (2026-10-18)


### ⚠ 破坏性变更

* 本发布版本包含以下契约变更。
    - C ABI 主版本号为 5。各自有 *_exact 替代的旧入口点被移除：ad_list_surfaces、ad_surface_list_count、ad_surface_list_get、ad_surface_list_free、ad_list_windows、ad_window_list_count、ad_window_list_get、ad_window_list_free、ad_launch_app、ad_release_window_fields、ad_get_tree、ad_find、ad_is、ad_focus_window、ad_window_op、ad_resolve_element 和 ad_execute_ref_action_with_policy，以及 AdSurfaceList 和 AdWindowList 句柄。AdScreenshotKind 移除 WINDOW（FULL_SCREEN 保留 2，1 被拒绝）；AdScreenshotTarget 移除 pid；AdExactSurfaceInfo 为版本 2，48 字节，带尾部未分类字段；ad_execute_by_ref 和 ad_execute_by_ref_timeout 的 snapshot_id 参数及 AdWaitPredicate.snapshot_id 字段被移除，因此 AdWaitPredicate 为 40 字节，AdWaitArgs 为 104。使用方必须针对重新生成的头文件重新构建。
    - 从所有接受 ref 的 CLI 命令、wait --element、snapshot --root、find --root、drag --from/--to、batch 条目和 FFI 中移除裸 @eN ref 输入。仅接受带快照限定的 @<snapshot_id>:eN，裸 ref 以 INVALID_ARGS 失败。仅用于限定裸 ref 的 --snapshot 标志从所有命令中移除，携带 snapshot 或 snapshot_id 字段的 batch 条目作为未知字段被拒绝。
    - get --property text 不再返回与 --property value 相同的值。它对 textfield、combobox、listbox、datefield 和 timefield 优先返回值，回退到可访问名称；其他所有角色优先返回可访问名称，回退到值。空字符串视为缺失。要保留先前答案，请显式请求 --property value。
    - 类型为 PermissionDenied 的 io 错误在 CLI 信封和 FFI 信封、返回码及最后错误中均返回 PERM_DENIED 而非 INTERNAL。
    - 元素已消失的 ref 操作不再以 TIMEOUT 结束。不可重试的解析失败立即返回；可重试的继续轮询，若后续无解析成功，则在截止时返回记录的 STALE_REF 或 AMBIGUOUS_TARGET。

### 新功能

* 在 x64 和 ARM64 上支持 Windows 10 1809+ 和 Windows Server 2019+ ([483a31d](https://github.com/vercel-labs/agent-desktop/commit/483a31d036175f54c34ce1fbd2cea762560ab78c))

## [0.9.4](https://github.com/vercel-labs/agent-desktop/compare/v0.9.3...v0.9.4) (2026-09-23)


### 问题修复

* 将 cargo lock 与 0.9.3 发布版本同步 ([#217](https://github.com/vercel-labs/agent-desktop/issues/217)) ([8c1502e](https://github.com/vercel-labs/agent-desktop/commit/8c1502e43c6c78fe856980f0bb7b3585f877d347))

## [0.9.3](https://github.com/vercel-labs/agent-desktop/compare/v0.9.2...v0.9.3) (2026-09-22)


### 问题修复

* 加固原生可访问性定位与投递 ([#210](https://github.com/vercel-labs/agent-desktop/issues/210)) ([882c4c4](https://github.com/vercel-labs/agent-desktop/commit/882c4c4695c0db7d75b83f2655ece178591deb6c))

## [0.9.2](https://github.com/vercel-labs/agent-desktop/compare/v0.9.1...v0.9.2) (2026-09-17)


### 问题修复

* 使无头桌面操作报告实际发生的情况，并新增 jev-desktop ([#206](https://github.com/vercel-labs/agent-desktop/issues/206)) ([0691eba](https://github.com/vercel-labs/agent-desktop/commit/0691eba25f6a7082b197bc73ad5daa0e3315c976))

## [0.9.1](https://github.com/vercel-labs/agent-desktop/compare/v0.9.0...v0.9.1) (2026-09-13)


### 问题修复

* 停止检查器关闭测试因预期重置而失败 ([22cad9e](https://github.com/vercel-labs/agent-desktop/commit/22cad9ec21136562ed191a8cfb827dc22378c554))
* **drag:** 默认将 `--wait-for` 限定到放置目标，配合 `--wait-for-scope from|to` ([cefc80a](https://github.com/vercel-labs/agent-desktop/commit/cefc80a89c95b21061573e366e3832bff6baa231))
* 继承的截止时间到期时保留 `wait_timeout` 信封 ([07913a7](https://github.com/vercel-labs/agent-desktop/commit/07913a7cdb85d79e1708063fd398c9f8571894e9))
* 防止 `wait --count` 在不完整快照上误报 ([319e9c6](https://github.com/vercel-labs/agent-desktop/commit/319e9c6c2e7c175fdec42d19b2f1614b9238a909))
* 在 trace 键脱敏中拆分首字母缩写边界 ([3042e0c](https://github.com/vercel-labs/agent-desktop/commit/3042e0c7a823508fd5417584f0459d428832c8a0))
* **cursor:** 在多代理模式下控制 `agent_id` 标记 ([58612ea](https://github.com/vercel-labs/agent-desktop/commit/58612eac673a45634e00a651872da043297e68d3))
* 在等待基线预捕获前跳过已结束会话的 batch 条目 ([899ce27](https://github.com/vercel-labs/agent-desktop/commit/899ce2702a68716a7704b7403050dd7f54161208))
* 为键盘和剪贴板操作恢复光标叠加层生命周期 ([c827cf9](https://github.com/vercel-labs/agent-desktop/commit/c827cf965