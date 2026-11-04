---
title: 打开器满足后立即等待 "closed" 会误判
date: 2026-10-19
category: runtime-errors
module: crates/linux/src/system/interaction/action_clock.rs, crates/linux/src/system/interaction/action_clock_store.rs, crates/linux/src/system/menu/menu_wait.rs
problem_type: runtime_error
component: atspi-menu-wait
symptoms:
  - "点击打开 GTK4 弹出菜单后，`wait --menu-closed` 立即返回，此时菜单尚未显示。"
  - "同一序列在单进程内运行 click 和 wait 时通过，拆成两次 CLI 调用时失败。"
tags: [linux, at-spi, gtk4, menu, wait, race, persisted-state]
---

## 问题

GTK4 弹出菜单在其打开器动作返回后 157 到 331 ms 才出现在无障碍树中（L18-2）。实时探测显示，Calculator 上 540 ms 打开，Files 上 688 ms 打开（L33-51）。打开器之后立即启动的 `wait --menu-closed` 轮询一个尚未可见的菜单，读到 "closed" 便返回。等待被它本应熬过的状态满足了。

## 解决方案

关闭方向的等待会持续到看见菜单打开，或距上次动作满 2 s，以先到者为准。`menu_wait.rs` 调用 `action_clock::recent_for(pid)`；时钟标记每个已启动的动作，即除发送前被拒绝的动作之外的一切（`action_clock::mark_outcome`）。

记忆必须跨进程存活。agent 将 `click` 和 `wait` 作为两次 CLI 调用运行，因此进程内的 `Instant` 在第二次调用中为空。`action_clock_store.rs` 在 `$XDG_RUNTIME_DIR/agent-desktop-actions` 下写入按进程的记录，时间戳为启动后的毫秒数（含挂起），使两个进程一致，且重启不会让旧记录显得新；并以进程实例令牌为键，使复用的 pid 不会继承它。最初的实时测试使用进程内时钟，10 次全部通过（L33-17）；只有双进程场景（L33-56）才会用到持久化存储。

## 预防

- 依赖先前动作的等待要在两个进程中测试，而非一个。使用进程内时钟的单元测试不会因此原因失败。
- 不要按单一工具包确定等待时长。弹出延迟按工具包测量（L18-2、L33-51）。
- 将 "尚未可见" 和 "已关闭" 视为不同状态；等待必须知道自己读到的是哪一种。