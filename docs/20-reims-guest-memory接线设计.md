# reims 侧 guest memory 接线设计（步骤 5）（2026-09-13）

> 承接 `docs/19`（guest memory 生命周期）：那篇的步骤 1–4 已在本仓库落地（owner 侧
> `HostRegion`/`DirtySet`/`GuestWindows` + 双驱动 smoke 证据），**步骤 5 未完成**——把 reims
> 的 guest RAM 区注册为 `HostRegion`、把 wire 命令的 allocation 映射到窗口。
>
> 本文是步骤 5 的设计输入，目标是：一个熟悉 reims 的工程师**不看聊天记录**就能开工。
> 所有结论来自实地读过的代码，行号可用 `rg -n` 复现；未核实的一律标"待确认"。

术语约定（与既有文档保持一致）：provider、lease、`HostRegion`、canonical path 保持原文。

**命名冲突提醒**：reims 仓库里已存在 `crates/reims-vgpu/src/host_window/`，那是**显示窗口**
（present/input/capture，见 `crates/reims-vgpu/src/host_window/mod.rs`），与本文的"宿主内存
窗口"无关。本文说"窗口"一律指 provider `HostRegion::borrowed_window` 派生出的
`BorrowedLease` 区间。

---

## 1. 现状与目标

**为什么做**：`docs/09` 的 Gate 2 要求"生产 guest/display 路径调用 canonical provider"。
当前 reims 的 guest RAM 导入是一条**自己的**完整轨道（`GuestRamImport` → `GuestSlice` →
Vulkan `VkBuffer` / Metal `MTLBuffer`），provider 侧的 owner 契约（`HostRegion`、
`DirtySet`、`GuestWindows`）**没有任何 reims 调用方**（`rg -n "metal_api" repos/fork-reims-vgpu`
命中为空，两个仓库间目前无依赖）。

**缺口在哪一层**：不在"有没有办法把 guest RAM 交给 GPU"——那是 reims 已经有、且已在真机
驱动起来的；缺的是**两个坐标系的对接**：

| 关注点 | reims 侧已有 | provider 侧已有 | 缺口 |
|---|---|---|---|
| 宿主区间 | `GuestRamRegion{gpa_base, host_va, len}` | `HostRegion{lease_id, owner_epoch, host_pointer, length, page_size}` | 字段可一一投影，但没有投影代码 |
| 身份 | `ImportId`（进程单调、永不复用） | `LeaseId` / `AllocationId` / `DeviceEpoch` | 需要一个稳定的派生约定 |
| 区间派生 | `GuestRamImport::slice(offset, len)`（按 granularity 向外扩） | `HostRegion::borrowed_window(allocation_id, offset, length)`（强制页对齐、越界拒绝） | 语义重叠、拒绝规则需对齐 |
| 提交证据 | completion stamp / fence | `LeaseLedger::bind/retire`、`BorrowedLeaseRegistry::retain/retire/release` | 需要把 stamp 观察接到 lease 退休 |
| 失效 | `BoundBuffers::retire_range/retire_ref`、`guest_ram_map::reset` | `GuestWindows::{retire, reclaim}` | 需要确定"谁先谁后" |
| 脏页 | `host_writes` / `render_writeback` / `mapping_write` 记账 + QEMU 侧 `guest_write_gen` | `DirtySet::mark_writebacks` | 需明确 owner 由谁消费 |

**目标**：让 reims 成为 provider 的 **owner**，而不是在 provider 里再建一条导入轨道。也就是
说，Vulkan 侧仍然由 reims 完成 `VK_EXT_external_memory_host` 导入（那是它已经验证过的能力），
provider 只消费"这段宿主区间是谁、有多长、下一次提交要借哪一段、什么时候可以回收"。

---

## 2. reims 侧现状核对（实地读过的文件与行号）

以下每一条都亲自读过；行号对应 `repos/fork-reims-vgpu` 的当前 checkout
（HEAD `747580f`，worktree 有 `vendor/qemu`、`vm/boot-windows.sh` 的既有 dirty 改动，
本文不改动任何文件）。

### 2.1 guest RAM 的注册点（关键发现：注册已经存在，只是没有交给 provider）

| 环节 | 位置 | 说明 |
|---|---|---|
| 宿主回调接口 | `crates/reims-vgpu/src/runtime/host.rs:494` | `HostOps::guest_ram_regions()`，默认返回 `CallbackMissing`；文档明说"the whole guest-memory import rail starts here"，**只在设备初始化调用一次** |
| QEMU 侧实现 | `crates/reims-vgpu/src/qemu/host_ops.rs:643` | 传 `FIRST_TRY=8` 个元素，不足再按 shim 报的总数重试；截断会报 `StillTruncated` |
| shim | `vendor/qemu/hw/display/reims-vgpu-shim.c:170` | `reims_vgpu_shim_guest_ram_regions()`：走 `flatview_for_each_range` + `address_space_memory` |
| 注册进结构体 | `vendor/qemu/hw/display/reims-vgpu-pci.c:985`、`vendor/qemu/hw/display/reims-vgpu-mmio.c:983` | 两条 pathway 都把该函数挂进 `.guest_ram_regions` |
| 区域形状 | `crates/reims-vgpu-memory/src/lib.rs:213` | `GuestRamRegion{gpa_base, host_va, len}`，`#[repr(C)]`，与 `include/reims_vgpu_qemu_abi.h` 同构（一致性测试在 `crates/reims-vgpu/src/qemu/abi.rs:997`） |

