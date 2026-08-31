# 第五轮：AP 未启动 + 单核死锁（2026-08-31）

## 新增观察（rippoll：WHvGet 直接轮询全部 vCPU）

1. 16 核：CPU0（BSP）在 XNU 内核跑（thread_setrun /
   task_restartable_ranges_synchronize），**CPU1-15 全部停在
   RIP=0x7f96b034, CR3=0x7f96a000**（OpenCore 的 AP hlt 循环）——
   **AP 从未进入 XNU**。
2. XNU 的 INIT/SIPI 从未经过 QEMU（whpx-msi 日志 0 条）：
   - kernel-irqchip=on：ICR 由 Hyper-V 处理；
   - kernel-irqchip=off：QEMU 模拟 LAPIC，但 whpx_send_msi 仍 0 条——
     **XNU 根本没写 ICR**（或写走 x2APIC MSR 路径未达 QEMU）。
3. 单核（SMP=1）同样卡死：RIP/RSP 连续多个轮询周期不变——
   卡点 unslid 0xffffff80003b6490 = _task_restartable_ranges_synchronize。
4. 栈回溯：choose_processor / thread_quantum_expire /
   ipc_thread_terminate——调度循环现场。
5. 去掉 #PF/#GP 拦截 → exit 4 立即回归——PendingEvent 注入是必须的。

## 当前假设

- WHvGet 在 VP running 时返回 stale（无 exit 即冻结）→ 真实死锁点在
  函数内某 spin（lck_mtx_lock 等锁）。
- 锁等待依赖中断（APIC timer 时间片到期 / IPI）——WHPX 下该中断
  未交付 → 死锁。
- 候选：QEMU 的 APIC timer 在 WHPX vcpu 阻塞时不被驱动；
  Hyper-V 的 synthetic LAPIC timer 未触发。

## 已验证的修复（累计）

- INIT IPI（vector 0）丢弃 → 修复。
- #PF 被 EXCP_DEBUG 吞 → PendingEvent 注入（必需，去掉即 exit 4）。
- SYSENTER MSR 写被丢弃 → 修复（boot 从必然崩溃推进到 task 层）。