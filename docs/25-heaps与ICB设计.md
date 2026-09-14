# heaps / ICB 设计：从"每资源一块显存"到放置、别名与间接执行（2026-09-14）

> 本文是图形/显示路径在 presentation 之后的下一个设计输入：定义 `MTLHeap` 语义（所有权、
> placement、aliasing、生命周期）与 indirect command buffer（ICB）语义（编码、继承、执行、
> 校验）在 canonical Metal provider 契约里的最小增量。它只做设计，不改代码。
> 前置：`docs/23`（render pipeline 启动）、`docs/24`（presentation/swapchain，其中 §6 Step 3–8
> 仍未完成）、`docs/14`/`docs/15`（range hazard 与共享分配）、`docs/13`（回收与设备丢失）、
> `docs/16` §1 的依赖序。代码锚点在 `metal-api-emulator@541be0c`（撰写时 checkout），行号会
> 漂移，引用前重跑 §8 的命令。

## 1. 问题与目标

### 1.1 为什么排在 presentation 之后

`docs/16` §1 钉的依赖序是：

```text
纹理资源 → sampler → 纹理读取的 compute 用例
   → render pipeline（顶点/片元）→ render pass 与附件
   → presentation / swapchain → heaps / ICB
```

heaps/ICB 排在 presentation 之后有两个结构性理由：

1. **heaps 是对 allocation 的一次"重排"，必须等资源全集稳定。** 现在每 admission 一次就把
   allocation 物化为一块独占 `VkDeviceMemory`/`MTLBuffer`/`MTLTexture`（§2.2/§2.3），没有"一块
   显存放多个资源"的概念。要引入 placement 与 aliasing，必须先知道有哪些可放置对象：compute
   buffer、采样/存储纹理、render 附件、present 目标。附件（`docs/23`）与呈现目标（`docs/24`）
   落地后，heap 才能表达"同一块显存先后被附件与呈现目标复用"，否则设计出来的是只有 buffer 的
   半成品 heap，后面还要返工。

2. **ICB 是对 render/compute pass 的"离线重编码"，必须等两种 pass 与对象 API encoder 稳定。**
   Metal 的 `MTLIndirectCommandBuffer` 记录的是"以后要执行的 draw/dispatch 命令"，间接 render
   command 继承 encoder 的 pipeline 与资源绑定。render pass 形状（`docs/23`）与对象 API 的
   encoder 面（`provider_api.rs:699` 现在只有 `compute_command_encoder`，render encoder 仍缺，
   见 `docs/24` §5.2）不定稿，ICB 的"继承什么、校验什么"就没有落点。

一句话：**presentation 之后才做 heaps/ICB，不是它们不重要，而是它们的契约里引用的一切——
附件、呈现目标、两种 pass、对象 API encoder——都刚在 `docs/23`/`docs/24` 里定了形状。**

### 1.2 heaps 要解决什么

`MTLHeap` 语义（Metal 侧真实 API：`MTLHeap` 协议、`MTLHeapDescriptor`、
`newHeapWithDescriptor:`、`[heap newBufferWithLength:options:]` / `[heap newTextureWithDescriptor:]`）
拆成四件可独立建模的事：

| 维度 | 语义 | 现状缺口 |
|---|---|---|
| 所有权 | 一块 GPU 显存块归一个 heap，heap 活着，块就活着 | 显存块绑死在单个 allocation 上，allocation 释放即销毁（§2.2） |
| placement | 在 heap 内按 offset+alignment 放置 buffer/texture | 每次 `bind_*_memory(..., 0)`，offset 恒为 0（§2.2） |
| aliasing | 两个生命周期不重叠的资源共享同一字节区 | `AliasMode::DistinctViews` 把别名物化为独立 device buffer，**从不共享字节**（§2.2） |
| 生命周期 | 资源释放归还子区间，heap 整体回收 | 没有"heap 内子区间归还"概念，只有整 allocation 回收（`docs/13`） |

现状等价于"每个资源一个 1-resource heap"，即 heap 语义的退化情形。要做的是把 placement 与
aliasing 从"物化为独立内存"变成"真正的共享显存块 + 显式 hazard 隔离"。

### 1.3 ICB 要解决什么

`MTLIndirectCommandBuffer` 语义（真实 API：`MTLIndirectCommandBuffer` 协议、
`MTLIndirectCommandBufferDescriptor`、`newIndirectCommandBufferWithDescriptor:maxCommandCount:options:`、
`MTLIndirectRenderCommand` / `MTLIndirectComputeCommand`、
`[encoder executeCommandsInBuffer:withRange:]`）拆成四件：

| 维度 | 语义 | 现状缺口 |
|---|---|---|
| 编码 | 把 draw/dispatch 命令离线写进 buffer，而非即时提交 | 只有即时提交，没有"写命令到 buffer"的面 |
| 继承 | indirect render command 继承 encoder 的 pipeline/顶点/资源绑定 | 无此概念；render encoder 本身还没进对象 API（§2.3） |
| 执行 | encoder 用 `executeCommandsInBuffer` 回放一段范围 | 无回放入口 |
| 校验 | 命令种类、继承是否完整、范围是否越界 | 无校验面 |

### 1.4 第一个可验证里程碑（一句话）

**一个双资源 aliased heap（两块生命周期不重叠的 buffer/texture 放到同一块显存、先后各自写回）
与一个单 draw 的 ICB（离线编码一次 draw、encoder 回放、附件字节与直连 draw 一致），在五条路径
上字节/计数一致。**

### 1.5 里程碑验收判据（可判定，不含主观项）

- **heap**：两个 resource 的 placement offset 由 fixture 写死；两者生命周期不重叠；先后各自
  写回的字节与 fixture 期望逐字节一致；两个 resource 的实际 backing 在 provider 侧落在同一块
  device memory 的不同 offset（用观测通道记录 placement，而非"看起来像共享"）。