**结论**：owner 侧要的 `HostRegion.host_pointer/length` 就是这里的 `host_va/len`；区间是
"QEMU 的 RAMBlock 映射"，在本进程内稳定到 VM 生命周期结束——这正是 provider
`HostRegion` 类型文档要求的语义（`metal-api-emulator/crates/metal-api-core/src/provider.rs:218`
附近的类型注释："a guest RAM region whose host mapping is stable for the registration's lifetime"）。

### 2.2 粒度（page_size）从哪来

- backend 实测并把三个数字一次性发布：`crates/reims-vgpu/src/backend/vulkan/engine/context.rs:995`
  调 `crate::runtime::guest_ram::latch_import_limits(...)`；
- 定义在 `crates/reims-vgpu-memory/src/lib.rs:814`（`latch_import_limits(align, budget, span_max)`），
  读取器是 `import_span_max()`（:858）、`import_budget()`（:874）、`granularity()`（:886）；
- Vulkan 侧的真实来源是驱动查询：`crates/reims-vgpu/src/backend/vulkan/caps/host_pointer.rs:303`
  `let min_alignment = host_props.min_imported_host_pointer_alignment;`（零或非 2 的幂在
  :304 被拒），字段与"**never assumed to be 4096**"的语义在 :161 `HostPointerCaps`/:170；
  它出现在 `vk_caps ... host_pointer_import=... host_pointer_align=...` 的选择行
  （`crates/reims-vgpu/src/backend/vulkan/caps/mod.rs:132`；注意 :163 的 `min_alignment: 4096`
  是 **fixture 值**，同文件 :197 起的单测在 :200 断言 `host_pointer_align=4096`，不能当真机结论）；
- **导入还会被 2 GiB 上限切块**：`crates/reims-vgpu/src/backend/vulkan/caps/host_pointer.rs:399`
  `IMPORT_SPAN_CEILING = 2 * 1024 * 1024 * 1024`（记录到某驱动把 `allocationSize` 截成 32 位），
  切块逻辑在 `crates/reims-vgpu/src/runtime/guest_ram_map.rs:732` `chunk_span()`。

**对设计的直接影响**：一个 RAMBlock ≠ 一个 `HostRegion`。在 4 GiB guest 上会有 2–3 个
HostRegion（还要乘 RAMBlock 数）。`docs/19` 说的"注册一次"仍然成立——注册单位是
"(chunk, device)"，不是 "(RAMBlock, 每次提交)"。这一点必须写进实现注释，否则后来者会以为
注册是一对一的。

### 2.3 GPA → 可绑定引用

- 惰性构建 + 一次性解析：`crates/reims-vgpu/src/runtime/guest_ram_map.rs` 的
  `static MAP: Mutex<Option<Resolved>>`（:236）与 `Resolved`（:239）、
  `resolve()`（:750）、`reset()`（:313）；
- 首帧预热的调用点是**协议握手**：`crates/reims-vgpu/src/runtime/mmio.rs:185`
  `crate::runtime::guest_ram_map::warm(host);`（同文件 :183 的注释解释了为什么不能留给第一次 draw；
  `warm` 定义在 `guest_ram_map.rs:382`）；
- 查询入口：`references_for_runs()`（:549）、`reference_for_pages()`（:637）、
  内部单点 `Resolved::reference()`（:261，二分查找 + `contains_gpa` 判定）、
  `import_end()`（:298，供跨 import 边界切分）；
- 窗口形状：`GuestSlice`（`crates/reims-vgpu-memory/src/lib.rs:735`），由
  `GuestRamImport::slice()`（:622）或 `slice_for_gpa()`（:659）产生，**会自动向外扩到
  granularity**，并用 `head()`/`requested()`/`bound_len()` 记下"真实起点/请求长度/绑定长度"
  （单测 `a_slice_is_widened_to_the_granularity_and_says_by_how_much`，:1325）；
- 独立宿主分配（packed alias）：`GuestRamImport::new_host_allocation()`（:525），
  这类 import 的 `gpa_base()` 是 `None`（单测 `a_packed_host_allocation_slices_only_by_relative_offset` 同文件）。

### 2.4 命令/批处理提交路径

- 线格式与分帧：`crates/reims-vgpu-wire/src/op.rs`（8 字节记录头，含 opcode/length 推导与
  "这不是 FIFO 包头"的说明）、FIFO 层在 `crates/reims-vgpu/src/runtime/decode/`（`mod.rs` 起）；
