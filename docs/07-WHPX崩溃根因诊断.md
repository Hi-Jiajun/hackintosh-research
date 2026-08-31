# WHPX 崩溃根因诊断（Bug B 全链路）

状态：**部分解决**。QEMU 侧两个真实缺陷已修复并验证；guest 侧根因链已定位到
panic 打印路径的 bcopy_phys/PHYSMAP_PTOV，最初 panic 仍在追。

## QEMU 侧修复（Windows 树已应用）

### 修复 1：INIT IPI（whpx-apic.c）

whpx_send_msi 原先丢弃所有 vector==0 的中断写入。INIT IPI（delivery=5）合法携带
vector 0，被丢弃导致 AP 永不启动。改为仅对固定投递拒绝：

    if (vector == 0 && delivery == 0) { ... return; }

### 修复 2：#PF 注入（whpx-all.c）

WHPX 把非 GPF 异常 exit 映射为 EXCP_DEBUG（无调试器时被 core 吞掉，#PF 到不了
guest）。改为把 #PF 作为 PendingEvent 直接注入回 guest（whpx_inject_back_pf）。

两者都值得独立上游提交。

## Guest 侧完整链（xnu-8796.141.3 符号级）

[初始 panic] -> panic 日志拷贝路径 debug_copyin -> bcopy_phys
  -> PHYSMAP_PTOV bounds exceeded panic（pmap.h，physmap 全局未初始化）
  -> 二次 trap -> sync_iss_to_iks SMEP 检查
  -> panic: type = SMEP/User NX fault，fault RIP = 0
  -> panic 打印 __doprnt：movsbl (%rsi,%r13),%edi，其中 RSI == CR3 值
  -> #PF(cr2=CR3 值) -> vm_fault(map=kernel_map=NULL)  // vm_init 之前
  -> map->never_faults 解引用 NULL+0x9e -> 嵌套递归 -> WHPX exit 4

关键实锤：
- 最初 trap 被中断 RIP = _bcopy_phys+0x8e（loose_ends.c，PHYSMAP_PTOV panic 分支）。
- panic 类型字符串 SMEP/User NX fault @ 0xffffff8000b66b79。
- panic 帧参数 fault RIP = 0（内核 supervisor 从地址 0 取指）。
- 崩溃时 physmap_base/physmap_max 全局 = ASCII/XML 垃圾（pmap_bootstrap 前）。

## 已排除（崩溃逐字节相同）

- -accel whpx,hyperv=off（CPUID 0x40000000 变 VMwareVMware）
- -accel whpx,kernel-irqchip=off
- OpenCore quirks：ProvideCustomSlide / ProvideCurrentCpuInfo /
  debug=0x100 msgbuf=1048576
- INIT IPI 修复（真实缺陷但非本崩溃根因）
- XSAVE EBX 0x240->0x340、CET_SS 关闭（正确性修复，非根因）
- QEMU 11.1 WHPX v3 补丁集缺失（本树已基本合并：window_priority、
  InitialApicId、interrupt priority 均在）

## 取证方法（可复现）

1. QEMU 内插：PF-DIAG 限流日志 + 首条 #PF vm_stop。
2. QMP：info registers（16 vCPU）、pmemsave 物理内存。
3. 页表走读（CR3 起 4 级）+ RBP 链回溯 + llvm-nm 符号解析。
4. slide 定位：movsbl 模式在镜像中的位置（TEXT_VMADDR 0xffffff8000330000,
   FILEOFF 0x130000）。

脚本（WSL /tmp）：whpx-*.py；内核镜像：research/xnu-kernel。

## 未决

- 最初的 panic 原因（pmap_bootstrap 之前；栈上 GDT/IDT/TSS 现场待解析）。
- panic 打印 fmt 指针=CR3 的参数错位机制。
- 下一步候选：第二条 #PF 停止抓更深现场；KVM 通路同点对照；
  bootstrap 流程与 WHPX 差异排查。