---
name: jev-desktop
description: >
  用一个自然语言目标驱动桌面应用，或一次只解决一个意图，全程不读取无障碍树。适用于自动化桌面应用、又不想让 150 个元素的 JSON 树占用上下文的场景。把完整目标交给 `run.mjs`，它会观察、决策并执行，直到目标达成；想自己掌握计划时，把单步交给 `act.mjs`。涵盖选哪个元素、用哪种操作、何时深入查看、何时停止。需要克隆 agent-desktop 仓库，并配置 `TYPESAFE_API_KEY`。
---

# jev-desktop

本技能需要克隆 agent-desktop 仓库（脚本位于 `scripts/jev/`，不随任何已安装的包分发），并配置 `TYPESAFE_API_KEY`。

两个入口共用同一套屏幕读取。`run.mjs` 接收一个目标并驱动执行直到达成。`act.mjs` 接收一步并把计划交还给你。两者都不会把树放进你的上下文。

## 循环

```sh
node scripts/jev/run.mjs --app Finder "open the Applications folder"
node scripts/jev/run.mjs --app TextEdit --cursor --text "A line to write." "write the supplied sentence into the document"
```

每一轮读取屏幕，在同一次请求中询问一个操作及该操作的目标，执行，然后再次读取。每轮打印一行 JSON，停止时再打印一行（已裁剪）。

```json
{"turn":{"step":1,"operation":"CLICK","target":"treeitem \"Applications\"",
         "confidence":1,"ok":true,"delivery":"delivered_verified","changed":true}}
{"stop":"done","confidence":0.96}
```

操作有 `CLICK`、`TYPE_TEXT`、`CHECK`、`UNCHECK`、`EXPAND`、`COLLAPSE`、`SCROLL`、`DRILL`、`WIDEN`、`WAIT`、`DONE` 和 `BLOCKED`。只提供屏幕上某元素能接收的操作：`DRILL` 仅用于报告"还有更多内容"的区域，`WIDEN` 仅在 `DRILL` 之后提供。`SCROLL` 始终向下滚动。`delivery` 对 `WAIT` 为 `waited`，对 `DRILL` 和 `WIDEN` 为 `looked`。

**每个操作只询问一次目标。** `click_target` 从声明 `Click` 的元素中选，`type_text_target` 从声明 `SetValue` 的元素中选，依此类推。只读取与所选操作匹配的那部分，因此目标绝不会与作用于它的动词不兼容。

**查看也是一种操作。** 窗口先用 `--skeleton` 读取，因此密集文档的首次查看成本与小型面板相同。被截断的区域会报告其内容量，`DRILL` 将其固定为后续轮次的根。`WIDEN` 恢复整个窗口。

**用 `CHECK` 和 `UNCHECK` 代替切换。** 两者都是幂等的：已处于目标状态的复选框保持不变。

**文本由你提供。** 这里不写入任何值，因此运行绝不会把你不选择的字符串放到屏幕上。每个值传一次 `--text`，按顺序消费；或传一个函数作为 `text` 选项，每个字段会用该字段的描述来询问它。没有剩余值可提供时，`TYPE_TEXT` 完全不提供，运行会停止而不是编造一个。

**文本通过应用接受的途径写入。** 直接写值（`set-value`）是一次经过验证的调用。只有当应用拒绝且拒绝信息表明可以安全重试时，才通过剪贴板和 `cmd+v` 粘贴（仅 macOS）。运行结束时恢复剪贴板，若其中不是文本则清空。从不使用逐字符按键：会丢字符、丢大写。

