# 线程

## 宿主线程约定

FFI 入口点可从任意宿主线程调用。库不会对 Accessibility（`AXUIElement`）或 Quartz 事件（`CGEvent`）调用施加一刀切的 macOS 主线程保护，Apple 也未对它们发布主线程要求（[Thread Safety Summary](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/Multithreading/ThreadSafetySummary/ThreadSafetySummary.html)）。使用 Cocoa 对象的代码路径会创建自己的自动释放池，FFI 初始化会启动一个 `NSThread`，因此 Cocoa 在任何工作线程调用 AppKit 之前就已安装好多线程锁。

## 并发与变更顺序

适配器注册表句柄可并发获取。读操作带有限截止时间，且不获取交互租约。

| 并发调用 | 约定 |
|------------------|----------|
| 读 + 读 | 两者可并发运行；均不等待交互租约 |
| 读 + 变更 | 读不等待租约，可能与变更重叠；其结果是某一时刻的观测，而非事务 |
| 变更 + 变更 | 同一 OS 用户下同一时刻只有一个租约持有者运行；等待方消耗其命令截止时间，若无法获取租约则返回结构化超时 |
| 在另一线程调用原生句柄 | 以 `AD_RESULT_ERR_INVALID_ARGS` 拒绝；请改用限定 ref |

快照、查找、获取、显示器/窗口/应用列表、截图、剪贴板读取、权限读取、状态和 trace 读取均为观测。Ref 操作、直接原生句柄操作、输入合成、剪贴板写入/清空、应用与窗口变更、通知操作和权限请求均为变更。例如，`ad_snapshot` 和 `ad_execute_by_ref` 可从不同线程调用，但快照可能与操作重叠；需要"先观测后操作"顺序的调用方必须先等待快照结果，再派发操作。

桌面变更使用进程无关的锁：macOS 和 Linux 上为 `/tmp/agent-desktop-<uid>/interaction.lock` 的建议锁，Windows 上为基于令牌派生的锁文件。因此 CLI 和 FFI 构建会为同一用户跨线程、跨进程及不同 HOME 值串行化变更。该锁是命令级而非事务级：另一参与者可能在观测与后续操作之间，或在两次独立操作之间改变 UI。

租约无法协调：

- 早于该锁的旧版 `agent-desktop` 二进制；
- 直接的人类输入或无关的自动化工具；
- 目标应用自身发起的状态变更。

调用方仍须使用严格 ref/精确窗口标识，并将过期或歧义目标视为正常的可重试自动化结果。

## 适配器销毁

适配器指针是不透明的注册表令牌。每次调用在执行平台工作前会获取一个保留的适配器所有者。`ad_adapter_destroy` 会撤销该令牌：已获取令牌的调用安全完成，之后开始的调用返回 `AD_RESULT_ERR_INVALID_ARGS`。并发销毁不会释放仍被在途调用引用的内存。

销毁适配器还会撤销该适配器在调用线程上创建的原生句柄。其他线程的句柄注册表会拒绝后续使用，因为所有者适配器令牌已不存在。

## 原生句柄的线程归属

`ad_find_exact` 返回不透明的 `AdNativeHandle`。`ad_resolve_element_exact` 从不产生该句柄：它以 `AD_RESULT_ERR_ACTION_NOT_SUPPORTED` 失败。原生句柄绑定适配器且具有线程亲和性：

- 在同一线程上解析、使用和释放句柄；
- 传入产生该句柄的同一适配器；
- 销毁其适配器后不得再使用该句柄。

违规会以 `AD_RESULT_ERR_INVALID_ARGS` 拒绝；库不会解引用伪造、跨适配器、跨线程、已释放或已撤销的令牌。当句柄需要跨异步任务或线程边界时，优先使用快照限定的 ref 和 `ad_execute_by_ref`。

## 日志回调线程

`ad_set_log_callback(cb)` 可从任意线程调用。回调可从任何调用 `ad_*` 函数的线程被调用。

通过 `NULL` 注销的回调仍可能收到另一线程上已在途的调用。请让回调及捕获状态在进程生命周期内保持存活，或在注销前静默所有活跃的适配器调用。

## 语言运行时

运行时串行化不等于线程亲和性。例如，CPython 的 GIL 并不保证两次 FFI 调用在同一 OS 线程上执行。原生句柄只应存放在线程受限的对象中，或避免使用它们而改用快照限定的 ref。当任务可在工作线程间迁移时，Rust、Swift、Node、Go 和托管运行时需要同样的纪律。

## 辅助功能权限身份

`ad_check_permissions` 调用 macOS 的 `AXIsProcessTrusted()`，它报告的是宿主可执行文件（`python3`、`node`、Swift 应用等）的信任状态，而非将 dylib 视为独立可执行文件。权限提示和部署指南必须指明加载 `libagent_desktop_ffi.dylib` 的宿主进程。

## 最后错误与阻塞调用

最后错误槽是线程局部的（见 error-handling.md）。

`ad_wait` 最多阻塞其调用线程至有限截止时间，并在该期间保留适配器。并发销毁适配器令牌会停止新调用，但不会使在途等待失效。