- **aliasing 防伪**：用哨兵字节（沿用 v13 附件 / v11 首字 `0xFFFFFFFF` 的先例）证明"共享后
  后写的资源确实覆盖了前一个"，而不是两个 resource 各占一块内存假装别名。
- **ICB 防伪**：ICB 编码的 draw 结果必须与同一条 render pass 的直连 draw 字节一致；且 ICB
  执行后附件字节 ≠ 哨兵，证明回放确实发生。
- **明确不进第一段**：可变 ICB 大小（`maxCommandCount` 精确预算先固定）、
  `MTLIndirectCommandBuffer` 的 CPU/GPU 并发填充（`MTLResourceStorageMode` 组合）、
  `MTLPurgeableState`、多 ICB 合并执行、heap 碎片整理/紧凑。

## 2. 现状核对

每条都能用 §8 的命令或同形 `rg -n` 复现。

### 2.1 `crates/metal-api-core/src/provider.rs`：allocation/view/AliasMode/能力位/admission

- **资源表是 allocation + lease，没有 heap。** `AllocationRecord`（`:2062`）只有
  `allocation_id`/`owner_epoch`/`size`；`ResourceTableSnapshot`（`:2091` 起）持有
  `allocations` 与 `leases` 两张 `BTreeMap`。没有 `HeapId`/`HeapRecord`/placement 字段。
- **别名能力是一个三值开关，不是放置语义。** `AliasMode`（`:4948`）只有
  `Refused`/`DistinctViews`/`ExplicitPolicy` 三值；`BufferRange`（`:4961`）与 `RangeSet`
  （`:5000`）是 `docs/14` 提出的 provider 侧 hazard 值单元，注释明确"nothing is wired to them
  yet"（`:4957`）。
- **存储模式闭集三值。** `StorageMode`（`:5041`）= `OwnedBytes`/`StagedLease`/`BorrowedNoCopy`。
  没有 heap-backed storage mode（真实 Metal 里 `MTLStorageMode` 与 heap 交叉，但契约层目前不
  表达）。
- **能力位已经用"字段存在但默认关闭"的风格扩过 render 与 present。** `ProviderCapabilities`
  （`:4215`）里有 `supports_render_passes`/`max_color_attachments`/`max_attachment_dimension`/
  `supported_color_formats`（`:4237-4255`）与 `supports_presentation`/`max_present_targets`/
  `supported_present_modes`/`max_present_image_count`（`:4268-4285`），并由
  `declares_render_support`（`:4286`）/`declares_presentation_support`（`:4298`）门控 `MCC1`
  是否携带这些位。**heaps/ICB 应沿用同一套风格，而不是另起一套。**
- **admission 已有两个能力门。** `admit`（`:4330`）先 `trace.validate()`，再
  `admit_render_passes`（`:4575`）再 `admit_present_actions`（`:4662`），拒绝用 typed slug
  （如 `render_passes_unsupported`/`present_targets_unsupported`）。heaps/ICB 应追加第三个、
  第四个门，同样在 reservation 之前 fail closed。

### 2.2 `crates/metal-api-vulkan/src/`：每 allocation 一个 device buffer / image

- **compute buffer 每 allocation 一块内存。** `lib.rs:3378` `create_buffer`（STORAGE_BUFFER、
  EXCLUSIVE）→ `:3400` `allocate_memory` → `:3410` `bind_buffer_memory(buffer, memory, 0)`。
  offset 恒为 0。borrowed lease 路径同构：`lib.rs:3475` `create_buffer` → `:3526`
  `allocate_memory`（带 `ImportMemoryHostPointerInfoEXT`）。
- **纹理每 allocation 一个 image。** `lib.rs:3146` `create_image`（TYPE_2D、LINEAR、SAMPLED），
  内存走 `lib.rs:2137-2165` 的 `create_image` helper：`:2141` `create_image` → `:2155`
  `allocate_memory` → `:2165` `bind_image_memory(image, memory, 0)`。
- **render 附件同构。** `render.rs:624` 附件 `create_image`（OPTIMAL）；读回 buffer 也是独立
  分配：`render.rs:808` `create_buffer`（TRANSFER_DST）→ `:831` `allocate_memory`。
- **别名被保守地物化为独立内存。** `compute_provider.rs:179-187` 把 `alias_mode` 设为
  `DistinctViews`，注释写明"every pool entry owns its own device buffer，so two disjoint views
  of one allocation never share GPU bytes"。这是**安全但不可用**的 heap：它用多份内存买到了
  正确性，没有表达 `MTLHeap` 的共享与复用。

### 2.3 `crates/metal-api-native/src/`：对象与能力位

- **能力位同 render/present 一起声明，没有 heap/ICB 位。** `native.rs:176-229` 构造
  `ProviderCapabilities`（`supports_render_passes` 在 `:214`、`supports_presentation` 在
  `:221`）；`render.rs:92-130` 的 `RenderCapabilityBits` 只有 render/present 两组位。全仓
  `rg -i "MTLHeap|IndirectCommandBuffer|heap|icb"` 无实现命中（唯一命中是 `provider_api.rs:299`
  一句 "shares the same allocation" 的注释，与本设计无关）。
- **native 侧每个 buffer/texture 也是独立 `MTLBuffer`/`MTLTexture`。** `native.rs:940/957/974`
  用 `MTLResourceOptions::StorageModeShared` 逐个创建，没有 heap-backed 资源。

### 2.4 `conformance/`：观测通道与 rail 标记

- **五后端观测通道表。** `compare.py:20-26` `ALLOCATION_OBSERVATIONS`：`native-metal` 是
  `gpu-buffer-readback`，其余四条是 `host-writeback-landing`。
- **count 契约。** `compare.py:539-540`：provider 后端的 result 字段要么是
  `{id, completion, writebacks, allocations}`，要么加 `copy_in/copy_out`（counted），再加
  `group_counts`（grouped）。`copy_in/copy_out` 的派生规则见 `docs/15` §5 与 `docs/24` §5.3。
