# 第二十七轮：桌面达成与修复交付（2026-09-03 深夜）

## 一句话
**带 GPU 的 macOS 13.6 guest 在 Windows/WHPX 上进入 Ventura 桌面**；boot 阶段全部卡点定位并修复，两个通用 WHPX 缺陷交付上游 PR。

## 过程（M1–M9 到桌面）

1. **M1–M6（stop/cont 之谜）**：QMP stop→cont / GDB attach 是唯一能让 guest 推过 memcpy 卡点的手段（4/4 复现）。逐一复刻（cancel+窗口、SuspendPartitionTime、全 VP FULL_STATE 读回、bitmap 重置）全部失败。候选成分未能单离，但推过后的随机 panic（APFS list head / zone 元数据 / logd 锁断言）提供了下一步线索。
2. **外部协作（Claude fix1）**：外部 AI 找到 xsave 陈旧写回 bug——WHvGetVirtualProcessorState(Xsave) 在 AMD/Hyper-V 上每 VP 硬失败，QEMU env 的 FP/AVX 影像陈旧，vm_stop/vm_start（即 attach/stop-cont）把它写回 Hyper-V 覆盖活寄存器 → "随机内存损坏"实为该写回的必然结果。修复：读失败标记 per-VP invalid，禁写回（还修了失败路径的 buffer 泄漏）。
3. **BQL MMIO 修复（我）**：WHPX MemoryAccess exit 路径 BQL-free 分发设备 MMIO（KVM/TCG/HVF 均持 BQL），违反设备模型。whpx_handle_mmio 加 BQL。
4. **带 GPU 的剩余损坏实锤**：fix1 + BQL 后带 GPU 仍随机 panic（APFS @3959，SecurityAgent 深度）。审计（子代理）枚举设备全部 guest-RAM 写路径后，实测 REIMS_VGPU_GUEST_IMPORT=off 消除损坏 → 实锤 GPU 裸导入（VK_EXT_external_memory_host 直写 guest RAM）在 guest 释放页后继续写（released_pages.rs 自述的缺陷类，与 Linux issue #70 同源）。
5. **桌面达成**：GUEST_IMPORT=off 后 1 分钟进桌面。条纹/闪烁 → SWAPCHAIN_FIFO=on + LAZY_WRITEBACK=off 消除鼠标闪；移动窗口仍花。
6. **dirty bitmap 实验（未完成）**：WHPX 的 whpx_log_sync 全量标脏导致 witness 失真；WHvQueryGpaRangeDirtyBitmap 在 VP 运行中返回 0x80370305（WHV_E_INVALID_VP_STATE），cancel+停驻窗口+suspend partition 后仍未成功（待查）。当前回退全量标脏。

## 关键结论
- 问题 A（memcpy 卡点）最终靠 AMD 直写（AMD-CORES）+ PF-NATIVE + xsave 修复的组合稳定越过；stop/cont 的"推过"效果最终归因于其 xsave 写回副作用被修复前的巧合。
- 问题 B（随机损坏）两个根因：① WHPX xsave 陈旧写回（QEMU 通用缺陷）；② 设备 GUEST_IMPORT 裸写已释放页（设备缺陷，Linux 同款）。
- WHPX 在这台宿主上的 xsave 读失败是硬错误（每 VP 每次），QEMU 上游同样受影响。

## 交付
- **steelbrain/qemu-reims-vgpu#4**：8 个修复（INIT IPI / PF 注入 / SYSENTER / CPUID leaf4 / TSC-deadline / MISC_ENABLE / XSAVE 保护 / BQL MMIO），基于 Hi-Jiajun/qemu-reims-vgpu:whpx-win-stability。
- GitHub #71、GitLab 4385、qemu-devel leaf4 线程均已更新进展。
- 证据：evidence/（serial-login-success-novgpu.log、serial-bqlfix-gpu.log、doctor-m9-bitmap0-race.log、claude-fix1.diff、whpx-all.c.before-fix-2026-09-03 等）。

## 遗留（改日）
1. 移动窗口块状花：dirty query 的 Hyper-V 前置条件（suspend 语义）待查；或走方向 A。
2. 方向 A：Windows 别名视图（QEMU anon RAM 改 section-backed + MapViewOfFileEx 打包别名）→ 恢复 GUEST_IMPORT=on 零拷贝。
3. qemu PR #4 review 跟进。
