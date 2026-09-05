# Metal provider contract v0

> 状态：Draft / local decision；B0 纯值 scaffold 已在本地实现，2026-09-05。
>
> 本文是 provider 接线前的可评审契约，不代表 upstream 已接受，也不代表任何生产
> rail 已切换。首版范围严格限制为 **compute-buffer**；texture、render、blit、ICB、
> window/present 和 guest packet decode 仍在契约之外。

## 1. 目的和边界

steel-brain 提出的方向要求 native Metal 和 Windows Vulkan emulator 接收同一份
Metal 语义输入。这里的“同一份”是指同一组已解析的 operation trace、资源身份、
dispatch 和可观察结果，不是让两个 provider 共享内部句柄或内存布局。

provider 位于 neutral model 和 host API 之间：

```text
neutral decode / transaction / memory owner
                |
        canonical Metal trace
                |
       +--------+--------+
       |                 |
 native Metal       Vulkan provider
```

provider 不得：

- 重新解释 vGPU packet 或拥有 guest scheduler、transaction、guest RAM lifetime；
- 在 Vulkan provider 中把 Vulkan-specific dispatch regions 暴露给 canonical trace；
- 以字符串错误或静默降级替代结构化 refusal；
- 把当前高层 `ComputeExecutor` snapshot 接口冒充最终生产 ABI。

B0 的最小执行单元是 **一个 device、一个 queue、一个 command buffer**，其中可以有
多个按序排列的 buffer-compute pass；首批只承诺 `ThreadsExact` direct dispatch。
这样既保留 Metal command-buffer 的顺序语义，也不把当前单次 snapshot executor 的
形状固化成最终 API。
B0 一条 trace 只绑定一个 logical function/pipeline contract；需要多个不同 function
的 command sequence 先拒绝，待后续 schema 明确其 per-pass identity。

当前实现状态：`metal-api-emulator` 的 `metal-api-core::provider` 已在本地提交
`b4dbb21`（纯值类型）和 `9d0ac29`（capability admission）；Vulkan 反射/limits
映射在 `be9c04e`。这些提交只增加 backend-neutral 数据模型、映射和 owner-level
测试，不改变现有 standalone/reims 执行路径，也没有定义最终 provider trait。

## 2. 生命周期和 owner

### 2.1 Device context

每个 provider device context 绑定一个明确的 guest/device epoch（离线测试可使用
独立的 off-guest context）。context 持有：

- provider identity 和 device epoch；
- 不可变 capability snapshot；
- pipeline/function cache owner；
- host completion/retirement owner。

cache 的 key 来自 guest 声明，不能跨 device epoch 共享。reims upstream-69 已将
Vulkan object cache 放到 `DeviceState`；adapter 可以请求一个由该 owner 承载的离线
`DeviceState`，但不能在 bridge 内复制另一套 cache owner。

这是目标契约，不是当前 adapter 的完整隔离承诺：现有 `ReimsVulkanExecutor` 只持有
`Arc<DeviceState>`，而 engine 的 completion/retirement 资源仍由 process-global
执行 owner 管理。per-device completion owner 要等后续 provider 接线后再实现。

### 2.2 句柄层级

| 对象 | 语义 | v0 线程/所有权规则 |
|---|---|---|
| `ProviderDevice` | capability、compile 和资源 namespace 的根 | 可被 orchestration 共享；绑定一个 epoch |
| `ProviderQueue` | 有序提交队列 | 属于一个 device；native Metal 的 thread-local 约束由 provider 内部处理 |
| `ProviderFunction` | 已解析的 function identity/module | immutable；不携带 guest raw pointer |
| `ProviderPipeline` | 编译/翻译后的 opaque pipeline | 只能用于所属 device；cache key 含 capability/translator revision |
| `ProviderBuffer` | allocation identity 的 opaque 引用 | allocation lifetime 覆盖所有 view 和 in-flight work |
| `ProviderCommandBuffer` | recording → committed → completed/failed/unknown | 单次提交；提交后不可再编码 |
| `ProviderEncoder` | command buffer 的一个 compute recording scope | 只在 owner 线程/上下文使用；end 前不得 commit |
| `CompletionToken` | 一次提交的 opaque fence/timeline identity | `Submitted` 不等于 `Completed`；不得复制成“已完成” |

