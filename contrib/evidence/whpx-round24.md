# 第二十四轮：根因实锤——Hyper-V shadow 页表 stale（2026-09-03）

## 决定性证据

- PF exit 时刻（vCPU 线程、BQL、exit 后）walk guest 页表：
  fault 地址的 PDPTE **已存在**（如 0x1f56c6027，2MB 大页 P|RW|US|PS）。
- 但 Hyper-V 仍报 #PF（exit 到 QEMU）——Hyper-V 的 shadow walk
  与 guest 页表（物理内存）不一致。
- guest 的 vm_fault 每轮"成功"（页表已有映射，无需修）→ iret →
  重试 → Hyper-V 仍 fault → 无限循环。这解释了全部观测：
  vm_fault 成功、PDPTE 非 0、rcx 恒 0x4000、guest 不 panic。

## 已尝试的修复（全部无效）

- 同值 CR3 写回（exit 时 VP 已停）：无效（Hyper-V 优化掉）。
- 换值 CR3（toggle PCD 位再写回）：无效（shadow 仍 stale）。
- 之前的 CR3 flush timer、watchdog 映射注入：均无效。

## 根因定性

- Hyper-V（WHPX）对"已 deliver 给 guest 的 page fault"不刷新其
  shadow 页表快照；guest 修页（写物理页表）后 Hyper-V 不感知。
- WHPX API 无显式 shadow/TLB flush 接口；CR3 写同值被优化、
  换值也不触发重建。

## 下一步候选

a) 查证 Hyper-V shadow 更新机制（EPT 的 dirty tracking 是否应
   感知 guest 页表写——或许 WHPX 有 enable 开关/已知 bug）；
b) QEMU 侧在 PF exit 时改 guest RIP 跳过 fault 指令（诊断性，
   看 boot 能否快速推进）；
c) 向 Microsoft/WHPX 社区报告（shadow 不刷新是可复现 bug）。

