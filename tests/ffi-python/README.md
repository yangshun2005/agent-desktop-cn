# FFI Python 冒烟测试工具

验证 C ABI 能在非 Rust 宿主（Python ctypes）中正常工作，并在 CI 中作为门禁。

## 测试内容

| 环节 | 检查项 |
|-----|-------|
| 1 | `ad_abi_version()` 等于头文件中的 `AD_ABI_VERSION_MAJOR`。捕获导入时头文件与动态库不匹配的问题。 |
| 2 | 每个 `ad_*_size()` getter 等于对应的 `AD_*_SIZE` 宏。捕获结构体布局漂移。 |
| 3 | `ad_version()` 返回含 `data.version` 的 `ok:true` JSON。 |
| 4 | `ad_adapter_create` → `ad_snapshot` → `ad_free_string` → `ad_adapter_destroy` 运行无崩溃、无泄漏。桩适配器返回干净的 `PLATFORM_NOT_SUPPORTED` 信封。 |
| 5 | 通过 `AD_EXPECT_REAL=1` 启用（可选 `AD_APP`，默认 `gnome-calculator`）。真实适配器在主线程和工作线程上均返回包含应用引用的完整快照。由 `tests/linux-dist/run-consumer.sh` 运行。 |
| Linux helper | 通过 `AD_EXPECT_LINUX_HELPER=1` 和 `AD_HELPER_EMPTY_PATH` 启用。由 `scripts/ci-ffi-helper-discovery-smoke.sh` 运行。在空 `PATH` 下，库能找到旁边的剪贴板所有者，外部 GTK 读取器能看到它设置的文本，清空操作能清空选区。 |
| 6 | 通过 `AD_EXPECT_MACOS_HELPER=1` 启用。由 `scripts/ci-ffi-helper-discovery-smoke.sh` 运行。检查 macOS helper 发现。 |

## 构建动态库

```bash
# From the repo root:
cargo build --locked --profile release-ffi -p agent-desktop-ffi --features stub-adapter
```

动态库在 macOS 上位于 `target/release-ffi/libagent_desktop_ffi.dylib`，在 Linux 上位于 `target/release-ffi/libagent_desktop_ffi.so`。

使用 `--profile release-ffi`。默认的 `release` profile 设置了 `panic = "abort"`，会静默禁用 FFI 的 `catch_unwind` 捕获。

`--features stub-adapter` 将真实适配器替换为对所有调用返回 `PLATFORM_NOT_SUPPORTED` 的桩适配器。CI 使用此构建，因此测试工具无需辅助功能权限即可运行。

## 本地运行

示例使用 macOS 路径。在 Linux 上请传入 `.so`。

```bash
python3 tests/ffi-python/smoke.py \
  target/release-ffi/libagent_desktop_ffi.dylib \
  crates/ffi/include/agent_desktop.h
```

或设置环境变量：

```bash
export AD_DYLIB_PATH=target/release-ffi/libagent_desktop_ffi.dylib
export AD_HEADER_PATH=crates/ffi/include/agent_desktop.h
python3 tests/ffi-python/smoke.py
```

## 桩强制校验（`AD_EXPECT_STUB=1`）

对桩构建运行测试工具时设置 `AD_EXPECT_STUB=1`。此时若 `ad_snapshot()` 返回 `ok:true`，测试工具会失败，说明动态库未使用 `--features stub-adapter` 构建。`ffi-panic-guard` CI 任务的 arm64 macOS 环节会设置此变量。因此真实适配器动态库无法通过桩门禁。

## 依赖

`smoke.py` 仅使用 Python 标准库，无需 `pip install`。Linux helper 环节是例外。`linux_helper.py` 以外部 GTK 客户端身份读取剪贴板，因此需要 PyGObject 及 GTK 3 typelib（`python3-gi` 和 `gir1.2-gtk-3.0`）。

## 真实适配器正常路径

环节 1 到 4 覆盖无需权限的部分：ABI 版本、结构体大小、`ad_version` 以及 `PLATFORM_NOT_SUPPORTED` 路径。真实适配器成功执行 `ad_snapshot` 的测试在别处。macOS 上由 `tests/e2e/run.sh` 通过 `scenarios/acceptance.sh` 覆盖，后者对 release FFI 库运行 `ffi_live.py`。该运行需要辅助功能权限。Linux 上由环节 5 覆盖。

## 其他文件

| 文件 | 用途 |
|------|---------|
| `linux_helper.py` | Linux helper 环节，以及用于已安装 CLI 的 `verify_cli`（由 `tests/linux-dist/run-npm-install.sh` 使用） |
| `test_linux_helper.py` | `linux_helper.py` 的单元测试：`python3 -I -m unittest discover -s tests/ffi-python -p 'test_linux_helper.py'` |
| `leak_probe.py` | 仅 Linux，CI 不运行：`python3 tests/ffi-python/leak_probe.py <library.so> [--calls N]`。它将库指向一个静默的 X11 TCP 服务器，检查重复超时的 `ad_list_displays` 调用是否保持宿主的线程和描述符计数平稳。 |