- 解码器：`crates/reims-vgpu/src/runtime/decode/{stream,render,compute,blit,event,resource}.rs`；
- 根/子 FIFO 排空与 stamp 回写：`crates/reims-vgpu/src/runtime/drain/mod.rs`（模块文档："Root/child
  FIFO drains, stamp writeback, and fail-visible command dispatch"）；
- 提交与完成：completion stamp 的发布/中断在
  `crates/reims-vgpu/src/backend/vulkan/engine/stamp_completion.rs`（模块文档解释 guest 的
  1 秒 watchdog 与"中断是唤醒而不是提示"）；
- **allocation 在 wire 侧的身份**：不是 provider 的 `AllocationId`，而是 task 局部对象引用
  + offset。这一点的权威描述在 `crates/reims-vgpu/src/runtime/bound_buffers.rs` 的模块文档
  （"Apple's host resolves a guest object reference to a host buffer once, when the object is
  created"），失效通知则是 `CmdMapMemory2` / `CmdUnmapMemory` / `CmdReplacePhysical` /
  `CmdSetObjectList` / `CmdDeleteObject` / `CmdDefineTask2` / `CmdDeleteTask`（同模块文档列出）。

### 2.5 Metal 相关对象（MTLBuffer 等）的映射位置

- **Vulkan 臂（本机可验证的那条）**：guest 支撑的线性目标在
  `crates/reims-vgpu/src/backend/vulkan/engine/linear_target_import.rs`
  （`:368` 检查 `ctx.external_memory_host.is_none()`，`:590` 取扩展，`:639` 调
  `get_memory_host_pointer_properties_ext`）；
- 绑定点：`crates/reims-vgpu/src/runtime/bound_buffers.rs:583`
  `.slice(packed.head + offset, span)`；等价 walk 版在 `crates/reims-vgpu/src/runtime/draw/vulkan.rs:3256`
  `import.slice(head, backing.size)`；
- **Metal 臂**：`crates/reims-vgpu/src/backend/metal/runtime.rs:23` 明确写着
  **"No importer is wired here yet."**——该模块只有 `new_buffer_from_host`，
  且今天的调用方只传本 crate 自己拥有的 CPU staging `Vec`
  （`crates/reims-vgpu/src/backend/metal/raw_metal.rs:452` 是 `newBufferWithBytesNoCopy` 的声明）；
  同文件 :86 附近记录了 Metal-direct 臂的对齐要求（base 必须是 guest page 对齐）。

### 2.6 现有测试/日志能证明什么

| 证据 | 位置 | 能证明 | 不能证明 |
|---|---|---|---|
| `guest_ram_span n=…/… gpa=… len=… mib=…` | `crates/reims-vgpu/src/runtime/guest_ram_map.rs:821` | 每个 span 的 base/len、导入顺序、切块结果 | 与 provider 侧 `HostRegion` 的任何对应关系 |
| `guest_import_levels ramblocks=n/m aliases=…` | `crates/reims-vgpu/src/backend/vulkan/census.rs:88`（调用 :78） | 惰性导入的真实水平（分母是 shim 报的 span 数，见 `span_census()` :331） | 谁在借哪一段窗口、何时归还 |
| `vk_caps ... host_pointer_import=supported host_pointer_align=<granularity>` | `crates/reims-vgpu/src/backend/vulkan/caps/mod.rs:132` | 该驱动的导入粒度与导入预算 | 运行时窗口拒绝率 |
| `GuestRamError` 的逐项 slug | `crates/reims-vgpu-memory/src/lib.rs:269`（枚举，`slug()` 逐项给出）与单测 `every_refusal_has_its_own_slug`（:1525，断言"两个检查不共用 slug"） | 每种拒绝在 fail 日志里可区分（如 `guest_ram_slice_end_past_import`） | provider 侧 `ContractError` 的对齐 |
| provider 侧 `provider_host_region_window ... import=no-copy write=in-place release=ok` | `metal-api-emulator/examples/metal-smoke/src/provider_suite.rs`（用例 `provider_host_region_window`，按名字检索） | "注册区间 → 派生窗口 → 无拷贝导入 → 设备原地写入 → lease 释放"这条链在 Lavapipe 与 RTX 5060 上都 PASS（`docs/19` §3.1 步骤 2） | reims 的 guest RAM 是否接得上（无调用方） |

**必须说清的边界**：`vk_caps` 与 `guest_import_levels` 证明的是"reims 自己那条轨道通了"，
**不等于** provider 侧 owner 契约被使用过。

---

## 3. 接线设计

### 3.1 谁在什么时机注册 `HostRegion`

**owner = reims 设备进程**；**注册时机 = `guest_ram_map` 首次解析成功的那一次**，也就是
`resolve()`（`guest_ram_map.rs:750`）返回 `Resolved{imports, refusal}` 之后，
或在 `warm()`（:382）把整个映射建好之后。不要新增第二个时机：`mmio.rs:185` 的握手点已经
在 backend 发布粒度之后（`:995` 的 latch 先于第一个引用），顺序天然成立。