- **`capture_rails` marker。** `compare.py:463-467` 校验 `capture_rails` 必须是**互异的已知
  backend**；`_render_plan`（`:450-510`）把附件落点压成 `(allocation, view, offset, length)`
  四元组，保证附件不能被 buffer writeback 冒充。
- **新 suite 必须工具先支持。** `tools/lavapipe-smoke.sh` 自动发现 `suite-v*.json`，所以
  `suite-v15.json`（heap/ICB）必须在 capture 工具与比较器都支持之后再提交（`docs/18` 的
  原始阻断点纪律）。

### 2.5 reims 侧已有 ICB/draw（边界，不是本文目标）

`docs/09` §7.3 把 `runtime/icb/metal.rs`（ICB materialize/fill/cache）与
`runtime/draw/metal/{mod,icb}.rs`（render encoder、PSO、attachments）列为 reims-vgpu 里
"暂不迁移的 direct-Metal consumers"。这是**guest/macOS 侧**的实现，跑在 reims 自己的 Metal
后端上；本文要设计的是 **Windows 侧 canonical provider** 的中立契约与等价实现。两者会通过
"同一份中立契约"在将来对齐，但本文不迁移、不改 reims 代码。

### 2.6 现状小结（一句话）

资源层是"每 allocation 一块独占显存、offset 恒为 0、别名用多份内存买正确"，执行层只有即时
compute/render 提交：heaps 缺 placement/aliasing/生命周期，ICB 缺离线编码/继承/回放/校验。

## 3. 三处可得性论证

先给结论，再给实测证据。凡未实测的一律标"待确认"，不臆造 API 名。

### 3.1 Windows（产品宿主，RTX 5060 via dzn/Dozen）

- **heap：有，核心 Vulkan 内存绑定可用。** dzn 实测 `memoryHeaps=2`、
  `maxMemoryAllocationCount=4096`（§8.2），`vkBindBufferMemory`/`vkBindImageMemory` 带 offset
  是核心功能。因此"一块 `VkDeviceMemory` + 多个 buffer/image 按 offset 绑定"（=`MTLHeap`
  placement）在 dzn 上可表达。aliasing（重叠绑定）需要 provider 自己做显式 hazard 隔离，对应
  Metal 的 `MTLHazardTrackingMode`，这是契约层要建模、provider 层要落地的部分。
- **ICB：DGC 不可用，只能走间接 draw/dispatch 或 secondary command buffer。** dzn 实测
  `multiDrawIndirect=true`、`drawIndirectFirstInstance=true`、`VK_KHR_draw_indirect_count`
  （`drawIndirectCount=true`），但**不暴露 `VK_EXT_device_generated_commands`**（§8.2 的 awk
  只匹配到 llvmpipe）。所以 `MTLIndirectCommandBuffer` 若映射到 DGC，在产品宿主上直接不可用；
  主契约必须落在 `VkDrawIndirectCommand`/`VkDrawIndexedIndirectCommand`/`VkDispatchIndirectCommand`
  + `vkCmdDrawIndirect`/`vkCmdDrawIndexedIndirect`/`vkCmdDispatchIndirect`（CPU 先编码命令再回放），
  或 `vkCmdExecuteCommands` 的 secondary command buffer。**这条映射选择是 §9 待确认第 1 条。**

### 3.2 WSL（本机 Lavapipe/llvmpipe 开发）

- **实例 1.4.357、三个 ICD（lvp/dzn/nvidia）** 与 `docs/24` §7.1 同源。llvmpipe 实测
  `apiVersion=1.4.354`、`memoryHeaps=1`。
- **llvmpipe 有 DGC。** 实测暴露 `VK_EXT_device_generated_commands`（rev 1）与
  `VK_KHR_maintenance4`，feature `deviceGeneratedCommands=true`（§8.2）。→ **DGC 在本机开发
  可用，但产品宿主 dzn 不可用**，所以 DGC 只能做探针/可选优化，绝不能进主契约。
- 结论：WSL/Lavapipe 能覆盖 placement 与 indirect draw/dispatch 等价物的全部开发验证；DGC
  只作为额外探针，用来回答"如果宿主支持 DGC，ICB 能省多少"。

### 3.3 Apple GPU（CI macOS runner）

- **真实 API 是 oracle 的天然能力**：`MTLHeap`/`MTLHeapDescriptor`/`newHeapWithDescriptor:`/
  `newBufferWithLength:options:`/`newTextureWithDescriptor:`；`MTLIndirectCommandBuffer`/
  `MTLIndirectCommandBufferDescriptor`/`newIndirectCommandBufferWithDescriptor:maxCommandCount:options:`/
  `MTLIndirectRenderCommand`/`MTLIndirectComputeCommand`/`executeCommandsInBuffer:withRange:`。
- **但这些 API 在 CI 的 Apple Paravirtual 无窗口设备上能否创建、能否读回字节，待确认。**
  与 `docs/24` §7.5 同一个约束：CI runner 无窗口、无真实 display，不能假设
  `MTLIndirectCommandBuffer` 的 GPU 填充/回放路径可用。**不臆造 CI 可用性**，用
  `--heap-selftest`/`--icb-selftest` 式一设备检查做替代证据（§5.3）。
- 结论：Apple 侧负责回答"中立契约与真实 Metal 语义是否一致"；是否可字节级比对，先待确认。

### 3.4 只能真机/Apple 上做的断言（明确标注）

- **aliasing 的字节级正确性**（两个资源共享同一显存、后写覆盖前写、读回不串台）只能在真实
  device 上验证 hazard 隔离是否真的成立：Windows dzn 真机 + Apple 真机。Lavapipe 的软件实现
  能跑，但证明不了硬件内存复用语义。
