# 观察命令

读取 UI 状态、从不改变状态的命令。运行 `agent-desktop <command> --help` 查看参数。

## snapshot

将无障碍树捕获为 JSON。交互元素会获得带限定前缀的 ref（`@s8f3k2p9:e1`），其中内嵌了快照 id。裸 `@eN` 会以 `INVALID_ARGS` 失败。

```bash
agent-desktop snapshot --app "System Settings" -i --compact
agent-desktop snapshot --app "App" --skeleton -i --compact
agent-desktop snapshot --root @s8f3k2p9:e3 -i --compact
agent-desktop snapshot --app "App" --surface menu -i
agent-desktop snapshot --app "App" --window-id w-1234 -i
```

输出结构：

```json
{ "ok": true, "command": "snapshot",
  "data": { "app": "System Settings", "window": { "id": "w-4521", "title": "General" },
    "ref_count": 14, "snapshot_id": "s8f3k2p9", "complete": true,
    "tree": { "role": "window", "name": "General", "children": [
      { "ref_id": "@s8f3k2p9:e1", "role": "button", "name": "About", "states": ["focused"] } ] } } }
```

当 macOS 通过进程读取无窗口应用时，`data.window` 会被省略。在 Windows 和 Linux 上，同一应用返回 `WINDOW_NOT_FOUND`。

每个 ref 都标明其所属快照，因此当你交错操作多个应用或窗口时，ref 仍能针对自己的快照解析。

选择范围：

- `-i --compact` 只保留交互元素并丢弃空包装节点。默认两者都用。
- `-i` 保留交互节点并裁剪容器，因此某行的静态文本标签会被丢弃。要把重复按钮与其所在行关联，用 `--root` 深入该行的容器并省略 `-i`。在 Windows 和 Linux 上这能识别该行，但跨行重复的按钮仍是孪生，其 ref 会被拒绝。
- `--skeleton` 将深度限制为 3，并为被裁剪的容器添加 `children_count`。每个被裁剪的分支会暴露其最深且可安全解析的深入目标；若边界没有稳定标识，则暴露最近的可解析祖先。
- `--root <REF>` 从先前找到的 ref 开始。重新深入某个 root 会替换该 root 上次深入产生的 ref，以及在其下进行的任何深入产生的 ref。被深入子树之外的 ref 继续有效，其中的 skeleton ref 会被替换，新 ref 加入同一快照 id。深入会继承其快照的全窗口标识计数，因此其 ref 的解析方式与 skeleton 相同。`--root` 不能与 `--surface` 或 `--wait-for` 组合。
- `--max-depth`（默认 10）限制深度。用 `--root` 到达更深的子树。
- 仅在需要坐标时使用 `--include-bounds`。
- 使用 `--app` 且无 `--window-id` 时，`snapshot` 和 `find` 取应用唯一的可见窗口，否则取唯一的聚焦窗口；两者都无法确定时返回 `AMBIGUOUS_TARGET` 并附候选。应用有多个窗口时请传 `--window-id`。

### 部分树

每个快照都带 `data.complete`。观察预算耗尽时，命令仍会成功，返回 `"complete": false`、`"truncated": true`、`"nodes_observed"` 以及它看到的树。每个有被裁剪后代的节点带 `"subtree_truncated": true`。读取 `complete`，不要等 `TIMEOUT`。调大 `--timeout-ms`（默认 3000）或用 `--root` 深入以完成树。`--root` 深入是全有或全无：读取不完整会返回 `TIMEOUT`。

`"identity_unproven": true` 表示快照未能及时读取整个窗口以证明其 ref。在 Windows 和 Linux 上，以及 macOS 上无地址的元素，这些 ref 会以 `identity_not_unique` 失败，因此操作前请用更大的 `--timeout-ms` 重新快照。

冷启动的 Chromium 或 Electron 应用可能需要 10 到 25 秒才能暴露其 Web 树。若新快照内容稀薄，调大 `--timeout-ms` 再快照一次。

### 哪些 ref 存在