字段来源：

| `HostRegion` 字段 | 来源 | 备注 |
|---|---|---|
| `host_pointer` | `GuestRamImport::host_base()`（`reims-vgpu-memory/src/lib.rs:603`） | 已是 granularity 对齐后的基址（`new()` :462 会 trim），**不要**用原始 `GuestRamRegion.host_va` |
| `length` | `GuestRamImport::len()`（:581） | 同样是 trim 之后的长度；必须满足 `length % page_size == 0`，否则 `HostRegion::validate()` 直接拒绝 |
| `page_size` | `granularity()`（:886） | 即 backend 实测的 `host_pointer.min_alignment`；**不是** guest 的 4 KiB，也不是 `IMPORT_SPAN_CEILING` 的约数假设 |
| `lease_id` | reims 侧生成，与 `ImportId`（`reims-vgpu-memory/src/lib.rs:235`，分配点 `ImportId::allocate` :239）一一对应 | provider 要求非零；`ImportId` 本身进程单调、永不复用，正好满足"陈旧 lease 不得在新 import 上 resolve"的要求 |
| `owner_epoch` | 设备 epoch（recreate 时递增） | 必须与 `guest_ram_map::reset()`（:313）同步：reset 之后所有窗口归属新 epoch，旧窗口的 `reclaim` 不可再影响新映射 |

补充规则：

- 一个 `GuestRamImport` 注册一个 `HostRegion`，`ImportId` 只在日志里出现（`ImportId::get()`
  的文档明确"nothing may key a GPU resource on this without also holding the
  `GuestRamImport`"），所以 lease_id 要用**另一个**空间，不能裸用 `ImportId` 的数值键做资源键。
- packed alias（`new_host_allocation`，:525）也要注册：它的 `gpa_base()` 是 `None`，
  但 `host_base/len` 齐备，`HostRegion` 没有 GPA 字段，所以形状上完全兼容。
- 注册失败（`HostRegion::validate()` 报 `InvalidHostRegionPageSize` / `UnalignedHostRegion` /
  `NullHostPointer`，见 `metal-api-emulator/crates/metal-api-core/src/provider.rs:234`）
  必须让该 import **整体退回 copying rail**，与 reims 现有 `filter_map(...ok())` 的部分导入
  语义一致（`guest_ram_map.rs:750` 之后的 `filter_map`），不允许静默半注册。

### 3.2 wire 命令的 allocation 如何映射到窗口

**AllocationId 派生（建议，待确认见 §7）**：`AllocationId = f(owner_epoch, task_id, reference)`
加一个进程内序号，理由是 reims 侧的持久身份就是"task 局部对象引用"
（`bound_buffers.rs` 模块文档），而 provider 只要求非零且稳定。注意
`bound_buffers.rs` 模块文档记录过一条测量：**offset 不是资源级身份**（精确窗口 fallback
才需要 offset 进 key），所以 AllocationId 不要含 offset，否则一个 reference 会碎成上千个
伪 allocation。

**窗口派生**：两种形态，都要走 `HostRegion::borrowed_window(allocation_id, offset, length)`
（`provider.rs:281`）：

1. **RAMBlock 内的连续 run**：`references_for_runs()`（`guest_ram_map.rs:549`）已经把
   一个 guest 窗口切成 import 内连续的 run（跨 import 边界会分组）。每个 run →
   一个窗口，`offset` 是 run 相对该 import `host_base` 的偏移（即 `GuestSlice` 的
   `resolve()` 结果，`reims-vgpu-memory/src/lib.rs:694` 的 `BoundRange{offset,len}`）。
2. **packed alias**：整个 allocation 是**一个**宿主分配（`docs/15` 的"每 allocation 一次
   import"），所以一个 reference = 一个 import = 一个 `HostRegion`；绑定时取
   `head + offset` 起的一段（`bound_buffers.rs:583`），映射成该 HostRegion 上的一个窗口。

**坐标约定（2026-09-14 实测补充，接线前必读）**：窗口不在注册区基址时，wire 侧的
`BufferView.offset` 必须携带**窗口在 allocation 内的相对偏移**，resource 表里该
allocation 的 size 必须是**整个注册区**、不是窗口长度。实测两条反例（都在
`provider-smoke` 的 `provider_host_region_window` 里复现过）：

- 视图 offset 写 0 而窗口偏移非 0 → `ProviderError { slug: resource_contract_invalid }`
  报 `lease LeaseId(120) range end 4 exceeds allocation size 20480`；
- allocation size 写成窗口长度 → `LeaseRangeOutOfBounds { end: 20480, allocation_size: 8192 }`。

也就是说三个坐标各司其职：**allocation = 整个注册区**、**reservation = 窗口**、
**BufferView.offset = 窗口内偏移（allocation 坐标）**。
   这才是 `docs/14` range hazard 说的"同一 allocation 多 view"在 reims 侧的真实来源。