- **ICB 与真实 Metal 的语义一致**只能 Apple 真机验证（`executeCommandsInBuffer:withRange:` 的
  继承与回放）。Windows 侧 indirect draw 等价物只能证明"Vulkan 三轨一致"，证明不了与 Metal
  语义一致。
- **placement 的 offset/alignment 约束**（`VkMemoryRequirements.alignment` vs `MTLHeap` 对齐）
  需要真机探针；Lavapipe 的 alignment 可能放宽，不能作为产品判据。
- 以上全部标"待确认"，验证办法写进 §8/§9。

## 4. 契约设计

### 4.1 能力位（显式、默认关闭，沿用 render/present 风格）

`ProviderCapabilities` 追加两组位，**字段存在但默认 0/空**，由新的
`declares_heap_support()`/`declares_icb_support()` 门控 `MCC1` 是否携带，与
`declares_render_support`（`provider.rs:4286`）同构：

```text
// heap 位组（默认关闭）
supports_heaps: bool
max_heap_bytes: u64                 // 0 = 无 heap
supported_heap_storage_modes: Vec<StorageMode>   // 空 = 无
supports_heap_aliasing: bool        // 第一版建议 false（§7.1）

// icb 位组（默认关闭）
supports_indirect_command_buffers: bool
max_indirect_commands: u32          // 0 = 无 ICB
supported_indirect_commands: Vec<IndirectCommandKind>   // 空 = 无
```

理由：与 `docs/23` §4.2 / `docs/24` §4.2 完全一致——旧 provider 的 legacy 能力帧字节不变，新位
只在 provider 真的声明时上 `MCC1`。

### 4.2 heaps 最小字段集与类型草图（字段级，不是可编译代码）

```text
struct HeapDescriptor {
    size: u64,                       // 第一版固定大小；碎片/紧凑不做
    storage_mode: StorageMode,       // 闭集三值复用；heap-backed 不进第一版
    allows_aliasing: bool,           // 第一版 false（§7.1）
}

struct HeapPlacement {               // 在 heap 内放置一个资源
    heap_id: HeapId,
    offset: u64,                     // 必须满足 alignment 与不溢出
    resource: HeapResource,          // buffer 尺寸 或 texture 描述
}
```

placement 校验（core 中立，provider 无关）：`offset + size <= heap.size`（溢出 fail closed）、
alignment 满足资源类型要求（alignment 值由 provider 在 admission 时回填，core 只校验不溢出与
offset 非负）、同一 heap 内两个 placement 若生命周期重叠且 `allows_aliasing=false` 则拒绝
字节区重叠（否则 aliasing 是真机才可验证的 hazard，见 §7.1）。

### 4.3 ICB 最小字段集与类型草图

```text
enum IndirectCommandKind { Draw, DrawIndexed, Dispatch }

struct IndirectCommandDescriptor {
    kind: IndirectCommandKind,
    // draw：顶点数/实例数；draw_indexed：索引；dispatch：threads
    // 继承面：indirect render command 继承 encoder 的 pipeline/顶点/资源绑定
    // indirect dispatch 继承 compute pipeline
}

struct IndirectCommandBufferDescriptor {
    max_commands: u32,               // 第一版固定，一次编码完
    kinds: Vec<IndirectCommandKind>, // 该 ICB 允许的命令种类
}
```

执行建模为"encoder 上的一次回放动作"：`execute(icb_id, range)`。校验面：`range` 不越界、
`kind` 在 `kinds` 白名单内、回放时继承所需的 pipeline/绑定已由 encoder 提供（否则
`icb_inheritance_missing` 拒绝）。

### 4.4 core 中立 vs provider/宿主侧

| 层 | 职责 |
|---|---|
| **core 中立**（`metal-api-core`） | `HeapId`/`HeapDescriptor`/`HeapPlacement`/`IndirectCommandBufferId`/`IndirectCommandKind`/`IndirectCommandDescriptor` 值类型、校验、能力位、`admit` 门、`MCC1` 加法式线格式 |
| **provider/宿主**（vulkan/native） | Vulkan：`VkDeviceMemory` slab + offset 绑定 + 显式 barrier；native：`MTLHeap` + `newBuffer/newTexture`、`MTLIndirectCommandBuffer` + `executeCommandsInBuffer` |
| **不进 core** | `VkDeviceMemory`/`MTLHeap`/`MTLIndirectCommandBuffer` handle、真实显存地址、`vkQueuePresentKHR`/窗口句柄 |

core 契约**只描述"要一块多大、什么模式、放什么、能否别名、回放哪些命令"**，不描述"这块显存
在哪、句柄是什么"。

### 4.5 加法式线格式与 typed-refuse slug

- **线格式**：沿用 `MCC1` 的 tagged 载荷加法式扩展（`docs/24` §4.1 的形状一/形状二选择对
  heap/ICB 同样适用：新增 tag，不重编旧 tag）。旧 trace 解码不变；能力位由 `declares_*` 门控
  只携带非默认位。
- **typed-refuse slug 清单**（沿用既有能力拒绝的 slug 风格：`render_passes_unsupported`
  `provider.rs:4581`、`present_targets_unsupported` `provider.rs:4668`，`capability_error`
  定义在 `:4733`、`ProviderError.slug` 在 `:5660`；最终名待确认）：`heap_unsupported`、
  `heap_alias_unsupported`、`heap_placement_overflow`、`heap_placement_misaligned`、
  `icb_unsupported`、`icb_command_unsupported`、`icb_inheritance_missing`、
  `icb_range_out_of_bounds`。**不用**泛化的 `provider_unavailable` 掩盖具体缺口。

### 4.6 明确不做（第一版之外，带触发条件）

