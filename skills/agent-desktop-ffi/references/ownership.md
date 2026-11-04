# 指针所有权

FFI 交给你的每个需由你持有的指针都有对应的释放函数。务必调用它：内存由 Rust 分配，C 的 `free()` 无法释放。借用指针（如 `ad_last_error_*` 字符串）永远不需要释放，其生命周期约定见 error-handling.md。

## 分配 / 释放对照表

### 命令返回的 JSON 字符串

这些入口点将需由调用方持有的、以 NUL 结尾的 JSON 信封写入 `*out`。用 `ad_free_string` 释放。发生基础设施错误时 `*out` 保持为 null（见 error-handling.md 中的"双重失败模式"）。

| 分配函数                                                                          | 释放函数                |
|-----------------------------------------------------------------------------------|-------------------------|
| `ad_version(&out)`                                                                | `ad_free_string(out)`   |
| `ad_status(adapter, &out)`                                                        | `ad_free_string(out)`   |
| `ad_snapshot(adapter, app, surface, max_depth, interactive_only, compact, &out)` | `ad_free_string(out)`   |
| `ad_execute_by_ref(adapter, ref_id, action, policy, &out)`                       | `ad_free_string(out)`   |
| `ad_execute_by_ref_timeout(adapter, ref_id, action, policy, timeout_ms, &out)`   | `ad_free_string(out)`   |
| `ad_wait(adapter, args, &out)`                                                    | `ad_free_string(out)`   |
| `ad_trace_show(adapter, limit, event_prefix, &out)`                               | `ad_free_string(out)`   |
| `ad_trace_export(adapter, limit, out_path, &out)`                                 | `ad_free_string(out)`   |

### 适配器生命周期

| 分配函数                                             | 释放函数                                |
|------------------------------------------------------|-----------------------------------------|
| `ad_adapter_create()`                                | `ad_adapter_destroy(adapter)`           |
| `ad_adapter_create_with_session(session)`            | `ad_adapter_destroy(adapter)`           |

### 不透明列表句柄

| 分配函数                                                     | 释放函数                                |
|--------------------------------------------------------------|-----------------------------------------|
| `ad_list_apps(adapter, &list)`                               | `ad_app_list_free(list)`                |
| `ad_list_displays(adapter, &list)`                           | `ad_display_list_free(list)`            |
| `ad_list_windows_exact(adapter, app, focused, &list)`        | `ad_exact_window_list_free(list)`       |
| `ad_list_surfaces_exact(adapter, pid, &list)`                | `ad_exact_surface_list_free(list)`      |
| `ad_list_notifications(adapter, filter, &list)`              | `ad_notification_list_free(list)`       |
| `ad_dismiss_all_notifications(adapter, f, &ok, &fail)`       | 对每个分别调用 `ad_notification_list_free`，或调用 `ad_dismiss_all_notifications_free(ok, fail)` |

### 应用 / 窗口生命周期

| 分配函数                                                  | 释放函数                                                            |
|-----------------------------------------------------------|---------------------------------------------------------------------|
| `ad_launch_app_exact(adapter, id, timeout, &out)`         | `ad_release_exact_window_fields(&out)`                              |

### 原始树与元素访问

| 分配函数                                                                               | 释放函数                                |
|----------------------------------------------------------------------------------------|-----------------------------------------|
| `ad_get_tree_exact(adapter, win, opts, &out)`                                          | `ad_free_tree(&out)`                    |
| `ad_find_exact(adapter, win, query, &handle)`                                          | `ad_free_handle(adapter, &handle)`：将 `handle.ptr` 置零，后续调用即为空操作 |

`ad_resolve_element_exact` 不分配任何内存：它总是以 `AD_RESULT_ERR_ACTION_NOT_SUPPORTED` 失败，并保持 `handle.ptr` 为 null。

### 操作结果

| 分配函数                                                                               | 释放函数                     |
|----------------------------------------------------------------------------------------|------------------------------|
| `ad_execute_action(adapter, handle, action, &out)`                                     | `ad_free_action_result(&out)` |
| `ad_execute_action_with_policy(adapter, handle, action, policy, &out)`                 | `ad_free_action_result(&out)` |
| `ad_notification_action(adapter, &request, &out)`：从 `ad_list_notifications` 设置 `request.identity`，并选择显式的非 headless 策略；顺序不匹配会失败关闭 | `ad_free_action_result(&out)` |

