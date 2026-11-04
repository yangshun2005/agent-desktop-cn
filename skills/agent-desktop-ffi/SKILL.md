---
name: agent-desktop-ffi
tags: ffi, c-bindings, cdylib, python, swift, node, go, rust-ffi
requirements:
  - agent-desktop-ffi
description: >
  基于 agent-desktop 的 PlatformAdapter 的 C-ABI 绑定。调用方
  （Python ctypes、Swift、Node ffi-napi、Go cgo、C++、Ruby fiddle）
  链接 libagent_desktop_ffi.{dylib,so,dll}，直接调用 `ad_*` 函数，
  无需每次调用都启动 CLI 二进制。标准 observe-act 工作流为：
  ad_init → ad_adapter_create[_with_session]
  → ad_snapshot → 读取限定 ref（@s8f3k2p9:e5）→ ad_execute_by_ref → ad_free_string
  → ad_adapter_destroy。
---

# agent-desktop-ffi

通过 C-ABI 直接访问与 CLI 相同的操作。使用 `release-ffi` profile 构建 cdylib：

```sh
cargo build --profile release-ffi -p agent-desktop-ffi
```

产物为 `target/release-ffi/libagent_desktop_ffi.dylib`（Linux 上为 `.so`，
Windows 上为 `agent_desktop_ffi.dll`），以及已提交的 C 头文件
`crates/ffi/include/agent_desktop.h`。默认 `release` profile 使用
`panic = "abort"`：`extern "C"` 函数内的 Rust panic 会让宿主进程 `SIGABRT`。
`release-ffi` 保留 `panic = "unwind"`，panic 捕获需要它。

四个参考主题，按需加载：

- [ownership.md](references/ownership.md)：FFI 返回的每个指针由谁分配、由谁释放。
- [error-handling.md](references/error-handling.md)：errno 风格的 last-error 契约、错误码、枚举校验、panic 边界。
- [threading.md](references/threading.md)：宿主线程契约、跨线程与跨进程的变更顺序、绑定到 adapter 的原生句柄、AXIsProcessTrusted 继承。
- [build-and-link.md](references/build-and-link.md)：预构建归档、ABI 握手、结构体大小校验、C 与 Python 示例。

## Observe-act 工作流（标准路径）

```
ad_init(AD_ABI_VERSION_MAJOR)                    // 校验头文件 ↔ dylib 匹配
adapter = ad_adapter_create_with_session("s1")   // 或 ad_adapter_create()
rc = ad_snapshot(adapter, "Finder", 0, 10, false, false, &json_out)
// 解析 json_out：在 data.tree 中定位快照限定的 ref
ad_free_string(json_out)
// 构建 action：
AdAction act = {0}; act.kind = AD_ACTION_KIND_CLICK;
rc = ad_execute_by_ref(adapter, "@s8f3k2p9:e5", &act, 0, &result_out)
ad_free_string(result_out)
ad_adapter_destroy(adapter)
```

`ad_snapshot` 返回 CLI 信封 `{version, ok, command, data}`。
`data.tree` 携带交互元素的快照限定 ref。信任树之前先读
`data.complete`：`false` 表示观测预算耗尽、树是部分的，并非错误。
`ad_snapshot` 始终以 `app` 的焦点窗口为目标；窗口定位、骨架模式和 `--root`
下钻未通过 ABI 暴露。裸 `@eN` ref 返回
`AD_RESULT_ERR_INVALID_ARGS`。

## 核心约束

- **ABI 握手。** 加载库后调用一次 `ad_init(AD_ABI_VERSION_MAJOR)`。不匹配返回
  `AD_RESULT_ERR_INVALID_ARGS`：中止。`ad_abi_version()` 返回库的 major 供显示。
  移除符号或布局变更时 major 递增；新增 `ad_*` 符号和错误码是增量的。
  固定你所链接的 `libagent_desktop_ffi` 确切版本。

- **会话 adapter。** `ad_adapter_create_with_session("session-id")` 将
  refmap 持久化限定到会话命名空间，类似 CLI `--session <id>`。ID 为 1
  到 64 个 ASCII 字母数字、`-` 或 `_` 字符。null ID 行为同
  `ad_adapter_create()`。无效 ID 返回 null（读取 `ad_last_error_*`）。

