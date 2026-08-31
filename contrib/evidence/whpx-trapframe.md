## 闭环：最初 trap = #GP @ identity 地址 0x400027a0（2026-08-31 第三轮）

第一条 PF 停止点的原始 trap 帧（所有 panic 帧之下）：

- **trapno = 0xd（#GP）**，**trap 时 RIP = 0x400027a0**（identity 映射低地址）。
- 0x4000000-0x4040000 物理全零（KVM 健康 boot 下同样全零）——
  不是内容缺失，而是 **guest 跳到了不该执行的地址**。
- 帧下方紧邻 GDT/TSS 加载现场：TSS 描述符 0x8000000000000083、
  GDT 基址 0xfffff69140000000。
- 内核物理基址实锤：kernBase = slide + 0x200000（mach-o magic 定位）。
- 该 trap 之上是已报告的 SMEP/User NX fault panic 链（sync_iss_to_iks）
  → 格式串指针=CR3 的 #PF → vm_fault(map=NULL) 递归 → exit 4。

## 工作假设

XNU bootstrap 最早期的段/TSS 设置（最初 trap 前后）在 WHPX 下计算出
0x400027a0 的跳转/返回目标，KVM 下不会——即 WHPX 在早期段寄存器/TR/TSS
状态同步上与 KVM 存在差异，把 guest 送进了 identity 区。

## 下一步

对比 WHPX 与 KVM 在 bootstrap 同一点上的 TR/TSS/段寄存器同步行为；
重点排查 QEMU whpx 后端对 WHvX64RegisterTr / 段寄存器 / ltr 的处理。