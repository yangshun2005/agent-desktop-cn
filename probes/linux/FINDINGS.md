# Linux 平台探索发现（子阶段 3.0）

证据台账，子阶段 3.0 所需。每一行将一个实验映射到观察到的行为和文档对齐结论，每一行都标明产生它的客户端栈及其结论所承载的范围。各行按 `docs/phases.md` 第 3 阶段评级。

探针脚本和原始捕获是开发辅助工具，保存在仓库之外，因此 `probe` 单元格标明探针区域和步骤，而非已提交的路径。`observed` 单元格引用结论所依据的捕获字段值。

## 如何阅读一行

| 列 | 含义 |
| --- | --- |
| `id` | 稳定的行 id（`L<area>-<n>`），由 phases.md 片段和底部的矛盾索引引用 |
| `probe` | 产生该行的探针区域和步骤 |
| `stack` | `dbus` = 通过 Gio 的原始 D-Bus，产品相关栈；`gi-atspi` = libatspi 便捷读取；`tool` = 外部工具，如 xdotool、xprop、xclip。组合用 " + " 连接 |
| `scope` | `api-contract` = 产品级，适用于任何符合规范的 AT-SPI 客户端。`app` = 特定于被测应用和版本。`box` = 特定于此环境。下游子阶段读取此列以了解哪些可推广 |
| `phases.md expectation` | 探针测试的 `docs/phases.md` 中的句子或声明，或无 |
| `observed` | 结论所依据的捕获字段值和测量值 |
| `verdict` | `CONFIRMS`、`CONTRADICTS`、`NEW-EDGE`、`DEFERRED` |
| `action` | 该发现所要求的行动。`DEFERRED` 行带有 `needs:` 和 `closure: 3.<n>`，即关闭它们的第 3 阶段子阶段 |

栈权威：当 `dbus` 行与 `gi-atspi` 或 `tool` 行不一致时，`dbus` 行胜出，因为 Rust 适配器使用原始 D-Bus。

## 环境

除探针单元格标明 XFCE 容器的行（在第二环境中测量）外，每一行都在下表中测量。

| 事实 | 值 |
| --- | --- |
| OS | Ubuntu 24.04.4 LTS，内核 7.0 通用，x86_64 |
| 桌面 | GNOME Shell 46.0（mutter 46.2），X11 会话；Wayland 行（L13）在同一机器上的隔离无头 gnome-shell 中测量 |
| 无障碍和总线 | AT-SPI core 2.52，dbus 1.14.10 |
| 工具包 | GTK 3.24.41，GTK 4.14.5，Qt 6.11.0（PyQt6 6.11.0） |
| 工具 | xdotool 3.20160805.1，xclip 0.13，wl-clipboard 2.2.1，wmctrl 1.07 |
| 应用 | VS Code 1.142.0，Google Chrome 154，snap Firefox 157 |
| XFCE 容器 | Ubuntu 24.04 arm64 容器，Xvfb 1440x900，XFCE 4.18（xfwm4 合成），AT-SPI core 2.52，GTK 3.24.41（Mousepad 0.6.1，Thunar 4.18），GTK 4.14.5（GNOME Text Editor 46.3），Qt 6.6.1（PyQt6） |

## 区域 L1 — 环境和总线引导