- **Ref 操作。** `ad_execute_by_ref` 是主要操作入口。它加载
  RefStore，严格解析 ref（快照无此 ref 或元素已变更时为 `STALE_REF`，
  多个候选时为 `AMBIGUOUS_TARGET`），运行实时可操作性预检（可见、稳定、启用、支持的操作、
  策略、可编辑），然后分发。禁用或不支持的目标在任何平台调用前失败；
  `AD_RESULT_ERR_ACTION_FAILED` 时检查报告为
  `ad_last_error_details()` 中的 JSON。预检前自动等待 5000 ms；
  `ad_execute_by_ref_timeout` 接受 `timeout_ms`（`-1` = 默认 5000 ms，`0` =
  单次）。`PressKey` 默认 `focus_fallback`；其他所有操作，
  包括 `TypeText`，默认 `headless`。传 `AD_POLICY_KIND_FOCUS_FALLBACK`
  (1) 允许焦点但不移动光标，或 `AD_POLICY_KIND_HEADED` (2) 允许
  物理光标和键盘投递。`ad_execute_ref_action_exact_with_policy`
  总是以 `AD_RESULT_ERR_ACTION_NOT_SUPPORTED` 失败：exact 条目没有
  保留的元素 token 和捕获证明，无法重新识别。
  改用 `ad_execute_by_ref`。

- **代际安全的直接 API。** 每个直接定位函数都接受
  进程代际证据：`AdExactRefEntry`、`AdExactWindowInfo`、
  `ad_list_windows_exact`、`ad_launch_app_exact` 及其他 `*_exact`
  符号。`AdRefEntry`、`AdWindowInfo` 和 `AdSurfaceInfo` 仅
  作为嵌入 exact 结构体的字段存在。`ad_list_surfaces_exact` 保留
  surface `id`。从 `AD_EXACT_SURFACE_INFO_VERSION` 2 起，`AdExactSurfaceInfo.unclassified`
  命名其探测无法读取的 surface 类型（例如 `sheet,menu`），或
  为 null。列出的类型旁缺失 `sheet` 或 `menu` 表示未知，而非
  不存在。仅 Windows adapter 填充它；macOS 和 Linux 返回 null。

- **原生句柄。** `ad_find_exact` 返回 `AdNativeHandle`。发布前会重新识别
  元素：将落地的句柄与所选条目记录的 role、原生 id、name 和 description
  比对，若锚定路径上被不同控件占据，则返回 `AD_RESULT_ERR_STALE_REF`
  而非句柄。
  `ad_resolve_element_exact` 接受 exact ref 条目，出于与 exact 操作调用
  相同的原因，总是以 `AD_RESULT_ERR_ACTION_NOT_SUPPORTED` 失败。
  `ad_execute_action`（headless，无预检）和
  `ad_execute_action_with_policy` 直接作用于句柄，绕过 ref 操作
  流水线。在所有平台上，于 `ad_adapter_destroy` 之前，在同一 adapter 和线程上
  用 `ad_free_handle(adapter, &handle)` 释放每个句柄。它会
  将 `handle.ptr` 置零，因此第二次调用是 no-op。

- **操作结果。** `AdActionResult.steps` 镜像 CLI `steps` 数组，
  用于激活链操作。每个 `AdActionStep` 有 `label` 和 `outcome` 字符串、
  `mechanism`（`AD_STEP_MECHANISM_SEMANTIC_API` = 1 或
  `AD_STEP_MECHANISM_PHYSICAL_SYNTHETIC` = 2，仅当 `has_mechanism` 时有效），
  以及 `verified`（仅当 `has_verified` 时有效）。`AdActionResult.disposition`
  携带与 `ad_last_error_delivery_semantics` 相同的投递/重试对。
  用 `ad_free_action_result(&out)` 释放结果。

- **显示器发现。** 使用 `AdScreenshotTarget.screen_index` 前先调用
  `ad_list_displays`。列表顺序即截图索引顺序。
  每个 `AdDisplayInfo` 携带稳定的 display ID、边界、主显示器标志和缩放。
  用 `ad_display_list_free` 释放列表。

