# 错误处理

FFI 采用 errno 风格的 last-error 模式。每个返回 `AdResult` 的函数成功时返回 `AD_RESULT_OK`（= 0），失败时返回负的错误码。失败发生时，线程局部的 last-error 状态会被填充；用 `ad_last_error_*` 访问器读取。

## 最小模式

```c
AdExactWindowInfo win;
AdResult rc = ad_launch_app_exact(adapter, "com.apple.finder", 5000, &win);
if (rc != AD_RESULT_OK) {
    const char *msg = ad_last_error_message();
    const char *sug = ad_last_error_suggestion();   // may be NULL
    fprintf(stderr, "launch_app failed (%d): %s\n", (int)rc, msg ? msg : "(no message)");
    if (sug) fprintf(stderr, "  suggestion: %s\n", sug);
    // no need to release the struct: out-param was zero-initialized
    return -1;
}
// ...use win...
ad_release_exact_window_fields(&win);
```

## Last-error 访问器

六个访问器读取调用线程的 last-error 槽位。前五个共享下面的每线程生命周期约定。`ad_last_error_delivery_semantics` 将普通值复制到调用方存储中，因此不涉及指针生命周期：

| 访问器                          | 返回值                                                        |
|---------------------------------|----------------------------------------------------------------|
| `ad_last_error_code()`          | 最后一次失败的 `AdResult` 码，或 `AD_RESULT_OK`    |
| `ad_last_error_message()`       | 人类可读的描述，或 null                            |
| `ad_last_error_suggestion()`    | 恢复提示，或 null                                         |
| `ad_last_error_platform_detail()` | 操作系统特定的诊断信息（AX 码、HRESULT、AT-SPI），或 null |
| `ad_last_error_details()`       | 结构化 JSON 详情，或 null：**敏感**（见下文）   |
| `ad_last_error_delivery_semantics(&sem)` | 为最后一次失败写入 `AdDeliverySemantics {delivery, retry}` 并返回 `AD_RESULT_OK`；未记录错误时两个字段均为 `UNKNOWN`（0）。`sem` 为 null 时返回 `AD_RESULT_ERR_INVALID_ARGS`。读取不会清除或替换槽位。 |

`delivery` 是 `AdDeliveryDisposition`：`AD_DELIVERY_DISPOSITION_UNKNOWN`（0）、`_NOT_DELIVERED`（1）、`_DELIVERY_UNCERTAIN`（2）、`_DELIVERED_UNVERIFIED`（3）、`_DELIVERED_VERIFIED`（4）。`retry` 是 `AdRetryDisposition`：`AD_RETRY_DISPOSITION_UNKNOWN`（0）、`_SAFE`（1）、`_UNSAFE`（2）。仅在 `_SAFE` 时重试。

`ad_last_error_details()` 返回带结构化上下文的 JSON 字符串：`ACTION_FAILED` 时的可操作性检查报告、`AMBIGUOUS_TARGET` 时的候选元素摘要、`wait` `TIMEOUT` 时最后观察到的状态等。详情可能包含用户屏幕上的元素名称、值和窗口标题。应视为敏感诊断信息，避免输出到共享日志。

## 生命周期约定

任何 `ad_last_error_*` 访问器返回的指针，在之后任意多次**成功**的 FFI 调用中保持有效。只有下一次**失败**的调用会轮换槽位，与 POSIX `errno` 和 `strerror` 相同。

```c
AdResult rc = ad_some_call(...);
const char *msg = ad_last_error_message();   // snapshot

ad_check_permissions(adapter);                // success
ad_check_permissions(adapter);                // success
printf("%s\n", msg);                          // still valid
```

之后的失败发生后，先前的指针可能悬空，因此要在下一次可能失败的调用之前读取。Last-error 是线程局部的：一个线程的失败不会影响另一个线程的槽位。

`ad_check_permissions` 不把 `Unknown` 视为成功。无法回答权限探测的桩适配器返回 `AD_RESULT_ERR_PLATFORM_NOT_SUPPORTED`。macOS 适配器仅在平台探测本身存在歧义时报告 `AD_RESULT_ERR_INTERNAL`；读取 `ad_last_error_*` 获取诊断信息。

## 错误码

数值在 ABI 上稳定。新错误码追加在末尾，现有值永不重新编号。要处理此列表之外的值，因为后续版本可能新增错误码。

