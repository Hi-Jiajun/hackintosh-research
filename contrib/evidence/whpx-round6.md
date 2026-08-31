# 第六轮：真实 panic 可见——cpu_thread_alloc 除零（2026-08-31）

boot-args 加 wdt=-1 serial=3 -noprogress 后，serial 终于输出 panic 全文：

```
panic: Kernel trap at 0xffffff800bb9bfaf, type 0=divide error
  _cpu_thread_alloc+0x29f / _cpu_thread_init / _smp_init / _machine_startup
NULL thread->t_tro pointer
MACH Reboot（循环重启）
```

反汇编：div 链 = esi(cpu 编号=1) / nPThreadsPerCore → 商 → 再除 → #DE
（商=0，即 nPThreadsPerCore > 1，而 CPUID 0xB.0 报 ebx=1）。

XNU 查了 0xB idx=0（SMT:1）和 idx=1（core:16）——拓扑输入正常，
但 topoParms.nPThreadsPerCore 仍 >1：填充来源另有（ACPI/其他叶子），
或 smp_init 时序问题。

## 两种交替故障

1. divide error panic（serial 可见）→ MACH Reboot 循环。
2. task_restartable_ranges_synchronize 死锁（无 panic 输出）。

## 仍存疑

- AP 全部停在 OpenCore hlt（0x7f96b034, CR3=0x7f96a000）——从未进 XNU。
- topoParms 的填充代码在 XNU 非开源部分。

## 关键工具积累

- whpx-rippoll（WHvGet 轮询 vCPU RIP/CR3/RSP）——定位卡点。
- wdt=-1 + serial=3 让 XNU panic 文本可见——后续调试必备。