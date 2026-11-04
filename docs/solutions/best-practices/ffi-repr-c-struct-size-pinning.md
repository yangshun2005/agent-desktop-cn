---
title: 在 FFI 边界的每一层固定 repr(C) 结构体大小
date: 2026-06-10
category: best-practices
module: crates/ffi
problem_type: best_practice
component: ffi
severity: high
applies_when:
  - 在 crates/ffi 中新增或修改公开的 repr(C) 结构体
  - 一个嵌套的 repr(C) 结构体按值嵌入另一个结构体（大小变化会静默传播）
  - 编写或更新已提交的 C ABI 头文件（include/agent_desktop.h）
  - 在 crates/ffi/tests/ 中新增 FFI 集成测试
  - C、Swift、Python、Go 或 Node 使用方在栈上或固定缓冲区中分配该结构体
tags:
  - ffi
  - abi
  - repr-c
  - struct-layout
  - static-assert
  - compile-time-pin
  - c-header
  - memory-safety
---

# 在 FFI 边界的每一层固定 repr(C) 结构体大小

## 背景

一个结构体按值嵌入另一个结构体时，外层结构体的大小取决于内层结构体。增大内层结构体会静默地增大外层结构体。此时按旧布局编译的 C 调用方会分配不足，Rust 读取会越过调用方的缓冲区。没有固定机制时，CI 会放过这类改动。

## 指导

对每个跨越 ABI 的 `repr(C)` 结构体，在四层固定其大小。每一层为不同的使用方捕获偏移。

1. **Rust。** 一个 `pub const` 大小、一个针对 `size_of` 的 `const _` 断言，以及一个 `extern "C"` 大小 getter（`crates/ffi/src/types/action.rs`）。
2. **C 头文件。** 大小宏由 Rust 常量生成。`_Static_assert` 行来自 `crates/ffi/cbindgen.toml` 中的 `trailer`。
3. **集成测试。** 在 `crates/ffi/tests/c_abi_layout.rs` 中检查大小、对齐、字段偏移和零值读取。
4. **使用方冒烟测试。** `tests/ffi-python/smoke.py` 通过 ctypes 加载动态库，并将每个 `ad_*_size()` getter 与其头文件宏比对。

trailer 是唯一没有生成器支撑的一层。新增固定结构体却不在 trailer 中添加对应的 `_Static_assert` 行，不会有任何提示。`cbindgen --verify` 从同一个 trailer 重新生成（`.github/workflows/ci.yml:739-742`），因此 CI 保持通过，而 C 侧的固定并不存在。在用 `scripts/update-ffi-header.sh` 重新生成头文件之前，先把断言行加入 `crates/ffi/cbindgen.toml`。

当一个已固定的结构体按值嵌入另一个结构体时，用同样的方式固定外层结构体。当使用方在运行时加载库时（例如 Python ctypes、Go cgo 或 Swift 不安全指针），必须提供 extern 大小 getter。