**对齐规则（这是最容易接错的一点）**：

- provider 侧强制 `offset`/`length` 都是 `page_size` 的倍数，否则 `UnalignedHostRegion`（:296）；
- reims 侧的 `GuestSlice` 已经"向外扩到 granularity 并记住真实起点"（:622 + 单测 :1325）。
  所以接线时**窗口对齐、view 精度**必须分开：窗口用 `bound_len()` 对齐的外扩区间，
  GPU 侧字节精度用 `head()`/`requested()` 表达（正是 `docs/14` 的 range hazard 负责冲突）。
  直接把 `requested()` 当窗口长度会 100% 触发 `UnalignedHostRegion`。
- page_size 越小越多的小窗口；实测粒度若为 64 KiB（部分 NVIDIA 配置）而不是 4 KiB，
  对齐会把很多 4 KiB 窗口放大——这是性能问题不是正确性问题，但要在文档里预期到。

**拒绝规则（必须与 reims 侧 `MapRefusal` 一致，不能一边放行一边报错）**：

| reims `MapRefusal` | 触发 | provider 侧对应 | 处置 |
|---|---|---|---|
| `NoBackendImport` | 无粒度（无扩展或 operator 关掉） | 不注册任何 `HostRegion` | 全量走 copying rail（`StagedLease`） |
| `HostRefused(...)` | shim 拒绝回答 | 无 | 保持现有行为，不注册 |
| `NoUsableRegion{spans}` | 每个 span 都被粒度/形状拒绝 | 无 | 保持现有行为，不注册 |
| `ImportExceedsHeap{needed,budget}` | 总和超堆预算 | 无（provider 只管单区间） | 整块退回 copying，不做部分导入 |
| `GpaNotInAnyImport{gpa}` | 未注册的 GPA | 无对应（还没到窗口层） | 保持拒绝 |
| `Scattered{...}` | 页散列，单窗口不可表达 | — | 落回 packed alias 或 gather |
| `OutsideImport(inner)` | 越界/溢出/零长/跨 import | `HostRegionWindowOutOfBounds`（:307/:3392）、`ArithmeticOverflow`、`ZeroLength` | 一律拒绝该次提交，不允许截断 |

### 3.3 脏页如何由 owner 消费

**分工**：provider 不做脏页跟踪（`provider.rs:340` 的 `DirtySet` 类型文档明说"the owner
consumes this to flush guest pages"），reims 侧也不通过 provider 回调拿脏页。链路是：

1. 提交/写回产生 `BufferWriteback{view_id, allocation_id, offset, bytes}`（:2442）；
2. owner 用纯函数 `DirtySet::mark_writebacks()`（:403）把这批 writeback 折成页对齐、
   升序不重叠的区间（`mark()` :367 会合并相邻/重叠）；
3. owner 把区间交给 reims 的既有 guest 写通道。

**reims 侧的既有事实（不要新建第二条通道）**：

- 设备自己写过哪些 guest 页已有账本：`crates/reims-vgpu/src/runtime/host_writes.rs`
  模块文档（"Which guest pages this device has written, and when"，并说明 hypervisor 的
  dirty bitmap 只看 guest CPU store，缺的正是"我们写过"这一半）；
- 落回 guest 页的两条实现：`crates/reims-vgpu/src/runtime/render_writeback/mod.rs`（从
  resident 落回，或延迟到 `runtime/writeback_debt.rs` 决定要付时）与
  `crates/reims-vgpu/src/runtime/mapping_write/mod.rs`（CPU 写 guest IOSurface 映射）；
- 与 hypervisor 的同步点已经在 QEMU ABI 里：`crates/reims-vgpu/src/qemu/abi.rs:75`
  （v13 的 `guest_written_pages`）与 :81（v12 的 `track_guest_writes`/
  `untrack_guest_writes`/`guest_write_gen`）。

**到 WHPX 的同步点（设计）**：`DirtySet` 的区间是"设备写过的 guest 物理页"，因此在
WHPX 场景下它**正好**是 `guest_written_pages` 想要的输入；owner 应在提交完成后、
解锁窗口之前把区间推给这条通道（顺序理由见 §3.4 第 3 步：必须先有字节落定，再让 guest
看见脏页）。**不做**的事：不用 `DirtySet` 反向驱动 provider，不要求 provider 在写回后
回调（那会破坏 `docs/13` 的"producer 不做 owner 的事"分层）。

### 3.4 失效/回收顺序（硬约束）

四步，顺序不可交换：

1. **地址层退休**：guest 宣布映射变化时，先按范围/引用退休解析结果
   （`BoundBuffers::retire_range` / `retire_ref`，`bound_buffers.rs`；由 `CmdMapMemory2`、
   `CmdUnmapMemory`、`CmdReplacePhysical`、`CmdSetObjectList`、`CmdDeleteObject`、
   `CmdDefineTask2`、`CmdDeleteTask` 驱动）。注意 reims 现有规则：**零长度通知不退休任何东西**，
   范围只在真正重叠时退休（模块内单测 `a_zero_length_range_retires_nothing`、
   `abutting_ranges_do_not_overlap`）。
