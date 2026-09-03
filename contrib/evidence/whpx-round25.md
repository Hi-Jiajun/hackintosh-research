# WHPX round25：boot 慢根因定位 + stall 破环实验（2026-09-03 深夜）

## 结论（根因已实锤）

**boot 15-20 分钟的根因**：Hyper-V 影子页表把 guest 内核访问页表页的窗口（physmap/递归映射）
**错误地映射到了一个旧的物理副本**。链条：

1. guest 的 pmap 通过该窗口写入 PTE → 写入落在旧副本（guest 自认为成功）；
2. guest 自己的读也走同一窗口 → 看到 present → vm_fault 走"已满足"捷径 → iret 重试；
3. 真实页表（QEMU 侧逐级 walk 确认）里对应叶子项永远是 0；
4. 硬件 walk 读真实页表 → 永远缺页 → 故障指令死循环。

**实证**：
- 真实 PT 页的叶子 PTE 连续 20 次采样全 0（guest 从未写进真实页表）；
- 向真实页表写入**任意** present PTE → cpu0 立刻离开 memcpy（两次复现，rip 直接跳走）。

## 关键实验记录（本轮）

| 实验 | 结果 |
|---|---|
| 原生 PF 投递（运行时摘 PF 拦截：cancel+park 翻转，首次内核态 PF 触发） | ✅ 成功；pv_list 双重释放竞态 panic 消失 |
| OpenCore re-enter 补丁（vm_fault 已满足分支改 fall-through） | ❌ OC 补丁从未应用（配置有效、内存未变，原因未查明）；且 QEMU 直写同样字节后 stall 不破 → re-enter 路线本身无效 |
| QEMU 直写内核（VMFAULT-DUMP 基础设施 + cpu_memory_rw_debug） | ✅ 读写通路都可用 |
| 合成 PTE（0x1000027）写入真实叶子 | ✅ **打破循环**（4KB 形态）；但 phys 错误 → 数据损坏 → 下一页继续 stall |
| 邻居扫描（±32→±4096 页）找 stale 副本 | ❌ 无候选（PT 池不在邻居范围或形态不同） |
| WHvQueryGpaRangeDirtyBitmap 脏页扫描 | ⚠️ 查询调用本身让 QEMU 段错误（A/B 已锁定）；map flag 已接受（hr=0）但查询崩溃 → 该 Windows 版本实现问题，路线暂弃（map flag 已回退） |
| host 侧 CR3 写（PCD 翻转）刷新 Hyper-V 结构缓存 | ❌ 无效（1GB 大页形态）——需 guest 侧真 mov-cr3（VMCS 级刷新） |
| stall 医生（每 15s 重新 walk + 逐层补缺失项，循环化） | ✅ 机制运转正常；4KB 形态可破，1GB/PDE 形态仍缺"真实故障地址"锚定 |

## stall 的多种形态

1. **4KB-PTE 缺失型**（故障页=rdi）：真实叶子 PTE=0，上层完好 → 合成 PTE 可破；
2. **PDE 缺失型**：PDE=0 → 需逐层补（医生循环化后可级联修复）；
3. **1GB-PDPTE 型**：真实 PDPTE=present（STALL-MAP 注入过）但 Hyper-V 缓存仍 absent → host 侧写无效，只等 guest 自己切 CR3（历史 15-20 分钟的破点 = 某个时刻该 CPU 切到用户任务）。

## 附带发现

- guest 内核与 asset 反汇编有实质漂移（vm_fault_internal 在 0x42e650、vm_fault_enter 在 0x4312b0 附近与 asset 相同，但函数内部 -0x400 偏移）；
- OpenCore 内核补丁（Kernel→Patch）在本环境完全不生效（配置有效、内存未变），原因未查明；
- 固件段 QEMU 段错误（139）间歇性 ~40%（7/17 次），与诊断代码无关的 WHPX 竞态，重试可过；
- 脏页位图粒度 = 每 4KB 一位、查询即清除（官方文档确认），范围必须落在已映射 RAM 内（低区 [0,2GB)、高区 [4GB,18GB)）。

## 树的状态（Windows 生产树）

- QEMU：原生投递 + watchdog 内核态门槛 + VMFAULT-DUMP + 循环化 stall 医生（walk/邻居扫描/合成兜底）+ rippoll 已禁；
- OpenCore.qcow2：5 个缺失 kext 已禁用（VoodooPS2/USBToolBox/UTBMap/AppleMCEReporterDisabler）、Target=7、Timeout=60、3 条补丁（2 条已证无效但无害）；备份 OpenCore.qcow2.bak-pfpatch；
- 完整备份：/mnt/c/hackintosh/reims-vgpu-backup-v6/；
- 上游 issue：steelbrain/reims-vgpu#71（GPU 断言 panic + 本 stall 根因评论）。

## 剩余路线

1. 4KB 形态的正确 phys：脏页查询崩溃需绕过（小范围试 / Is...Present 探测 / 其他 stale 页发现通道）；
2. 1GB 形态：guest 侧真 CR3 切换（中断注入促调度，或等 guest 自然切换）；
3. 故障地址精确锚定：rdi 只在部分形态正确，需按"首个 PF 的真实 CR2"动态选择。