trait 的具体 Rust `Send`/`Sync` 标注要由各 provider 实现，但 canonical trace 必须
是纯值类型，可跨线程传递；native queue/encoder 句柄不能被假定为 `Send`。

## 3. Canonical compute trace

### 3.1 Trace envelope

一条 trace 在 validation 后不可变，至少包含：

```text
Trace {
    schema_version,
    device_epoch,
    operation_id,
    function,
    pipeline_contract,
    encoder_policy: { dispatch_type },
    passes: [
        { pipeline, buffers, dispatch },
        ...
    ],
    completion_policy: HostReadback | SubmitOnly | FutureGuestLanding,
}
```

- `schema_version`：拒绝未知版本，不猜字段含义；
- `device_epoch`：只用于 owner validation，防止 pipeline/buffer 跨 guest lifetime 使用，
  不作为 native/Vulkan parity 的相等字段；
- `operation_id`：跨 provider 对齐日志、oracle 和 refusal；
- `function`：稳定 logical entry + semantic digest；source format 只是 provider input
  metadata，不能单独决定 parity；
- `pipeline_contract`：反射出的 descriptor/access/footprint proof；
- `passes`：同一 command buffer 内的有序 compute passes；每个 pass 的 buffers 按
  logical Metal binding 排序；
- `encoder_policy`：整个 encoder/segment 共用的 dispatch type；B0 不允许在 passes
  之间切换该值；
- `dispatch`：每个 pass 的 canonical Metal launch，不包含 Vulkan 内部 region 切分；
- `completion_policy`：v0 只承诺 CPU-visible readback 或 `SubmitOnly`（只返回 completion
  token，不返回 CPU bytes）；guest-page landing 是后续扩展，不能在 B0 偷渡 raw guest
  pointer。

### 3.2 Function 和 pipeline

```text
FunctionIdentity {
    logical_digest,
    entry_name,
    source: SanitizedLl | BinaryAir | Metallib,
}
```

`logical_digest + entry_name` 才是 Gate 1 对齐的 function identity；`source` 记录 provider
实际收到的是 MTLB、AIR 还是 sanitized IR。v0 可以拒绝无法解析的 `Metallib`，但
refusal 必须发生在 compile/resolve 边界，不能扫描容器中的“第一个 wrapper”冒充
function-name resolver。`SanitizedLl` 和 `BinaryAir` 是当前离线原型支持的输入，不等于
最终 guest MTLB 合同。只有同一 logical fixture、同一 semantic digest 才进入 parity；
原始字节不同不自动构成 parity 差异。

`PipelineContract` 至少包含：

- reflected buffer bindings 和 `Read`/`Write`/`ReadWrite` access；
- 每个可写 binding 的 normalized 静态/affine footprint proof，或明确 `Unbounded`
  refusal；这是 provider admission metadata，不是 native/Vulkan parity identity；
- dispatch kind（B0 只实现 `ThreadsExact`；`Threadgroups` 是保留的 future extension，
  provider 可 typed-refuse）；
- required local size、push-constant/argument layout，以及已规范化的 shader capability
  requirements；Vulkan-specific capability 名称只作为 provider admission metadata；
- translator revision/digest，供 cache key 和诊断记录使用。parity 比较 logical
  access/bytes，不要求两种 provider 的 proof serialization 相同。

### 3.3 BufferView

```text
BufferView {
    view_id,
    metal_binding,
    allocation_id,
    offset,
    length,
    access: Read | Write | ReadWrite | Unused,
    attribute_stride: optional,
    source: OwnedBytes | StagedLease(lease_id) | BorrowedNoCopy(lease_id),
}
```

`BufferLease` 是 trace 外部的 owner-issued capability：

```text
BufferLease {
    lease_id,
    allocation_id,
    owner_epoch,
}
```

