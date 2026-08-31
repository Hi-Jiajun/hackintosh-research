# WHPX SYSENTER 突破（2026-08-31 第四轮）

## 缺陷

WHPX 拦截 guest 对 IA32_SYSENTER_CS/ESP/EIP（0x174-0x176）的写
（MsrAccess exit），QEMU whpx 后端完全没有处理：unknown msr → 静默忽略。

guest 写 SYSENTER_EIP = 0xfffff6b9400027a0（bootstrap identity 映射地址，
低位正是之前最初 trap 帧里的 0x400027a0）、
SYSENTER_ESP = 0xfffff6b940084200。写入被丢弃 → partition 的 SYSENTER
状态陈旧 → guest 死于嵌套 fault 风暴（exit 4）。

## 本地修复

1. MsrAccess exit：SYSENTER 三 MSR 改为 known；读返回 env 缓存，写更新 env。
2. x86_emul 兜底路径（whpx_simulate_rdmsr/wrmsr）同样处理这三个 MSR，
   不再 raise #GP。

## 发现的 Hyper-V 限制

WHvSetVirtualProcessorRegisters 拒绝 EIP=bootstrap 地址的 SYSENTER 状态
（hr=c0350005 InvalidVpRegisterValue）；写 0 被接受。当前本地构建把
partition 侧 SYSENTER 钉为 0，env 保留 guest 真实值。

## 结果

exit 4 崩溃消失：
- guest 通过 pmap 初始化（CR4 出现 SMEP/SMAP/OSXSAVE），
- 16 vCPU 全部到达 XNU task 层初始化（_task_restartable_ranges_synchronize），
- VM 在 200 秒观察窗内保持 running。

## 剩余问题

- guest 停在 task_restartable_ranges_synchronize（疑似等 IPI/AST）。
- 仍有 17 条 vector 0 中断丢弃。
- 偶发 bcopy_phys panic 现场的 #GP。

## 上游状态（同日）

- steelbrain/reims-vgpu：PR #57（Windows 移植）已合并。
- steelbrain/qemu-reims-vgpu：PR #2（qemu shims）已合并，
  reims-vgpu 子模块已指向。
- 本 fork master 已 fast-forward 同步到上游 1ac9820。