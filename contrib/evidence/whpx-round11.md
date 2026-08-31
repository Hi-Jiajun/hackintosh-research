# 第十一轮：AP 启动之谜——XNU 从不自己启动 AP（2026-08-31）

## 关键结构事实（flash subagent 源码级实锤）

- xnu x86 从不自己启动 AP：唯一入口是闭源 AppleACPIPlatform.kext
  调 ml_processor_register()（注册遍 start=FALSE → ml_set_max_cpus →
  启动遍 start=TRUE → cpu_topology_start_cpu → i386_start_cpu → ICR）。
- i386_start_cpu 第一条语句就是 LAPIC_WRITE_ICR(DM_INIT)（mp_native.c:74）。
- QEMU 侧 0 条 ICR = 执行流从未到达 intel_startCPU()。

## 观测

- serial 无任何 cpu_topology / cpu_data_alloc / Started cpu 日志
  （-topo boot-arg 已加）。
- guest 已过 ml_wait_max_cpus（跑到 in6_mc_join 之后）⇒ kext 是活的。

## 最可能

G0（kext 从未发起启动遍）或 G2（max_cpus≤1，启动遍静默跳过每个 AP）：
两者都零日志、零 panic、零 ICR、boot 继续。

## 下一步候选

1. 验证 -topo 生效 + 完整 serial（KVM 对照的 cpu_data_alloc 行）。
2. DSDT 检查：AP 的 Processor/ACPI0007 对象 + _STA 与 MADT 是否一致
   （AppleACPIPlatform 判断可启动 CPU 数的依据）。
3. WHPX 的 ACPI 表传递与 KVM 的差异对比。