`source` 描述请求方提供的输入方式；provider 实际选择的 Vulkan/Metal storage mode
不写回 canonical trace，只作为 capability/diagnostic metadata。离线 B0 可以使用
`OwnedBytes`，生产 no-copy 路径必须由 neutral memory owner 提供 `lease_id`。提交
成功后 provider 将本次 trace 的所有 lease reservation 关联到返回的
`CompletionToken`；owner 维护跨 pass/command-buffer 的 borrow count，并只能在所有
关联 token 进入终态后释放 lease。单个 token 不得被当成 allocation lifetime 的唯一
凭据。

约束：

1. `offset + length` 必须以 checked arithmetic 落在 allocation 范围内；
2. `view_id`、`allocation_id` 与 binding 身份分离，允许表达同一 backing 的多个 view alias；
3. `StagedLease`/`BorrowedNoCopy` 的内容和（如适用）地址必须保持到对应 completion；copy path 只能改变内部
   传输，不得改变 trace 的可观察语义；
4. provider 不接收裸 guest pointer。guest page import、mapping 和 lease 由 neutral
   memory owner 负责；
5. access 是已解析事实，不由 Vulkan descriptor 或一次 readback 结果反推；
6. `offset`、`length` 和所有 writeback range 的单位都是 bytes；
7. 在 `Submitted` 到 completion 终态之间，owner 不得 CPU mutate/reallocate
   `BorrowedNoCopy` backing；需要读写时必须经过显式同步或新的 lease reservation。

### 3.4 Dispatch

```text
Dispatch {
    kind: ThreadsExact | Threadgroups,
    grid: [u64; 3],
    threads_per_threadgroup: [u64; 3],
}
```

v0 只承诺 `encoder_policy.dispatch_type = Serial` 且 `kind = ThreadsExact` 的 direct
dispatch；`Concurrent` 或 `Threadgroups` 必须先有对应的 ordering/hazard 合同，否则
typed-refuse。canonical trace 保留 Metal/wire 的 `NSUInteger` 宽度为
`[u64; 3]`；对 `ThreadsExact`，`grid` 是 thread count；对 `Threadgroups`，`grid` 是
group count。Vulkan provider 在 adapter 边界 checked-narrow 到 engine 所需的 `u32`，
溢出必须报告稳定的 `dispatch_dimension_overflow` refusal，不得截断。Vulkan provider
可以在内部把 `ThreadsExact` 分解为 tail regions，但 canonical trace 只携带 Metal
grid/local。间接 dispatch、stage-in 和动态 imageblock 在 v0 以 typed refusal 表示，
而不是塞入未定义的默认值。

## 4. Alias、写回和可见性

alias 是必须保留的语义字段，不是优化提示：

- 相同 `allocation_id` 的多个 view 可以指向不同 range；
- overlapping writable views 必须有明确 hazard/写回策略；无法证明时拒绝；
- provider 只能按 allocation identity 产生一次 canonical writeback，或返回带 view
  range 的确定性 writeset；不能按 binding 次数盲目复制；
- 当前 `metal-api-core` 为了 MVP 拒绝同一 backing 的多 binding alias，这项 refusal
  仍然有效，直到 alias contract 和 parity case 完成；
- read-only/unused view 不得出现在 writable output 集合中。

`Readback` 的逻辑形式为：

```text
Writeback {
    view_id,
    allocation_id,
    range: { offset, length },
    bytes,
}
```

返回顺序必须稳定（allocation/view/range 的 canonical order），并且每个 byte 在报告
completion 前已经对 CPU 可见。B0 只承诺 host-visible writeback；直接写 guest pages
的 `Landed { bytes, lease }` 是后续扩展，必须等 neutral owner 的 guest-write
debt/settle 合同完成后再加入。不能把“已提交 copy”报告为“已完成”。

## 5. Completion contract

状态机固定为：

```text
Recording -> Submitted -> CompletedVisible
                     \-> Failed
                     \-> SubmittedUnknown (device loss / process failure)
```

对外结果使用显式 disposition，而不是一个含义不足的 `completion_known: bool`：

```text
CompletionDisposition =
    NotSubmitted
  | Submitted { token: CompletionToken }
  | CompletedVisible { token: CompletionToken }
  | TimedOut { token: CompletionToken }
  | Failed { token: optional }
  | DeviceLost { token: optional }
  | SubmittedUnknown { token: optional }
```

