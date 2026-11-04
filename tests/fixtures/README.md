# 测试夹具

手工编写的黄金文件，测试用它们比对序列化输出。没有一个是来自真实应用的抓取。

| 夹具 | 断言方 |
|---|---|
| `skeleton-tree.json` | `crates/core/src/snapshot/snapshot_tests.rs` 中的 `test_skeleton_fixture_matches_golden` |
| `drilldown-refmap.json` | `crates/core/src/snapshot/snapshot_ref_merge_tests.rs` 中的 `test_drilldown_refmap_matches_golden_fixture` |
| `trace_show/trace/*.jsonl` | `crates/core/src/commands/trace_tests.rs` 中的 `show_merges_fixture_segments_with_expected_shape` |

JSON 夹具在编译期加载，因此文件缺失或改名会导致构建失败：

```rust
let golden = include_str!("../../../../tests/fixtures/skeleton-tree.json");
```

任何改动夹具所固定结构的变更，都必须在同一次提交中更新该夹具。