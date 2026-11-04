Linux 需要 glibc 2.35 或更新版本。

`dlopen` 和 Python `ctypes` 调用方按路径加载 `lib/libagent_desktop_ffi.so`。该库没有 SONAME，因此链接它的程序会记录文件名。链接时用 `-L` 指定 `lib/` 目录，运行时用 `-rpath` 指向该目录。

完整 API 见 https://agent-browser.dev/desktop/ffi。

- 变更操作在交互锁下跨线程和进程逐个执行。读取可与变更重叠，多个线程可共享同一适配器。原生句柄须在创建它的适配器所在线程上解析、使用和释放。见 https://agent-browser.dev/desktop/ffi/ownership-and-threading。
- `ad_snapshot` 接收应用名，没有窗口 id 或根元素参数。
- 剪贴板调用只携带文本。`ad_set_clipboard` 启动分离的 `lib/agent-desktop-linux-clipboard-owner` 进程。宿主退出后该进程继续提供文本，直到另一客户端取得选区或 `ad_clear_clipboard` 结束它。库在自身镜像旁查找辅助程序，从不通过 `PATH`，因此两个文件须放在同一目录。Wayland 需安装 `wl-clipboard`。
- 观察需要会话无障碍总线（`at-spi2-core`）。在 Ubuntu 22.04 上，at-spi2-core 桥接在 `Action.GetActions` 处中止。其他发行版需要桥接修复 dc0dc33。