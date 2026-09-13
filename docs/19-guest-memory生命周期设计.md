# Guest memory 生命周期设计（2026-09-14）

> 承接 `docs/14`（range hazard 与共享分配）与 `docs/15`（共享 per-allocation device buffer）：
> 那两篇把"同一 allocation 的多 view + range 预约 + 一次 import"打通，本文定义**guest memory
> 本身**的 owner 侧接口与生命周期。目标场景是 reims/vGPU 线（`docs/09` 的 Gate 2/3）。

## 1. 现状（本仓库已具备的地基）

| 能力 | 位置 | 状态 |
|---|---|---|
| range 级 hazard（重叠判定、写优先） | `BufferRange`/`RangeSet`（`a5a710a`） | 完成 |
| 同一 allocation 多 view 的 admission 与 reservation | `fb5f8fd`/`00d6008` | 完成 |
| 每 allocation 一个 device buffer（一次 import 的挂靠点） | `3554435`/`7073e7a` | 完成 |
| host-pointer 无拷贝导入（Vulkan `VK_EXT_external_memory_host`、native `newBufferWithBytesNoCopy`） | `42974ac`/`7660f20` | 完成 |
| 跨进程映射传递（`SCM_RIGHTS`/命名 section） | `79c9706`/`8439f7e` | 完成 |
| lease 退休契约（`LeaseLedger`，终态才释放） | `d26359c` | 完成 |
| **owner 侧宿主区间注册与窗口派生** | `HostRegion`（本日新增） | 完成（步骤 1） |

缺的是**生命周期**：谁在什么时刻把 guest 页交给设备、设备写完后谁负责标记脏页/失效、
guest 侧释放页时在途访问如何收尾。

## 2. 关键决策（建议）

1. **注册而非逐次传递**：guest RAM 由 owner 在 VM 启动时注册一次（base + length + page size），
   之后每次提交只派生窗口。这与 `docs/15` 选定的"每 allocation 一次 import"一致，也避免
   每次提交跨越 FFI 边界传指针。
2. **窗口一律页对齐**：`HostRegion::borrowed_window` 已强制 offset/length 按 page_size 对齐；
   GPU 侧访问的字节精度由 view 的 offset/length 表达（`docs/14` 的 range hazard 负责冲突）。
3. **脏页归 owner**：provider 不做脏页跟踪。设备回写后，owner 依据 writeback 的
   `(allocation, offset, bytes)` 标记对应 guest 页脏（WHPX 场景下再同步给 guest）；
   provider 只承诺"写回在 lease 退休前完成且字节精确"。
4. **失效序列**：`UnmapMemory`/guest 页释放时，owner 必须先确认该窗口的所有 lease 已退休
   （`LeaseLedger` 的终态），再回收宿主映射；未退休的窗口不允许回收——这是 `docs/13`
   有界放弃语义的下游。
5. **不假设零拷贝**：`BorrowedNoCopy` 不可用时（无 host-pointer 导入能力）退化为
   `StagedLease` 拷贝路径，语义不变、只是性能退化；`docs/14` 的 range 层同样适用。

## 3. 实施步骤（每步独立可验证）

1. **owner 侧宿主区间（本文档同日完成）**：`HostRegion` + `borrowed_window` + 校验与单测；
   零行为变化。
2. **窗口 → 提交的接线演示**：在 `provider-smoke` 增加一个用例，用注册的宿主缓冲（测试内
   分配的一段内存当作"guest RAM"）派生窗口、以 `BorrowedNoCopy` 提交、断言设备写入原地
   可见、并走 `LeaseLedger` 退休（Lavapipe + RTX 5060）。
3. **脏页记账接口**：core 增加 `DirtySet`（页对齐的 range 集）与从 writeback 派生的纯函数
   + 单测；owner 侧消费。
4. **失效/回收协议**：定义 `GuestWindowLease` 的 owner 侧状态机（active → retired →
   reclaimable），与 `LeaseLedger` 对接；单测覆盖"未退休不得回收"。
5. **reims 侧接线（跨仓库）**：用 reims 的 guest RAM 区注册 `HostRegion`，把 wire 命令的
   allocation 映射到窗口；这一步需要 reims 侧的配合，另开文档。

## 3.1 实施状态

- **步骤 1 完成（`2f35ce5`）**：`HostRegion` + `borrowed_window` + 校验与两个单测；零行为变化。
- **步骤 2 完成（`306e4e2`）**：`provider-smoke` 的 `provider_host_region_window` —— 注册 8 KiB
  页对齐宿主区间、owner 类型拒绝越界窗口、派生窗口、无拷贝导入、设备原地写入、lease 释放；
  **Lavapipe 与 RTX 5060 双驱动 PASS**（真机日志 26 PASS，归档
  `evidence/guest-region-smoke-306e4e2-2026-09-14/`）。

## 4. 边界（本文不做）

- 不做 guest 页内容的迁移/交换（那是 VM 侧的职责）；
- 不改变现有 trace/wire 格式（窗口信息经 `BorrowedNoCopy` 的既有 lease 通道表达）；
- 不承诺 GPU 与 CPU 的缓存一致性：一致性由 owner 的同步点（WHPX 侧 flush/invalidate）
  负责，provider 只保证提交边界上的字节语义。

## 5. 与 Gate 2/3 的关系

- Gate 2 需要"生产 guest/display 路径调用 canonical provider"：步骤 2/4 提供 owner 契约，
  步骤 5 是真正的接线；
- Gate 3（VM E2E）还需要 WHPX guest RAM 的脏页导出与显示路径，属于本设计的下游工作。