- `commit` 只证明提交成功，返回 `CompletionToken`；
- `wait(token, timeout)` 单独报告 `CompletedVisible`、`TimedOut`、`DeviceLost` 或
  `SubmittedUnknown`；`TimedOut` 是一次观察结果，不会自动把可再次等待的 token 变成
  terminal failure；
- timeout 是已提交 fence 的等待上限，不是 compile/lock/encode 的全调用 deadline；
- host readback 和 completion stamp 必须在同一 CPU-visible 合同之后；guest-page
  publication 属于后续扩展；
- provider context 被 poison/abandon 后，后续 trace 必须明确拒绝，不能复用未知状态的
  command buffer 或 pipeline。

当前 reims adapter 的同步入口只用于离线测试：它强制 retire，即使请求没有 readback。
产品异步入口的 readback-driven completion policy 不应因此改变。

## 6. Capabilities 和 admission

capability 是 device context 的 immutable snapshot，至少包括：

| 类别 | v0 字段 |
|---|---|
| dispatch | exact（B0）、threadgroups（future）、serial/concurrent、最大 local size、最大 invocations、最大 group count |
| resources | 最大 storage-buffer descriptors、单 buffer range、alias mode |
| shader | SPIR-V/AIR capability set、push-constant/argument range |
| memory | no-copy import、host-visible staging；guest-page landing 为后续扩展 |
| completion | fence/timeline、CPU wait；deferred output 为后续扩展 |

admission 顺序固定为：

1. schema、owner/epoch 和结构合法性；
2. function/pipeline resolve 与 reflection；
3. capability/footprint/descriptor limit；
4. lease 和 backing lifetime；
5. encode；
6. submit；
7. wait、readback 和 publication。

每一步都必须能说明是否已经产生副作用。未通过 admission 的 trace 不得创建本次
请求的 Vulkan/Metal command objects。

## 7. Refusal model

拒绝是稳定结构，而不是只留 `ExecutorError(String)`：

```text
ProviderError {
    phase: Resolve | Compile | Encode | Submit | Wait | Readback,
    class: Args | Capability | Resource | Compile | Execute | DeviceLost | Internal,
    slug,
    fields: { binding, allocation, requested, maximum, ... },
    retryability,
    completion: CompletionDisposition,
    detail,
}
```

规则：

- `slug` 是稳定测试/观测身份；`detail` 可变，不可反过来作为机器判断；
- fields 只保留决策所需事实，不记录裸地址、凭据或大段 shader 内容；
- submit 前 refusal 使用 `NotSubmitted`，提交后 wait/device-loss 需明确
  `SubmittedUnknown`/`DeviceLost`；
- 目标是由 provider adapter 将 error 映射到 `ComputeStatus::RailRefused`，保留 class、
  slug 和字段；当前 upstream Vulkan product path 仍有 `Unsupported`/`MetalFailed`
  映射，因此这不是现状承诺；
- 每个 provider 必须发布自己的 `ProviderError -> canonical refusal` 映射表；Gate 1
  只比较映射后的 normalized class/slug/关键 fields。未映射的 provider-specific
  refusal 是 contract gap，不得拿原始 `Unsupported`/`MetalFailed` 文本直接比较；
- 同一 trace 的比较不包含内部错误文本或 Vulkan/Metal 的句柄。

## 8. Parity oracle

同一 trace 的 Gate 1 比较以下可观察结果：

1. writable allocation/range 的 bytes；
2. completion 状态及 publication 时序；
3. read-only、alias、limit、unsupported 等映射后的 normalized refusal class/slug/fields；
4. resource visibility 和 hazard 结果；
5. deterministic ordering（同一 trace 多次执行的 writeset 顺序）。

不比较：native/Vulkan handle、allocator 地址、cache 形状、SPIR-V 内部布局、Metal
command-buffer 指针，或 provider-specific source format/artifact。source format 只用于
admission；parity 必须先确认相同的 logical fixture、entry 和 semantic digest。

测试层级：

- native Metal：macOS reference oracle；
- Vulkan provider：Windows/RTX 目标实现；
- Lavapipe：无 VM 的开发控制环境；
- VM/guest/display：只在 Gate 3 验证 wire、guest memory、present 和 completion
  end-to-end，不替代离线 parity。

