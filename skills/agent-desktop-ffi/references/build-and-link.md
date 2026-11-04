# 构建与链接

## 构建 cdylib

```sh
cargo build --profile release-ffi -p agent-desktop-ffi
```

输出：

- macOS：`target/release-ffi/libagent_desktop_ffi.dylib`
- Linux：`target/release-ffi/libagent_desktop_ffi.so`
- Windows：`target/release-ffi/agent_desktop_ffi.dll`

生成的头部文件为 `crates/ffi/include/agent_desktop.h`。

`--profile release-ffi` 保留 `panic = "unwind"`，每个 `extern "C"` 入口点内部的 `catch_unwind` 捕获需要它。默认的 `release` profile 使用 `panic = "abort"`，会静默使这些捕获失效。

## 预构建归档

每个 GitHub 发布版本都附带以下平台的预构建归档：

- macOS arm64 和 x86_64
- Linux x64 和 arm64（glibc）
- Windows x64 和 ARM64 MSVC

归档命名为 `agent-desktop-ffi-v<version>-<target>.tar.gz`（Windows 上为 `.zip`），其中 `<target>` 是 Rust 目标三元组。每个归档解压到同名目录，包含 `lib/`（dylib、so 或 dll）、`include/agent_desktop.h`、`LICENSE` 和 `README.md`。`lib/` 还包含一个必须与库放在一起的辅助程序：macOS 上是 `agent-desktop-macos-helper`，Linux 上是 `agent-desktop-linux-clipboard-owner`。Windows 上包含 `agent_desktop_ffi.dll.lib` 导入库，供 MSVC 链接方链接使用。完整性：与发布资产中的 `checksums.txt` 比对。供应链验证：每个发布版本都附带 Sigstore 证明；部署前用 `gh attestation verify <file> --repo vercel-labs/agent-desktop` 验证。

## ABI 握手（先做这一步）

`dlopen` / `LoadLibrary` 之后，在调用任何其他函数之前，先比较 dylib 主版本号与你编译时使用的头部：

```c
AdResult rc = ad_init(AD_ABI_VERSION_MAJOR);
if (rc != AD_RESULT_OK) {
    // header and dylib have incompatible major versions
    fprintf(stderr, "ABI mismatch: %s\n", ad_last_error_message());
    return -1;
}
```

或者，读取 dylib 原始主版本号自行比较：

```c
uint32_t dylib_major = ad_abi_version();
if (dylib_major != AD_ABI_VERSION_MAJOR) {
    fprintf(stderr, "ABI major: header=%u dylib=%u\n",
            AD_ABI_VERSION_MAJOR, dylib_major);
    abort();
}
```

版本不匹配时 `ad_init` 返回 `AD_RESULT_ERR_INVALID_ARGS`，诊断信息在 `ad_last_error_message` 中。不要再调用任何函数。

## 结构体大小校验

结构体布局可能与 C 不一致的语言（Python ctypes、Go cgo、JNI 等）必须在将每个固定大小的结构体传给库之前进行校验。FFI 提供三层校验：

1. **头部宏**（`AD_*_SIZE`）和
2. **运行时 getter**（`ad_*_size()`，返回 Rust 侧编译的 `size_t`，与宏相等）：

   | 结构体 | 宏 | Getter |
   |--------|-------|--------|
   | `AdAction` | `AD_ACTION_SIZE` | `ad_action_size()` |
   | `AdActionResult` | `AD_ACTION_RESULT_SIZE` | `ad_action_result_size()` |
   | `AdActionStep` | `AD_ACTION_STEP_SIZE` | `ad_action_step_size()` |
   | `AdDisplayInfo` | `AD_DISPLAY_INFO_SIZE` | `ad_display_info_size()` |
   | `AdDragParams` | `AD_DRAG_PARAMS_SIZE` | `ad_drag_params_size()` |
   | `AdElementState` | `AD_ELEMENT_STATE_SIZE` | `ad_element_state_size()` |
   | `AdExactRefEntry` | `AD_EXACT_REF_ENTRY_SIZE` | `ad_exact_ref_entry_size()` |
   | `AdExactSurfaceInfo` | `AD_EXACT_SURFACE_INFO_SIZE` | `ad_exact_surface_info_size()` |
   | `AdExactWindowInfo` | `AD_EXACT_WINDOW_INFO_SIZE` | `ad_exact_window_info_size()` |
   | `AdRefEntry` | `AD_REF_ENTRY_SIZE` | `ad_ref_entry_size()` |
   | `AdWaitArgs` | `AD_WAIT_ARGS_SIZE` | `ad_wait_args_size()` |