元素的角色为交互式或声明了主操作时才会获得 ref。`scrollarea` 和 `disclosure` 获得 ref，因为 `scroll`、`expand` 和 `collapse` 需要它们。静态文本和惰性容器留在树中但没有 ref。Web 内容几乎每个节点都声明了聚焦、右键和滚动到，因此仅凭这些不足以获得 ref。`find` 会为此类匹配返回 `bounds`。用 `--xy`（右键、悬停、拖拽）驱动它们，或滚动容器。

### ref 重新识别

命令在执行前会从 ref 重新识别元素。在 macOS 上 ref 携带元素自身的地址，因此完全相同的孪生元素各自保留其 ref。在 Windows 和 Linux 上，仅当 ref 是其快照捕获时窗口中唯一具有该角色、原生 id、名称和描述的元素，且仍恰好有一个存活元素匹配时，ref 才能解析。skeleton、深度受限或深入捕获都按整个窗口计数，因此浅树与完整树证明的是同一件事。值和边界从不计入标识。

- `STALE_REF`，`details.kind: saved_ref_absent`：没有存活元素匹配存储的标识。重新快照。macOS 上 `retained_object_stale` 表示记录的地址已失效或不再匹配。
- `AMBIGUOUS_TARGET`，`details.kind: saved_ref_ambiguous`：出现了相同元素，因此两个存活元素匹配。`identity_not_unique`：ref 在捕获时未被证明唯一，因此未做任何搜索。这是完全相同的孪生元素，每次快照都会再次拒绝；或是快照未能及时完整读取的窗口，用更大的 `--timeout-ms` 重新快照即可修复。
- `TIMEOUT`，`details.kind: resolution_identity_unknown`：实时读取未应答，因此标识未证明。未发送任何操作；重试。

非 macOS 上，完全相同的孪生元素没有 ref 路径：只有 `--headed mouse-click --xy X,Y` 能选择其一。它会抢占前台，Linux Wayland 会话会拒绝。

### 表面

表面是覆盖层：`menu`、`menubar`、`sheet`、`popover`、`alert`、`focused`，以及某些平台上的 OS shell 类型。请求前先从 `status` 读取 `supported_surfaces`。此 OS 没有的类型会返回 `INVALID_ARGS`，并在 `details` 中附支持列表。shell 表面无需 `--app`；macOS 上 `menu-bar-extras` 配合 `--app` 读取该应用自己的状态项。关闭时返回 `WINDOW_NOT_FOUND`；用 `open-system-surface` 唤起。

对话框和菜单用表面快照，不要用窗口快照。要到达覆盖层内的某一项，用 `find --surface` 而非完整转储。

## find

按角色、名称、值、文本、描述或原生 id 搜索。知道目标时优先用它。

```bash
agent-desktop find --app "App" --role button --name "OK" --exact
agent-desktop find --root @s8f3k2p9:e4 --role textfield --first
agent-desktop find --app "App" --surface menubar --name "Save" --exact --first
agent-desktop find --app "App" --role button --count
```

- 先缩小范围再扩大。`--root` 搜索一个 ref 的子树。`--surface` 搜索覆盖层。两者只花几百字节而非完整转储。它们不能组合。
- `--name` 和 `--text` 模糊匹配。加 `--exact` 进行不区分大小写的精确匹配。变更前若需确认匹配唯一，先用 `--limit 2`。
- `--text` 也匹配子树文本包含该文本的祖先。用 `--name` 匹配节点本身。
- `--role` 不区分大小写；`textarea`、`textbox` 和 `searchfield` 归并为 `textfield`。角色过滤无匹配时，`data.roles_present` 列出被搜索树中的角色。用它区分角色名写错与屏幕为空。
- 无名元素省略 `name`。其内容在 `value` 中，用 `--value` 搜索。
- 大树（如文件对话框）可能超过默认 5000 ms 的 `--timeout-ms`。调大它或缩小搜索。此处超时不代表应用卡死。

