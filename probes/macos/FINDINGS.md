# macOS 平台发现

适配器所依赖的 macOS 辅助功能行为的证据台账。每行将一个实验映射到观察结果。M1 各行的探针源码位于 `ax-remote-token/`（`tok.c`、`scan.c`、`bench.c`、`unapproved.m`、`cover.c`、`web.c` 及 `rows.html`）；用 `clang -framework ApplicationServices <file>.c` 构建每个文件。

## 如何阅读一行

| 列 | 含义 |
| --- | --- |
| `id` | 稳定的行 id（`M<area>-<n>`） |
| `probe` | 产生该行的探针和步骤 |
| `scope` | `api-contract` 适用于任何暴露辅助功能的应用；`app` 特定于被测应用 |
| `observed` | 结论所依据的测量值 |
| `conclusion` | 该行确立的内容 |

## 环境

macOS 26.4（arm64），运行于 lume VM，已登录 GUI 会话，辅助功能已授予 SSH 上下文。应用：E2E 测试夹具应用（`tests/fixture-app`，SwiftUI，ref-churn 视图）和 Finder（AppKit，纯文件文件夹的列表视图）。

## M1：持久的跨进程元素令牌

| id | probe | scope | observed | conclusion |
| --- | --- | --- | --- | --- |
| M1-1 | `probe.c` 通过 `dlsym` 查找符号 | api-contract | `_AXUIElementRemoteTokenCreate` 和 `_AXUIElementCreateWithRemoteToken` 在 ApplicationServices 中可解析；`_AXUIElementCopyRemoteToken` 和 `AXUIElementCreateWithRemoteToken` 不存在 | 这对私有符号在 macOS 26.4 上存在。它是私有 API：运行时查找，缺失时回退。 |
| M1-2 | `tok.c dump`，然后在新进程中 `tok.c resolve`，夹具行共享标识符 `shared-title` | api-contract | Alpha 令牌 `f802…6f636f6335…` 和 Beta 令牌 `…36…` 各自在新进程中解析为相同的角色、标识符和值（`Alpha`、`Beta`） | 在一个进程中创建的令牌在另一进程中解析为同一元素。格式：pid、4 字节标签、每应用元素 id（20 字节；Finder 列表行使用 48 字节形式）。 |
| M1-3 | 夹具：过滤掉 Alpha 行，解析两个已保存的令牌 | api-contract | Alpha 令牌：`kAXErrorInvalidUIElement`（-25202）。Beta 令牌：仍为 Beta。 | 被移除元素的令牌会失效。它不会落到移位到其位置的行上，而这正是属性匹配无法排除的错误行情况。 |
| M1-4 | 夹具：恢复行（一个新的、相同的 Alpha），比较令牌 | api-contract | 新 Alpha 令牌 `…44…` 与旧的 `…35…` 不同；旧令牌仍以 -25202 失败 | 令牌不会为相同的替代品复用，因此失效的令牌永远不会复活到相似元素上。 |
| M1-5 | 夹具：重启应用，解析已保存的令牌 | api-contract | -25204（无法完成） | 令牌随进程消亡；进程代际检查作为第二道防线保留。 |
| M1-6 | 夹具：三次无关的列表重新渲染，解析 `stable-editor` 令牌 | app | 仍解析为 `stable-editor` | 令牌在 SwiftUI 视图无关部分的重新渲染中存活。 |
| M1-7 | Finder 列表视图：`scan.c dump`（253 个元素，55 行），添加 3 个文件使行重新排序并移位，`scan.c check` | app | same=252，dead=0，changed=1（"8 items" 状态文本变为 "11 items"） | 在密集的 AppKit 列表中，令牌在列表重新排序和行移位中持续解析。`scan.c` 仅比较角色和标签，因此该行本身不能证明每个令牌背后的元素是原始对象；M1-3 和 M1-4 为夹具证明了这一点。 |
| M1-8 | Finder：删除 `delta.txt`，`scan.c check` | app | same=243，dead=9（被删除行的元素），changed=1 | 删除一行会杀死该行的令牌，且没有已保存的令牌解析到角色或标签不同的元素。 |
| M1-9 | Finder：将 `golf.txt` 重命名为 `golf2.txt`，`scan.c check` | app | 被重命名行的令牌解析为显示 `golf2.txt` 的同一元素 | 令牌随其元素经历文本变更；身份是对象，而非其标签。 |
| M1-10 | 在 Finder 中运行 `bench.c`：遍历到 `hotel.txt`，然后解析其令牌 50 次 | app | 树遍历 53.2 ms，遍历 206 个元素；令牌重建加一次角色读取：最小 0.024 ms，平均 0.028 ms | 从令牌重建元素远比树遍历便宜。这测量的是令牌调用和一次读取，而非完整的受保护解析器，后者还增加进程、窗口和角色检查。 |
| M1-11 | `unapproved.m`：已批准的 SSH 上下文为夹具的 `stable-editor` 字段创建令牌；通过 LaunchServices 启动的 ad hoc 签名应用（因此 TCC 向该应用收费）接收令牌，创建元素，读取其值，将其设为 `poisoned-by-unapproved` 并按下它 | api-contract | 未批准应用：`AXIsProcessTrusted=0`，元素已创建，读取 -25211，设置 -25211，按下 -25211（`kAXErrorAPIDisabled`）；之后该字段仍读取 `Stable value` | 令牌是地址，不是能力：每次使用仍需要调用者自己的辅助功能权限。将会话 refmap 中的令牌存储起来不会授予未批准应用任何它本已无法猜到的内容，也不会增加可借出权限的共享进程。 |
| M1-12 | `cover.c`：遍历应用的所有元素（子元素和窗口），为每个元素创建令牌，从中重建元素并用 `CFEqual` 比较 | api-contract | Finder：685 个元素，685 个令牌，685 个相等。Safari 打开一篇 Wikipedia 文章：417 / 417 / 417。Chrome 148 打开同一文章：9906 / 9906 / 9906。令牌大小 12 到 44 字节 | 通过辅助功能暴露的每个元素都有令牌，涵盖 AppKit、WebKit 和 Chromium 内容，且每个令牌都重建出完全相同的元素。 |
| M1-13 | `web.c` + `rows.html` 在 Chrome 中：为四个相同的 `Delete` 按钮捕获令牌，然后页面移除一行，编辑另一行的文本，最后用 `innerHTML` 重新渲染整个列表 | app | t0 全部存活；移除 Beta 后：Beta 失效，其余三个存活；文本编辑后：不变；完全重新渲染后：全部四个失效 | 在 Chromium Web 内容中，相同的孪生元素各自保留自己的令牌，被移除节点的令牌失效且不会落到邻居上，文本变更保留令牌，被替换的 DOM 节点获得新元素，因此对它们的引用会变为陈旧。 |

尚未测量：除夹具和 Chrome Web 内容之外的其他提供者的非复用性、Electron 应用、每次刷新都重建辅助功能树的应用，以及跨 macOS 主要版本的行为。令牌更频繁地失效只会让引用返回 `STALE_REF`；它永远不会解析到另一个元素。