3. **头部中的 C11 静态断言**（`#ifndef AGENT_DESKTOP_ABI_ASSERTS`）在 C 编译期捕获不匹配。

在加载时、构建或传递这些结构体之前，将你绑定中对应的 `sizeof` 与 getter 比较。将 `AdExactRefEntry.version` 设为 `AD_EXACT_REF_ENTRY_VERSION`，`.size` 设为 `AD_EXACT_REF_ENTRY_SIZE`，`.process_instance` 设为快照发出的代际令牌。

## 完整示例：Python ctypes 冒烟测试

`tests/ffi-python/smoke.py` 是 Python 使用者的参考。它仅使用 Python 标准库。测试项：

- 第 1 项：`ad_abi_version()` 与 `AD_ABI_VERSION_MAJOR` 比较。
- 第 2 项：`AdAction`、`AdActionResult`、`AdActionStep`、`AdDragParams`、`AdElementState`、`AdRefEntry` 和 `AdWaitArgs` 的大小 getter 与对应宏比较。
- 第 3 项：`ad_version()`，解析 JSON，`ad_free_string`。
- 第 4 项：`ad_adapter_create`、`ad_snapshot`、`ad_free_string`、`ad_adapter_destroy`。stub-adapter 构建会返回 `PLATFORM_NOT_SUPPORTED` 信封。

运行：

```bash
python3 tests/ffi-python/smoke.py \
  target/release-ffi/libagent_desktop_ffi.dylib \
  crates/ffi/include/agent_desktop.h
```

或使用环境变量：

```bash
AD_DYLIB_PATH=target/release-ffi/libagent_desktop_ffi.dylib \
AD_HEADER_PATH=crates/ffi/include/agent_desktop.h \
python3 tests/ffi-python/smoke.py
```

`tests/ffi-python/README.md` 涵盖 stub-adapter 构建和 `AD_EXPECT_*` 开关。

## 最小 C 示例

```c
#include <stdio.h>
#include "agent_desktop.h"

int main(void) {
    // 1. ABI handshake
    if (ad_init(AD_ABI_VERSION_MAJOR) != AD_RESULT_OK) {
        fprintf(stderr, "ABI mismatch: %s\n", ad_last_error_message());
        return 1;
    }

    // 2. Create adapter
    AdAdapter *adapter = ad_adapter_create();
    if (!adapter) {
        fprintf(stderr, "adapter_create failed: %s\n", ad_last_error_message());
        return 1;
    }

    // 3. Check permissions
    AdResult rc = ad_check_permissions(adapter);
    if (rc != AD_RESULT_OK) {
        fprintf(stderr, "permission denied: %s\n", ad_last_error_message());
        ad_adapter_destroy(adapter);
        return 1;
    }

    // 4. Snapshot the focused window
    char *json_out = NULL;
    rc = ad_snapshot(adapter,
                     NULL,                       // app (null = focused window)
                     AD_SNAPSHOT_SURFACE_WINDOW, // surface
                     10,                         // max_depth
                     false,                      // interactive_only
                     false,                      // compact
                     &json_out);
    if (rc == AD_RESULT_OK && json_out) {
        printf("%s\n", json_out);
        // parse the JSON to find qualified refs in data.tree
        ad_free_string(json_out);
    } else if (json_out) {
        // command-level error: ok:false envelope, still must free
        fprintf(stderr, "snapshot error: %s\n", json_out);
        ad_free_string(json_out);
    } else {
        // infrastructure error: *out is null
        fprintf(stderr, "snapshot failed: %s\n", ad_last_error_message());
    }

    ad_adapter_destroy(adapter);
    return 0;
}
```