每个非计数响应都返回拥有其 ref 的 `snapshot_id`：

```json
{ "data": { "snapshot_id": "s8f3k2p9",
  "matches": [ { "ref_id": "@s8f3k2p9:e5", "role": "button", "name": "OK", "states": ["enabled"] } ] } }
```

无 `ref_id` 的匹配是上下文匹配：可读文本但无操作目标。平台报告时它带 `bounds`。带 `ref_id` 的匹配从不带 `bounds`；用 `get --property bounds` 读取。

## get

读取 ref 的一个属性：`text`（默认）、`value`、`title`、`bounds`、`role`、`states`。

- `title` 是无障碍名称。它回答"这个按钮写的是什么"。
- `text` 是人读到的内容。对 `textfield`、`combobox`、`listbox`、`datefield` 和 `timefield` 它是内容，其他角色则是名称。首选的一半为空时，另一半作答。复选框回答其标签，而非 `1`。
- `value` 是与角色无关的原始值（文本、滑块位置）。
- 适配器支持时，`text`、`value`、`bounds` 和 `states` 读取实时状态。缺失的实时值或边界返回 `null`。可选实时读取不可用时，`get` 可回退到保存的 ref 条目的值、名称、边界或状态。
- 只有 `bounds` 在 `data.live` 中报告是否实时。为 `false` 时矩形来自快照，可能已过期。传给 `mouse-click` 前先检查。

## is

检查一个布尔值：`visible`（默认）、`enabled`、`checked`、`focused`、`expanded`、`selected`。输出：`{ "ref", "property", "result" }`。对 `enabled`，未知的原生值返回 `result: false` 并带 `applicable: false`。这表示未知，而非禁用。

## screenshot

```bash
agent-desktop screenshot --app "Finder" out.png
agent-desktop screenshot --screen 0 display.png
```

无路径时，PNG 以 base64 返回在 `data` 中。`--screen` 接受来自 `list-displays` 的显示器索引；`0` 是主显示器。截图是你的证据，不是操作成功的证明。权限被拒返回 `PERM_DENIED`（见平台技能）。

## list-displays

列出显示器为 `{ id, bounds, is_primary, scale }`，主显示器在前。配合 `screenshot --screen` 时用数组索引，不要用 `id`。

## list-surfaces

```bash
agent-desktop list-surfaces --app "Finder"
```

列出应用当前呈现的覆盖层表面，即 `snapshot --surface` 接受的值。shell 表面属于 OS，从不在此出现。若窗口条目带 `"unclassified": ["sheet"]` 或 `["menu"]`，表示检查无法读取该窗口（常见于繁忙应用）。将缺失条目视为未知，而非不存在。

## open-system-surface

```bash
agent-desktop --headed open-system-surface --surface action-center
```

唤起 OS shell 表面。在 Windows 和 Linux 上返回其呈现的窗口，形式为 `list-windows` 使用的 `w-<id>`；macOS 上 `data` 仅为 `{"surface": "dock"}` 且省略 `window`，因为没有窗口拥有这些表面。它会抢占前台，因此在 Windows 和 Linux 上严格无头会被 `POLICY_DENIED` 拒绝；请传 `--headed`。macOS 上 `dock` 和 `menu-bar-extras` 始终在屏，因此无需 `--headed` 即可无头返回。已存在的表面返回而不再唤起。OS 未暴露的类型返回 `INVALID_ARGS` 并指明应改用哪些类型。`start-menu` 是 Windows 类型，`action-center` 存在于 Windows 和 Linux（GNOME）；两者都不是 macOS 类型。macOS 上用 `dock` 或 `menu-bar-extras`，并通过点击其 `menu-bar-extras` 项打开控制中心或通知中心。类型见平台技能。

## Chromium 和 Electron 应用

无障碍路径是默认路径。对已运行的应用以及原生菜单、对话框和窗口，它是唯一路径。对可全新启动的应用的 Web 内容，`launch --cdp` 加 CDP 客户端比 skeleton 遍历更快。见 `commands-system.md`。