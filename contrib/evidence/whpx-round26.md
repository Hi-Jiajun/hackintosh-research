# WHPX round26：宿主侧杠杆穷尽 + PE_init panic 根因 + stall 形态图谱（2026-09-03 深夜续）

## 结论

1. **宿主侧影子刷新杠杆已全部穷尽**。Hyper-V/WHPX 的影子页表持有首次 walk 的结构缓存，此后：
   - 宿主直写真实叶子（缺失补位 / D/A 补位 / US 清除 / A 位心跳）→ 无效；
   - guest 执行 invlpg → 无效；
   - guest 执行 mov cr3, cr3（CR3STUB 实证 stub 取指成功、执行成功）→ 无效；
   - **CR3 错根切换**（cancel 后写 CR3=orig^0x1000 再写回，WHPX 必然感知根变化）→ 无效；
   - guest 亲手写真实页表页（走 EPT 写保护/影子同步路径）→ 无效；
   - 脏页查询崩溃、LAPIC 状态 API 不可用、VP 生命周期红线 → 均不可用。
   结论：**当前 stall 形态与 1GB 型同族——结构缓存永久陈旧，宿主侧无解**。

2. **PE_init_iokit panic 根因 = 我们自己的合成 PTE**。此前 rescue 兜底写入
   0x1000027（phys=0x1000000 瞎猜）让 XNU 的 physmap 写落到 1MB 处的错误物理内存，
   数据结构被破坏 → XNU 在 PE_init/IOKit 阶段 panic（栈上发现 panic 参数
   'PE_init_iokit'，RSP 指向 panic 结构）。合成兜底已移除——**宁可卡着，不可损坏**。

3. **OpenCore 'Prelinked patcher Not Found' 根因实锤**：不是特征码漂移。13/14 号
   补丁的特征码在 guest 真实内核字节里**原样存在**（vmf-real.bin 偏移 0x16b5/0x16c5 命中），
   但 OpenCore 找不到——xnu 的 vm_fault_internal 位于 __TEXT_EXEC 段，
   Prelinked patcher 不搜该段。这同时解释了历史悬案 Kernel→Patch 完全不生效。
   （TRIM 补丁连 result 行都没有，OC 文件日志在本环境只写出 5 行即止。）

## 关键实验记录（本轮）

| 实验 | 结果 |
|---|---|
| KPATCH13/14 直写（QEMU 直写 vm_fault_internal 的 je→jmp，写前特征码校验） | ✅ 首次真正写入 guest 内核（vmf=ffffff801550a650）；但语义（强制 re-enter）不破环——XNU 的写仍落旧副本 |
| CR3STUB（CR2 页内写 stub：invlpg + mov cr3,cr3 + jmp 回原 RIP） | ✅ stub 执行成功（hr=0），❌ 不刷新影子 |
| CR3STUB-v2（追加 guest 上下文写真实叶子 PTE，翻转 A 位） | ✅ 写发生（叶子值可见变化），❌ 仍 fault 循环 |
| CR3WIGGLE（cancel 后 CR3 写错根再写回） | ✅ h1=h2=0，❌ 无效 |
| memcpy physmap-noop patch（入口 jmp 到 padding 检查区：bt rdi,38 → physmap 直接 ret） | ✅ 写入验证通过（va=ffffff8009301050）；单用不破环（死循环 rip 在 memcpy 内部不经过入口） |
| SKIPRET / SKIPDEEP / SKIPINS（弹栈 / 深扫弹多层 / 跳 fault 指令） | ⚠️ 能推进 guest（RSP 逐层后退），但**弹栈撕坏调用链 → 直接导致 PE_init panic**；已禁用 |
| 宽扫描救援（全 RAM 逐页找旧副本同 slot 健康 PTE） | ❌ 本轮 0 命中（本 boot 落在 PDPTE 缺失型，本无叶子可救）；代码保留 |
| GDB RSP 纯 python 客户端（m/p/P 协议读内存/寄存器/写 RIP/RSP） | ✅ 新外部操控通道，随时介入 guest 实验 |

## stall 形态图谱（更新版）

1. **4KB-PTE 缺失型**：真实叶子缺失，旧副本可找 → 正确 PTE 救援可破；
2. **D=0 型**：叶子 present 但 D/A 缺失 → 医生补位可破（已修）；
3. **健康叶子型**：叶子 P|RW|US|A|D 全齐仍 fault → 影子结构缓存陈旧，**无解**；
4. **PDPTE 缺失型（1GB 型）**：level=1 entry=0，整 1GB 区未映射 → **无解**；
5. 每次 boot 随机落入其一——可救形态出现时医生能推进，无解形态只能重启碰运气。

## 教训

- **合成 PTE 的 phys 必须正确**：瞎猜 phys 会静默损坏 guest 内存，在下游（PE_init）
  以 panic 形式爆发，与真故障形态难区分。
- **弹栈/跳指令是破坏性救援**：能让 guest 前进，也会把状态推入 panic；只能作为
  最后手段且用完即弃。
- tail -10 会漏掉一次性事件（KPATCH-MEMCPY 其实早成功了，多轮误判）。

## 下一步

- 多形态自动处理 + 无解形态重启碰运气；
- 等上游 issue #71 / 微软修 WHPX shadow 同步；
- 树清理：把禁用的实验块开关化、删除无效 OpenCore 补丁 12/13/14。