2. **GPU 层退休**：该窗口的每次提交在 provider 侧 `LeaseLedger::bind(lease_id, token)`
   （`provider.rs:1397`）；观察到 retired 才 `retire(token)`（:1444）。
   未退休的 token 意味着 lease 不能释放（`release_ready`/`release` :1482/:1489）；
   `BorrowedLeaseRegistry` 侧的 `retain`（:1867）/`retire`（:1897）/`release`（:1911）
   是同一规则的无拷贝版本："release refuses while any retain is outstanding"。
3. **窗口退休 → 回收**：`GuestWindows::retire(lease)`（:477）之后才允许
   `reclaim(lease)`（:494）；未退休的回收被 `GuestWindowStillActive` 拒绝且**窗口保持注册**
   （单测 `guest_windows_refuse_reclaim_before_retirement` :4019）。
4. **归还宿主范围**：只有第 3 步成功，packed alias 的宿主分配才可以被释放/复用；
   RAMBlock import 本身（`guest_ram_map` 的 import）**不回收**——它活到 VM 生命周期结束，
   回收的是"这一次借出去的那段窗口的登记"。

另外两条必须写进实现的规则：

- **packed alias 要等该 reference 的所有 offset 分辨率都退休**：reims 现在按
  `(task, reference)` 存一份 packed import 并用 offset 表达 view（`bound_buffers.rs:583`），
  所以 alias 的释放条件是该 reference 的**所有**绑定都退休，而不是某一次 draw 退休。
- **设备重建**：`guest_ram_map::reset()`（:313）丢掉的 import 与 provider 侧设备销毁必须
  成对。`ImportId` 永不复用的语义保证旧 `GuestSlice` 不会 resolve 到新设备上；接线时必须
  保证 `owner_epoch` 也随之递增，否则 `HostRegion::validate()` 的非零检查挡不住"同 epoch
  新映射"这种静默错误。

### 3.5 错误路径 → 错误类型

provider 侧的 `ContractError` 已经有到 `ProviderErrorClass` 的映射
（`provider.rs:3072`–:3075），接线要沿用，不要另造字符串：

| 情形 | provider 类型（行号） | 映射后的 slug | owner 应做什么 |
|---|---|---|---|
| 窗口越界 | `HostRegionWindowOutOfBounds` | `Args` 类，slug `host_region_invalid` | 拒绝该次提交；退休/重解析该引用，不要截断后重试 |
| offset/length 未对齐 | `UnalignedHostRegion` | `Args` 类，slug `host_region_invalid` | 这是 reims 接线 bug（忘了用外扩区间），应当 fail closed 并打印 |
| 注册基址未按 page_size 对齐 | `UnalignedHostRegion{field:"pointer"}` | `Args` 类，slug `host_region_invalid` | 同上；owner 必须按最粗粒度分配/对齐注册基址（2026-09-14 实测补充） |
| page_size 非法（0 或非 2 的幂） | `InvalidHostRegionPageSize` | — | 注册期拒绝，退回 copying rail |
| 未注册 GPA | reims `GpaNotInAnyImport` | — | 保持 reims 现有拒绝路径 |
| lease 未退休就回收 | `GuestWindowStillActive` | `guest_window_still_active` | 说明有在途提交；**保持窗口注册**，等 stamp 观察后重试 |
| 未知 lease 的退休观察 | `UnknownLease` | — | 拒绝，防止跨 owner 误退休 |
| 设备丢失 | reims 侧 `vk::Result::ERROR_DEVICE_LOST`（`crates/reims-vgpu/src/backend/vulkan/engine/queue_owner.rs:48`、:443） | `docs/13` §3.3 的设备丢失快速路径 | 走 provider teardown：所有 lease 视为退休并释放；`DirtySet` 不再导出到 guest（字节可信度已丢） |

补充：`docs/13` 的 `ProviderHealth` 与 `AbandonmentBudget` 是这一层的上游契约——
`Exhausted` 时窗口不再接受新提交，而不是继续导入；`DeviceLost` 是唯一的"批量退休"授权。

---

## 4. 可验证性

每步都给出"Linux/WSL 先验证"与"Windows RTX 5060 真机验证"，并标出只能真机做的部分。

### 步骤 A：注册等价物（本机可全验）

```sh
# provider 侧：区间形状与窗口拒绝（纯 CPU，无 GPU）
cd /home/hiliang/hackintosh/metal-api-emulator
cargo test -p metal-api-core host_region          # :4144/:4167 两个单测
cargo test -p metal-api-core guest_windows       # :4019 未退休不得回收

# reims 侧：现有导入轨道仍然通过（回归基线）
cd /home/hiliang/hackintosh/repos/fork-reims-vgpu
cargo test -p reims-vgpu-memory
```

