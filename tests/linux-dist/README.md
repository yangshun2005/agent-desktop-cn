# Linux 发行版检查

支持的实时桌面最低版本为 Ubuntu 24.04。CLI 和 FFI 构建产物仍在固定版本的 Ubuntu 22.04 容器中构建，且必须通过 glibc 2.35 最低版本检查。`linux-consumer` 任务在托管的 Ubuntu 24.04 x64 和 ARM64 运行器上，于私有 Openbox 会话中直接运行这些提取出的构建产物。二进制加载器最低版本并不认证更旧的桌面栈。

Ubuntu 22.04 的 ATK 桥接 2.38.0 在应用的 `Action.GetActions` 内部中止：它声明了 `(sss)` 条目，却追加了四个字符串。该库的树遍历会调用此方法；应用中止后，它留下 `TIMEOUT` 和零个节点。故障出在应用进程内的桥接中，而非 `at-spi2-registryd` 或 Rust `zbus` 库。Ubuntu 24.04 的 2.52.0 桥接包含[上游修复](https://github.com/GNOME/at-spi2-core/commit/dc0dc33)。

在 22.04 上，消费者在 dbus 为 `a(sss)` 记录签名错误后，以 `windows-exact: FAIL code=TIMEOUT rc=-9 nodes=0` 失败。一个隔离探针将官方 Ubuntu [2.38.0](https://archive.ubuntu.com/ubuntu/pool/main/a/at-spi2-atk/at-spi2-atk_2.38.0.orig.tar.xz) 和 [2.52.0](https://archive.ubuntu.com/ubuntu/pool/main/a/at-spi2-core/at-spi2-core_2.52.0.orig.tar.xz) 源码压缩包中未经修改的 `impl_GetActions` 处理程序针对 libdbus 1.12.20 编译，并提供一个伪造的 ATK 动作。Jammy（22.04）处理程序以该确切签名错误经 SIGABRT 退出；Noble（24.04）处理程序以回复签名 `a(sss)` 退出 0。该探针测量的是有故障的序列化器，而非完整的 22.04 桌面复现。

## 运行检查

从同一检出构建 CLI、FFI 库和剪贴板所有者。使用发布工作流布局创建 CLI 和 FFI 归档。在前三个命令各自的私有会话中运行每个命令。`scripts/private-session.sh --wm` 启动私有会话总线、Xvfb 显示和 Openbox；它在 `$AGENT_DESKTOP_WM_ROOT/usr/bin/openbox` 查找 Openbox（`AGENT_DESKTOP_WM_ROOT` 默认为 `/`）。需要时将其设为备用安装根目录。检查所需的 apt 包列在 `.github/workflows/release-dry-run.yml` 的 `linux-consumer` 任务中；npm 检查还需要 Node 和 npm。

```sh
export AGENT_DESKTOP_WM_ROOT=/
scripts/private-session.sh --wm -- tests/linux-dist/run-consumer.sh <extracted-ffi-directory>
scripts/private-session.sh --wm -- scripts/ci-ffi-helper-discovery-smoke.sh
scripts/private-session.sh --wm -- tests/linux-dist/run-npm-install.sh <archives-directory>
python3 -I -m unittest discover -s tests/ffi-python -p 'test_linux_helper.py'
python3 -I -m unittest discover -s tests/linux-dist -p 'test_*.py'
```

`<extracted-ffi-directory>` 是提取出的 FFI 归档，包含 `lib/libagent_desktop_ffi.so` 和 `include/agent_desktop.h`。`<archives-directory>` 包含此架构的 CLI 归档（`agent-desktop-v*-<target>.tar.gz`）；`run-npm-install.sh` 通过一个 PATH 优先的 `curl` 替身提供它，以便 postinstall 下载和校验和路径运行。

FFI 发现脚本期望库位于 `target/release-ffi`（或 `target/release`），所有者位于 `target/release`；消费者任务将两者从提取出的 FFI 归档复制到那里。脚本将两者复制到新目录，并在剪贴板调用期间清空 PATH。外部 GTK 读取器必须在适配器销毁后收到载荷，然后在清除后观察到空选择。npm 检查在安装的 CLI 进程退出后应用相同契约。发布试运行安装 GTK 3 typelib，运行伪造契约，并从该任务提取出的归档中准备发现检查。

消费者还在 Calculator 注册时，针对提取出的库和头文件运行真实的 ctypes 冒烟测试，要求完整的主线程和工作线程快照带引用。冒烟测试失败则消费者失败。C 和 npm 点击检查都要求在派发前有不同于 `7` 的可读基线、唯一的按钮引用，以及独立回读 `7`。剪贴板清除仅接受读取器的无文本状态；观察者错误则检查失败。上面列出的确定性测试用伪造对象覆盖这些契约。

要确认在没有辅助程序时发现会失败，仅准备库，将 `AD_DYLIB_PATH` 指向它，设置 `AD_HEADER_PATH=crates/ffi/include/agent_desktop.h`、`AD_EXPECT_LINUX_HELPER=1` 并将 `AD_HELPER_EMPTY_PATH` 设为空目录，然后在私有会话中运行 `python3 tests/ffi-python/smoke.py`。它必须以缺少剪贴板所有者错误失败。借用其他检出的辅助程序会使此检查失效。

## 测试设计规则

- C 生命周期分支运行二十次创建/快照/销毁循环，并检查已销毁的句柄返回 `INVALID_ARGS`。它不启动快照线程与销毁竞争，因为调度无法确定哪个操作先获取句柄。Rust 测试 `destroy_revokes_new_calls_without_invalidating_in_flight_owners`（`crates/ffi/src/adapter_tests.rs`）显式持有所有者，并在无调度竞争的情况下测试在途保留。
- 外部剪贴板读取器显式选择 GTK 3 和 GDK 3。GDK 不指定版本会在同时安装两个工具包的主机上选择 GDK 4，并阻止任何剪贴板观察；伪造 GI 测试会拒绝该不匹配。
- 私有总线清理（`private_bus.py`）通过 PID 句柄发送信号，并在打开句柄前后检查进程代次和精确运行时成员资格，然后等待观察到的退出。它不使用带固定延迟的 PID 列表。伪造进程测试会拒绝 PID 重用和外部运行时成员资格。

## 证据

此工具链的测量结果记录在 `probes/linux/FINDINGS.md`：`L22-16`（glibc 2.35 最低版本）、`L22-18`（apt 包集）、`L22-19`（托管运行器能力）和 `L22-20`（npm 安装和分离的剪贴板所有权）。