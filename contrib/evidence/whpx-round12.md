# 第十二轮：stall 检测器 + MSR 0x1a0 (IA32_MISC_ENABLE) 缺口（2026-08-31）

## stall 自动诊断器（Windows 树诊断代码）

- whpx_vcpu_run 的 5s rippoll 升级：连续 3 轮（15s）全部 vCPU RIP 不变
  且至少一个 vCPU 处于 XNU 内核空间（>= 0xffffff8000000000）时,
  vm_stop(RUN_STATE_PAUSED) 暂停 VM。
- 门控原因：OVMF/OpenCore 空闲 hlt 循环会误触发（首版单发式已弃用）。
- vm_stop 后其余 vCPU 走 Canceled 退出挂起 VP,之后 WHvGet/gdb 读数准确。

## 抓到的状态（stall 触发时, gdbstub 读寄存器）

- CPU0 RIP=0xffffff800ed01057, eflags=0x46（IF=0 中断关闭）
- slide=0xE700000（0x100000 对齐, 经 disasm 指令边界验证）
- unslid RIP=0xffffff8000601057 = `_rt_setgate+0x327` 内 `xorl %edx,%edx;
  callq _sa_copy`（网络路由表 gate 设置, 与上一轮 _in6_mc_join 同属
  网络初始化链）
- 15 个 AP 仍停在 OpenCore hlt 循环 0x7f96b034（AP 从未启动, G0/G2 待查）

## cont 后的新缺口：RDMSR 0x1a0 → #GP → panic

- QMP cont 后 guest 继续跑（说明此前并非永久卡死,而是 IF=0 长无退出阶段
  被检测器捕获——之后 kext 加载列表打印, AppleACPIPlatform 6.1 等）
- WHPX-GPF-DIAG: rip=0xffffff800f1b4daa（__PRELINK_TEXT 区 kext 代码）,
  ibytes=`0f 32 a8 01 74 56 ...` = rdmsr; test $1,%al; je ——
  Lilu/WhateverGreen 探测 Fast Strings 位
- QEMU WHPX MsrAccess 兜底：0x1a0 非已知 MSR 且未开 ignore-unknown-msr
  → 抛 #GP → guest panic → MACH Reboot

## 修复（Windows 树）

- MsrAccess exit 分支 + whpx_simulate_rdmsr/wrmsr 各加
  MSR_IA32_MISC_ENABLE（TCG 语义：读回 env->msr_ia32_misc_enable,
  写直接存入; 默认值 MSR_IA32_MISC_ENABLE_DEFAULT=1）


## 追加：CPU0 真实卡点 = rep movsb 无限 #PF 循环（确定性）

- 两次 boot CPU0 都停在 0xffffff8000101057（unslid）：
  `movq %rdi,%rax; movq %rdx,%rcx; cld; rep movsb; retq` ——
  xnu 的 memcpy 核心（_bcopy/_bcopy_no_overwrite 共用）
- slide 判定修正：0xAF00000（boot 2）；0xE70000 是 boot 1 误判
  （真实指令由 PF-DIAG ibytes=f3a4... 定死）
- #PF：cr2=0xffffffea3b318000（=rdi, 16KB 复制目标）, err=0x2
  （写、页不存在）。rcx 恒 0x4000 = 零进展死循环
- 调用链：OSCollection::iterateObjects block → bcopy → rep movsb
  （IOKit 初始化阶段, serial 停在 kdp_core 之后）
- 页表 walk：PML4[0x1fd]→PDPT 0xfaaf000, PDPTE[0x1a8]=0 ——
  guest 页表里该 1GB 区未映射, 且 handler 重试后仍未建立
  → 怀疑注入的 #PF 未真正 deliver / handler 未运行 / 修页后
    重试仍 fault（诊断中：whpx-pf-seq + whpx-exitseq）


## 追加：KVM 对照（关键排除证据）

- WSL 旧构建（11.0.5, whpx-fixes build 目录）boot Boot0002 循环
  = 旧 QEMU 版本问题；qemu-build.sh 重建后（11.0.50）OpenCore 正常
- KVM + rails 镜像：XNU 启动后 CPU0 在 userspace(0x7ff8...)
  与 kernel 间正常切换、多个 CR3 —— 健康 boot 到 launchd 阶段
- KVM 下无 rep movsb #PF 循环 → 该循环是 WHPX 特有
- WSL reims-vgpu 窗口（dzn）D3D12 Removing Device → segfault：
  与 AndrowsVm（安卓模拟器）GPU-PV 冲突嫌疑，与 QEMU/KVM 无关
- 诊断污染嫌疑：gdb si（单步）走 whpx_set_exception_exit_bitmap
  (1<<DB) 覆盖式设置 partition 异常位图，可能清掉 PF 拦截位 →
  "PF 只 exit 一次后不再 exit" 可能是我单步诊断造成的假象
## 遗留

- AP 未启动（G0/G2）仍是主线问题, 与网络栈慢无关
- 检测器留作诊断工具（可重复触发, QMP cont 恢复）

