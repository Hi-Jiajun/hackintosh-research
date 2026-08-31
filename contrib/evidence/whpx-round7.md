# 第七轮：除零精确值——cpu_data 拓扑字段异常（2026-08-31）

#DE 拦截停止现场（QMP 4493）读取：

- R15 = 0xffffff800690a100（cpu_data）
- [r15+0xc8] = 1
- [r15+0xcc] = 16
- [r15+0x190] = 16
- [r15+0x194] = 16

cpu_thread_alloc 的计算（反汇编实锤）：

  topoParms.nLThreadsPerCore = [0x194] / [0x190] = 1
  topoParms.nPThreadsPerCore = [0xcc] / [0xc8] = 16/1 = 16

除零链：esi = [r15+0xc8] = 1；1 / 16 = 0；div 0 → #DE。

即：WHPX 下 cpu_data 的 +0xc8/+0xcc 字段 = 1/16，
XNU 算出 nPThreadsPerCore = 16（CPUID 说每核 1 线程），
后续除法商为 0 触发除零 panic。

## 待对照

KVM（正常 boot）下同字段的值——正在对照中。