| 不做项 | 理由 | 什么时候做 |
|---|---|---|
| heap aliasing（`allows_aliasing=true`） | 需要真机验证 hazard 隔离，且与 `docs/14` range hazard 强耦合（§7.1） | placement 跑通、`docs/14` 的 range hazard 落地后 |
| 可变 ICB 大小 / CPU-GPU 并发填充 | 需要 `MTLResourceStorageMode` 组合与在途状态机 | 单次编码回放跑通后 |
| `MTLPurgeableState` / heap 紧凑 | 需要真实显存压力与驱逐语义 | 有内存预算需求时 |
| 多 ICB 合并执行 | 需要跨 ICB 依赖与 range 语义 | 单 ICB 稳定后 |
| heap 碎片整理 / 不可移动资源 | 移动需要"资源地址不变"约束，破坏 placement 的简单性 | 有碎片实证时 |

## 5. 五路径比较方式

### 5.1 观测通道是否复用现有 allocations/writebacks/copy 计数

- **heaps**：aliasing 与 placement 的最终正确性仍落在 `writebacks`/`allocations` 的字节上，
  复用 `compare.py:88-125` 的 `_compare_observation`。新增的是 **heap 段**：每个 placement 的
  `(heap_id, resource, offset)` 与"该 heap 内放置的资源集合"，用来证明"两个 resource 真的落在
  同一 heap"。
- **count 口径（待确认）**：`copy_in/copy_out` 是"按 distinct allocation 计数"（`docs/15` §5），
  引入 heap 后，heap 内每个 resource 是继续各自算一个 allocation，还是按 heap 算一个？**这条
  决定 `compare.py` 强制分支与真机归档判据，必须一次说清"v15 起强制、v14 不变"**（`docs/24`
  §7.6 同款纪律）。
- **ICB**：回放的最终结果仍是 attachment/buffer 字节，复用 `_render_plan`（`:450-510`）；新增
  **icb 段**：编码的命令数与回放范围。

### 5.2 `capture_rails` marker 怎么用

- 新 suite 用 `capture_rails`（`compare.py:463-467`）声明哪些 rail 欠它，未点名的 rail 不得
  上报（沿用 v13 只有 `vulkan` 一条的先例，`docs/23` §5.2）。
- **heap placement 用例**：预计只有 `vulkan` 一条 rail 可字节比对（native 的 `MTLHeap` 是否
  可读回待确认）。
- **ICB 用例**：Windows 侧 `vulkan`/`vulkan-objects`/`vulkan-objects-async` 三条轨可跑
  （indirect draw 等价物）；Apple 侧是否点名，取决于 `--icb-selftest` 在 Apple 上能否字节读回。

### 5.3 天然不可比较的 rail 与替代证据形式

- **Apple 侧 heap aliasing / ICB 回放最可能天然不可比较**：只要 Apple Paravirtual 无窗口设备
  不能读回 `MTLHeap` 共享区或 `executeCommandsInBuffer` 结果，`native-metal` 与两条
  `native-metal-provider*` 就无法产出同样字节。处理与 v13/`docs/24` §5.5 同构：**`capture_rails`
  不点名这三条 rail**，改用 `--heap-selftest`/`--icb-selftest` 式"一设备检查"（独立运行、退出码
  非零即失败、JSON 输出、归档 `evidence/`）。
- 若等价物连在 Apple 上都表达不了，替代证据降级为"对象创建 + 生命周期"级断言，并在报告里
  **显式写"该 rail 不参与 heap/ICB 比对"**，不允许静默缺席。
- **不得**用"看起来共享/看起来回放了"充当替代证据（`docs/23` §1.3 已定纪律）。

## 6. 实施步骤（每步独立可验证）

**Step 1 — core 纯类型与校验。** 新增 `HeapId`/`HeapDescriptor`/`HeapPlacement`/`IndirectCommandKind`/
`IndirectCommandDescriptor`/`IndirectCommandBufferDescriptor` 值类型及其校验（heap size 非零、
placement offset 非负且不溢出、alignment 校验、ICB `max_commands` 非零、kind 白名单、
range 不越界）+ 单测。
*验收*：新增单测通过；全仓行为零变化（v1–v14 捕获与测试计数不变）。先例：`docs/14` Step 1、
`docs/23` Step 1、`docs/24` Step 1 都是"纯类型先行、行为零变化"。

**Step 2 — trace 与线格式。** 按 §4.5 新增 tag 承载 heap/ICB 载荷；能力位加 heap/icb 位组
（§4.1），`declares_*` 门控 legacy 字节；IPC 往返 + "历史字节帧不变"回归；截断/超长/未知
kind 仍 typed-refuse。
*验收*：Lavapipe 全套件（v1–v14）不变；新增 heap/ICB 往返用例通过；旧 trace 字节不变。

**Step 3 — Vulkan heap 轨（先 placement，不做 aliasing）。** `VkDeviceMemory` slab + 多个
buffer/image 按 offset `bind_*_memory`；记录 placement 观测；`allows_aliasing=false` 时拒绝
字节区重叠。
*验收*：**本地 Lavapipe** 上双资源 placement 用例字节一致；`provider-capture --suite` 过
`compare.py --check`；placement 观测证明两资源落在同一 heap 的不同 offset。

**Step 4 — Vulkan ICB 轨。** 用 `VkDrawIndirectCommand`/`VkDispatchIndirectCommand` +
`vkCmdDrawIndirect`/`vkCmdDispatchIndirect` 做等价回放（CPU 先编码命令再回放）；DGC 只做探针
（§3.2）。
*验收*：**本地 Lavapipe** 上 ICB 回放的单 draw 附件字节与直连 draw 一致，且 ≠ 哨兵；
`compare.py --check` 过；ICB 段计数正确。

**Step 5 — 观测通道与 count 口径定稿。** `compare.py` 增加 heap 段与 icb 段；定稿 §5.1 的
count 口径；新增 `conformance/test_suite_v15.py` 钉住规则（placement 必须点名同一 heap、
ICB 回放结果不能由 buffer writeback 冒充、未点名 rail 不得上报）。
*验收*：纯 Python 测试绿（不依赖 GPU）；正反例都覆盖；v1–v14 plan 不受影响。

