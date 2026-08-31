# 第十轮：TSC-deadline 隐藏修复，boot 到 kdp_core（2026-08-31）

## 第 5 个 QEMU 修复（未 commit）

irqchip=on 时 Hyper-V 的 LAPIC 模拟暴露 TSC-deadline 能力但写 LVT timer
（x2APIC 0x832 = 0x400dd）时 #GP → lapic_init panic。
修复：CPUID.1.ECX 隐藏 TSC-deadline 位（CPUID_EXT_TSC_DEADLINE_TIMER），
XNU 回退 one-shot 模式。

结果：lapic_init #GP 消失；boot 推进到 kdp_core zlib 初始化（比之前深）。

## 路径对比

- kicoff：QEMU 模拟 LAPIC，timer 中断不触发（WHvRun 无 timeout）→ 卡。
- irqchip=on：Hyper-V 模拟 LAPIC，TSC-deadline #GP → panic（已修）。
- 修复后：boot 到 kdp_core 后卡（新位置，flash subagent 分析中）。

## 当前卡点

CPU0 RIP=0xffffff80035aa839（连续不变），serial 停在 kdp_core zlib。