编译（macOS 和 Linux）：

```sh
clang -I./crates/ffi/include main.c \
      -L./target/release-ffi -lagent_desktop_ffi \
      -Wl,-rpath,"$PWD/target/release-ffi" \
      -o snapshot_demo
```

macOS 安装名为 `@rpath/libagent_desktop_ffi.dylib`，因此链接方需要指向库目录的 rpath，否则 dyld 在启动时会中止。`dlopen` 和 ctypes 调用方按路径解析，无需额外配置。

## C 中的观察-操作工作流

解析快照 JSON 并提取合格 ref ID 后：

```c
AdAction act = {0};              // zero-init before setting any field
act.kind = AD_ACTION_KIND_CLICK;

char *result = NULL;
AdResult rc = ad_execute_by_ref(
    adapter,
    "@s8f3k2p9:e5", // qualified ref from snapshot data.tree
    &act,
    AD_POLICY_KIND_HEADLESS,
    &result
);
if (result) {
    // parse JSON: ok:true on success, ok:false on STALE_REF etc.
    printf("%s\n", result);
    ad_free_string(result);
}
if (rc != AD_RESULT_OK) {
    fprintf(stderr, "execute_by_ref failed (%d): %s\n",
            (int)rc, ad_last_error_message());
}
```

要输入文本，设置动作类型和文本字段：

```c
AdAction type_act = {0};
type_act.kind = AD_ACTION_KIND_TYPE_TEXT;
type_act.text = "hello";
// TypeText runs headless by default; opt in to focus fallback explicitly:
rc = ad_execute_by_ref(adapter, "@s8f3k2p9:e3", &type_act,
                       AD_POLICY_KIND_FOCUS_FALLBACK, &result);
```

## 最小 Python ctypes 示例

```python
import ctypes, json
from ctypes import c_int32, c_uint8, c_bool, c_char_p, POINTER, c_void_p

lib = ctypes.CDLL("./target/release-ffi/libagent_desktop_ffi.dylib")

# ABI handshake
lib.ad_init.restype = c_int32
lib.ad_init.argtypes = [ctypes.c_uint32]
AD_ABI_VERSION_MAJOR = 7  # sync with header macro
rc = lib.ad_init(AD_ABI_VERSION_MAJOR)
assert rc == 0, f"ABI mismatch: rc={rc}"

# Adapter lifecycle
lib.ad_adapter_create.restype = c_void_p
lib.ad_adapter_create.argtypes = []
lib.ad_adapter_destroy.restype = None
lib.ad_adapter_destroy.argtypes = [c_void_p]

# Snapshot
lib.ad_snapshot.restype = c_int32
lib.ad_snapshot.argtypes = [c_void_p, c_char_p, c_int32, c_uint8, c_bool, c_bool,
                             POINTER(c_char_p)]
lib.ad_free_string.restype = None
lib.ad_free_string.argtypes = [c_char_p]

lib.ad_last_error_message.restype = c_char_p
lib.ad_last_error_message.argtypes = []

adapter = lib.ad_adapter_create()
assert adapter, "ad_adapter_create() returned null"

out = c_char_p()
rc = lib.ad_snapshot(adapter, None, 0, 10, False, False, ctypes.byref(out))
if out.value:
    envelope = json.loads(out.value)
    lib.ad_free_string(out)
    print("ok:", envelope.get("ok"))
else:
    msg = lib.ad_last_error_message()
    print("error:", msg.decode() if msg else "(no message)")

lib.ad_adapter_destroy(adapter)
```