**Step 6 — 对象 API。** core 侧在 render encoder 之后补 heap/ICB 对象 API（`provider_api.rs:699`
现在只有 `compute_command_encoder`，heap/ICB 的创建与执行都挂在 encoder/device 上）；
`provider-capture.rs` 的 objects / objects-async 两条形状接上 heap/ICB。
*验收*：三条 Vulkan 轨在同一条 heap/ICB 用例上字节与计数一致；对象 API 的 commit/wait/readback
生命周期沿用现有回归。

**Step 7 — native 对称。** `crates/metal-api-native/src/` 加 `MTLHeap` +
`newBufferWithLength:options:`/`newTextureWithDescriptor:` 与 `MTLIndirectCommandBuffer` +
`executeCommandsInBuffer:withRange:`；能力位按 §4.1 加。
*验收*：**前置条件**是 native 的 heap/ICB selftest 在 Apple GPU 上通过（与 `docs/24` Step 7
同构：先 selftest，再翻能力位）。**Apple GPU 验收**：`native-metal` / `native-metal-provider` /
`native-metal-provider-objects` 三轨先内部一致，再谈五路径。

**Step 8 — conformance v15 与五路径。** 新增 `suite-v15.json`（双资源 aliased heap + 单 draw
ICB），`capture_rails` 按 §5.3 定；`tools/lavapipe-smoke.sh` 的自动发现严格后置；`README.md` 与
`RENDER-CAPTURE.md` 增补 heap/ICB 一节。
*验收*：Lavapipe 全套件绿；**RTX 5060 真机**三轨捕获 + `--check` 全 PASS；五路径
`compare-captures` 在能报 heap/ICB 的 rail 上一致；证据归档 `evidence/conformance-v15-<hash>-<date>/`。

## 7. 风险与开放问题

### 7.1 aliasing hazard（最核心）

两个资源共享同一显存区时，"后写覆盖前写、读回不串台"必须由 provider 显式保证。这与
`docs/14` 的 range hazard 是同一类问题，但单位从"buffer 半开区间"变成"heap 内的 placement
字节区"。Metal 用 `MTLHazardTrackingMode` 让驱动自动跟踪；Vulkan 需要 provider 自己下 barrier。
**待确认**：第一版是否只做 placement（`allows_aliasing=false`），把 aliasing 留给 `docs/14`
range hazard 落地之后。验证办法：真实 device 上让两个生命周期不重叠的资源复用同一区，读回
证明无串台（§3.4）。

### 7.2 heap 预算 / 碎片

`VkMemoryRequirements` 的 `alignment`/`size` 因设备而异，固定 size 的 heap 会出现对齐浪费与
碎片。资源不可移动（移动会破坏 placement 的地址不变量），碎片只能靠预留策略缓解。**待确认**：
第一版 heap 是否固定 size、是否要求 placement 顺序放置、是否提供"可用字节查询"（对应
`MTLHeap` 的 `maxAvailableSizeWithAlignment:`，是否在契约层表达待定）。验证办法：真机打印
heap 预算与 placement 对齐（§8.2 的 `memoryHeaps` 数只是起点）。

### 7.3 ICB 的 resource inheritance 与 hazard

indirect command 继承 encoder 的 pipeline/绑定，回放时这些资源必须仍在途且生命周期覆盖回放
窗口。若 ICB 编码时引用的资源在回放前被释放，就是新的 hazard 面，与 `docs/14` 的整资源在途
语义耦合。**待确认**：ICB 是"编码即冻结绑定"还是"回放时再解析绑定"；前者更接近 Metal 继承
语义，后者更接近 Vulkan secondary command buffer。验证办法：跨提交释放绑定资源、再回放，观察
typed-refuse 是否 fail closed。

### 7.4 与 `docs/13` 回收语义的耦合

heap 是长生命周期容器，其内资源释放 ≠ heap 释放；`docs/13` 的 `AbandonmentBudget`/`Health`/
设备丢失回收目前按 allocation 粒度。引入 heap 后要回答：预算耗尽时是按 heap 还是按 placement
回收；设备丢失后 heap 及其放置资源是否整体失效。**待确认**：第一版是否把 heap 当作
`ResourceTableSnapshot` 里的一个新 allocation 记录（`AllocationRecord` 只加一个 `heap_id`），
从而复用 `docs/13` 的回收，还是新增独立生命周期。验证办法：设备丢失注入回归（`docs/13` 已有
模拟注入先例）扩展到 heap。

### 7.5 其它

- **DGC 只在 llvmpipe 有、dzn 没有**（§3.1/§3.2）：主契约绝不能依赖 DGC；若将来想用 DGC 做
  优化，要单独设计"支持则走 DGC、不支持则退化"的路径，且退化路径必须与优化路径字节一致。
- **dzn 的 `apiVersion=1.2.354` 且 `shaderInt8=false`** 与 `docs/23` §2.2 的"设备选择要求
  Vulkan 1.3 + shaderInt8 + shaderInt64"看起来不一致（§8.2 实测）。这不在本文范围，但若
  `select_physical_device`（`lib.rs:750-790`）据此拒选 dzn，heap/ICB 真机判据就会没有宿主。
  **待确认**，留给设备选择轨，本文只标注。

## 8. 抽样验证（`rg -n` 命令与原始输出）

以下命令在撰写时的 checkout 上执行，输出为原始粘贴（行号会随提交漂移，引用前重跑）。

### 8.1 每 allocation 一块 device memory（offset 恒为 0）

```sh
cd /home/hiliang/hackintosh/metal-api-emulator
rg -n "create_buffer|allocate_memory|bind_buffer_memory|bind_image_memory" crates/metal-api-vulkan/src/lib.rs | head
rg -n "alias_mode = AliasMode" crates/metal-api-vulkan/src/compute_provider.rs
```