**离开本机的内容。** 每一轮都会向决策端点（默认 TypeSafe 的 `api.typesafe.ai`，或 `TYPESAFE_BASE_URL` 指向的地址）发送屏幕描述：每个元素的角色、无障碍名称或描述、其值最多六十个字符、状态、窗口标题，以及最近的操作，排除提供的文本和详细执行证据。这足以把私有文档或已填写表单的内容发送到配置的远程服务。安全文本字段已由 agent-desktop 扣留，绝不会进入请求。给任一脚本传 `--no-values` 可同时扣留其他所有字段的值；定位会更难，因为值往往是区分两行未命名行的唯一依据。默认包含值。`run.mjs` 会从操作轮次及后续最近操作历史中隐去显式提供的字符串，包括后续目标名称或执行证据重复它们的情况。实时屏幕描述是独立的：如果应用显示了该文本，其无障碍名称或字段值仍可能按上述设置到达远程服务。`act.mjs` 在 stdout 中保留其精确的可执行 `argv`，因此该字段可能包含通过 `--text` 提供的值；stdout 也可能包含无障碍名称和元素路径。选择端点和输出目的地时请考虑这一暴露面。

**难以撤销的步骤不会悄悄执行。** 普通步骤需要目标置信度 0.70，被评为破坏性的（0.50 及以上）需要 0.90，低于 0.55 不执行任何操作。未达标时运行停止并列出它本会操作的候选，由你决定。`act.mjs` 在其单次请求中获得评级。`run.mjs` 用第二次模型调用（计入 80 次）评估风险，仅针对置信度 0.70 到 0.90 的目标，因此默认情况下目标置信度 0.90 及以上的破坏性步骤不经评级直接执行。若该调用未返回数字，该步骤计为破坏性。给 `run.mjs` 或 `act.mjs` 传 `--confirm-destructive`，可在每个被评为破坏性的步骤上停止等待确认，无论置信度如何。

**`--cursor` 让运行可观察。** 它启动一个会话，显示一个光标在操作生效前移动到每个元素，结束时关闭。若会话无法启动，运行继续但不显示光标。光标绘制在元素所在位置，因此请把应用置于终端前面，否则光标会出现在终端后面。

它在以下情况停止：`DONE`、`BLOCKED`、置信度低于风险要求、40 次操作后（`WAIT`、`DRILL` 和 `WIDEN` 计入）、80 次模型调用后（风险评级计入）、或连续三轮无变化（`WAIT` 轮不计入）。操作失败时以 `action_failed` 停止，保留错误码和投递状态，且绝不重复。聚焦、剪贴板写入或粘贴失败同样结束运行。二进制在 `set-value` 失败时未报告错误处置，则以 `BINARY_TOO_OLD` 失败。

可操作元素超过一次选择所能承载的屏幕会在请求和其报告的轮次中说明（`truncated`），策略被告知要查看区域内部，而不是判定目标不可能。

`--root @ref` 在你已知区域时从该区域内部开始。

## 一次一步

你掌握目标、计划和记忆。你发送一句话。你得到一个小对象。

```sh
node scripts/jev/act.mjs --app TextEdit --execute \
  --text "Morning over the dock." "type this text into the main writing area"
```

```json
{
  "ok": true,
  "intent": "type this text into the main writing area",
  "app": "TextEdit", "window": "Untitled 3", "window_id": "w-4521", "surface": "sheet",
  "element": { "ref": "@s1:e139", "role": "textfield", "name": "Save As:",
               "where": "sheet \"save\" > group" },
  "command": "set-value",
  "argv": ["set-value", "@s1:e139", "poem.txt"],
  "decision": "act",
  "why": null,
  "confidence": { "target": 0.99, "command": 0.88 },
  "gates": { "present": 0.94, "destructive": 0.31, "needs_text": 0.87 },
  "runner_up": [{ "ref": "@s1:e137", "what": "textfield", "p": 0.02 }],
  "notes": [],
  "executed": { "ok": true, "delivery": "delivered_verified" }
}
```

## 调用

