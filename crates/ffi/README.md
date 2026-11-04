# agent-desktop-ffi

基于 [`PlatformAdapter`](../../crates/core/src/adapter/mod.rs) trait 的 C-ABI 库。它为 C、C++、Python（ctypes）、Swift、Go（cgo）和 Node（ffi-napi）构建共享库。调用在进程内运行，程序无需为每条命令启动 CLI。

完整指南见 https://agent-browser.dev/desktop/ffi，涵盖链接、API、所有权和线程。面向 agent 的版本见 [`agent-desktop-ffi` skill](../../skills/agent-desktop-ffi/SKILL.md)。

## 构建

```bash
cargo build --profile release-ffi -p agent-desktop-ffi
```

产物位于 `target/release-ffi/`：macOS 上为 `libagent_desktop_ffi.dylib`，Linux 上为 `libagent_desktop_ffi.so`，Windows 上为 `agent_desktop_ffi.dll`。

使用 `--profile release-ffi`，不要用 `--release`。`release` profile 设置了 `panic = "abort"`，会静默关闭 `extern "C"` 边界处的 `catch_unwind` 捕获。`release-ffi` profile 保留 `panic = "unwind"`。

源码构建还需要库旁边的平台辅助程序：macOS 上为 `agent-desktop-macos-helper`，Linux 上为 `agent-desktop-linux-clipboard-owner`。

对于无需辅助功能权限的 CI 和测试，构建 stub 适配器。它对每次适配器调用都返回 `PLATFORM_NOT_SUPPORTED`。

```bash
cargo build --profile release-ffi -p agent-desktop-ffi --features stub-adapter
```

`--features stub-adapter` 用空操作替换真实平台适配器，对每次适配器调用都返回 `PLATFORM_NOT_SUPPORTED`。

## ABI 接口

| 分组 | 关键符号 |
|-------|-------------|
| ABI 握手 | `ad_abi_version()`, `ad_init(expected_major)` |
| 适配器生命周期 | `ad_adapter_create()`, `ad_adapter_create_with_session(id)`, `ad_adapter_destroy()` |
| 命令支持的 JSON 入口 | `ad_snapshot`, `ad_execute_by_ref`, `ad_execute_by_ref_timeout`, `ad_wait`, `ad_version`, `ad_status`, `ad_trace_show`, `ad_trace_export` |
| 日志回调 | `ad_set_log_callback(fn(level, msg))` — 将非结构化 tracing 输出到回调 |
| 结构化文件 trace | 与 CLI 相同的 JSONL 契约，由 `trace: on` 会话清单控制（无 ABI 变更）；创建会话适配器前先执行 `session start` |
| Errno 最后错误 | `ad_last_error_code()`, `ad_last_error_message()`, `ad_last_error_suggestion()`, `ad_last_error_platform_detail()` |
| 类型访问器 / 大小获取器 | `ad_*_size()`, `ad_*_list_{count,get,free}()`, `ad_image_buffer_*()` |
| 释放辅助函数 | `ad_free_string()`, `ad_free_handle()`, `ad_free_tree()`, `ad_free_action_result()` |

完整声明列表见 [`include/agent_desktop.h`](include/agent_desktop.h)。
使用方必须在 `dlopen` 后调用 `ad_init(AD_ABI_VERSION_MAJOR)`，以验证
加载的 dylib 与使用方编译时所用的头文件匹配。

直接窗口、surface 和 ref 定位使用带版本号的
`AdExactRefEntry`、`AdExactWindowInfo` 和 `AdExactSurfaceInfo` 结构体，
它们携带所观察身份的进程代次和 surface ID。
ABI 6 和 `AD_EXACT_REF_ENTRY_VERSION` 2 在 `AdExactRefEntry` 内的 `AdRefGeometry`
中新增 `bounds_space`：一个 `AD_BOUNDS_SPACE_*` 常量，指明 `bounds` 的坐标系，
macOS 和 Windows 上相对于屏幕，Linux 上相对于窗口。

ABI 7 使两个 exact-entry ref 调用
`ad_execute_ref_action_exact_with_policy` 和 `ad_resolve_element_exact` 以
`AD_RESULT_ERR_ACTION_NOT_SUPPORTED` 失败关闭：exact entry 不携带保留的
元素 token，也没有捕获证明，因此无法重新识别。请通过快照 ref 使用
`ad_execute_by_ref` 执行操作，并用 `ad_find_exact` 获取句柄。
`ad_execute_by_ref` 和 `ad_wait` 只接受快照限定的 ref
（`@<snapshot_id>:eN`）。

## 命令包装器

每个命令支持的 JSON 入口在 `src/commands/` 下都有一个规范模块。
构建脚本只配置 dylib 安装名，从不改写源文件。

## 线程安全（macOS）

适配器入口可在任意宿主线程运行。AX 和 CG 调用没有一刀切的主线程限制，Apple 明确说明
[`NSWorkspace.shared`](https://developer.apple.com/documentation/appkit/nsworkspace/shared)
可从任意线程安全使用。Cocoa 支持的路径会建立每线程自动释放池；FFI 初始化会启动一次
`NSThread`，以便 Cocoa 启用 Apple 为 POSIX 线程宿主记录的锁。

原生句柄更严格：解析、使用和释放每个句柄必须在同一线程、使用同一适配器。变更通过
规范交互租约跨进程串行化。完整证据和顺序契约见 FFI skill 的线程参考。

## 错误模型

每个返回 `AdResult` 的函数在失败时设置线程局部的最后错误状态。
用 `ad_last_error_code()` / `ad_last_error_message()` /
`ad_last_error_suggestion()` 读取。`ad_last_error_message()` 返回的指针
在同一线程下一次*失败*调用前保持有效——成功调用不会改动它（POSIX errno 语义）。

## 完整文档

`skills/agent-desktop-ffi/` — 构建与链接指南、所有权模型、线程规则和错误处理模式。