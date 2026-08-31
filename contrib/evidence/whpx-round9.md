# 第九轮：leaf4 修复成功，除零消失；卡点移到 AP/IPI ack（2026-08-31）

## leaf 4 修复（第 4 个 QEMU 修复，已 commit/push 到 whpx-fixes）

cpuidExitList 加 0x4：Hyper-V 的 leaf4 回答退化（首 subleaf type==0），
XNU 的 cores_per_package 兜底=1；QEMU 回答后 EAX[31:26]=0xF → 16。

结果：除零 #DE 消失；guest 推进到：
- VM bootstrap、调度器初始化（timeslicing quantum 10000us）、
- Mach 对象（mig_table 54 / mach_kobj 374）、
- TSC Deadline Timer enabled。

## 死锁机制实锤（subagent 3）

task_restartable_ranges_synchronize 死锁 = 等目标 CPU 的 AST_RESET_PCS
IPI ack（hw_wait_while_equals32 无限自旋，lock_panic_timeout 默认 0 无 panic）。
排除锁重入自死锁（task 锁在等待时显式释放 + in_flight 位短路）。

## 当前卡点

BSP 卡在 _sched_ipi_action+0x6ac（单核 SMP=1 同样位置）：
XNU 认为存在多处理器（leaf4 报 16 cores），调度时给 AP 发 IPI 等 ack，
但 AP 从未启动（15 个 AP 停在 OpenCore hlt 0x7f96b034）——
INIT/SIPI 未发出或未交付（whpx-msi 日志 0 条）。

## 下一步

- 验证 XNU 是否发 INIT/SIPI（QEMU 侧对 x2APIC ICR MSR 0x830 / ICR MMIO 加日志）。
- AP 启动的前置条件排查（APIC base、INIT 交付）。