| 参数 | 含义 |
| --- | --- |
| `--app <name>` | 必填。 |
| `--window-id <id>` | 读取应用的一个窗口，id 由 `agent-desktop list-windows --app <name>` 报告。仅当应用有多个可见窗口且都未聚焦时需要，否则会以 `AMBIGUOUS_TARGET` 失败。运行停留在该窗口：运行在别处打开的窗口不会被读取。若窗口在操作后关闭，`run.mjs` 报告该轮并以 `window_closed` 停止。窗口外的 `--root` 或打开的 surface 会给出 `root_outside_window` 或 `surface_outside_window`（`run.mjs` 中为停止，`act.mjs` 中为 `error`）。每个 `run.mjs` 停止都在 `screen.window_id` 中报告它读取的窗口。`run.mjs` 也接受该参数。 |
| `--no-values` | 从决策请求中扣留无障碍字段值。两个脚本都支持；默认包含值。 |
| `--execute` | 仅 `act.mjs`。当 `decision` 为 `act` 时运行命令。不加则什么都不运行。 |
| `--text "…"` | 意图所需的文本。Jev 返回选项，绝不返回字符串。`run.mjs` 可重复传；`act.mjs` 只接受一个值。 |
| `--root @ref` | 在一个容器内解析，而非整个窗口。 |
| `--bin <path>` | 仅 `act.mjs`。agent-desktop 二进制。默认用 release 构建，然后是 `PATH`。`run.mjs` 改为读取 `AGENT_DESKTOP_BIN`，默认相同。 |

环境变量：`TYPESAFE_API_KEY` 必填（两个脚本缺失时以 `TYPESAFE_API_KEY unset` 失败）。`TYPESAFE_BASE_URL` 和 `TYPESAFE_MODEL` 见下文。`JEV_VERIFY_MS` 设置粘贴回读超时（毫秒，默认 2000，范围 0 到 60000）。

`argv` 是精确的 agent-desktop 命令。`double-click` 和 `hover` 带 `--headed`；`scroll` 始终向下。

**把意图表述为动作，而非元素。** `"type this text into the main writing area"` 解析为 `type`。`"the main writing area"` 未指明操作，可能解析为 `focus`。像人一样描述目标，`"the field holding the name the file will be saved under"`，而不是按树的命名方式。

## 解读 `decision`

| `decision` | 含义 | 你该做什么 |
| --- | --- | --- |
| `act` | 一个元素明确匹配，且置信度达到该操作风险的门槛。 | 无需操作。带 `--execute` 时已执行。 |
| `confirm` | 匹配合理但低于门槛。 | 询问用户，或改写意图后再次调用。 |
| `abstain` | Jev 回答 `none`，元素可能不在本屏幕，或两个元素同样匹配。 | 屏幕不是你以为的位置。打开所需 surface 后再次调用。 |
| `needs_text` | 命令需要文本但未提供。 | 带 `--text` 再次调用。 |

`why` 对除 `act` 外的每个 `decision` 用一句话给出原因（`act` 时为 `null`）。`runner_up` 显示它还考虑了哪些，通常足以区分错误屏幕和模糊意图。

## 脚本决定的事，而非 Jev

- **风险决定门槛。** 答案说明做什么；置信度说明是否执行。普通操作需要 0.70。Jev 评为 `destructive` 且达 0.50 及以上的需要 0.90：写文件、删除、发送、确认警告。低于 0.55 完全不执行。
- **命令与元素核对。** 两个问题并行回答，彼此不可见，因此代码会核对所选动词与元素声明的操作。只读组合框得到 `click`，绝不是 `set-value`。`--text` 载荷配非文本动词会纠正动词，因为调用方提供文本是 Jev 没有的证据。
- **打开的 surface 优先。** 当 sheet、alert、菜单或 popover 弹出时，脚本先读窗口，再读该 surface，并提供 surface 的元素而非窗口的。在窗口树中这些元素带 `offscreen`，会被全部丢弃。
- **禁用、隐藏和无操作的元素从不提供**，treeitem 内的单元格也不提供（旁边的行会操作）。其他都提供，包括未命名的行：行靠其持有的值区分，Choice 用完整列表比用短名单表现更好。

## 请求的结构

`act.mjs` 发送一次调用，含五个问题。它们并行回答，因此推测性问题消耗 token 而不增加延迟。

| 问题 | 类型 | 询问 |
| --- | --- | --- |
| `target` | choice | 意图指哪个元素，外加 `none`。 |
| `command` | choice | 它要求 16 个交互动词中的哪个。 |
| `present` | noul | 该事物是否在此屏幕上？ |
| `destructive` | noul | 这是否难以撤销？ |
| `needs_text` | noul | 这是否需要调用方提供文本？ |