- **错误。** 失败的调用设置线程局部 last-error 状态；在下一次失败调用前
  用 `ad_last_error_*` 访问器读取。`ad_last_error_details()`
  返回结构化 JSON（`ACTION_FAILED` 时为可操作性报告，`AMBIGUOUS_TARGET` 时为候选
  摘要），可能包含用户屏幕上的元素名称、值和
  窗口标题：不要将其写入共享日志。
  `ad_last_error_delivery_semantics` 报告失败的操作是否
  已投递、重试是否安全。无效枚举判别值返回
  `AD_RESULT_ERR_INVALID_ARGS`。见 [error-handling.md](references/error-handling.md)。

- **线程。** 入口点可从任意宿主线程调用。原生句柄
  留在创建它们的 adapter 和线程上；跨线程传递限定 ref。
  桌面变更获取与 CLI 相同的跨进程交互租约；读取不获取租约，携带有限
  截止时间。读/读、读/变更、变更/变更矩阵见
  [threading.md](references/threading.md)。

- **追踪。** 文件追踪（与 CLI `--trace` 相同的 JSONL 契约；段携带
  `event`、`ts_ms`、`seq` 和脱敏字段）仅在会话具有
  `trace: on` 的 manifest 时启用，该 manifest 须在 adapter 创建前由 `session start`
  （或磁盘上等效的 manifest）创建。仅调用 `ad_adapter_create_with_session`
  不写入任何内容。每个 OS 进程追加到一个段，
  `~/.agent-desktop/sessions/<id>/trace/<pid>-<procTs>.jsonl`，长驻宿主
  复用它。`ad_trace_show`（JSON 信封）和 `ad_trace_export`（自包含
  HTML 文件）读取 adapter 会话的合并时间线；`limit` `-1` = CLI 默认值。

- **日志回调。** `ad_set_log_callback(cb)` 注册一个进程级回调
  用于诊断，与会话 manifest 无关。它以 JSON 形式接收后续同一线程上
  `ad_*` 调用内发出的 `tracing` 事件；FFI
  从不安装或替换宿主进程的全局 tracing subscriber。`cb`
  接收 `int32_t` 级别（1=ERROR ... 5=TRACE）和仅在调用期间有效的
  `const char *msg`。传 `NULL` 注销。敏感字段
  值（password、token、text 等）替换为 `{"redacted":true}`。
  回调不得跨 C ABI 展开：在内部捕获 C++ 异常和 Rust panic，
  违反可能导致宿主中止。它可能同时从多个宿主
  线程触发，且在 `NULL` 注销后短暂触发：保持回调和
  其捕获的任何数据在进程生命周期内有效。

- **等待。** `ad_wait(adapter, args, &out)` 运行 CLI `wait` 命令。设置
  `mode.pause`（`AdOptionalU64`，ms）、`mode.element`、`mode.window`（等待具有该标题的
  窗口）、`mode.text`，或 `mode.surfaces.menu`、
  `.menu_closed`、`.notification` 之一；添加 `predicate` 组（`predicate`、
  `value`、`action`、`count`）和 `scope.timeout_ms` / `scope.app`。CLI 事件
  等待（`--event`、`--window-id`）未暴露。零初始化 `AdWaitArgs`，
  调用前对照 `AD_WAIT_ARGS_SIZE` / `ad_wait_args_size()` 校验结构体大小。
  输出为 `{version, ok, command, data}` 信封，用
  `ad_free_string` 释放。`ad_wait` 阻塞调用线程最多 `scope.timeout_ms` ms。
  并发的 `ad_adapter_destroy` 不会使进行中的等待失效。

- **文本输入隐私。** 在 macOS 上，focus-fallback 或 headed 文本插入可能
  短暂使用剪贴板处理非 ASCII 文本。对于敏感文本，当目标
  支持可设置值时，优先使用 `AD_ACTION_KIND_SET_VALUE` 配合
  `AD_POLICY_KIND_HEADLESS`。

- **`ad_get_tree_exact` 与 `ad_snapshot`。** `ad_get_tree_exact` 返回原始扁平
  广度优先树：无 ref、无 refmap 持久化、无 JSON 信封（用
  `ad_free_tree` 释放）。用于自定义遍历或 UI 检查。对于通过
  `ad_execute_by_ref` 驱动操作的 observe-act agent，从
  `ad_snapshot` 开始。