| 名称                                  | i32   | 含义                                    |
|---------------------------------------|-------|--------------------------------------------|
| `AD_RESULT_OK`                        |   0   | 成功                                    |
| `AD_RESULT_ERR_PERM_DENIED`           |  -1   | 缺少辅助功能 / 输入权限   |
| `AD_RESULT_ERR_ELEMENT_NOT_FOUND`     |  -2   | Ref 解析 / 查找未命中                    |
| `AD_RESULT_ERR_APP_NOT_FOUND`         |  -3   | Bundle/PID 查找未命中                     |
| `AD_RESULT_ERR_ACTION_FAILED`         |  -4   | 动作已派发但被拒绝             |
| `AD_RESULT_ERR_ACTION_NOT_SUPPORTED`  |  -5   | 平台无法执行此动作        |
| `AD_RESULT_ERR_STALE_REF`             |  -6   | Ref 早于 UI 变更；需重新快照      |
| `AD_RESULT_ERR_WINDOW_NOT_FOUND`      |  -7   | 窗口过滤条件未匹配到任何窗口              |
| `AD_RESULT_ERR_PLATFORM_NOT_SUPPORTED`|  -8   | 此操作系统上 API 不可用                 |
| `AD_RESULT_ERR_TIMEOUT`               |  -9   | 等待超过截止时间                     |
| `AD_RESULT_ERR_INVALID_ARGS`          | -10   | 空指针、错误枚举、无效 UTF-8      |
| `AD_RESULT_ERR_NOTIFICATION_NOT_FOUND`| -11   | 通知索引越界或已重排 |
| `AD_RESULT_ERR_INTERNAL`              | -12   | 内部失败或捕获到 Rust panic      |
| `AD_RESULT_ERR_SNAPSHOT_NOT_FOUND`    | -13   | 请求的快照 ref 存储缺失    |
| `AD_RESULT_ERR_POLICY_DENIED`         | -14   | 当前动作策略阻止此回退 |
| `AD_RESULT_ERR_AMBIGUOUS_TARGET`      | -15   | 严格重新识别找到多个候选；需重新快照 |
| `AD_RESULT_ERR_APP_UNRESPONSIVE`      | -16   | 不确定的变更后只读存活探测失败 |

## Ref 令牌校验

接受 ref 的入口点只接受一种规范形式：

- `@<snapshot_id>:e<N>` 是限定 ref。它内嵌其快照 ID。

不接受裸 `@e<N>` ref。

`N` 是不带符号、最多 10 位十进制数字的正 `u32`。快照 ID 为 3–64 个 ASCII 字母数字、`-` 或 `_` 字符。格式错误的令牌、无效 UTF-8 或裸 ref 在派发前返回 `AD_RESULT_ERR_INVALID_ARGS`。格式正确但保存的快照不存在时返回 `AD_RESULT_ERR_SNAPSHOT_NOT_FOUND`；保存的快照没有匹配的本地 ref 时返回 `AD_RESULT_ERR_STALE_REF`。

快照查找限于适配器的命名空间。用 `ad_adapter_create_with_session` 创建的适配器绝不会在全局命名空间或另一个会话中搜索匹配的快照 ID。

## 非主线程迁移

自 ABI v3 起，适配器入口点可在任意宿主线程上运行，包括 macOS 主线程之外，因此 `AD_RESULT_ERR_INTERNAL` 绝不是非主线程的结果。原生句柄保持线程亲和性：从另一个线程或适配器使用，或在释放或撤销后使用，返回 `AD_RESULT_ERR_INVALID_ARGS`。并发与所有权约定见 [threading.md](threading.md)。

## 枚举校验

每个 `#[repr(i32)]` 枚举字段都在 C 边界处校验。超出范围的判别值返回 `AD_RESULT_ERR_INVALID_ARGS` 并附带 last-error 诊断，而不是未定义行为。校验的字段：`AdAction.kind`（`AdActionKind`）、`AdMouseEvent.kind`（`AdMouseEventKind`）、`AdMouseEvent.button`（`AdMouseButton`）、`AdScrollParams.direction`（`AdDirection`）、`AdTreeOptions.surface`（`AdSnapshotSurface`）、`AdScreenshotTarget.kind`（`AdScreenshotKind`）、`AdWindowOp.kind`（`AdWindowOpKind`），以及 `ad_execute_by_ref` / `ad_execute_action_with_policy` / `ad_execute_ref_action_exact_with_policy` 的 `policy` 参数（`AdPolicyKind`）。

## 命令驱动的 JSON 入口点：双重失败模式

`ad_snapshot`、`ad_execute_by_ref`、`ad_execute_by_ref_timeout`、`ad_wait`、`ad_status`、`ad_version`、`ad_trace_show` 和 `ad_trace_export` 有两种不同的失败模式：

- **参数 / 基础设施失败**（空适配器、无效 UTF-8、错误判别值、上下文错误）：`*out` 设为 null，不进行分配，last-error 槽位是唯一的失败指示。
- **命令级失败**（应用未找到、STALE_REF、TIMEOUT 等）：`*out` 设为堆分配的 JSON 字符串，含 `"ok":false` 和 `"error"` 载荷。调用方**仍必须**用 `ad_free_string(*out)` 释放它。last-error 槽位也会被设置。

在决定是否释放之前，始终检查 `*out` 是否为 null。

## Panic 安全

每个可能 panic 的 `extern "C"` 入口点都将函数体包裹在 `catch_unwind` 中。FFI 内部的 Rust panic 表现为 `AD_RESULT_ERR_INTERNAL`，消息为 `"rust panic in FFI boundary"`，而不是宿主崩溃。这仅适用于用 `release-ffi` profile 构建的 cdylib，因为工作区的 `release` profile 使用 `panic = "abort"`。