关键输出（摘要）：`lib.rs:3378`/`:3475` 两处 `create_buffer`、`:3400`/`:3526` 两处
`allocate_memory`、`:3410` `bind_buffer_memory(..., 0)`；`compute_provider.rs:187`
`capabilities.alias_mode = AliasMode::DistinctViews`（注释见 §2.2）。

### 8.2 本机 Vulkan 可得性（`vulkaninfo` 实测，2026-09-14）

```sh
vulkaninfo --summary 2>/dev/null | sed -n '1,40p'
vulkaninfo 2>/dev/null | awk '/^GPU[0-9]:/{gpu=$0} /VK_EXT_device_generated_commands/{print gpu " => " $0} /VK_KHR_maintenance4/{print gpu " => " $0}'
```

关键结论：GPU0 = `Microsoft Direct3D12 (NVIDIA GeForce RTX 5060)`（dzn，`apiVersion=1.2.354`，
`memoryHeaps=2`，`maxMemoryAllocationCount=4096`）；GPU1 = `llvmpipe`（Lavapipe，
`apiVersion=1.4.354`，`memoryHeaps=1`）。`VK_EXT_device_generated_commands` 与
`VK_KHR_maintenance4` **只出现在 GPU1（llvmpipe）**；GPU0（dzn）有
`VK_KHR_draw_indirect_count`、`multiDrawIndirect=true`、`drawIndirectFirstInstance=true`、
`drawIndirectCount=true`，但没有 DGC。dzn 的 `shaderInt8=false`（§7.5 备注）。

### 8.3 core 能力位与 admission 门

```sh
rg -n "pub struct ProviderCapabilities|declares_render_support|declares_presentation_support|fn admit_render_passes|fn admit_present_actions|pub enum AliasMode|pub enum StorageMode" crates/metal-api-core/src/provider.rs
```

关键输出（摘要）：`:4215`/`:4286`/`:4298`/`:4575`/`:4662`/`:4948`/`:5041`。

### 8.4 观测通道与 rail 标记

```sh
rg -n "ALLOCATION_OBSERVATIONS|capture_rails|copy_in|copy_out" conformance/compare.py | head
```

关键输出（摘要）：`:20-26` 五后端观测表、`:463-467` `capture_rails` 校验、`:539-540` counted
字段集。

## 9. 待确认清单（实现前必须钉死）

1. **ICB 的 Vulkan 映射**（§3.1/§7.3）：indirect draw/dispatch（CPU 编码）还是 secondary
   command buffer（`vkCmdExecuteCommands`）；DGC 只做探针。这决定 Step 4 与 ICB 继承语义。
2. **heap 第一版是否只做 placement、不做 aliasing**（§7.1）：`allows_aliasing=false` 起步，把
   aliasing 留给 `docs/14` range hazard。推荐此路径。
3. **count 口径**（§5.1）：heap 内资源是各算一个 allocation 还是按 heap 算一个；
   `copy_in/copy_out` 是否 v15 起强制、v14 不变。
4. **Apple 侧 selftest 形式**（§5.3）：`--heap-selftest`/`--icb-selftest` 是否是最终形式；以及
   `allocation_observation` 这个名称（`compare.py:20-26`）在出现 heap/ICB 后是否仍合适。
5. **ICB 继承冻结点**（§7.3）：编码即冻结绑定 vs 回放再解析；前者更贴近 Metal，后者更贴近
   Vulkan secondary command buffer。
6. **heap 与 `docs/13` 回收的耦合**（§7.4）：heap 作为 allocation 扩展字段复用回收，还是独立
   生命周期。
7. **heap 预算/对齐**（§7.2）：固定 size 起步、placement 顺序放置；`maxAvailableSizeWithAlignment:`
   等价物是否进契约。
8. **dzn 设备选择不一致**（§7.5）：`shaderInt8=false`/`apiVersion=1.2.354` 与 `docs/23` §2.2 的
   选择条件冲突，是否影响 heap/ICB 真机宿主，留给设备选择轨确认。

## 10. 实施状态（2026-09-14 更新）

- **Step 1 完成**：heap/ICB 值类型与校验（`HeapId`/`HeapDescriptor`/`HeapPlacement`/`HeapResource`、
  `IndirectCommandBufferDescriptor`/`IndirectCommandDescriptor`/`IndirectCommandRange`）、默认关闭的
  能力位与 8 个 typed-refuse slug（`2f9e4ed`）；admission 门本轮为独立 `pub` 方法，Step 2 起接进
  `ProviderCapabilities::admit`。评审 Approved（3 个 Minor 记入延后清单）。
- **Step 2 完成**：`ComputeTrace` 新增可选 `heap`/`indirect` 载荷（`HeapPayload`/`IndirectCommandPayload`）、
  MCC1 新增加法式 `SUBMIT_HEAP_ICB_REQUEST = 0x10` 与 tagged 尾段、能力位按 `declares_heap_support()`/
  `declares_icb_support()` 进入扩展能力帧、`admit` 在碰资源前接线 `admit_heap_payload`/
  `admit_indirect_payload`（默认快照 slug `heap_unsupported`/`icb_unsupported`）（`e8a9b21`）。
  评审抓到"能力帧无条件追加 heap/ICB 字段"的真实回归 → 修复为**可选尾段**（编码按 `declares_*` 门控、
  解码按剩余字节判定），并用 pre-heap 提交 `2ad57d1` 实测出的帧字节做钉子（`7a9c98e`）。
  合并 `ad48dad`，CI run `34787709274` 五 job 全绿；v1–v14 捕获字节不变。