Choice 接受 255 个选项，因此一次最多放入 254 个元素（两个脚本）。当 `act.mjs` 首次通过低于 0.70 时，前五名会用更丰富的描述重新询问。元素超过 254 个的屏幕会在 `notes` 中说明；使用 `--root @ref`。

## 通过 OpenRouter 决策

决策端点是 `TYPESAFE_BASE_URL`，默认 TypeSafe 原生的 `https://api.typesafe.ai/v1/systemone`。配置的端点必须使用 HTTPS；HTTP 仅允许 `localhost` 或字面回环 IP，以保持本地测试路由可用。URL 凭据被拒绝，重定向失败关闭，因此 bearer key 无法转发到其他端点。将其指向 OpenRouter 的 Decisions 路由，即可用 OpenRouter key 作为 `TYPESAFE_API_KEY`：

```sh
TYPESAFE_BASE_URL=https://openrouter.ai/api/alpha/decisions \
TYPESAFE_MODEL=typesafe/jev-1.13 \
TYPESAFE_API_KEY=sk-or-... \
node scripts/jev/run.mjs --app Finder "open the Applications folder"
```

`TYPESAFE_MODEL` 可保持 `jev-latest` 默认值：对 OpenRouter，裸名称会作为注册的 `~typesafe/<name>` 别名发送。或固定一个路由 id 如 `typesafe/jev-1.13`。不要设 `typesafe/jev-latest`：OpenRouter 返回 `400 Model does not exist`。OpenRouter 的 Decisions 端点处于 alpha，形态可能变化。

## 已知限制

- `act.mjs` 每次调用读取屏幕一次。`run.mjs` 在每个操作前后读取，`DRILL` 或 `WIDEN` 后跳过读取。无固定区域时，sheet、alert、菜单或 popover 打开时每次读取会取第二次快照。密集应用上一次读取约 3 秒；`--root` 便宜得多。
- `act.mjs` 不重试：429 结束调用。`run.mjs` 对 429、503 和 529 重试两次，分别在 500 ms 和 1000 ms 后，然后失败。
- 不支持按住输入，因此没有持续按键，也没有按住拖拽。见 `crates/core/src/commands/input_hold_policy.rs`。
- `press` 不在此解析。它不需要元素，直接发送即可。

## 文件

| 文件 | 内容 |
| --- | --- |
| `scripts/jev/policy.mjs` | 操作、屏幕如何描述、何时停止。此处不接触任何应用。 |
| `scripts/jev/screen.mjs` | 命令能到达哪些元素、哪个 surface 打开、元素如何描述。两个入口共用。 |
| `scripts/jev/desktop.mjs` | 唯一与 agent-desktop 通信的代码。 |
| `scripts/jev/run.mjs` | 循环及其命令行。 |
| `scripts/jev/act.mjs` | 单步解析器。 |

## 检查

`run.mjs` 中的 `WAIT` 在再次读取屏幕前暂停 250 ms。失败的操作立即以 `action_failed` 停止运行，保留错误码和投递状态，且绝不重复。聚焦、剪贴板写入或粘贴失败同样结束运行。`set-value` 失败时未报告错误处置的二进制以 `BINARY_TOO_OLD` 失败。

`run.mjs` 仅在运行于 `done` 停止时退出 0；其他任何停止及任何错误（`{"ok":false,"error":…}`）退出 1，用法错误退出 2。`act.mjs` 除非执行的命令失败或未执行（`decision` 非 `act` 时 `executed.code` 为 `NOT_EXECUTED`）否则退出 0；错误退出 1，用法错误退出 2。

粘贴后回读字段。若未持有请求的文本，运行以 `TEXT_VERIFICATION_FAILED` 停止，不再输入。回读比较精确字符串，因此规范化文本的字段（裁剪、改变大小写或换行）即使文本已落位也会报告失败。读取会重试直到 `JEV_VERIFY_MS` 超时后才失败。

```sh
node scripts/jev/act.test.mjs
node scripts/jev/run.test.mjs
node scripts/jev/run.integration.test.mjs
node scripts/jev/evidence.test.mjs
```

轮次为调用方保留原生 `steps`、`post_state` 和 `details`