| id | probe | stack | scope | phases.md expectation | observed | verdict | action |
| --- | --- | --- | --- | --- | --- | --- | --- |
| L1-1 | OS、桌面和库版本 | tool | box | 最低 OS 要求："Ubuntu 22.04+ / Fedora 38+"，"GNOME 42+（主要目标）"，"已安装 `at-spi2-core` 包（GNOME 默认）" | `os_release.VERSION: 24.04.4 LTS`，`tools.gnome-shell.version: GNOME Shell 46.0`，`packages.at-spi2-core: 2.52.0-1build1`，`packages.libgtk-4-1: 4.14.5+ds`，`packages.libgtk-3-0t64: 3.24.41`，`packages.dbus: 1.14.10`，`kernel: 7.0.0-38-generic`，`arch: x86_64` | CONFIRMS | 3.1：无；被测机器位于所述下限之内。22.04 / Fedora 38 / GNOME 42 的下限声明未在此处验证 |
| L1-2 | 基础安装中的输入和剪贴板工具 | tool | box | 显示服务器检测："验证已安装 `xdotool`（X11）或 `ydotool`（Wayland）；若缺失则报错并给出安装说明"以及"`xclip`（X11）或 `wl-clipboard`（Wayland）" | `packages_absent` 列出 `xdotool`、`xclip`、`xsel`、`wl-clipboard`、`wmctrl`、`ydotool`、`flatpak`、`python3-pyatspi`；这些全部仅来自用户空间树（`tools.xdotool.origin: user-space`） | CONFIRMS | 3.8 / 3.10：缺失工具错误路径在标准 GNOME 46 安装上可达，并非边缘情况；安装指南必须指明发行版包 |
| L1-3 | 工具路径和版本（用户空间树来源） | tool | box | 无 | `tools.xdotool.version: xdotool version 3.20160805.1`，`tools.xclip.version: xclip version 0.13`，`tools.wmctrl.version: 1.07`，`tools.wl-copy.version: wl-clipboard 2.2.1`，`tools.weston.version: weston 13.0.0`，`tools.ydotool.status: present`（help 无输出），`tools.ydotoold.status: absent`；还有 `tools.google-chrome.version: Google Chrome 154.0.8037.97`，`tools.firefox.version: Mozilla Firefox 157.0.1`，`tools.python3.version: Python 3.12.3`；每个工具的绝对路径记录在捕获中 | NEW-EDGE | 3.8：xdotool 是 2016 发布线（3.20160805.1）；输入行必须对照该版本阅读，更新的标志不可用 |
| L1-4 | 用于检测的显示服务器环境 | tool | box | 显示服务器检测："检查 `$WAYLAND_DISPLAY` ... 若设置，使用 Wayland 路径"；"检查 `$DISPLAY` ... 若设置且无 Wayland，使用 X11 路径" | `session_env.DISPLAY: :0`，`session_env_unset` 包含 `WAYLAND_DISPLAY`，`session_env.XDG_SESSION_TYPE: tty`（远程 shell），`wayland_feasibility.gdm_wayland_enabled_conf: ["WaylandEnable=false"]`，`wayland_feasibility.session_is_wayland: false` | CONFIRMS | 3.1 / 3.8：环境变量顺序对 X11 部分有效。在图形会话的远程 shell 中 `XDG_SESSION_TYPE` 读为 `tty`，不得用于检测。Wayland 部分未验证（L1-17） |
| L1-5 | 冷总线上的存在性检查（名称所有者检查的失败方向） | dbus | api-contract | §3.1 AT-SPI2 总线检测："检查 D-Bus 会话总线上 `org.a11y.Bus` 是否存在" | 冷私有会话，7 次保留运行加预热：`pre_name_has_owner: false` 而 `pre_listed_activatable: true`；`no_auto_start_get_address.remote_error: org.freedesktop.DBus.Error.NameHasNoOwner`；普通 `GetAddress` 随后成功且 `post_name_has_owner: true`（`get_address_first_call.ok: true`） | CONTRADICTS | 3.1：用自动启动的 `org.a11y.Bus.GetAddress` 调用检测，绝不用 `NameHasOwner` 或非自动启动调用；那些会将健康的冷会话报告为缺失 |
| L1-6 | 活动会话上的存在性检查（通过方向） | dbus | box | 无 | `dbus_live_session.org.a11y.Bus_has_owner.value: [true]`，`dbus_live_session.org.a11y.Bus_activatable: true`，`dbus_live_session.a11y_bus_address.scheme: unix`，`dbus_live_session.org.a11y.atspi.Registry_has_owner.value: [true]`，`dbus_live_session.registry_child_count.value: 19` | NEW-EDGE | 3.1：在热会话上两种检查一致；L1-5 中的分歧仅在首次激活前出现 |
| L1-7 | 缺失和损坏的 `org.a11y.Bus` 服务 | dbus | api-contract | §3.1："若总线未运行，返回 `PLATFORM_NOT_SUPPORTED`" | 无服务文件（2 次运行）：`absent_service[*].get_address_default_flags.remote_error: org.freedesktop.DBus.Error.ServiceUnknown`，0.12-0.47 ms，`listed_activatable: false`；Exec 缺失的服务（2 次运行）：`broken_service_exec_missing[*].get_address_default_flags.remote_error: org.freedesktop.DBus.Error.Spawn.ExecFailed`，1.27-1.43 ms，`listed_activatable: true`；非自动启动变体两者均返回 `NameHasNoOwner` | NEW-EDGE | 3.1：将 ServiceUnknown（包未安装）和 Spawn.ExecFailed（已安装但无法启动）映射到 PLATFORM_NOT_SUPPORTED，并使用不同的 `platform_detail`；两者均在 2 ms 内失败，因此缺失情况无需超时预算 |
| L1-8 | 原始 D-Bus 上的冷激活延迟（7 次最小值，丢弃预热） | dbus | box | 无 | `cold_latency_ms.get_address_first_call: {min 6.11, median 6.3, max 7.8}` ms，`connected_to_a11y_bus: {6.45, 6.66, 8.17}`，`registry_usable: {11.21, 11.36, 13.17}`，`warm_get_address: {0.13, 0.15, 0.23}`（n=49）；`runs_with_usable_registry: 7` / 7 | NEW-EDGE | 3.1：冷总线到可用注册表的首次命令成本约 11-13 ms；适配器在此栈上无需启动重试循环，每会话仅需一次自动启动调用 |
| L1-9 | 空注册表是成功，不是失败 | dbus | api-contract | 无 | 新总线：`registry.child_count: 0`，`registry.usable: true`，`a11y_bus_names: [org.a11y.atspi.Registry, org.freedesktop.DBus]`，`session_survivors_while_running: [at-spi-bus-laun, at-spi2-registr, dbus-daemon]` | NEW-EDGE | 3.1 / 3.2：将已回答的 `ChildCount 0` 报告为"总线已启动，无应用"，与读取错误分开；不要将 0 个子节点映射为总线故障 |
| L1-10 | `org.a11y.Status` 属性 | dbus | box | §3.1：GNOME 提示检查 `toolkit-accessibility` | 活动：`dbus_live_session.IsEnabled.value: true`，`dbus_live_session.ScreenReaderEnabled.value: false`，`gsettings["org.gnome.desktop.interface toolkit-accessibility"]: true`；新私有会话读取 `status_IsEnabled.value: true`，`status_ScreenReaderEnabled.value: false` | NEW-EDGE | 3.1：`IsEnabled` 在每个桌面上均可通过 D-Bus 读取，不同于 `gsettings`，但它不是诊断：其为 false 时总线和应用仍工作（L1-16） |
| L1-11 | GTK4 应用在 a11y 总线存在前启动（总线可激活） | dbus | app | 无 | `gtk4_app.cold[0]`：`pre_name_has_owner: false`，然后 `app_triggered_a11y_bus_activation: true`，`app_registration_from_registry_poll.registered: true`，`tree_nodes: 4`，`tree_read_failures: 0`，`leak_check_shared_a11y_bus.app_found_on_shared_a11y_bus: false` | NEW-EDGE | 3.1 / 3.2：GTK4 应用（4.14.5）自身激活总线，因此适配器无需先启动它 |
| L1-12 | 总线存在时 GTK4 应用树形状（热，通过方向） | dbus | app | 无 | `gtk4_app.warm[0]`：按深度的节点：`application`（"python3"，Accessible+Application）-> `application`（"BusProbe window"，Accessible+Action+Component）-> `button`（"Probe button"，Action+Component）-> `label`（"Probe button"，Accessible+Action+Component+Text）；`tree_nodes: 4`，`tree_read_failures: 0`，与冷运行相同 | NEW-EDGE | 3.3：GTK4 `ApplicationWindow` 通过 `GetRoleName` 报告角色名 `application`，而非 `frame` 或 `window`；角色映射不得从 GTK4 假定窗口角色。`label` 子节点重复按钮名称并声明 Action。在 L3-2 中按数字角色重访：GTK4 窗口是角色号 23（frame） |
| L1-13 | GTK4 应用在总线前启动，总线后启动（失败方向，2 次独立运行） | dbus | app | 无 | `gtk4_app.late[0]` 和 `[1]`：`a11y_bus_present_when_app_ready: false`，`late_bus_name_has_owner: true`（约 11 ms 内启动），`app_registration_from_registry_poll.registered: false` 在 12 s 后（约 1155 次轮询），`registry_children_seen: 0`；两次运行中 `leak_check_shared_a11y_bus.app_found_on_shared_a11y_bus: true` | NEW-EDGE | 3.1 / 3.2：无总线启动的应用永远不会在后来出现的总线上注册，因此快照在应用重启前无法看到它；此处应用注册在另一个（共享）a11y 总线上，因此适配器必须从 `GetAddress` 解析总线，不得假定每次登录一个总线。选择另一总线的机制未隔离 |
| L1-14 | 会话退出时的总线守护进程生命周期 | tool | api-contract | 无 | 每个场景：`cleanup.survivors_after_session_daemon_exit: []`，`cleanup.survivors_after_cleanup: 0`（启动器、registryd 和 a11y dbus-daemon 均随其会话总线退出） | NEW-EDGE | 3.1：激活总线后无需清理；测试可依赖其随私有会话结束 |
| L1-15 | 探针的活动会话隔离 | tool | box | 无 | `shared_x_property_before: "_AT_SPI_BUS:  no such atom on any window."`，`shared_x_property_unchanged: true`，`a11y_socket_under_private_runtime_dir: true` | NEW-EDGE | 3.1：GNOME 46 上共享 X 根窗口无 `_AT_SPI_BUS` 属性，因此适配器无法从 X11 根属性定位总线，必须使用 `org.a11y.Bus.GetAddress` |
| L1-16 | `toolkit-accessibility` 为 false 时的行为 | dbus | box | §3.1：GNOME 提示检查 `toolkit-accessibility` | 私有会话（自有 `XDG_RUNTIME_DIR`，私有 `XDG_CONFIG_HOME` 下的 `GSETTINGS_BACKEND=keyfile`；`memory` 后端是每进程的，因此 `gsettings set` 从未到达启动器，首次运行读取真实 dconf），探针拥有的 `gnome-calculator`，发布二进制 `status`，每个方向一次运行。false：`private gsettings: false`，`GetAddress` 正常，`IsEnabled: (False,)`，`registry child_count=2 names=['xdg-desktop-portal-gtk', 'gnome-calculator'] found_calc=True waited_s=0.5`，`status`：`"accessibility":{"state":"granted"}`。true：`private gsettings: true`，`IsEnabled: (True,)`，相同注册表和 `granted`。之后活动会话：`gsettings get ... toolkit-accessibility` `true`，`/run/user/1000/at-spi/bus_0` 存在 | NEW-EDGE | 3.1：`toolkit-accessibility=false` 不阻止 `Get