观测点（结构性验证，不依赖 GPU）：`HostRegion::validate()` 对非法 page_size 的拒绝；
`guest_ram_span n=…/… gpa=… len=… mib=…`（`guest_ram_map.rs:821`）在开机后**每个 span
各一行**。**待确认**：目前没有一条日志把 `HostRegion` 注册与 `guest_ram_span` 对齐，
所以这一步的"投影正确性"只能靠新单测断言，不能靠日志（见 §7）。

### 步骤 B：窗口派生（Lavapipe 可验）

```sh
# 已有证据（步骤 2，provider_host_region_window）；2026-09-14 起该用例扩展为
# 对齐矩阵 + 回收链断言，重跑用下面这个入口（注意：`--executor standalone` 不打印它）
cd /home/hiliang/hackintosh/metal-api-emulator
cargo run --locked -p metal-smoke --bin provider-smoke
# 期望出现（参数化 PASS 行；用例体在 provider_suite.rs，按名字检索即可）：
#   PASS provider_host_region_window page_size=<实测> page_source=backend_measured
#        refusals=unaligned_offset|unaligned_length|out_of_bounds|granularity
#        import=no-copy write=in-place outside_window=untouched release=ok
#   PASS provider_guest_window_reclaim lease=<n> retired=false reclaimable=false
#        refusal=guest_window_still_active release_ready=false
#   PASS provider_guest_window_reclaim lease=<n> retired=true reclaimable=true reclaimed=true
```

- 本机期望：Lavapipe 上 `import=no-copy` 成功、设备原地写入可见、lease 释放通过。
- **粒度不再是假设**：用例从 `provider.no_copy_alignment()`（即驱动上报的
  `minImportedHostPointerAlignment`）取 `page_size`，并额外用 2× 粒度做一次"粗粒度注册"
  门禁，因此即使后端恰好报 4096 该用例也不会退化成恒过。RTX 5060 上请核对打印出的
  `page_size=` 是否 ≠ 4096。
- **回收链断言**：`provider_guest_window_reclaim` 证明"未退休不可回收、退休后可回收"，
  对应 §3.4 的单向链。
- reims 侧的等价断言**当前不存在**（没有 provider 调用方），所以这一步在 Linux 上只能验证到
  "窗口边界/对齐/拒绝规则"这一层——用 `GuestRamImport::slice()` 现有单测风格补即可：
  外扩对齐、越界拒绝、跨 import 边界切分成两个 run。

### 步骤 C：Windows RTX 5060 真机

```powershell
# 真机观测点一：驱动粒度与预算（决定 HostRegion.page_size）
#   期望：host_pointer_import=supported 且 host_pointer_align=<实测粒度>
Select-String -Path .\reims-*.log -Pattern 'vk_caps .*host_pointer'

# 真机观测点二：导入水平（分母是 shim 报的 span 数）
Select-String -Path .\reims-*.log -Pattern 'guest_import_levels|guest_ram_span'

# 真机观测点三：拒绝必须带具名 slug（不是笼统的"导入失败"）
Select-String -Path /tmp/reims-vgpu-fail.log -Pattern 'guest_ram_'
```

判据：`guest_ram_span` 的每个 span 都能在 `guest_import_levels` 的 `ramblocks=n/m` 里找到
对应（惰性导入意味着开机瞬间 `n` 可以小于 `m`，但驱动第一次引用后必须收敛）；
任何被拒绝的窗口都必须在 fail 日志里出现 `guest_ram_*` 的具名 slug。

**只能真机验证的部分（必须在真机上做，不要用 Linux 结果代替）**：

1. `min_alignment` 的真实取值（决定 page_size 与窗口放大倍数）；
2. `IMPORT_SPAN_CEILING` 的必要性（那 2 GiB 上限来自实测到的一次驱动截断，
   `host_pointer.rs:399` 的注释是唯一出处）；
3. Metal-direct 臂的 `newBufferWithBytesNoCopy` 基址对齐（`backend/metal/runtime.rs:86`），
   以及该臂**目前根本没有导入器**（:23）；
4. 设备丢失的真实恢复路径（`docs/13` §3.3；当前主要是注入模拟，见记忆中的未完成清单）。

---

## 5. 边界与未完成项（本文不做）

- **不改 trace/wire 线格式**：窗口信息经既有 lease 通道表达（`docs/19` §4 的同一约束）；
  本文不新增 opcode，也不改 `reims-vgpu-wire` 的记录头（`wire/src/op.rs`）。
- **不做 dma-buf**：`reims-vgpu-memory` 的模块文档明确记录 dma-buf rail 已被 host-pointer
  rail 取代（"What this replaces"），本文不复活它。
- **不承诺 pin/可撤销**：模块文档 "# What this type does not promise"（:39）明确
  `VK_EXT_external_memory_host` 不保证页被 pin、也不保证可撤销；
  "页被回收/重映射"仍是 PTE 损坏类问题，靠既有的 surface page-ownership 守卫。