- **Step 3 完成**：Vulkan heap placement（`ff5385f` + 修复轮 `9f0e853`）：placements 与升序
  distinct owned allocation 一一对应（数量/尺寸/heap-id 不符走 `heap_placement_mismatch`，texture
  走 `heap_placement_unsupported`），单 `VkDeviceMemory` slab + 逐 allocation `bind_buffer_memory`，
  per-view upload/readback/writeback 不变；观测器
  `heap_placement_observations() -> Vec<HeapPlacementObservation{heap_id, allocation_id, offset, byte_size}>`
  每次 submit 整向量替换；能力位翻转（`supports_heaps=true`、`max_heap_bytes=64 MiB`、
  `[OwnedBytes]`、`supports_heap_aliasing=false`）。评审 Spec ✅ / Needs work（1 Important：bind 失败
  清理顺序先 free 后销毁 buffer）→ 修复轮已落地并加无设备单测钉顺序，另修 placement 尺寸裁决与观测
  无界累积。合并 `4d7aefb`，CI run `34817436502` 五 job 全绿；RTX 5060 真机
  `PASS provider_heap_placement heap=61 same_slab=true offsets=0,256 writeback=exact observations=2`
  （证据 `evidence/windows-rtx5060-heaps-step3-4d7aefb-2026-09-14/`）。
- **Step 4 第一增量完成**：Vulkan indirect draw 回放（`fbf6973`）：`render.rs` 新增
  `create_indirect_draw`（CPU 写一条 `VkDrawIndirectCommand` 到 host-visible `INDIRECT_BUFFER`）与
  `execute_indirect_render_pass`，`vkCmdDrawIndirect` 回放与直连 draw 相同的 2×2 附件；能力位
  `supports_indirect_command_buffers=true`、`max_indirect_commands=1`、
  `supported_indirect_commands=[Draw]`。多 pass / presenting pass / indexed draw / dispatch 命令均
  typed 拒绝（`icb_command_unsupported`）。证据：`cargo test -p metal-api-vulkan`
  （76 lib + 2 + 10 e2e，新增 `an_indirect_draw_replays_the_same_attachment_bytes_as_a_direct_draw`
  与 dispatch 拒绝用例）；**RTX 5060 真机**（Windows 目标编译的 `render_e2e` 二进制跑 10/10 通过，
  `indirect draw readback: 40 80 c0 ff ×4`）在
  `evidence/windows-rtx5060-icb-indirect-draw-fbf6973-2026-09-14/`；CI run `34819349891` 五 job 全绿。
- **Step 5（ICB 半）完成**：render case 新增可选 `icb` 段（`kind/max_commands/kinds/range/command`
  白名单；render case 只放行 draw），capture 侧要求恰好
  `{"kind","start","count","commands"}` 且逐字段等于 suite 声明；`provider-capture` 把 `icb` 段
  翻译成 trace 的 `indirect` 载荷，并按 marker 跳过未点名 rail 的 render case；报告段来自 provider
  自己的 `icb_replay_observations()`（不是请求回显）。规则测试在 `conformance/test_suite_v15.py`
  （222 Python 单测全绿）。CI run `34821068667` 五 job 全绿；**RTX 5060 真机** suite 路径证据在
  `evidence/windows-rtx5060-icb-suite-f9c57f0-2026-09-14/`（`icb={"kind":"draw","start":0,
  "count":1,"commands":1}` + `4080c0ff`×4）。
- **Step 4 完成**：indirect dispatch 回放（`8bcfbff`）：`VkDispatchIndirectCommand` 编码与
  `vkCmdDispatchIndirect` 回放，threadgroups 必须等于计划中唯一 full-workgroup region 的
  group_count（否则 `icb_command_unsupported`），能力位扩为 `[Draw, Dispatch]`；评审 Spec ✅ /
  Approved（证据：94 vulkan 测试 + GATES_OK）。CI run `34822802058` 五 job 全绿。
- **Step 6 第一增量完成**：`provider-capture` 把 compute case 的 `icb`（dispatch）也翻译成 trace
  载荷（单 dispatch、draw/dispatch 参数按 kind 校验、render trace 不继承 declaring case 的 icb），
  并从 `icb_replay_observations()` 报告段（`98f9d03`）。**RTX 5060 真机** suite 路径证据在
  `evidence/windows-rtx5060-icb-dispatch-suite-98f9d03-2026-09-14/`
  （`icb={"kind":"dispatch",...}` + `fefefefe` 写回）。CI run `34823203862` 五 job 全绿。
- **Step 7/8 剩余**：native 侧 `MTLHeap`/`MTLIndirectCommandBuffer`（需 Apple selftest）、
  committed `suite-v15.json` 与 v15 五路径、对象 API 的 heap/ICB 形状（Step 6 剩余）、
  indexed draw 与 aliasing。
- **Step 5（heap 半）完成**：`compare.py` 新增 heap 段规则与 `conformance/test_suite_v15.py`（合成
  suite，20 个正反例）：suite 侧 heap 白名单（单 slab、`allows_aliasing=false`、placement 覆盖
  完整 allocation、按 allocation 序、越界/重叠拒绝）+ case 级 `capture_rails` marker（有 heap 必有
  marker）；capture 侧要求点名 rail 上报恰好
  `{"heap","same_slab","placements"}` 且逐字段等于 suite 声明，未点名 rail 不得上报 case；v1–v14
  的 plan 与捕获不变（`python3 -m unittest discover` 203 tests OK，Lavapipe v13/v14 捕获实测不变）。
  **count 口径裁决**：heap 内每个 resource 仍按 distinct allocation 计数（copy_in/copy_out 语义
  不变），原因是第一增量没有 aliasing；若将来放行 aliasing，再在 v16 起改口径。
  ICB 段（`icb` 观测与 count）与 Step 4 的捕获接线一起落地。
- 仍待裁决（§9）：ICB 的等价物选择（DGC 只在 llvmpipe 可用，dzn 走 indirect draw/secondary CB）、
  placement alignment 的 provider 回填接口、heap 观测是否进报告 schema、Apple 侧 heap/ICB 的
  selftest 形式。
