# 适配器可靠性一致性测试

每个平台适配器都必须满足相同的 ref 与 action 契约。测试基于 `PlatformAdapter` 语义编写，因此 macOS AX、Windows UIA 和 Linux AT-SPI 共用同一套预期。

可复用的命令路径辅助模块是 `tests/conformance/ref_action_contract.rs`。`window_identity_contract.rs` 覆盖窗口标识，`macos_notification_contract.rs` 覆盖通知（仅 macOS）。`src/tests/conformance.rs` 挂载这三者，并对 mock 适配器运行 ref 与 action 辅助模块。驱动真实适配器的 fixture 可以用适配器提供的 ref 调用同一辅助模块。

## 必需门禁

| 领域 | 必需行为 |
|------|----------|
| 快照 ref | ref 按深度优先、带快照限定（`@<snapshot_id>:eN`）；裸 `@eN` 会被拒绝并返回 `INVALID_ARGS` |
| 严格解析 | 仅当标识仍然匹配时 ref 才解析成功；过期 ref 返回 `STALE_REF` |
| 歧义 | 多个可能匹配返回 `AMBIGUOUS_TARGET`，绝不任意点击 |
| 可操作性 | ref 动作在派发前检查实时可见性、稳定性、启用状态、支持的动作、策略和可编辑性 |
| 等待恢复 | `wait --element` 轮询其限定 ref 中嵌入的快照，解析时遵守调用方超时，并报告最后观察到的谓词状态 |
| 快照作用域 | 限定 ref 在所选会话命名空间内解析其自身快照，绝不搜索其他会话 |
| 追踪 | 会话清单带 `trace: on`（来自 `session start`）时，命令在会话目录下按进程写入 JSONL 分段。`--trace <path>` 覆盖为单文件。除非 `--trace-strict`，否则为尽力而为。 |
| 会话生命周期 | `session start/end/list/gc` 管理清单、当前会话指针和追踪目录 |
| FFI 一致性 | FFI ref 动作在适配器派发前使用严格解析和可操作性检查 |

## 平台矩阵

| Fixture | macOS AX | Windows UIA | Linux AT-SPI |
|---------|:--------:|:-----------:|:------------:|
| 两个相同按钮产生歧义 | 必需 | 必需 | 必需 |
| 快照后 ref 消失 | 必需 | 必需 | 必需 |
| 禁用按钮在派发前阻止点击 | 必需 | 必需 | 必需 |
| 文本字段支持 value/type 可操作性 | 必需 | 必需 | 必需 |
| 会话 A 的最新 refmap 对会话 B 不可见 | 必需 | 必需 | 必需 |
| 批量项会话覆盖继承的会话 | 必需 | 必需 | 必需 |
| FFI `AdRefEntry` 保留完整标识信封 | 必需 | 必需 | 必需 |

## 添加平台

添加适配器专属的集成 fixture，但保持预期错误和 JSON 形状一致。优先使用语义平台动作（`AXPress`、UIA Invoke、Value 和 Selection 模式、AT-SPI actions）。坐标输入置信度较低，是回退方案，因此必须在策略或命令选择中显式声明。

## Windows 私有存储门禁

用原生测试验证 Windows 私有文件行为。仅交叉编译不构成证据。Windows CI 通道原生运行这些测试套件：

| 行为 | 原生测试 |
|------|----------|
| 作为路径组件、父级或叶子的 junction 被拒绝，且其目标处不会写入任何内容 | `crates/windows/src/system/private_file/path_tests.rs` |
| 写入持有已验证祖先的 no-follow 句柄期间，该祖先不能被重命名 | `path_tests.rs` 中的 `a_pinned_directory_cannot_be_renamed_while_its_no_follow_handle_is_held` |
| 原子写入、追加和有界读取时拒绝外部所有者 | `owner_tests.rs` |
| 新建构建产物在创建和替换过程中保留继承的 SYSTEM、Administrators 和用户授权，且绝不含 `BUILTIN\Users` | `owner_tests.rs` 中的 `a_profile_artifact_keeps_system_admins_and_user_and_never_users_across_create_and_replace` |
| 远程或无法确定的存储拒绝写入和追加接口 | `locality_tests.rs` |
| 原子替换在存在打开的读取者和并发写入者时成功，且不留下临时残留 | `replace_tests.rs` |
| 二进制和 FFI 适配器安装加固接缝，且通过 junction 状态根执行 `session start` 被拒绝，目标下不写入任何内容 | `src/tests/windows_private_file_install.rs`、`crates/ffi/tests/c_abi_windows_private_file_install.rs` |

模块创建的每个私有目录都会获得受保护的 DACL（`dacl::protect_directory`），授予 SYSTEM、Administrators、令牌用户和符合所有者条件的 SID。`dacl::require_private` 在使用前检查该 DACL。在此类目录中创建的文件会继承这些授权。`owner_tests.rs` 中的测试 `a_profile_artifact_keeps_system_admins_and_user_and_never_users_across_create_and_replace` 固定了这一点。