## 9. 现有代码的映射

| 现有位置 | 在 v0 中的角色 | 当前状态 |
|---|---|---|
| `metal-api-core::ComputeExecutor` | snapshot compatibility adapter | 保留，不升级为最终 trait |
| `metal-api-vulkan::TranslatedComputePipeline` | translator/reflection boundary | `provider_contract()` 可生成 v0 metadata，仍不拥有 canonical trace |
| `metal-api-vulkan::VulkanExecutor` | standalone provider test backend | `provider_capabilities()` 可报告 limits；执行仍走旧 snapshot API |
| `metal-api-reims-vulkan::ReimsVulkanExecutor` | reims off-VM adapter | 复用 `DeviceState`；completion owner 仍非 per-device，非生产 canonical rail |
| `backend/metal/compute.rs` | native compute seam | 下一阶段包装，不改 neutral owner |
| `runtime/compute_exec/{metal,vulkan}.rs` | trace/staging conversion sites | 迁移时保持共享 orchestration |

当前 adapter 的 `ComputeSubmission` 缺少 allocation identity、alias range、completion
token 和结构化 error，因此只能作为 v0 contract 的 compatibility harness。任何新
provider code 都应先把这些信息补进 trace/contract，再把 snapshot API 接到测试适配器。

## 10. 实施顺序和 release gates

### Phase B0：纯值 contract/types

- 在 `metal-api-core/src/provider.rs`（或独立 `metal-api-provider` crate）定义 trace、
  BufferView/Lease、Dispatch、Capability、Completion 和 ProviderError；不引入 `ash`、
  `metal`、QEMU 或 guest pointer；
- 已在 `metal-api-core/src/provider.rs` 落地上述纯值类型和 capability admission；当前
  用 owner-level fixtures 测试 alias、range、state machine、refusal serialization；
- 保留 `ComputeExecutor` 的现有 smoke，证明兼容层没有行为回归。

本地验证：facade workspace 的 core provider/旧 API 测试共 24 个，reims adapter 3 个、
standalone Vulkan 7 个，全部通过；rustdoc、格式、core/Vulkan warnings-denied clippy
通过。`provider_contract()`/`provider_capabilities()` 尚未接入 `execute()`，因此这不
改变既有 GPU smoke 的执行路径。

### Phase B1：Vulkan provider adapter

- 在 `metal-api-vulkan/src/provider.rs` 将 `TranslatedComputePipeline` 的
  reflection/footprint 结果映射到 contract，已实现并先覆盖 `ThreadsExact`；
- 将 Vulkan 内部 region plan、descriptor 和 fence 隐藏在 provider implementation；
- 输出 canonical writeset 和 typed completion/refusal；
- standalone 与 reims adapter 使用同一 trace fixture。

### Phase B2：native Metal capture/provider

- 在 `backend/metal/compute.rs` 包装 function/PSO、buffer bind、encode、submit、wait、
  readback；
- `runtime/compute_exec/metal.rs` 只负责 neutral staging、lease 和 writeback conversion；
- 逐项记录 alias、thread-local queue、no-copy lifetime 和 nested session 差异。

### Release gates

- Gate 1：同一 trace 的 native/Vulkan parity（含 refusal cases）；
- Gate 2：canonical Metal rail 实际调用 provider，且 neutral orchestration 不旁路；
- Gate 3：VM/guest/display end-to-end；
- 任一 gate 未通过，都不删除 direct Vulkan control path，也不宣称 100% conformance。

## 11. Open decisions

- MTLB function-name table 的解析 owner 和 format version；
- `logical_digest`/semantic digest 的规范化算法、版本和 entry 组合规则；
- allocation identity 如何从 `GuestRamImport`/resource registry 传到 provider；
- overlapping writable alias 的 deterministic merge/writeback 规则；
- async provider 的 cancellation/device-loss recovery；
- native Metal oracle 的 trace capture 格式和版本兼容；
- `ProviderError` 到 canonical refusal 以及现有 `ComputeStatus`/`EncodeStatus` 的完整
  映射表。

在这些问题有真实合同或实验依据前，provider 必须 typed-refuse，不能猜测或静默复制
当前 Vulkan rail 的行为。
