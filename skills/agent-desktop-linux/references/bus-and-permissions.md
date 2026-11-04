# Linux 上的总线与权限

观察和语义操作无需权限对话框。AT-SPI 无障碍总线是否响应决定一切。

## CLI 如何找到总线

1. 连接到当前会话的 D-Bus 会话总线（来自 `DBUS_SESSION_BUS_ADDRESS` 或运行时目录）。
2. 在 `org.a11y.Bus` 上调用 `GetAddress`。该调用会自动启动总线启动器，因此尚未使用过无障碍功能的健康会话仍会响应。
3. 连接到获得的地址，并读取 AT-SPI 注册表根节点的 `ChildCount`。后续每条命令复用该连接，连接级失败后重新连接。

无障碍总线不是会话总线。用 `status` 判断健康状态，不要用 `busctl --user list` 或名称所有者检查：冷会话中 `org.a11y.Bus` 在首次 `GetAddress` 前没有所有者，因此健康会话看起来像缺失。

## 三种总线状态

| 状态 | `status` 显示 | 含义 |
|-------|----------------|---------|
| 已响应，健康 | `accessibility` 为 `granted`；`supported_surfaces` 列出界面；`list-apps` 返回行 | 正常 |
| 已响应，为空 | `accessibility` 为 `granted`；`list-apps` 不返回行 | 总线已启动但无应用注册。这是成功，不是故障。启动一个应用，或重启在总线存在前启动的应用 |
| 不可用 | `accessibility` 为 `denied` 并附建议；`supported_surfaces` 为空；命令以 `PLATFORM_NOT_SUPPORTED` 失败 | 无法访问会话总线或无障碍总线。`supported_surfaces` 为空是结果，不是独立问题 |

在无障碍总线存在前启动的应用，不会在之后出现的总线上注册：重启它。`toolkit-accessibility` GSettings 键为假不会阻止总线或应用工作。`IsEnabled` 属性不是诊断依据。

## 命令

在手动启动的会话中，首次使用时启动的 AT-SPI 注册表会在启动总线的进程终端上打印 `SpiRegistry daemon is running with well-known name - org.a11y.atspi.Registry`。这不是 CLI 输出：CLI 的 stdout 始终是一个 JSON 信封，成功时 stderr 为空。

```bash
agent-desktop status
agent-desktop permissions
```

| `permissions` 字段 | Linux 值 |
|---------------------|-------------|
| `accessibility` | 总线响应时为 `granted`；无法访问时为 `denied` 并附具体原因建议 |
| `screen_recording` | X11 上为 `not_required`；Wayland 上为 `unknown`（只有截图请求才会揭示答案） |
| `automation` | `not_required` |

`permissions --request` 在 Linux 上无可请求项。

## 总线不可用：四种原因

每种原因返回 `PLATFORM_NOT_SUPPORTED`、`details.retryable: false`、指明下一步的建议，以及 `platform_detail` 中的原始 D-Bus 错误。

| 原因 | 建议 | 含义 |
|-------|------------|---------|
| 无会话总线 | 在桌面会话内运行，或用 `dbus-run-session -- <command>` 启动一个 | `DBUS_SESSION_BUS_ADDRESS` 和运行时套接字缺失 |
| 未安装 AT-SPI | 安装 `at-spi2-core`；Flatpak：添加 `--talk-name=org.a11y.Bus`；snap：连接 `desktop-legacy` 或 `unity7` | `GetAddress` 返回 `ServiceUnknown` 或 `NameHasNoOwner` |
| 启动器无法启动 | 查看 `systemctl --user status at-spi-dbus-bus.service` | 启动器生成失败 |
| 套接字消失 | 重启 `at-spi-dbus-bus.service`，或注销后重新登录 | 无障碍总线套接字缺失或拒绝连接 |

超时是 `TIMEOUT`，不属于上述原因，也不能证明总线状态。会话中途死亡的总线返回 `INTERNAL` 并附重试建议。注册表读取、shell 所有者查找和权限探测会丢弃失败的缓存连接，以便后续调用重新连接。已建立的适配器在恢复期间不会自动启动替代总线，因此重复调用会一直返回 `INTERNAL`，直到无障碍服务重新运行。新适配器保持正常的总线激活。仅凭连接丢失无法证明操作是否已送达。除非操作的处置说明重试是安全的，否则不要重试。

在总线由其他进程拥有的机器上，不要执行重启建议。该建议适用于你自己的桌面。在共享主机上，共享运行时目录的第二个启动器可能删除活动总线套接字，破坏所有其他客户端。改用私有会话：

```bash
XDG_RUNTIME_DIR=$(mktemp -d) dbus-run-session -- agent-desktop status
```

私有的 `XDG_RUNTIME_DIR`（`mktemp -d` 使其权限为 0700）让新的总线启动器远离活动套接字。

## 会话要求

- 以拥有该会话的用户运行 CLI。其他用户或会话的进程不会按名称匹配。
- 目标应用必须运行在同一会话总线上。环境变量指向另一总线的进程视为另一会话的进程。
- 沙箱应用（Flatpak、snap）仅在沙箱共享总线时才在总线上。参见 `references/shell-tray-and-notifications.md`。

## 状态根目录

状态根目录（`~/.agent-desktop` 或 `AGENT_DESKTOP_HOME`）必须对用户私有。工具创建的根目录权限为 0700。组可写或对其他用户开放的根目录会使写入该目录的命令（例如 `snapshot`）以 `INTERNAL` 失败，消息为 "private file parent is accessible by group or other users"。`status` 仍可运行。修复已有根目录：

```bash
chmod 700 "${AGENT_DESKTOP_HOME:-$HOME/.agent-desktop}"
```

不要在 umask 为 002 时用 `mkdir -p` 预先创建它。

交互锁位于 `/tmp/agent-desktop-<uid>/`，在状态根目录之外。它按用户串行化输入。`TIMEOUT` 伴随 `lock_timeout` 和高 `contention_count` 表示同一用户的另一个 agent 正在驱动输入。