`ad_execute_ref_action_exact_with_policy` 不分配任何内存：它总是以 `AD_RESULT_ERR_ACTION_NOT_SUPPORTED` 失败，并保持 `out` 为零值。

### 剪贴板与图像缓冲区

| 分配函数                                    | 释放函数                                                            |
|---------------------------------------------|---------------------------------------------------------------------|
| `ad_get_clipboard(adapter, &text)`          | `ad_free_string(text)`                                              |
| `ad_get(adapter, handle, property, &text)`  | `ad_free_string(text)`：属性不存在时 text 可能为 null；`ad_free_string(NULL)` 是空操作 |
| `ad_screenshot(adapter, target, &buf)`      | `ad_image_buffer_free(buf)`（buf 不透明；通过 `ad_image_buffer_{data,size,width,height,format}` 读取） |

## 规则

- 每个释放函数都**容忍 null**。`ad_free_tree(NULL)`、`ad_free_handle(adapter, NULL)`、`ad_free_string(NULL)` 等均为空操作。列表访问器（`ad_*_list_count`、`_get`）同样接受 null，分别返回 `0` / `NULL`。`ad_free_handle` 先检查适配器，因此即使句柄为 null，适配器为 null 时也返回 `AD_RESULT_ERR_INVALID_ARGS`。
- **对列表句柄和 `AdImageBuffer` 重复释放是未定义行为。** 不透明包装由 `Box::into_raw` 分配；第二次调用会对已释放的内存执行 `Box::from_raw`。释放后务必将指针置为 `NULL`。
- **`ad_free_handle` 可安全重复调用**：它在平台释放后将 `handle.ptr` 置零，后续调用看到 `NULL`，直接返回 `AD_RESULT_OK`，不会再次进入 `CFRelease`。
- **先释放句柄，再销毁其适配器。** `ad_execute_action*` 需要产生该句柄的存活适配器，且必须在同一线程。`ad_adapter_destroy` 不会释放未归还的句柄。`ad_free_handle` 只比较适配器指针值，因此销毁后传入同一指针仍会成功；在此之前句柄一直保持已分配状态。适配器错误、跨线程、伪造或未置零的已释放副本会返回 `AD_RESULT_ERR_INVALID_ARGS`，保持结构体不变，且绝不解引用外部内存。
- 结构体内的指针（`.id`、`.title`、`.app_name`、每个 `AdNotificationInfo.body` 等）由该结构体的所属释放函数（list_free / release_fields）释放；不要单独对它们调用 `ad_free_string()`。
- `AdActionResult` 拥有 `action`、`ref_id`、`post_state`、`post_state.states`、`steps`，以及每个 `steps[i].label` / `steps[i].outcome`；只能通过 `ad_free_action_result(&out)` 释放全部。将返回的计数视为只读元数据；释放未修改的结果结构体。
- 释放后所有权**不会**转回 Rust。保留一个本地 `NULL` 以防意外复用。

## 输出参数置零

每个可能失败的 FFI 函数都会在任何守卫（指针校验、UTF-8 校验、枚举校验）**之前**将其输出参数置零。出错时调用配对的释放函数是安全的：内部所有指针保证为 null，所有计数为零，因此释放是空操作，而不是对前一个调用方分配的重复释放。

具体而言：

- `ad_get_clipboard` 在调用适配器前写入 `*out = NULL`：出错时不会看到陈旧缓冲区。
- `ad_launch_app_exact` 在调用平台前写入 `*out = zeroed AdExactWindowInfo`；对零初始化结构体调用 `ad_release_exact_window_fields(&out)` 是空操作。
- `ad_screenshot` 在分配图像缓冲区前写入 `*out = NULL`：截图失败时不会留下陈旧指针。
- `ad_snapshot`、`ad_execute_by_ref`、`ad_execute_by_ref_timeout`、`ad_wait`、`ad_status`、`ad_version`、`ad_trace_show`、`ad_trace_export` 在任何守卫之前写入 `*out = NULL`，因此基础设施失败时不会发生分配，`ad_free_string(NULL)` 是安全的空操作。
- `ad_*_list` 和 `ad_find_exact` 对其句柄 / 列表输出参数应用同样的模式。