- **不做 Metal-direct 导入器落地**：`backend/metal/runtime.rs:23` 的"No importer is wired
  here yet"是本文的现状而不是目标；本文只保证设计在该臂上形状成立。
- **不做 guest 页内容迁移/交换**，不承诺 GPU 与 CPU 的缓存一致性（一致性由 owner 的同步点
  负责，`docs/19` §4）。
- **不改 QEMU ABI**：脏页走 v12/v13 既有通道（`qemu/abi.rs:75`/:81），不新增回调。
- **不声称完整 Metal conformance**：本文只覆盖 guest memory 这一个语义域。

---

## 6. 与现有文档的引用关系

| 文档 | 与本篇的关系 |
|---|---|
| `docs/13`（有界回收与错误传播） | 上游：`AbandonmentBudget`/`ProviderHealth`/设备丢失快速路径（§3.1–3.3）决定本篇 §3.5 的"未退休不得回收"与"设备丢失即批量退休" |
| `docs/14`（资源范围与共享分配） | 上游：range hazard 与"同一 allocation 多 view"决定 §3.2 的窗口粒度；写回合并规则决定 §3.3 的脏页折叠语义 |
| `docs/15`（共享分配与宿主字节互斥） | 上游：packed alias = "每 allocation 一个宿主分配"，正是 §3.2 形态 2 的挂靠点 |
| `docs/19`（guest-memory 生命周期） | **直接前序**：步骤 1–4 已完成（`HostRegion`/`borrowed_window`/`DirtySet`/`GuestWindows`），本篇是步骤 5 的设计输入 |
| `docs/09`（Metal API 统一后端目标） | 目标层：Gate 2 = 生产 guest/display 路径调用 canonical provider；Gate 3 = VM E2E（含 WHPX 脏页导出与显示路径） |

---

**2026-09-14 深夜进展（fork `reims-vgpu`，分支 `guest-memory-wiring`）**：

| 步骤 | 状态 |
|---|---|
| 投影 | 已落地：`guest_ram_map::host_regions`（`d2f6b07`）|
| 注册候选（对齐数学） | 已落地：`registration_candidates` + 结构化拒绝（`fea4ba5`）|
| 注册账本（窗口派生 / 计数退休 / reset 失效） | 已落地：`GuestRamRegistrations`（`7ade996`，53 条模块测试）|
| **注册调用点** | **已落地**：`register_imports` 在握手建立的导入上驱动投影 → 候选 → 账本，并输出本文件 §7 第 3 条要求的诊断行（`d38e706`）。**不改变任何导入决策** |
| 窗口派生接进 bind 路径 | **仍未做**：账本能派生窗口，但 `bound_buffers` 的绑定路径还没消费它 |
| `reset`/epoch 与设备重建配对 | **仍未做**：账本支持 `reset()` 递增 epoch，但触发点尚未接线 |
| 真机 `page_size` | 仍未做（见下条）|

诊断行形态（可直接与 `guest_ram_span` 对上）：

```text
guest_ram_registration imports=<n> candidates=<m> skipped=<k> epoch=<e> registered=<r>
```

## 7. 待确认（不要当成已核实事实使用）

1. **`min_alignment` 的真机取值**：`caps/mod.rs:199` 断言的是 4096，但那是 fixture；
   NVIDIA 真机是否仍为 4 KiB，必须真机读 `vk_caps ... host_pointer_align=` 才能定
   `HostRegion.page_size`。
2. **`AllocationId` 的稳定派生方式**：是否需要在 reims 侧新增一个 id 空间，还是复用
   `(task, reference)` 加进程内序号；§3.2 给的是建议，未与 provider 侧确认。
3. **统一诊断行**：目前没有一条日志能把"注册的 `HostRegion`"与"`guest_ram_span`"对上，
   §4 步骤 A 因此只能靠单测断言，不能靠日志；需要新增一行诊断（本文不写代码，故只记录需求）。
4. **packed alias 的所有权口径**：reims 用 `GuestRamImport::new_host_allocation()`（:525）建
   alias，provider 侧只看到"一个宿主区间"，没有任何 GPA 概念——语义可行（`HostRegion` 无 GPA
   字段），但两边对"这段内存归谁释放"的口径需要在实现时写死并加断言。
5. **WHPX 脏页导出的具体实现位置**：`docs/19` §5 已把它标为下游；本篇只确定了"复用
   `guest_written_pages` 通道、不由 provider 回调"这一条原则。

**下一步建议（3 条以内）**

1. 在 reims 落一个**只读**的 `guest_ram_map::host_regions(host)` 查询：把 import 列表投影成
   `HostRegion` 形状并加单测（不改任何行为，Linux 可全验）。
2. 在 provider 侧补一条 `AllocationId` 派生约定 + 一个跨结构的 borrowed 窗口 smoke
   （Lavapipe 跑，形状对齐后再接 reims）。
3. 真机读一次 `vk_caps` 的 `host_pointer_align`，把 `page_size` 口径冻结进实现。
